# Culture Event Finder：React + Django API 重構設計 v5

- 日期：2026-08-23
- 狀態：已與 owner 逐項確認並批准
- 取代：`2026-08-22-culture-event-finder-design-v4.md`（v4 標記 SUPERSEDED）
- 前身：`taiwan_culture_event_info_django_jinja2`（Django + Jinja2 server-rendered）

> 本文件自足。執行時不需開啟 v1 / v2 / v3 / v4。

**Phase 編號全文統一為 0 到 7，見 §9。** v4 的 §1 與 §6 用舊的一套編號
（Fly 是 Phase 3、Cloud Run 是 Phase 4），與同一份文件 §9 的里程碑對不上，
會讓實作者在前端做完就 merge master，正是 §6.4 說會卡死 CI 的那個失敗。

## 1. 背景與定位

現況是一個 Django server-rendered 網站，透過台灣文化部 (MoC) open data API 查詢藝文活動。
產品定位：**owner 自己 + 朋友真的在用的實用工具**，不是 portfolio 展示品、暫不需要 SEO。

重構的驅動需求：

1. 前後端改為 **React (Vite) SPA + Django JSON API**，兼顧 owner 的 SDET 職涯發展。
2. Hosting 分兩階段：**Phase 5 續用 Fly.io**（現有 app 仍服役，遷移風險為零，先出貨）；
   **Phase 6 才遷 Google Cloud Run** 追求 $0（always-free tier）。Phase 6 另開 branch，不 block 主線。
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
  k8s 對本專案（單 container、單 machine）無運行價值，列為 Phase 7 學習用 side quest。

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
                      "googleSearchUrl" } ],
        "meta": { "rawCount", "matchedCount", "cacheAge" } }

GET /health
    → 200，平台 health check。不碰任何外部依賴，只證明 process 活著。
