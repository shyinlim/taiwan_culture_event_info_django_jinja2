# Culture Event Finder：React + Django API 重構設計 v4

- 日期：2026-08-22
- 狀態：已與 owner 逐項確認並批准
- 取代：`2026-07-19-culture-event-finder-design-v3.md`（v3 標記 SUPERSEDED）
- 前身：`taiwan_culture_event_info_django_jinja2`（Django + Jinja2 server-rendered）

> 本文件自足。執行時不需開啟 v1 / v2 / v3。

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

- 搜尋條件進 URL (shareable URL)：之後要加很容易
- 結果二次篩選/排序
- SEO / SSR
- 活動內容機器翻譯（活動資料維持資料源語言）
- 第二個國家的實作（只做架構）
- **k8s 不進主線**：Fly.io 走 `fly.toml` + Dockerfile → Firecracker microVM，
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
│   ├── health/                  # health check endpoints
│   └── Dockerfile.dev           # dev 用 python container
├── frontend/                    # Vite + React + TypeScript
│   ├── Dockerfile.dev           # dev 用 node container
│   └── src/
│       ├── components/          # SearchForm, EventList, EventCard, ErrorMessage,
│       │                        # SkeletonCard, LanguageSwitch, Icon
│       ├── design.css           # 毛玻璃 design system（CSS variables + .glass 等）
│       ├── api.ts               # 集中所有 fetch 呼叫
│       ├── utils/               # format.ts（純函式，可單測）
│       └── locales/             # zh.json / en.json
├── docs/poc/                    # POC HTML 迭代紀錄（design reference，唯讀）
├── Dockerfile                   # prod multi-stage（repo root）
├── docker-compose.dev.yml       # dev 環境（repo root）
└── fly.toml
```

**`docker-compose.dev.yml` 與兩個 `Dockerfile.dev` 放在 repo root / `backend/` / `frontend/`**，
不放 `deployment_tcei/`：該目錄會被整個刪除。

分層原則：views (HTTP) → services (業務邏輯 + cache) → providers (外部資料源)。
每層單獨可測、單獨可替換。**不再加第四層**：不引 DRF、不引 serializer、不引 repository。
`_event_to_json` 用十行手寫 dict 就是正確答案。

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
    → 200，平台 health check。不碰任何外部依賴，只證明 process 活著。

GET /health/upstream
    → 200 {"upstream": "ok"} 或 503 {"upstream": "failing", "since": <ts>}
      監控用。讀 cache 裡的上游錯誤旗標，本身不打 MoC。
```

錯誤格式統一：`{ "error": { "code": "...", "message": "..." } }`

- 參數缺漏/非法 → 400
- 未支援的國家 → 404
- 上游 (MoC) 失敗或 timeout → 502，前端顯示「資料來源暫時無法使用」
- 「查無結果」是 200 + 空陣列，與錯誤明確區分

**參數一律對照 provider 白名單驗證。** `category` 必須在 `provider.categories` 的 value
集合內，`location` 必須在 `provider.locations` 的 value 集合內，否則 400。
只檢查 `isdigit()` 是不夠的：`category=999999` 會實際打上游並佔用一個 cache entry，
連續丟不同 category 可以把 LocMemCache 預設的 300 個 entry 上限洗掉，
之後每個真實查詢都變成 cache miss。

**404 的判定放在 view 層**，用 `PROVIDERS.get(country)` 直接判斷。
services 不 raise `KeyError`，view 的 try 區塊只留 `except UpstreamError`。
若讓 `except KeyError` 包住整個 service 呼叫，內部任何深層 `KeyError`
都會變成假的「不支援這個國家」，排查方向會被完全帶偏。

### 3.2 Provider 架構（多國擴展點）

- `base.py` 定義抽象 interface：`fetch_events(category) -> list[Event]`、
  `locations`、`categories`、國家 metadata。加上 `Event` dataclass 與 `UpstreamError`。
- `taiwan.py` 實作 MoC API。SSL `verify=False` workaround（`cloud.culture.tw` 憑證缺
  Subject Key Identifier，Python 3.13 拒連）封裝在此檔內並附註解。
- Registry 用簡單 dict：`PROVIDERS = {"tw": TaiwanProvider()}`。
- 現有 `data.py` 的 hardcoded Location/EventCategory 移入 `taiwan.py`。

**界線：不要再加 provider factory、不要加 `providers/registry.py`、
不要讓 `get_provider` 做 fallback。** 這個 ABC 只有一個實作，它換到的是
README「Adding a country」那幾步真的成立，成本只有一個 `@abstractmethod`。

#### 台灣的 locations 必須是 22 個縣市

實測 MoC 資料（category 1/6/17 共 1320 筆 showInfo）：`宜蘭縣` 有 9 筆，
但既有 `data.py` 的清單裡沒有宜蘭，使用者永遠選不到。連江同樣缺漏。
清單必須補齊為 22 個縣市。

#### 地名比對前必須做異體字正規化

實測同一批資料：`台北市` 有 11 筆，其餘寫 `臺北市`。
`location in e.location` 是字面 substring 比對，使用者選「臺北」時那 11 筆被靜默丟掉。
過濾前把查詢值與資料值都做一次 `replace("台", "臺")` 再比對。
使用者看到的失敗是「查無結果」，與真的沒活動長得一模一樣，是最難被回報的一種 bug。

### 3.3 Cache (cache-aside)

- 用 Django cache framework (`django.core.cache`)，業務 code 只碰 `cache.get/set`。
- Key：`events:{country}:{category}`；TTL：12 小時 (`43200` 秒)。
- MoC API 只按 category 查詢（location/月份是本地過濾）。「每 12 小時最多打 12 次上游、
  跨 user 共用」是 best-effort 上界，不是硬保證：LocMemCache 是 per-process 記憶體。
  故 prod 用 `--workers 1`；Fly 側 `min_machines_running = 0` 且維持單 machine。
