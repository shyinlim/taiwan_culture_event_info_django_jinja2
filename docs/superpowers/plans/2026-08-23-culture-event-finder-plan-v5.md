# Culture Event Finder Refactor：Implementation Plan v5

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** `docs/superpowers/specs/2026-08-23-culture-event-finder-design-v5.md`

> ⚠️ **本文件自足。執行時不需開啟舊 plan。**
> `2026-08-22-culture-event-finder-plan-v4.md`、`2026-07-19-...-plan-v3.md`、
> `2026-07-18-...-plan-v2.md`、`2026-07-11-culture-event-finder-refactor.md`
> 全部 SUPERSEDED，且都包含在本 plan 順序下會造成損害的指令。**不要開它們。**

> **task 編號與 v4 不同。** 前端 scaffold 從 T4 提前成 T5，原本的 T5/T6/T7 各往後移一號；
> 原本的 T10 拆成 T10a/T10b/T10c。T8 之後的編號不變。對照表在文末附錄。

**Goal:** 把 Taiwan culture-event 的 Django/Jinja2 網站重構成 React (Vite + TS + Tailwind) SPA + Django JSON API，以單一 container 部署到 Fly.io，具備 per-country provider 架構（先只做台灣）、cache-aside 與 zh/en UI i18n。

**Architecture:** 後端分層 views (HTTP) → services (cache + 過濾/排序) → providers (每國一個外部資料源)。前端是 Vite 靜態 build，由同一個 Django container 透過 WhiteNoise 服務。無 react-router、無 DB 依賴、無 CORS。dev 環境走 docker-compose 兩個 service。

**Tech Stack:** Django 5.2 LTS、uv (依賴管理)、toolkitsy (logging)、requests、pytest + pytest-django + responses；Vite + React + TypeScript + Tailwind v4 + Vitest；Docker multi-stage；Fly.io + GitHub Actions (flyctl)。

**任務順序的設計理由：** Phase 1 先把最終目錄結構、settings、dev 環境、**以及前端 scaffold** 立好，後面每個 task 都直接在最終路徑上寫 code。不這樣做的話 settings 要改三次、compose 要寫三次、dev Dockerfile 要寫三次，而且中間態產出的東西全部會被下一步刪掉。

**前端 scaffold 為什麼排在後端之前（v5 的改動）：** makefile 的 `test` target 會 `cd frontend`。scaffold 排在後端三個 task 之後的話，那三個 task 期間 `make test` 會在那一行直接 abort，而 `make test` 是 owner 唯一背下來的指令。順序改過來之後 makefile 與 compose 各只寫一次、`make test` 從 T6 到收工全程可用。代價是 T5 結束時前端只有 Vite 的預設頁面，還沒有任何自己的東西。

## Global Constraints

以下是專案層級的要求，**每個 task 的要求都隱含包含本節**：

- **Python 3.13、Django 5.2 LTS。**
- **依賴管理單一來源是 uv。** `pyproject.toml` (PEP 621) + `uv.lock` 是 source of truth。不得留下 `requirements.txt`、`[build-system]` 或 `[tool.poetry]`。
- **`uv` binary 版本釘死為 `ghcr.io/astral-sh/uv:0.12.5`**，勿用 `:latest`。出現在兩處（`backend/Dockerfile.dev`、root `Dockerfile`），兩處必須一致。
- **dev container 用 `uv sync --frozen`（不加 `--no-dev`，要保留 pytest 等 dev deps）；prod Dockerfile 才加 `--no-dev`。**
- **backend container 的 venv 必須放在 bind mount 之外**：`ENV UV_PROJECT_ENVIRONMENT=/opt/venv`。venv 若落在 `/web/.venv`，host 的 macOS arm64 版本會覆蓋 container 的 linux 版本。
- **`fly.toml` 不得刪除。** Phase 5 仍要用它部署，Phase 6 遷移 Cloud Run 完成後才刪。
- **build 期必須同時注入 `SECRET_KEY` 與 `ALLOWED_HOSTS`。** `collectstatic`、`make test`、CI 的 `test-backend` 三處都會觸發 settings 的 fail-fast 守衛，而**兩個變數都是 fail-fast，只注入 `SECRET_KEY` 是跑不起來的**（v4 的本行與 spec §6.1 都漏了 `ALLOWED_HOSTS`）。用 `RUN SECRET_KEY=build-only-not-used ALLOWED_HOSTS=build-only python ...` 這種行內注入，**不可以用 `ENV SECRET_KEY=`**（假值會留在 image 裡變成 runtime 預設），**不可以用 `DEBUG=True` 繞過**（會把 debug 帶進 prod image）。
- **Fly.io 不會注入 `$PORT`**（Cloud Run 才會）。Dockerfile 的 `ENV PORT` 必須與 `fly.toml` 的 `[http_service].internal_port` 對齊，否則 proxy 打不到，且因 `min_machines_running = 0`（scale-to-zero）不會立刻被發現。
- **`fly.toml` 必須定義 `[[http_service.checks]]`，而且要帶 `Host` header。** `[http_service]` 不會自動生出 HTTP health check；沒有它的話 port 對不齊時 `fly deploy` 仍回報成功。少了 `Host` header 則 check 會被 `ALLOWED_HOSTS` 擋成 400。
- **`SECRET_KEY` 與 `ALLOWED_HOSTS` 都要真的 fail-fast**，非 dev 且未注入就 `ImproperlyConfigured`，兩者都不留 fallback。
- **prod gunicorn 用 `--workers 1 --threads 8 --worker-class gthread --timeout 60`。** 單 process 是為了 LocMemCache 只有一份；threads 是為了 15 秒的上游請求不會把整個 process 佔住。
- **WhiteNoise 用 plain storage**（不設 `STATICFILES_STORAGE`）：Vite 已對檔名做 content-hash，manifest storage 會重複 hash 且可能 500。
- **cache TTL：12 小時（`43200` 秒）。** Cache key 格式：`events:{country}:{category}`。
- **月份過濾用區間重疊判斷，不是比對開始月份。** 實測 MoC `category=6` 的 439 筆 showInfo 有 386 筆跨月（88%），只比 `start_time` 會讓展覽整個類別查不到。測試必須涵蓋一個 1 月開跑、12 月結束的活動查 9 月要命中。
- **地名比對前做 `臺` / `台` 正規化。** 實測有 11 筆寫 `台北市`，字面 substring 會靜默丟掉。
- **台灣的 locations 是 20 個前綴、涵蓋 22 個縣市**（實測 `宜蘭縣` 有 9 筆但舊清單裡沒有）。新竹市與新竹縣共用 `新竹` 前綴，嘉義市與嘉義縣共用 `嘉義`，所以清單長度是 20。**斷言寫 20，測試名稱寫 `test_location_prefixes_cover_22_counties`。** 把 22 當成清單長度會讓人補上新竹市與嘉義市，弄壞三處斷言加一個 checkpoint。
- **參數一律對照 provider 白名單驗證**，`isdigit()` 不夠。而且 `isdigit()` 對上標數字回 `True`（`'²'.isdigit()` 是 `True` 但 `int('²')` 拋 `ValueError`），所以判定要寫 `category.isascii() and category.isdigit()`；月份 regex 要收斂成 `(19|20)\d{2}-(0[1-9]|1[0-2])`，否則 `0000-01` 會通過驗證再死在 `strptime`。兩者的症狀都是 500 HTML 而不是合約承諾的 400 JSON。
- **`location` 的白名單比對要先做 `臺`/`台` 正規化再查集合**，與過濾階段用同一個函式。不然 `location=台北` 這個書籤會拿到 400，而它的過濾邏輯本來會命中。
- **上游回應要先確認是 `list`。** `response.json()` 只保證是合法 JSON。`payload = response.json()` 的下一行就 `if not isinstance(payload, list): raise UpstreamError(...)`，解析迴圈另外 `except (AttributeError, TypeError)` 轉 `UpstreamError`。不補的話政府 API 包一層 `{"data": [...]}` 就是 500 而不是 502，而且格式漂移的訊號完全沒亮。
- **`pytest.ini` 必須有 `python_files = test_*.py tests.py`。** pytest 預設只收 `test_*.py` 與 `*_test.py`，Django 慣例的 `health/tests.py` 會被靜默略過。實測 `pytest .` 對只有 `health/tests.py` 的目錄收到 0 個測試並以 exit code 5 結束，而 `pytest .` 正是 makefile、CI、與每一個 checkpoint 用的指令。
- **`make test` 拆成 `test-backend` 與 `test-frontend`。** `test` 呼叫兩者。前端那半在 T5 之後才接上去。
- **前端時間一律用本地時區，不可用 `toISOString()`。**
- **每個查詢回應都要帶 `meta: {rawCount, matchedCount, cacheAge}`。** 這是本專案唯一的事後診斷手段（`fly logs` 無保留期 + scale-to-zero）。前端不顯示，但 owner 一個 curl 就分得出是上游沒資料、還是自己的過濾壞了。
- **branch 策略：Phase 0–5 全程在 feature branch 進行，只有部署驗證通過後才 merge master。** master 上現有的 `deploy.yml` 硬編 `culture/tests.py`、`tech_stack/tests.py`，T2 刪掉這些 app 後若誤 merge，CI 會全紅且 `flyctl deploy` 永遠跑不到，prod 卡死且無告警。
- **`main_project/main_project/settings.py` 被本地 hook `protect_sensitive.py` 擋住 Read。** 只有 T2 會碰到它，而 T2 是整份取代，用 Bash 寫檔即可，不需要先讀。T2 之後此摩擦永久解除。
- **前端視覺 source of truth 是 `docs/poc/20260719_155200_ui_design_v27.html`**（POC gate 已於 2026-07-19 通過）。CSS 與 Tailwind class 組合皆抄自該檔，不得自行發明視覺方向；該檔唯讀，不可修改。禁用粉紅/magenta。badge 的裝飾性 emoji（「🔥 熱賣中」）owner 已於 2026-08-22 確認保留。
- **後端測試從 repo root 跑**：`cd backend && DEBUG=True uv run python -m pytest . -v`。
- **dev container 的 `CMD` 用 `uv run --frozen --no-sync`。** 裸的 `uv run` 每次啟動都重新 resolve 並 sync，而專案目錄是 bind mount 的 host repo，lock 稍微 drift 就會被 container 內的 process 改寫回你的工作目錄；Linux host 上因權限不符會直接起不來。
- **dev compose 的 backend service 要加 `user: "${UID:-1000}:${GID:-1000}"`。** build 時的 `chown` 會被 runtime 的 bind mount 蓋掉，Linux host 上 `appuser` 連 `__pycache__` 都寫不了。macOS 的 Docker Desktop 會假裝 ownership 所以本機測不出來，而 compose 存在的理由正是「誰進來環境都一致」。
- 不引入 react-router、不引入 Redux、不引入重型 i18n 套件、不引入 icon library、不引入 DRF。
- **toolkitsy 尚無 http 模組**（PyPI 0.1.0 已驗證）。外部 HTTP 用 `requests` 並隔離在 `taiwan.py`，未來單檔替換。

## Phase 0：規劃（先於所有 task）

- [x] **spec v5 + plan v5 定稿**（本文件即是）
- [x] **POC HTML gate 通過**：owner 於 2026-07-19 定案 `docs/poc/20260719_155200_ui_design_v27.html`
- [x] **emoji badge 確認**：owner 於 2026-08-22 確認「🔥 熱賣中」保留
- [ ] **owner 查 fly.io dashboard 的 billing**（blocking：**owner 未明確回覆查核結果前，不得開始 T1**）

  到 fly.io dashboard → Billing，確認現有 Fly app 是否吃 grandfathered 免費額度。
  Fly 於 2024-10-07 對新用戶取消免費方案，舊有用戶保留原額度，但方案一改就回不去。
  **若確認在扣錢，Phase 6 必須設定明確 deadline**（建議上線後 30 天內），不可停留在「隨時做」。

---

## Phase 1：骨架先立好

### Task 1: uv 遷移

**Files:**
- Modify: `pyproject.toml`（全部重寫）
- Create: `uv.lock`（generated）
- Delete: `poetry.lock`、`requirements.txt`

**Interfaces:**
- Produces: `uv run` / `uv sync` workflow，供後續所有 task 使用；deps: django 5.2.x、requests、toolkitsy、gunicorn、whitenoise；dev deps: pytest、pytest-django、responses、pytest-cov。

- [ ] **Step 1: 整份改寫 `pyproject.toml` (PEP 621)**

```toml
[project]
name = "culture-event-finder"
version = "1.0.0"
description = "Culture event search：Django JSON API + React SPA"
authors = [{ name = "taurus5650", email = "taurus_5650@hotmail.com" }]
readme = "README.md"
requires-python = ">=3.13"
dependencies = [
    "django>=5.2,<5.3",
    "requests>=2.32",
    # taiwan.py 直接 import urllib3（關掉 InsecureRequestWarning）。
    # 它目前是 requests 的傳遞依賴，但直接 import 的東西就要直接宣告，
    # 不然哪天 requests 換掉 vendored urllib3，錯誤會出現在一個看不出關聯的地方。
    "urllib3>=2.0",
    "toolkitsy>=0.1.0",
    "gunicorn>=23.0",
    "whitenoise>=6.7",
]

[dependency-groups]
dev = [
    "pytest>=8.3",
    "pytest-django>=4.10",
    "pytest-cov>=6.0",
    "responses>=0.25",
]
```

**沒有 `[build-system]`，也沒有 `[tool.poetry]`，這是刻意的。** 留著任何一段，uv 會判定這是要 build 的 package；而 prod Dockerfile 只 COPY `pyproject.toml` 與 `uv.lock`（沒有 source、沒有 README），build 會失敗。

- [ ] **Step 2: 產生 lock、安裝、刪掉 Poetry/pip 殘留**

```bash
uv lock && uv sync
git rm poetry.lock requirements.txt
```

Expected: `uv.lock` 產生、`.venv/` 有安裝內容。（若機器沒有 uv：`curl -LsSf https://astral.sh/uv/install.sh | sh`。）

- [ ] **Step 3: 機械驗證 Poetry 真的清乾淨了**

```bash
grep -q "build-system\|tool.poetry" pyproject.toml && echo "FAIL: poetry leftovers" || echo "OK: no poetry leftovers"
```

Expected: `OK: no poetry leftovers`。若是 `FAIL`，回 Step 1 重新整份取代，不要用 append 或 merge。

- [ ] **Step 4: 確認舊測試在 uv + Django 5.2 下仍然通過**

```bash
cd main_project && uv run python -m pytest . -v
```

Expected: 現有測試（culture / health_check / tech_stack）全部 PASS。若 Django 5.2 的 deprecation 造成失敗，最小幅度修正並在 commit message 註明。

- [ ] **Step 5: 確認 toolkitsy import 可用**

```bash
uv run python -c "from toolkitsy.logger import logger, configure, set_correlation_id; configure(); logger.info('toolkitsy ok')"
```

Expected: 印出含 `toolkitsy ok` 的格式化 log 行。

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml uv.lock && git add -u
git commit -m "chore: migrate dependency management from poetry to uv, add toolkitsy"
```

**本 task 的 local 驗收：** Step 3 印出 `OK`，Step 4 測試全綠，Step 5 印出 log 行。三個都對就可以停下來休息。

---

### Task 2: 清理舊 app + 重構成 `backend/` + `config/` + 全新 settings ★ checkpoint

> ⚠️ **這個 task 不可中斷。** `git mv` 之後、settings / wsgi / pytest.ini 改完之前，
> 所有 Python 進入點都指向不存在的 module，repo 處於起不來的狀態。
> 一次做完 Step 1 到 Step 10 再休息。預估要改的檔案：刪 6 個目錄、
> 新增 8 個檔案、修改 4 個檔案，中間要跑一次 pytest。

**Files:**
- Delete: `main_project/culture/`、`main_project/tech_stack/`、`main_project/utility/`、`main_project/main_project/templates/`、`main_project/main_project/views.py`、`main_project/health_check/templates/`、`main.py`、`deployment_tcei/`、Material Dashboard static assets
- Move: `main_project/` → `backend/`；`backend/main_project/` → `backend/config/`；`backend/health_check/` → `backend/health/`
- Create（全新內容）: `backend/config/settings.py`、`backend/config/urls.py`、`backend/health/views.py`、`backend/health/urls.py`、`backend/health/tests.py`、`backend/health/apps.py`、`backend/pytest.ini`
- Modify: `backend/manage.py`、`backend/config/wsgi.py`、`backend/config/asgi.py`、`.gitignore`

**Interfaces:**
- Produces: `backend/` + `config/` 最終結構；`GET /health` → `{"status": "ok"}`；`settings.py` 的 hook Read-block 摩擦解除（之後可正常用 Read / Edit）。後續所有 task 直接在這個結構上寫 code。

- [ ] **Step 1: 打回頭錨點的 tag**

```bash
git tag pre-cleanup
git tag | grep pre-cleanup
```

Expected: 印出 `pre-cleanup`。刪錯任何檔案都可以用 `git checkout pre-cleanup -- <path>` 拿回單檔。

- [ ] **Step 2: 導出 Material Dashboard 靜態資源的完整清單（不可用 `head` 截斷）**

```bash
git ls-files main_project | grep -i 'static' > /tmp/to-delete.txt
wc -l /tmp/to-delete.txt
cat /tmp/to-delete.txt
```

人工掃過整份清單，確認裡面沒有要保留的東西（這個階段 `frontend/` 還不存在，所以清單裡不可能有前端資產；若出現任何 `frontend` 字樣，停下來人工確認）。

- [ ] **Step 3: 刪除舊 apps / assets / `main.py` / `deployment_tcei/`**

```bash
git rm -r main_project/culture main_project/tech_stack main_project/utility \
  main_project/main_project/templates main.py deployment_tcei
git rm main_project/main_project/views.py
git rm main_project/health_check/templates/health_check.html
# --ignore-unmatch 不可省。Step 2 的清單是從整個 main_project 掃出來的，
# 裡面含 main_project/culture/static/*，而上面第一行已經把 culture/ 整個刪掉了。
# 少了這個旗標，git rm -f 對已不存在的 pathspec 會 exit non-zero，
# 整個 xargs 跟著倒掉，repo 就停在半遷移狀態 —— 而這是一個標了「不可中斷」的 task。
xargs git rm -f --ignore-unmatch < /tmp/to-delete.txt
ls fly.toml   # 確認還在：這是刻意保留，Phase 5 還要用它部署
```

- [ ] **Step 4: 機械驗證刪乾淨了**

```bash
git ls-files | grep -ci material || echo "0 (OK)"
git ls-files main_project | grep -ci static || echo "0 (OK)"
ls fly.toml docs/poc >/dev/null && echo "OK: fly.toml 與 docs/poc 都還在"
```

Expected: 前兩行輸出 `0 (OK)`，第三行輸出 `OK: ...`。任何一項不符就停下來，用 `git checkout pre-cleanup -- <path>` 復原後重做。

- [ ] **Step 5: 搬移目錄**

```bash
git mv main_project backend
git mv backend/main_project backend/config
git mv backend/health_check backend/health
```

- [ ] **Step 6: 寫全新 `backend/config/settings.py`（整份取代）**

這個檔案在舊路徑被 hook 擋住 Read，但這裡是**整份新建在新路徑**，直接寫即可，不需要先讀舊的。

```python
import os
from pathlib import Path

from django.core.exceptions import ImproperlyConfigured

BASE_DIR = Path(__file__).resolve().parent.parent      # backend/
REPO_ROOT = BASE_DIR.parent
FRONTEND_DIST = REPO_ROOT / "frontend" / "dist"

DEBUG = os.environ.get("DEBUG", "False") == "True"

# dev fallback 只在 DEBUG=True 時生效。非 dev 而沒注入就直接爆炸，不准安靜地照常運作。
# 這代表 collectstatic、make test、CI 都必須注入一個假的 SECRET_KEY（Global Constraints）。
_DEV_FALLBACK = {
    "SECRET_KEY": "django-insecure-dev-only-key",
    "ALLOWED_HOSTS": "localhost,127.0.0.1,backend",
}


def _required_env(name: str) -> str:
    value = os.environ.get(name)
    if value:
        return value
    if DEBUG:
        return _DEV_FALLBACK[name]
    raise ImproperlyConfigured(
        f"{name} must be set via environment variable when DEBUG is not True"
    )


SECRET_KEY = _required_env("SECRET_KEY")
# prod 網域一律由平台注入（Phase 5 是 fly.toml 的 [env]，Phase 6 換成 Cloud Run）。
# fallback 刻意不含 .fly.dev： 放了的話 env 漏注入時 Django 會安靜地照常運作。
ALLOWED_HOSTS = _required_env("ALLOWED_HOSTS").split(",")

INSTALLED_APPS = [
    "django.contrib.contenttypes",
    "django.contrib.staticfiles",
    "health",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.middleware.common.CommonMiddleware",
]

ROOT_URLCONF = "config.urls"
WSGI_APPLICATION = "config.wsgi.application"

# DIRS 是空的：SPA 的 index.html 由 config/urls.py 的 spa() 直接讀 bytes 回傳，
# 不走 template engine（T12 有說明為什麼）。這裡留一份最小設定只是為了讓
# Django 的 system check 不抱怨。
TEMPLATES = [{
    "BACKEND": "django.template.backends.django.DjangoTemplates",
    "DIRS": [],
    "APP_DIRS": False,
    "OPTIONS": {"context_processors": []},
}]

# 無 models：資料全來自外部 API + cache。sqlite in-memory 只為了讓 pytest-django 跑得起來。
DATABASES = {"default": {"ENGINE": "django.db.backends.sqlite3", "NAME": ":memory:"}}

STATIC_URL = "/static/"
STATIC_ROOT = REPO_ROOT / "staticfiles"
STATICFILES_DIRS = [FRONTEND_DIST] if FRONTEND_DIST.exists() else []
# WhiteNoise 刻意用 plain storage（預設值，不設 STATICFILES_STORAGE）：
# Vite 已經對檔名做 content-hash，manifest storage 會重複 hash 且可能 500（spec §6.1）
#
# 但 plain storage 要自己教 WhiteNoise 認得 Vite 的檔名。它預設認的是
# name.<12 位 hex>.ext，Vite 產的是 index-DcJk2sLm.js（破折號 + base64url），
# 一個都不符合，結果是每個已經 hash 過的資產都拿到 max-age=60 而不是 immutable，
# 回訪的使用者每分鐘重下載整包 JS。這個問題只在 prod 出現、而且永遠不會報錯，
# 本機與 CI 都看不到（spec §6.1）。
def _vite_hashed(path, url):
    return url.startswith(STATIC_URL) and "-" in url.rsplit("/", 1)[-1]


WHITENOISE_IMMUTABLE_FILE_TEST = _vite_hashed

LANGUAGE_CODE = "en-us"
TIME_ZONE = "Asia/Taipei"
USE_TZ = False
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
    }
}

from toolkitsy.logger import configure as _configure_logging  # noqa: E402

_configure_logging()  # console only；Fly 直接收 stdout
```

`events` app 與 `CorrelationIdMiddleware` 現在還不存在，T7 會把它們加進來。

- [ ] **Step 7: 全新路由與 `health` app**

`backend/config/urls.py`：

```python
from django.urls import include, path

urlpatterns = [
    path("health", include("health.urls")),
]
```

`backend/health/urls.py`：

```python
from django.urls import path

from . import views

urlpatterns = [path("", views.health, name="health")]
```

`backend/health/views.py`：

```python
from django.http import JsonResponse


def health(request):
    """Liveness only：deliberately touches no external dependency.

    上游（MoC）掛掉時這裡照樣回 200，所以它只夠給 Fly 的 health check 用。
    真正的上游監控是 uptime 服務去打一個真實的搜尋 URL（spec §3.4、T16 Step 8）。
    """
    return JsonResponse({"status": "ok"})
```

`backend/health/tests.py`（整份取代）：

```python
import json


def test_health(client):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert json.loads(resp.content) == {"status": "ok"}
```

`backend/health/apps.py`：

```python
from django.apps import AppConfig


class HealthConfig(AppConfig):
    name = "health"
```

- [ ] **Step 8: 更新模組參照與 pytest 設定**

`backend/manage.py`（整份取代：舊版第 5 行 import 了已刪除的 `utility`）：

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys


def main():
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


if __name__ == '__main__':
    main()
```

`backend/config/wsgi.py` 與 `backend/config/asgi.py`：把 `main_project.settings` 改成 `config.settings`（各一行 `os.environ.setdefault` 字串取代）。

`backend/pytest.ini`（整份取代）：

```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
addopts = --nomigrations
python_files = test_*.py tests.py
```

`--nomigrations` 不可省：專案沒有任何 model，每次跑測試對 in-memory sqlite 跑一輪 contenttypes migration 是純浪費。

**`python_files` 這一行也不可省。** pytest 預設只收 `test_*.py` 與 `*_test.py`，
而下一步要寫的 `backend/health/tests.py` 走的是 Django 慣例的檔名，兩者都不符，
會被**靜默略過**。實測結果：

```
pytest.ini 沒有 python_files + 只有 health/tests.py  →  collected 0 items，exit code 5
```

exit code 5 會讓下一步驗收的 `&&` 串接直接斷掉，而錯誤訊息看起來像測試環境壞了，
不像檔名不符。往後每一次 `pytest .`（makefile、CI、每個 checkpoint 用的都是它）
也都會少跑 health 那一組而不吭聲。

- [ ] **Step 9: `.gitignore` 加 `staticfiles/`**

```bash
grep -q '^staticfiles/$' .gitignore || echo 'staticfiles/' >> .gitignore
grep -n 'staticfiles' .gitignore
```

Expected: 印出含 `staticfiles/` 的一行。

不加的話：`STATIC_ROOT` 指向它，而後續步驟用 `git add -A`，只要在那之前本機跑過一次 `collectstatic`，整包 build 產物會被 commit 進去。

- [ ] **Step 10: 驗證整條路徑**

```bash
cd backend && DEBUG=True uv run python -m pytest . -v && cd ..
```

Expected: `test_health` PASS，沒有 `ModuleNotFoundError`。

```bash
cd backend && DEBUG=True uv run python manage.py runserver 8000 &
sleep 3
curl -s http://127.0.0.1:8000/health; echo
kill %1
```

Expected: `{"status": "ok"}`

再確認 fail-fast 真的會 fail：

```bash
cd backend && uv run python -c "import config.settings" 2>&1 | tail -1
```

Expected: 含 `ImproperlyConfigured` 與 `SECRET_KEY must be set` 的錯誤訊息。若這裡沒有爆炸，代表守衛沒生效，回 Step 6 檢查。

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "refactor: remove legacy jinja2 apps, restructure into backend/config with fresh settings"
```

**本 task 的 local 驗收：** `curl http://127.0.0.1:8000/health` 回 `{"status": "ok"}`，且刻意不設 `DEBUG` 時 settings 會 `ImproperlyConfigured`。兩個都對就可以停。

---

### Task 3: dev docker-compose（先只有 backend service）

**Files:**
- Create: `backend/Dockerfile.dev`、`docker-compose.dev.yml`（repo root）
- Modify: `makefile`（整份取代）

**Interfaces:**
- Consumes: T2 產出的 `backend/` 結構
- Produces: `make dev` 起 backend（:8000）；`make test`、`make dev-reset`、`make install-host`、`make run-prod` 四個 target。T4 會把 frontend service 加進同一個 compose 檔。

**frontend service 刻意還不加。** `frontend/` 目錄要到 T4 才存在，先寫進 compose 會讓 `make dev` 直接 build 失敗。T4 的最後一步負責補上並驗證 HMR。