```

**`meta` 是這個專案唯一的事後診斷工具，不是裝飾。** `fly logs` 只有即時串流、
machine 又是 scale-to-zero，半年後「查無結果」的回報進來時沒有任何 log 可讀（§3.4）。
三個數字分別回答三個問題：`rawCount` 是上游給了幾筆（0 代表 MoC 那邊沒資料或格式變了）、
`matchedCount` 是本地過濾後剩幾筆（`rawCount` 大而 `matchedCount` 為 0 代表過濾邏輯壞了）、
`cacheAge` 是這份資料在 cache 裡幾秒了（`null` 代表這次是 miss）。
前端不顯示它們，但 owner 一個 curl 就能分辨是上游問題還是自己的問題。

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

#### ⚠️ 驗證通過但轉型爆炸的兩個洞

驗證與轉型用的不是同一套判定時，會出現「守衛說沒問題、下一行就 500」的情況。
這兩個洞都是回 Django 500 HTML，不是合約承諾的 400 JSON，前端只看得到「發生錯誤」。

1. **`str.isdigit()` 對上標數字回 True。** `'²'.isdigit()` 是 `True`，
   而 `int('²')` 拋 `ValueError`。`?category=²` 通過守衛、死在轉型。
   判定要寫 `category.isascii() and category.isdigit()`。
2. **月份 regex 允許不存在的年份。** `^\d{4}-(0[1-9]|1[0-2])$` 接受 `0000-01`，
   而 `datetime.strptime("0000-01-01", "%Y-%m-%d")` 拋
   `ValueError: year 0 is out of range`。regex 收斂成
   `(19|20)\d{2}-(0[1-9]|1[0-2])`。

**`location` 的白名單比對要先正規化再查。** 過濾階段會把 `台` 折成 `臺`（見 §3.2），
但白名單比對如果拿原值去查集合，`location=台北` 這個書籤會拿到 400，
而它的過濾邏輯本來是會命中的。兩處要用同一個正規化函式。

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

#### 台灣的 locations 是 20 個前綴，涵蓋 22 個縣市

實測 MoC 資料（category 1/6/17 共 1320 筆 showInfo）：`宜蘭縣` 有 9 筆，
但既有 `data.py` 的清單裡沒有宜蘭，使用者永遠選不到。連江同樣缺漏。
清單必須涵蓋全部 22 個縣市。

**但清單的長度是 20 不是 22。** 新竹市與新竹縣共用 `新竹` 這個前綴，
嘉義市與嘉義縣共用 `嘉義`，所以 20 個前綴涵蓋 22 個縣市。
測試斷言的數字是 20，測試名稱要寫成
`test_location_prefixes_cover_22_counties`，不要寫 `test_location_count_is_twenty`。
把「22」當成清單長度會讓人補上新竹市與嘉義市，弄壞三處斷言加一個 checkpoint。

#### 上游回應必須先確認是 list

`response.json()` 只保證是合法 JSON，不保證是陣列。政府 open data 改版包一層
`{"data": [...]}`、或維護頁回 `{"message": "..."}` 帶 HTTP 200，都會讓
`for item in payload` 拿到 `str`，`item.get(...)` 拋 `AttributeError`。
`AttributeError` 不是 `UpstreamError`，view 的 `except UpstreamError` 接不到，
使用者拿到 500 而不是設計好的 502，而且上游健康訊號完全沒動。

`payload = response.json()` 的下一行就要 `if not isinstance(payload, list): raise UpstreamError(...)`，
而且解析迴圈本身要 `except (AttributeError, TypeError)` 轉成 `UpstreamError`。
這正是 §3.4 的格式漂移訊號要抓的東西，不補的話它會從網子裡漏出去。

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
- **失敗也要進 cache（negative caching），TTL 60 秒。** 只在成功時寫 cache 的話，
  上游持續失敗期間**每一個 request 都會重打一次 15 秒的上游**，八個 thread 全部
  停在那裡，正是本節加 gthread 要避免的狀況，而且是在上游最虛弱的時候加倍打它。
  失敗時寫一個短 TTL 的失敗標記，60 秒內的後續請求直接回 502 不再打上游。
  60 秒夠短，上游恢復後最多一分鐘就會重試。
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
- **不做 `/health/upstream`。** v4 的設計是 provider 失敗時寫一個 cache 旗標、
  開一個 endpoint 讀它、監控指向那個 endpoint。這個設計的前提不成立，理由有三個，
  任一個都足以讓它永遠回綠燈：
  1. 旗標只在 cache **miss** 的路徑上被寫，也就是只有真人搜尋才會動。
     沒人搜的時段，MoC 掛掉不會留下任何痕跡。
  2. 旗標放在 LocMemCache，而 machine 是 scale-to-zero。機器一停旗標就消失，
     重啟後必然是「乾淨」狀態，與上游實際健康無關。
  3. 監控每次去讀的是一個唯讀的 cache view，它自己從不碰 MoC。
     所以這個「上游監控」整條鏈路裡沒有任何一段真的接觸上游。

  取代方案：**uptime 監控直接打一個真實的搜尋 URL**
  （`/api/v1/tw/events?category=6&location=臺北&month=<當月>`），
  對 HTTP 502 或 `"events": []` 告警。它會真的走完 provider、真的碰到 MoC，
  而且順便證明過濾邏輯還活著。頻率見 §6.5 的監控設定。
- **格式漂移的 sanity 訊號**：一次 fetch 若 raw payload 非空但 parsed events 為 0，
  先 `logger.error` 再拋 `UpstreamError`。MoC 改欄位名時 HTTP 仍是 200、mock 仍全綠，
  這個訊號是唯一會亮的東西。**順序不可以顛倒**：v4 的 code 直接 raise 沒有 log，
  等於這個「唯一會亮的東西」一行紀錄都不會留下。
- **每個查詢回應都帶 `meta`（§3.1）。** 這是取代旗標的事後診斷手段：
  不需要 log 保留期，也不需要監控，owner 一個 curl 就分得出上游問題與自己的問題。
- **已知限制**：`fly logs` 只有即時串流，machine 是 scale-to-zero，
  事後拿到 `X-Request-ID` 也還原不了當時的 log。README 要寫明這一點，
  不要讓那個 header 看起來像可以追查的東西。
- **correlation id 在 `--threads 8` 下必須先驗過。** 八個 request 共用一個 process，
  `toolkitsy.logger.set_correlation_id` 若用 module global 而非 `contextvars`
  或 `threading.local`，log 會互相錯掛，而且錯掛比沒有 id 更糟。
  後端 checkpoint 要跑一次 `inspect.getsource` 確認它用的是哪一種，
  確認不了就不要裝這個 middleware。

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
  - **卡片的時間必須顯示區間，不是只有 `startTime`。** 後端整節 §3.3 在打
    區間重疊比對的仗，`endTime` 也一路傳到前端的型別裡，卡片只 render `startTime`
    等於把區間丟掉。展覽有 88% 跨月，使用者查九月會看到每一張卡都寫「01/01」，
    合理判斷這個站的資料是舊的。同月顯示 `07/12 19:30 – 07/14`，
    跨月顯示 `2026/01/01 – 2026/12/31`，`endTime` 為 `null` 時只顯示開始時間
  - **票價是自由文字，不可以直接前綴 `$`。** MoC 那一欄實際會出現
    「洽詢主辦單位」「0」「免費」。純數字才加 `$`，`0` 顯示成「免費」，
    其餘原樣顯示不加前綴
  - **結果計數列顯示完整條件**（「臺北 · 音樂 · 2026/07 共 12 筆」）。
    預設類別是 provider 的第一個類別而不是「全部」，MoC 也沒有 all-category 查詢能力，
    條件不顯性寫出來的話使用者會誤以為看到的是全類別
  - **Loading**：skeleton 卡片（`--skel` 變數，雙主題各自可見，含分隔線 + 票價列 skeleton）
  - **四種狀態都要有畫面**：`idle` / `loading` / `empty` / `error`。
    `idle` 不可以是一片空白，否則使用者一進站看不出要按搜尋鈕，
    也分不出「還沒搜」與「查無結果」。空結果與錯誤各配一個大型線條 SVG 插圖，
    外層加虛線邊框容器 (`border-dashed`) 與 `--surface-2` 底色；錯誤畫面附重試按鈕
  - **空結果的文案叫使用者做什麼，那個東西就必須存在。** v4 的文案寫
    「或者清除篩選條件重新搜尋」而 `SearchForm` 裡沒有任何清除鈕。
    空結果是冷門搜尋最常見的結果，所以這是全站最多人會讀到的一段字。
    兩條路擇一：`SearchForm` 真的加一顆「重設」鈕（把四個欄位回到預設值），
    或把文案改成「換個地區或月份再試一次」。**選加鈕**，因為手機上
    重設四個 `<select>` 要點八下
  - **loading 超過 8 秒要換一段文案。** 冷機器加上 15 秒的上游 timeout，
    最壞情況是 20 秒的骨架動畫配一片安靜。8 秒後把 skeleton 上方的文字換成
    「第一次查詢比較慢，正在向文化部要資料」，讓使用者知道站沒死。
    不做取消鈕（多一個狀態要管），但這段文案不可省
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
- **語言也要持久化，做法與主題完全一樣。** v4 只持久化了主題，`lang` 是裸的
  `useState<Lang>("zh")`。看英文的朋友每次進站都要重切一次，而且他不會知道
  這個站記得住主題卻記不住語言。lazy initializer 讀 `localStorage`，
  讀不到才看 `navigator.language`。
- **搜尋要能被取消，否則兩次搜尋會互相蓋掉。** `api.ts` 已經收 `AbortSignal`，
  但 v4 的 `handleSearch` 從來沒傳。類別 chip 是直接觸發搜尋的（見上面的
  「類別 chips」），所以「點展覽（cache miss，最多 15 秒）再點音樂（cache hit，
  200 毫秒）」是一根手指就會走到的路徑：音樂先渲染，展覽後到把畫面換掉，
  而音樂的 chip 還亮著。每次搜尋開一個 `AbortController`，
  新的搜尋先 abort 舊的，`AbortError` 不寫 state。
  `loadCountries` 已經是這個寫法，照抄即可。
- **切換畫面要進 history，否則手機的返回鍵會直接離站。** `view` 是純 state，
  About 頁按返回等於離開網站。這與 §1 排除的 shareable URL 是兩件事：
  那個排除的是把搜尋條件寫進網址，這裡要的只是 `history.pushState` 加一個
  `popstate` listener，兩行。
- **重試按鈕要接對對象**：初次載入 `/countries` 失敗時 `form.country` 是空字串，
  若重試接的是搜尋，會打 `/api/v1//events` 而永遠失敗，使用者按幾次都一樣、
  下拉選單全空無法自救。錯誤來源要分開記，重試依來源決定重載 countries 或重搜。