- **`--workers 1` 必須搭配 `--threads 8 --worker-class gthread`。**
  gunicorn 預設是 sync worker，一個 worker 一次只吃一個 request。cache miss 時
  `requests.get(timeout=15)` 會佔住整個 process，這段期間靜態檔、`/health`、
  其他人的搜尋全部排隊；排到超過 gunicorn 的 request timeout 時 worker 被 kill，
  LocMemCache 整個歸零，本節的上界假設就在流量最高的時候破功。
  gthread 仍是單一 process，LocMemCache 只有一份，前提不變。
  `--timeout` 要設得比上游的 15 秒大（用 60）。
- cache-aside 的 check-then-set 無 lock，冷啟動瞬間並發仍可能對同一 category 重複打上游：屬可接受取捨（MoC 免費、不影響正確性），不加 lock 以維持 code 簡單。
- 已知限制：scale-to-zero 時 cache 消失：可接受。未來接付費 API 時僅改 settings
  換持久 backend，業務 code 不動。
- **Cloud Run 上此上界僅在 `--max-instances=1` 時成立**（見 §6.6）。

#### 月份過濾必須用區間重疊判斷

API 收 ISO `month=2026-07`，MoC 的 `show['time']` 是 `YYYY/MM/DD HH:MM:SS`，
兩者格式不同，比對前必須轉換。**但只轉換格式是不夠的。**

實測 MoC `category=6`（展覽）：439 筆 showInfo 裡 **386 筆跨月**（88%），
`endTime` 一筆都沒缺，最長的是 `2026/01/01 → 2026/12/31` 的常設展。
若只比對 `start_time` 的年月，這類活動在開幕月之後的每一個月都查不到，
等於展覽這個類別整個是壞的。

正確條件是區間重疊：活動的 `[start, end or start]` 與查詢月的 `[月初, 下月初)`
有交集就算命中。

測試必須涵蓋：一個 1 月開跑、12 月結束的活動，查 9 月要查得到。

### 3.4 Observability

- **Logging 用 owner 自有套件 `toolkitsy`**：
  `from toolkitsy.logger import logger, configure, set_correlation_id`。
  settings 啟動時 `configure()`（console only）；
  CorrelationIdMiddleware 每個 request 設 correlation id + 回 `X-Request-ID` header。
- **上游健康旗標**：provider 拋 `UpstreamError` 時，services 寫
  `cache.set("last_upstream_error", timestamp, 3600)`；成功時 `cache.delete`。
  `/health/upstream` 讀這個旗標。監控指向這個 endpoint，不是 `/health`。
- **格式漂移的 sanity 訊號**：一次 fetch 若 raw payload 非空但 parsed events 為 0，
  `logger.error` 並打同一個旗標。MoC 改欄位名時 HTTP 仍是 200、mock 仍全綠、
  uptime 仍全綠，這個訊號是唯一會亮的東西。
- **已知限制**：`fly logs` 只有即時串流，machine 是 scale-to-zero，
  事後拿到 `X-Request-ID` 也還原不了當時的 log。README 要寫明這一點，
  不要讓那個 header 看起來像可以追查的東西。

## 4. 前端設計

- **Vite + React + TypeScript**：型別寫到夠用，不用進階泛型、不用 Redux、不用 server components。
- **Tailwind CSS**：mobile-first utility，搭配 `design.css` 的 CSS variables（見 §4.1）。
- **i18n**：UI 文案走 `locales/zh.json` / `en.json` + 一個輕量 context/hook
  （不引重型 i18n 套件），預設中文。活動資料維持資料源語言。
  切換語言時同步設定 `document.documentElement.lang`（`zh-Hant` / `en`）。
- **不用 react-router**：只有搜尋頁和 About 兩個畫面，用 in-app state 切換。
  但 `config/urls.py` 仍要有一條 catch-all 把非 `/api`、非 `/static`、非 `/health`
  的路徑導回 `index.html`，否則打錯字的網址會拿到 Django 的裸 404 純文字頁，
  使用者會以為站掛了。