- [ ] **Step 1: 建立 `backend/Dockerfile.dev`**

```dockerfile
FROM python:3.13-slim
LABEL purpose='dev-only, not for production'

# venv 必須放在 bind mount (/web) 之外。compose 會把 host repo 掛到 /web，
# 若 venv 建在 /web/.venv，host 的 macOS arm64 版本會覆蓋 container 內的 linux 版本，
# uv run 會拿到錯的 python，或就地重建 venv 把 linux binary 寫回 host repo。
ENV UV_PROJECT_ENVIRONMENT=/opt/venv
ENV PYTHONUNBUFFERED=1 TZ=Asia/Taipei PATH="/opt/venv/bin:$PATH"

RUN groupadd -r appuser && useradd -r -g appuser -m appuser

COPY --from=ghcr.io/astral-sh/uv:0.12.5 /uv /uvx /bin/

WORKDIR /web
COPY pyproject.toml uv.lock ./
# 注意：dev container 需要 pytest 等 dev dependencies，不可加 --no-dev（僅 prod Dockerfile 加）
RUN uv sync --frozen

COPY . .
RUN chown -R appuser:appuser /web /opt/venv
USER appuser

EXPOSE 8000
# --frozen --no-sync 兩個旗標都不可省。裸的 uv run 每次啟動都會重新 resolve 並 sync，
# 而 /web 是 bind mount 的 host repo：lock 只要稍微 drift，container 內的 process
# 就會把 uv.lock 改寫回你的工作目錄。Linux host 上因為檔案屬於 host UID，
# 這一步會直接 PermissionError 起不來；macOS 上則是靜默改掉你的 lock。
CMD ["uv", "run", "--frozen", "--no-sync", "python", "backend/manage.py", "runserver", "0.0.0.0:8000"]
```

`COPY . .` 只在 build 當下複製一次；跑起來後由 compose 的 bind mount 蓋過，達成即時改即時生效。build context 是 repo root，所以 `COPY pyproject.toml uv.lock ./` 不受 Dockerfile 自己放在 `backend/` 底下影響（Docker `COPY` 一律相對 build context）。

- [ ] **Step 2: 建立 `docker-compose.dev.yml`（repo root）**

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile.dev
    environment:
      - DEBUG=True
    # build 時的 chown 會被 runtime 的 bind mount 整個蓋掉，檔案樹仍屬於 host 的 UID。
    # 不設這一行的話，Linux host 上 container 內的 appuser 連 __pycache__ 都寫不了。
    # macOS 的 Docker Desktop 會假裝 ownership，所以這個問題在 mac 上測不出來，
    # 而 compose 存在的理由正是「誰進來開發環境都一致」。
    user: "${UID:-1000}:${GID:-1000}"
    ports:
      - '8000:8000'
    volumes:
      - .:/web
    working_dir: /web
```

沒有顯式的 `networks:`：Compose 會自動建一個 default bridge，service 之間仍可用 service name 做 DNS 解析，所以 T4 的 `vite.config.ts` 寫 `http://backend:8000` 照樣運作。少一個要維護的具名資源。

`UID` / `GID` 在 macOS 與多數 Linux shell 下不是預設匯出的環境變數，所以寫了 `:-1000` 的預設值。Linux 使用者若 UID 不是 1000，在 `make dev` 之前 `export UID GID` 一次即可，README 的維運段要提一句。

- [ ] **Step 3: `.dockerignore` 加 `.venv`**

```bash
grep -q '^\.venv$' .dockerignore || echo '.venv' >> .dockerignore
grep -n '.venv' .dockerignore
```

Expected: 印出含 `.venv` 的一行。host 的 venv 不該進 build context。

- [ ] **Step 4: 整份取代 `makefile`**

```makefile
.DEFAULT_GOAL := help

.PHONY: help
help:
	@echo "  make dev            - 起 dev container (docker-compose)"
	@echo "  make dev-reset      - 砍掉 named volume 重建；裝新前端套件後要跑這個"
	@echo "  make install-host   - host 另裝一份 frontend node_modules，給 IDE 用"
	@echo "  make test           - backend + frontend 測試"
	@echo "  make test-backend   - 只跑 pytest"
	@echo "  make test-frontend  - 只跑 vitest"
	@echo "  make run-prod       - 本機 build 並跑 production container"

.PHONY: dev
dev:
	docker compose -f docker-compose.dev.yml up --build

.PHONY: dev-reset
dev-reset:
	# frontend 的 node_modules 走 named volume，只在第一次建立時從 image 複製內容。
	# 之後 package.json 改了、image 重 build 了，volume 裡仍是舊的，--build 也救不回來。
	# 症狀是「image 裡有這個套件、container 裡沒有」。裝新套件之後跑這個。
	docker compose -f docker-compose.dev.yml down -v
	docker compose -f docker-compose.dev.yml up --build

.PHONY: install-host
install-host:
	# node_modules 用 named volume 隔離在 container 內（host mac ARM 與 container linux
	# 的原生依賴不能共用，蓋掉會讓 esbuild 之類崩潰）。host 上不存在這份，編輯器的
	# TS server / eslint / import 跳轉全失效。這裡另外在 host 裝一份「只給編輯器讀」，
	# 兩份吃同一個 package-lock.json，不會漂移（spec §5.2）。
	cd frontend && npm ci

.PHONY: test-backend
test-backend:
	cd backend && DEBUG=True uv run python -m pytest . -v

.PHONY: test-frontend
test-frontend:
	# frontend/ 要到 T4 才存在。在那之前這個 target 印一行就結束，
	# 不可以直接寫 cd frontend：make 會在那一行 abort，
	# 而 make test 是 owner 唯一背下來的指令。T4 的 Step 12 會把它換成真的 npm test。
	@echo "frontend 尚未 scaffold（T4 才建），略過"

.PHONY: test
test: test-backend test-frontend

.PHONY: run-prod
run-prod:
	# 依賴 repo root 的 Dockerfile，那是 T12 才產出的檔案。
	# 在 T12 之前跑這個 target 一定失敗，不是你漏做了什麼。
	docker build -t cef-local .
	docker run --rm -e PORT=8080 -e SECRET_KEY=local-run-only -e ALLOWED_HOSTS='*' -p 8080:8080 cef-local
```

`make test` 帶 `DEBUG=True` 是必要的：settings 的守衛在非 dev 時會拒絕啟動（Global Constraints）。

- [ ] **Step 5: 驗證 backend container 起得來**

```bash
docker compose -f docker-compose.dev.yml up --build -d backend
sleep 8
curl -s http://127.0.0.1:8000/health; echo
```

Expected: `{"status": "ok"}`

確認 venv 真的在 bind mount 之外（這是最容易靜默出錯的一項）：

```bash
docker compose -f docker-compose.dev.yml exec backend sh -c 'which python; ls -d /opt/venv'
ls .venv 2>/dev/null | head -1 && echo "host 的 .venv 仍在（正常，兩份互不干擾）"
docker compose -f docker-compose.dev.yml down
```

Expected: `which python` 印出 `/opt/venv/bin/python`（**不是** `/web/.venv/bin/python`）。若印出 `/web/...`，回 Step 1 檢查 `UV_PROJECT_ENVIRONMENT`。

- [ ] **Step 6: Commit**

```bash
git add backend/Dockerfile.dev docker-compose.dev.yml makefile .dockerignore
git commit -m "feat: add dev docker-compose with backend service, venv outside bind mount"
```

**本 task 的 local 驗收：** `make dev` 起得來，另開 terminal `curl http://127.0.0.1:8000/health` 回 `{"status": "ok"}`，且 container 內 `which python` 是 `/opt/venv/bin/python`。

### Task 4: Vite + React + TS + Tailwind + Vitest scaffold ★ checkpoint

**Files:**
- Create: `frontend/`（scaffold）、`frontend/Dockerfile.dev`
- Modify: `frontend/vite.config.ts`、`frontend/tsconfig.app.json`、`frontend/src/index.css`、`frontend/package.json`、`docker-compose.dev.yml`

**Interfaces:**
- Produces: `npm run dev`（port 5173，proxy `/api` → `backend:8000`）、`npm run build`（輸出 `frontend/dist/`，asset base `/static/`）、`npm test`（vitest run）；compose 的 frontend service。

- [ ] **Step 1: Scaffold**

`frontend/` 此時不存在（T3 刻意沒建），所以不會有「目錄非空」的互動提示。

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend && npm install && npm install tailwindcss @tailwindcss/vite && npm install -D vitest
```

- [ ] **Step 2: 當場檢查 scaffold 給了什麼 tsconfig**

`npm create vite@latest` 每次拿到的 template 版本不同，下面兩個設定會直接決定後面幾個 task 過不過 `tsc`。現在看一眼，比事後 debug 便宜。

```bash
cd frontend && cat tsconfig.app.json && node -p "require('./package.json').devDependencies.vite"
```

記下三件事：

1. `resolveJsonModule` 有沒有出現（多半沒有）：Step 3 會補。
2. `erasableSyntaxOnly` 有沒有被設成 `true`：有的話，T8 的 `ApiError` 不能用
   parameter property 寫法（T8 已經寫成一般欄位，所以兩種情況都過得了）。
3. `vite` 的版號。`npm create vite@latest` 拿到的是當下最新版，底層 bundler
   可能已經換成 rolldown。目前確認相容：`@tailwindcss/vite@4.x` 的 peer 涵蓋
   Vite 5 到 8、Vite 8 要求 node `^20.19 || >=22.12`（`node:22-slim` 過關）、
   `base` / `server.proxy` / `server.host` 的語法沒變。

scaffold 完把版號釘住，避免日後重跑時配方不成立：

```bash
cd frontend && npm pkg set devDependencies.vite="$(node -p "require('./package.json').devDependencies.vite.replace('^','')")"
git add package.json package-lock.json
```

**另外一個抄 POC 時會踩到的坑**：POC v27 用的是 Tailwind v3 CDN，這裡裝的是 v4。
v3 的 `shadow-sm` 對應 v4 的 `shadow-xs`，且 `border` 的預設顏色在 v4 改成 `currentColor`。
抄過來的地方會出現陰影變重、邊框顏色跑掉的細微偏差。T10 與 T11 的視覺對照要特別比對這兩項，
看到差異先想到這裡，不要以為是自己 CSS 抄錯。

- [ ] **Step 3: `tsconfig.app.json` 補 `resolveJsonModule`**

在 `compilerOptions` 裡加一行：

```json
"resolveJsonModule": true
```

不加的話 T9 的 `import zh from "./locales/zh.json"` 會讓 `tsc --noEmit` 直接失敗，
錯誤訊息是 `Cannot find module`，看起來像檔案路徑寫錯，實際上是編譯器設定。

- [ ] **Step 4: 整份取代 `frontend/vite.config.ts`**

```ts
/// <reference types="vitest/config" />
import tailwindcss from "@tailwindcss/vite";
import react from "@vitejs/plugin-react";
import { defineConfig } from "vitest/config";

export default defineConfig(({ mode }) => ({
  // Production assets are served by Django/WhiteNoise under /static/
  base: mode === "production" ? "/static/" : "/",
  plugins: [react(), tailwindcss()],
  server: {
    // 讓 host 瀏覽器連得進 container 內的 Vite dev server
    host: "0.0.0.0",
    // container 內用 compose service 名，不是 127.0.0.1。
    // 刻意不設 changeOrigin：預設 false 時轉發保留 Host 127.0.0.1:5173，
    // Django 去掉 port 後剛好落在 ALLOWED_HOSTS 裡。設成 true 的話 Host 會變成
    // "backend"，所有 /api 立刻 400 DisallowedHost，而且只在 dev 出現、
    // 看起來像後端壞了（spec §4.2）。
    proxy: { "/api": "http://backend:8000" },
    // mac 的 docker bind mount file-watching 事件不可靠，不開 polling HMR 不會觸發。
    watch: { usePolling: true, interval: 1000 },
  },
  test: { environment: "node" },
}));
```

`defineConfig` 從 `vitest/config` 匯入，不是從 `vite`：從 `vite` 匯入時加 `test` 區塊會直接型別錯誤。

- [ ] **Step 5: `frontend/src/index.css` 內容整份取代為**

```css
@import "tailwindcss";
```

- [ ] **Step 6: `frontend/package.json` 的 scripts 加一行**

```json
"test": "vitest run"
```

- [ ] **Step 7: 建立 `frontend/Dockerfile.dev`**

```dockerfile
FROM node:22-slim

RUN mkdir -p /app && chown node:node /app
WORKDIR /app
USER node

COPY --chown=node:node package.json package-lock.json ./
RUN npm ci

COPY --chown=node:node . .

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

`node:22-slim` 與 T12 prod multi-stage 的 stage 1 版本對齊，避免 dev/prod node 版本漂移。

- [ ] **Step 8: `docker-compose.dev.yml` 加上 frontend service**

整份取代成：

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile.dev
    environment:
      - DEBUG=True
    ports:
      - '8000:8000'
    volumes:
      - .:/web
    working_dir: /web

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    environment:
      # Vite 把 chokidar 3.6.0 bundle 進 dist，該版本確實讀這個環境變數。
      # 與 vite.config.ts 的 server.watch.usePolling 是同一件事的兩個保險。
      - CHOKIDAR_USEPOLLING=true
      - CHOKIDAR_INTERVAL=1000
    ports:
      - '5173:5173'
    volumes:
      - ./frontend:/app
      # named volume 隔離 node_modules：host (mac ARM) 的版本若蓋掉 container (linux) 的，
      # esbuild 等原生依賴會直接崩潰。代價是 volume 會 stale，裝新套件後要 make dev-reset。
      - frontend_node_modules:/app/node_modules
    depends_on:
      - backend

volumes:
  frontend_node_modules: {}
```

- [ ] **Step 9: 驗證 build 與型別**

```bash
cd frontend && npx tsc --noEmit && npm run build && ls dist/assets
```

Expected: tsc 乾淨；`dist/assets/` 有 hash 過的 `index-*.js`。

- [ ] **★ Step 10: checkpoint：全棧起得來，HMR 真的會觸發**

```bash
make dev
```

另開 terminal：

```bash
curl -s -o /dev/null -w "backend: %{http_code}\n" http://127.0.0.1:8000/health
curl -s -o /dev/null -w "frontend: %{http_code}\n" http://127.0.0.1:5173/
```

Expected: 兩個都是 `200`。

驗證 HMR：編輯 `frontend/src/App.tsx`，把 `<h1>Vite + React</h1>` 改成
`<h1>Vite + React HMR test</h1>`，存檔後回頭看跑 `make dev` 那個 terminal 的 log，
應該印出一行含 `hmr update` 的訊息。看到這行就代表 bind mount + polling + Vite
整條路徑都通了。這個編輯留著即可，T11 會整份重寫 `App.tsx`。

驗證 proxy：

```bash
curl -s http://127.0.0.1:5173/health
```

Expected: `{"status": "ok"}`。這是**經由 Vite 代理**拿到的後端回應，
證明 `server.proxy` 的設定通了。

> **不要拿 `/api/v1/countries` 驗這一步。** 這個 task 現在排在後端三個 task 之前
> （v5 的順序改動），那個 endpoint 要到 T7 才存在，拿它來驗會得到 404
> 而讓人以為 proxy 壞了。`/health` 在 T2 就有了。
> 這也代表 `vite.config.ts` 的 proxy 要同時代理 `/api` 與 `/health` 兩條前綴。

`Ctrl+C` 結束。

- [ ] **Step 3: Commit**

```bash
git add frontend docker-compose.dev.yml
git commit -m "feat: scaffold vite react-ts frontend with tailwind, vitest, dev compose service"
```

**本 task 的 local 驗收：** 瀏覽器開 `http://127.0.0.1:5173` 有 Vite 預設畫面，
改一行文字會自動更新，`curl :5173/health` 經由代理拿到後端的 JSON。

- [ ] **Step 12: 把 `test-frontend` 接進 makefile**

T3 寫 makefile 時 `frontend/` 還不存在，所以 `test-frontend` 那個 target 當時是
空殼（只印一行「frontend 尚未 scaffold」）。現在把它接上：

```makefile
test-frontend:
	cd frontend && npm test
```

```bash
make test
```

Expected: 後端測試全綠，前端 vitest 印出 `no test files found` 之類的訊息但**不報錯**
（此時還沒有任何前端測試，T8 才開始寫）。

> **這一步是 v5 把 scaffold 提前的整個理由。** v4 的 makefile 在 T3 就硬寫了
> `cd frontend && npm test`，而 `frontend/` 要到後端三個 task 之後才出現，
> 所以那三個 task 期間 `make test` 會在那一行直接 abort。

---

**Phase 1 結束狀態：** 最終目錄結構已就位，settings 是全新的且 fail-fast 真的會 fail，dev 的兩個 container 都起得來，`make test` 前後端兩半都跑得動。後面每個 task 都直接在最終路徑上寫 code，不需要再改結構，也不需要再動 makefile 與 compose。

---
## Phase 2：後端

### Task 5: Provider layer (base + TaiwanProvider + registry)

**Files:**
- Create: `backend/events/__init__.py`（空檔）、`backend/events/apps.py`、`backend/events/providers/__init__.py`、`backend/events/providers/base.py`、`backend/events/providers/taiwan.py`
- Create: `backend/events/tests/__init__.py`（空檔）、`backend/events/tests/test_providers.py`

**Interfaces:**
- Produces: `Event` dataclass (`title: str, start_time: datetime, end_time: datetime|None, location: str, location_name: str|None, on_sales: str|None, price: str|None`)；`UpstreamError(Exception)`；`normalize_place(text: str) -> str`；`BaseProvider` 帶 `code/name/locations/categories` 屬性 + `fetch_events(category_id: int) -> list[Event]`；`PROVIDERS: dict[str, BaseProvider]`，皆在 `events.providers`。T6（services）與 T7（API）消費這些介面。
- **刻意不提供 `get_provider()`。** 404 的判定放在 view 層用 `PROVIDERS.get(country)`，services 不 raise `KeyError`。若讓 `except KeyError` 包住整個 service 呼叫，內部任何深層 `KeyError` 都會變成假的「不支援這個國家」，排查方向會被完全帶偏（spec §3.1）。

- [ ] **Step 1: 寫失敗的測試**

`backend/events/tests/test_providers.py`:

```python
import pytest
import requests
import responses

from events.providers import PROVIDERS
from events.providers.base import UpstreamError, normalize_place
from events.providers.taiwan import MOC_API_URL, TaiwanProvider

MOC_PAYLOAD = [
    {
        "title": "模擬音樂會1",
        "showInfo": [{
            "location": "臺北市中正區中山南路21-1號",
            "time": "2026/07/12 19:30:00",
            "locationName": "國家音樂廳",
            "onSales": "Y",
            "price": "500",
            "endTime": "2026/07/12 21:30:00",
        }],
    },
    {
        "title": "壞資料活動",
        "showInfo": [{"location": "臺中市", "time": "not-a-date"}],
    },
    {
        "title": "沒地點活動",
        "showInfo": [{"location": "", "time": "2026/07/13 10:00:00"}],
    },
]


class TestRegistry:
    def test_taiwan_registered(self):
        assert PROVIDERS["tw"].code == "tw"

    def test_unknown_country_absent(self):
        assert PROVIDERS.get("xx") is None

    def test_metadata_shape(self):
        p = PROVIDERS["tw"]
        assert {"value": "臺北", "label": {"zh": "臺北", "en": "Taipei"}} in p.locations
        assert any(c["value"] == 1 and c["label"]["zh"] == "音樂" for c in p.categories)

    def test_yilan_and_lienchiang_are_present(self):
        # 實測 MoC category 1/6/17 共 1320 筆：宜蘭縣有 9 筆，而舊清單裡沒有宜蘭，
        # 使用者永遠選不到。連江同樣缺漏。
        values = {loc["value"] for loc in PROVIDERS["tw"].locations}
        assert "宜蘭" in values
        assert "連江" in values

    def test_location_prefixes_cover_22_counties(self):
        # 20 個前綴涵蓋 22 個縣市：新竹市與新竹縣共用「新竹」，嘉義市與嘉義縣共用「嘉義」。
        # 名稱不寫 is_twenty： 那與 spec 標題的「22 個縣市」對不上，
        # 3am 讀起來像測試壞了，而修法會是補上新竹市與嘉義市、弄壞三處斷言。
        assert len(PROVIDERS["tw"].locations) == 20


class TestNormalizePlace:
    def test_converts_tai_variant(self):
        # 實測同一批資料：11 筆寫「台北市」，其餘寫「臺北市」。
        assert normalize_place("台北市中正區") == "臺北市中正區"

    def test_leaves_standard_form_alone(self):
        assert normalize_place("臺北市中正區") == "臺北市中正區"


class TestTaiwanFetch:
    @responses.activate
    def test_success_parses_and_skips_bad_rows(self):
        responses.get(MOC_API_URL, json=MOC_PAYLOAD)
        events = TaiwanProvider().fetch_events(category_id=1)
        assert len(events) == 1  # bad-time and empty-location rows skipped
        e = events[0]
        assert e.title == "模擬音樂會1"
        assert e.start_time.strftime("%Y-%m") == "2026-07"
        assert e.end_time.strftime("%Y-%m-%d") == "2026-07-12"
        assert e.location_name == "國家音樂廳"

    @responses.activate
    def test_http_error_raises_upstream_error(self):
        responses.get(MOC_API_URL, status=500)
        with pytest.raises(UpstreamError):
            TaiwanProvider().fetch_events(category_id=1)

    @responses.activate
    def test_non_json_raises_upstream_error(self):
        responses.get(MOC_API_URL, body="<html>maintenance</html>")
        with pytest.raises(UpstreamError):
            TaiwanProvider().fetch_events(category_id=1)

    @responses.activate
    def test_network_failure_raises_upstream_error(self):
        responses.get(MOC_API_URL, body=requests.ConnectionError("boom"))
        with pytest.raises(UpstreamError):
            TaiwanProvider().fetch_events(category_id=1)

    @responses.activate
    def test_timeout_raises_upstream_error(self):
        # timeout 是 verify=False 打政府 API 最可能發生的失敗模式，spec §8 明列要測。
        responses.get(MOC_API_URL, body=requests.Timeout("slow"))
        with pytest.raises(UpstreamError):
            TaiwanProvider().fetch_events(category_id=1)
```

- [ ] **Step 2: 跑測試確認會失敗**

```bash
cd backend && DEBUG=True uv run python -m pytest events/tests/test_providers.py -v
```

Expected: FAIL / collection error：`ModuleNotFoundError: No module named 'events'`。

- [ ] **Step 3: 實作**

`backend/events/apps.py`:

```python
from django.apps import AppConfig


class EventsConfig(AppConfig):
    name = "events"
```

`backend/events/providers/base.py`:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime


class UpstreamError(Exception):
    """External data source failed (network, HTTP error, bad payload)."""


def normalize_place(text: str) -> str:
    """Fold the 台 / 臺 variant so place matching does not silently drop rows.

    MoC 的 location 是自由文字。實測有 11 筆寫「台北市」、其餘寫「臺北市」，
    字面 substring 比對會把前者丟掉，而使用者看到的失敗是「查無結果」，
    與真的沒活動長得一模一樣，是最難被回報的一種 bug（spec §3.2）。
    """
    return text.replace("台", "臺")


@dataclass
class Event:
    title: str
    start_time: datetime
    end_time: datetime | None
    location: str
    location_name: str | None
    on_sales: str | None
    price: str | None


class BaseProvider(ABC):
    """One provider per country: knows its data source and its option lists.

    界線（spec §3.2）：不要再加 provider factory、不要加 registry.py、
    不要讓查表做 fallback。這個 ABC 只有一個實作，它換到的是
    README「Adding a country」那幾步真的成立。
    """

    code: str                # e.g. "tw"：used in API paths
    name: dict[str, str]     # {"zh": "台灣", "en": "Taiwan"}
    locations: list[dict]    # [{"value": "臺北", "label": {"zh": "臺北", "en": "Taipei"}}]
    categories: list[dict]   # [{"value": 1, "label": {"zh": "音樂", "en": "Music"}}]

    @abstractmethod
    def fetch_events(self, category_id: int) -> list[Event]:
        """Fetch ALL events for one category. Raises UpstreamError on failure."""
```

`backend/events/providers/taiwan.py`:

```python
from datetime import datetime

import requests
import urllib3
from toolkitsy.logger import logger

from .base import BaseProvider, Event, UpstreamError

# cloud.culture.tw 的 SSL 憑證缺少 Subject Key Identifier，Python 3.13 會拒絕連線。
# 外部政府 API 無法修改其憑證，只好關閉驗證並抑制警告；只影響這一個資料源。
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

MOC_API_URL = "https://cloud.culture.tw/frontsite/trans/SearchShowAction.do"
TIME_FORMAT = "%Y/%m/%d %H:%M:%S"

# 20 個地區前綴，涵蓋台灣 22 個縣市（新竹與嘉義各含市與縣，用同一個前綴比對）。
# 舊清單只有 18 個，漏掉宜蘭與連江： 實測 MoC 資料裡宜蘭縣有 9 筆，使用者永遠選不到。
_LOCATIONS = [
    ("臺北", "Taipei"), ("新北", "New Taipei City"), ("基隆", "Keelung"),
    ("桃園", "Taoyuan"), ("新竹", "Hsinchu"), ("苗栗", "Miaoli"),
    ("臺中", "Taichung"), ("彰化", "Changhua"), ("南投", "Nantou"),
    ("雲林", "Yunlin"), ("嘉義", "Chiayi"), ("臺南", "Tainan"),
    ("高雄", "Kaohsiung"), ("屏東", "Pingtung"), ("宜蘭", "Yilan"),
    ("花蓮", "Hualien"), ("臺東", "Taitung"), ("澎湖", "Penghu"),
    ("金門", "Kinmen"), ("連江", "Lienchiang"),
]

_CATEGORIES = [
    (1, "音樂", "Music"), (2, "戲劇", "Theater"), (3, "舞蹈", "Dance"),
    (4, "親子", "Family"), (5, "獨立音樂", "Indie Music"), (6, "展覽", "Exhibition"),
    (7, "講座", "Lecture"), (8, "電影", "Movie"), (11, "綜藝", "Variety Show"),
    (17, "演唱會", "Concert"), (19, "研習課", "Workshop"), (200, "閱讀", "Reading"),
]