- **countries 載入完成前送出鈕要 disabled**，否則網路慢時先按下去會打出雙斜線 URL。
- **每個畫面都要有一個 `<h1>`，而且不能被關在某一個分支裡。** v4 的桌機 `<h1>`
  只寫在搜尋分支內、手機的那個是 `sm:hidden`，所以桌機開 About 頁時整份 DOM
  最高只到 `<h2>`。`<h1>` 要放在 `view` 判斷之外。
- **圖示按鈕與有文字的按鈕，`aria-label` 規則是相反的。** 純 icon 的按鈕
  （主題、搜尋、關於）必須補 `aria-label`。但語言切換鈕看得到的字是 `EN` 或 `中`，
  再掛一個「切換語言」的 `aria-label` 會讓兩者毫無交集，語音控制使用者說
  「click EN」點不到（WCAG 2.5.3 Label in Name）。有可見文字時就讓文字當
  accessible name，不要另外掛 label。
- **外連一律 `rel="noopener noreferrer"`。** 現代瀏覽器的 `noreferrer` 已隱含
  `noopener`，但舊的內嵌 webview 不一定，而補上去成本是零。
- **選項比對前先轉字串。** `LabeledOption.value` 的型別是 `string | number`，
  而類別快捷 chip 硬寫的是 `number`。後端哪天把 value 序列化成字串，
  `c.value === q.id` 就整排 chip 靜默消失，沒有 error 也沒有 console warning。
  兩邊都套 `String()` 再比。

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
- **backend 的 `CMD` 必須是 `uv run --frozen --no-sync`。** 裸的 `uv run` 每次啟動
  都會重新 resolve 並 sync，而專案目錄是 bind mount 的 host repo。lock 只要稍微
  drift，container 內的 process 就會把 `uv.lock` 改寫回你的工作目錄；
  Linux host 上因為權限不符會直接起不來。