- 頁面結構（依定案 POC `docs/poc/20260719_155200_ui_design_v27.html`）：
  - **背景場景**：深色 radial-gradient 底 + 抽象曲線 SVG 線稿（低透明度、
    bronze/cyan 漸層描邊）+ 雙色燈光 (`--lamp-1` 暖銅、`--lamp-2` 冷青) +
    兩顆跨面板光暈 (bronze/cyan)，是毛玻璃 blur 的視覺素材
  - **導覽**：桌機左側毛玻璃 icon rail（搜尋/關於/主題切換/語言切換）；
    手機隱藏 rail，改頂部毛玻璃 bar。**手機 bar 必須含語言切換**，
    四顆 `h-9 w-9` 按鈕在 375px 放得下（標題 `flex-1` 會自動縮）
  - **國家選擇器**：搜尋卡標題列右上角，膠囊容器內：台灣是 active
    (`.btn-primary`) chip；日本/韓國是 `opacity-60` chip（不帶「即將推出」文字，
    純用視覺降權表示未開放）。用 `aria-disabled` + onClick 攔截取代 `disabled`，
    並補 `title` / `aria-label`，讓鍵盤與螢幕閱讀器接收得到「未開放」這個訊息
  - **搜尋列**：Airbnb 風格「搜尋膠囊」(`.search-capsule`)：地區、類別、年、月四個
    `search-field`（label 在上、`<select>` 在下，欄位間以 `border-right`/`border-bottom`
    分隔），尾端一個圓形/膠囊搜尋鈕。月份維持年 + 月兩個 `<select>`；刻意不用
    `<input type="month">`：桌面版 Firefox 與 Safari 全版本不支援，會 fallback 成純文字框
  - **類別 chips**：`地區/類別/年/月` 之外另有一列**四個**帶 icon 的類別快捷 chip
    （展覽/表演/音樂/市集，各配一個 inline SVG icon：frame/masks/music/tent），
    數量對齊 POC，不渲染全部 12 個類別（12 個 chip 在手機上會換三行、佔滿第一屏）。
    chip 與類別 `<select>` 是同一個 `value.category` 的兩個入口：
    select 是慢速精確設定，**chip 點下去直接觸發搜尋**（快捷就要一步到位，
    只改 state 不重搜會讓使用者以為篩選壞了）。兩者都要 `aria-pressed`
  - **結果**：卡片式列表：漸層 banner + 白色線條幾何裝飾（三組輪流，hover 時
    banner SVG 有 1.5s 慢速 zoom 動效）、活動名稱（hover 變 accent 色）、時間、
    地點（點擊開 Google Map，新分頁）、卡片底部一條分隔線後放票價 + 一個連往
    Google 搜尋的按鈕；手機單欄、桌機三欄 grid。狀態 badge 文案含裝飾性 emoji
    （「🔥 熱賣中」，owner 已於 2026-08-22 確認保留）
  - **結果計數列顯示完整條件**（「臺北 · 音樂 · 2026/07 共 12 筆」）。
    預設類別是 provider 的第一個類別而不是「全部」，MoC 也沒有 all-category 查詢能力，
    條件不顯性寫出來的話使用者會誤以為看到的是全類別
  - **Loading**：skeleton 卡片（`--skel` 變數，雙主題各自可見，含分隔線 + 票價列 skeleton）
  - **四種狀態都要有畫面**：`idle` / `loading` / `empty` / `error`。
    `idle` 不可以是一片空白，否則使用者一進站看不出要按搜尋鈕，
    也分不出「還沒搜」與「查無結果」。空結果與錯誤各配一個大型線條 SVG 插圖，
    外層加虛線邊框容器 (`border-dashed`) 與 `--surface-2` 底色；錯誤畫面附重試按鈕
  - **About 頁**：合併原 tech_stack 頁內容（tech stack 表 + 作者連結），同樣走毛玻璃面板
- **視覺方向（已凍結，POC v27）**：毛玻璃 (glassmorphism)。深色抽象背景
  (`radial-gradient(circle at 80% 20%, #1e1812 0%, #05070f 65%)`)；
  accent 是單一 copper/ochre 色系 `#B57004`（`--accent-cool: #7a4700` →
  `--accent: #B57004` 漸層）；**不用粉紅/magenta**；
  dark/light 雙主題由 CSS variables（`[data-theme]`）驅動，兩個主題都必須是
  真正透亮的毛玻璃，不是換色而已。

### 4.1 視覺 source of truth

**唯一的視覺 source of truth 是 `docs/poc/20260719_155200_ui_design_v27.html`。**
POC gate 於 2026-07-19 通過（owner 迭代 27 版後口頭定案，未另交付截圖）。
POC 保留在 `docs/poc/` 作 design reference，不是丟棄式產物。

寫 React 元件時對照該檔抄 CSS 與 Tailwind class 組合：

- `<style>` 區塊整段抄成 `frontend/src/design.css`（CSS variables、`.scene`/`.glass`/
  `.search-capsule`/`.btn-primary`/`.btn-secondary` 等）
- inline SVG icon 抄成 `Icon.tsx` 元件，不引 icon library。
  **只抄實際會用到的八個**：search / info / moon / sun / calendar / pin / ticket / refresh，
  加上四個類別 chip 用的 music / tent / masks / frame，共 12 個。
  `alert` 不需要（ErrorMessage 用自己的 inline path）

毛玻璃四要件（POC 迭代驗證出的經驗值，改 CSS 時不可破壞）：

1. 玻璃後方要有結構化視覺素材（v27 是抽象曲線 SVG + 雙色燈光）可供 blur 扭曲
2. 面板填色極低不透明度（dark 2%／light 55%）、v27 blur 為 **36 到 40px**、
   `brightness(>1)` 讓面板比周圍亮
3. `::after` 斜向 sheen 高光，light/dark 各自獨立調校
4. 彩色光暈要**跨越玻璃面板邊界**（外側銳利、內側模糊）

### 4.2 前端的三個隱含前提

這三項不寫下來的話，實作時會踩到而且不容易連結到原因：

1. **時間一律用本地時區。** 後端整套釘死 `Asia/Taipei`（`TIME_ZONE`、`ENV TZ`、
   `USE_TZ=False`）。前端取預設月份**不可以用 `toISOString()`**，那是 UTC，
   台灣時間每月 1 號 00:00 到 08:00 之間會抓成上個月，而且極難重現。
   用 `getFullYear()` + `getMonth()+1` 組字串。
2. **`tsconfig.app.json` 要開 `resolveJsonModule`。** Vite 的 react-ts template
   預設沒開，`import zh from "./locales/zh.json"` 會讓 `tsc --noEmit` 直接失敗。
3. **Vite proxy 不設 `changeOrigin`。** 預設 `false` 時轉發保留 Host
   `127.0.0.1:5173`，Django 去掉 port 後剛好落在 `ALLOWED_HOSTS` 裡。
   哪天有人加上 `changeOrigin: true`，Host 變成 `backend`，所有 `/api` 立刻 400，
   而且只在 dev 出現、看起來像後端壞了。dev compose 的 backend 同時把
   `backend` 加進 `ALLOWED_HOSTS`，兩種寫法都涵蓋。

### 4.3 可測性與 a11y 的最低要求

- **有分支的邏輯抽成純函式放 `utils/`**，配 vitest。目前有兩處：
  年月字串切片重組、切國家時重設 location/category。元件本身不寫 render test。
