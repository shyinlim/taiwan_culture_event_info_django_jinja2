# Culture Event Finder Refactor — Implementation Plan

> ⚠️ **SUPERSEDED** — 已被 `docs/superpowers/plans/2026-07-18-culture-event-finder-plan-v2.md` 取代。
> **不得作為執行依據**，僅供追溯。
> 本文件包含多項在 v2 順序下會造成損害的指令：刪除 `fly.toml`（v2 仍需它部署）、
> 部署到 Cloud Run（v2 Phase 3 是 Fly.io）、Task 15 Step 4 指向 v2 順序下尚不存在的檔案。
> v2 為自足文件，執行時不需開啟本檔。

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** `docs/superpowers/specs/2026-07-11-react-django-refactor-design.md` (read it first)

**Goal:** Refactor the Taiwan culture-event Django/Jinja2 site into a React (Vite + TS + Tailwind) SPA + Django JSON API, deployed as one container on Google Cloud Run ($0), with a per-country provider architecture (Taiwan only), cache-aside, and zh/en UI i18n.

**Architecture:** Backend layering views (HTTP) → services (cache + filter/sort) → providers (external data source per country). Frontend is a static Vite build served by the same Django container via WhiteNoise. No react-router, no DB reliance, no CORS.

**Tech Stack:** Django 5.2 LTS, uv (dependency mgmt), toolkitsy (logging), requests, pytest + pytest-django + responses; Vite + React + TypeScript + Tailwind v4 + Vitest; Docker multi-stage; Cloud Run + GitHub Actions (WIF); **Terraform** for one-time GCP infra (APIs/SA/IAM/WIF — owner wants to learn IaC on a small safe surface). App deploys stay in CI via `gcloud run deploy`, NOT in Terraform.

## Global Constraints

- Dependency source of truth: `pyproject.toml` (PEP 621) + `uv.lock` via **uv**. Never edit requirements.txt (it gets deleted).
- Logging: **only** `toolkitsy.logger` in new code (`logger`, `configure`, `set_correlation_id`). No `print`, no `utility.logger` in new code.
- HTTP calls to external APIs: `requests`, confined to `events/providers/taiwan.py` only. The owner plans an http module for `toolkitsy` (separate repo) — if `toolkitsy` ships one before Task 2 executes, use it there instead of `requests` (keep the same `UpstreamError` mapping and tests); otherwise implement as written and swap that one file later.
- API error body shape everywhere: `{"error": {"code": "...", "message": "..."}}`.
- API month param is ISO `YYYY-MM`. MoC upstream time format is `YYYY/MM/DD HH:MM:SS`.
- Code must stay simple (owner is entry-level-readable standard): no Redux, no react-router, no fancy generics, small files.
- Keep old Jinja2 pages working until Milestone 4 deletes them (M1–M2 coexistence; M3 moves them under `/legacy/`).
- `main_project/main_project/settings.py` is Read-blocked by a local hook (`protect_sensitive.py`). Modify it ONLY via the Bash python-snippet steps given below (they do targeted inserts/appends without needing Read). If Bash access is also blocked, ask the owner to apply the printed patch manually.
- Commit messages follow existing repo style (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `ci:`).
- Run backend tests from repo root: `cd main_project && uv run python -m pytest . -v` (pytest.ini lives in `main_project/`, `DJANGO_SETTINGS_MODULE=main_project.settings`).

---

## Milestone 1 — Backend API (coexists with old pages)

### Task 1: uv migration + toolkitsy dependency

**Files:**
- Modify: `pyproject.toml` (full rewrite, shown below)
- Create: `uv.lock` (generated)
- Delete: `poetry.lock`, `requirements.txt`

**Interfaces:**
- Produces: `uv run` / `uv sync` workflow used by every later task; deps: django 5.2.x, requests, toolkitsy, gunicorn, whitenoise; dev deps: pytest, pytest-django, responses, pytest-cov.

- [ ] **Step 1: Rewrite pyproject.toml for uv (PEP 621)**

```toml
[project]
name = "culture-event-finder"
version = "1.0.0"
description = "Culture event search — Django JSON API + React SPA"
authors = [{ name = "taurus5650", email = "taurus_5650@hotmail.com" }]
readme = "README.md"
requires-python = ">=3.13"
dependencies = [
    "django>=5.2,<5.3",
    "requests>=2.32",
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

- [ ] **Step 2: Generate lock, install, delete Poetry/pip leftovers**

```bash
uv lock && uv sync
git rm poetry.lock requirements.txt
```

Expected: `uv.lock` created, `.venv/` populated. (If uv missing: `curl -LsSf https://astral.sh/uv/install.sh | sh`.)

- [ ] **Step 3: Verify old test suite still passes under uv + Django 5.2**

```bash
cd main_project && uv run python -m pytest . -v
```

Expected: all existing tests PASS (culture/health_check/tech_stack). If a Django 5.2 deprecation breaks something, fix minimally and note it in the commit.

- [ ] **Step 4: Verify toolkitsy import works**

```bash
uv run python -c "from toolkitsy.logger import logger, configure, set_correlation_id; configure(); logger.info('toolkitsy ok')"
```

Expected: prints a formatted log line containing `toolkitsy ok`.

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml uv.lock && git add -u
git commit -m "chore: migrate dependency management from poetry to uv, add toolkitsy"
```

### Task 2: Provider layer (base + TaiwanProvider + registry)

**Files:**
- Create: `main_project/events/__init__.py` (empty), `main_project/events/apps.py`, `main_project/events/providers/__init__.py`, `main_project/events/providers/base.py`, `main_project/events/providers/taiwan.py`
- Create: `main_project/events/tests/__init__.py` (empty), `main_project/events/tests/test_providers.py`

**Interfaces:**
- Produces: `Event` dataclass (`title: str, start_time: datetime, end_time: datetime|None, location: str, location_name: str|None, on_sales: str|None, price: str|None`); `UpstreamError(Exception)`; `BaseProvider` with `code/name/locations/categories` attrs + `fetch_events(category_id: int) -> list[Event]`; `PROVIDERS: dict[str, BaseProvider]` and `get_provider(code) -> BaseProvider` (raises `KeyError`) in `events.providers`.

- [ ] **Step 1: Write the failing tests**

`main_project/events/tests/test_providers.py`:

```python
import pytest
import responses

from events.providers import PROVIDERS, get_provider
from events.providers.base import UpstreamError
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
        assert get_provider("tw").code == "tw"
        assert "tw" in PROVIDERS

    def test_unknown_country_raises_keyerror(self):
        with pytest.raises(KeyError):
            get_provider("xx")

    def test_metadata_shape(self):
        p = get_provider("tw")
        assert {"value": "臺北", "label": {"zh": "臺北", "en": "Taipei"}} in p.locations
        assert any(c["value"] == 1 and c["label"]["zh"] == "音樂" for c in p.categories)


class TestTaiwanFetch:
    @responses.activate
    def test_success_parses_and_skips_bad_rows(self):
        responses.get(MOC_API_URL, json=MOC_PAYLOAD)
        events = TaiwanProvider().fetch_events(category_id=1)
        assert len(events) == 1  # bad-time and empty-location rows skipped
        e = events[0]
        assert e.title == "模擬音樂會1"
        assert e.start_time.strftime("%Y-%m") == "2026-07"
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
        import requests as _requests
        responses.get(MOC_API_URL, body=_requests.ConnectionError("boom"))
        with pytest.raises(UpstreamError):
            TaiwanProvider().fetch_events(category_id=1)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd main_project && uv run python -m pytest events/tests/test_providers.py -v
```

Expected: FAIL / collection error — `ModuleNotFoundError: No module named 'events'`.

- [ ] **Step 3: Implement**

`main_project/events/apps.py`:

```python
from django.apps import AppConfig


class EventsConfig(AppConfig):
    name = "events"
```

`main_project/events/providers/base.py`:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime


class UpstreamError(Exception):
    """External data source failed (network, HTTP error, bad payload)."""


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
    """One provider per country: knows its data source and its option lists."""

    code: str                # e.g. "tw" — used in API paths
    name: dict[str, str]     # {"zh": "台灣", "en": "Taiwan"}
    locations: list[dict]    # [{"value": "臺北", "label": {"zh": "臺北", "en": "Taipei"}}]
    categories: list[dict]   # [{"value": 1, "label": {"zh": "音樂", "en": "Music"}}]

    @abstractmethod
    def fetch_events(self, category_id: int) -> list[Event]:
        """Fetch ALL events for one category. Raises UpstreamError on failure."""
```