- **non-root user 加 bind mount 在 Linux host 上會壞。** build 時的 `chown`
  被 runtime 的 mount 蓋掉，檔案樹仍屬於 host 的 UID，container 內的 `appuser`
  連 `__pycache__` 都寫不了。macOS 的 Docker Desktop 會假裝 ownership 所以本機測不出來。
  compose 的 backend service 要加 `user: "${UID:-1000}:${GID:-1000}"`。
  這一條直接關係到本節「任何人 clone 下來跑 `make dev` 就有一致環境」這個目的。

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
`make test-backend`、`make test-frontend`、`make test`、
`make run-prod`（本機 build & run production container）。

`make run-prod` 依賴 repo root 的 Dockerfile，那是後面的 task 才產出的檔案；
makefile 定稿時要在該 target 註解說明，避免執行者撞牆後懷疑是自己漏做。

**`make test` 必須拆成 `test-backend` 與 `test-frontend` 兩個 target。**
v4 把兩者寫在同一個 target 裡，而 `frontend/` 要到前端 scaffold 那個 task 才存在。
在那之前 `make test` 會在 `cd frontend` 這一行直接 abort，
而 `make test` 正是 owner 唯一背下來的指令。`test` 呼叫另外兩個，
前端那半在 scaffold 之後才接進去。

## 6. 部署

### 6.1 Phase 5 目標平台：Fly.io

- **Multi-stage Dockerfile**（repo root）：
  stage 1 (`node:22-slim`) `vite build` → stage 2 (`python:3.13-slim`) Django + gunicorn，
  WhiteNoise 服務 React build 產物 + `/api` JSON。單一 container、單一網址、無 CORS。
- 新增依賴：`gunicorn`、`whitenoise`。
- Python stage 需跑 `collectstatic`；`STATICFILES_DIRS` 指向 `frontend/dist`。
- WhiteNoise 用 plain storage（非 manifest storage）：Vite 已對檔名做 content-hash，
  manifest storage 會重複 hash 且可能 500。
- **但 plain storage 要自訂 `WHITENOISE_IMMUTABLE_FILE_TEST`。** WhiteNoise 預設
  的判定是 `name.<12 位 hex>.ext`，Vite 產出的是 `index-DcJk2sLm.js`（破折號 +
  base64url），一個都不符合。結果是每一個已經 content-hash 過的資產都拿到
  `max-age=60` 而不是 immutable，回訪的使用者每分鐘重下載整包 JS。
  這個問題**只在 prod 出現而且永遠不會報錯**，本機與 CI 都看不到。
- **`base: "/static/"` 只在 production 生效，所以 `public/` 的資產只在 prod 壞。**
  Vite 會改寫 `index.html` 裡的字面引用，但 JSX 裡 runtime 寫死的
  `<img src="/hero.png">` 不會被加前綴，dev 正常、prod 404。
  對策是 CI 的 build-smoke 放一個 `public/` 資產並 curl 它（§6.2）。
- prod stage 也要跑 non-root user。dev container 有做而 prod 沒做是反過來的。
- CMD：`gunicorn --chdir backend config.wsgi:application --bind 0.0.0.0:${PORT:-8080}
  --workers 1 --threads 8 --worker-class gthread --timeout 60`

#### ⚠️ build 期的 SECRET_KEY

**這是整條部署鏈最早的斷點。**

settings 的 fail-fast 守衛（見下）在 `collectstatic` 執行時就會觸發：
build 環境沒有 `DEBUG` 也沒有 `SECRET_KEY`，於是直接 `ImproperlyConfigured`，
`docker build` 失敗。連帶本機 smoke test、CI 的 build-smoke job、
`make run-prod`、Fly 的 remote build 全部起不來。