- **`data-testid` 只加在四個關鍵節點**：搜尋鈕、類別 chip 列容器、結果 grid、
  四種狀態容器。其餘不鋪。不加的話可用的 selector 只剩 i18n 文字
  （切 EN 就全掛）或 Tailwind 動態 class（chip active 會整串換掉）。
- **focus 樣式不可以只有 `outline: none`。** 必須補
  `:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; }`，
  否則鍵盤 Tab 時完全看不到焦點在哪。
- **`appearance: none` 拿掉原生下拉箭頭後必須補 chevron**（`::after` 純 CSS 即可），
  否則四個欄位在視覺上是純文字，看不出可以點。
- **結果區要有 live region**：EventList 與空結果容器加 `role="status"`，
  loading 的 grid 加 `aria-busy="true"`。否則螢幕閱讀器使用者按下搜尋後
  沒有任何播報，會以為按鈕沒作用。
- **主題要持久化**：`useState` 用 lazy initializer 讀 `localStorage`，
  讀不到就看 `prefers-color-scheme`；effect 裡寫回 `localStorage`。
  同時在 `index.html` 的 `<head>` 放三行 inline script 先設好 `data-theme`，
  這是避免 FOUC 唯一可靠的做法。
- **重試按鈕要接對對象**：初次載入 `/countries` 失敗時 `form.country` 是空字串，
  若重試接的是搜尋，會打 `/api/v1//events` 而永遠失敗，使用者按幾次都一樣、
  下拉選單全空無法自救。錯誤來源要分開記，重試依來源決定重載 countries 或重搜。
- **countries 載入完成前送出鈕要 disabled**，否則網路慢時先按下去會打出雙斜線 URL。

## 5. 開發環境

### 5.1 dev 走 docker-compose（兩個 service）

採市面主流配方，讓任何人 clone 下來跑 `make dev` 就有一致的環境：

- `docker-compose.dev.yml`（repo root）起兩個 service：
  - `backend`：Django `runserver 0.0.0.0:8000`
  - `frontend`：Vite dev server，port 5173，`server.proxy` 把 `/api` 轉發到 `backend:8000`
- **backend 的 venv 必須放在 bind mount 之外。** backend 掛 `.:/web`，
  若 venv 建在 `/web/.venv`，host 的 macOS arm64 venv 會覆蓋 container 內的 linux venv，
  `uv run` 拿到錯的 python，或就地重建 venv 把 linux binary 寫回 host repo，
  反過來弄壞 host 的 IDE。`Dockerfile.dev` 設
  `ENV UV_PROJECT_ENVIRONMENT=/opt/venv` 與 `ENV PATH="/opt/venv/bin:$PATH"`。
- **bind mount 原始碼，named volume 隔離 `node_modules`**：若讓 host (mac ARM) 的
  `node_modules` 蓋掉 container (linux) 的，esbuild 等原生依賴會直接崩潰。
- **`server.host: '0.0.0.0'`**：否則 host 瀏覽器連不進 container 內的 Vite。
- **`CHOKIDAR_USEPOLLING=true`**：macOS/Windows 的 docker file-watching 事件不可靠，
  不開 polling 則 HMR 不會觸發。Vite 把 chokidar 3.6.0 bundle 進 dist，
  該版本確實讀這個環境變數，寫法有效。加 `CHOKIDAR_INTERVAL=1000` 降低 CPU 空轉。
- container 內跑 non-root user。
- Dockerfile 先 `COPY package.json package-lock.json`，再 `COPY` 其餘：吃 layer cache。

**已知取捨**：proxy 指向 Docker DNS 名稱 `backend:8000`，所以只能走 `make dev`，
不能在 host 上單獨 `npm run dev`（那樣 `/api` 代理不到）。

### 5.2 host 也要裝一份 node_modules（給 IDE 用）

因為 `node_modules` 被 named volume 隔離在 container 內，host 上不存在，
編輯器的 TS server / eslint / import 跳轉會全部失效。

解法（市面標準做法）：host 另跑一次 `npm ci`。

- host 那份**只給編輯器讀**，container 那份才是實際執行的
- 兩份吃同一個 `package-lock.json`，不會漂移
- makefile 提供 `make install-host` target，並在註解說明為何要裝兩份

### 5.3 named volume 會 stale，要有 reset 出口

`frontend_node_modules` 這顆 named volume 只在第一次建立時從 image 複製內容。
之後 `package.json` 改了、image 重 build 了，volume 裡仍是舊的，
`make dev` 加 `--build` 也救不回來。症狀是「image 裡有這個套件、container 裡沒有」。

makefile 提供 `make dev-reset`（`down -v` 後重建）。裝新套件之後跑它。

### 5.4 dev 常用指令

由 makefile 提供：`make dev`（起 compose）、`make dev-reset`、`make install-host`、
`make test`、`make run-prod`（本機 build & run production container）。

`make run-prod` 依賴 repo root 的 Dockerfile，那是後面的 task 才產出的檔案；
makefile 定稿時要在該 target 註解說明，避免執行者撞牆後懷疑是自己漏做。

## 6. 部署

### 6.1 Phase 3 目標平台：Fly.io

- **Multi-stage Dockerfile**（repo root）：
  stage 1 (`node:22-slim`) `vite build` → stage 2 (`python:3.13-slim`) Django + gunicorn，
  WhiteNoise 服務 React build 產物 + `/api` JSON。單一 container、單一網址、無 CORS。
- 新增依賴：`gunicorn`、`whitenoise`。
- Python stage 需跑 `collectstatic`；`STATICFILES_DIRS` 指向 `frontend/dist`。
- WhiteNoise 用 plain storage（非 manifest storage）：Vite 已對檔名做 content-hash，
  manifest storage 會重複 hash 且可能 500。