class TaiwanProvider(BaseProvider):
    code = "tw"
    name = {"zh": "台灣", "en": "Taiwan"}
    locations = [{"value": zh, "label": {"zh": zh, "en": en}} for zh, en in _LOCATIONS]
    categories = [{"value": cid, "label": {"zh": zh, "en": en}} for cid, zh, en in _CATEGORIES]

    def fetch_events(self, category_id: int) -> list[Event]:
        try:
            response = requests.get(
                url=MOC_API_URL,
                params={"method": "doFindTypeJ", "category": category_id},
                verify=False,
                timeout=15,
            )
        except requests.RequestException as e:
            raise UpstreamError(f"MoC API request failed: {e}") from e

        if response.status_code != 200:
            raise UpstreamError(f"MoC API returned HTTP {response.status_code}")

        try:
            payload = response.json()
        except ValueError as e:
            raise UpstreamError(f"MoC API returned non-JSON body: {e}") from e

        # response.json() 只保證是合法 JSON，不保證是陣列。政府 open data 改版包一層
        # {"data": [...]}、或維護頁回 {"message": "..."} 帶 HTTP 200，都會讓下面的
        # item 變成 str，item.get(...) 拋 AttributeError。AttributeError 不是
        # UpstreamError，view 的 except 接不到，使用者拿到 500 而不是設計好的 502，
        # 而且格式漂移的燈完全沒亮。
        if not isinstance(payload, list):
            raise UpstreamError(
                f"MoC API returned {type(payload).__name__}, expected list"
            )

        events = []
        try:
            for item in payload:
                title = item.get("title") or "Untitled"
                for show in item.get("showInfo", []):
                    event = self._parse_show(title, show)
                    if event:
                        events.append(event)
        except (AttributeError, TypeError) as e:
            # 陣列裡的元素形狀變了（例如從 dict 變成 str）。同上，要轉成 UpstreamError
            # 才會走到 502 這條設計好的路徑。
            raise UpstreamError(f"MoC API item shape changed: {e}") from e

        # 格式漂移的 sanity 訊號（spec §3.4）：MoC 改欄位名時 HTTP 仍是 200、
        # mock 仍全綠。raw 非空但一筆都 parse 不出來是唯一會亮的燈。
        if payload and not events:
            message = (
                f"MoC returned {len(payload)} raw items but none parsed："
                "payload format may have changed"
            )
            # 先 log 再 raise，順序不可顛倒。spec §3.4 稱這個訊號是「唯一會亮的東西」，
            # 而直接 raise 不會留下任何一行紀錄，等於那盞燈沒接電。
            logger.error(message)
            raise UpstreamError(message)
        return events

    def _parse_show(self, title: str, show: dict) -> Event | None:
        try:
            start_time = datetime.strptime(show["time"], TIME_FORMAT)
        except (KeyError, TypeError, ValueError):
            logger.warning(f"Skip show with bad time {show.get('time')!r} ({title})")
            return None
        location = show.get("location")
        if not location:
            logger.warning(f"Skip show without location ({title})")
            return None
        # endTime 與 time 同為 MoC 的 YYYY/MM/DD HH:MM:SS；一併轉 datetime，讓 API 的
        # startTime/endTime 格式一致 (皆 ISO 8601)。缺漏/格式異常 → None，不讓整筆掉。
        end_raw = show.get("endTime")
        end_time = None
        if end_raw:
            try:
                end_time = datetime.strptime(end_raw, TIME_FORMAT)
            except (TypeError, ValueError):
                logger.warning(f"Bad endTime {end_raw!r} ({title})")
        return Event(
            title=title,
            start_time=start_time,
            end_time=end_time,
            location=location,
            location_name=show.get("locationName"),
            on_sales=show.get("onSales"),
            price=show.get("price"),
        )
```

`backend/events/providers/__init__.py`:

```python
from .base import BaseProvider
from .taiwan import TaiwanProvider

# 直接寫成字面 dict。對一個單元素 list 做 dict comprehension 只是把
# 「這裡只有一個 provider」這件事藏起來，加第二個國家時照樣是加一行。
PROVIDERS: dict[str, BaseProvider] = {"tw": TaiwanProvider()}
```

- [ ] **Step 4: 把 `events` 加進 `INSTALLED_APPS`**

`backend/config/settings.py` 的 `INSTALLED_APPS` 改成：

```python
INSTALLED_APPS = [
    "django.contrib.contenttypes",
    "django.contrib.staticfiles",
    "events",
    "health",
]
```

- [ ] **Step 5: 跑測試確認會過**

```bash
cd backend && DEBUG=True uv run python -m pytest events/tests/test_providers.py -v
```

Expected: 12 PASS。

- [ ] **Step 6: Commit**

```bash
git add backend/events backend/config/settings.py
git commit -m "feat: add events provider layer with taiwan MoC provider (22 counties, tai-variant folding)"
```

**本 task 的 local 驗收：** 12 個測試全綠，其中 `test_yilan_and_lienchiang_are_present`
與 `test_converts_tai_variant` 是這次新補的兩個實測 bug 的守門員。

---

### Task 6: Services layer (cache-aside + 區間重疊過濾 + 排序)

**Files:**
- Create: `backend/events/services.py`
- Test: `backend/events/tests/test_services.py`

**Interfaces:**
- Consumes: `events.providers.PROVIDERS`、`Event`、`UpstreamError`、`normalize_place`（T5）。
- Produces: `search_events(provider, category_id, location, month) -> tuple[list[Event], dict]`：`month` 是 ISO `YYYY-MM`；回傳 `(matched_events, meta)`，`meta` 是 `{"rawCount": int, "matchedCount": int, "cacheAge": int|None}`；raises `UpstreamError`。常數 `CACHE_TTL_SECONDS = 43200`、`FAILURE_TTL_SECONDS = 60`。T7（API）消費這些。
- **不產出 `upstream_status()`**（v4 有，v5 移除，理由見本 task 的 code 註解與 spec §3.4）。
- **`search_events` 收的是 provider 物件不是國家代碼**，這樣 404 的判定完全留在 view 層，services 不需要也不會 raise `KeyError`。

- [ ] **Step 1: 寫失敗的測試**

`backend/events/tests/test_services.py`:

```python
from datetime import datetime
from unittest.mock import MagicMock

import pytest
import responses
from django.core.cache import cache

from events import services
from events.providers import PROVIDERS
from events.providers.base import Event, UpstreamError
from events.providers.taiwan import MOC_API_URL


def make_event(title="演出", start="2026/07/12 19:30:00", end=None, location="臺北市中正區"):
    fmt = "%Y/%m/%d %H:%M:%S"
    return Event(
        title=title,
        start_time=datetime.strptime(start, fmt),
        end_time=datetime.strptime(end, fmt) if end else None,
        location=location, location_name=None,
        on_sales="Y", price="500",
    )


@pytest.fixture(autouse=True)
def clear_cache():
    cache.clear()
    yield
    cache.clear()


@pytest.fixture
def fake_provider():
    provider = MagicMock()
    provider.code = "tw"
    provider.fetch_events.return_value = [
        make_event("七月台北", "2026/07/12 19:30:00", location="臺北市中正區"),
        make_event("七月台北較早", "2026/07/01 10:00:00", location="臺北市大安區"),
        make_event("八月台北", "2026/08/03 19:30:00", location="臺北市中正區"),
        make_event("七月高雄", "2026/07/15 19:30:00", location="高雄市鹽埕區"),
        make_event("台字異體", "2026/07/20 14:00:00", location="台北市信義區"),
        make_event("常設展", "2026/01/01 09:00:00", "2026/12/31 18:00:00", "臺北市士林區"),
    ]
    return provider


def test_filters_by_location_and_iso_month(fake_provider):
    result, _ = services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    titles = [e.title for e in result]
    assert titles == ["常設展", "七月台北較早", "七月台北", "台字異體"]  # sorted by start_time


def test_tai_variant_row_is_not_dropped(fake_provider):
    # 實測 MoC 有 11 筆寫「台北市」。字面 substring 比對會靜默丟掉它們。
    result, _ = services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    assert "台字異體" in [e.title for e in result]


def test_event_spanning_months_is_found_in_the_middle(fake_provider):
    # 實測 MoC category=6 的 439 筆 showInfo 有 386 筆跨月（88%），endTime 一筆都沒缺。
    # 只比對 start_time 的話展覽這個類別整個查不到。
    result, _ = services.search_events(fake_provider, 1, location="臺北", month="2026-09")
    assert [e.title for e in result] == ["常設展"]


def test_month_with_no_overlap_returns_empty(fake_provider):
    result, _ = services.search_events(fake_provider, 1, location="高雄", month="2026-09")
    assert result == []


def test_cache_hit_skips_second_upstream_call(fake_provider):
    services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    services.search_events(fake_provider, 1, location="高雄", month="2026-07")
    assert fake_provider.fetch_events.call_count == 1


def test_different_category_is_separate_cache_entry(fake_provider):
    services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    services.search_events(fake_provider, 2, location="臺北", month="2026-07")
    assert fake_provider.fetch_events.call_count == 2


def test_meta_separates_upstream_empty_from_filter_bug(fake_provider):
    """rawCount 大而 matchedCount 為 0，代表過濾把東西吃掉了，不是上游沒資料。

    這是這個專案唯一分得出這兩件事的方法：fly logs 沒保留期、machine scale-to-zero，
    半年後收到「查無結果」的回報時，只剩這兩個數字可讀（spec §3.1）。
    """
    _, meta = services.search_events(fake_provider, 1, location="高雄", month="2026-09")
    assert meta["rawCount"] == 6
    assert meta["matchedCount"] == 0
    assert meta["cacheAge"] is None  # 第一次是 miss


def test_second_call_reports_cache_age(fake_provider):
    services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    _, meta = services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    assert meta["cacheAge"] is not None


def test_upstream_failure_is_cached_so_it_is_not_retried_immediately(fake_provider):
    """失敗要進 cache，否則上游掛掉期間每個 request 都重打一次 15 秒的上游。

    八個 thread 全停在那裡，正是 --threads 8 要避免的狀況，而且是在上游
    最虛弱的時候加倍打它（spec §3.3）。
    """
    fake_provider.fetch_events.side_effect = UpstreamError("down")
    with pytest.raises(UpstreamError):
        services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    with pytest.raises(UpstreamError):
        services.search_events(fake_provider, 1, location="臺北", month="2026-07")
    # 第二次直接被冷卻期擋下，沒有再打上游
    assert fake_provider.fetch_events.call_count == 1


def test_failure_cooldown_is_per_category(fake_provider):
    """一個 category 掛掉不可以連坐其他 category。"""
    fake_provider.fetch_events.side_effect = UpstreamError("down")
    with pytest.raises(UpstreamError):
        services.search_events(fake_provider, 1, location="臺北", month="2026-07")

    fake_provider.fetch_events.side_effect = None
    result, _ = services.search_events(fake_provider, 2, location="臺北", month="2026-07")
    assert [e.title for e in result] == ["常設展", "七月台北較早", "七月台北", "台字異體"]


MOC_PAYLOAD = [{
    "title": "端到端音樂會",
    "showInfo": [{
        "location": "臺北市中正區中山南路21-1號",
        "time": "2026/07/12 19:30:00",
        "endTime": "2026/07/12 21:30:00",
        "locationName": "國家音樂廳",
        "onSales": "Y",
        "price": "500",
    }],
}]


@responses.activate
def test_end_to_end_from_moc_string_to_iso_month_filter():
    """不 mock provider：走真的 TaiwanProvider，從 MoC 的 "2026/07/12 19:30:00"
    字串一路走到 ISO month="2026-07" 的過濾結果。

    沒有這條測試的話，月份格式轉換的兩端各自被 mock 掉（services 測試餵已 parse 好的
    datetime、provider 測試只驗 parse 不驗過濾），沒有任何一條測試涵蓋整段（spec §8）。
    """
    responses.get(MOC_API_URL, json=MOC_PAYLOAD)
    result, _ = services.search_events(PROVIDERS["tw"], 1, location="臺北", month="2026-07")
    assert [e.title for e in result] == ["端到端音樂會"]


@responses.activate
def test_non_list_payload_becomes_upstream_error_not_attribute_error():
    """政府 API 改版包一層 {"data": [...]} 時，必須是 502 不是 500。

    不擋的話 item 會是 str、item.get 拋 AttributeError，view 的
    except UpstreamError 接不到，使用者拿到 Django 的 500 HTML（spec §3.2）。
    """
    responses.get(MOC_API_URL, json={"data": MOC_PAYLOAD})
    with pytest.raises(UpstreamError):
        services.search_events(PROVIDERS["tw"], 1, location="臺北", month="2026-07")
```

- [ ] **Step 2: 跑測試確認會失敗**

```bash
cd backend && DEBUG=True uv run python -m pytest events/tests/test_services.py -v
```

Expected: FAIL：import error（`events.services` 尚不存在）。

- [ ] **Step 3: 實作**

`backend/events/services.py`:

```python
import time
from calendar import monthrange
from datetime import datetime

from django.core.cache import cache
from toolkitsy.logger import logger

from .providers.base import BaseProvider, Event, UpstreamError, normalize_place

CACHE_TTL_SECONDS = 60 * 60 * 12  # 12h：event data changes slowly (spec §3.3)
# 失敗也要進 cache。只在成功時寫 cache 的話，上游持續失敗期間每一個 request 都會
# 重打一次 15 秒的上游，八個 thread 全停在那裡，正是 --threads 8 要避免的狀況，
# 而且是在上游最虛弱的時候加倍打它。60 秒夠短，上游恢復後最多一分鐘就重試（spec §3.3）。
FAILURE_TTL_SECONDS = 60


def _month_bounds(month: str) -> tuple[datetime, datetime]:
    """'2026-07' -> (2026-07-01 00:00:00, 2026-07-31 23:59:59)."""
    start = datetime.strptime(f"{month}-01", "%Y-%m-%d")
    last_day = monthrange(start.year, start.month)[1]
    end = start.replace(day=last_day, hour=23, minute=59, second=59)
    return start, end


def _matches(event: Event, location: str, start: datetime, end: datetime) -> bool:
    if normalize_place(location) not in normalize_place(event.location):
        return False
    # 區間重疊，不是比對開始月份。實測 88% 的展覽跨月，只比 start_time 會讓
    # 一檔 1 月開跑、12 月結束的常設展在 2 月到 12 月全部查不到（spec §3.3）。
    event_end = event.end_time or event.start_time
    return event.start_time <= end and event_end >= start


def search_events(
    provider: BaseProvider, category_id: int, location: str, month: str
) -> tuple[list[Event], dict]:
    """month is ISO 'YYYY-MM' (API contract). Raises UpstreamError when the source is down.

    收 provider 物件而不是國家代碼：404 的判定完全留在 view 層，這裡不 raise KeyError。
    回傳 (符合條件的活動, meta)。meta 是這個專案唯一的事後診斷手段（spec §3.1）：
    fly logs 沒有保留期、machine 又 scale-to-zero，半年後「查無結果」的回報進來時
    沒有任何 log 可讀，只剩這三個數字分得出是上游沒資料還是自己的過濾壞了。
    """
    cache_key = f"events:{provider.code}:{category_id}"
    failure_key = f"{cache_key}:failed"

    # 上一次失敗還在冷卻期內就直接拒絕，不要再打上游一次。
    last_failure = cache.get(failure_key)
    if last_failure is not None:
        logger.info(f"Cache hit (negative): {cache_key}")
        raise UpstreamError(f"upstream failed recently: {last_failure}")

    cached = cache.get(cache_key)
    if cached is None:
        # 這一行 log 讓 cache 這一層可以在本機被驗：同一個查詢跑兩次，
        # 第一次 miss、第二次 hit。沒有它的話 cache-aside 只被 MagicMock 驗過，
        # 而且 prod 上只能靠回應時間去猜（spec §8）。
        logger.info(f"Cache miss: {cache_key}")
        try:
            events = provider.fetch_events(category_id)
        except UpstreamError as e:
            cache.set(failure_key, str(e), FAILURE_TTL_SECONDS)
            raise
        cache.set(cache_key, (events, int(time.time())), CACHE_TTL_SECONDS)
        cache_age = None
    else:
        events, stored_at = cached
        cache_age = int(time.time()) - stored_at
        logger.info(f"Cache hit: {cache_key} (age={cache_age}s)")

    start, end = _month_bounds(month)
    matched = sorted(
        (e for e in events if _matches(e, location, start, end)),
        key=lambda e: e.start_time,
    )
    meta = {
        "rawCount": len(events),
        "matchedCount": len(matched),
        "cacheAge": cache_age,
    }
    return matched, meta
```

**沒有 `upstream_status()`，也沒有 `/health/upstream`。** v4 的設計是失敗時寫一個
cache 旗標、開一個 endpoint 讀它、監控指向那個 endpoint。這條鏈路的前提不成立，
三個理由任一個都足以讓它永遠回綠燈（spec §3.4）：旗標只在 cache **miss** 的路徑上
被寫，所以沒人搜尋的時段 MoC 掛掉不留痕跡；旗標放在 LocMemCache 而 machine 是
scale-to-zero，機器一停就消失；監控讀的是唯讀 cache view，整條鏈路沒有任何一段
真的碰到上游。取代方案是讓 uptime 監控直接打一個真實的搜尋 URL（T16 Step 8）。

- [ ] **Step 4: 跑測試確認會過**

```bash
cd backend && DEBUG=True uv run python -m pytest events/tests/test_services.py -v
```

Expected: 8 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/events/services.py backend/events/tests/test_services.py
git commit -m "feat: add events service layer with cache-aside, interval month filter, upstream flag"
```

**本 task 的 local 驗收：** 8 個測試全綠。特別確認
`test_event_spanning_months_is_found_in_the_middle` 是綠的，那條擋的是 88% 的展覽資料。

---

### Task 7: API endpoints + wiring ★ checkpoint

**Files:**
- Create: `backend/events/views.py`、`backend/events/urls.py`、`backend/events/middleware.py`
- Modify: `backend/config/urls.py`、`backend/config/settings.py`、`backend/health/urls.py`、`backend/health/views.py`、`backend/health/tests.py`
- Test: `backend/events/tests/test_api.py`

**Interfaces:**
- Consumes: `services.search_events`（T6）、`PROVIDERS`、`normalize_place`（T5）。
- Produces: `GET /api/v1/countries` → `[{code, name, locations, categories}]`；`GET /api/v1/<country>/events?category=&location=&month=` → `{"events": [{title, startTime, endTime, location, locationName, onSales, price, googleMapUrl, googleSearchUrl}], "meta": {rawCount, matchedCount, cacheAge}}`，`startTime`/`endTime` 為 ISO 8601；錯誤格式 `{"error": {"code": "...", "message": "..."}}`。前端（T8）消費這組精確欄位，`meta` 除外（前端不顯示它）。

- [ ] **Step 1: 寫失敗的測試**

`backend/events/tests/test_api.py`:

```python
import json
from datetime import datetime
from unittest.mock import patch

from events.providers.base import Event, UpstreamError


def _fake_events():
    return [Event(
        title="模擬音樂會", start_time=datetime(2026, 7, 12, 19, 30),
        end_time=datetime(2026, 7, 12, 21, 30), location="臺北市中正區中山南路21-1號",
        location_name="國家音樂廳", on_sales="Y", price="500",
    )]


class TestCountriesApi:
    def test_returns_taiwan(self, client):
        resp = client.get("/api/v1/countries")
        assert resp.status_code == 200
        data = json.loads(resp.content)
        assert data[0]["code"] == "tw"
        assert data[0]["name"]["en"] == "Taiwan"
        assert len(data[0]["locations"]) == 20
        assert len(data[0]["categories"]) == 12


class TestEventsApi:
    URL = "/api/v1/tw/events"
    OK_PARAMS = {"category": "1", "location": "臺北", "month": "2026-07"}

    @patch("events.views.services.search_events", return_value=_fake_events())
    def test_success_shape(self, _mock, client):
        resp = client.get(self.URL, self.OK_PARAMS)
        assert resp.status_code == 200
        event = json.loads(resp.content)["events"][0]
        assert event["title"] == "模擬音樂會"
        assert event["startTime"] == "2026-07-12T19:30:00"
        assert "query=" in event["googleMapUrl"]
        assert "q=" in event["googleSearchUrl"]

    @patch("events.views.services.search_events", return_value=[])
    def test_empty_result_is_200_with_empty_list(self, _mock, client):
        resp = client.get(self.URL, self.OK_PARAMS)
        assert resp.status_code == 200
        assert json.loads(resp.content) == {"events": []}

    def test_non_numeric_category_400(self, client):
        resp = client.get(self.URL, {**self.OK_PARAMS, "category": "abc"})
        assert resp.status_code == 400
        assert json.loads(resp.content)["error"]["code"] == "INVALID_PARAM"

    def test_category_outside_whitelist_400(self, client):
        # isdigit() 不夠：category=999999 會實際打上游並佔用一個 cache entry，
        # 連續丟不同 category 可以把 LocMemCache 預設的 300 個 entry 上限洗掉。
        resp = client.get(self.URL, {**self.OK_PARAMS, "category": "999999"})
        assert resp.status_code == 400
        assert json.loads(resp.content)["error"]["code"] == "INVALID_PARAM"

    def test_location_outside_whitelist_400(self, client):
        resp = client.get(self.URL, {**self.OK_PARAMS, "location": "東京"})
        assert resp.status_code == 400

    def test_bad_month_400(self, client):
        resp = client.get(self.URL, {**self.OK_PARAMS, "month": "2026/07"})
        assert resp.status_code == 400

    def test_unknown_country_404(self, client):
        resp = client.get("/api/v1/xx/events", self.OK_PARAMS)
        assert resp.status_code == 404
        assert json.loads(resp.content)["error"]["code"] == "UNKNOWN_COUNTRY"

    @patch("events.views.services.search_events", side_effect=UpstreamError("down"))
    def test_upstream_failure_502(self, _mock, client):
        resp = client.get(self.URL, self.OK_PARAMS)
        assert resp.status_code == 502
        assert json.loads(resp.content)["error"]["code"] == "UPSTREAM_ERROR"

    def test_response_has_request_id_header(self, client):
        resp = client.get("/api/v1/countries")
        assert resp.headers.get("X-Request-ID")
```

`backend/health/tests.py`（整份取代：加上 upstream endpoint 的測試）:

```python
import json

def test_health(client):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert json.loads(resp.content) == {"status": "ok"}
```

（v4 這裡還有兩個 `/health/upstream` 的測試。v5 移除了那個 endpoint，
理由見 T6 的說明與 spec §3.4：它讀的旗標在 scale-to-zero 下必然消失，
而且只有真人搜尋才會被寫，所以它永遠回綠燈。）

- [ ] **Step 2: 跑測試確認會失敗**

```bash
cd backend && DEBUG=True uv run python -m pytest events/tests/test_api.py health/tests.py -v
```

Expected: FAIL：404（路由尚未接上）/ import error。

- [ ] **Step 3: 實作 views / urls / middleware**

`backend/events/views.py`:

```python
import re
from urllib.parse import quote_plus

from django.http import JsonResponse
from toolkitsy.logger import logger

from . import services
from .providers import PROVIDERS
from .providers.base import Event, UpstreamError, normalize_place

# 年份收斂成 19xx/20xx。用 \d{4} 的話 0000-01 會通過驗證，然後死在
# strptime 的 "year 0 is out of range"，回 Django 500 HTML 而不是合約的 400 JSON。
MONTH_PATTERN = re.compile(r"^(19|20)\d{2}-(0[1-9]|1[0-2])$")


def _error(status: int, code: str, message: str) -> JsonResponse:
    return JsonResponse({"error": {"code": code, "message": message}}, status=status)


def _event_to_json(event: Event) -> dict:
    return {
        "title": event.title,
        "startTime": event.start_time.isoformat(),
        "endTime": event.end_time.isoformat() if event.end_time else None,
        "location": event.location,
        "locationName": event.location_name,
        "onSales": event.on_sales,
        "price": event.price,
        "googleMapUrl": f"https://www.google.com/maps/search/?api=1&query={quote_plus(event.location)}",
        "googleSearchUrl": f"https://www.google.com/search?q={quote_plus(event.title)}",
    }


def countries(request):
    data = [
        {"code": p.code, "name": p.name, "locations": p.locations, "categories": p.categories}
        for p in PROVIDERS.values()
    ]
    return JsonResponse(data, safe=False)


def events(request, country: str):
    # 404 在這裡判，不靠 except KeyError 包住整個 service 呼叫： 那樣的話
    # 內部任何深層 KeyError 都會變成假的「不支援這個國家」（spec §3.1）。
    provider = PROVIDERS.get(country)
    if provider is None:
        return _error(404, "UNKNOWN_COUNTRY", f"country '{country}' is not supported")

    category = request.GET.get("category", "")
    location = request.GET.get("location", "")
    month = request.GET.get("month", "")

    # isascii() 不可省。str.isdigit() 對上標數字回 True（'²'.isdigit() 是 True），
    # 而 int('²') 拋 ValueError。少了它，?category=² 會通過守衛、死在下一行的
    # int()，回 Django 500 HTML 而不是合約承諾的 400 JSON（spec §3.1）。
    if not (category.isascii() and category.isdigit()):
        return _error(400, "INVALID_PARAM", "category must be an integer")
    # 白名單比對。只擋非數字是不夠的（spec §3.1）。
    if int(category) not in {c["value"] for c in provider.categories}:
        return _error(400, "INVALID_PARAM", f"category '{category}' is not available for '{country}'")
    # 白名單比對前先正規化，與過濾階段用同一個函式。拿原值去查集合的話，
    # location=台北 這個書籤會拿到 400，而它的過濾邏輯本來會命中（spec §3.1）。
    known_locations = {normalize_place(loc["value"]) for loc in provider.locations}
    if normalize_place(location) not in known_locations:
        return _error(400, "INVALID_PARAM", f"location '{location}' is not available for '{country}'")
    if not MONTH_PATTERN.match(month):
        return _error(400, "INVALID_PARAM", "month must be YYYY-MM")

    try:
        result, meta = services.search_events(provider, int(category), location, month)
    except UpstreamError as e:
        logger.error(f"Upstream failure: {e}")
        return _error(502, "UPSTREAM_ERROR", "Data source is temporarily unavailable")

    logger.info(
        f"Search {country}/{category}/{location}/{month}: "
        f"raw={meta['rawCount']} matched={meta['matchedCount']} cacheAge={meta['cacheAge']}"
    )
    # meta 一起回。前端不顯示它，但這是 owner 半年後唯一能用 curl 分辨
    # 「上游沒資料」與「我的過濾壞了」的東西（spec §3.1）。
    return JsonResponse({"events": [_event_to_json(e) for e in result], "meta": meta})
```

`backend/events/urls.py`:

```python
from django.urls import path

from . import views

urlpatterns = [
    path("countries", views.countries, name="api_countries"),
    path("<str:country>/events", views.events, name="api_events"),
]
```

`backend/events/middleware.py`:

> **裝這個 middleware 之前先跑這一行**（spec §3.4）：
>
> ```bash
> cd backend && DEBUG=True uv run python -c \
>   "import inspect, toolkitsy.logger as L; print(inspect.getsource(L.set_correlation_id))"
> ```
>
> 要看到 `ContextVar` 或 `threading.local()`。若它用的是 module global，
> 在 `--threads 8` 之下八個 request 共用一個 process，log 會互相錯掛，
> **而錯掛的 id 比沒有 id 更糟**：它看起來像可信的追查線索。
> 確認不了就不要裝這個 middleware，直接跳到下一個檔案。

```python
import uuid

from toolkitsy.logger import set_correlation_id


class CorrelationIdMiddleware:
    """Give every request a short correlation id; toolkitsy logs include it.

    已知限制（spec §3.4）：machine 是 scale-to-zero、fly logs 只有即時串流，
    事後拿到這個 id 也還原不了當時的 log。README 有寫明這一點。
    """

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        correlation_id = uuid.uuid4().hex[:8]
        set_correlation_id(correlation_id)
        response = self.get_response(request)
        response["X-Request-ID"] = correlation_id
        return response
```

`backend/health/views.py`（整份取代）:

```python
from django.http import JsonResponse


def health(request):
    """Liveness only：deliberately touches no external dependency.

    上游（MoC）掛掉時這裡照樣回 200，所以它只夠給 Fly 的 health check 用。
    真正的上游監控是 uptime 服務去打一個真實的搜尋 URL（spec §3.4、T16 Step 8）。
    """
    return JsonResponse({"status": "ok"})
```

`backend/health/urls.py`（整份取代）:

```python
from django.urls import path

from . import views

urlpatterns = [
    path("", views.health, name="health"),
]
```

`backend/config/urls.py`（整份取代）:

```python
from django.urls import include, path

urlpatterns = [
    path("health", include("health.urls")),
    path("api/v1/", include("events.urls")),
]
```