必須做到：collectstatic 那一行帶 build-only 假值。**`ALLOWED_HOSTS` 同樣是
fail-fast，所以兩個都要帶，只帶 `SECRET_KEY` 那行是跑不起來的**
（v4 在本節與 Global Constraints 都漏了 `ALLOWED_HOSTS`，照抄的人會在
「這一節本來要防的那一步」build 失敗）：

```dockerfile
RUN SECRET_KEY=build-only-not-used ALLOWED_HOSTS=build-only \
    python backend/manage.py collectstatic --noinput
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

prod 網域一律由平台的環境變數注入（Phase 5 是 `fly.toml` 的 `[env]`，
Phase 6 換成 Cloud Run），fallback 只涵蓋 local dev，刻意不含 `.fly.dev`。

dev 的判定**只實作 `DEBUG` 一個訊號**。v4 的內文寫「`DJANGO_ENV` 或 `DEBUG`」
但 settings 只讀 `DEBUG`，README 又只列 `DJANGO_ENV`，三處各說各話。
統一成 `DEBUG`，環境變數表也只留 `DEBUG`。

### 6.2 CI/CD

**GitHub Actions**（push master）四個 job：

1. `test-backend`：pytest（要注入假 SECRET_KEY）
2. `test-frontend`：vitest
3. `build-smoke`：真的 `docker build` 起 container，curl `/health`、`/`、
   `/api/v1/countries`、一個放在 `public/` 的資產（驗 §6.1 的 base path），
   以及一個不存在的路徑（驗 SPA catch-all 回 200 不是 500）。
   失敗時要 `docker logs`（`if: failure()`），
   否則只看得到 curl 的非零離開碼，分不出是啟動失敗還是路由不對。
4. `deploy`：`needs: [test-backend, test-frontend, build-smoke]`，
   `if: github.ref == 'refs/heads/master'`，用 `flyctl deploy` + `secrets.FLY_API_TOKEN`

紅燈不部署。**不需要** `permissions.id-token: write`（那是 Workload Identity Federation
用的，Phase 6 才需要）。

**build-smoke 不可以打真實的 MoC。** v4 讓它 curl events endpoint 再
`grep -q 'events'`，兩個問題：政府 API 一有狀況你的 pipeline 就紅燈且擋住 deploy；
而且 `grep -q 'events'` 對 `{"events": []}` 也會過，正是 §8.1 明文拒絕的通過條件。
contract 漂移由 `test-backend` 的 `responses` mock 負責，
真實 MoC 的驗證留在後端 checkpoint 與 cutover 那兩個人工關卡。

**workflow 要有 `concurrency` group。** 單一 machine 就地替換，
兩次快速 push 會讓兩個 `flyctl deploy` 對同一台機器賽跑。

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

**Phase 0–5 全程在 feature branch 進行，只有 Phase 5 部署驗證通過後才 merge master。**

（v4 這一行寫「Phase 0–3」，配上 §1 的舊編號會被讀成「前端做完就 merge」。
判準不是 phase 編號而是這件事：**master 上的舊 CI 還在跑已被刪掉的 app 的測試，
所以 merge 的時間點必須晚於新 CI 上線。**）

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

**所以本專案的 rollback 路徑不是 `fly deploy -i <sha>`。** 那個指令只換 image，
而壞掉的組合正是「新 config 配舊 image」。真正的路徑是：

```bash
git checkout pre-phase5-fly-config && fly deploy
```

也就是連 config 一起滾回、完整重 build，要好幾分鐘。**這一行必須寫進 README，
不能只寫在 spec 裡**：半夜第一個伸手的人會照直覺去換 image，
然後得到一個「rollback 成功但站還是不通」的結果，並開始懷疑 Fly 壞了。

必須做到：
- 改 Dockerfile / fly.toml **之前**，先 `fly releases --image` 記下當前 sha
- fly.toml 變更前打 git tag `pre-phase5-fly-config`，這個 tag 名字要寫進 README
- **`fly volumes destroy` 要延後。** 舊 image 的設定吃 `/web/db` 的 sqlite volume，
  volume 一刪 rollback 路徑就永久斷了。等新版穩定跑滿一週再刪，
  不要跟部署驗證同一天。刪之前先 `fly volumes list` 確認 id

#### prod cutover 是一個明確的時間點

首次 `flyctl deploy` 到現役 app 就是正式切換，**不是 dry-run**。
這一步之後現網從 Jinja2 舊站變成新 SPA；若 PORT、SECRET_KEY、ALLOWED_HOSTS
任一出錯，舊 machine 已被替換，站是掛的，而此時 CI 還沒接好。

做這一步之前必須確認：health check 已在 `fly.toml` 裡、
`fly secrets list` 看得到 `SECRET_KEY`、`fly config validate` 通過。

**`fly config validate` 只驗語法，證不了這一節在乎的任何一件事。**
port 對不對得上、health check 的 Host header 會不會被 `ALLOWED_HOSTS` 擋，
兩者都要到真的部署才現形。這是 Fly 的先天限制，不是可以補的驗收，
所以下面那個 staging 演練不是選配。

想無風險演練的話，開一次性的 app：`fly apps create cef-staging`
+ `flyctl deploy -a cef-staging`，驗完 `fly apps destroy cef-staging`。
**演練時要帶 `-e ALLOWED_HOSTS=cef-staging.fly.dev`**：不帶的話這場演練
剛好跳過風險最高的那個變數，等於演了一場沒有主角的戲。

**cutover 本身有停機。** 單一 machine 就地替換，最少 10 到 30 秒，
失敗的話沒有上限。要縮短就在切換前 `fly scale count 2`，
讓 rolling 策略有地方轉移流量，驗完再 `fly scale count 1`。
（維持兩台會讓 §3.3 的 LocMemCache 上界假設破功，所以是暫時的。）

#### 監控設定

**一個 monitor，每 30 分鐘，打真實搜尋 URL。**（owner 於 2026-08-23 選定省錢方案）

- 目標：`/api/v1/tw/events?category=6&location=臺北&month=<當月>`，
  對 HTTP 502 或 `"events": []` 告警。理由見 §3.4：只打 `/health` 或
  讀 cache 旗標的監控，在 MoC 掛掉時照樣是綠燈
- 頻率不可以是 5 分鐘。`min_machines_running = 0` 的省錢設定要成立，
  前提是機器真的會停；每 5 分鐘打一次會讓它 24 小時醒著，
  等於付了常開的錢卻拿到 scale-to-zero 的冷啟動體驗。v4 排了兩個 5 分鐘的 monitor，
  正是這個組合
- 代價要講清楚：機器停著的時候，使用者第一次搜尋最壞要等 20 秒
  （冷啟動加上上游 15 秒 timeout），而且 cache 是空的。
  §4 的「loading 超過 8 秒換文案」就是為這個情境寫的

另外 `fly secrets set` 會立刻觸發一次現役 app 的 release 與 machine 重啟。
舊 code 不吃這個 secret 所以無害，但那不是無副作用的指令。

### 6.6 Phase 6：遷移 Cloud Run（另開 branch）

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
- **`--concurrency=8` 必須跟著設。** Cloud Run 預設一個 instance 同時收 80 個
  request，而 container 只有 8 個 gunicorn thread。差額會排隊到 request timeout。
  這個數字要跟 `--threads` 對齊，改一邊就要改另一邊。
- **`--max-instances=1` 是一個刻意的天花板，要寫在 README 裡。** 超過
  8 個並行加上排隊就會回 429。這是 LocMemCache 換來的代價，不是設定錯誤，
  半年後看到 429 的人要能在 README 找到這句話。
- billing budget alert 設在 $1，不是 $10。目標是 $0，$1 就該收到信。
- 2026-02-03 起部分服務要求開啟 billing，信用卡一定要綁。

遷移順序：Cloud Run 上線並驗證數天後，才 `fly apps destroy` 並刪除 `fly.toml`，不空窗。

**Phase 0 的 blocking 前置**：owner 必須先查 fly.io dashboard 的 billing，
確認是否吃 grandfathered 免費額度（Fly 於 2024-10-07 對新用戶取消免費方案，
舊有用戶保留原額度，但方案一改就回不去）。若確認在扣錢，
Phase 6 必須設定明確 deadline（建議 Phase 5 上線後 30 天內），不可停留在「隨時做」。

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

- `fly.toml`：Phase 5 仍要用它部署，Phase 6 遷移完成後才刪
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
  - **`pytest.ini` 也必須有 `python_files = test_*.py tests.py`。**
    pytest 預設只收 `test_*.py` 與 `*_test.py`，而 Django 慣例的 `health/tests.py`
    兩者都不符，會被靜默略過。實測 `pytest .` 對一個只有 `health/tests.py`
    的目錄收到 0 個測試並以 exit code 5 結束，而 `pytest .` 正是 makefile、
    CI、與每一個 checkpoint 用的指令。這一行不加的話：骨架階段的驗收會
    以「測試指令壞掉」的形式失敗，之後的每一次全測都會靜默少跑 health 那組
- **必須有一條端到端不 mock provider 的 services 測試。**
  用 `responses` mock MoC 的 HTTP 回應，走真的 `TaiwanProvider` →
  `search_events(month="2026-07")`，斷言拿到資料。
  否則月份格式轉換的兩端各自被 mock 掉（services 測試餵已 parse 好的 datetime、
  provider 測試只驗 parse 不驗過濾），沒有任何一條測試從
  `"2026/07/12 19:30:00"` 走到 `month=2026-07` 的結果。
- **跨月必須有專屬測試**：一個 1 月開跑、12 月結束的活動，查 9 月要命中。
- **前端 Vitest（輕量）**：日期格式化、`currentMonth()`、年月重組、
  切國家時的重設邏輯等純函式；不追 component 覆蓋率。
- **i18n 那個 task 不可以用 `tsc --noEmit` 當驗收。** 「無輸出」證明的只是型別對，
  不證明缺 key 有 fallback、不證明語言挑對、不證明切換鈕會 render。
  要三個 Vitest 斷言：缺 key 回傳 key 本身、`pickLabel` 在 `en` 下拿到英文、
  切換後 `document.documentElement.lang` 真的變了。Vitest 在 scaffold 那個 task
  就裝好了，這裡是零新依賴。
- **cache 那一層要留一行 log。** v4 的 cache-aside 只對 `MagicMock` 驗過，
  而且整個專案沒有任何一行印出 cache 是 hit 還是 miss。
  加 `logger.info("cache %s key=%s", ...)` 之後，那個 task 當場可以本機驗
  （同一個查詢跑兩次看 log），而且 cutover 的驗收不用再靠回應時間去猜。
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
Phase 0：規劃 ✅（2026-08-23 完成）
  spec + plan 定稿（本文件 + plan v5）
  POC HTML → owner 確認（gate）✅ docs/poc/20260719_155200_ui_design_v27.html
  owner 查 fly.io billing（blocking）：尚未回報，第一個 task 開工前必須完成
  結束狀態：plan 定稿、視覺方向凍結、Phase 6 是否需要 deadline 已確定

Phase 1：骨架先立好
  uv 遷移 → 目錄重構成 backend/ + config/ + 全新 settings → dev compose
  → 前端 scaffold（Vite + React + TS + Tailwind + Vitest）
  結束狀態：新目錄結構下 Django 起得來、make test 前後端都跑得動、make dev 兩個
  service 都起得來
  （前端 scaffold 排進 Phase 1 而不是 Phase 3：makefile 與 compose 因此只寫一次，
    而且 make test 從這裡開始到收工都是可用的。v4 把它排在後端之後，
    導致中間三個 task 期間 make test 會在 cd frontend 那行 abort）

Phase 2：後端
  providers → services → API endpoints（★ checkpoint：打真實 MoC）
  結束狀態：API 回得出真實資料，跨月與異體字都驗過

Phase 3：前端
  types/api/format → i18n → UI 元件三段（★ checkpoint 各一次）
  → App 組裝 + About（★ checkpoint：全流程手動 E2E）
  結束狀態：docker-compose 起得來，SPA 打新 API 全流程可用

Phase 4：清理與 prod image
  刪舊 apps/assets → prod Dockerfile + 本機 prod-like smoke test（★ checkpoint）
  → repo 改名 + README
  結束狀態：codebase 乾淨，prod 仍是 Fly 上的舊版

Phase 5：上 Fly.io
  fly.toml + health check + rollback 前置手續 → CI 重寫
  → prod cutover + 驗證（★ checkpoint）
  結束狀態：新版在 Fly.io serve 真實流量，一個 uptime 監控每 30 分鐘打真實搜尋 URL

Phase 6：遷移 Cloud Run（另開 branch，不 block）
  Terraform → owner 手動跑 gcp-setup → CI 換 gcloud run deploy
  → 驗證數天 → fly apps destroy + 刪 fly.toml

Phase 7：k8s（另開 branch，隨時，純學習）
  kind + manifest 跑同一個 prod image
```