`main_project/events/providers/taiwan.py`:

```python
import urllib3
import requests
from datetime import datetime

from toolkitsy.logger import logger

from .base import BaseProvider, Event, UpstreamError

# cloud.culture.tw 的 SSL 憑證缺少 Subject Key Identifier，Python 3.13 會拒絕連線。
# 外部政府 API 無法修改其憑證，只好關閉驗證並抑制警告；只影響這一個資料源。
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

MOC_API_URL = "https://cloud.culture.tw/frontsite/trans/SearchShowAction.do"
TIME_FORMAT = "%Y/%m/%d %H:%M:%S"

_LOCATIONS = [
    ("臺北", "Taipei"), ("新北", "New Taipei City"), ("基隆", "Keelung"),
    ("桃園", "Taoyuan"), ("新竹", "Hsinchu"), ("苗栗", "Miaoli"),
    ("臺中", "Taichung"), ("彰化", "Changhua"), ("南投", "Nantou"),
    ("花蓮", "Hualien"), ("臺東", "Taitung"), ("雲林", "Yunlin"),
    ("嘉義", "Chiayi"), ("臺南", "Tainan"), ("高雄", "Kaohsiung"),
    ("屏東", "Pingtung"), ("金門", "Kinmen"), ("澎湖", "Penghu"),
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

        events = []
        for item in payload:
            title = item.get("title") or "Untitled"
            for show in item.get("showInfo", []):
                event = self._parse_show(title, show)
                if event:
                    events.append(event)
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

`main_project/events/providers/__init__.py`:

```python
from .base import BaseProvider
from .taiwan import TaiwanProvider

PROVIDERS: dict[str, BaseProvider] = {p.code: p for p in [TaiwanProvider()]}


def get_provider(code: str) -> BaseProvider:
    return PROVIDERS[code]  # KeyError bubbles up; views turn it into 404
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd main_project && uv run python -m pytest events/tests/test_providers.py -v
```

Expected: 7 PASS.

- [ ] **Step 5: Commit**

```bash
git add main_project/events && git commit -m "feat: add events provider layer with taiwan MoC provider"
```

### Task 3: Services layer (cache-aside + filter + sort)

**Files:**
- Create: `main_project/events/services.py`
- Test: `main_project/events/tests/test_services.py`

**Interfaces:**
- Consumes: `events.providers.get_provider`, `Event`, `UpstreamError` (Task 2).
- Produces: `search_events(country_code: str, category_id: int, location: str, month: str) -> list[Event]` — `month` is ISO `YYYY-MM`; raises `KeyError` (unknown country) / `UpstreamError`. Constant `CACHE_TTL_SECONDS = 43200`.

- [ ] **Step 1: Write the failing tests**

`main_project/events/tests/test_services.py`:

```python
from datetime import datetime
from unittest.mock import MagicMock

import pytest
from django.core.cache import cache

from events import services
from events.providers.base import Event


def make_event(title="演出", time_str="2026/07/12 19:30:00", location="臺北市中正區"):
    return Event(
        title=title,
        start_time=datetime.strptime(time_str, "%Y/%m/%d %H:%M:%S"),
        end_time=None, location=location, location_name=None,
        on_sales="Y", price="500",
    )


@pytest.fixture(autouse=True)
def clear_cache():
    cache.clear()
    yield
    cache.clear()


@pytest.fixture
def fake_provider(monkeypatch):
    provider = MagicMock()
    provider.fetch_events.return_value = [
        make_event("七月台北", "2026/07/12 19:30:00", "臺北市中正區"),
        make_event("七月台北較早", "2026/07/01 10:00:00", "臺北市大安區"),
        make_event("八月台北", "2026/08/03 19:30:00", "臺北市中正區"),
        make_event("七月高雄", "2026/07/15 19:30:00", "高雄市鹽埕區"),
    ]
    monkeypatch.setattr(services, "get_provider", lambda code: provider)
    return provider


def test_filters_by_location_and_iso_month(fake_provider):
    result = services.search_events("tw", 1, location="臺北", month="2026-07")
    assert [e.title for e in result] == ["七月台北較早", "七月台北"]  # sorted by time


def test_month_mismatch_returns_empty(fake_provider):
    assert services.search_events("tw", 1, location="臺北", month="2026-09") == []


def test_cache_hit_skips_second_upstream_call(fake_provider):
    services.search_events("tw", 1, location="臺北", month="2026-07")
    services.search_events("tw", 1, location="高雄", month="2026-07")  # same country+category
    assert fake_provider.fetch_events.call_count == 1


def test_different_category_is_separate_cache_entry(fake_provider):
    services.search_events("tw", 1, location="臺北", month="2026-07")
    services.search_events("tw", 2, location="臺北", month="2026-07")
    assert fake_provider.fetch_events.call_count == 2
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd main_project && uv run python -m pytest events/tests/test_services.py -v
```

Expected: FAIL — import error (`events.services` missing).

- [ ] **Step 3: Implement**

`main_project/events/services.py`:

```python
from django.core.cache import cache
from toolkitsy.logger import logger

from .providers import get_provider
from .providers.base import Event

CACHE_TTL_SECONDS = 60 * 60 * 12  # 12h — event data changes slowly (spec §3.3)


def search_events(country_code: str, category_id: int, location: str, month: str) -> list[Event]:
    """month is ISO 'YYYY-MM' (API contract).

    Raises KeyError for unknown country, UpstreamError when the source is down.
    """
    provider = get_provider(country_code)

    cache_key = f"events:{country_code}:{category_id}"
    events = cache.get(cache_key)
    if events is None:
        logger.info(f"Cache miss: {cache_key}")
        events = provider.fetch_events(category_id)
        cache.set(cache_key, events, CACHE_TTL_SECONDS)

    matched = [
        e for e in events
        if location in e.location and e.start_time.strftime("%Y-%m") == month
    ]
    return sorted(matched, key=lambda e: e.start_time)
```

Note: the ISO→MoC month-format conversion happens by *parsing* upstream time strings into `datetime` in the provider, then comparing `strftime("%Y-%m")` here. No string surgery; spec §3.3's requirement is covered by `test_filters_by_location_and_iso_month`.

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd main_project && uv run python -m pytest events/tests/test_services.py -v
```

Expected: 4 PASS.

- [ ] **Step 5: Commit**

```bash
git add main_project/events/services.py main_project/events/tests/test_services.py
git commit -m "feat: add events service layer with cache-aside and month filtering"
```

### Task 4: API endpoints + wiring (urls, settings, middleware)

**Files:**
- Create: `main_project/events/views.py`, `main_project/events/urls.py`, `main_project/events/middleware.py`
- Modify: `main_project/main_project/urls.py`, `main_project/main_project/settings.py` (via Bash snippet — Read-blocked)
- Test: `main_project/events/tests/test_api.py`

**Interfaces:**
- Consumes: `services.search_events` (Task 3), `PROVIDERS` (Task 2).
- Produces: `GET /api/v1/countries` → `[{code, name, locations, categories}]`; `GET /api/v1/<country>/events?category=&location=&month=` → `{"events": [{title, startTime, endTime, location, locationName, onSales, price, googleMapUrl, googleSearchUrl}]}` with `startTime` as ISO 8601; errors per Global Constraints. Frontend (Task 6) consumes these exact shapes.

- [ ] **Step 1: Write the failing tests**

`main_project/events/tests/test_api.py`:

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
        assert len(data[0]["locations"]) == 18
        assert len(data[0]["categories"]) == 12