- [ ] **Step 4: 把 middleware 加進 settings**

`backend/config/settings.py` 的 `MIDDLEWARE` 改成：

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.middleware.common.CommonMiddleware",
    "events.middleware.CorrelationIdMiddleware",
]
```

- [ ] **Step 5: 跑完整後端測試**

```bash
cd backend && DEBUG=True uv run python -m pytest . -v
```

Expected: 全部 PASS（providers 12 + services 8 + api 9 + health 3 = 32）。

- [ ] **★ Step 6: checkpoint：打真實 MoC，量資料大小，確認 contract**

> **這個 checkpoint 的通過條件刻意不接受空陣列。** 月份過濾壞掉時 API 永遠回空，
> 與「這個月剛好沒活動」在 checkpoint 上長得一模一樣（spec §8.1）。
> 所以先直接打上游確認哪個 category 有資料，再對該組合斷言筆數大於 0。

先確認上游現在有什麼，並順手量資料量（spec §10 要求把數字寫回文件）：

```bash
curl -sk "https://cloud.culture.tw/frontsite/trans/SearchShowAction.do?method=doFindTypeJ&category=6" -o /tmp/moc6.json
wc -c /tmp/moc6.json
python3 -c "
import json
d = json.load(open('/tmp/moc6.json'))
shows = [s for item in d for s in item.get('showInfo', [])]
print('items:', len(d), 'showInfo:', len(shows))
months = sorted({s['time'][:7] for s in shows if s.get('time')})
print('months present:', months[:12])
print('spanning:', sum(1 for s in shows if s.get('endTime') and s['endTime'][:7] != s.get('time','')[:7]))
"
```

從輸出的 `months present` 裡挑一個**確定有資料的月份**，記成 `HAVE_MONTH`（例如 `2026-09`）。
把 `wc -c` 與 `showInfo` 的數字補進 spec §10 的「單 category 的資料量未量測」那一條。

起 server 驗證三件事：

```bash
cd backend && DEBUG=True uv run python manage.py runserver 8000 &
sleep 3

# 1. countries 回傳形狀正確（20 locations、12 categories）
curl -s "http://127.0.0.1:8000/api/v1/countries" | python3 -c "import json,sys; d=json.load(sys.stdin); assert len(d[0]['locations'])==20, len(d[0]['locations']); assert len(d[0]['categories'])==12; print('countries OK')"

# 2. events 端點打真實 MoC，該月份必須有資料（空陣列一律算沒過）
HAVE_MONTH=2026-09   # ← 換成上面挑出來的那個月份
curl -s "http://127.0.0.1:8000/api/v1/tw/events?category=6&location=%E8%87%BA%E5%8C%97&month=${HAVE_MONTH}" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); assert 'events' in d, d; n=len(d['events']); assert n>0, f'FAIL: 0 events：月份過濾可能壞了'; print('events OK', n, 'items')"

# 3. 錯誤格式正確
curl -s -o /tmp/e1.json -w "%{http_code}\n" "http://127.0.0.1:8000/api/v1/xx/events?category=1&location=%E8%87%BA%E5%8C%97&month=2026-07"; cat /tmp/e1.json; echo
curl -s -o /tmp/e2.json -w "%{http_code}\n" "http://127.0.0.1:8000/api/v1/tw/events?category=999999&location=%E8%87%BA%E5%8C%97&month=2026-07"; cat /tmp/e2.json; echo

# 3b. 兩個「驗證通過但轉型爆炸」的洞：兩個都必須回 400 JSON，不可以是 500 HTML
curl -s -o /dev/null -w "superscript category: %{http_code}\n" "http://127.0.0.1:8000/api/v1/tw/events?category=%C2%B2&location=%E8%87%BA%E5%8C%97&month=2026-07"
curl -s -o /dev/null -w "year zero: %{http_code}\n" "http://127.0.0.1:8000/api/v1/tw/events?category=1&location=%E8%87%BA%E5%8C%97&month=0000-01"

# 3c. 台/臺 異體字書籤不可以拿到 400
curl -s -o /dev/null -w "tai variant: %{http_code}\n" "http://127.0.0.1:8000/api/v1/tw/events?category=1&location=%E5%8F%B0%E5%8C%97&month=2026-07"

# 4. meta 三個欄位都在，而且 rawCount 大於 matchedCount（有東西被過濾掉才合理）
curl -s "http://127.0.0.1:8000/api/v1/tw/events?category=1&location=%E8%87%BA%E5%8C%97&month=2026-07" \
  | python3 -c "import json,sys; m=json.load(sys.stdin)['meta']; print(m); assert m['rawCount'] >= m['matchedCount']; assert m['cacheAge'] is not None, 'FAIL: 第二次呼叫應該是 cache hit'"

kill %1
```

Expected：`countries OK`；`events OK N items` 且 **N 大於 0**；第一個 curl 回 `404` 且 body 含 `UNKNOWN_COUNTRY`；第二個回 `400` 且含 `INVALID_PARAM`；`superscript category`、`year zero` 都是 `400`（若是 `500` 代表 `isascii()` 或 month regex 漏了）；`tai variant` 是 `200`；meta 那行印出三個數字且不 assert 失敗。

**全部符合才算 checkpoint 通過。** 若第 2 項是 0 筆，回 T6 檢查 `_matches` 的區間判斷，不要放行。

- [ ] **Step 7: Commit**

```bash
git add backend
git commit -m "feat: add /api/v1 endpoints with whitelist validation, upstream health endpoint"
```

**本 task 的 local 驗收：** 32 個測試全綠，加上 Step 6 的四項 curl 全部符合。
這是進前端之前的最後一道關卡。

**Phase 2 結束狀態：** API 回得出真實資料，跨月與異體字都在測試與真實資料上驗過。

---
## Phase 3：前端

### Task 8: Types, API client, format utils (TDD)

**Files:**
- Create: `frontend/src/types.ts`、`frontend/src/api.ts`、`frontend/src/utils/format.ts`、`frontend/src/utils/search.ts`
- Test: `frontend/src/utils/format.test.ts`、`frontend/src/utils/search.test.ts`、`frontend/src/api.test.ts`

**Interfaces:**
- Consumes: backend API 形狀（T7 給的精確欄位名）。
- Produces: `fetchCountries(signal?): Promise<Country[]>`；`fetchEvents(country, params, signal?): Promise<{events: EventItem[]}>`；`ApiError` 帶 `.status`；`formatEventTime(iso)`；`currentMonth(now?)`；`withYear(month, year)`；`withMonth(month, mm)`；`resetForCountry(value, country)`。T10 / T11 消費這些。

- [ ] **Step 1: 寫失敗的測試**

`frontend/src/utils/format.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { currentMonth, formatEventTime } from "./format";

describe("formatEventTime", () => {
  it("renders MM/DD HH:mm from ISO string", () => {
    expect(formatEventTime("2026-07-12T19:30:00")).toBe("07/12 19:30");
  });
  it("returns the input when unparseable", () => {
    expect(formatEventTime("whatever")).toBe("whatever");
  });
});

describe("currentMonth", () => {
  it("uses local time, not UTC", () => {
    // 台灣時間每月 1 號 00:00 到 08:00 之間，toISOString() 會抓成上個月。
    // 後端整套釘死 Asia/Taipei，前端也必須用本地時間（spec §4.2）。
    const firstOfSeptemberEarlyMorning = new Date(2026, 8, 1, 3, 0, 0);
    expect(currentMonth(firstOfSeptemberEarlyMorning)).toBe("2026-09");
  });
  it("pads single-digit months", () => {
    expect(currentMonth(new Date(2026, 0, 15))).toBe("2026-01");
  });
});
```

`frontend/src/utils/search.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { resetForCountry, withMonth, withYear } from "./search";
import type { Country } from "../types";

const TW: Country = {
  code: "tw",
  name: { zh: "台灣", en: "Taiwan" },
  locations: [{ value: "臺北", label: { zh: "臺北", en: "Taipei" } }],
  categories: [{ value: 6, label: { zh: "展覽", en: "Exhibition" } }],
};

describe("withYear / withMonth", () => {
  it("replaces only the year part", () => {
    expect(withYear("2026-07", "2027")).toBe("2027-07");
  });
  it("replaces only the month part", () => {
    expect(withMonth("2026-07", "12")).toBe("2026-12");
  });
});

describe("resetForCountry", () => {
  it("picks the first location and category of the new country", () => {
    const next = resetForCountry(
      { country: "jp", category: "99", location: "Tokyo", month: "2026-07" },
      TW,
    );
    // 切國家時 location/category 不能留舊值或空字串： 後端白名單比對一定回 400
    expect(next).toEqual({ country: "tw", category: "6", location: "臺北", month: "2026-07" });
  });
});
```

`frontend/src/api.test.ts`:

```ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { fetchEvents } from "./api";

const fetchMock = vi.fn();
vi.stubGlobal("fetch", fetchMock);
afterEach(() => fetchMock.mockReset());

function respondWith(status: number, body: unknown) {
  fetchMock.mockResolvedValue({
    ok: status >= 200 && status < 300,
    status,
    json: () => Promise.resolve(body),
  });
}

describe("fetchEvents", () => {
  it("builds the query and returns events", async () => {
    respondWith(200, { events: [] });
    const result = await fetchEvents("tw", { category: 6, location: "臺北", month: "2026-07" });
    expect(result.events).toEqual([]);
    const url = fetchMock.mock.calls[0][0] as string;
    expect(url).toContain("/api/v1/tw/events?");
    expect(url).toContain("month=2026-07");
  });

  it("throws ApiError with server message on failure", async () => {
    respondWith(502, { error: { code: "UPSTREAM_ERROR", message: "Data source is temporarily unavailable" } });
    await expect(fetchEvents("tw", { category: 6, location: "臺北", month: "2026-07" }))
      .rejects.toMatchObject({ status: 502, message: "Data source is temporarily unavailable" });
  });
});
```

- [ ] **Step 2: 跑測試確認會失敗**

```bash
cd frontend && npm test
```

Expected: FAIL：找不到模組。

- [ ] **Step 3: 實作**

`frontend/src/types.ts`:

```ts
export interface LabeledOption {
  value: string | number;
  label: Record<string, string>; // {"zh": "...", "en": "..."}
}

export interface Country {
  code: string;
  name: Record<string, string>;
  locations: LabeledOption[];
  categories: LabeledOption[];
}

export interface EventItem {
  title: string;
  startTime: string;
  endTime: string | null;
  location: string;
  locationName: string | null;
  onSales: string | null;
  price: string | null;
  googleMapUrl: string;
  googleSearchUrl: string;
}

export interface SearchValue {
  country: string;
  category: string;
  location: string;
  month: string;
}
```

`frontend/src/api.ts`:

```ts
import type { Country, EventItem } from "./types";

const API_BASE = "/api/v1";

export class ApiError extends Error {
  // 刻意不用 parameter property (constructor(public status: number))：
  // 那是會產生 runtime code 的 TS 語法，scaffold 若開了 erasableSyntaxOnly 會編不過。
  status: number;

  constructor(status: number, message: string) {
    super(message);
    this.status = status;
  }
}

async function getJson<T>(path: string, signal?: AbortSignal): Promise<T> {
  const resp = await fetch(`${API_BASE}${path}`, { signal });
  if (!resp.ok) {
    let message = `HTTP ${resp.status}`;
    try {
      const body = await resp.json();
      message = body?.error?.message ?? message;
    } catch {
      // keep default message
    }
    throw new ApiError(resp.status, message);
  }
  return resp.json();
}

export function fetchCountries(signal?: AbortSignal): Promise<Country[]> {
  return getJson("/countries", signal);
}

export function fetchEvents(
  country: string,
  params: { category: string | number; location: string; month: string },
  signal?: AbortSignal,
): Promise<{ events: EventItem[] }> {
  const query = new URLSearchParams({
    category: String(params.category),
    location: params.location,
    month: params.month,
  });
  return getJson(`/${country}/events?${query}`, signal);
}
```

`frontend/src/utils/format.ts`:

```ts
export function formatEventTime(iso: string): string {
  const match = iso.match(/^\d{4}-(\d{2})-(\d{2})T(\d{2}):(\d{2})/);
  if (!match) return iso;
  const [, month, day, hour, minute] = match;
  return `${month}/${day} ${hour}:${minute}`;
}

/** 顯示活動的時間「區間」，不是只有開始時間。
 *
 * 後端整節在打區間重疊比對的仗，endTime 也一路傳到這裡，卡片只顯示 startTime
 * 等於把區間丟掉。實測展覽有 88% 跨月，使用者查九月會看到每一張卡都寫「01/01」，
 * 合理判斷這個站的資料是舊的（spec §4）。
 *
 * 同月：07/12 19:30 – 07/14
 * 跨月：2026/01/01 – 2026/12/31（跨月時補上年份，不然 01/01 – 12/31 看不出是哪年）
 * 無 endTime：只顯示開始時間
 */
export function formatEventRange(startIso: string, endIso: string | null): string {
  const start = formatEventTime(startIso);
  if (!endIso) return start;

  const startYm = startIso.slice(0, 7);
  const endYm = endIso.slice(0, 7);
  if (startYm === endYm) {
    // 同月：結束只給日期，時間省略（多數活動每天的結束時間相同，重複沒有資訊量）
    return `${start} – ${endIso.slice(8, 10) === startIso.slice(8, 10) ? endIso.slice(11, 16) : endIso.slice(5, 10).replace("-", "/")}`;
  }
  return `${startIso.slice(0, 10).replace(/-/g, "/")} – ${endIso.slice(0, 10).replace(/-/g, "/")}`;
}

/** 票價是 MoC 的自由文字欄位，不可以無條件前綴 $。
 *
 * 實測會出現「洽詢主辦單位」「0」「免費」。直接寫 `$ {price}` 會產出
 * 「$ 洽詢主辦單位」與「$ 0」（spec §4）。回傳 null 代表這一格不要渲染。
 */
export function formatPrice(price: string | null, freeLabel: string): string | null {
  if (!price) return null;
  const trimmed = price.trim();
  if (!trimmed) return null;
  if (!/^\d+$/.test(trimmed)) return trimmed;        // 自由文字，原樣顯示不加前綴
  return Number(trimmed) === 0 ? freeLabel : `$ ${trimmed}`;
}

/** "YYYY-MM" in LOCAL time.
 *
 * 不可以用 new Date().toISOString().slice(0, 7)： 那是 UTC，台灣時間每月 1 號
 * 00:00 到 08:00 之間會抓成上個月，而且極難重現（spec §4.2）。
 * 收一個可選的 Date 參數是為了可測。
 */
export function currentMonth(now: Date = new Date()): string {
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, "0")}`;
}
```

`frontend/src/utils/search.ts`:

```ts
import type { Country, SearchValue } from "../types";

export function withYear(month: string, year: string): string {
  return `${year}-${month.slice(5, 7)}`;
}

export function withMonth(month: string, mm: string): string {
  return `${month.slice(0, 4)}-${mm}`;
}

/** Switching country must also reset location/category.
 *
 * 後端用白名單比對 location 與 category（spec §3.1），留著舊國家的值或空字串
 * 一定回 400 INVALID_PARAM。
 */
export function resetForCountry(value: SearchValue, country: Country): SearchValue {
  return {
    ...value,
    country: country.code,
    location: String(country.locations[0]?.value ?? ""),
    category: String(country.categories[0]?.value ?? ""),
  };
}
```

- [ ] **Step 4: 跑測試確認會過**

```bash
cd frontend && npm test && npx tsc --noEmit
```

Expected: 7 PASS，tsc 乾淨。

- [ ] **Step 5: Commit**

```bash
git add frontend/src
git commit -m "feat: add typed api client, local-time month util, search form pure functions"
```

**本 task 的 local 驗收：** `npm test` 7 個綠。`currentMonth` 那條測試擋的是
每月 1 號早上才會出現的時區 bug。

---

### Task 9: i18n (locales + hook + LanguageSwitch)

**Files:**
- Create: `frontend/src/locales/zh.json`、`frontend/src/locales/en.json`、`frontend/src/i18n.tsx`、`frontend/src/components/LanguageSwitch.tsx`

**Interfaces:**
- Produces: `LanguageProvider`、`useLang(): {lang, setLang}`、`useT(): (key: string) => string`、`pickLabel(label, lang)`。所有元件用 `useT()` 取文案；選項標籤用 `pickLabel`。

- [ ] **Step 1: Locale 檔案**

`frontend/src/locales/zh.json`:

```json
{
  "app.title": "藝文活動查詢",
  "nav.search": "查活動",
  "nav.about": "關於",
  "nav.theme": "切換主題",
  "nav.language": "切換語言",
  "search.country": "國家",
  "search.location": "地區",
  "search.category": "類別",
  "search.month": "月份",
  "search.year": "年份",
  "search.submit": "搜尋",
  "search.reset": "重設條件",
  "search.comingSoon": "即將推出",
  "search.slow": "第一次查詢比較慢，正在向文化部要資料",
  "results.idleTitle": "選好條件後按搜尋",
  "results.idleHint": "挑一個地區、類別與月份，看看最近有什麼可以去",
  "results.emptyTitle": "找不到符合條件的活動",
  "results.emptyHint": "試試看更換地區、類別或月份，或按搜尋卡上的「重設條件」從頭來過",
  "results.count": "筆活動",
  "error.title": "資料讀取失敗",
  "error.retry": "重新整理",
  "error.upstream": "資料來源暫時無法使用，請稍後再試",
  "error.generic": "發生錯誤，請稍後再試",
  "event.onSales": "熱賣中",
  "event.free": "免費",
  "event.search": "Google 搜尋此活動",
  "event.map": "地圖",
  "about.title": "關於這個網站",
  "about.body": "透過各國政府 open data 查詢藝文活動，目前支援台灣 (文化部)。"
}
```

`frontend/src/locales/en.json`:

```json
{
  "app.title": "Culture Event Finder",
  "nav.search": "Search",
  "nav.about": "About",
  "nav.theme": "Toggle theme",
  "nav.language": "Switch language",
  "search.country": "Country",
  "search.location": "Location",
  "search.category": "Category",
  "search.month": "Month",
  "search.year": "Year",
  "search.submit": "Search",
  "search.reset": "Reset filters",
  "search.comingSoon": "Coming soon",
  "search.slow": "First search takes a while. Fetching from the Ministry of Culture.",
  "results.idleTitle": "Pick your filters, then search",
  "results.idleHint": "Choose a location, category and month to see what is on",
  "results.emptyTitle": "No events match these filters",
  "results.emptyHint": "Try another location, category or month, or use Reset filters on the search card",
  "results.count": "events",
  "error.title": "Failed to load events",
  "error.retry": "Retry",
  "error.upstream": "Data source is temporarily unavailable. Please try again later.",
  "error.generic": "Something went wrong. Please try again later.",
  "event.onSales": "On sale",
  "event.free": "Free",
  "event.search": "Search on Google",
  "event.map": "Map",
  "about.title": "About this site",
  "about.body": "Search culture events via government open data. Currently supports Taiwan (Ministry of Culture)."
}
```

- [ ] **Step 2: `frontend/src/i18n.tsx`**

```tsx
import { createContext, useContext, useEffect, useState, type ReactNode } from "react";
import en from "./locales/en.json";
import zh from "./locales/zh.json";

const dicts = { zh, en } as const;
export type Lang = keyof typeof dicts;

const HTML_LANG: Record<Lang, string> = { zh: "zh-Hant", en: "en" };

const LangContext = createContext<{ lang: Lang; setLang: (l: Lang) => void }>({
  lang: "zh",
  setLang: () => {},
});

const LANG_KEY = "cef-lang";

export function LanguageProvider({ children }: { children: ReactNode }) {
  // 語言要持久化，做法與主題完全一樣。v4 只持久化了主題，lang 是裸的 useState("zh")：
  // 看英文的朋友每次進站都要重切一次，而且他不會知道這個站記得住主題卻記不住語言（spec §4）。
  const [lang, setLang] = useState<Lang>(() => {
    try {
      const saved = localStorage.getItem(LANG_KEY);
      if (saved === "zh" || saved === "en") return saved;
    } catch {
      // Safari 無痕模式會讓 localStorage 直接 throw。讀不到就照預設走，不要讓整頁掛掉。
    }
    return navigator.language.startsWith("en") ? "en" : "zh";
  });

  useEffect(() => {
    // 不設的話螢幕閱讀器會用錯發音字典，瀏覽器翻譯也會誤判（spec §4）
    document.documentElement.lang = HTML_LANG[lang];
    try {
      localStorage.setItem(LANG_KEY, lang);
    } catch {
      // 同上：寫不進去就算了，這一輪的切換仍然有效
    }
  }, [lang]);

  return <LangContext.Provider value={{ lang, setLang }}>{children}</LangContext.Provider>;
}

export function useLang() {
  return useContext(LangContext);
}

export function useT() {
  const { lang } = useLang();
  return (key: string) => (dicts[lang] as Record<string, string>)[key] ?? key;
}

export function pickLabel(label: Record<string, string>, lang: Lang): string {
  return label[lang] ?? label.zh ?? Object.values(label)[0] ?? "";
}
```

- [ ] **Step 3: `frontend/src/components/LanguageSwitch.tsx`**

```tsx
import { useLang } from "../i18n";

export default function LanguageSwitch({ small = false }: { small?: boolean }) {
  const { lang, setLang } = useLang();
  const next = lang === "zh" ? "en" : "zh";
  return (
    <button
      type="button"
      // 刻意沒有 aria-label。純 icon 的按鈕（主題、搜尋、關於）必須補，
      // 但這一顆看得到的字是 EN 或 中，再掛一個「切換語言」的 label 會讓
      // 可見文字與 accessible name 毫無交集，語音控制使用者說「click EN」點不到
      // （WCAG 2.5.3 Label in Name）。有可見文字時就讓文字當 accessible name（spec §4.3）。
      onClick={() => setLang(next)}
      className={`rail-btn ${small ? "h-9 w-9" : "h-11 w-11"} rounded-full flex items-center justify-center text-xs font-semibold transition hover:bg-[var(--surface-2)]`}
    >
      {lang === "zh" ? "EN" : "中"}
    </button>
  );
}
```

`small` prop 是給手機頂部 bar 用的。手機 bar 必須放得下語言切換，否則整個 i18n
在手機上等於不存在（spec §4）。

- [ ] **Step 4: 寫三個 Vitest 斷言**

`frontend/src/i18n.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import { pickLabel } from "./i18n";
import zh from "./locales/zh.json";
import en from "./locales/en.json";

describe("i18n", () => {
  it("兩份字典的 key 完全一致", () => {
    // 少一個 key 的症狀是畫面上出現原始 key 字串（例如 results.emptyHint），
    // 而那只會在切到該語言時才看得到。這裡一次比對整組。
    expect(Object.keys(zh).sort()).toEqual(Object.keys(en).sort());
  });

  it("pickLabel 在 en 下拿英文，缺英文時退回中文", () => {
    expect(pickLabel({ zh: "臺北", en: "Taipei" }, "en")).toBe("Taipei");
    expect(pickLabel({ zh: "臺北" }, "en")).toBe("臺北");
  });
});
```

`frontend/src/i18n.dom.test.tsx`（需要 `npm i -D jsdom` 並在 `vite.config.ts`
的 `test` 區塊設 `environment: "jsdom"`）:

```tsx
import { render, fireEvent } from "@testing-library/react";
import { describe, expect, it } from "vitest";

import { LanguageProvider } from "./i18n";
import LanguageSwitch from "./components/LanguageSwitch";

describe("LanguageSwitch", () => {
  it("切換後 <html lang> 跟著變", () => {
    const { getByText } = render(
      <LanguageProvider><LanguageSwitch /></LanguageProvider>,
    );
    expect(document.documentElement.lang).toBe("zh-Hant");
    fireEvent.click(getByText("EN"));
    expect(document.documentElement.lang).toBe("en");
  });
});
```

> **這一步取代 v4 的驗收方式。** v4 這個 task 的驗收只有 `npx tsc --noEmit`，
> 而「無輸出」證明的只是型別對：不證明缺 key 有 fallback、不證明語言挑對、
> 不證明切換鈕會 render。這是整份 plan 唯一一個看不到任何結果的 task（spec §8）。
>
> 若不想為了一個 DOM 測試裝 jsdom 與 testing-library，第二個檔案可以省略，
> 改成在 T11 的手動 E2E 清單裡確認 `<html lang>`。但**第一個檔案不可省**，
> 它零新依賴而且擋掉最常見的 i18n bug。

- [ ] **Step 5: 跑測試與 commit**

```bash
cd frontend && npx tsc --noEmit && npm test
git add frontend/src && git commit -m "feat: add zh/en i18n context, html lang sync, persisted language"
```

Expected: tsc 乾淨，vitest 綠燈。

Expected: tsc 乾淨無錯。這一步會用到 T4 加的 `resolveJsonModule`；若這裡報
`Cannot find module './locales/zh.json'`，回 T4 Step 3。

**本 task 的 local 驗收：** `npx tsc --noEmit` 乾淨。

---

> **T10 拆成三段（v5 的改動）。** v4 是一個 880 行的 task，做完九個元件才第一次
> 看到畫面。拆成 10a / 10b / 10c 之後每一段結束都能開瀏覽器看到那一段做出來的東西，
> 而 fixture 頁面的機制本來就有，只是從跑一次變成跑三次。
> 三段的 **Files** 與 **Interfaces** 合起來就是下面這一份。

### Task 10a: design system + 狀態元件 ★ checkpoint

做完這一段你會看到：骨架動畫、idle/空結果畫面、錯誤畫面，三種狀態的真實外觀。

**Files（10a、10b、10c 全部）:**
- Create: `frontend/src/design.css`
- Modify: `frontend/src/index.css`（加一行 import）
- Create under `frontend/src/components/`: `Icon.tsx`、`SkeletonCard.tsx`、`ErrorMessage.tsx`、`EmptyState.tsx`、`EventCard.tsx`、`EventList.tsx`、`SearchForm.tsx`

**10a 這一段只碰**：`design.css`、`index.css`、`Icon.tsx`、`SkeletonCard.tsx`、`EmptyState.tsx`、`ErrorMessage.tsx`

**Interfaces:**
- Consumes: `EventItem`、`Country`、`SearchValue`（T8）；`useT`、`useLang`、`pickLabel`（T9）；`formatEventRange`、`formatPrice`、`withYear`、`withMonth`、`resetForCountry`（T8）。
- Produces: `<SearchForm countries value onChange onSubmit loading disabled />`（`onSubmit(next?: SearchValue)`；`disabled: boolean` 是必填，countries 載完前為 `true`）；`<EventList events summary />`；`<EmptyState title hint testId? />`；`<ErrorMessage message onRetry />`；`<SkeletonCard />`；`<Icon name size? />`；`design.css` 的 class 與 CSS variables。T11 消費這些。
  （v4 這一行漏了 `disabled` 與 `testId`，而實作裡兩者都是必要的。照這一行寫 T11 會型別錯誤。）

**動工前先用瀏覽器開一次定案 POC**（`cd docs/poc && python3 -m http.server 8899`，
開 `http://127.0.0.1:8899/20260719_155200_ui_design_v27.html`），切換 dark/light
與四種結果狀態，知道成品長什麼樣再寫。

- [ ] **Step 1: `frontend/src/design.css`（抄自 v27 `<style>` 區塊，加上 a11y 補強）**