- prod stage 也要跑 non-root user。dev container 有做而 prod 沒做是反過來的。
- CMD：`gunicorn --chdir backend config.wsgi:application --bind 0.0.0.0:${PORT:-8080}
  --workers 1 --threads 8 --worker-class gthread --timeout 60`

#### ⚠️ build 期的 SECRET_KEY

**這是整條部署鏈最早的斷點。**

settings 的 fail-fast 守衛（見下）在 `collectstatic` 執行時就會觸發：
build 環境沒有 `DEBUG` 也沒有 `SECRET_KEY`，於是直接 `ImproperlyConfigured`，
`docker build` 失敗。連帶本機 smoke test、CI 的 build-smoke job、
`make run-prod`、Fly 的 remote build 全部起不來。

必須做到：collectstatic 那一行帶 build-only 假值。

```dockerfile
RUN SECRET_KEY=build-only-not-used python backend/manage.py collectstatic --noinput
```

**不可以用 `ENV SECRET_KEY=`**，那會讓假值留在 image 裡變成 runtime 預設，
等於廢掉整個守衛。**不可以用 `DEBUG=True` 繞過**，那會把 debug 帶進 prod image。

同樣的道理，`make test` 與 CI 的 `test-backend` 也要注入假 SECRET_KEY，
否則本機測試與 CI 一起掛。

#### ⚠️ PORT：Fly 不注入 `$PORT`

Cloud Run 會自動注入 `$PORT`；**Fly.io 不會**：Fly 是靠 `fly.toml` 的
`[http_service].internal_port` 指定容器監聽哪個 port。

若 Dockerfile 用 `${PORT:-8080}` 而 `fly.toml` 仍是現有的 `internal_port = 8787`，
容器會監聽 8080、Fly proxy 打 8787 → 兩邊對不上。
搭配 `min_machines_running = 0`（scale-to-zero），**壞掉不會立刻被發現**。

必須做到：
- Dockerfile 加 `ENV PORT=8080`
- `fly.toml` 的 `internal_port` 改為 `8080`
- 部署驗證用 `curl`，不可只用瀏覽器（瀏覽器可能吃到快取而誤判成功）

#### ⚠️ fly.toml 必須定義 HTTP health check

`[http_service]` 不會自動生出 HTTP health check，要自己寫 `[[http_service.checks]]`。
沒有它的話，port 對不齊時 `fly deploy` 會**回報成功**、machine 起得來、proxy 打不到，
然後靜默壞掉。(推論：Fly 在無 check 時是否仍做 TCP 層等待，未查到官方明文)

check 要**同時**帶 Host header，否則會被 `ALLOWED_HOSTS` 擋成 400，
變成「加了 check 反而部署失敗」：

```toml
[[http_service.checks]]
  grace_period = "10s"
  interval = "30s"
  method = "get"
  path = "/health"
  timeout = "5s"
  [http_service.checks.headers]
    Host = "taiwan-culture-event-info.fly.dev"
```

#### ⚠️ SECRET_KEY 與 ALLOWED_HOSTS 都要真的 fail-fast

設計哲學是「設錯要立刻暴露，不准安靜地照常運作」。兩個變數都要做到，
**不可以只有註解宣稱**：

- `SECRET_KEY`：非 dev 且未注入 → `ImproperlyConfigured`。
- `ALLOWED_HOSTS`：非 dev 且環境變數不存在 → `ImproperlyConfigured`，**不留 fallback**。
  留 fallback 的話漏注入時 Django 照常啟動、Fly 的 check 照樣綠燈，
  只有真實使用者拿到 400 DisallowedHost。

dev 的判定用顯式訊號（`DJANGO_ENV` 或 `DEBUG=True`），不要用「沒設就當 prod」，
那會讓 build 與測試一起被守衛擋下。

prod 網域一律由平台的環境變數注入（Phase 3 是 `fly.toml` 的 `[env]`，
Phase 4 換成 Cloud Run），fallback 只涵蓋 local dev，刻意不含 `.fly.dev`。

### 6.2 CI/CD

**GitHub Actions**（push master）四個 job：

1. `test-backend`：pytest（要注入假 SECRET_KEY）
2. `test-frontend`：vitest
3. `build-smoke`：真的 `docker build` 起 container，curl `/health`、`/`、
   `/api/v1/countries`，並且 **grep events endpoint 的欄位名**（`startTime`），
   確認前後端 contract 沒有各自漂移。失敗時要 `docker logs`（`if: failure()`），
   否則只看得到 curl 的非零離開碼，分不出是啟動失敗還是路由不對。
4. `deploy`：`needs: [test-backend, test-frontend, build-smoke]`，
   `if: github.ref == 'refs/heads/master'`，用 `flyctl deploy` + `secrets.FLY_API_TOKEN`

紅燈不部署。**不需要** `permissions.id-token: write`（那是 Workload Identity Federation
用的，Phase 4 才需要）。

`astral-sh/setup-uv` 用 `v10`。

**repo 維持 public。** build-smoke 每次 PR 與 push 都跑一次完整 multi-stage build，
public repo 的 Actions 分鐘數不計費；轉 private 的話免費額度是 2000 分鐘/月，
密集開發期會在月中耗盡，deploy job 排不進去。改名 repo 時不要動 visibility。

### 6.3 依賴管理單一來源：uv

`pyproject.toml` (PEP 621) + `uv.lock` 是 source of truth；
Dockerfile 用 `uv sync --frozen --no-dev`。
刪除 `requirements.txt` 與 Poetry 設定，消除 split-brain。

- **`pyproject.toml` 必須真的沒有 `[build-system]` 與 `[tool.poetry]`。**
  留著的話 uv 會判定這是要 build 的 package，而 Dockerfile 只 COPY 了
  `pyproject.toml` 與 `uv.lock`（沒有 source、沒有 README），build 會失敗。
  遷移後要有一個機械可判的驗收：`grep -q "build-system\|tool.poetry" pyproject.toml`
  應為無命中。
