# Culture Event Finder — React + Django API 重構設計

> ⚠️ **SUPERSEDED** — 已被 `2026-07-18-culture-event-finder-design-v2.md` 取代。
> **不得作為執行依據**，僅供追溯。
> 本文件的 hosting 決策（直上 Cloud Run）、清理清單（刪 fly.toml / docker-compose）、
> 本地開發流程（兩個裸 process）與里程碑順序皆已失效。

- 日期：2026-07-11
- 狀態：已與 owner 逐項確認並批准
- 前身：taiwan_culture_event_info_django_jinja2 (Django + Jinja2 server-rendered)

## 1. 背景與定位

現況是一個 Django server-rendered 網站，透過台灣文化部 (MoC) open data API 查詢藝文活動。
Owner 確認的產品定位：**自己 + 朋友真的在用的實用工具**，不是 portfolio 展示品、暫不需要 SEO。

本次重構的驅動需求 (皆已逐項確認)：

1. 前後端改為 **React (Vite) SPA + Django JSON API**，兼顧 owner 的 SDET 職涯發展 (內部測試平台多為 React)。
2. Hosting 改為 **Google Cloud Run 單一 container，$0** (always-free tier；Fly.io 已無免費方案)。
3. UI 全面重做：**mobile-first responsive、卡片式列表**，丟掉 Material Dashboard 後台模板。
4. **多國擴展架構**：先做骨架，只實作台灣；未來加國家不推翻重寫。
5. **UI 多語言 (zh/en)**：用免費方案 (locale JSON)，不用付費翻譯服務。
6. **Cache 機制**：cache-aside + TTL，為未來付費 API 預留，跨 user 共用。
7. Code 難度控制在 owner 可獨立維護的程度 (參考 shyin-coder 標準：entry-level SDET 看得懂)。

### 明確排除 (YAGNI)

- 搜尋條件進 URL (shareable URL) — 之後要加很容易
- 結果二次篩選/排序
- SEO / SSR
- 活動內容機器翻譯 (活動資料維持資料源語言)
- 第二個國家的實作 (只做架構)

## 2. Repo 結構 (monorepo)

Repo 改名為 `culture-event-finder` (GitHub 改名，舊名自動 redirect)。

```
culture-event-finder/
├── backend/                     # Django (由 main_project 改造)
│   ├── config/                  # settings / urls / wsgi
│   ├── events/                  # 唯一的業務 app
│   │   ├── providers/
│   │   │   ├── base.py          # Provider 抽象 interface + registry
│   │   │   └── taiwan.py        # MoC API 呼叫 + 台灣 locations/categories 設定
│   │   ├── services.py          # cache-aside + 過濾/排序 (純函式)
│   │   ├── views.py             # 薄：只做 HTTP in/out 與參數驗證
│   │   └── tests/
│   └── health/                  # health check endpoint (Cloud Run 用)
├── frontend/                    # Vite + React + TypeScript
│   └── src/
│       ├── components/          # SearchForm, EventList, EventCard,
│       │                        # ErrorMessage, SkeletonCard, LanguageSwitch
│       ├── api.ts               # 集中所有 fetch 呼叫
│       └── locales/             # zh.json / en.json
└── deployment/                  # Dockerfile (multi-stage)、docker-compose (dev)
    └── (GitHub Actions 在 .github/workflows/)
```

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
    → 200，Cloud Run health check