```css
/* Glass design system：source of truth: docs/poc/20260719_155200_ui_design_v27.html
   毛玻璃四要件（spec §4.1）：抽象曲線+雙色燈光供 blur 扭曲、極低不透明度面板
   （dark 2% / light 55%）+ 36-40px blur + brightness 提亮、::after sheen 高光、
   彩色光暈跨面板邊界。改值前先回 POC 檔比對。

   降級出口（spec §10）：低階手機若掉幀，把 --panel 提到 0.5 並把 .glass 的
   backdrop-filter 整行註解掉，其餘不動。 */
:root, [data-theme="dark"] {
  --wall-grad: radial-gradient(circle at 80% 20%, #1e1812 0%, #05070f 65%);
  --lamp-1: rgba(181, 112, 4, 0.25);
  --lamp-2: rgba(14, 165, 233, 0.12);
  --floor: rgba(1, 2, 8, 0.95);

  --panel: rgba(255, 255, 255, 0.02);
  --panel-border: rgba(255, 255, 255, 0.32);
  --panel-border-dim: rgba(255, 255, 255, 0.08);
  --sheen: rgba(255, 255, 255, 0.08);
  --surface-2: rgba(255, 255, 255, 0.03);
  --chip-dis: rgba(255, 255, 255, 0.04);
  --skel: rgba(255, 255, 255, 0.06);
  --text: #ffffff;
  --text-muted: #cbd5e1;
  --text-faint: #94a3b8;

  /* single copper/ochre accent */
  --accent: #B57004;
  --accent-cool: #7a4700;
  --accent-deep: #543000;
  --link: #d99c38;
}
[data-theme="light"] {
  --wall-grad: radial-gradient(circle at 80% 20%, #fef3c7 0%, #cbd5e1 65%);
  --lamp-1: rgba(181, 112, 4, 0.2);
  --lamp-2: rgba(14, 165, 233, 0.08);
  --floor: rgba(15, 23, 42, 0.05);

  --panel: rgba(255, 255, 255, 0.55);
  --panel-border: rgba(255, 255, 255, 0.95);
  --panel-border-dim: rgba(15, 23, 42, 0.15);
  --sheen: rgba(255, 255, 255, 0.35);
  --surface-2: rgba(15, 23, 42, 0.05);
  --chip-dis: rgba(15, 23, 42, 0.08);
  --skel: rgba(15, 23, 42, 0.06);
  --text: #0f172a;
  --text-muted: #334155;
  --text-faint: #64748b;
  --accent: #B57004;
  --accent-cool: #7a4700;
  --accent-deep: #543000;
  --link: #B57004;
}
html, body { height: 100%; }
body {
  font-family: -apple-system, "Noto Sans TC", sans-serif;
  background: #05070f;
  position: relative;
  isolation: isolate;
  transition: background .3s;
}
[data-theme="light"] body { background: #cbd5e1; }

.scene { position: fixed; inset: 0; z-index: -1; overflow: hidden; }
.wall { position: absolute; inset: 0; background: var(--wall-grad); transition: background .3s; }
.lamp-light {
  position: absolute; inset: 0;
  background:
    radial-gradient(ellipse 65% 50% at 85% 5%, var(--lamp-1), transparent 75%),
    radial-gradient(ellipse 55% 45% at 15% 95%, var(--lamp-2), transparent 80%);
  pointer-events: none;
}
.floor-shadow {
  position: absolute; inset: 0;
  background: linear-gradient(180deg, transparent 55%, var(--floor) 100%);
  pointer-events: none;
}

.blob { position: absolute; border-radius: 9999px; opacity: 0.75; filter: blur(36px); pointer-events: none; }
.blob-bronze { width: 550px; height: 550px; top: -120px; right: 5%;
  background: radial-gradient(circle, rgba(181,112,4,0.2) 0%, rgba(122,71,0,0.06) 50%, transparent 80%); }
.blob-cyan { width: 450px; height: 450px; bottom: 15%; left: 35%;
  background: radial-gradient(circle, rgba(14,165,233,0.15) 0%, rgba(3,105,161,0.04) 50%, transparent 80%); }

.glass {
  position: relative;
  background: var(--panel);
  border: 1.5px solid var(--panel-border-dim);
  border-top: 1.8px solid var(--panel-border);
  border-left: 1.8px solid var(--panel-border-dim);
  border-right: 1.8px solid var(--panel-border);
  box-shadow:
    0 40px 90px -25px rgba(0,0,0,0.85),
    inset 0 1px 0 rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(40px) saturate(160%) brightness(1.05);
  -webkit-backdrop-filter: blur(40px) saturate(160%) brightness(1.05);
  transition: background .3s, border-color .3s;
}
[data-theme="light"] .glass {
  backdrop-filter: blur(36px) saturate(150%) brightness(1.02);
  -webkit-backdrop-filter: blur(36px) saturate(150%) brightness(1.02);
  box-shadow: 0 40px 90px -25px rgba(15,23,42,0.1);
}
.glass::after {
  content: "";
  position: absolute; inset: 0; border-radius: inherit; pointer-events: none;
  background: linear-gradient(135deg, rgba(255,255,255,0.08) 0%, rgba(255,255,255,0.02) 20%, transparent 40%);
}
[data-theme="light"] .glass::after {
  background: linear-gradient(135deg, rgba(255,255,255,0.35) 0%, rgba(255,255,255,0.1) 25%, transparent 50%);
}
.glass > * { position: relative; z-index: 1; }

/* Unified Airbnb-style search bar */
.search-capsule {
  display: flex;
  background: var(--surface-2);
  border: 1px solid var(--panel-border-dim);
  box-shadow: 0 4px 20px -5px rgba(0,0,0,0.2);
  overflow: hidden;
  transition: box-shadow 0.3s, border-color 0.3s;
}
.search-capsule:hover, .search-capsule:focus-within {
  box-shadow: 0 8px 30px -5px rgba(0,0,0,0.3);
  border-color: rgba(255, 255, 255, 0.2);
}
[data-theme="light"] .search-capsule:hover { border-color: rgba(15, 23, 42, 0.3); }
.search-field {
  flex: 1;
  position: relative;
  padding: 0.75rem 1.25rem;
  display: flex;
  flex-direction: column;
  justify-content: center;
  border-bottom: 1px solid var(--panel-border-dim);
  transition: background 0.2s;
}
@media (min-width: 640px) {
  .search-field {
    border-bottom: none;
    border-right: 1px solid var(--panel-border-dim);
    padding: 0.5rem 1rem;
  }
}
.search-field:last-of-type { border-bottom: none; border-right: none; }
.search-field:hover, .search-field:focus-within { background: rgba(255, 255, 255, 0.04); }
[data-theme="light"] .search-field:hover,
[data-theme="light"] .search-field:focus-within { background: rgba(15, 23, 42, 0.03); }

/* appearance: none 拿掉了原生下拉箭頭，這裡補一個，否則四個欄位在視覺上是純文字，
   看不出可以點（spec §4.3） */
.search-field::after {
  content: "";
  position: absolute;
  right: 1.25rem;
  top: 50%;
  width: 8px; height: 8px;
  border-right: 2px solid var(--text-faint);
  border-bottom: 2px solid var(--text-faint);
  transform: translateY(-25%) rotate(45deg);
  pointer-events: none;
}
@media (min-width: 640px) { .search-field::after { right: 1rem; } }

.search-label {
  /* 0.65rem 的 --text-faint 疊在近乎透明的毛玻璃上，戶外強光下讀不到。
     放大到 0.75rem 並改用 --text-muted（spec §4.3） */
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  color: var(--text-muted);
  margin-bottom: 2px;
}
.search-select {
  background: transparent !important;
  color: var(--text);
  font-size: 0.875rem;
  font-weight: 600;
  border: none;
  outline: none;
  appearance: none;
  cursor: pointer;
  width: 100%;
  padding-right: 1.25rem;
}

/* outline: none 不可以沒有替代品，否則鍵盤 Tab 時完全看不到焦點在哪（spec §4.3） */
.search-select:focus-visible,
button:focus-visible,
a:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
  border-radius: 4px;
}

.btn-primary {
  background: linear-gradient(135deg, var(--accent-cool), var(--accent));
  color: #ffffff;
  border-radius: 9999px;
  font-weight: 600;
  transition: background 0.2s, transform 0.2s, box-shadow 0.2s;
}
.btn-primary:hover {
  background: var(--accent);
  transform: scale(1.02);
  box-shadow: 0 8px 25px -5px var(--accent-cool);
}

.btn-secondary {
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: var(--text-muted);
  border-radius: 9999px;
  font-weight: 500;
  transition: all 0.2s ease-out;
}
.btn-secondary:hover {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.25);
  color: #ffffff;
  transform: translateY(-1px);
}
[data-theme="light"] .btn-secondary {
  background: rgba(15, 23, 42, 0.04);
  border: 1px solid rgba(15, 23, 42, 0.12);
  color: var(--text-muted);
}
[data-theme="light"] .btn-secondary:hover {
  background: rgba(15, 23, 42, 0.1);
  border-color: rgba(15, 23, 42, 0.2);
  color: var(--text);
}

.chip.active {
  background: linear-gradient(135deg, var(--accent-cool), var(--accent)) !important;
  color: #ffffff !important;
  font-weight: 600;
  border-color: transparent !important;
  box-shadow: 0 4px 15px -3px var(--accent-cool);
}
.chip[aria-disabled="true"] { opacity: 0.6; cursor: not-allowed; }

.rail-btn.active {
  background: linear-gradient(135deg, var(--accent-cool), var(--accent));
  color: #ffffff;
  box-shadow: 0 4px 15px -3px var(--accent-cool);
}

.card-hover {
  backdrop-filter: blur(16px) saturate(140%);
  -webkit-backdrop-filter: blur(16px) saturate(140%);
}
.card-hover:hover { transform: translateY(-6px); box-shadow: 0 25px 50px -12px rgba(0,0,0,0.6); border-color: rgba(255,255,255,0.15); }
[data-theme="light"] .card-hover:hover { box-shadow: 0 25px 50px -12px rgba(15,23,42,0.2); }

.card-img-svg { transition: transform 1.5s cubic-bezier(0.16, 1, 0.3, 1); }
.card-hover:hover .card-img-svg { transform: scale(1.08); }

.icon { stroke: currentColor; fill: none; stroke-width: 1.8; stroke-linecap: round; stroke-linejoin: round; }

.state-panel {
  text-align: center;
  padding: 6rem 1rem;
  background: var(--surface-2);
  border: 1px dashed var(--panel-border-dim);
  border-radius: 1.5rem;
  margin-top: 1rem;
}
```

- [ ] **Step 2: `frontend/src/index.css` 加 import**

在 Tailwind import 之後加一行：

```css
@import "./design.css";
```

- [ ] **Step 3: `frontend/src/components/Icon.tsx`**

path 抄自 v27 的 `<symbol>` defs；React 版直接 per-name render，不走 `<use>` 間接層。
**只放實際會用到的 12 個**：八個 UI icon + 四個類別 chip icon。
v27 的 `alert` 不放，ErrorMessage 用自己的 inline path。

```tsx
// stroke icon set from docs/poc/20260719_155200_ui_design_v27.html：no icon library
const PATHS = {
  search: (
    <>
      <circle cx="11" cy="11" r="7" />
      <path d="M21 21l-4.3-4.3" />
    </>
  ),
  info: (
    <>
      <circle cx="12" cy="12" r="9" />
      <path d="M12 11v5" />
      <path d="M12 8h.01" />
    </>
  ),
  moon: <path d="M20 14.5A8.5 8.5 0 1 1 9.5 4a7 7 0 0 0 10.5 10.5z" />,
  sun: (
    <>
      <circle cx="12" cy="12" r="4" />
      <path d="M12 2v2M12 20v2M4.9 4.9l1.4 1.4M17.7 17.7l1.4 1.4M2 12h2M20 12h2M4.9 19.1l1.4-1.4M17.7 6.3l1.4-1.4" />
    </>
  ),
  calendar: (
    <>
      <rect x="3" y="5" width="18" height="16" rx="2" />
      <path d="M8 3v4M16 3v4M3 10h18" />
    </>
  ),
  pin: (
    <>
      <path d="M12 22s7-7.4 7-12.6A7 7 0 0 0 5 9.4C5 14.6 12 22 12 22z" />
      <circle cx="12" cy="9.5" r="2.3" />
    </>
  ),
  ticket: (
    <>
      <path d="M3 9a2 2 0 0 1 0 4v2a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-2a2 2 0 0 1 0-4V7a2 2 0 0 0-2-2H5a2 2 0 0 0-2 2z" />
      <path d="M12 5v14" strokeDasharray="2 3" />
    </>
  ),
  refresh: (
    <>
      <path d="M21 12a9 9 0 1 1-3-6.7" />
      <path d="M21 3v6h-6" />
    </>
  ),
  music: (
    <>
      <path d="M9 18V5l12-2v13" />
      <circle cx="6" cy="18" r="3" />
      <circle cx="18" cy="16" r="3" />
    </>
  ),
  tent: (
    <>
      <path d="M19 20L12 4 5 20" />
      <path d="M12 15L9 20h6z" />
    </>
  ),
  masks: (
    <>
      <path d="M4 10c0-4 4-8 8-8s8 4 8 8c0 5-4 10-8 10-4 0-8-5-8-10z" />
      <path d="M9 9h.01M15 9h.01M12 14c-1 0-2 .5-2 1h4c0-.5-1-1-2-1z" />
    </>
  ),
  frame: (
    <>
      <rect x="3" y="3" width="18" height="18" rx="2" ry="2" />
      <path d="M3 9h18M9 21V9" />
    </>
  ),
};

export type IconName = keyof typeof PATHS;

export default function Icon({ name, size = 18 }: { name: IconName; size?: number }) {
  return (
    <svg className="icon" style={{ width: size, height: size }} viewBox="0 0 24 24" aria-hidden="true">
      {PATHS[name]}
    </svg>
  );
}
```

- [ ] **Step 4: `frontend/src/components/SkeletonCard.tsx`**

```tsx
export default function SkeletonCard() {
  return (
    <div className="rounded-3xl bg-[var(--surface-2)] border border-[var(--panel-border-dim)] p-6 animate-pulse space-y-4">
      <div className="h-32 bg-[var(--skel)] rounded-2xl -m-6 mb-4" />
      <div className="h-5 bg-[var(--skel)] rounded w-3/4" />
      <div className="h-4 bg-[var(--skel)] rounded w-1/2" />
      <div className="h-4 bg-[var(--skel)] rounded w-2/3" />
      <div className="border-t border-[var(--panel-border-dim)] pt-4 mt-4">
        <div className="h-4 bg-[var(--skel)] rounded w-1/3" />
      </div>
    </div>
  );
}
```

- [ ] **Step 5: `frontend/src/components/EmptyState.tsx`（idle 與空結果共用）**

`idle` 不可以是一片空白，否則使用者一進站看不出要按搜尋鈕，也分不出
「還沒搜」與「查無結果」（spec §4）。兩種狀態共用同一個容器，只換文案。

```tsx
export default function EmptyState({
  title, hint, testId,
}: { title: string; hint: string; testId: string }) {
  return (
    <div className="state-panel" role="status" data-testid={testId}>
      <svg className="mx-auto mb-5" style={{ width: 80, height: 80 }} viewBox="0 0 100 100"
        fill="none" stroke="var(--text-faint)" strokeWidth="1.5" aria-hidden="true">
        <circle cx="44" cy="44" r="26" />
        <path d="M63 63 L84 84" strokeLinecap="round" />
        <path d="M44 10 V2 M44 86 v-8" strokeDasharray="2 4" />
        <path d="M8 44 H16 M72 44 h8" strokeDasharray="2 4" />
      </svg>
      <p className="text-[var(--text)] font-bold text-lg mb-2">{title}</p>
      <p className="text-[var(--text-muted)] text-sm max-w-sm mx-auto">{hint}</p>
    </div>
  );
}
```

- [ ] **Step 6: `frontend/src/components/ErrorMessage.tsx`**

```tsx
import { useT } from "../i18n";
import Icon from "./Icon";

export default function ErrorMessage({ message, onRetry }: { message: string; onRetry: () => void }) {
  const t = useT();
  return (
    <div className="state-panel" role="alert" data-testid="state-error">
      <svg className="mx-auto mb-5" style={{ width: 80, height: 80 }} viewBox="0 0 100 100"
        fill="none" stroke="var(--accent)" strokeWidth="1.5" aria-hidden="true">
        <path d="M50 12 L92 84 L8 84 Z" strokeLinejoin="round" />
        <path d="M50 38 V60" strokeLinecap="round" />
        <circle cx="50" cy="72" r="2" fill="var(--accent)" />
      </svg>
      <p className="text-[var(--text)] font-bold text-lg mb-2">{t("error.title")}</p>
      <p className="text-[var(--text-muted)] text-sm mb-6">{message}</p>
      <button
        type="button"
        onClick={onRetry}
        className="btn-secondary rounded-full px-6 py-2.5 text-sm font-bold inline-flex items-center gap-2 transition hover:scale-105"
      >
        <Icon name="refresh" />
        {t("error.retry")}
      </button>
    </div>
  );
}
```

- [ ] **★ Step 7: 10a 的 checkpoint：三種狀態的畫面**

把 `frontend/src/App.tsx` 暫時整份取代為：

```tsx
import EmptyState from "./components/EmptyState";
import ErrorMessage from "./components/ErrorMessage";
import SkeletonCard from "./components/SkeletonCard";

export default function App() {
  return (
    <div className="min-h-screen p-8 space-y-8">
      <div className="glass rounded-[40px] p-8">
        <h2 className="mb-4 font-bold">loading</h2>
        <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
          <SkeletonCard /><SkeletonCard /><SkeletonCard />
        </div>
      </div>
      <div className="glass rounded-[40px] p-8">
        <h2 className="mb-4 font-bold">idle</h2>
        <EmptyState title="選好條件後按搜尋" hint="挑一個地區、類別與月份" />
      </div>
      <div className="glass rounded-[40px] p-8">
        <h2 className="mb-4 font-bold">error</h2>
        <ErrorMessage message="資料來源暫時無法使用" onRetry={() => alert("retry")} />
      </div>
    </div>
  );
}
```

```bash
make dev
```

開 `http://127.0.0.1:5173`，確認五項：

1. 三個玻璃面板都是**真的透亮**（看得到背後的背景），不是灰底
2. skeleton 有動畫，且含分隔線與票價列的骨架
3. idle 與 error 各有一個虛線邊框容器與線條插圖，兩者長得不一樣
4. 按 Tab 走一遍，**每個可聚焦元素都看得到 focus 環**（`:focus-visible` 有生效）
5. 切 dark / light 兩個主題各看一次，兩邊都要是透亮玻璃不是換色

第 4 項與第 5 項是最容易在後面才發現、而且發現時已經散落在九個元件裡的兩項。

`Ctrl+C` 結束。這段 App.tsx 會被 10b 的 checkpoint 覆蓋，不是技術債。

**10a 的 local 驗收：** 上面五項全過。

---

### Task 10b: 活動卡片與列表 ★ checkpoint

做完這一段你會看到：用假資料排出來的卡片 grid，含時間區間與票價的實際排版。

**10b 這一段只碰**：`EventCard.tsx`、`EventList.tsx`

- [ ] **Step 1: `frontend/src/components/EventCard.tsx`**

```tsx
import { useT } from "../i18n";
import type { EventItem } from "../types";
import { formatEventRange, formatPrice } from "../utils/format";
import Icon from "./Icon";

// v27 的三組 banner：漸層底 + 白色線條幾何。依卡片序輪流，讓 grid 有節奏又不需要圖片資源
const BANNERS = [
  {
    gradient: "linear-gradient(135deg,#c2410c,#d97706)",
    art: (
      <>
        <circle cx="248" cy="20" r="38" />
        <circle cx="248" cy="20" r="24" strokeDasharray="3 5" />
        <path d="M30 92 L58 44 L86 92 Z" />
        <path d="M140 20 V44 M128 32 H152" strokeWidth="1.6" />
      </>
    ),
  },
  {
    gradient: "linear-gradient(135deg,#1e3a8a,#3b82f6)",
    art: (
      <>
        <rect x="220" y="18" width="52" height="52" transform="rotate(16 246 44)" />
        <path d="M20 30 A46 46 0 0 1 66 76" strokeDasharray="3 5" />
        <path d="M120 84 C150 40 200 96 244 60" strokeDasharray="1 7" strokeLinecap="round" />
      </>
    ),
  },
  {
    gradient: "linear-gradient(135deg,#0d9488,#115e59)",
    art: (
      <>
        <path d="M252 14 C255 34 262 41 282 44 C262 47 255 54 252 74 C249 54 242 47 222 44 C242 41 249 34 252 14 Z" />
        <circle cx="52" cy="76" r="30" />
        <circle cx="52" cy="76" r="18" strokeDasharray="3 5" />
        <path d="M140 26 L166 70 L114 70 Z" />
      </>
    ),
  },
];

export default function EventCard({ event, index }: { event: EventItem; index: number }) {
  const t = useT();
  const banner = BANNERS[index % BANNERS.length];
  return (
    <article className="card-hover group rounded-3xl overflow-hidden bg-[var(--surface-2)] border border-[var(--panel-border-dim)] transition-all duration-300">
      <div className="h-32 relative flex items-end p-4 overflow-hidden" style={{ background: banner.gradient }}>
        <svg className="card-img-svg absolute inset-0 w-full h-full" viewBox="0 0 300 128" fill="none"
          stroke="rgba(255,255,255,.2)" strokeWidth="1.2" preserveAspectRatio="xMidYMid slice" aria-hidden="true">
          {banner.art}
        </svg>
        {event.onSales === "Y" && (
          <span className="relative text-xs font-semibold bg-black/50 backdrop-blur-md text-white px-3 py-1.5 rounded-full shadow-sm">
            🔥 {t("event.onSales")}
          </span>
        )}
      </div>
      <div className="p-6">
        <h3 className="font-bold text-lg leading-snug mb-3 text-[var(--text)] group-hover:text-[var(--link)] transition-colors">
          <a href={event.googleSearchUrl} target="_blank" rel="noopener noreferrer">
            {event.title}
          </a>
        </h3>
        <p className="text-sm text-[var(--text-muted)] mb-2 flex items-center gap-2">
          <span className="opacity-70"><Icon name="calendar" size={16} /></span>
          {/* 區間不是只有開始時間。展覽 88% 跨月，只顯示 startTime 會讓
              查九月的人看到每張卡都寫 01/01，以為資料是舊的（spec §4）。 */}
          {formatEventRange(event.startTime, event.endTime)}
        </p>
        <a href={event.googleMapUrl} target="_blank" rel="noopener noreferrer"
          className="text-sm hover:underline flex items-start gap-2 mb-5" style={{ color: "var(--link)" }}>
          <span className="mt-0.5 opacity-70"><Icon name="pin" size={16} /></span>
          <span className="leading-relaxed">{event.locationName ?? event.location}</span>
        </a>
        <div className="flex items-center justify-between border-t border-[var(--panel-border-dim)] pt-4 mt-2">
          {/* 票價是自由文字。直接 `$ {price}` 會產出「$ 洽詢主辦單位」與「$ 0」。 */}
          {formatPrice(event.price, t("event.free")) && (
            <p className="text-sm font-bold text-[var(--text)]">
              {formatPrice(event.price, t("event.free"))}
            </p>
          )}
          {/* 文案講清楚去向是 Google 搜尋結果。寫「詳細資訊」的話使用者會期待完整
              活動說明，點開發現是搜尋結果頁會覺得被丟包（spec §4） */}
          <a href={event.googleSearchUrl} target="_blank" rel="noopener noreferrer"
            className="text-xs font-bold px-3 py-1.5 rounded-full bg-[var(--surface-2)] text-[var(--text)] hover:bg-white/10 transition inline-flex items-center gap-1.5">
            {t("event.search")}
            <span aria-hidden="true">↗</span>
          </a>
        </div>
      </div>
    </article>
  );
}
```

- [ ] **Step 2: `frontend/src/components/EventList.tsx`**

```tsx
import type { EventItem } from "../types";
import EventCard from "./EventCard";

export default function EventList({ events, summary }: { events: EventItem[]; summary: string }) {
  return (
    // role="status" 讓螢幕閱讀器在搜尋完成時播報結果，否則按下搜尋後沒有任何回饋，
    // 使用者會以為按鈕沒作用（spec §4.3）
    <div role="status" data-testid="state-success">
      <p className="mb-3 text-sm text-[var(--text-muted)]">{summary}</p>
      <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3" data-testid="results-grid">
        {events.map((event, i) => (
          <EventCard key={`${event.title}-${event.startTime}-${i}`} event={event} index={i} />
        ))}
      </div>
    </div>
  );
}
```

- [ ] **★ Step 3: 10b 的 checkpoint：卡片 grid**

把 `frontend/src/App.tsx` 暫時整份取代為（三筆假資料刻意涵蓋三種時間與票價形態）：

```tsx
import EventList from "./components/EventList";
import type { EventItem } from "./types";

const FIXTURES: EventItem[] = [
  { title: "同月多天的表演", startTime: "2026-07-12T19:30:00", endTime: "2026-07-14T21:30:00",
    location: "臺北市中正區中山南路21-1號", locationName: "國家音樂廳", onSales: "Y",
    price: "500", googleMapUrl: "https://example.com", googleSearchUrl: "https://example.com" },
  { title: "跨月的常設展", startTime: "2026-01-01T09:00:00", endTime: "2026-12-31T18:00:00",
    location: "臺北市士林區", locationName: "故宮", onSales: null,
    price: "0", googleMapUrl: "https://example.com", googleSearchUrl: "https://example.com" },
  { title: "沒有結束時間也沒有票價的活動", startTime: "2026-07-20T14:00:00", endTime: null,
    location: "台北市信義區", locationName: null, onSales: null,
    price: "洽詢主辦單位", googleMapUrl: "https://example.com", googleSearchUrl: "https://example.com" },
];

export default function App() {
  return (
    <div className="min-h-screen p-8">
      <div className="glass rounded-[40px] p-8">
        <EventList events={FIXTURES} summary="臺北 · 展覽 · 2026/07：3 筆活動" />
      </div>
    </div>
  );
}
```

```bash
make dev
```

開 `http://127.0.0.1:5173`，確認五項：

1. **第一張卡的時間是區間**（`07/12 19:30 – 07/14`），不是只有開始時間
2. **第二張卡顯示 `2026/01/01 – 2026/12/31`**，跨月時有帶年份。
   這是 88% 的展覽資料長的樣子，只顯示 `01/01` 的話使用者會以為資料是舊的
3. **第三張卡只顯示開始時間**（`endTime` 為 `null` 不可以印出 `– null`）
4. **票價三種形態各自正確**：`$ 500`、`免費`（不是 `$ 0`）、`洽詢主辦單位`（不是 `$ 洽詢主辦單位`）
5. 手機 375px 單欄、桌機 1440px 三欄；hover 卡片時 banner 有慢速 zoom

第 1 到 4 項是 v5 補的修正，特別確認。

`Ctrl+C` 結束。

**10b 的 local 驗收：** 上面五項全過。

---

### Task 10c: 搜尋卡 ★ checkpoint

做完這一段你會看到完整的搜尋介面，而且四種狀態一次全部到齊。

**10c 這一段只碰**：`SearchForm.tsx`

- [ ] **Step 1: `frontend/src/components/SearchForm.tsx`**

三個關鍵行為，寫錯任何一個都會讓使用者以為壞掉：

1. **類別 chip 點下去直接觸發搜尋**，不是只改 state。chip 長得像 filter tab
   （點了會立刻換內容），只改 state 的話 chip 變 active 但下方還是舊卡片。
2. **chip 觸發搜尋時要把新值一起傳出去**（`onSubmit(next)`）。React 的 state
   更新是非同步的，`onChange(next)` 之後立刻 `onSubmit()` 會用到舊的 `form`。