- **dev 用的 container 需要 pytest 等 dev dependencies，`uv sync --frozen`
  不可加 `--no-dev`。** 只有 prod Dockerfile 才加。
- uv binary 釘特定版號（`ghcr.io/astral-sh/uv:0.12.5`），勿用 `:latest`。
  三處（backend dev Dockerfile、prod Dockerfile、Global Constraints）要一致。

### 6.4 branch 策略（必須遵守）

**Phase 0–3 全程在 feature branch 進行，只有 Phase 3 部署驗證通過後才 merge master。**

原因：master 上現有的 `.github/workflows/deploy.yml` 硬編路徑跑
`culture/tests.py`、`tech_stack/tests.py`。清理階段刪除這些 app 後若誤 merge 進 master，
任何後續 push（含緊急 hotfix）都會在 test 步驟失敗，`flyctl deploy` 永遠跑不到，
**prod 卡死在最後一個成功版本且無告警**。

若 prod 期間需要 hotfix：直接在 master 上改，事後 rebase 進 feature branch。

### 6.5 rollback 手續

Fly 的 rollback 是 `fly releases --image` 取得舊 image hash，
再 `fly deploy -i <sha>`。**但 rollback 不會還原 config**：fly.toml / env / secrets 一律使用當前版本。

這代表：若已把 `internal_port` 從 8787 改成 8080，滾回舊 image（監聽 8787）
會配上新 fly.toml，**rollback 指令執行成功但服務仍然不通**。

必須做到：
- 改 Dockerfile / fly.toml **之前**，先 `fly releases --image` 記下當前 sha
- fly.toml 變更前打一個 git tag，確保知道要連 config 一起滾回
- **`fly volumes destroy` 要延後。** 舊 image 的設定吃 `/web/db` 的 sqlite volume，
  volume 一刪 rollback 路徑就永久斷了。等新版穩定跑滿一週再刪，
  不要跟部署驗證同一天。刪之前先 `fly volumes list` 確認 id

#### prod cutover 是一個明確的時間點

首次 `flyctl deploy` 到現役 app 就是正式切換，**不是 dry-run**。
這一步之後現網從 Jinja2 舊站變成新 SPA；若 PORT、SECRET_KEY、ALLOWED_HOSTS
任一出錯，舊 machine 已被替換，站是掛的，而此時 CI 還沒接好。

做這一步之前必須確認：health check 已在 `fly.toml` 裡、
`fly secrets list` 看得到 `SECRET_KEY`、`fly config validate` 通過。

想無風險演練的話，開一次性的 app：`fly apps create cef-staging`
+ `flyctl deploy -a cef-staging`，驗完 `fly apps destroy cef-staging`。

另外 `fly secrets set` 會立刻觸發一次現役 app 的 release 與 machine 重啟。
舊 code 不吃這個 secret 所以無害，但那不是無副作用的指令。

### 6.6 Phase 4：遷移 Cloud Run（另開 branch）

GCP 一次性 infra 用 **Terraform** 管理（owner 指定，作為 IaC 學習）：
enable APIs、deployer service account + IAM roles、Workload Identity Federation、
billing budget alert。App 部署不進 Terraform（CI 的 `gcloud run deploy` 負責）；
tfstate 存本機並 gitignore。

$0 目標的真實條件：

- Cloud Run 的 Always Free 是每月 2,000,000 requests、360,000 GB-seconds 記憶體、
  180,000 vCPU-seconds、北美出向 1 GB，per billing account，且只涵蓋 request-based billing。
  以朋友等級的流量，compute 這三格用不到 1%，**Cloud Run 本身確實是 $0**。
- **破口在 Artifact Registry，同一個帳單帳戶只有 0.5 GB 儲存。**
  `python:3.13-slim` + venv + `frontend/dist` 大約 200 到 400 MB，
  兩三個 revision 就吃掉額度。必須設 cleanup policy 只保留最近 2 個 image。
- 部署時明確 `--min-instances=0`（`min-instances > 0` 走 instance-based billing，
  不吃 free tier）。
- **`--max-instances=1`**，否則預設 concurrency 80 / max 100，流量一來就開第二個
  instance，各自一份 LocMemCache，§3.3 的上界假設整個破功。
- billing budget alert 設在 $1，不是 $10。目標是 $0，$1 就該收到信。
- 2026-02-03 起部分服務要求開啟 billing，信用卡一定要綁。

遷移順序：Cloud Run 上線並驗證數天後，才 `fly apps destroy` 並刪除 `fly.toml`，不空窗。

**Phase 0 的 blocking 前置**：owner 必須先查 fly.io dashboard 的 billing，
確認是否吃 grandfathered 免費額度（Fly 於 2024-10-07 對新用戶取消免費方案，
舊有用戶保留原額度，但方案一改就回不去）。若確認在扣錢，
Phase 4 必須設定明確 deadline（建議 Phase 3 上線後 30 天內），不可停留在「隨時做」。

註：停著的 machine 仍收 rootfs 費用（每 1 GB 停機 30 天 $0.15），
volume 是 $0.15/GB/月，所以 §6.5 那顆孤兒 volume 拖著不刪是有成本的。

## 7. 清理清單

刪除：

- `main_project/main_project/templates/backup.html`（425 行未使用）
- `main.py`（PyCharm 產生的 hello world）
- Material Dashboard 全部 static assets 與模板
- Select2 / FontAwesome / Google Fonts 等 CDN 依賴
- `tech_stack` app（內容併入前端 About 頁）
- `culture` app、`utility/`
- `requirements.txt`、Poetry 設定
- `deployment_tcei/`（舊 Dockerfile 與 docker-compose 所在目錄）

**保留（不可刪）**：

