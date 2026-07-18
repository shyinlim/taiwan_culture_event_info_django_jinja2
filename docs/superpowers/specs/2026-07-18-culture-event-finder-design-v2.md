# Culture Event Finder — React + Django API 重構設計 v2

- 日期：2026-07-18
- 狀態：已與 owner 逐項確認並批准
- 取代：`2026-07-11-react-django-refactor-design.md`（v1）
- 前身：`taiwan_culture_event_info_django_jinja2`（Django + Jinja2 server-rendered）

> 本文件自足。執行時不需開啟 v1。v1 已標記 SUPERSEDED，其中的 hosting 決策、
> 清理清單與里程碑順序皆已失效。

## 0. v2 相對 v1 改了什麼（僅供追溯，執行時不需理會）

| 項目 | v1 | v2 | 理由 |
|---|---|---|---|
| Hosting 路徑 | 直接遷 Cloud Run，Fly.io 下線 | Phase 3 續用 Fly.io；Phase 4 才遷 Cloud Run | 降風險、先出貨。Fly app 仍存活，舊帳號可能吃 grandfathered 額度 |
| Terraform | 主線 task，含 PAUSE POINT | 移到 Phase 4（另開 branch） | 移除主線唯一的 owner-gated 中斷點 |
| 清理重構 | 最後一個 milestone，在部署之後 | 前移到 Phase 2，在上線之前 | 避免 Dockerfile 寫兩次；提早解除 settings.py 的 hook 摩擦 |
| dev 環境 | 兩個裸 process | docker-compose 兩個 service | owner 現有習慣即 docker dev；符合市面主流做法 |
| docker-compose | 在刪除清單 | 保留並改造成 dev 環境 | 同上 |
| UI 設計 | 無 mockup，直接寫 React | Phase 0 加 POC HTML gate | owner 首次寫 React，fail-fast |
| k8s | 未提及 | Phase 5 side quest，不進主線 | Fly / Cloud Run 皆非 k8s，manifest 與出貨路徑零交集 |

## 1. 背景與定位

現況是一個 Django server-rendered 網站，透過台灣文化部 (MoC) open data API 查詢藝文活動。
產品定位：**owner 自己 + 朋友真的在用的實用工具**，不是 portfolio 展示品、暫不需要 SEO。

重構的驅動需求：

1. 前後端改為 **React (Vite) SPA + Django JSON API**，兼顧 owner 的 SDET 職涯發展。
2. Hosting 分兩階段：**Phase 3 續用 Fly.io**（現有 app 仍服役，遷移風險為零，先出貨）；
   **Phase 4 才遷 Google Cloud Run** 追求 $0（always-free tier）。Phase 4 另開 branch，不 block 主線。
3. UI 全面重做：**mobile-first responsive、卡片式列表**，丟掉 Material Dashboard 後台模板。
4. **多國擴展架構**：先做骨架，只實作台灣；未來加國家不推翻重寫。
5. **UI 多語言 (zh/en)**：用免費方案 (locale JSON)，不用付費翻譯服務。
6. **Cache 機制**：cache-aside + TTL，為未來付費 API 預留，跨 user 共用。
7. Code 難度控制在 owner 可獨立維護的程度（entry-level SDET 看得懂）。
8. **dev 環境 container 化**：docker-compose 起 Django + Vite，符合市面主流做法。

### 明確排除 (YAGNI)

- 搜尋條件進 URL (shareable URL) — 之後要加很容易
- 結果二次篩選/排序
- SEO / SSR
- 活動內容機器翻譯（活動資料維持資料源語言）
- 第二個國家的實作（只做架構）
- **k8s 不進主線** — Fly.io 走 `fly.toml` + Dockerfile → Firecracker microVM，
  Cloud Run 是 serverless container，兩者皆不吃 k8s manifest。
  Fly Kubernetes (FKS) 仍在 closed beta 且官方不建議 production。
  k8s 對本專案（單 container、單 machine）無運行價值，列為 Phase 5 學習用 side quest。

## 2. Repo 結構 (monorepo)

Repo 改名為 `culture-event-finder`（GitHub 改名，舊名自動 redirect）。