3. **快捷 chip 只有四個**，不是把全部 12 個類別都渲染出來（12 個 chip 在手機上
   會換三行、佔滿第一屏）。完整清單走 `<select>`。

```tsx
import { pickLabel, useLang, useT } from "../i18n";
import type { Country, SearchValue } from "../types";
import { resetForCountry, withMonth, withYear } from "../utils/search";
import Icon, { type IconName } from "./Icon";

interface Props {
  countries: Country[];
  value: SearchValue;
  onChange: (value: SearchValue) => void;
  onSubmit: (next?: SearchValue) => void;
  loading: boolean;
  disabled: boolean;
}

// spec §4：未上線國家後端沒有資料，chip 內容 hardcode 在前端。
// ⚠️ 新國家上線時必須從這裡移除，否則前端會同時顯示一個 active chip
// 和一個未開放 chip（spec §10 的「加國家不是三步」）。
const COMING_SOON = [
  { code: "JP", label: { zh: "日本", en: "Japan" } },
  { code: "KR", label: { zh: "韓國", en: "Korea" } },
];

// 四個快捷 chip，數量對齊 POC v27。key 是後端 provider 定義的 category id（T5 的 taiwan.py）。
// POC 示意的第四個是「市集」，但 MoC 沒有這個 category，改用「親子」配同一個 tent icon。
const QUICK_CATEGORIES: { id: number; icon: IconName }[] = [
  { id: 6, icon: "frame" },   // 展覽
  { id: 2, icon: "masks" },   // 戲劇
  { id: 1, icon: "music" },   // 音樂
  { id: 4, icon: "tent" },    // 親子
];

const GRADIENT = "linear-gradient(135deg, var(--accent-cool), var(--accent))";

// 月份選擇器用年 + 月兩個 <select>，不用 <input type="month">：
// 桌面版 Firefox / Safari 全版本不支援 month picker，會 fallback 成純文字框。
const NOW = new Date();
const YEARS = [NOW.getFullYear(), NOW.getFullYear() + 1].map(String);
const MONTHS = Array.from({ length: 12 }, (_, i) => String(i + 1).padStart(2, "0"));

export default function SearchForm({ countries, value, onChange, onSubmit, loading, disabled }: Props) {
  const t = useT();
  const { lang } = useLang();
  const country = countries.find((c) => c.code === value.country);

  function pickCategory(id: number) {
    const next = { ...value, category: String(id) };
    onChange(next);
    // 把新值一起傳出去：setState 是非同步的，只呼叫 onSubmit() 會用到舊的 form
    onSubmit(next);
  }

  const quickChips = QUICK_CATEGORIES.filter((q) =>
    // 兩邊都套 String()。LabeledOption.value 的型別是 string | number，
    // 而 QUICK_CATEGORIES 硬寫的是 number。後端哪天把 value 序列化成字串，
    // 嚴格比對會讓整排 chip 靜默消失，沒有 error 也沒有 console warning（spec §4.3）。
    country?.categories.some((c) => String(c.value) === String(q.id)),
  );

  return (
    <div>
      {/* Country selector：top-right pill group, per v27 layout */}
      <div className="flex items-center gap-2 self-start md:self-auto bg-[var(--surface-2)] p-1.5 rounded-full border border-[var(--panel-border-dim)] shadow-sm w-fit mb-6 md:mb-8 ml-auto">
        {countries.map((c) => (
          <button
            key={c.code}
            type="button"
            aria-pressed={c.code === value.country}
            onClick={() => onChange(resetForCountry(value, c))}
            className={
              c.code === value.country
                ? "btn-primary shrink-0 flex items-center gap-2 px-4 py-1.5 text-sm shadow"
                : "shrink-0 flex items-center gap-2 px-3 py-1.5 rounded-full text-sm text-[var(--text-muted)] hover:bg-[var(--surface-2)]"
            }
          >
            <span className="h-5 w-5 rounded-full bg-white/20 flex items-center justify-center text-[10px] font-bold">
              {c.code.toUpperCase()}
            </span>
            {pickLabel(c.name, lang)}
          </button>
        ))}
        {COMING_SOON.map((c) => (
          // aria-disabled 而不是 disabled：disabled 的 button 不可 focus，
          // 鍵盤與螢幕閱讀器使用者連它存在都不會知道（spec §4）
          <button
            key={c.code}
            type="button"
            aria-disabled="true"
            title={`${pickLabel(c.label, lang)}（${t("search.comingSoon")}）`}
            aria-label={`${pickLabel(c.label, lang)}（${t("search.comingSoon")}）`}
            onClick={(e) => e.preventDefault()}
            className="chip shrink-0 flex items-center gap-2 px-3 py-1.5 rounded-full text-sm text-[var(--text-muted)]"
          >
            <span className="h-5 w-5 rounded-full bg-[var(--panel-border-dim)] flex items-center justify-center text-[10px] font-bold">
              {c.code}
            </span>
            {pickLabel(c.label, lang)}
          </button>
        ))}
      </div>

      <form
        onSubmit={(e) => { e.preventDefault(); onSubmit(); }}
        className="search-capsule mb-8 flex-col sm:flex-row rounded-3xl sm:rounded-full"
      >
        <div className="search-field">
          <label className="search-label" htmlFor="sf-location">{t("search.location")}</label>
          <select
            id="sf-location"
            className="search-select"
            value={value.location}
            onChange={(e) => onChange({ ...value, location: e.target.value })}
          >
            {country?.locations.map((o) => (
              <option className="text-black" key={String(o.value)} value={String(o.value)}>
                {pickLabel(o.label, lang)}
              </option>
            ))}
          </select>
        </div>
        <div className="search-field">
          <label className="search-label" htmlFor="sf-category">{t("search.category")}</label>
          <select
            id="sf-category"
            className="search-select"
            value={value.category}
            onChange={(e) => onChange({ ...value, category: e.target.value })}
          >
            {country?.categories.map((o) => (
              <option className="text-black" key={String(o.value)} value={String(o.value)}>
                {pickLabel(o.label, lang)}
              </option>
            ))}
          </select>
        </div>
        <div className="search-field">
          <label className="search-label" htmlFor="sf-year">{t("search.year")}</label>
          <select
            id="sf-year"
            className="search-select"
            value={value.month.slice(0, 4)}
            onChange={(e) => onChange({ ...value, month: withYear(value.month, e.target.value) })}
          >
            {YEARS.map((y) => (
              <option className="text-black" key={y} value={y}>{y}</option>
            ))}
          </select>
        </div>
        <div className="search-field">
          <label className="search-label" htmlFor="sf-month">{t("search.month")}</label>
          <select
            id="sf-month"
            className="search-select"
            value={value.month.slice(5, 7)}
            onChange={(e) => onChange({ ...value, month: withMonth(value.month, e.target.value) })}
          >
            {MONTHS.map((m) => (
              <option className="text-black" key={m} value={m}>{m}</option>
            ))}
          </select>
        </div>
        <div className="p-3 sm:p-2 flex items-center justify-center">
          <button
            type="submit"
            disabled={loading || disabled}
            data-testid="search-submit"
            className="w-full sm:w-12 h-12 rounded-2xl sm:rounded-full flex items-center justify-center text-sm shadow-lg gap-2 text-white hover:scale-105 transition-transform disabled:opacity-50"
            style={{ background: GRADIENT }}
          >
            <Icon name="search" />
            <span className="sm:hidden font-semibold">{t("search.submit")}</span>
          </button>
        </div>
      </form>

      {/* 四個快捷 chip。點下去直接搜尋： 快捷就要一步到位，只改 state 不重搜
          會讓使用者以為篩選壞了（spec §4）。完整 12 個類別走上面的 <select>。 */}
      <div className="flex flex-wrap items-center gap-2.5 mb-8" data-testid="category-chips">
        {/* 重設鈕。空結果的文案叫使用者「重設條件」，那個東西就必須存在——
            v4 的文案寫了「清除篩選條件」而畫面上根本沒有這顆鈕。空結果是冷門搜尋
            最常見的結果，所以那是全站最多人會讀到的一段字（spec §4）。
            手機上手動改回四個 select 要點八下。 */}
        <button
          type="button"
          disabled={loading || disabled}
          onClick={() => onChange(resetForCountry(value, country!))}
          className="chip btn-secondary flex items-center gap-1.5 px-4 py-2 rounded-full text-sm font-medium disabled:opacity-50"
        >
          <Icon name="refresh" size={16} />
          {t("search.reset")}
        </button>
        {quickChips.map((q) => {
          const option = country!.categories.find((c) => String(c.value) === String(q.id))!;
          const val = String(q.id);
          const active = value.category === val;
          return (
            <button
              key={val}
              type="button"
              aria-pressed={active}
              disabled={loading || disabled}
              onClick={() => pickCategory(q.id)}
              className={`chip ${active ? "active" : "btn-secondary"} flex items-center gap-1.5 px-4 py-2 rounded-full text-sm font-medium disabled:opacity-50`}
            >
              <Icon name={q.icon} size={16} />
              {pickLabel(option.label, lang)}
            </button>
          );
        })}
      </div>
    </div>
  );
}
```

- [ ] **★ Step 2: 10c 的 checkpoint：用死資料把四種畫面都看過一次**

四個 task 的視覺風險不可以全部堆到 T11 一次結算。這裡用一個丟棄式的 fixture 頁
把 `SearchForm` / `SkeletonCard` / `EventList` / `EmptyState` / `ErrorMessage`
都渲染出來看一眼。這段 code 在 T11 Step 2 本來就會被整份覆蓋，不是技術債。

把 `frontend/src/App.tsx` 暫時整份取代為：

```tsx
import EmptyState from "./components/EmptyState";
import ErrorMessage from "./components/ErrorMessage";
import EventList from "./components/EventList";
import SkeletonCard from "./components/SkeletonCard";
import SearchForm from "./components/SearchForm";
import { useT } from "./i18n";
import type { Country, EventItem, SearchValue } from "./types";
import { useState } from "react";

const FIXTURE_COUNTRY: Country = {
  code: "tw",
  name: { zh: "台灣", en: "Taiwan" },
  locations: [{ value: "臺北", label: { zh: "臺北", en: "Taipei" } }],
  categories: [
    { value: 1, label: { zh: "音樂", en: "Music" } },
    { value: 2, label: { zh: "戲劇", en: "Theater" } },
    { value: 4, label: { zh: "親子", en: "Family" } },
    { value: 6, label: { zh: "展覽", en: "Exhibition" } },
  ],
};

const FIXTURE_EVENTS: EventItem[] = [0, 1, 2].map((i) => ({
  title: `示範活動 ${i + 1}`,
  startTime: `2026-07-1${i}T19:30:00`,
  endTime: null,
  location: "臺北市中正區中山南路21-1號",
  locationName: "國家音樂廳",
  onSales: i === 0 ? "Y" : null,
  price: "500",
  googleMapUrl: "https://www.google.com/maps",
  googleSearchUrl: "https://www.google.com/search?q=demo",
}));

export default function App() {
  const t = useT();
  const [form, setForm] = useState<SearchValue>({
    country: "tw", category: "6", location: "臺北", month: "2026-07",
  });
  return (
    <>
      <div className="scene"><div className="wall" /><div className="lamp-light" /></div>
      <div className="max-w-7xl mx-auto px-4 py-6 space-y-8">
        <main className="glass rounded-[40px] p-6 sm:p-10">
          <SearchForm countries={[FIXTURE_COUNTRY]} value={form} onChange={setForm}
            onSubmit={() => {}} loading={false} disabled={false} />
          <EmptyState title={t("results.idleTitle")} hint={t("results.idleHint")} testId="state-idle" />
          <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3 mt-4" aria-busy="true">
            <SkeletonCard /><SkeletonCard /><SkeletonCard />
          </div>
          <div className="mt-4">
            <EventList events={FIXTURE_EVENTS} summary="臺北 · 展覽 · 2026/07 共 3 筆活動" />
          </div>
          <EmptyState title={t("results.emptyTitle")} hint={t("results.emptyHint")} testId="state-empty" />
          <ErrorMessage message={t("error.upstream")} onRetry={() => {}} />
        </main>
      </div>
    </>
  );
}
```

```bash
cd frontend && npx tsc --noEmit
make dev
```

開 `http://127.0.0.1:5173`，逐項確認：

1. 搜尋膠囊四個欄位都有 chevron 箭頭，看得出可以點
2. 用 Tab 鍵走過四個 select 與搜尋鈕，每一個都看得到 accent 色的 focus 框
3. 四個帶 icon 的快捷 chip（展覽/戲劇/音樂/親子），不是 12 個
4. 日本/韓國 chip 有降權，滑鼠停留會出現「即將推出」tooltip
5. skeleton、卡片、空結果、錯誤四種畫面都出得來，虛線邊框容器樣式一致
6. 用 devtools 手動把 `<html data-theme>` 改成 `light` 再改回 `dark`，
   兩個主題的玻璃都要是透亮的，不是只有換色
7. 併排開 `docs/poc/20260719_155200_ui_design_v27.html` 比對配色與間距。
   POC 是 Tailwind v3、這裡是 v4，陰影（`shadow-sm` → `shadow-xs`）與
   邊框預設色（v4 改成 `currentColor`）會有細微差異，看到就是這個原因（T4 Step 2）

8. **「重設條件」鈕在 chip 那一列的最前面**，按下去四個欄位回到預設值。
   空結果的文案叫使用者做這件事，那顆鈕就必須存在（spec §4）

第 1、2、8 項是 v5 補的修正，特別確認。

- [ ] **Step 2b: 補一條 SearchForm 的純函式測試**

`resetForCountry` 現在有兩個呼叫點（切國家、重設鈕），值得一條測試釘住它：

```bash
cd frontend && npm test
```

Expected: T8 寫的那組測試仍然全綠（`resetForCountry` 的行為沒變，只是多了一個呼叫點）。

- [ ] **Step 3: Commit**

```bash
cd frontend && npx tsc --noEmit
git add frontend/src && git commit -m "feat: add search form with reset, quick chips, focus styles"
```

**10c 的 local 驗收：** 瀏覽器上四種狀態畫面都看得到，Tab 鍵看得到 focus 框，重設鈕會動。

**T10 三段結束狀態：** 九個元件與 design system 全部到位，四種狀態都親眼看過。

---

### Task 11: App 組裝 + About ★ checkpoint

**Files:**
- Modify: `frontend/src/App.tsx`（整份取代）、`frontend/src/main.tsx`、`frontend/index.html`
- Delete: `frontend/src/App.css`、`frontend/src/assets/react.svg`（scaffold 殘留物）

**Interfaces:**
- Consumes: T8、T9、T10a、T10b、T10c 的所有產出。
- Produces: 完整 SPA：背景場景、桌機 icon rail / 手機頂部 bar（含語言切換）、
  dark/light 主題切換與持久化、搜尋 + about 兩個畫面、idle/loading/empty/success/error 五種狀態。

- [ ] **Step 1: `frontend/index.html` 加防 FOUC 的 inline script**

在 `<head>` 的最後、`</head>` 之前加：

```html
<script>
  // 在 React 掛載之前就把 data-theme 設好。用 useEffect 設的話會先 render 一次
  // 舊主題再閃一下，這是避免 FOUC 唯一可靠的做法（spec §4.3）。
  (function () {
    try {
      var saved = localStorage.getItem("theme");
      var prefersLight = window.matchMedia("(prefers-color-scheme: light)").matches;
      document.documentElement.dataset.theme = saved || (prefersLight ? "light" : "dark");
    } catch (e) {
      document.documentElement.dataset.theme = "dark";
    }
  })();
</script>
```

- [ ] **Step 2: `frontend/src/main.tsx`**

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
import { LanguageProvider } from "./i18n";
import "./index.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <LanguageProvider>
      <App />
    </LanguageProvider>
  </StrictMode>,
);
```

- [ ] **Step 3: `frontend/src/App.tsx`（整份取代，覆蓋掉 T10 的 fixture 頁）**

版面結構抄自定案 POC `docs/poc/20260719_155200_ui_design_v27.html`。

三個容易寫錯的地方，各自對應一個使用者會踩到的失敗：

1. **重試按鈕要依錯誤來源決定重試什麼。** `/countries` 失敗時 `form.country`
   是空字串，若重試接的是搜尋，會打 `/api/v1//events` 而永遠失敗，使用者
   按幾次都一樣、下拉選單全空無法自救。
2. **countries 載入完成前送出鈕要 disabled**，否則網路慢時先按下去會打出雙斜線 URL。
3. **主題要持久化**，讀 `localStorage`、寫 `localStorage`，初值由 `index.html`
   的 inline script 決定。

```tsx
import { useCallback, useEffect, useRef, useState } from "react";
import { ApiError, fetchCountries, fetchEvents } from "./api";
import EmptyState from "./components/EmptyState";
import ErrorMessage from "./components/ErrorMessage";
import EventList from "./components/EventList";
import Icon from "./components/Icon";
import LanguageSwitch from "./components/LanguageSwitch";
import SearchForm from "./components/SearchForm";
import SkeletonCard from "./components/SkeletonCard";
import { pickLabel, useLang, useT } from "./i18n";
import type { Country, EventItem, SearchValue } from "./types";
import { currentMonth } from "./utils/format";
import { resetForCountry } from "./utils/search";

type Status = "idle" | "loading" | "success" | "error";
type ErrorSource = "countries" | "search";

const TECH_STACK: { label: string; url: string; role: Record<string, string> }[] = [
  { label: "Django 5.2 LTS", url: "https://www.djangoproject.com/", role: { zh: "後端框架", en: "Backend framework" } },
  { label: "uv", url: "https://docs.astral.sh/uv/", role: { zh: "依賴管理", en: "Dependency management" } },
  { label: "toolkitsy", url: "https://pypi.org/project/toolkitsy/", role: { zh: "Logging", en: "Logging" } },
  { label: "React + TypeScript", url: "https://react.dev/", role: { zh: "前端框架", en: "Frontend framework" } },
  { label: "Vite", url: "https://vitejs.dev/", role: { zh: "前端 build tool", en: "Frontend build tool" } },
  { label: "Tailwind CSS", url: "https://tailwindcss.com/", role: { zh: "樣式框架", en: "CSS framework" } },
  { label: "pytest", url: "https://docs.pytest.org/", role: { zh: "後端測試", en: "Backend testing" } },
  { label: "Vitest", url: "https://vitest.dev/", role: { zh: "前端測試", en: "Frontend testing" } },
  { label: "Fly.io", url: "https://fly.io/", role: { zh: "部署", en: "Deployment" } },
];

function readTheme(): "dark" | "light" {
  try {
    const saved = localStorage.getItem("theme");
    if (saved === "dark" || saved === "light") return saved;
    return window.matchMedia("(prefers-color-scheme: light)").matches ? "light" : "dark";
  } catch {
    return "dark";
  }
}

export default function App() {
  const t = useT();
  const { lang } = useLang();
  const [view, setView] = useState<"search" | "about">(
    () => (location.hash === "#about" ? "about" : "search"),
  );

  // 切畫面要進 history，否則手機按返回鍵會直接離開網站（spec §4.3）。
  // 這與 spec §1 排除的 shareable URL 是兩件事：那個排除的是把「搜尋條件」寫進網址。
  const goto = useCallback((next: "search" | "about") => {
    if (next === view) return;
    history.pushState(null, "", next === "about" ? "#about" : "#");
    setView(next);
  }, [view]);

  useEffect(() => {
    const onPop = () => setView(location.hash === "#about" ? "about" : "search");
    addEventListener("popstate", onPop);
    return () => removeEventListener("popstate", onPop);
  }, []);
  const [theme, setTheme] = useState<"dark" | "light">(readTheme);
  const [countries, setCountries] = useState<Country[]>([]);
  const [form, setForm] = useState<SearchValue>({
    country: "", category: "", location: "", month: currentMonth(),
  });
  const [status, setStatus] = useState<Status>("idle");
  const [events, setEvents] = useState<EventItem[]>([]);
  const [errorCode, setErrorCode] = useState("error.generic");
  const [errorSource, setErrorSource] = useState<ErrorSource>("search");

  useEffect(() => {
    document.documentElement.dataset.theme = theme;
    try {
      localStorage.setItem("theme", theme);
    } catch {
      // private mode / storage blocked：主題仍然可以切，只是不會被記住
    }
  }, [theme]);

  const loadCountries = useCallback((signal?: AbortSignal) => {
    return fetchCountries(signal)
      .then((data) => {
        setCountries(data);
        const first = data[0];
        if (first) setForm((f) => resetForCountry(f, first));
        setStatus("idle");
      })
      .catch((err) => {
        if (err instanceof DOMException && err.name === "AbortError") return;
        setErrorSource("countries");
        setErrorCode("error.generic");
        setStatus("error");
      });
  }, []);

  useEffect(() => {
    const ac = new AbortController();
    loadCountries(ac.signal);
    return () => ac.abort();
  }, [loadCountries]);

  // 每次搜尋開一個 AbortController，新的搜尋先 abort 舊的。
  //
  // 不做的話：點展覽（cache miss，最多 15 秒）再點音樂（cache hit，200 毫秒），
  // 音樂先渲染、展覽後到把畫面換掉，而音樂的 chip 還亮著。類別 chip 是直接觸發
  // 搜尋的，所以這是一根手指就會走到的路徑（spec §4.3）。
  // loadCountries 已經是這個寫法，這裡照抄。
  const searchAbort = useRef<AbortController | null>(null);

  const handleSearch = useCallback(async (next?: SearchValue) => {
    const query = next ?? form;
    if (!query.country) return;  // countries 還沒載完，不要打出 /api/v1//events

    searchAbort.current?.abort();
    const ac = new AbortController();
    searchAbort.current = ac;

    setStatus("loading");
    try {
      const result = await fetchEvents(query.country, {
        category: query.category,
        location: query.location,
        month: query.month,
      }, ac.signal);
      setEvents(result.events);
      setStatus("success");
    } catch (err) {
      // 被自己 abort 掉的請求不是錯誤，什麼都不要寫。
      // 少了這個判斷，舊請求的 abort 會把新請求的 loading 畫面蓋成錯誤畫面。
      if (err instanceof DOMException && err.name === "AbortError") return;
      setErrorSource("search");
      // 存 error code 而不是翻譯過的字串：存字串的話之後切成 EN，
      // 畫面上那句錯誤訊息仍會是中文
      setErrorCode(err instanceof ApiError && err.status === 502 ? "error.upstream" : "error.generic");
      setStatus("error");
    }
  }, [form]);

  // loading 超過 8 秒換一段文案。冷機器加上上游 15 秒 timeout，最壞是 20 秒的
  // 骨架動畫配一片安靜，手機使用者會重整，重整又從頭來（spec §4）。
  const [slow, setSlow] = useState(false);
  useEffect(() => {
    if (status !== "loading") {
      setSlow(false);
      return;
    }
    const timer = setTimeout(() => setSlow(true), 8000);
    return () => clearTimeout(timer);
  }, [status]);

  const onRetry = useCallback(() => {
    if (errorSource === "countries") {
      setStatus("idle");
      loadCountries();
    } else {
      handleSearch();
    }
  }, [errorSource, loadCountries, handleSearch]);

  const railBtn = (active: boolean, small = false) =>
    `rail-btn ${active ? "active" : ""} ${small ? "h-9 w-9" : "h-11 w-11"} rounded-full flex items-center justify-center transition hover:bg-[var(--surface-2)]`;

  const country = countries.find((c) => c.code === form.country);
  const locationLabel = country?.locations.find((o) => String(o.value) === form.location);
  const categoryLabel = country?.categories.find((o) => String(o.value) === form.category);
  // 條件顯性寫出來：預設類別是 provider 的第一個類別而不是「全部」，
  // 不寫的話使用者會誤以為看到的是全類別（spec §4）
  const summary = [
    locationLabel ? pickLabel(locationLabel.label, lang) : form.location,
    categoryLabel ? pickLabel(categoryLabel.label, lang) : form.category,
    form.month.replace("-", "/"),
  ].join(" · ") + `：${events.length} ${t("results.count")}`;

  return (
    <>
      {/* 背景場景（spec §4）：深色 radial-gradient + 抽象曲線 SVG + 雙色燈光 +
          兩顆跨面板光暈 (bronze/cyan)，是毛玻璃 blur 的視覺素材 */}
      <div className="scene">
        <div className="wall" />
        <svg className="absolute inset-0 w-full h-full opacity-[0.22] pointer-events-none" viewBox="0 0 1440 900"
          fill="none" preserveAspectRatio="xMidYMid slice" aria-hidden="true">
          <path d="M-100 850 C300 680 600 800 1000 480 C1300 200 1500 380 1600 -80" stroke="url(#bg-grad-1)" strokeWidth="4.5" strokeLinecap="round" />
          <path d="M150 950 C450 580 750 850 1150 380" stroke="url(#bg-grad-2)" strokeWidth="2.5" strokeDasharray="10 10" />
          <circle cx="85%" cy="15%" r="280" stroke="url(#bg-grad-3)" strokeWidth="1.8" />
          <circle cx="85%" cy="15%" r="180" stroke="url(#bg-grad-3)" strokeWidth="1.2" />
          <circle cx="20%" cy="80%" r="350" stroke="url(#bg-grad-1)" strokeWidth="1.8" />
          <circle cx="20%" cy="80%" r="220" stroke="url(#bg-grad-1)" strokeWidth="1.2" strokeDasharray="6 6" />
          <path d="M500 -50 C700 200 600 400 900 600" stroke="url(#bg-grad-2)" strokeWidth="1.5" />
          <defs>
            <linearGradient id="bg-grad-1" x1="0" y1="0" x2="1" y2="1">
              <stop offset="0%" stopColor="#3b82f6" stopOpacity="0.8" />
              <stop offset="100%" stopColor="#B57004" stopOpacity="0.1" />
            </linearGradient>
            <linearGradient id="bg-grad-2" x1="0" y1="0" x2="1" y2="1">
              <stop offset="0%" stopColor="#B57004" stopOpacity="0.6" />
              <stop offset="100%" stopColor="#3b82f6" stopOpacity="0.1" />
            </linearGradient>
            <linearGradient id="bg-grad-3" x1="0" y1="0" x2="1" y2="1">
              <stop offset="0%" stopColor="#B57004" stopOpacity="0.4" />
              <stop offset="100%" stopColor="#06b6d4" stopOpacity="0.1" />
            </linearGradient>
          </defs>
        </svg>
        <div className="lamp-light" />
        <div className="blob blob-bronze" />
        <div className="blob blob-cyan" />
        <div className="floor-shadow" />
      </div>

      <div className="max-w-7xl mx-auto px-4 py-6 flex gap-4">
        {/* 桌機左側 icon rail */}
        <aside className="glass hidden sm:flex flex-col items-center gap-3 rounded-full px-2.5 py-5 h-fit sticky top-6">
          <button type="button" aria-label={t("nav.search")} aria-pressed={view === "search"}
            onClick={() => goto("search")} className={railBtn(view === "search")}>
            <Icon name="search" />
          </button>
          <button type="button" aria-label={t("nav.about")} aria-pressed={view === "about"}
            onClick={() => goto("about")} className={railBtn(view === "about")}>
            <Icon name="info" />
          </button>
          <div className="w-6 h-px bg-[var(--panel-border-dim)] my-2" />
          <button type="button" aria-label={t("nav.theme")}
            onClick={() => setTheme(theme === "dark" ? "light" : "dark")} className={railBtn(false)}>
            <Icon name={theme === "dark" ? "moon" : "sun"} />
          </button>
          <LanguageSwitch />
        </aside>

        <div className="flex-1 min-w-0">
          {/* 手機頂部 bar（rail 在 sm 以下隱藏）。語言切換必須放在這裡，
              不然整個 i18n 在手機上等於不存在（spec §4） */}
          <div className="glass sm:hidden flex items-center justify-between rounded-3xl px-4 py-4 mb-5">
            <h1 className="font-bold flex items-center gap-2 text-base text-[var(--text)] tracking-wide flex-1 min-w-0 truncate">
              <Icon name="ticket" />
              {t("app.title")}
            </h1>
            <div className="flex gap-1.5 shrink-0">
              <button type="button" aria-label={t("nav.search")} aria-pressed={view === "search"}
                onClick={() => goto("search")} className={railBtn(view === "search", true)}>
                <Icon name="search" size={16} />
              </button>
              <button type="button" aria-label={t("nav.about")} aria-pressed={view === "about"}
                onClick={() => goto("about")} className={railBtn(view === "about", true)}>
                <Icon name="info" size={16} />
              </button>
              <button type="button" aria-label={t("nav.theme")}
                onClick={() => setTheme(theme === "dark" ? "light" : "dark")} className={railBtn(false, true)}>
                <Icon name={theme === "dark" ? "moon" : "sun"} size={16} />
              </button>
              <LanguageSwitch small />
            </div>
          </div>

          {view === "about" ? (
            <main className="glass rounded-[40px] p-6 sm:p-10 shadow-2xl">
              {/* 這個 <h1> 不可省。桌機的另一個 <h1> 關在下面的搜尋分支裡、
                  手機那個是 sm:hidden，所以桌機開 About 時整份 DOM 最高只到 <h2>，
                  靠標題階層瀏覽的螢幕閱讀器使用者會直接撞牆（spec §4.3）。 */}
              <h1 className="text-2xl font-bold mb-5 text-[var(--text)]">{t("about.title")}</h1>
              <p className="text-base text-[var(--text-muted)] leading-relaxed mb-8 max-w-2xl">{t("about.body")}</p>
              <h3 className="font-bold text-lg mb-4 text-[var(--text)]">Tech Stack</h3>
              <div className="rounded-3xl overflow-hidden mb-8 border border-[var(--panel-border-dim)] divide-y divide-[var(--panel-border-dim)] max-w-2xl shadow-sm">
                {TECH_STACK.map((item, i) => (
                  <div key={item.label} className={`flex p-4 text-sm ${i % 2 === 0 ? "bg-[var(--surface-2)]" : ""}`}>
                    <a className="w-48 shrink-0 font-bold text-[var(--text)] hover:underline"
                      target="_blank" rel="noopener noreferrer" href={item.url}>{item.label}</a>
                    <span className="text-[var(--text-muted)]">{pickLabel(item.role, lang)}</span>
                  </div>
                ))}
              </div>
              <p className="text-sm font-semibold text-[var(--text-muted)] pt-4 border-t border-[var(--panel-border-dim)]">
                作者：
                <a className="hover:underline transition" style={{ color: "var(--link)" }} target="_blank" rel="noopener noreferrer"
                  href="https://github.com/taurus5650">GitHub</a>
              </p>
            </main>
          ) : (
            <main className="glass rounded-[40px] p-6 sm:p-10 shadow-2xl">
              <div className="flex flex-col md:flex-row md:items-center justify-between mb-8 gap-4">
                <h1 className="hidden sm:flex font-bold text-2xl items-center gap-3 text-[var(--text)] tracking-tight">
                  <span className="inline-flex h-11 w-11 items-center justify-center rounded-2xl text-[var(--text)] bg-[var(--surface-2)] border border-[var(--panel-border-dim)] shadow-inner">
                    <Icon name="ticket" />
                  </span>
                  {t("app.title")}
                </h1>
              </div>
              <SearchForm countries={countries} value={form} onChange={setForm}
                onSubmit={handleSearch} loading={status === "loading"}
                disabled={countries.length === 0} />
              {status === "idle" && (
                <EmptyState title={t("results.idleTitle")} hint={t("results.idleHint")} testId="state-idle" />
              )}
              {status === "loading" && (
                <>
                  {/* 8 秒後才出現。冷機器加上上游 15 秒 timeout，最壞是 20 秒的
                      骨架動畫配一片安靜；手機使用者會以為站死了而重整，
                      重整又從頭來一次（spec §4）。role="status" 讓螢幕閱讀器也聽得到。 */}
                  {slow && (
                    <p role="status" className="text-sm text-[var(--text-muted)] mb-4 text-center">
                      {t("search.slow")}
                    </p>
                  )}
                  <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3" aria-busy="true" data-testid="state-loading">
                    <SkeletonCard /><SkeletonCard /><SkeletonCard />
                  </div>
                </>
              )}
              {status === "error" && <ErrorMessage message={t(errorCode)} onRetry={onRetry} />}
              {status === "success" && events.length === 0 && (
                <EmptyState title={t("results.emptyTitle")} hint={t("results.emptyHint")} testId="state-empty" />
              )}
              {status === "success" && events.length > 0 && (
                <EventList events={events} summary={summary} />
              )}
            </main>
          )}
        </div>
      </div>
    </>
  );
}
```