### 9.0 ★ checkpoint 放在哪裡

判準是「這一步之後要退回去很貴」，不是「這一步很難」。所以 checkpoint 要標在
不可逆的點與最後一道防線上，v4 那四個全部落在中段，兩個最危險的時刻反而沒有標。

1. **刪舊 app 那個 task**：刪掉六個目錄、換掉整份 settings，是第一個不可逆點
2. **後端 API endpoints**：第一次打到真實 MoC
3. **前端 scaffold**：第一次看到瀏覽器畫面
4. **UI 元件三段各一次**：每一段結束都能開瀏覽器看到那一段做出來的東西
5. **App 組裝**：全流程手動 E2E
6. **prod image 的本機 smoke test**：真實部署前的最後一道防線，
   plan 自己寫了「唯一一次真實部署前的最後防線」卻沒標
7. **cutover 後的 curl 驗證**：port 與 health check 這兩個靜默殺手
   終於被證明的那一刻

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
  MoC 掛掉時它照樣 200。緩解：監控打真實搜尋 URL（§3.4、§6.5）。
  v4 的緩解手段是 `/health/upstream`，但那個 endpoint 讀的旗標在
  scale-to-zero 下必然消失、而且只有真人搜尋才會被寫，
  所以它是「看起來有監控」而不是有監控，比沒有更危險。