```

錯誤格式統一：`{ "error": { "code": "...", "message": "..." } }`

- 參數缺漏/非法 → 400
- 未支援的國家 → 404
- 上游 (MoC) 失敗或 timeout → 502，前端顯示「資料來源暫時無法使用」
- 「查無結果」是 200 + 空陣列，與錯誤明確區分

### 3.2 Provider 架構 (多國擴展點)

- `base.py` 定義抽象 interface：`fetch_events(category) -> list[Event]`、
  `locations`、`categories`、國家 metadata。
- `taiwan.py` 實作 MoC API。現有的 SSL `verify=False` workaround (政府憑證缺
  Subject Key Identifier，Python 3.13 拒連) 封裝在此檔內並附註解。
- Registry 用簡單 dict：`PROVIDERS = {"tw": TaiwanProvider()}`。
  未來加日本 = 新增 `japan.py` + registry 加一行。
- 現有 `data.py` 的 hardcoded Location/EventCategory 移入 `taiwan.py`
  (它們本來就是台灣專屬設定)。

### 3.3 Cache (cache-aside)

- 用 Django cache framework (`django.core.cache`)，業務 code 只碰 `cache.get/set`。
- Key：`events:{country}:{category}`；TTL：12 小時。
- **月份格式轉換 (必做)**：API 收 ISO `month=2026-07`，但 MoC 的 `show['time']`
  是 `YYYY/MM/DD HH:MM:SS`。services 層過濾前必須把 `2026-07` 轉成 `2026/07`
  再比對，否則永遠查無結果。此轉換屬台灣 provider 的資料格式知識，測試必須涵蓋。
- MoC API 只按 category 查詢 (location/月份是本地過濾)。**「每 12 小時最多打 12 次上游、
  跨 user 共用」是 best-effort 上界，不是硬保證**：LocMemCache 是 per-process 記憶體，
  每個 gunicorn worker 與每個 Cloud Run instance 各自持有獨立 cache，實際上游呼叫次數
  ≈ 12 × worker 數 × instance 數。為貼近此假設，prod 用 `--workers 1` + `--max-instances 1`
  (見 §5)。另外 cache-aside 的 check-then-set 無 lock，冷啟動瞬間並發仍可能對同一
  category 重複打上游 — 屬可接受取捨 (MoC 免費、不影響正確性)，故不加 lock 以維持 code 簡單。
- Backend 現階段用 LocMemCache。已知限制：Cloud Run scale-to-zero 時 cache
  消失 — 可接受 (MoC 免費，重打不痛)。未來接付費 API 時僅改 settings 換
  持久 backend (如 Neon/Supabase 免費 Postgres 做 DatabaseCache)，業務 code 不動。

## 4. 前端設計

- **Vite + React + TypeScript**：owner 有 Playwright (TS 生態) 底子；型別寫到夠用，
  不用進階泛型、不用 Redux、不用 server components。
- **Tailwind CSS**：mobile-first utility。
- **i18n**：UI 文案走 `locales/zh.json` / `en.json` + 一個輕量 context/hook
  (不引重型 i18n 套件)，右上角切換，預設中文。活動資料維持資料源語言。
- **不用 react-router**：只有搜尋頁和 About 兩個畫面，用 in-app state 切換。
  這同時避開 SPA fallback routing 問題 (WhiteNoise 不會把未知路徑 catch-all
  到 index.html，BrowserRouter 重新整理會 404)。
- 頁面結構：
  - 搜尋列：國家 (只有台灣時隱藏)、地區、類別、月份選擇器 (年 + 月兩個 `<select>`；
    刻意不用 `<input type="month">` — 桌面版 Firefox 與 Safari 全版本不支援，會 fallback
    成純文字框，使用者得自己猜格式輸入，反而更容易打錯)、搜尋鈕
  - 結果：卡片式列表 — 活動名稱、時間、地點 (點擊開 Google Map)、票價、
    售票狀態 badge；手機單欄、桌機多欄 grid
  - Loading：skeleton 卡片
  - 空結果與錯誤是兩種不同的畫面
  - About 頁：合併原 tech_stack 頁內容 (tech stack 表 + 作者連結)
- 視覺方向：乾淨留白、清楚的層級，像簡化版 KKTIX 列表頁。

## 5. 部署與 CI/CD

- **Multi-stage Dockerfile**：
  stage 1 (node) `vite build` → stage 2 (python slim) Django + gunicorn，
  WhiteNoise 服務 React build 產物 + `/api` JSON。單一 container、單一網址、無 CORS。
  - 新增依賴：`gunicorn`、`whitenoise` (現有 requirements 皆無)。
  - CMD 用 `gunicorn config.wsgi --bind 0.0.0.0:$PORT` — Cloud Run 注入
    `$PORT` (預設 8080)，不可 hardcode 現在的 8787。
  - Python stage 需跑 `collectstatic`；`STATICFILES_DIRS` 指向 `frontend/dist`。
  - WhiteNoise 用 plain storage (非 manifest storage)：Vite 已對檔名做
    content-hash，manifest storage 會重複 hash 且可能 500。
  - `ALLOWED_HOSTS` 需含 `*.run.app` (用環境變數設定)，否則 Django 回 400。
- **依賴管理單一來源：uv** (owner 指定)。`pyproject.toml` (PEP 621) + `uv.lock`
  是 source of truth；Dockerfile 用 `uv sync --frozen --no-dev`。
  刪除 `requirements.txt` 與 Poetry 設定，消除 split-brain。
- **本地開發流程**：dev 是兩個 process — Vite dev server (HMR) 用
  `server.proxy` 把 `/api` 轉發到 Django `:8000`；單一 container 只在 prod。
  makefile targets 隨之改寫 (`run-dev` 起兩個 process)。
- **Cloud Run**：scale-to-zero、min-instances 0、**max-instances 1** (低流量足夠，且讓
  §3.3 的「跨 user 共用 cache」假設成立)，落在 always-free 額度內 ($0)。公開端點 (`--allow-unauthenticated`)
  加一個 GCP budget alert (寫進 Terraform)，避免被爬蟲打爆超出免費額度而不自知。
- **GitHub Actions** (push master)：pytest + vitest → **docker build + 合體 container 煙測**
  (curl `/health`、`/`、`/api/v1/countries`，確認 collectstatic + WhiteNoise 服務 SPA 與 `/api`
  真的一起起得來 — 這條路徑 test-backend/test-frontend 都不會跑到) →
  push Artifact Registry → `gcloud run deploy`。紅燈不部署。
  現有 deploy.yml 整份重寫 (它跑的 per-app pytest 與 `make run-prod` 都會失效)。
- GCP 一次性 infra 用 **Terraform** 管理 (owner 指定，作為 IaC 學習)：
  enable APIs、deployer service account + IAM roles、Workload Identity
  Federation。App 部署不進 Terraform (CI 的 `gcloud run deploy` 負責)；
  tfstate 存本機並 gitignore。手動步驟只剩：開專案、綁 billing、
  填 GitHub Actions variables、首次部署驗證 (寫成 checklist)。
- **遷移順序**：Cloud Run 上線並驗證後，才下線 Fly.io app，不空窗。

## 6. 清理清單

刪除：

- `main_project/main_project/templates/backup.html` (425 行未使用)
- `main.py` (PyCharm 產生的 hello world)
- Material Dashboard 全部 static assets 與模板 (base.html 的 sidenav、
  breadcrumb、fixed-plugin configurator)
- Select2 / FontAwesome / Google Fonts 等 CDN 依賴
- `tech_stack` app (內容併入前端 About 頁)
- `requirements.txt` (uv 為單一依賴來源，見 §5；Poetry 設定亦一併刪除)
- Fly.io 相關設定 (fly.toml、deploy.yml 的 Fly 步驟、docker-compose-prod) — 於遷移完成後

保留：

- `health_check` (改為 `health/`，Cloud Run 需要)
- SSL workaround (封裝進 provider)

替換：

- **Logging 改用 owner 自有套件 `toolkitsy` (PyPI 0.1.0)**：
  `from toolkitsy.logger import logger, configure, set_correlation_id`。
  settings 啟動時 `configure()` (console only，Cloud Run 收 stdout)；
  新增 CorrelationIdMiddleware 每個 request 設 correlation id +
  回 `X-Request-ID` header，取代原 `utility/logger.py` + `culture/middleware.py`。
  `utility/` 於 M4 隨舊 apps 一併刪除。
- HTTP 請求：toolkitsy 尚無 http 模組 (owner 計畫在 toolkitsy repo 開發)。
  本重構不等它：暫用 `requests` 並集中在 provider 檔案；toolkitsy 發版後
  單檔替換 (若執行時已發版，Task 2 直接採用)。

## 7. 測試策略

- **後端 pytest (完整)**：
  - providers：mock MoC (用 `responses`)，涵蓋正常/空回應/非 JSON/HTTP 錯誤/timeout
  - services：cache hit/miss 行為、過濾與排序 edge cases (跨月、格式異常的時間)
  - API contract：status codes、錯誤格式、參數驗證
- **前端 Vitest (輕量)**：日期格式化、API 錯誤處理等純邏輯；不追 component 覆蓋率。
- CI 兩邊都跑。

## 8. 里程碑

每個里程碑結束都是可部署狀態：

1. **後端重構** — providers/services/API + 完整測試；舊 Jinja2 頁面暫時共存
   (共存期間不得移除 tech_stack 模板與 handler404 依賴的模板，避免舊頁面壞掉)
2. **React 前端** — Vite + TS + Tailwind，打新 API
3. **合體部署** — multi-stage Dockerfile、Cloud Run 上線、CI/CD 切換、Fly.io 下線
4. **收尾** — 清理清單執行、repo 改名、README 重寫

## 9. 風險與已知取捨

- Cloud Run 冷啟動 2-10 秒 (低流量下常見) — 可接受，前端有 loading 回饋。
  注意冷啟動與 LocMemCache 清空同時發生：每次閒置後的第一次搜尋 = 冷啟動 +
  cache miss。若實際體感太差，再考慮 min-instances=1 (會開始產生小額費用)。
- LocMemCache 不跨 instance、不跨 gunicorn worker、不耐重啟 — 已知，見 §3.3 升級路徑。
- MoC API 無 SLA、憑證有問題 — provider 層隔離，錯誤有明確 UX。
- Owner 首次寫 React/TS — code 難度刻意壓低，元件小而少 (約 6 個)。