- `fly.toml`：Phase 3 仍要用它部署，Phase 4 遷移完成後才刪
- `docker-compose.dev.yml`、`backend/Dockerfile.dev`、`frontend/Dockerfile.dev`
- `docs/poc/`：視覺 design reference（§4.1）
- `health_check`（改為 `backend/health/`）
- SSL workaround（封裝進 provider）

**大量刪除要有機械可判的完成條件。** Material Dashboard 那批靜態資源不可以靠
`grep | head -20` 臨場判斷（`head` 本身就會截斷）。做法是先導出完整清單到檔案、
人工掃過、刪完用 `git ls-files | grep -ci material` 期望輸出 0 當驗收。
刪除前打 `git tag pre-cleanup`，刪錯就 `git checkout pre-cleanup -- <path>` 拿回單檔。

**`.gitignore` 必須加 `staticfiles/`。** `STATIC_ROOT` 指向它，`.dockerignore`
有排除但 `.gitignore` 沒有，而清理階段用的是 `git add -A`。
只要在那之前本機跑過一次 `collectstatic`，整包 build 產物會被 commit 進去。

HTTP 請求：toolkitsy 尚無 http 模組（PyPI 0.1.0 已驗證）。
暫用 `requests` 並集中在 provider 檔案；toolkitsy 發版後單檔替換。

## 8. 測試策略