- **省錢方案的代價是冷啟動**（owner 於 2026-08-23 選定）：
  `min_machines_running = 0` 加上每 30 分鐘一次的監控，機器大部分時間是停的。
  使用者第一次搜尋最壞等 20 秒且 cache 全空。緩解是 §4 的 8 秒文案，
  不是技術解，是把等待講清楚。
- **誤 merge master 會卡死 CI**：見 §6.4。
- **Fly 費用未確認**：見 §6.6 的 Phase 0 blocking 前置。
- **`verify=False` 是永久且靜音的**：`urllib3.disable_warnings` 是 process 全域。
  MoC 哪天把憑證修好、或換一個 host，沒有任何東西會通知 owner 可以拿掉它。
  README 的維運段要列出這一行並註明「這是暫時解，狀態未被監控」。
- **12 個 category 只實測過 3 個**：category 1/6/17 驗過 payload 形狀，
  其餘九個未驗證。(推論：若某個 category 的 `showInfo` 結構不同，
  會走到「raw 非空但 parsed 為 0」那條 sanity 路徑，回 502 而不是靜默空白)
  緩解：後端 checkpoint 順手把 12 個都打一次，記下筆數。
- **網域硬編在兩個地方**：health check 的 Host header 與 `ALLOWED_HOSTS`
  都寫死 `taiwan-culture-event-info.fly.dev`。換自訂網域或搬 Cloud Run 兩邊都要改，
  漏一邊的症狀是「部署成功但全站 400」。
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
| `DEBUG` | dev：`docker-compose.dev.yml` | Django | 預設走 prod 分支 |
| `PORT` | Dockerfile 的 `ENV PORT=8080`；Cloud Run 自動注入 | gunicorn | Fly 不注入，靠 Dockerfile 的預設值 |
| `FLY_API_TOKEN` | GitHub repo secret | CI 的 deploy job | deploy job 失敗 |
| `UV_PROJECT_ENVIRONMENT` | `backend/Dockerfile.dev` | uv | venv 落在 bind mount 內被 host 覆蓋 |
| `UID` / `GID` | dev：host shell（compose 讀，有預設值 1000） | dev container 的 non-root user | Linux host 上 container 寫不了 bind mount |