```
culture-event-finder/
├── backend/                     # Django
│   ├── config/                  # settings / urls / wsgi
│   ├── events/                  # 唯一的業務 app
│   │   ├── providers/
│   │   │   ├── base.py          # Provider 抽象 interface + registry
│   │   │   └── taiwan.py        # MoC API 呼叫 + 台灣 locations/categories 設定
│   │   ├── services.py          # cache-aside + 過濾/排序（純函式）
│   │   ├── views.py             # 薄：只做 HTTP in/out 與參數驗證
│   │   └── tests/
│   └── health/                  # health check endpoint
├── frontend/                    # Vite + React + TypeScript
│   ├── Dockerfile.dev           # dev 用 node container
│   └── src/
│       ├── components/          # SearchForm, EventList, EventCard,
│       │                        # ErrorMessage, SkeletonCard, LanguageSwitch
│       ├── api.ts               # 集中所有 fetch 呼叫
│       └── locales/             # zh.json / en.json
├── Dockerfile                   # prod multi-stage（repo root）
├── docker-compose.dev.yml       # dev 環境（repo root，不放 deployment_tcei/）
└── fly.toml
```

**`docker-compose.dev.yml` 與 `frontend/Dockerfile.dev` 必須放在 repo root / frontend/**，
不可放 `deployment_tcei/` — 該目錄在 Phase 2 會被整個刪除，放進去會連坐消失。

分層原則：views (HTTP) → services (業務邏輯 + cache) → providers (外部資料源)。
每層單獨可測、單獨可替換。

## 3. 後端設計

### 3.1 API contract

```
GET /api/v1/countries
    → [{ "code": "tw", "name": {...}, "locations": [...], "categories": [...] }]
      前端下拉選單的資料來源；只回傳已註冊的國家。

GET /api/v1/{country}/events?category=6&location=臺北&month=2026-07
    → { "events": [ { "title", "startTime", "endTime", "location",
                      "locationName", "onSales", "price", "googleMapUrl",
                      "googleSearchUrl" } ] }

GET /health
    → 200，平台 health check
```

錯誤格式統一：`{ "error": { "code": "...", "message": "..." } }`

- 參數缺漏/非法 → 400
- 未支援的國家 → 404
- 上游 (MoC) 失敗或 timeout → 502，前端顯示「資料來源暫時無法使用」
- 「查無結果」是 200 + 空陣列，與錯誤明確區分

### 3.2 Provider 架構（多國擴展點）

- `base.py` 定義抽象 interface：`fetch_events(category) -> list[Event]`、
  `locations`、`categories`、國家 metadata。
- `taiwan.py` 實作 MoC API。SSL `verify=False` workaround（`cloud.culture.tw` 憑證缺
  Subject Key Identifier，Python 3.13 拒連）封裝在此檔內並附註解。
- Registry 用簡單 dict：`PROVIDERS = {"tw": TaiwanProvider()}`。
  未來加日本 = 新增 `japan.py` + registry 加一行。
- 現有 `data.py` 的 hardcoded Location/EventCategory 移入 `taiwan.py`。

### 3.3 Cache (cache-aside)

- 用 Django cache framework (`django.core.cache`)，業務 code 只碰 `cache.get/set`。
- Key：`events:{country}:{category}`；TTL：12 小時 (`43200` 秒)。
- **月份格式轉換（必做）**：API 收 ISO `month=2026-07`，但 MoC 的 `show['time']`
  是 `YYYY/MM/DD HH:MM:SS`。services 層過濾前必須把 `2026-07` 轉成 `2026/07`
  再比對，否則永遠查無結果。測試必須涵蓋。
- MoC API 只按 category 查詢（location/月份是本地過濾）。「每 12 小時最多打 12 次上游、
  跨 user 共用」是 best-effort 上界，不是硬保證：LocMemCache 是 per-process 記憶體。
  故 prod 用 `--workers 1`；Fly 側 `min_machines_running = 0` 且維持單 machine。
- cache-aside 的 check-then-set 無 lock，冷啟動瞬間並發仍可能對同一 category 重複打上游 —
  屬可接受取捨（MoC 免費、不影響正確性），不加 lock 以維持 code 簡單。
- 已知限制：scale-to-zero 時 cache 消失 — 可接受。未來接付費 API 時僅改 settings
  換持久 backend，業務 code 不動。

## 4. 前端設計

- **Vite + React + TypeScript**：型別寫到夠用，不用進階泛型、不用 Redux、不用 server components。
- **Tailwind CSS**：mobile-first utility。
- **i18n**：UI 文案走 `locales/zh.json` / `en.json` + 一個輕量 context/hook
  （不引重型 i18n 套件），右上角切換，預設中文。活動資料維持資料源語言。
- **不用 react-router**：只有搜尋頁和 About 兩個畫面，用 in-app state 切換。
  這同時避開 SPA fallback routing 問題（WhiteNoise 不會 catch-all 到 index.html，
  BrowserRouter 重新整理會 404）。
- 頁面結構：
  - 搜尋列：國家（只有台灣時隱藏）、地區、類別、月份選擇器（年 + 月兩個 `<select>`；
    刻意不用 `<input type="month">` — 桌面版 Firefox 與 Safari 全版本不支援，
    會 fallback 成純文字框）、搜尋鈕
  - 結果：卡片式列表 — 活動名稱、時間、地點（點擊開 Google Map）、票價、
    售票狀態 badge；手機單欄、桌機多欄 grid
  - Loading：skeleton 卡片
  - 空結果與錯誤是兩種不同的畫面
  - About 頁：合併原 tech_stack 頁內容（tech stack 表 + 作者連結）
- 視覺方向：乾淨留白、清楚的層級，像簡化版 KKTIX 列表頁。

### 4.1 POC HTML gate（Phase 0）

owner 首次寫 React。為避免視覺方向錯誤導致 6 個元件全部重寫，
**寫 React 元件前必須先產出一頁靜態 POC HTML 並取得 owner 確認**。

- 形式：單一 `.html` 檔 + Tailwind CDN + 寫死的假資料，無 build step
- 必須涵蓋：搜尋列、卡片 grid、skeleton 載入狀態、空結果畫面、錯誤畫面
- **交付判準**：起一個 local server，截 375px（手機）與 1440px（桌機）兩張圖交給 owner。
  owner 明確回覆確認後，此 gate 才算通過。
- **定性：丟棄式 prototype。** HTML 檔本身不進 repo 主線，
  但其中調校過的 Tailwind class 組合（卡片、badge、間距）直接抄進 React 元件的 JSX。

## 5. 開發環境

### 5.1 dev 走 docker-compose（兩個 service）

採市面主流配方，避免 host 環境漂移：

- `docker-compose.dev.yml`（repo root）起兩個 service：
  - `backend`：Django `runserver 0.0.0.0:8000`
  - `frontend`：Vite dev server，port 5173，`server.proxy` 把 `/api` 轉發到 `backend:8000`
- **bind mount 原始碼，named volume 隔離 `node_modules`** — 若讓 host (mac ARM) 的
  `node_modules` 蓋掉 container (linux) 的，esbuild 等原生依賴會直接崩潰。
- **`server.host: '0.0.0.0'`** — 否則 host 瀏覽器連不進 container 內的 Vite。
- **`CHOKIDAR_USEPOLLING=true`** — macOS/Windows 的 docker file-watching 事件不可靠，
  不開 polling 則 HMR 不會觸發。
- container 內跑 non-root user。
- Dockerfile 先 `COPY package.json package-lock.json`，再 `COPY` 其餘 — 吃 layer cache。

### 5.2 host 也要裝一份 node_modules（給 IDE 用）

因為 `node_modules` 被 named volume 隔離在 container 內，host 上不存在，
編輯器的 TS server / eslint / import 跳轉會全部失效。

解法（市面標準做法）：host 另跑一次 `npm ci`。

- host 那份**只給編輯器讀**，container 那份才是實際執行的
- 兩份吃同一個 `package-lock.json`，不會漂移
- makefile 提供 `make install-host` target，並在註解說明為何要裝兩份

### 5.3 dev 常用指令

由 makefile 提供：`make dev`（起 compose）、`make install-host`、`make test`、
`make run-prod`（本機 build & run production container）。

## 6. 部署

### 6.1 Phase 3 目標平台：Fly.io

- **Multi-stage Dockerfile**（repo root）：
  stage 1 (`node:22-slim`) `vite build` → stage 2 (`python:3.13-slim`) Django + gunicorn，
  WhiteNoise 服務 React build 產物 + `/api` JSON。單一 container、單一網址、無 CORS。
- 新增依賴：`gunicorn`、`whitenoise`。
- Python stage 需跑 `collectstatic`；`STATICFILES_DIRS` 指向 `frontend/dist`。
- WhiteNoise 用 plain storage（非 manifest storage）：Vite 已對檔名做 content-hash，
  manifest storage 會重複 hash 且可能 500。
- CMD：`gunicorn --chdir backend config.wsgi:application --bind 0.0.0.0:${PORT:-8080} --workers 1`
  （`config.wsgi` 而非 `main_project.wsgi` — Phase 2 已完成目錄重構）

#### ⚠️ PORT：Fly 不注入 `$PORT`

**這是本專案最容易靜默失敗的一點。**

Cloud Run 會自動注入 `$PORT`；**Fly.io 不會** — Fly 是靠 `fly.toml` 的
`[http_service].internal_port` 指定容器監聽哪個 port。

若 Dockerfile 用 `${PORT:-8080}` 而 `fly.toml` 仍是現有的 `internal_port = 8787`，
容器會監聽 8080、Fly proxy 打 8787 → 兩邊對不上，health check 永遠失敗。
搭配 `min_machines_running = 0`（scale-to-zero），**壞掉不會立刻被發現**，
要等下一個訪客觸發冷啟動。

必須做到：
- Dockerfile 加 `ENV PORT=8080`
- `fly.toml` 的 `internal_port` 改為 `8080`
- 部署驗證用 `curl`，不可只用瀏覽器（瀏覽器可能吃到快取而誤判成功）

#### ⚠️ ALLOWED_HOSTS 用環境變數

Django 的 `ALLOWED_HOSTS` 若寫死 `*.run.app`，部署到 `*.fly.dev` 會對所有請求
回 400 DisallowedHost — 等於上線當下全站掛，且瀏覽器上看起來像伺服器故障，排查成本高。

必須用環境變數注入：Phase 3 傳 Fly 網域，Phase 4 遷 Cloud Run 時改傳 `*.run.app`。

#### Dockerfile 刻意寫成 Cloud Run 相容

`$PORT` 的寫法（`${PORT:-8080}` + `ENV PORT`）刻意同時滿足 Fly 與 Cloud Run，
Phase 4 遷移時 image 可直接重用，不需重寫。這不是 YAGNI —
Phase 4 是已確認要做的事，只是排在後面。

### 6.2 CI/CD

**GitHub Actions**（push master）四個 job：

1. `test-backend` — pytest
2. `test-frontend` — vitest
3. `build-smoke` — 真的 `docker build` 起 container，curl `/health`、`/`、
   `/api/v1/countries`，確認 collectstatic + WhiteNoise 服務 SPA 與 `/api`
   真的一起起得來。這條路徑 test-backend/test-frontend 都不會跑到。
4. `deploy` — `needs: [test-backend, test-frontend, build-smoke]`，
   `if: github.ref == 'refs/heads/master'`，用 `flyctl deploy` + `secrets.FLY_API_TOKEN`

紅燈不部署。**不需要** `permissions.id-token: write`（那是 Workload Identity Federation
用的，Phase 4 才需要）。

### 6.3 依賴管理單一來源：uv

`pyproject.toml` (PEP 621) + `uv.lock` 是 source of truth；
Dockerfile 用 `uv sync --frozen --no-dev`。
刪除 `requirements.txt` 與 Poetry 設定，消除 split-brain。

**注意**：dev 用的 container 需要 pytest 等 dev dependencies，
`uv sync --frozen` **不可加 `--no-dev`**。只有 prod Dockerfile 才加。

### 6.4 branch 策略（必須遵守）

**Phase 0–3 全程在 feature branch 進行，只有 Phase 3 部署驗證通過後才 merge master。**

原因：master 上現有的 `.github/workflows/deploy.yml` 硬編路徑跑
`culture/tests.py`、`tech_stack/tests.py`。Phase 2 刪除這些 app 後若誤 merge 進 master，
任何後續 push（含緊急 hotfix）都會在 test 步驟失敗，`flyctl deploy` 永遠跑不到，
**prod 卡死在最後一個成功版本且無告警**。

若 prod 期間需要 hotfix：直接在 master 上改，事後 rebase 進 feature branch。

### 6.5 rollback 手續

Fly 的 rollback 是 `fly releases --image` 取得舊 image hash，
再 `fly deploy -i <sha>`。**但 rollback 不會還原 config** —
fly.toml / env / secrets 一律使用當前版本。

這代表：若已把 `internal_port` 從 8787 改成 8080，滾回舊 image（監聽 8787）
會配上新 fly.toml，**rollback 指令執行成功但服務仍然不通**。

必須做到：
- 改 Dockerfile / fly.toml **之前**，先 `fly releases --image` 記下當前 sha
- fly.toml 變更前打一個 git tag，確保知道要連 config 一起滾回

### 6.6 Phase 4：遷移 Cloud Run（另開 branch）

GCP 一次性 infra 用 **Terraform** 管理（owner 指定，作為 IaC 學習）：
enable APIs、deployer service account + IAM roles、Workload Identity Federation、
billing budget alert。App 部署不進 Terraform（CI 的 `gcloud run deploy` 負責）；
tfstate 存本機並 gitignore。

遷移順序：Cloud Run 上線並驗證數天後，才 `fly apps destroy` 並刪除 `fly.toml`，不空窗。

**Phase 0 的 blocking 前置**：owner 必須先查 fly.io dashboard 的 billing，
確認是否吃 grandfathered 免費額度（Fly 於 2024 年對新用戶取消免費方案，
但舊有 Hobby/Launch/Scale 用戶保留原額度）。若確認在扣錢，
Phase 4 必須設定明確 deadline（建議 Phase 3 上線後 30 天內），不可停留在「隨時做」。

## 7. 清理清單

刪除（Phase 2 執行）：

- `main_project/main_project/templates/backup.html`（425 行未使用）
- `main.py`（PyCharm 產生的 hello world）
- Material Dashboard 全部 static assets 與模板
- Select2 / FontAwesome / Google Fonts 等 CDN 依賴
- `tech_stack` app（內容併入前端 About 頁）
- `culture` app、`utility/`
- `requirements.txt`、Poetry 設定
- `deployment_tcei/`（舊 Dockerfile 與 docker-compose 所在目錄）

**保留（不可刪）**：

- `fly.toml` — Phase 3 仍要用它部署，Phase 4 遷移完成後才刪
- `docker-compose.dev.yml`、`frontend/Dockerfile.dev` — dev 環境，
  已於 Phase 1 建立在 repo root / frontend/，不在 `deployment_tcei/` 內
- `health_check`（改為 `backend/health/`）
- SSL workaround（封裝進 provider）

替換：

- **Logging 改用 owner 自有套件 `toolkitsy`**：
  `from toolkitsy.logger import logger, configure, set_correlation_id`。
  settings 啟動時 `configure()`（console only）；
  新增 CorrelationIdMiddleware 每個 request 設 correlation id + 回 `X-Request-ID` header，
  取代原 `utility/logger.py` + `culture/middleware.py`。
- HTTP 請求：toolkitsy 尚無 http 模組（PyPI 0.1.0 已驗證）。
  暫用 `requests` 並集中在 provider 檔案；toolkitsy 發版後單檔替換。

## 8. 測試策略

- **後端 pytest（完整）**：
  - providers：mock MoC（用 `responses`），涵蓋正常/空回應/非 JSON/HTTP 錯誤/timeout
  - services：cache hit/miss 行為、過濾與排序 edge cases（跨月、格式異常的時間）
  - API contract：status codes、錯誤格式、參數驗證
- **前端 Vitest（輕量）**：日期格式化、API 錯誤處理等純邏輯；不追 component 覆蓋率。
- **Phase 2 結束前的本機 prod-like container smoke test**：
  `docker build` + `docker run` + curl `/health`、`/`、`/api/v1/countries`。
  目的是把「目錄重構是否正確」與「prod 環境能否啟動」這兩個變數拆開驗證，
  不讓它們疊在 Phase 3 唯一一次真實部署裡。
- CI 三個 test job 都跑。

## 9. 里程碑

```
Phase 0 — 規劃
  spec v2 + plan v2 定稿
  POC HTML → owner 確認（gate）
  owner 查 fly.io billing（blocking）
  結束狀態：plan 定稿、視覺方向凍結、Phase 4 是否需要 deadline 已確定

Phase 1 — 開發（backend + frontend）
  uv 遷移 → dev compose → providers → services → API（★ checkpoint）
  → 前端 scaffold → api client → i18n → UI 元件 → App 組裝
  結束狀態：docker-compose 起得來，SPA 打新 API 全流程可用

Phase 2 — 清理與重構
  刪舊 apps/assets → backend/+config/ 重構 + 全新 settings
  → prod Dockerfile + 本機 prod-like container smoke test → repo 改名 + README 骨架
  （prod Dockerfile 歸屬 Phase 2 而非 Phase 3：§8 要求 smoke test 在 Phase 2 結束前完成，
    而 smoke test 需要它；它只依賴重構後的目錄與 frontend/dist，不依賴 Fly）
  結束狀態：codebase 乾淨，prod 仍是 Fly 上的舊版

Phase 3 — 上 Fly.io
  fly.toml + rollback 前置手續 → CI 重寫 → 部署驗證 + README Live URL
  結束狀態：新版在 Fly.io serve 真實流量，有 uptime 監控

Phase 4 — 遷移 Cloud Run（另開 branch，不 block）
  Terraform → owner 手動跑 gcp-setup → CI 換 gcloud run deploy
  → 驗證數天 → fly apps destroy + 刪 fly.toml

Phase 5 — k8s（另開 branch，隨時，純學習）
  kind + manifest 跑同一個 prod image
```

## 10. 風險與已知取捨

- **Fly 不注入 `$PORT`** — 見 §6.1。最高風險項，且因 scale-to-zero 會靜默失敗。
- **`ALLOWED_HOSTS` 平台網域不符** — 見 §6.1。上線當下全站 400。
- **rollback 不還原 config** — 見 §6.5。滾回舊 image 未必能恢復服務。
- **清理前移的代價**：Phase 3 首次真實部署時，工作樹已無舊結構可供 diff。
  緩解：Phase 2 結束前的本機 prod-like smoke test（§8）把變數拆開。
- **scale-to-zero + 無監控 = 靜默壞掉**：只做一次性人工驗證的話，
  問題要等朋友來用才會發現。緩解：Phase 3 加免費 uptime check 定期 ping `/health`。
- **誤 merge master 會卡死 CI** — 見 §6.4。
- **Fly 費用未確認** — 見 §6.6 的 Phase 0 blocking 前置。
- LocMemCache 不跨 instance、不跨 gunicorn worker、不耐重啟 — 已知，見 §3.3。
- MoC API 無 SLA、憑證有問題 — provider 層隔離，錯誤有明確 UX。
- **MoC 回應格式在開發期間可能改版**（政府 open data 常見）：測試用 `responses` mock
  固定的是撰寫當下的格式，mock 全綠不代表真實 API 沒變。緩解：Phase 1 的 checkpoint
  （Task 5）與 Phase 3 部署驗證（Task 17）各打一次真實 MoC 核對格式。
- Owner 首次寫 React/TS — code 難度刻意壓低，元件小而少（約 6 個），
  且有 §4.1 的 POC gate 先凍結視覺方向。
- `main_project/main_project/settings.py` 被本地 hook `protect_sensitive.py` 擋住 Read，
  Phase 1 期間只能透過 Bash python-snippet 修改。Phase 2 產出全新 settings 後此摩擦解除。