- **後端 pytest（完整）**：
  - providers：mock MoC（用 `responses`），涵蓋正常/空回應/非 JSON/HTTP 錯誤/**timeout**。
    timeout 正是 `verify=False` 打政府 API 最可能發生的失敗模式，不可漏
  - services：cache hit/miss 行為、過濾與排序 edge cases
  - API contract：status codes、錯誤格式、參數驗證（含白名單外的 category / location）
  - `pytest.ini` 要有 `addopts = --nomigrations`。專案沒有任何 model，
    每次跑測試對 in-memory sqlite 跑一輪 contenttypes migration 是純浪費
- **必須有一條端到端不 mock provider 的 services 測試。**
  用 `responses` mock MoC 的 HTTP 回應，走真的 `TaiwanProvider` →
  `search_events(month="2026-07")`，斷言拿到資料。
  否則月份格式轉換的兩端各自被 mock 掉（services 測試餵已 parse 好的 datetime、
  provider 測試只驗 parse 不驗過濾），沒有任何一條測試從
  `"2026/07/12 19:30:00"` 走到 `month=2026-07` 的結果。
- **跨月必須有專屬測試**：一個 1 月開跑、12 月結束的活動，查 9 月要命中。
- **前端 Vitest（輕量）**：日期格式化、`currentMonth()`、年月重組、
  切國家時的重設邏輯等純函式；不追 component 覆蓋率。
- **清理階段結束前的本機 prod-like container smoke test**：
  `docker build` + `docker run` + curl `/health`、`/`、`/api/v1/countries`。
  目的是把「目錄重構是否正確」與「prod 環境能否啟動」這兩個變數拆開驗證，
  不讓它們疊在唯一一次真實部署裡。
- CI 三個 test job 都跑。

### 8.1 checkpoint 的通過條件不可以無法證偽

**「回傳筆數 N ≥ 0 皆可」不是通過條件。** 月份過濾壞掉時 API 永遠回空陣列，
與「這個月剛好沒活動」在 checkpoint 上長得一模一樣。

所有涉及查詢結果的 checkpoint 都要：先打一次上游確認某個 category 與月份
確實有資料，再對該組合斷言 `len(events) > 0`。空陣列一律算沒過。

### 8.2 錯誤路徑要有人走過

手動驗收清單必須包含錯誤畫面與空結果畫面，而且是分開的兩項，
不可以寫成「看到卡片或空結果畫面」這種二選一。

- 錯誤畫面：`docker compose stop backend` 後按搜尋
- 空結果畫面：指定一個確定沒活動的月份

否則 ErrorMessage 這個元件在整個開發過程中一次都不會被執行到。

## 9. 里程碑

```
Phase 0：規劃 ✅（2026-08-22 完成）
  spec + plan 定稿（本文件 + plan v4）
  POC HTML → owner 確認（gate）✅ docs/poc/20260719_155200_ui_design_v27.html
  owner 查 fly.io billing（blocking）：尚未回報，第一個 task 開工前必須完成
  結束狀態：plan 定稿、視覺方向凍結、Phase 4 是否需要 deadline 已確定

Phase 1：骨架先立好
  目錄重構成 backend/ + config/ + 全新 settings → uv 遷移 → dev compose
  結束狀態：新目錄結構下 Django 起得來、測試跑得動、make dev 可用
  （這一段排在最前面，後續每個 task 都在最終路徑上寫 code，
    不需要中途改 settings 三次、寫 compose 三次）

Phase 2：後端
  providers → services → API endpoints（★ checkpoint：打真實 MoC）
  結束狀態：API 回得出真實資料，跨月與異體字都驗過

Phase 3：前端
  scaffold → types/api/format → i18n → UI 元件（★ checkpoint：死資料看四種畫面）
  → App 組裝 + About（★ checkpoint：全流程手動 E2E）
  結束狀態：docker-compose 起得來，SPA 打新 API 全流程可用

Phase 4：清理與 prod image
  刪舊 apps/assets → prod Dockerfile + 本機 prod-like smoke test → repo 改名 + README
  結束狀態：codebase 乾淨，prod 仍是 Fly 上的舊版

Phase 5：上 Fly.io
  fly.toml + health check + rollback 前置手續 → CI 重寫 → prod cutover + 驗證
  結束狀態：新版在 Fly.io serve 真實流量，有 uptime 監控指向 /health/upstream

Phase 6：遷移 Cloud Run（另開 branch，不 block）
  Terraform → owner 手動跑 gcp-setup → CI 換 gcloud run deploy
  → 驗證數天 → fly apps destroy + 刪 fly.toml

Phase 7：k8s（另開 branch，隨時，純學習）
  kind + manifest 跑同一個 prod image
```

### 9.1 每個 task 收尾都要能在 local 測一次

owner 的開發時間是零碎的，每個 task 結束時必須有一個當下可跑、看得到結果的驗收。
純 `tsc --noEmit` 或純單元測試綠燈不算「看得到結果」，涉及畫面的 task
要有實際開瀏覽器的步驟。

**不可中斷的 task 組**（中途停下來 repo 會處於起不來的狀態，plan 要標明）：

1. 目錄重構那一個 task：`git mv` 之後、settings/wsgi/pytest.ini 改完之前，
   所有 Python 進入點都指向不存在的 module
2. 刪除舊 app 那一個 task：清 import 與刪目錄之間停下來會 `ModuleNotFoundError`
3. fly.toml 變更到部署驗證：`internal_port` 改了但還沒部署新 image 的期間，
   config 與線上 image 不一致，此時任何人手動 deploy 或 rollback 都會拿到不通的組合

## 10. 風險與已知取捨

- **build 期 SECRET_KEY**：見 §6.1。最早的斷點，會擋掉 build、test、CI 三條路。
- **Fly 不注入 `$PORT`**：見 §6.1。且因 scale-to-zero 會靜默失敗。
- **fly.toml 缺 health check**：見 §6.1。缺了的話 port 對不齊時 deploy 仍回報成功。
- **`ALLOWED_HOSTS` 平台網域不符**：見 §6.1。上線當下全站 400。
- **rollback 不還原 config，且刪 volume 會讓 rollback 永久失效**：見 §6.5。
- **prod cutover 沒有回頭路**：見 §6.5。整份 plan 唯一會讓現有網站中斷的一步。
- **清理前移的代價**：首次真實部署時，工作樹已無舊結構可供 diff。
  緩解：Phase 4 結束前的本機 prod-like smoke test（§8）把變數拆開。
- **scale-to-zero + 監控指錯對象 = 靜默壞掉**：`/health` 不碰外部依賴，
  MoC 掛掉時它照樣 200。緩解：監控指向 `/health/upstream`（§3.4）。
- **誤 merge master 會卡死 CI**：見 §6.4。
- **Fly 費用未確認**：見 §6.6 的 Phase 0 blocking 前置。
- LocMemCache 不跨 instance、不跨 gunicorn worker、不耐重啟：已知，見 §3.3。
- MoC API 無 SLA、憑證有問題：provider 層隔離，錯誤有明確 UX。
- **MoC 回應格式可能靜默改版**（政府 open data 常見）：改欄位名時 HTTP 仍 200、
  mock 仍全綠、uptime 仍全綠。緩解：§3.4 的 sanity 訊號（raw 非空但 parsed 為 0）。
- **`backdrop-filter` 的裝置負擔**：v27 的 `.glass` 是 `blur(40px)` 加兩顆 36px 光暈，
  是全螢幕面板。低階 Android 可能掉幀甚至白屏，而驗證只在 mac 上做過。
  **這條目前沒有實質緩解**，只有 fallback 方案：提高 `--panel` 不透明度並移除 blur
  （CSS variables 一處改）。design.css 要預留註解好的低配值，臨時要降級不用重想。
  部署驗證要包含一次真實手機的捲動與 hover 順暢度確認。
- **單 category 的資料量未量測**：整個設計建立在「一次抓完某 category 的全部活動
  塞進 LocMemCache」，但沒有實測過單次回應的筆數與大小，1GB VM 要裝 12 個 category。
  (推論：撐爆的話會是 OOM kill + 間歇性 502 且無告警)
  緩解：後端 checkpoint 打真實 MoC 時順手量 `wc -c` 與筆數寫回本節；
  若單 category 超過幾 MB，LocMemCache 加 `OPTIONS: {"MAX_ENTRIES": 20}` 一行就夠。
- **加國家不是三步**：README 不可以承諾「新增一個 provider 檔就好」。
  已知至少四處要動：前端 `COMING_SOON` 硬編陣列要清（不清的話新國家會同時出現
  一個 active chip 和一個未開放 chip）、`location` 的 substring 比對綁死中文地址習慣、
  `fetch_events(category)` 的簽名假設「一次抓完整個 category 再本地過濾」、
  i18n 只有 zh/en 兩個 dict。現在不預先抽象（YAGNI 正確），但要把卡點寫下來。
- Owner 首次寫 React/TS：code 難度刻意壓低，元件小而少，
  且 §4.1 的 POC gate 已凍結視覺方向，v27 檔案是可對照的 source of truth。

## 11. 環境變數

README 要有這張表。半年後要重建環境或輪替 token 時，這是唯一該看的地方。

| 變數 | 設在哪 | 誰用它 | 沒設會怎樣 |
|---|---|---|---|
| `SECRET_KEY` | Fly：`fly secrets set`；build：collectstatic 那行的假值；test：makefile 與 CI | Django | 非 dev 時啟動即 `ImproperlyConfigured` |
| `ALLOWED_HOSTS` | Fly：`fly.toml` 的 `[env]` | Django | 非 dev 時啟動即 `ImproperlyConfigured` |
| `DEBUG` / `DJANGO_ENV` | dev：`docker-compose.dev.yml` | Django | 預設走 prod 分支 |
| `PORT` | Dockerfile 的 `ENV PORT=8080`；Cloud Run 自動注入 | gunicorn | Fly 不注入，靠 Dockerfile 的預設值 |
| `FLY_API_TOKEN` | GitHub repo secret | CI 的 deploy job | deploy job 失敗 |
| `UV_PROJECT_ENVIRONMENT` | `backend/Dockerfile.dev` | uv | venv 落在 bind mount 內被 host 覆蓋 |