`DJANGO_ENV` 不存在。v4 的內文提過它但 settings 沒實作，不要照著找。

## 12. v4 → v5 改了什麼

v5 是 v4 加上一輪六視角 review 的修正，架構、視覺方向、技術選型全部不變。
改動集中在七個必修項與四個 owner 決策：

**必修（不改會壞）**

1. `pytest.ini` 加 `python_files`，否則每一次全測都靜默漏掉 health 那組（§8）
2. 刪舊 app 那個 task 的檔案清單順序，否則會在標了「不可中斷」的 task 裡 abort（見 plan）
3. Phase 編號全文統一，否則會照 §6.4 的舊敘述太早 merge master（本文件開頭 + §6.4 + §9）
4. `isascii()` 與月份 regex，否則驗證通過的參數會在轉型時噴 500（§3.1）
5. 上游回應先確認是 list，否則格式漂移會變成 500 而不是 502（§3.2）
6. 卡片顯示時間區間，否則展覽類別的 88% 看起來像舊資料（§4）
7. dev container 的 `uv run --frozen --no-sync`，否則會改寫 host 的 `uv.lock`（§5.1）

**owner 決策（2026-08-23）**

1. 開 v5，不覆蓋 v4
2. 前端 scaffold 從 Phase 3 提前到 Phase 1（§9）
3. UI 元件那個 task 拆成三段，每段都能開瀏覽器驗（§9）
4. Fly 走省錢方案：一個 monitor 每 30 分鐘，接受冷啟動（§6.5）

**仍未處理、待 owner 決定的 overdesign 項目**（review 提出，v5 沒動）：
裝飾用 SVG 約 150 行、i18n 的雙語 label schema 約 60 行、
`COMING_SOON` 日韓 chip 與四個類別快捷 chip 約 70 行。
這三項都是刪了會改變視覺或功能的東西，不在「修正」範圍內。