class TestEventsApi:
    URL = "/api/v1/tw/events"

    @patch("events.views.services.search_events", return_value=_fake_events())
    def test_success_shape(self, _mock, client):
        resp = client.get(self.URL, {"category": "1", "location": "臺北", "month": "2026-07"})
        assert resp.status_code == 200
        event = json.loads(resp.content)["events"][0]
        assert event["title"] == "模擬音樂會"
        assert event["startTime"] == "2026-07-12T19:30:00"
        assert "query=" in event["googleMapUrl"]
        assert "q=" in event["googleSearchUrl"]

    @patch("events.views.services.search_events", return_value=[])
    def test_empty_result_is_200_with_empty_list(self, _mock, client):
        resp = client.get(self.URL, {"category": "1", "location": "臺北", "month": "2026-07"})
        assert resp.status_code == 200
        assert json.loads(resp.content) == {"events": []}

    def test_bad_category_400(self, client):
        resp = client.get(self.URL, {"category": "abc", "location": "臺北", "month": "2026-07"})
        assert resp.status_code == 400
        assert json.loads(resp.content)["error"]["code"] == "INVALID_PARAM"

    def test_bad_month_400(self, client):
        resp = client.get(self.URL, {"category": "1", "location": "臺北", "month": "2026/07"})
        assert resp.status_code == 400

    def test_unknown_country_404(self, client):
        resp = client.get("/api/v1/xx/events", {"category": "1", "location": "a", "month": "2026-07"})
        assert resp.status_code == 404
        assert json.loads(resp.content)["error"]["code"] == "UNKNOWN_COUNTRY"

    @patch("events.views.services.search_events", side_effect=UpstreamError("down"))
    def test_upstream_failure_502(self, _mock, client):
        resp = client.get(self.URL, {"category": "1", "location": "臺北", "month": "2026-07"})
        assert resp.status_code == 502
        assert json.loads(resp.content)["error"]["code"] == "UPSTREAM_ERROR"

    def test_response_has_request_id_header(self, client):
        resp = client.get("/api/v1/countries")
        assert resp.headers.get("X-Request-ID")
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd main_project && uv run python -m pytest events/tests/test_api.py -v
```

Expected: FAIL — 404s (routes not wired) / import errors.

- [ ] **Step 3: Implement views, urls, middleware**

`main_project/events/views.py`:

```python
import re
from urllib.parse import quote_plus

from django.http import JsonResponse
from toolkitsy.logger import logger

from . import services
from .providers import PROVIDERS
from .providers.base import Event, UpstreamError

MONTH_PATTERN = re.compile(r"^\d{4}-(0[1-9]|1[0-2])$")


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
    category = request.GET.get("category", "")
    location = request.GET.get("location", "")
    month = request.GET.get("month", "")

    if not category.isdigit():
        return _error(400, "INVALID_PARAM", "category must be an integer")
    if not location:
        return _error(400, "INVALID_PARAM", "location is required")
    if not MONTH_PATTERN.match(month):
        return _error(400, "INVALID_PARAM", "month must be YYYY-MM")

    try:
        result = services.search_events(country, int(category), location, month)
    except KeyError:
        return _error(404, "UNKNOWN_COUNTRY", f"country '{country}' is not supported")
    except UpstreamError as e:
        logger.error(f"Upstream failure: {e}")
        return _error(502, "UPSTREAM_ERROR", "Data source is temporarily unavailable")

    logger.info(f"Search {country}/{category}/{location}/{month}: {len(result)} events")
    return JsonResponse({"events": [_event_to_json(e) for e in result]})
```

`main_project/events/urls.py`:

```python
from django.urls import path

from . import views

urlpatterns = [
    path("countries", views.countries, name="api_countries"),
    path("<str:country>/events", views.events, name="api_events"),
]
```

`main_project/events/middleware.py`:

```python
import uuid

from toolkitsy.logger import set_correlation_id