- [ ] **Step 4: 清掉 scaffold 殘留物，跑檢查**

```bash
cd frontend && rm -f src/App.css src/assets/react.svg
npx tsc --noEmit && npm test && npm run build
```

Expected: 全部乾淨。

- [ ] **★ Step 5: checkpoint：手動 E2E，16 項**

```bash
make dev
```

開瀏覽器 `http://127.0.0.1:5173`，逐項驗證：

1. **一進站看得到 idle 畫面**（「選好條件後按搜尋」），不是一片空白
2. 下拉選單有資料（20 個地區、12 個類別）、月份預先帶入本月
3. 國家選擇器在標題列右上角：台灣是漸層 active chip；日本/韓國降權，
   滑鼠停留出現「即將推出」tooltip
4. 按搜尋出現 skeleton 再變卡片，計數列顯示完整條件（「臺北 · 展覽 · 2026/07：N 筆活動」）
5. **點類別快捷 chip 會直接重新搜尋**，卡片內容跟著換（不是只有 chip 變色）
6. **錯誤畫面**：另開 terminal 跑 `docker compose -f docker-compose.dev.yml stop backend`，
   回瀏覽器按搜尋 → 出現虛線邊框的錯誤畫面 + 重試鈕。
   跑 `docker compose -f docker-compose.dev.yml start backend` 後按重試 → 恢復正常
7. **空結果畫面**：選一個確定沒活動的組合（例如「連江 + 演唱會 + 明年 12 月」）→
   出現「找不到符合條件的活動」，與第 6 項是完全不同的畫面
8. **主題持久化**：切成 light → 重新整理 → 仍然是 light
9. EN/中 切換會翻整頁文案（含 About 的角色欄），且錯誤訊息也跟著翻
10. **devtools 切到 375px**：rail 消失、頂部 bar 出現且**四顆按鈕都在（含語言切換）**、
    卡片單欄、搜尋膠囊改直向堆疊；1440px：三欄 grid
11. 「關於」頁：tech stack 表 + GitHub 連結
12. **語言持久化**：切成 EN → 重新整理 → 仍然是 EN（v4 只持久化了主題）
13. **返回鍵**：進「關於」頁 → 按瀏覽器返回 → 回到搜尋頁，**不是離開網站**
14. **快速連點兩個類別 chip**：最後停下來的那個 chip 是 active，
    而且下方的卡片就是那個類別的。舊請求不可以後到把畫面蓋掉（AbortController）
15. **About 頁在桌機寬度下有 `<h1>`**：devtools 的 Elements 搜尋 `h1`，要找得到一個
16. **8 秒文案**：把 backend 停掉再啟動製造一次冷查詢，或直接在 devtools 的
    Network 面板開 Slow 3G，確認等超過 8 秒時 skeleton 上方出現「第一次查詢比較慢」

第 1、5、6、7、8、10 項是 v4 就有的驗收；第 12 到 16 項是 v5 補的修正，特別確認。

`Ctrl+C` 結束。

- [ ] **Step 6: Commit**

```bash
git add frontend
git commit -m "feat: assemble spa with idle state, theme persistence, source-aware retry"
```

**本 task 的 local 驗收：** 上面 16 項全過。

**Phase 3 結束狀態：** `make dev` 起得來，SPA 打新 `/api/v1` 全流程可用。

---
## Phase 4：prod image 與文件

### Task 12: prod multi-stage Dockerfile + SPA 路由 + 本機 smoke test ★ checkpoint

**Files:**
- Create: `Dockerfile`（repo root）
- Modify: `.dockerignore`、`backend/config/urls.py`

**Interfaces:**
- Consumes: T2 的 `backend/config/` 結構、T11 的 `frontend/`
- Produces: 可 build 的 prod container，供 T14（fly.toml 指向它）與 T15（CI 用它）使用

- [ ] **Step 1: `backend/config/urls.py` 加 SPA 路由與 catch-all**

整份取代：

```python
from django.http import Http404, HttpResponse
from django.urls import include, path, re_path

from .settings import FRONTEND_DIST

_INDEX = FRONTEND_DIST / "index.html"


def spa(request):
    """回 Vite 產出的 index.html。

    刻意不用 TemplateView：那會把 HTML 餵進 Django 的 template engine，
    多出一條「這份 HTML 不能含 {{ 或 {%」的隱藏規則，而那規則不在任何錯誤訊息裡。
    直接讀 bytes 沒有這個問題。

    dist 不存在時回 404 而不是 500：dev 環境與只跑後端測試的 CI job 都沒有 dist，
    讓它 500 的話會蓋掉真正的錯誤（spec §6.1）。
    """
    if not _INDEX.exists():
        raise Http404("frontend not built")
    return HttpResponse(_INDEX.read_bytes(), content_type="text/html")


urlpatterns = [
    path("health", include("health.urls")),
    path("api/v1/", include("events.urls")),
    # 沒有 react-router，所以不需要真正的 SPA fallback routing。
    # 但少了 catch-all 的話，打錯字的網址會拿到 Django 的裸 404 純文字頁，
    # 使用者會以為站掛了（spec §4）。
    #
    # 用裸的 ^.*$ 就好，不需要 negative lookahead：Django 由上往下依序比對，
    # api/ 與 health 在上面已經被吃掉，static/ 由 WhiteNoise 的 middleware
    # 在進到 urls 之前就攔截了。v4 那條 (?!api/|static/|health) 是重複防守，
    # 而且是這份 plan 裡最難讀的一行。
    re_path(r"^.*$", spa, name="spa"),
]
```

`FRONTEND_DIST` 在 settings 裡已經定義過（`STATICFILES_DIRS` 用同一個值）。

dev 環境不受影響：`make dev` 的前端由 Vite dev server 服務，Django 只出 `/api` 與 `/health`。

- [ ] **Step 2: 寫 root `Dockerfile`**

```dockerfile
# ---- Stage 1: build the React SPA ----
FROM node:22-slim AS frontend
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

# ---- Stage 2: Django + gunicorn (serves SPA via WhiteNoise) ----
FROM python:3.13-slim
# 釘特定 uv 版本，勿用 :latest： 其餘依賴都靠 uv.lock 釘死，uv binary 也要可重現。
COPY --from=ghcr.io/astral-sh/uv:0.12.5 /uv /uvx /bin/

ENV PYTHONUNBUFFERED=1 TZ=Asia/Taipei
ENV UV_PROJECT_ENVIRONMENT=/opt/venv
ENV PATH="/opt/venv/bin:$PATH"
# 刻意同時滿足 Fly（不注入 $PORT，靠此 default）與 Cloud Run（會注入 $PORT 覆蓋掉這個值）：
# Phase 6 遷 Cloud Run 時這個 image 可以直接重用，不需重寫（spec §6.1）
ENV PORT=8080

RUN groupadd -r appuser && useradd -r -g appuser -m appuser

WORKDIR /web
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY backend/ ./backend/
COPY --from=frontend /app/dist ./frontend/dist

# SECRET_KEY 只存在於這一層的指令環境，不會留在 image 裡。
# 用 ENV SECRET_KEY= 的話假值會變成 runtime 預設，等於廢掉 settings 的守衛；
# 用 DEBUG=True 繞過的話會把 debug 帶進 prod image。兩種都不可以（spec §6.1）。
RUN SECRET_KEY=build-only-not-used ALLOWED_HOSTS=build-only \
    python backend/manage.py collectstatic --noinput

RUN chown -R appuser:appuser /web /opt/venv
USER appuser

# --workers 1：LocMemCache 是 per-process，多 worker 會各自持有獨立 cache。
# --threads 8 --worker-class gthread：gunicorn 預設的 sync worker 一次只吃一個 request，
#   cache miss 時 15 秒的上游請求會把 /health、靜態檔、其他人的搜尋全部堵住，
#   排到超時 worker 被 kill、cache 歸零，而且是在流量最高的時候破功（spec §3.3）。
#   threads 仍在同一個 process，LocMemCache 只有一份，前提不變。
# --timeout 60：要比上游的 15 秒 timeout 大。
CMD exec gunicorn --chdir backend config.wsgi:application \
    --bind 0.0.0.0:${PORT:-8080} \
    --workers 1 --threads 8 --worker-class gthread --timeout 60
```

- [ ] **Step 3: 整份取代 root `.dockerignore`**

```
.git
.venv
frontend/node_modules
frontend/dist
staticfiles
docs
readme
**/__pycache__
*.log
```

- [ ] **Step 4: 本機跑後端測試（跟 container 能否啟動分開驗證）**

```bash
cd backend && DEBUG=True uv run python -m pytest . -v && cd ..
```

Expected: 全綠。

- [ ] **★ Step 5: 本機 prod-like container smoke test（checkpoint）**

這是唯一一次真實部署前的最後防線（spec §8）。下一個真的會動到 prod 的動作
是 T16 的 cutover，而那一步沒有回頭路。這一步過不了就不要往下走。

```bash
docker build -t cef-local .
```

Expected: build 成功走完 `collectstatic`。**若在 collectstatic 那層失敗並顯示
`ImproperlyConfigured: SECRET_KEY must be set`**，代表 Step 2 的行內注入沒寫對，
回去看那一行，不要改成 `ENV SECRET_KEY=` 或 `DEBUG=True`。

```bash
docker run --rm -e PORT=8080 -e SECRET_KEY=smoke-test-only -e ALLOWED_HOSTS='*' \
  -p 8080:8080 -d --name cef cef-local
sleep 6
curl -s http://127.0.0.1:8080/health; echo
```

Expected: `{"status": "ok"}`

```bash
curl -s http://127.0.0.1:8080/ | grep -o '<title>[^<]*'
```

Expected: Vite build 出的 `<title>` 內容，代表 WhiteNoise 有把 `frontend/dist/index.html` 服務出來。

```bash
curl -s -o /dev/null -w "catch-all: %{http_code}\n" http://127.0.0.1:8080/some/typo/path
```

Expected: `catch-all: 200`（不是 404），代表打錯字的網址會回到 SPA。

```bash
curl -s http://127.0.0.1:8080/api/v1/countries | head -c 120; echo
curl -s "http://127.0.0.1:8080/api/v1/tw/events?category=6&location=%E8%87%BA%E5%8C%97&month=$(date +%Y-%m)" | head -c 200; echo
```

Expected: 第一個以 `[{"code":"tw"` 開頭；第二個是 `{"events": [...]}` 形狀的 JSON。

確認 `public/` 的資產在 prod 拿得到（**只有 prod 會壞的一項**）：

```bash
# T4 的 scaffold 在 frontend/public/ 放了一個 favicon.svg
curl -s -o /dev/null -w "public asset: %{http_code}\n" http://127.0.0.1:8080/static/favicon.svg
```

Expected: `public asset: 200`。`base: "/static/"` 只在 production build 生效，
Vite 會改寫 `index.html` 裡的字面引用，但 JSX 裡 runtime 寫死的 `/foo.png`
不會被加前綴。dev 一切正常、prod 404，而且沒有任何錯誤訊息（spec §6.1）。

確認 hash 過的資產拿到 immutable cache header：

```bash
ASSET=$(curl -s http://127.0.0.1:8080/ | grep -o '/static/assets/index-[^"]*\.js' | head -1)
curl -s -I "http://127.0.0.1:8080$ASSET" | grep -i cache-control
```

Expected: 含 `immutable`。若是 `max-age=60`，代表 settings 的
`WHITENOISE_IMMUTABLE_FILE_TEST` 沒生效，回 T2 檢查那個函式。

確認 gunicorn 真的是 gthread：

```bash
docker exec cef sh -c 'cat /proc/1/cmdline | tr "\0" " "; echo'
docker stop cef
```

Expected: 指令列含 `--threads 8` 與 `--worker-class gthread`。

- [ ] **Step 6: Commit**

```bash
git add Dockerfile .dockerignore backend/config/urls.py
git commit -m "feat: single-container production build (spa + api), spa catch-all, gthread worker"
```

**本 task 的 local 驗收：** `make run-prod` 起得來，`curl localhost:8080/health`
回 `{"status": "ok"}`，`/some/typo/path` 回 200。

---

### Task 13: repo 改名 + README

**Files:**
- Modify: `README.md`（整份改寫）
- Delete: `readme/*.png`（過時截圖）

**Interfaces:**
- Consumes: T3 定稿的 makefile 指令
- Produces: 給朋友 / 未來自己看的 README；Live URL 欄位留給 T16 填

- [ ] **Step 1: 整份改寫 `README.md`**

```markdown
# Culture Event Finder

Search culture events via government open data. Currently supports Taiwan
(Ministry of Culture); the provider architecture is ready for more countries.

**Live:** TBD-in-T16（部署驗證通過後填入 Fly.io 網址）

## Architecture

React (Vite + TS + Tailwind) SPA + Django JSON API, shipped as ONE container.
Backend layering: views → services (cache-aside, 12h TTL) → providers (one
per country).

| Layer | Stack |
|---|---|
| Frontend | React 19, Vite, TypeScript, Tailwind v4 |
| Backend | Django 5.2, uv, toolkitsy (logging) |
| Tests | pytest + responses / Vitest |
| Deploy | Docker multi-stage → Fly.io, GitHub Actions (flyctl) |

## Dev

    make dev            # backend + frontend dev container (docker-compose)
    make dev-reset      # 砍掉 named volume 重建；裝新前端套件後要跑這個
    make install-host   # host 另裝一份 frontend node_modules，給 IDE 用
    make test           # backend + frontend 測試
    make run-prod       # 本機 build 並跑 production container

前端只能走 `make dev`，不能在 host 上單獨 `npm run dev`：Vite 的 `/api` proxy
指向 Docker DNS 名稱 `backend:8000`，host 上解析不到。

## Environment variables

| 變數 | 設在哪 | 誰用它 | 沒設會怎樣 |
|---|---|---|---|
| `SECRET_KEY` | Fly：`fly secrets set`；build：Dockerfile 的 collectstatic 行內；test：makefile 與 CI | Django | 非 dev 時啟動即 `ImproperlyConfigured` |
| `ALLOWED_HOSTS` | Fly：`fly.toml` 的 `[env]` | Django | 非 dev 時啟動即 `ImproperlyConfigured` |
| `DEBUG` | dev：`docker-compose.dev.yml` | Django | 預設走 prod 分支 |
| `PORT` | Dockerfile 的 `ENV PORT=8080`；Cloud Run 自動注入 | gunicorn | Fly 不注入，靠 Dockerfile 的預設值 |
| `FLY_API_TOKEN` | GitHub repo secret | CI 的 deploy job | deploy job 失敗 |

## Monitoring

- `GET /health`：liveness only。**不碰外部依賴**，MoC 掛掉時它照樣回 200。
  只夠給 Fly 的 health check 用，不要拿它當監控。
- **uptime 監控打的是一個真實的搜尋 URL**，每 30 分鐘一次，對 HTTP 502 或
  `"events": []` 告警。這是唯一會真的碰到上游、而且順便證明過濾邏輯還活著的做法。

### 站看起來沒壞但查不到東西時，先看 meta

每個搜尋回應都帶 `meta`，一個 curl 就分得出是誰的問題：

```bash
curl -s "https://taiwan-culture-event-info.fly.dev/api/v1/tw/events?category=6&location=臺北&month=2026-09" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['meta'])"
```

- `rawCount` 為 0 → 上游那邊就沒資料，或 MoC 改了回應格式
- `rawCount` 大而 `matchedCount` 為 0 → 上游有資料但被本地過濾吃掉了，
  查 `_matches()`（地名比對或區間重疊）
- `cacheAge` 為 `null` → 這次是 cache miss，剛打過上游；有數字代表這份資料
  已經在記憶體裡放了幾秒

### 已知限制

machine 是 scale-to-zero，`fly logs` 只有即時串流，沒有歷史保存。
使用者回報問題時附上的 `X-Request-ID` **無法**拿來回查當時的 log。
上面那個 `meta` 就是為了取代這件事而存在的。

### 這幾樣東西不會自己告訴你該換掉了

- `taiwan.py` 的 `verify=False` + `urllib3.disable_warnings`：MoC 的憑證缺
  Subject Key Identifier，Python 3.13 拒連，所以關掉驗證。**這是 process 全域而且永久靜音的。**
  哪天 MoC 修好憑證或換 host，沒有任何東西會通知你可以拿掉它
- 20 個地區前綴與 12 個類別 id 是硬編的。MoC 增減縣市或類別時不會有錯誤，
  只會有「使用者選不到」
- health check 的 `Host` header 與 `ALLOWED_HOSTS` 兩處都寫死了
  `taiwan-culture-event-info.fly.dev`。換自訂網域或搬 Cloud Run 時兩邊都要改，
  漏一邊的症狀是「部署成功但全站 400」

## Rollback

**不要用 `fly deploy -i <sha>`。** 那個指令只換 image 不換 config，
而本專案的 `internal_port` 從 8787 改成了 8080，所以換回舊 image 會得到
「新 config 配舊 image」這個更壞的組合，指令會成功、站會繼續不通。

真正的 rollback 是連 config 一起滾回：

```bash
git checkout pre-phase5-fly-config && fly deploy
```

要完整重 build，好幾分鐘。這是單機器就地替換換來的代價。

## Adding a country

架構撐得住，但不是「新增一個檔案就好」。實際要動四處：

1. `backend/events/providers/<country>.py`：subclass `BaseProvider`，
   實作 `fetch_events(category_id)` 與 `locations` / `categories`
2. 在 `backend/events/providers/__init__.py` 的 `PROVIDERS` 註冊它
3. **從 `frontend/src/components/SearchForm.tsx` 的 `COMING_SOON` 移除該國**，
   否則畫面上會同時出現一個 active chip 和一個未開放 chip
4. `frontend/src/locales/*.json` 補該國需要的文案

還有兩個可能會擋路的假設：
- `services._matches` 的地名比對是子字串 + `台/臺` 正規化，綁死中文地址習慣
- `fetch_events(category_id)` 假設「一次抓完整個 category 再本地過濾」。
  若新的上游需要帶日期或分頁參數，`base.py` 的簽名要加參數
```

- [ ] **Step 2: 刪掉過時截圖並 commit**

```bash
git rm readme/culture.png readme/deployment.png readme/logs.png
git add README.md
git commit -m "docs: rewrite readme with env table, monitoring caveats, honest adding-a-country steps"
```

- [ ] **Step 3（OWNER）：GitHub Settings 改名 repo 為 `culture-event-finder`**

Settings → General → Repository name。GitHub 會自動幫舊網址設 redirect。

**不要動 visibility，維持 public。** build-smoke 每次 PR 與 push 都跑一次完整
multi-stage build；public repo 的 Actions 分鐘數不計費，轉 private 的話免費額度
是 2000 分鐘/月，密集開發期會在月中耗盡，deploy job 排不進去。

- [ ] **Step 4: 本機同步 remote，確認沒有殘留舊名字**

```bash
git remote set-url origin git@github.com:taurus5650/culture-event-finder.git
git remote -v
```

Expected: 兩行都顯示新的 `culture-event-finder.git`。

```bash
grep -rn 'taiwan_culture_event_info_django_jinja2' \
  --include='*.py' --include='*.ts' --include='*.tsx' --include='*.md' \
  --include='*.yml' --include='*.toml' . 2>/dev/null
```

Expected: 沒有命中，或只剩 `docs/` 底下的舊 plan/spec（那些是歷史文件，不用改）。
`fly.toml` 的 `app = 'taiwan-culture-event-info'` 是 Fly app 名稱，**刻意不改**
（Phase 6 才整個下線）。

**本 task 的 local 驗收：** `git remote -v` 顯示新網址，README 在 GitHub 上排版正常。

**Phase 4 結束狀態：** codebase 乾淨，prod image build 得出來且本機驗證過，
prod 仍是 Fly 上的舊版。

---

## Phase 5：上 Fly.io

> ⚠️ **T14 + T15 + T16 建議當成同一段做完。** `fly.toml` 的 `internal_port`
> 改成 8080 之後、新 image 部署上去之前，config 與線上 image 不一致；
> 這段期間任何人手動 `fly deploy` 或 rollback 都會拿到不通的組合。

### Task 14: `fly.toml` 配置 + health check + rollback 前置手續

**這是整份 plan 風險最高的 task。** PORT 沒對齊、`ALLOWED_HOSTS` 沒設對、
health check 沒定義，真實部署當下就會全站掛掉，而且因為 `min_machines_running = 0`
（scale-to-zero），不會立刻被發現。

**Files:**
- Modify: `fly.toml`（整份改寫）

**Interfaces:**
- Consumes: T12 產出的 root `Dockerfile`（監聽 `${PORT:-8080}`，`ENV PORT=8080`）
- Produces: 給 T15 的 CI deploy job 使用的部署設定

- [ ] **Step 1: rollback 前置手續：動 `fly.toml` 之前先記下當前狀態**

Fly 的 rollback（`fly deploy -i <sha>`）只換 image，**不會還原 `fly.toml` / env / secrets**。
等一下要把 `internal_port` 從 `8787` 改成 `8080`；之後若要滾回舊 image（監聽 8787），
套用的仍是新版 `fly.toml`（8080），**rollback 指令會顯示成功但服務仍然不通**。

```bash
fly releases --image -a taiwan-culture-event-info | head -5
```

Expected: 印出目前的 release 列表。**把第一行的 IMAGE sha 抄下來**，
貼到你自己的筆記或下面那個 commit message 裡，現在就抄。

```bash
git tag pre-phase5-fly-config
git tag | grep pre-phase5
```

Expected: 印出 `pre-phase5-fly-config`。這個 tag 標記「fly.toml 變更前」那個 commit，
未來若要回到「8787 image + 8787 config」的組合，從這個 tag 開始，而不是只滾 image。

- [ ] **Step 2: 整份改寫 `fly.toml`**

```toml
app = 'taiwan-culture-event-info'
primary_region = 'hkg'

[env]
  DEBUG = 'False'
  ALLOWED_HOSTS = 'taiwan-culture-event-info.fly.dev'

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = 'stop'
  auto_start_machines = true
  min_machines_running = 0

# [http_service] 不會自動生出 HTTP health check。沒有這一段的話，port 對不齊時
# fly deploy 會回報成功、machine 起得來、proxy 打不到，然後靜默壞掉（spec §6.1）。
[[http_service.checks]]
  grace_period = '10s'
  interval = '30s'
  method = 'get'
  path = '/health'
  timeout = '5s'
  # check 是從機器內部發出的，Host header 預設不是網域名。少了這一段，
  # check 會拿到 Django 的 400 DisallowedHost，變成「加了 check 反而部署失敗」。
  [http_service.checks.headers]
    Host = 'taiwan-culture-event-info.fly.dev'

[[vm]]
  # 256mb 給一個 gunicorn worker + LocMemCache 夠用，而且比 1gb 便宜得多。
  # 若後端 checkpoint 量到單一 category 的資料大到裝不下（spec §10 的未量測項），
  # 先加 LocMemCache 的 MAX_ENTRIES 再考慮加記憶體。
  memory = '256mb'
  cpu_kind = 'shared'
  cpus = 1
```

跟現有版本比，四個改動，每個都要能講出理由：

1. **移除 `[build] dockerfile = 'deployment_tcei/Dockerfile'`**：該目錄已在 T2 刪除；
   root 現在就有 `Dockerfile`（T12 產出），Fly 預設會抓 repo root 的 `Dockerfile`。
2. **`internal_port` 從 `8787` 改成 `8080`**：對齊 T12 Dockerfile 的 `ENV PORT=8080`。
   Fly 不會注入 `$PORT`，它是靠這個欄位告訴 proxy 該打容器的哪個 port。
3. **新增 `[[http_service.checks]]`**：見上面的註解。
4. **移除 `[[mounts]]`（原本掛 `sqlite_data` volume）**：全新 settings 的
   `DATABASES` 是 `sqlite3` + `:memory:`（無 models、無需持久化）。
   **volume 本身先不刪**，見 T16 Step 7。

`ALLOWED_HOSTS` 放 `[env]` 不放 `fly secrets`：它不是敏感值，寫進版控的 `fly.toml`
反而更好追蹤。

- [ ] **Step 3: 設定 prod 的 `SECRET_KEY`**

settings 在非 dev 且沒注入 `SECRET_KEY` 時會直接拒絕啟動，部署前必須先設好。

```bash
fly secrets set -a taiwan-culture-event-info \
  SECRET_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(50))')
```

Expected: `Secrets are staged for the first deployment`，或已有 machine 時顯示 release 更新。

**這不是無副作用的指令**：`fly secrets set` 會立刻觸發一次現役 app 的 release 與
machine 重啟。舊 code 不吃這個 secret 所以無害，但要知道它動了什麼。

```bash
fly secrets list -a taiwan-culture-event-info
```

Expected: 列表中看得到 `SECRET_KEY`。

- [ ] **Step 4: 驗證 `fly.toml` 語法（不觸發真實部署）**

```bash
fly config validate -c fly.toml
```

Expected: `Configuration is valid`。

> **這一步只驗語法，證不了這個 task 存在的任何一個理由。** port 對不對得上、
> health check 的 `Host` header 會不會被 `ALLOWED_HOSTS` 擋，兩者都要到真的
> 部署才現形。這是 Fly 的先天限制，不是可以補的驗收，所以下一步的 staging
> 演練不是選配（spec §6.5）。

- [ ] **Step 4b: 開一次性的 staging app 演練整條部署**

```bash
fly apps create cef-staging
fly secrets set -a cef-staging SECRET_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(50))')
# -e ALLOWED_HOSTS 不可省：不帶的話這場演練剛好跳過風險最高的那個變數，
# 等於演了一場沒有主角的戲（spec §6.5）。
flyctl deploy -a cef-staging -e ALLOWED_HOSTS=cef-staging.fly.dev
```

驗完再收掉：

```bash
curl -s -o /dev/null -w 'staging health: %{http_code}\n' https://cef-staging.fly.dev/health
curl -s https://cef-staging.fly.dev/api/v1/countries | head -c 120; echo
fly apps destroy cef-staging
```

Expected: `staging health: 200`，countries 以 `[{"code":"tw"` 開頭。
兩者其一失敗就在 staging 上修到過，**不要拿現役 app 當試驗場**。

- [ ] **Step 5: Commit**

```bash
git add fly.toml
git commit -m "fix: align fly.toml internal_port with dockerfile, add http health check, inject ALLOWED_HOSTS"
```

**本 task 的 local 驗收：** `fly config validate` 通過，`fly secrets list`
看得到 `SECRET_KEY`，而且你已經把舊 image sha 抄在某個看得到的地方。

---

### Task 15: GitHub Actions 重寫（flyctl）

**Files:**
- Modify: `.github/workflows/deploy.yml`（整份改寫）

**Interfaces:**
- Consumes: T12 的 root `Dockerfile`、T14 的 `fly.toml`
- Produces: push master 時自動測試 + 部署到 Fly.io

- [ ] **Step 1: 整份改寫 `.github/workflows/deploy.yml`**

```yaml
name: Test and Deploy to Fly.io

on:
  push:
    branches: [master]
  pull_request:

# 單一 machine 就地替換，兩次快速 push 會讓兩個 flyctl deploy 對同一台機器賽跑。
# cancel-in-progress 用在 deploy 上有風險（可能停在半途），所以只排隊不取消。
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

jobs:
  test-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v10
      # --frozen：lock 與 pyproject 不同步時直接紅燈，確保 CI 測的是 prod build 會用的那組版本
      - run: uv sync --frozen
      # DEBUG=True 是必要的：settings 的守衛在非 dev 時會拒絕啟動
      - run: cd backend && DEBUG=True uv run python -m pytest . -v

  test-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: cd frontend && npm ci && npm test && npm run build

  # 合體 container 煙測：test-backend（無 frontend/dist）與 test-frontend（只有 vitest）
  # 都跑不到「collectstatic + WhiteNoise 服務 SPA + /api」一起動的路徑。
  build-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t smoke:latest .
      - run: docker run -d -p 8080:8080 -e PORT=8080 -e ALLOWED_HOSTS='*' -e SECRET_KEY=ci-smoke-only --name smoke smoke:latest
      - name: Probe the running container
        run: |
          for i in $(seq 1 15); do curl -sf http://localhost:8080/health && break || sleep 2; done
          curl -sf http://localhost:8080/ | grep -qi '<title>' || (echo "SPA index missing" && exit 1)
          curl -sf http://localhost:8080/api/v1/countries | grep -q '"tw"' || (echo "countries API broken" && exit 1)
          # public/ 的資產只在 prod 會 404（base: "/static/" 只在 production build 生效）。
          # 本機與 dev 都看不到這個問題，所以這裡是唯一的網子（spec §6.1）。
          curl -sf -o /dev/null http://localhost:8080/static/favicon.svg || (echo "public asset 404 in prod build" && exit 1)
          # catch-all 這條 route 只在有 frontend/dist 時才走得到，後端測試永遠碰不到它。
          curl -sf -o /dev/null http://localhost:8080/some/typo/path || (echo "SPA catch-all broken" && exit 1)
      # 失敗時只看得到 curl 的非零離開碼，分不出是啟動失敗還是路由不對
      - name: Container logs on failure
        if: failure()
        run: docker logs smoke

  deploy:
    if: github.ref == 'refs/heads/master'
    needs: [test-backend, test-frontend, build-smoke]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install flyctl
        run: curl -L https://fly.io/install.sh | sh
      - name: Deploy to Fly.io
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
        run: $HOME/.fly/bin/flyctl deploy --remote-only
```

跟舊版比，關鍵差異：

- **不需要 `permissions.id-token: write`**：那是 Workload Identity Federation 用的，
  只有 Phase 6 遷 Cloud Run 才需要。
- **沒有任何 `vars.GCP_*`**：那些是 Phase 6 才存在的 variables。
- 測試路徑全部指向 `backend/`，取代舊版硬編 `culture/tests.py`、`tech_stack/tests.py`。

**build-smoke 刻意不打真實的 MoC。** v4 讓它 curl events endpoint 再 `grep -q 'events'`，
兩個問題：政府 API 一有狀況你的 pipeline 就紅燈且**擋住所有 deploy**（包含緊急修正）；
而且 `grep -q 'events'` 對 `{"events": []}` 也會過，正是 spec §8.1 明文拒絕的通過條件
——一個永遠會過的檢查比沒有檢查更糟，因為它看起來像有在把關。
contract 漂移由 `test-backend` 的 `responses` mock 負責，真實 MoC 的驗證留在
T7 的 checkpoint 與 T16 Step 3 這兩個人工關卡。

- [ ] **Step 2（OWNER）：取得 `FLY_API_TOKEN` 並設進 GitHub repo secret**

在本機（不是 CI 裡）跑：

```bash
fly tokens create deploy -x 999999h
```

Expected: 印出一段 `FlyV1 ...` 開頭的 token 字串。

到 GitHub repo → Settings → Secrets and variables → Actions → New repository secret，
name 填 `FLY_API_TOKEN`，value 貼上剛才的 token，Add secret。

（workflow 本身無法自己設定 secret，這一步必須手動在網頁上完成一次。）

- [ ] **Step 3: Commit，開 draft PR 觀察 CI**

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: rewrite deploy pipeline for flyctl (backend/ paths, contract probe, failure logs)"
```

**注意 trigger 條件**：`on:` 是 `push.branches: [master]` 加 `pull_request`。
單純 `git push origin <feature-branch>`（沒開 PR、也沒 push 到 master）**兩個條件都不滿足，
Actions 頁面不會出現任何 run**：不要因為看不到 job 就以為 workflow 寫錯了。

```bash
git push origin <feature-branch>
gh pr create --base master --head <feature-branch> --draft \
  --title "Refactor: react + django api" \
  --body "驗證 CI 用，先不 merge。實際 merge 由 T16 執行。"
```

Expected: PR 建立後 Actions 出現一個 run，`test-backend`、`test-frontend`、
`build-smoke` 三個 job 綠燈。`deploy` job 因為 `if: github.ref == 'refs/heads/master'`
不會在 PR 上跑，這是預期行為。

**這個 PR 先留著不 merge**：T16 才是真正 merge 的時機。

**本 task 的 local 驗收：** GitHub Actions 頁面看到三個綠燈的 job。

---

### Task 16: prod cutover + 驗證 + uptime check ★ checkpoint

> ⚠️ **Step 2 是 prod 正式切換點，不是 dry-run。** 這一步之後現網從 Jinja2 舊站
> 變成新 SPA。若 PORT、SECRET_KEY、ALLOWED_HOSTS 任一出錯，舊 machine 已被替換，
> 站是掛的。這是整份 plan 唯一會讓現有網站中斷的一步。

**Files:**
- Modify: `README.md`（填入 Live URL）

**Interfaces:**
- Consumes: T14 的 `fly.toml`、T15 已綠燈的 CI
- Produces: 有真實流量、有監控的 prod

- [ ] **Step 1: cutover 前的四項確認**

四項都要親眼看到，缺一不可：

```bash
grep -A6 'http_service.checks' fly.toml     # 1. health check 在，而且有 Host header
fly secrets list -a taiwan-culture-event-info | grep SECRET_KEY   # 2. secret 在
fly config validate -c fly.toml             # 3. config 合法
git tag | grep pre-phase5-fly-config        # 4. rollback 的 tag 在
```

Expected: 第一個印出含 `path = '/health'` 與 `Host =` 的區塊；第二個有一行 `SECRET_KEY`；
第三個是 `Configuration is valid`；第四個印出那個 tag 名字。

**第 4 項是這一步唯一的保險。** 本專案的 rollback **不是** `fly deploy -i <sha>`：
`internal_port` 從 8787 改成了 8080，換回舊 image 會得到「新 config 配舊 image」，
指令會成功、站會繼續不通。真正的路徑是 `git checkout pre-phase5-fly-config && fly deploy`，
要完整重 build，好幾分鐘（spec §6.5）。tag 不在就先補上再往下走。

- [ ] **Step 1b: 暫時擴到兩台，縮短切換的停機**

單一 machine 就地替換，最少停 10 到 30 秒，失敗的話沒有上限。

```bash
fly scale count 2 -a taiwan-culture-event-info
```

驗證通過之後（Step 3 之後）縮回一台：

```bash
fly scale count 1 -a taiwan-culture-event-info
```

**不可以長期維持兩台**：LocMemCache 是 per-process，兩台各持一份 cache，
spec §3.3 的「每 12 小時最多打 12 次上游」上界會直接破功。這只是切換期間的暫時措施。

**若想無風險演練**，先開一個一次性的 app 打：

```bash
fly apps create cef-staging
flyctl deploy -a cef-staging
curl -s -o /dev/null -w '%{http_code}\n' https://cef-staging.fly.dev/health
fly apps destroy cef-staging --yes
```

成本是幾分鐘的 machine 時間。`ALLOWED_HOSTS` 要臨時改成 staging 網域才會通，
所以這一步只驗「image 起不起得來」，不驗網域設定。

- [ ] **Step 2: prod cutover：部署到現役 app**

```bash
flyctl deploy --remote-only
```

Expected: build 成功、release 建立、**health check 通過**（輸出會顯示 machine 的
health state 變成 `passing`）。

**若這步失敗**：問題只會出在 Dockerfile / fly.toml / Fly 帳號三者之一，跟 CI 無關。
常見死因與對應位置：
- `internal_port` 沒對齊 8080 → T14 Step 2
- `SECRET_KEY` 沒設 → T14 Step 3
- health check 被 400 擋掉（缺 Host header）→ T14 Step 2
- token 過期 → `fly auth login`

在本機修到過為止再進 Step 3。

- [ ] **★ Step 3: 用 `curl` 驗證，不能只靠瀏覽器（checkpoint）**

瀏覽器可能吃到快取，看起來正常但其實打到的是舊 revision。

**這一步是整份 plan 裡兩個靜默殺手唯一被證明的時刻**：`ENV PORT` 有沒有對上
`internal_port`、health check 的 `Host` header 會不會被 `ALLOWED_HOSTS` 擋。
T14 的 `fly config validate` 只驗語法，兩者都驗不到（Global Constraints）。

```bash
H=https://taiwan-culture-event-info.fly.dev
curl -s -o /dev/null -w 'health: %{http_code}\n' $H/health
curl -s $H/ | grep -o '<title>[^<]*'
curl -s $H/api/v1/countries | head -c 200; echo
# 真實搜尋：這是監控之後要打的那一條，先手動確認它會回 200 且不是空陣列
curl -s "$H/api/v1/tw/events?category=6&location=%E8%87%BA%E5%8C%97&month=$(date +%Y-%m)" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('meta:', d['meta']); assert len(d['events'])>0, 'FAIL: 空陣列'"
```

Expected: `health: 200`；Vite 的 `<title>`；以 `[{"code":"tw"` 開頭的 JSON；
最後一行印出 meta 且不 assert 失敗。
若 countries 回 400 加 `DisallowedHost` 字樣，代表 `ALLOWED_HOSTS` 沒生效，回 T14 Step 2。
若 health 打不到但 `fly deploy` 剛才回報成功，那就是 port 沒對齊，回 T14 Step 2。

- [ ] **Step 4: 驗證 cache 真的有生效**

同一組條件連打兩次，log 應該只出現一次 cache miss。這同時證明 `--workers 1`
真的只有一份 cache。

```bash
fly logs -a taiwan-culture-event-info &
URL="https://taiwan-culture-event-info.fly.dev/api/v1/tw/events?category=6&location=%E8%87%BA%E5%8C%97&month=$(date +%Y-%m)"
curl -s -o /dev/null "$URL"; sleep 2; curl -s -o /dev/null "$URL"
sleep 3; kill %1
```

Expected: log 裡 `Cache miss: events:tw:6` 只出現一次。出現兩次代表 cache 沒生效或
不只一個 process。

- [ ] **Step 5: merge master**

```bash
git checkout master
git merge --no-ff <feature-branch>
git push origin master
```

（T15 開的 draft PR 會在這次 push 後自動關閉。）

到 GitHub Actions 看 `deploy` job：四個 job 全綠。這次 CI 部署的內容跟 Step 2
手動部署的是同一個 commit，所以不會改變線上行為。

- [ ] **Step 6: 手機實機驗證一次完整搜尋流程**

用手機瀏覽器開 `https://taiwan-culture-event-info.fly.dev/`，選地區 + 類別 + 月份，
按搜尋。確認：

1. 出現卡片列表（或正確的空結果畫面），不是白畫面
2. 卡片欄位：活動名稱、時間、地點、票價、售票 badge：都有正常顯示，
   沒有 `undefined` 或空白。有欄位空掉的話先 `curl` `/api/v1/tw/events?...`
   看原始 JSON，跟 T5 `test_providers.py` 的 mock fixture 比對是哪個欄位改了名
3. **捲動與卡片 hover 順不順。** `.glass` 是 `blur(40px)` 的全螢幕面板加兩顆 36px 光暈，
   低階 Android 可能掉幀甚至白屏，而先前只在 mac 上驗證過（spec §10）。
   若明顯卡頓，照 `design.css` 頂部註解的降級出口改一次
4. 選一個展覽類別 + 下個月，確認**跨月的常設展查得到**（這是 88% 的展覽資料）

- [ ] **Step 7: 填入 README 的 Live URL**

```markdown
**Live:** https://taiwan-culture-event-info.fly.dev
```

```bash
git add README.md
git commit -m "docs: fill in live url after fly.io deployment verified"
git push origin master
```

- [ ] **Step 8: 設定 uptime check：一個 monitor，30 分鐘，打真實搜尋 URL**

到 UptimeRobot（或同類免費服務）建立**一個** HTTP(s) monitor：

- URL：`https://taiwan-culture-event-info.fly.dev/api/v1/tw/events?category=6&location=臺北&month=<當月>`
- Interval：**30 分鐘**
- 告警條件：HTTP 502，以及回應內容含 `"events":[]`
  （UptimeRobot 的 keyword monitor 選「keyword exists → down」）
- Alert 寄自己的 email

**為什麼不是 `/health`：** 它不碰任何外部依賴，MoC 掛掉時照樣回 200，
你會完全不知道站已經沒用，要等朋友抱怨（spec §3.4）。

**為什麼不是 v4 的 `/health/upstream`：** 那個 endpoint 已經移除。它讀的旗標
只在 cache miss 的路徑上被寫（沒人搜尋就永遠不會亮）、放在 LocMemCache
（machine 一停就清空）、而監控讀的是唯讀 cache view（整條鏈路沒有一段真的碰上游）。
三個理由任一個都足以讓它永遠回綠燈。

**為什麼是 30 分鐘不是 5 分鐘：** `min_machines_running = 0` 這個省錢設定要成立，
前提是機器真的會停。每 5 分鐘打一次會讓它 24 小時醒著，等於付了常開的錢
卻拿到 scale-to-zero 的冷啟動體驗。v4 排的是兩個 5 分鐘的 monitor，正是這個組合。

**這個選擇的代價，要接受：** 機器停著的時候，使用者第一次搜尋最壞要等 20 秒
（冷啟動加上上游 15 秒 timeout），而且 cache 是空的。T11 的「loading 超過 8 秒
換文案」就是為這個情境寫的。（owner 於 2026-08-23 選定省錢方案。
要改成常開的話：`min_machines_running = 1` + monitor 頻率隨意，
代價是持續計費，好處是 cache 保溫且 health check 真的會跑。）

Expected: 建立後幾分鐘內兩個 monitor 都顯示 `Up`。

- [ ] **Step 9: 記下孤兒 volume，但先不要刪**

T14 從 `fly.toml` 移除了 `[[mounts]]`，但 `sqlite_data` 這個 volume 還在帳號裡
持續佔資源（volume 是 $0.15/GB/月）。

```bash
fly volumes list -a taiwan-culture-event-info
```

把 volume id 抄下來，**現在先不要刪**。舊 image 的設定吃這個 volume，
一刪 `fly deploy -i <舊sha>` 這條 rollback 路徑就永久斷了。

**等新版穩定跑滿一週之後**再回來執行（在行事曆上排一個提醒）：

```bash
fly volumes destroy <volume-id> -a taiwan-culture-event-info
```

- [ ] **Step 10: 絕對不執行的指令**

```bash
# 不要跑這個： Fly 現在就是 prod
# fly apps destroy taiwan-culture-event-info
```

任何時候看到有人建議跑它，先確認是不是已經到了 Phase 6 驗證數天之後。

**本 task 的 local 驗收：** 手機上開 Live URL 搜尋得到活動，兩個 uptime monitor
都是 `Up`，而且你已經在行事曆上排好一週後刪 volume 的提醒。

---

## Phase 6：遷移 Cloud Run（另開 branch，不 block 主線）

另開 branch，不影響 master 上已經在跑的 Fly.io prod。
**前置**：Phase 0 的 fly.io billing 查核結果決定這個 phase 有沒有 deadline。

流程：GCP 一次性 infra（enable APIs、deployer service account + IAM roles、
Workload Identity Federation、billing budget alert）用 **Terraform** 管理；
app 的每次部署不進 Terraform，留在 CI 的 `gcloud run deploy`，tfstate 存本機並 gitignore。
Owner 手動跑一次 `gcp-setup`（建專案、綁 billing、`terraform apply`、把 output 填進
GitHub Actions variables）。接著把 deploy job 從 `flyctl deploy` 換成 `gcloud run deploy`
（需要 `permissions.id-token: write` 與 WIF 驗證）。驗證數天穩定後，才 `fly apps destroy`
並把 `fly.toml` 刪除，刻意保留重疊期。

**這個 phase 的三個已知陷阱，寫 plan 時不要漏**（spec §6.6）：

1. **$0 的破口在 Artifact Registry，不在 Cloud Run。** 同一個帳單帳戶只有 0.5 GB 儲存，
   而 image 約 200 到 400 MB，兩三個 revision 就吃完。必須設 cleanup policy
   只保留最近 2 個 image。
2. **`--max-instances=1` 不可省。** 預設 concurrency 80 / max 100，流量一來就開第二個
   instance，各自一份 LocMemCache，spec §3.3 的上界假設整個破功。
3. **`--min-instances=0`**（`> 0` 走 instance-based billing，不吃 free tier）。
   billing budget alert 設在 $1，不是 $10。

Terraform 檔案與 `docs/deployment/gcp-setup.md` 的完整內容，留到該 branch 開始時
另外寫一份 plan，此處不展開。

## Phase 7：k8s（另開 branch，隨時，純學習）

另開 branch，起 `kind` 本機叢集，寫最基本的 Deployment + Service manifest，
跑同一個 Phase 5/6 已經在用的 prod image。純粹是 side quest，跟出貨路徑零交集。

---

## 附錄：v4 → v5 的 task 對照

| v5 | Phase | 內容 | v4 對應 | 為什麼動 |
|---|---|---|---|---|
| T1 | 1 | uv 遷移 | T1 | 不變 |
| T2 | 1 | 清理 + 重構 + 全新 settings ★ | T2 | 加 ★（第一個不可逆點）、`pytest.ini` 加 `python_files`、刪檔加 `--ignore-unmatch`、WhiteNoise immutable test |
| T3 | 1 | dev docker-compose | T3 | `uv run --frozen --no-sync`、compose 加 `user:`、`make test` 拆兩半 |
| T4 | 1 | 前端 scaffold ★ | **T7** | 提前到 Phase 1，讓 `make test` 全程可用；proxy 驗收改打 `/health` |
| T5 | 2 | providers | **T4** | 加 `isinstance(payload, list)` 守衛、log 移到 raise 之前 |
| T6 | 2 | services | **T5** | 回傳改 `(events, meta)`、加 negative caching、加 cache hit/miss log、移除 upstream 旗標 |
| T7 | 2 | API + checkpoint ★ | **T6** | `isascii()`、月份 regex 收斂、location 先正規化、回傳帶 `meta`、移除 `/health/upstream` |
| T8 | 3 | types / api / utils | T8 | 加 `formatEventRange`、`formatPrice` |
| T9 | 3 | i18n | T9 | 語言持久化、拿掉 LanguageSwitch 的 `aria-label`、**驗收從 `tsc --noEmit` 改成三條 Vitest** |
| T10a | 3 | design system + 狀態元件 ★ | T10 前半 | 拆出來，自己一個 checkpoint |
| T10b | 3 | 卡片與列表 ★ | T10 中段 | 拆出來；卡片顯示時間區間與票價三形態 |
| T10c | 3 | 搜尋卡 ★ | T10 後半 | 拆出來；加「重設條件」鈕、chip 比對套 `String()` |
| T11 | 3 | App 組裝 ★ | T11 | 搜尋加 `AbortController`、About 補 `<h1>`、返回鍵進 history、8 秒慢速文案 |
| T12 | 4 | prod Dockerfile + SPA 路由 ★ | T12 | catch-all 改純 view 不走 template engine、加 ★、smoke test 加 public asset 與 cache header |
| T13 | 4 | repo 改名 + README | T13 | README 加 meta 排查段、rollback 真實路徑、會腐爛清單 |
| T14 | 5 | fly.toml + health check | T14 | 記憶體 1gb 改 256mb、加 staging 演練（含 `ALLOWED_HOSTS`） |
| T15 | 5 | CI 重寫 | T15 | 加 `concurrency`、拿掉打真實 MoC 的探針、加 public asset 與 catch-all 探針 |
| T16 | 5 | prod cutover ★ | T16 | 加 rollback tag 確認、切換前後 `fly scale count`、監控改成一個 30 分鐘的真實搜尋 URL |

**v5 沒有處理的三件事**（review 提出，屬於會改變視覺或功能的決策，等 owner 拍板）：
裝飾用 SVG 約 150 行、i18n 的雙語 label schema 約 60 行、
`COMING_SOON` 日韓 chip 與四個類別快捷 chip 約 70 行。

---

## 附錄：v3 → v4 的 task 對照（承襲自 v4，僅供追溯）

僅供追溯，執行時不需開啟 v3。

| v4 | Phase | 內容 | v3 對應 |
|---|---|---|---|
| T1 | 1 | uv 遷移 | v3 T1（拿掉 Step 6-8 的 dev Dockerfile 改寫，那份檔案在 v4 不存在） |
| T2 | 1 | 清理 + 重構 backend/config + 全新 settings | v3 T11 + T12 合併並前移（settings 只寫一次而不是三次） |
| T3 | 1 | dev docker-compose（僅 backend） | v3 T2（venv 移出 bind mount、加 dev-reset、frontend service 延後） |
| T5 | 2 | providers | v3 T3（補 22 縣市、異體字正規化、timeout 測試、格式漂移訊號） |
| T6 | 2 | services | v3 T5（月份改區間重疊、加上游健康旗標、端到端測試） |
| T7 | 2 | API + checkpoint | v3 T6（白名單驗證、404 移到 view、checkpoint 不接受空陣列） |
| T4 | 3 | 前端 scaffold + compose frontend service | v3 T2 後半 + T7（resolveJsonModule、vitest config、polling 改 vite.config） |
| T8 | 3 | types / api / utils | v3 T4（currentMonth 改本地時區、抽出 search.ts 純函式、AbortSignal） |
| T9 | 3 | i18n | v3 T8（加 html lang 同步、LanguageSwitch 的 small 變體） |
| T10 | 3 | UI 元件 + checkpoint | v3 T9（focus ring、chevron、live region、4 個 chip、fixture 頁 checkpoint） |
| T11 | 3 | App 組裝 + checkpoint | v3 T10（idle 畫面、主題持久化、依來源重試、手機語言切換） |
| T12 | 4 | prod Dockerfile + SPA catch-all | v3 T13（build 期 SECRET_KEY、gthread、non-root、catch-all 路由） |
| T13 | 4 | repo 改名 + README | v3 T14（加 env 表、監控說明、誠實版 adding a country） |
| T14 | 5 | fly.toml + health check | v3 T15（新增 `[[http_service.checks]]`、volume 改為延後刪除） |
| T15 | 5 | CI 重寫 | v3 T16（setup-uv v10、events contract probe、失敗時印 logs） |
| T16 | 5 | prod cutover + 驗證 | v3 T17（dry-run 正名為 cutover、加 cache 驗證、監控改指 upstream） |