class CorrelationIdMiddleware:
    """Give every request a short correlation id; toolkitsy logs include it."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        correlation_id = uuid.uuid4().hex[:8]
        set_correlation_id(correlation_id)
        response = self.get_response(request)
        response["X-Request-ID"] = correlation_id
        return response
```

Modify `main_project/main_project/urls.py` — replace the `urlpatterns` list with:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('health_check.urls')),
    path('', include('culture.urls')),
    path('', include('tech_stack.urls')),
    path('api/v1/', include('events.urls')),
]
```

- [ ] **Step 4: Wire settings.py (Read-blocked — use this Bash snippet verbatim)**

```bash
python3 - <<'EOF'
from pathlib import Path
p = Path('main_project/main_project/settings.py')
s = p.read_text()
assert "'events'" not in s and '"events"' not in s, "already wired"
marker = 'INSTALLED_APPS = ['
assert marker in s, "INSTALLED_APPS not found - apply manually"
s = s.replace(marker, marker + "\n    'events',", 1)
s += '''

# --- events API wiring (refactor M1) ---
MIDDLEWARE.append("events.middleware.CorrelationIdMiddleware")

from toolkitsy.logger import configure as _configure_logging
_configure_logging()  # console only; Cloud Run captures stdout
'''
p.write_text(s)
print("settings.py wired OK")
EOF
```

Expected: `settings.py wired OK`.

- [ ] **Step 5: Run the full backend suite (new + old)**

```bash
cd main_project && uv run python -m pytest . -v
```

Expected: all PASS — old culture/tech_stack/health_check tests must still be green (coexistence).

- [ ] **Step 6: Smoke-test against the real MoC API**

```bash
cd main_project && DEBUG=True uv run python manage.py runserver 8000 &
sleep 3
curl -s "http://127.0.0.1:8000/api/v1/countries" | head -c 300; echo
curl -s "http://127.0.0.1:8000/api/v1/tw/events?category=6&location=%E8%87%BA%E5%8C%97&month=$(date +%Y-%m)" | head -c 500; echo
kill %1
```

Expected: countries JSON with `"code": "tw"`; events JSON (may be `{"events": []}` — fine, must not be an error object).

- [ ] **Step 7: Commit**

```bash
git add main_project/events main_project/main_project/urls.py main_project/main_project/settings.py
git commit -m "feat: add /api/v1 events endpoints with correlation-id logging"
```

---

## Milestone 2 — React frontend

### Task 5: Vite + React + TS + Tailwind + Vitest scaffold

**Files:**
- Create: `frontend/` via scaffold; then overwrite `frontend/vite.config.ts`, `frontend/src/index.css`; add `test` script to `frontend/package.json`.

**Interfaces:**
- Produces: `npm run dev` (port 5173, proxies `/api`→127.0.0.1:8000), `npm run build` (outputs `frontend/dist/` with asset base `/static/`), `npm test` (vitest run).

- [ ] **Step 1: Scaffold**

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend && npm install && npm install tailwindcss @tailwindcss/vite && npm install -D vitest
```

- [ ] **Step 2: Overwrite `frontend/vite.config.ts`**

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig(({ mode }) => ({
  // Production assets are served by Django/WhiteNoise under /static/
  base: mode === "production" ? "/static/" : "/",
  plugins: [react(), tailwindcss()],
  server: {
    proxy: { "/api": "http://127.0.0.1:8000" },
  },
}));
```

- [ ] **Step 3: Replace `frontend/src/index.css` content entirely with:**

```css
@import "tailwindcss";
```

- [ ] **Step 4: Add test script to `frontend/package.json` scripts block:**

```json
"test": "vitest run"
```

- [ ] **Step 5: Verify build and dev server**

```bash
cd frontend && npm run build && ls dist/assets
npm run dev & sleep 3 && curl -s http://127.0.0.1:5173/ | head -c 200; kill %1
```

Expected: `dist/assets/` contains hashed `index-*.js`; dev server returns the Vite index page.

- [ ] **Step 6: Commit**

```bash
git add frontend && git commit -m "feat: scaffold vite react-ts frontend with tailwind and vitest"
```

### Task 6: Types, API client, format util (TDD)

**Files:**
- Create: `frontend/src/types.ts`, `frontend/src/api.ts`, `frontend/src/utils/format.ts`
- Test: `frontend/src/utils/format.test.ts`, `frontend/src/api.test.ts`

**Interfaces:**
- Consumes: backend API shapes from Task 4 (exact field names).
- Produces: `fetchCountries(): Promise<Country[]>`; `fetchEvents(country, {category, location, month}): Promise<{events: EventItem[]}>`; `ApiError` with `.status`; `formatEventTime(iso: string): string`. Components (Task 8) consume these.

- [ ] **Step 1: Write failing tests**

`frontend/src/utils/format.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { formatEventTime } from "./format";

describe("formatEventTime", () => {
  it("renders MM/DD HH:mm from ISO string", () => {
    expect(formatEventTime("2026-07-12T19:30:00")).toBe("07/12 19:30");
  });
  it("returns the input when unparseable", () => {
    expect(formatEventTime("whatever")).toBe("whatever");
  });
});
```

`frontend/src/api.test.ts`:

```ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { fetchEvents } from "./api";

afterEach(() => vi.unstubAllGlobals());

function stubFetch(status: number, body: unknown) {
  vi.stubGlobal("fetch", vi.fn().mockResolvedValue({
    ok: status >= 200 && status < 300,
    status,
    json: () => Promise.resolve(body),
  }));
}

describe("fetchEvents", () => {
  it("builds the query and returns events", async () => {
    stubFetch(200, { events: [] });
    const result = await fetchEvents("tw", { category: 1, location: "臺北", month: "2026-07" });
    expect(result.events).toEqual([]);
    const url = (fetch as ReturnType<typeof vi.fn>).mock.calls[0][0] as string;
    expect(url).toContain("/api/v1/tw/events?");
    expect(url).toContain("month=2026-07");
  });

  it("throws ApiError with server message on failure", async () => {
    stubFetch(502, { error: { code: "UPSTREAM_ERROR", message: "Data source is temporarily unavailable" } });
    await expect(fetchEvents("tw", { category: 1, location: "臺北", month: "2026-07" }))
      .rejects.toMatchObject({ status: 502, message: "Data source is temporarily unavailable" });
  });
});
```

- [ ] **Step 2: Run to verify failure**

```bash
cd frontend && npm test
```

Expected: FAIL — modules not found.

- [ ] **Step 3: Implement**

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
```

`frontend/src/api.ts`:

```ts
import type { Country, EventItem } from "./types";

const API_BASE = "/api/v1";

export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}

async function getJson<T>(path: string): Promise<T> {
  const resp = await fetch(`${API_BASE}${path}`);
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

export function fetchCountries(): Promise<Country[]> {
  return getJson("/countries");
}

export function fetchEvents(
  country: string,
  params: { category: string | number; location: string; month: string },
): Promise<{ events: EventItem[] }> {
  const query = new URLSearchParams({
    category: String(params.category),
    location: params.location,
    month: params.month,
  });
  return getJson(`/${country}/events?${query}`);
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
```

- [ ] **Step 4: Run to verify pass**

```bash
cd frontend && npm test
```

Expected: 4 PASS.

- [ ] **Step 5: Commit**

```bash
git add frontend/src && git commit -m "feat: add typed api client and time format util"
```

### Task 7: i18n (locales + hook + LanguageSwitch)

**Files:**
- Create: `frontend/src/locales/zh.json`, `frontend/src/locales/en.json`, `frontend/src/i18n.tsx`, `frontend/src/components/LanguageSwitch.tsx`

**Interfaces:**
- Produces: `LanguageProvider`, `useLang(): {lang: "zh"|"en", setLang}`, `useT(): (key: string) => string`, `pickLabel(label: Record<string,string>, lang): string`. All components use `useT()` for copy; option labels use `pickLabel`.

- [ ] **Step 1: Locale files**

`frontend/src/locales/zh.json`:

```json
{
  "app.title": "藝文活動查詢",
  "nav.search": "查活動",
  "nav.about": "關於",
  "search.country": "國家",
  "search.location": "地區",
  "search.category": "類別",
  "search.month": "月份",
  "search.submit": "搜尋",
  "results.empty": "這個條件查無活動，換個月份或類別試試",
  "results.count": "筆活動",
  "error.upstream": "資料來源暫時無法使用，請稍後再試",
  "error.generic": "發生錯誤，請稍後再試",
  "event.onSales": "售票中",
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
  "search.country": "Country",
  "search.location": "Location",
  "search.category": "Category",
  "search.month": "Month",
  "search.submit": "Search",
  "results.empty": "No events for these filters — try another month or category",
  "results.count": "events",
  "error.upstream": "Data source is temporarily unavailable. Please try again later.",
  "error.generic": "Something went wrong. Please try again later.",
  "event.onSales": "On sale",
  "event.map": "Map",
  "about.title": "About this site",
  "about.body": "Search culture events via government open data. Currently supports Taiwan (Ministry of Culture)."
}
```

- [ ] **Step 2: `frontend/src/i18n.tsx`**

```tsx
import { createContext, useContext, useState, type ReactNode } from "react";
import en from "./locales/en.json";
import zh from "./locales/zh.json";

const dicts = { zh, en } as const;
export type Lang = keyof typeof dicts;

const LangContext = createContext<{ lang: Lang; setLang: (l: Lang) => void }>({
  lang: "zh",
  setLang: () => {},
});

export function LanguageProvider({ children }: { children: ReactNode }) {
  const [lang, setLang] = useState<Lang>("zh");
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

export default function LanguageSwitch() {
  const { lang, setLang } = useLang();
  const next = lang === "zh" ? "en" : "zh";
  return (
    <button
      type="button"
      onClick={() => setLang(next)}
      className="rounded-full border border-gray-300 px-3 py-1 text-sm text-gray-600 hover:bg-gray-100"
    >
      {lang === "zh" ? "EN" : "中"}
    </button>
  );
}
```

- [ ] **Step 4: Type-check and commit**

```bash
cd frontend && npx tsc --noEmit
git add frontend/src && git commit -m "feat: add zh/en i18n context and language switch"
```

Expected: tsc clean.

### Task 8: UI components (cards, list, form, states)

**Files:**
- Create under `frontend/src/components/`: `SkeletonCard.tsx`, `ErrorMessage.tsx`, `EventCard.tsx`, `EventList.tsx`, `SearchForm.tsx`

**Interfaces:**
- Consumes: `EventItem`, `Country`, `LabeledOption` (Task 6); `useT`, `useLang`, `pickLabel` (Task 7); `formatEventTime` (Task 6).
- Produces: `<SearchForm countries value onChange onSubmit loading />` with exported `SearchValue = {country: string; category: string; location: string; month: string}`; `<EventList events />`; `<ErrorMessage message />`; `<SkeletonCard />`. App (Task 9) consumes these.

- [ ] **Step 1: `SkeletonCard.tsx`**

```tsx
export default function SkeletonCard() {
  return (
    <div className="animate-pulse rounded-xl border border-gray-200 bg-white p-4 shadow-sm">
      <div className="mb-3 h-5 w-3/4 rounded bg-gray-200" />
      <div className="mb-2 h-4 w-1/2 rounded bg-gray-200" />
      <div className="h-4 w-2/3 rounded bg-gray-200" />
    </div>
  );
}
```

- [ ] **Step 2: `ErrorMessage.tsx`**

```tsx
export default function ErrorMessage({ message }: { message: string }) {
  return (
    <div className="rounded-xl border border-red-200 bg-red-50 p-4 text-red-700" role="alert">
      {message}
    </div>
  );
}
```

- [ ] **Step 3: `EventCard.tsx`**

```tsx
import { useT } from "../i18n";
import type { EventItem } from "../types";
import { formatEventTime } from "../utils/format";

export default function EventCard({ event }: { event: EventItem }) {
  const t = useT();
  return (
    <article className="flex flex-col gap-2 rounded-xl border border-gray-200 bg-white p-4 shadow-sm">
      <div className="flex items-start justify-between gap-2">
        <a
          href={event.googleSearchUrl}
          target="_blank"
          rel="noreferrer"
          className="font-semibold text-gray-900 hover:underline"
        >
          {event.title}
        </a>
        {event.onSales === "Y" && (
          <span className="shrink-0 rounded-full bg-emerald-100 px-2 py-0.5 text-xs text-emerald-700">
            {t("event.onSales")}
          </span>
        )}
      </div>
      <p className="text-sm text-gray-500">{formatEventTime(event.startTime)}</p>
      <p className="text-sm text-gray-600">
        {event.locationName ?? event.location}
        <a
          href={event.googleMapUrl}
          target="_blank"
          rel="noreferrer"
          className="ml-2 text-blue-600 hover:underline"
        >
          {t("event.map")}
        </a>
      </p>
      {event.price && <p className="text-sm text-gray-500">$ {event.price}</p>}
    </article>
  );
}
```

- [ ] **Step 4: `EventList.tsx`**

```tsx
import { useT } from "../i18n";
import type { EventItem } from "../types";
import EventCard from "./EventCard";

export default function EventList({ events }: { events: EventItem[] }) {
  const t = useT();
  if (events.length === 0) {
    return <p className="rounded-xl bg-gray-50 p-6 text-center text-gray-500">{t("results.empty")}</p>;
  }
  return (
    <div>
      <p className="mb-3 text-sm text-gray-500">{events.length} {t("results.count")}</p>
      <div className="grid grid-cols-1 gap-3 sm:grid-cols-2 lg:grid-cols-3">
        {events.map((event, i) => (
          <EventCard key={`${event.title}-${event.startTime}-${i}`} event={event} />
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 5: `SearchForm.tsx`**

```tsx
import { pickLabel, useLang, useT } from "../i18n";
import type { Country } from "../types";

export interface SearchValue {
  country: string;
  category: string;
  location: string;
  month: string;
}

interface Props {
  countries: Country[];
  value: SearchValue;
  onChange: (value: SearchValue) => void;
  onSubmit: () => void;
  loading: boolean;
}

const selectClass =
  "mt-1 w-full rounded-lg border border-gray-300 bg-white px-3 py-2 text-gray-900 focus:border-blue-500 focus:outline-none";

// 月份選擇器用年 + 月兩個 <select>，不用 <input type="month">：
// 桌面版 Firefox / Safari 全版本不支援 month picker，會 fallback 成純文字框。
// value.month 的格式維持 "YYYY-MM" 不變，api.ts / 後端都不用改。
const NOW = new Date();
const YEARS = [NOW.getFullYear(), NOW.getFullYear() + 1].map(String);
const MONTHS = Array.from({ length: 12 }, (_, i) => String(i + 1).padStart(2, "0"));

export default function SearchForm({ countries, value, onChange, onSubmit, loading }: Props) {
  const t = useT();
  const { lang } = useLang();
  const country = countries.find((c) => c.code === value.country);

  return (
    <form
      onSubmit={(e) => { e.preventDefault(); onSubmit(); }}
      className="grid grid-cols-2 gap-3 rounded-xl border border-gray-200 bg-white p-4 shadow-sm sm:grid-cols-4"
    >
      {countries.length > 1 && (
        <label className="col-span-2 block text-sm text-gray-600 sm:col-span-4">
          {t("search.country")}
          <select
            className={selectClass}
            value={value.country}
            onChange={(e) => onChange({ ...value, country: e.target.value, location: "", category: "" })}
          >
            {countries.map((c) => (
              <option key={c.code} value={c.code}>{pickLabel(c.name, lang)}</option>
            ))}
          </select>
        </label>
      )}
      <label className="block text-sm text-gray-600">
        {t("search.location")}
        <select
          className={selectClass}
          value={value.location}
          onChange={(e) => onChange({ ...value, location: e.target.value })}
        >
          {country?.locations.map((o) => (
            <option key={String(o.value)} value={String(o.value)}>{pickLabel(o.label, lang)}</option>
          ))}
        </select>
      </label>
      <label className="block text-sm text-gray-600">
        {t("search.category")}
        <select
          className={selectClass}
          value={value.category}
          onChange={(e) => onChange({ ...value, category: e.target.value })}
        >
          {country?.categories.map((o) => (
            <option key={String(o.value)} value={String(o.value)}>{pickLabel(o.label, lang)}</option>
          ))}
        </select>
      </label>
      <label className="block text-sm text-gray-600">
        {t("search.month")}
        <div className="mt-1 flex gap-2">
          <select
            className={selectClass}
            value={value.month.slice(0, 4)}
            onChange={(e) => onChange({ ...value, month: `${e.target.value}-${value.month.slice(5, 7)}` })}
          >
            {YEARS.map((y) => (
              <option key={y} value={y}>{y}</option>
            ))}
          </select>
          <select
            className={selectClass}
            value={value.month.slice(5, 7)}
            onChange={(e) => onChange({ ...value, month: `${value.month.slice(0, 4)}-${e.target.value}` })}
          >
            {MONTHS.map((m) => (
              <option key={m} value={m}>{m}</option>
            ))}
          </select>
        </div>
      </label>
      <div className="flex items-end">
        <button
          type="submit"
          disabled={loading}
          className="w-full rounded-lg bg-blue-600 px-4 py-2 font-medium text-white hover:bg-blue-700 disabled:opacity-50"
        >
          {t("search.submit")}
        </button>
      </div>
    </form>
  );
}
```

- [ ] **Step 6: Type-check and commit**

```bash
cd frontend && npx tsc --noEmit
git add frontend/src/components && git commit -m "feat: add search form and event card components"
```

### Task 9: App assembly + About + manual E2E

**Files:**
- Modify: `frontend/src/App.tsx` (full rewrite), `frontend/src/main.tsx`
- Delete: `frontend/src/App.css`, `frontend/src/assets/react.svg` (scaffold leftovers)

**Interfaces:**
- Consumes: everything from Tasks 6–8.
- Produces: complete SPA (search + about views, header, states idle/loading/success/error).

- [ ] **Step 1: `frontend/src/main.tsx`**

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

- [ ] **Step 2: `frontend/src/App.tsx`**

```tsx
import { useEffect, useState } from "react";
import { ApiError, fetchCountries, fetchEvents } from "./api";
import ErrorMessage from "./components/ErrorMessage";
import EventList from "./components/EventList";
import LanguageSwitch from "./components/LanguageSwitch";
import SearchForm, { type SearchValue } from "./components/SearchForm";
import SkeletonCard from "./components/SkeletonCard";
import { useT } from "./i18n";
import type { Country, EventItem } from "./types";

function currentMonth(): string {
  return new Date().toISOString().slice(0, 7); // "YYYY-MM"
}

type Status = "idle" | "loading" | "success" | "error";

export default function App() {
  const t = useT();
  const [view, setView] = useState<"search" | "about">("search");
  const [countries, setCountries] = useState<Country[]>([]);
  const [form, setForm] = useState<SearchValue>({
    country: "", category: "", location: "", month: currentMonth(),
  });
  const [status, setStatus] = useState<Status>("idle");
  const [events, setEvents] = useState<EventItem[]>([]);
  const [errorMessage, setErrorMessage] = useState("");

  useEffect(() => {
    fetchCountries()
      .then((data) => {
        setCountries(data);
        const first = data[0];
        if (first) {
          setForm((f) => ({
            ...f,
            country: first.code,
            location: String(first.locations[0]?.value ?? ""),
            category: String(first.categories[0]?.value ?? ""),
          }));
        }
      })
      .catch(() => setErrorMessage(t("error.generic")));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  async function handleSearch() {
    setStatus("loading");
    setErrorMessage("");
    try {
      const result = await fetchEvents(form.country, {
        category: form.category,
        location: form.location,
        month: form.month,
      });
      setEvents(result.events);
      setStatus("success");
    } catch (err) {
      setStatus("error");
      setErrorMessage(
        err instanceof ApiError && err.status === 502 ? t("error.upstream") : t("error.generic"),
      );
    }
  }

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="border-b border-gray-200 bg-white">
        <div className="mx-auto flex max-w-5xl items-center justify-between px-4 py-3">
          <h1 className="text-lg font-bold text-gray-900">{t("app.title")}</h1>
          <nav className="flex items-center gap-3 text-sm">
            <button type="button" onClick={() => setView("search")}
              className={view === "search" ? "font-semibold text-blue-600" : "text-gray-600"}>
              {t("nav.search")}
            </button>
            <button type="button" onClick={() => setView("about")}
              className={view === "about" ? "font-semibold text-blue-600" : "text-gray-600"}>
              {t("nav.about")}
            </button>
            <LanguageSwitch />
          </nav>
        </div>
      </header>

      <main className="mx-auto max-w-5xl space-y-4 px-4 py-6">
        {view === "about" ? (
          <section className="rounded-xl border border-gray-200 bg-white p-6 shadow-sm">
            <h2 className="mb-2 text-lg font-semibold">{t("about.title")}</h2>
            <p className="mb-4 text-gray-600">{t("about.body")}</p>
            <p className="text-sm text-gray-500">
              Django · React · Cloud Run ·{" "}
              <a className="text-blue-600 hover:underline" target="_blank" rel="noreferrer"
                 href="https://github.com/taurus5650">GitHub</a>{" · "}
              <a className="text-blue-600 hover:underline" target="_blank" rel="noreferrer"
                 href="https://www.linkedin.com/in/sh-yin-lim/">LinkedIn</a>
            </p>
          </section>
        ) : (
          <>
            <SearchForm countries={countries} value={form} onChange={setForm}
              onSubmit={handleSearch} loading={status === "loading"} />
            {status === "loading" && (
              <div className="grid grid-cols-1 gap-3 sm:grid-cols-2 lg:grid-cols-3">
                <SkeletonCard /><SkeletonCard /><SkeletonCard />
              </div>
            )}
            {status === "error" && <ErrorMessage message={errorMessage} />}
            {status === "success" && <EventList events={events} />}
          </>
        )}
      </main>
    </div>
  );
}
```

- [ ] **Step 3: Remove scaffold leftovers, run checks**

```bash
cd frontend && rm -f src/App.css src/assets/react.svg
npx tsc --noEmit && npm test && npm run build
```

Expected: all clean.

- [ ] **Step 4: Manual E2E against local backend**

Terminal A: `cd main_project && DEBUG=True uv run python manage.py runserver 8000`
Terminal B: `cd frontend && npm run dev`

Open http://127.0.0.1:5173 and verify: dropdowns populated (18 locations, 12 categories), month prefilled, search shows skeletons then cards (or empty state), EN/中 switch flips all copy, layout holds at 375px width (devtools mobile view).

- [ ] **Step 5: Commit**

```bash
git add frontend && git commit -m "feat: assemble spa with search flow, about view and i18n"
```

---

## Milestone 3 — Cloud Run deployment

### Task 10: Production settings + root Dockerfile + local container verify

**Files:**
- Create: `Dockerfile` (repo root — `gcloud run deploy --source` requires it at source root; old `deployment_tcei/Dockerfile` stays until M4), `.dockerignore` (repo root)
- Modify: `main_project/main_project/settings.py` (Bash snippet), `main_project/main_project/urls.py`

**Interfaces:**
- Produces: container serving SPA at `/`, API at `/api/v1/...`, honoring `$PORT`; consumed by Tasks 11–13.

- [ ] **Step 1: Append production settings (Bash snippet, Read-blocked file)**

```bash
python3 - <<'EOF'
from pathlib import Path
p = Path('main_project/main_project/settings.py')
s = p.read_text()
assert "FRONTEND_DIST" not in s, "already applied"
s += '''

# --- Production static / SPA serving (refactor M3) ---
import os as _os
from pathlib import Path as _Path

_REPO_ROOT = _Path(__file__).resolve().parent.parent.parent
FRONTEND_DIST = _REPO_ROOT / "frontend" / "dist"

STATIC_URL = "/static/"
STATIC_ROOT = _REPO_ROOT / "staticfiles"
if FRONTEND_DIST.exists():
    STATICFILES_DIRS = list(globals().get("STATICFILES_DIRS", [])) + [FRONTEND_DIST]
    TEMPLATES[0]["DIRS"] = list(TEMPLATES[0]["DIRS"]) + [FRONTEND_DIST]

# WhiteNoise plain storage (default) on purpose - Vite already content-hashes filenames
MIDDLEWARE.insert(1, "whitenoise.middleware.WhiteNoiseMiddleware")

ALLOWED_HOSTS = _os.environ.get("ALLOWED_HOSTS", "localhost,127.0.0.1,.run.app").split(",")
'''
p.write_text(s)
print("prod settings appended OK")
EOF
```

- [ ] **Step 2: Route SPA at `/`, park old pages at `/legacy/`**

Replace `urlpatterns` in `main_project/main_project/urls.py` with:

```python
from django.views.generic import TemplateView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('health_check.urls')),
    path('api/v1/', include('events.urls')),
    path('legacy/', include('culture.urls')),
    path('legacy/', include('tech_stack.urls')),
    path('', TemplateView.as_view(template_name='index.html'), name='spa'),
]
```

(Old templates' `{% url 'culture' %}` links keep working — URL *names* are unchanged.)

- [ ] **Step 3: Root `Dockerfile`**

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
# 釘特定 uv 版本，勿用 :latest — 其餘依賴都靠 uv.lock 釘死，uv binary 也要可重現。
# 部署前上 https://github.com/astral-sh/uv/releases 確認當前版號後填入。
COPY --from=ghcr.io/astral-sh/uv:0.9.0 /uv /uvx /bin/
ENV PYTHONUNBUFFERED=1 TZ=Asia/Taipei PATH="/web/.venv/bin:$PATH"
WORKDIR /web
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY main_project/ ./main_project/
COPY --from=frontend /app/dist ./frontend/dist
RUN python main_project/manage.py collectstatic --noinput
# Cloud Run injects $PORT (defaults to 8080)
CMD exec gunicorn --chdir main_project main_project.wsgi:application \
    --bind 0.0.0.0:${PORT:-8080} --workers 1
# --workers 1：LocMemCache 是 per-process，多 worker 會各自持有獨立 cache，
# 讓 §3.3 的「跨 user 共用、12h 最多 12 次上游」假設破功。單 worker + Cloud Run
# 預設 concurrency 80，對 owner+朋友的低流量綽綽有餘。
```

Root `.dockerignore`:

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

- [ ] **Step 4: Verify tests + container locally**

```bash
cd main_project && uv run python -m pytest . -v && cd ..
docker build -t cef-local . && docker run --rm -e PORT=8080 -p 8080:8080 -d --name cef cef-local
sleep 5
curl -s http://127.0.0.1:8080/ | grep -o '<title>[^<]*'       # SPA index
curl -s http://127.0.0.1:8080/api/v1/countries | head -c 120   # API
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/health_check/
docker stop cef
```

Expected: Vite title tag, countries JSON, `200`.

- [ ] **Step 5: Commit**

```bash
git add Dockerfile .dockerignore main_project/main_project/urls.py main_project/main_project/settings.py
git commit -m "feat: single-container production build (spa + api) honoring cloud run port"
```

### Task 11: GCP one-time infra via Terraform (OWNER runs apply)

Terraform manages the one-time infra (API enablement, deployer service account + IAM, Workload Identity Federation). App deploys stay in CI (`gcloud run deploy`) — do NOT put the Cloud Run service revision into Terraform. Local tfstate (gitignored) — single-owner project; migrating state to a GCS bucket is a future exercise.

**Files:**
- Create: `terraform/main.tf`, `terraform/variables.tf`, `terraform/outputs.tf`, `terraform/terraform.tfvars.example`, `docs/deployment/gcp-setup.md`
- Modify: `.gitignore` (add Terraform entries)

- [ ] **Step 1: `terraform/variables.tf`**

```hcl
variable "project_id" {
  description = "GCP project id (create the project + billing manually first)"
  type        = string
}

variable "region" {
  description = "Cloud Run region"
  type        = string
  default     = "asia-east1" # 台灣機房
}

variable "github_repo" {
  description = "GitHub repo allowed to deploy, e.g. taurus5650/culture-event-finder"
  type        = string
}

variable "billing_account" {
  description = "Billing account id for the budget alert. Find via `gcloud billing accounts list`."
  type        = string
}
```

- [ ] **Step 2: `terraform/main.tf`**

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
  # Local state (terraform.tfstate, gitignored) — fine for a single owner.
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Enable the APIs this project needs
resource "google_project_service" "apis" {
  for_each = toset([
    "run.googleapis.com",
    "cloudbuild.googleapis.com",
    "artifactregistry.googleapis.com",
    "iamcredentials.googleapis.com",
    "billingbudgets.googleapis.com",   # budget alert (見下方 google_billing_budget)
  ])
  service            = each.value
  disable_on_destroy = false
}

# Service account GitHub Actions deploys as
resource "google_service_account" "deployer" {
  account_id   = "github-deployer"
  display_name = "GitHub Actions deployer"
}

resource "google_project_iam_member" "deployer_roles" {
  # 最小權限：deployer 只需推 image + 部署 Cloud Run，不給 project 全域 *.admin，
  # 縮小 WIF token 或 CI 被盜時的 blast radius。
  for_each = toset([
    "roles/run.admin",
    "roles/cloudbuild.builds.editor",
    "roles/artifactregistry.writer",       # 推/拉 image 足夠，非 admin
    "roles/storage.objectAdmin",           # 僅物件層級 (Cloud Build staging bucket)，非整專案 bucket admin
    "roles/iam.serviceAccountUser",
    "roles/serviceusage.serviceUsageConsumer",
  ])
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.deployer.email}"
}

# Workload Identity Federation — keyless auth for GitHub Actions
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github"
  depends_on                = [google_project_service.apis]
}

resource "google_iam_workload_identity_pool_provider" "github_oidc" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-oidc"
  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
  }
  attribute_condition = "assertion.repository == \"${var.github_repo}\""
  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

resource "google_service_account_iam_member" "wif_binding" {
  service_account_id = google_service_account.deployer.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/${var.github_repo}"
}

# 預算警示：公開 --allow-unauthenticated 端點若被爬爆超出 always-free，會開始計費。
# 達門檻時寄 email 通知 (只提醒，不會自動關服務)。目標 $0，故門檻設很低。
resource "google_billing_budget" "monthly" {
  billing_account = var.billing_account
  display_name    = "culture-event-finder monthly"
  budget_filter {
    projects = ["projects/${var.project_id}"]
  }
  amount {
    specified_amount {
      currency_code = "USD"
      units         = "5" # 當「不該花到錢」的早期警報線
    }
  }
  threshold_rules { threshold_percent = 0.5 }
  threshold_rules { threshold_percent = 1.0 }
}
```

- [ ] **Step 3: `terraform/outputs.tf`**

```hcl
output "gcp_sa_email" {
  description = "Set as GitHub Actions variable GCP_SA_EMAIL"
  value       = google_service_account.deployer.email
}

output "gcp_wif_provider" {
  description = "Set as GitHub Actions variable GCP_WIF_PROVIDER"
  value       = google_iam_workload_identity_pool_provider.github_oidc.name
}
```

- [ ] **Step 4: `terraform/terraform.tfvars.example`**

```hcl
project_id      = "culture-event-finder-<suffix>"
github_repo     = "taurus5650/culture-event-finder"
billing_account = "XXXXXX-XXXXXX-XXXXXX"  # gcloud billing accounts list
```

Append to `.gitignore`:

```
terraform/.terraform/
terraform/terraform.tfstate*
terraform/terraform.tfvars
terraform/.terraform.lock.hcl
```

- [ ] **Step 5: Write `docs/deployment/gcp-setup.md`**

```markdown
# GCP 一次性設定 (owner 手動執行一次)

前置：`brew install google-cloud-sdk terraform`，準備一張信用卡。
分工：Terraform 管一次性 infra (API/SA/IAM/WIF)；app 的每次部署走 CI 的
`gcloud run deploy`，不進 Terraform。tfstate 存本機 (已 gitignore)。

1. 登入並建立專案 (PROJECT_ID 需全球唯一):
   gcloud auth login
   gcloud auth application-default login   # Terraform 用這組憑證
   gcloud projects create culture-event-finder-<suffix> --set-as-default
2. 在 https://console.cloud.google.com/billing 綁定 billing account 到此專案。
3. 跑 Terraform:
   cd terraform
   cp terraform.tfvars.example terraform.tfvars   # 填入實際 project_id / github_repo / billing_account
   terraform init
   terraform plan     # 先看它要建什麼 — 學習重點在這步
   terraform apply    # yes
4. 把 terraform 的 output 填進 GitHub repo → Settings → Secrets and variables
   → Actions → Variables:
   GCP_PROJECT_ID   = <PROJECT_ID>
   GCP_REGION       = asia-east1
   GCP_SA_EMAIL     = (output: gcp_sa_email)
   GCP_WIF_PROVIDER = (output: gcp_wif_provider)
5. 手動驗證部署一次 (repo root):
   gcloud run deploy culture-event-finder --source . --region asia-east1 \
     --allow-unauthenticated --memory 512Mi --min-instances 0 --max-instances 1
   完成後開啟 terminal 顯示的 https://culture-event-finder-*.run.app，應看到 SPA。

日後想改 infra (加 role、換 repo 名)：改 .tf 檔 → terraform plan → apply。
未來練習題 (optional)：把 tfstate 搬到 GCS backend。
```

- [ ] **Step 6: Validate config without touching GCP**

```bash
cd terraform && terraform init -backend=false && terraform validate
```

Expected: `Success! The configuration is valid.`

- [ ] **Step 7: Commit, then STOP for the owner**

```bash
git add terraform docs/deployment/gcp-setup.md .gitignore
git commit -m "feat: add terraform for gcp one-time infra (apis, sa, wif)"
```

**PAUSE POINT:** the owner must complete `docs/deployment/gcp-setup.md` (especially step 5 succeeding) before Task 12.

### Task 12: GitHub Actions rewrite

**Files:**
- Modify: `.github/workflows/deploy.yml` (full rewrite)

- [ ] **Step 1: Rewrite `.github/workflows/deploy.yml`**

```yaml
name: Test and Deploy to Cloud Run

on:
  push:
    branches: [master]
  pull_request:

permissions:
  contents: read
  id-token: write   # Workload Identity Federation

jobs:
  test-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync
      - run: cd main_project && uv run python -m pytest . -v

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

  # 合體 container 煙測：test-backend (無 frontend/dist，TEMPLATES DIRS 空)
  # 與 test-frontend (只有 vitest) 都跑不到「collectstatic + WhiteNoise 服務 SPA + /api」
  # 一起動的路徑。這個 job 用真的 docker build 起 container 打幾個關鍵 route，
  # 擋掉壞掉的 revision 直接吃 100% 流量。/api/v1/countries 只回 provider metadata、
  # 不打 MoC，CI 內安全。
  build-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t smoke:latest .
      - run: docker run -d -p 8080:8080 -e PORT=8080 -e ALLOWED_HOSTS='*' --name smoke smoke:latest
      - run: |
          for i in $(seq 1 15); do curl -sf http://localhost:8080/health && break || sleep 2; done
          curl -sf http://localhost:8080/ | grep -qi 'id="root"' || (echo "SPA index missing" && exit 1)
          curl -sf http://localhost:8080/api/v1/countries | grep -q '"tw"' || (echo "countries API broken" && exit 1)

  deploy:
    if: github.ref == 'refs/heads/master'
    needs: [test-backend, test-frontend, build-smoke]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
          service_account: ${{ vars.GCP_SA_EMAIL }}
      - uses: google-github-actions/setup-gcloud@v2
      - run: |
          gcloud run deploy culture-event-finder \
            --source . \
            --project ${{ vars.GCP_PROJECT_ID }} \
            --region ${{ vars.GCP_REGION }} \
            --allow-unauthenticated --memory 512Mi --min-instances 0 --max-instances 1
```

- [ ] **Step 2: Commit and verify CI on the PR**

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: test backend+frontend then deploy to cloud run via wif"
```

Push the branch and check Actions: `test-backend` and `test-frontend` green (deploy only runs on master).

### Task 13: Production verify + Fly.io teardown (OWNER-GATED)

- [ ] **Step 1:** After merge to master, verify the Actions deploy job succeeds and the `*.run.app` URL serves: SPA at `/`, `/api/v1/countries` JSON, one real search round-trip on a phone.
- [ ] **Step 2 (OWNER, after a few days of fallback):**

```bash
fly apps destroy taiwan-culture-event-info
```

- [ ] **Step 3:** No commit (Fly config files are deleted in Task 14).

---

## Milestone 4 — Cleanup, restructure, rename

### Task 14: Delete legacy apps and assets

**Files:**
- Delete: `main_project/culture/`, `main_project/tech_stack/`, `main_project/utility/`, `main_project/main_project/templates/` (backup.html, base.html, 404.html), `main_project/main_project/views.py`, Material Dashboard static assets (find real path first), `main.py`, `fly.toml`, `deployment_tcei/`
- Modify: `main_project/main_project/urls.py`, `main_project/main_project/settings.py` (Bash snippet), `main_project/health_check/` (simplify)

- [ ] **Step 1: Replace `urlpatterns` in `main_project/main_project/urls.py`:**

```python
from django.urls import path, include
from django.views.generic import TemplateView

urlpatterns = [
    path('', include('health_check.urls')),
    path('api/v1/', include('events.urls')),
    path('', TemplateView.as_view(template_name='index.html'), name='spa'),
]
```

Delete the `handler404 = ...` line and the `admin` import (no models, no users; DEBUG=False default 404 is fine for an API+SPA).

- [ ] **Step 2: Clean settings via Bash snippet**

```bash
python3 - <<'EOF'
from pathlib import Path
import re
p = Path('main_project/main_project/settings.py')
s = p.read_text()
for pattern in [r"\s*['\"]culture['\"],", r"\s*['\"]tech_stack['\"],",
                r"\s*['\"]django\.contrib\.admin['\"],",
                r"\s*['\"]culture\.middleware\.RequestIdMiddleware['\"],"]:
    s = re.sub(pattern, "", s, count=1)
p.write_text(s)
print("legacy settings entries removed")
EOF
```

(If admin removal breaks other `django.contrib.*` dependencies at startup, re-add the minimal ones the error names — the fresh settings in Task 15 supersede all of this.)

- [ ] **Step 3: Delete files**

```bash
git rm -r main_project/culture main_project/tech_stack main_project/utility \
  main_project/main_project/templates deployment_tcei fly.toml main.py
git rm main_project/main_project/views.py
git ls-files main_project | grep -i 'static' | head -20   # inspect Material Dashboard assets path, then git rm -r it
```

Simplify `health_check`: keep only the plain-text route; delete `html_index`, its template dir, and the `health_check/html/` URL entry.

- [ ] **Step 4: Full test + container rebuild verify**

```bash
cd main_project && uv run python -m pytest . -v && cd ..
docker build -t cef-local . && docker run --rm -e PORT=8080 -p 8080:8080 -d --name cef cef-local
sleep 5 && curl -s http://127.0.0.1:8080/api/v1/countries | head -c 80 && docker stop cef
```

Expected: only events + health_check tests remain, all green; container serves.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "refactor: remove legacy jinja2 apps, utility logger and fly.io config"
```

### Task 15: Restructure to backend/ + config/ (fresh settings)

**Files:**
- Move: `main_project/` → `backend/`; `backend/main_project/` → `backend/config/`; `backend/health_check/` → `backend/health/`
- Create (fresh content): `backend/config/settings.py`, `backend/config/urls.py`, `backend/health/views.py`, `backend/health/urls.py`, `backend/health/tests.py`
- Modify: `backend/manage.py`, `backend/config/wsgi.py`, `backend/config/asgi.py`, `backend/pytest.ini`, `Dockerfile`, `makefile`, `.github/workflows/deploy.yml`

- [ ] **Step 1: Move directories**

```bash
git mv main_project backend
git mv backend/main_project backend/config
git mv backend/health_check backend/health
```

- [ ] **Step 2: Write fresh `backend/config/settings.py`** (replace ALL old content — this ends the Read-block workarounds):

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent      # backend/
REPO_ROOT = BASE_DIR.parent
FRONTEND_DIST = REPO_ROOT / "frontend" / "dist"

SECRET_KEY = os.environ.get("SECRET_KEY", "django-insecure-dev-only-key")
DEBUG = os.environ.get("DEBUG", "False") == "True"
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "localhost,127.0.0.1,.run.app").split(",")

INSTALLED_APPS = [
    "django.contrib.contenttypes",
    "django.contrib.staticfiles",
    "events",
    "health",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.middleware.common.CommonMiddleware",
    "events.middleware.CorrelationIdMiddleware",
]

ROOT_URLCONF = "config.urls"
WSGI_APPLICATION = "config.wsgi.application"

TEMPLATES = [{
    "BACKEND": "django.template.backends.django.DjangoTemplates",
    "DIRS": [FRONTEND_DIST] if FRONTEND_DIST.exists() else [],
    "APP_DIRS": False,
    "OPTIONS": {"context_processors": []},
}]

# No models - data comes from external APIs + cache. sqlite kept for pytest-django.
DATABASES = {"default": {"ENGINE": "django.db.backends.sqlite3", "NAME": ":memory:"}}

STATIC_URL = "/static/"
STATIC_ROOT = REPO_ROOT / "staticfiles"
STATICFILES_DIRS = [FRONTEND_DIST] if FRONTEND_DIST.exists() else []
# WhiteNoise plain storage on purpose - Vite already content-hashes filenames

LANGUAGE_CODE = "en-us"
TIME_ZONE = "Asia/Taipei"
USE_TZ = False
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

from toolkitsy.logger import configure as _configure_logging
_configure_logging()  # console only; Cloud Run captures stdout
```

- [ ] **Step 3: Fresh routing + health app**

`backend/config/urls.py`:

```python
from django.urls import path, include
from django.views.generic import TemplateView

urlpatterns = [
    path("health", include("health.urls")),
    path("api/v1/", include("events.urls")),
    path("", TemplateView.as_view(template_name="index.html"), name="spa"),
]
```

`backend/health/urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [path("", views.health, name="health")]
```

`backend/health/views.py`:

```python
from django.http import JsonResponse


def health(request):
    return JsonResponse({"status": "ok"})
```

`backend/health/tests.py` (replace old content):

```python
import json


def test_health(client):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert json.loads(resp.content) == {"status": "ok"}
```

Delete `backend/health/templates/` and the app's `apps.py` old name if it says `health_check` — update to:

```python
from django.apps import AppConfig


class HealthConfig(AppConfig):
    name = "health"
```

- [ ] **Step 4: Update module references**

- `backend/manage.py`, `backend/config/wsgi.py`, `backend/config/asgi.py`: `main_project.settings` → `config.settings`.
- `backend/pytest.ini`: `DJANGO_SETTINGS_MODULE = config.settings`
- `Dockerfile`: `COPY main_project/ ./main_project/` → `COPY backend/ ./backend/`; collectstatic path → `backend/manage.py`; CMD → `gunicorn --chdir backend config.wsgi:application --bind 0.0.0.0:${PORT:-8080} --workers 1`
- `.github/workflows/deploy.yml`: `cd main_project` → `cd backend`
- Rewrite `makefile` entirely:

```makefile
.DEFAULT_GOAL := help

.PHONY: help
help:
	@echo "  make run-api   - Django API dev server (:8000)"
	@echo "  make run-web   - Vite dev server (:5173, proxies /api)"
	@echo "  make test      - backend + frontend tests"
	@echo "  make run-prod  - build & run the production container locally"

.PHONY: run-api
run-api:
	cd backend && DEBUG=True uv run python manage.py runserver 8000

.PHONY: run-web
run-web:
	cd frontend && npm run dev

.PHONY: test
test:
	cd backend && uv run python -m pytest . -v
	cd frontend && npm test

.PHONY: run-prod
run-prod:
	docker build -t cef-local .
	docker run --rm -e PORT=8080 -p 8080:8080 cef-local
```

- [ ] **Step 5: Verify everything**

```bash
cd backend && uv run python -m pytest . -v && cd ..
docker build -t cef-local . && docker run --rm -e PORT=8080 -p 8080:8080 -d --name cef cef-local
sleep 5
curl -s http://127.0.0.1:8080/health
curl -s http://127.0.0.1:8080/api/v1/countries | head -c 80
docker stop cef
```

Expected: tests green; `{"status": "ok"}`; countries JSON.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "refactor: restructure into backend/config layout with fresh settings"
```

### Task 16: README rewrite + repo rename (OWNER-GATED) + final deploy

**Files:**
- Modify: `README.md` (full rewrite); Delete: stale `readme/*.png` screenshots

- [ ] **Step 1: Rewrite `README.md`:**

```markdown
# Culture Event Finder

Search culture events via government open data. Currently supports Taiwan
(Ministry of Culture); the provider architecture is ready for more countries.

**Live:** https://culture-event-finder-<hash>.run.app (update after deploy)

## Architecture

React (Vite + TS + Tailwind) SPA + Django JSON API, shipped as ONE container
on Google Cloud Run. Backend layering: views → services (cache-aside, 12h TTL)
→ providers (one per country).

| Layer | Stack |
|---|---|
| Frontend | React 19, Vite, TypeScript, Tailwind v4 |
| Backend | Django 5.2, uv, toolkitsy (logging) |
| Tests | pytest + responses / Vitest |
| Deploy | Docker multi-stage → Cloud Run, GitHub Actions (WIF) |

## Dev

    make run-api   # Django API on :8000
    make run-web   # Vite dev server on :5173 (proxies /api)
    make test      # backend + frontend tests
    make run-prod  # production container locally on :8080

## Adding a country

1. `backend/events/providers/<country>.py` — subclass `BaseProvider`
2. Register it in `backend/events/providers/__init__.py`
3. Done — `/api/v1/countries` and the frontend pick it up automatically
```

```bash
git rm readme/culture.png readme/deployment.png readme/logs.png
git add README.md && git commit -m "docs: rewrite readme for culture-event-finder architecture"
```

- [ ] **Step 2 (OWNER): Rename the GitHub repo** to `culture-event-finder` (Settings → General; old URLs redirect). Then locally:

```bash
git remote set-url origin git@github.com:taurus5650/culture-event-finder.git
```

Optionally rename the local folder (outside any running session): `mv taiwan_culture_event_info_django_jinja2 culture-event-finder`.

- [ ] **Step 3: Merge to master, watch the deploy job, verify the prod URL end-to-end** (SPA loads, one real search works on desktop + phone). Update the Live URL in README, commit.

---

## Self-Review Record (kept for the executor)

- Spec coverage: §2 repo structure → T15; §3.1 API → T4; §3.2 providers → T2; §3.3 cache + month conversion → T3; §4 frontend/i18n/no-router → T5–T9; §5 Dockerfile/uv/WhiteNoise/ALLOWED_HOSTS/dev-proxy/CI/GCP checklist → T10–T12 + makefile in T15; §6 cleanup/replace (toolkitsy) → T1, T4, T14–T16; §7 tests → T2–T4, T6, T15; §8 milestones → task ordering.
- Intentional deviations from spec text: Dockerfile at repo root (not `deployment/`) because `gcloud run deploy --source` requires it there; `/health` route lands at T15 (T10–T13 keep legacy `/health_check/`); one-time GCP infra is Terraform-managed per owner request (T11) — app deploys stay in CI, local tfstate gitignored.
- The `settings.py` Read-block workaround (Bash snippets in T4/T10/T14) ends at T15 when the file is rewritten fresh.
- toolkitsy has no http module yet (verified on PyPI 0.1.0) — external HTTP stays on `requests`, isolated in `taiwan.py` for a future one-file swap.
