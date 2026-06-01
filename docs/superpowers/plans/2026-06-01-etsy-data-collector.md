# Etsy Data Collector Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a compliant Etsy data collector that can gather public listing/shop research data without relying on login-wall bypasses or anti-bot circumvention.

**Architecture:** Use Etsy Open API v3 as the primary data source and store normalized results locally. Keep any browser/page inspection as a diagnostics-only module for checking what fields appear publicly, not for bypassing Etsy defenses.

**Tech Stack:** Python 3.11+, `httpx`, `pydantic`, `pytest`, SQLite, optional Playwright only for manual diagnostics.

---

## Current Login/Access Finding

Manual web inspection on 2026-06-01 found:

- Etsy public homepage and market/search pages show content while signed out, including item titles, prices, images, shop links, and a visible `Sign in` entry.
- A public market page such as `https://www.etsy.com/market/handmade_necklace` is visible through a normal browser/search rendering path without logging in.
- Local command-line requests from this machine did not reliably fetch Etsy HTML: `curl` hit `Recv failure: Connection reset by peer`, and Python `urllib` received HTTP 403. Treat this as automated-request blocking rather than a login requirement.
- Etsy provides Open API v3 and documents API requests using an `x-api-key` header, with OAuth tokens required for endpoints/scopes that need user authorization.
- Etsy API Terms prohibit automated systems to access, analyze, or scrape Etsy site/API/data unless Etsy expressly authorizes it. Therefore the implementation should not include CAPTCHA bypass, proxy rotation to evade blocks, login-cookie harvesting, or hidden browser automation intended to defeat Etsy protections.

## File Structure

- Create: `README.md`
  - Explains setup, credentials, allowed use, and command examples.
- Create: `.env.example`
  - Documents `ETSY_API_KEY`, `ETSY_API_SECRET`, optional OAuth token fields, rate settings, and SQLite path.
- Create: `pyproject.toml`
  - Defines Python package metadata, dependencies, scripts, and pytest config.
- Create: `src/etsy_collector/config.py`
  - Loads environment configuration with validation.
- Create: `src/etsy_collector/api_client.py`
  - Wraps Etsy Open API requests, retries, rate limiting, and errors.
- Create: `src/etsy_collector/models.py`
  - Defines normalized listing, shop, image, and run metadata models.
- Create: `src/etsy_collector/storage.py`
  - Creates SQLite schema and upserts normalized records.
- Create: `src/etsy_collector/collector.py`
  - Orchestrates keyword/shop collection through API client and storage.
- Create: `src/etsy_collector/cli.py`
  - Provides command-line entrypoints for smoke checks and collection runs.
- Create: `src/etsy_collector/browser_diagnostics.py`
  - Optional diagnostics that opens public pages and records visible field availability only.
- Create: `tests/test_config.py`
  - Verifies env parsing and missing credential behavior.
- Create: `tests/test_models.py`
  - Verifies API payload normalization.
- Create: `tests/test_storage.py`
  - Verifies schema creation and upsert behavior.
- Create: `tests/test_api_client.py`
  - Verifies request headers, pagination, retry classification, and 403 handling.
- Create: `tests/test_collector.py`
  - Verifies collection orchestration using mocked API responses.

## Tasks

### Task 1: Project Scaffold

**Files:**
- Create: `pyproject.toml`
- Create: `README.md`
- Create: `.env.example`
- Create: `src/etsy_collector/__init__.py`

- [ ] **Step 1: Add package metadata and dependencies**

Create `pyproject.toml`:

```toml
[project]
name = "etsy-collector"
version = "0.1.0"
description = "Compliant Etsy Open API data collector"
requires-python = ">=3.11"
dependencies = [
  "httpx>=0.27",
  "pydantic>=2.7",
  "pydantic-settings>=2.3",
  "python-dotenv>=1.0",
  "tenacity>=8.3",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.2",
  "pytest-httpx>=0.30",
]
diagnostics = [
  "playwright>=1.44",
]

[project.scripts]
etsy-collector = "etsy_collector.cli:main"

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
```

- [ ] **Step 2: Add credential template**

Create `.env.example`:

```bash
ETSY_API_KEY=
ETSY_API_SECRET=
ETSY_OAUTH_ACCESS_TOKEN=
ETSY_SQLITE_PATH=data/etsy.sqlite3
ETSY_REQUEST_TIMEOUT_SECONDS=30
ETSY_MAX_RETRIES=3
ETSY_MIN_REQUEST_INTERVAL_SECONDS=1.0
```

- [ ] **Step 3: Add README**

Create `README.md`:

```markdown
# Etsy Collector

This project collects Etsy research data through Etsy Open API v3.

It does not bypass login walls, CAPTCHA, anti-bot protections, or Etsy access controls. Browser diagnostics are only for checking which fields are publicly visible in a normal signed-out page.

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env
```

Fill `.env` with Etsy developer credentials.

## Smoke Check

```bash
etsy-collector smoke
```

## Collect

```bash
etsy-collector collect-keyword "handmade necklace" --limit 100
```
```

- [ ] **Step 4: Add package init**

Create `src/etsy_collector/__init__.py`:

```python
__all__ = ["__version__"]

__version__ = "0.1.0"
```

- [ ] **Step 5: Run scaffold verification**

Run:

```bash
python3 -m pip install -e ".[dev]"
pytest -q
```

Expected: install succeeds; pytest reports no tests collected or all tests pass.

### Task 2: Configuration

**Files:**
- Create: `src/etsy_collector/config.py`
- Create: `tests/test_config.py`

- [ ] **Step 1: Write failing config tests**

Create `tests/test_config.py`:

```python
import pytest

from etsy_collector.config import Settings


def test_settings_load_required_api_credentials():
    settings = Settings(
        etsy_api_key="key123",
        etsy_api_secret="secret456",
        etsy_sqlite_path="data/test.sqlite3",
    )

    assert settings.api_key_header == "key123:secret456"
    assert str(settings.sqlite_path).endswith("data/test.sqlite3")


def test_settings_rejects_blank_api_key():
    with pytest.raises(ValueError):
        Settings(etsy_api_key="", etsy_api_secret="secret456")
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
pytest tests/test_config.py -q
```

Expected: FAIL because `etsy_collector.config` does not exist.

- [ ] **Step 3: Implement config**

Create `src/etsy_collector/config.py`:

```python
from pathlib import Path

from pydantic import Field, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="ETSY_")

    etsy_api_key: str = Field(alias="ETSY_API_KEY")
    etsy_api_secret: str = Field(alias="ETSY_API_SECRET")
    etsy_oauth_access_token: str | None = Field(default=None, alias="ETSY_OAUTH_ACCESS_TOKEN")
    etsy_sqlite_path: Path = Field(default=Path("data/etsy.sqlite3"), alias="ETSY_SQLITE_PATH")
    etsy_request_timeout_seconds: float = Field(default=30, alias="ETSY_REQUEST_TIMEOUT_SECONDS")
    etsy_max_retries: int = Field(default=3, alias="ETSY_MAX_RETRIES")
    etsy_min_request_interval_seconds: float = Field(default=1.0, alias="ETSY_MIN_REQUEST_INTERVAL_SECONDS")

    @field_validator("etsy_api_key", "etsy_api_secret")
    @classmethod
    def require_non_blank(cls, value: str) -> str:
        if not value.strip():
            raise ValueError("Etsy API credentials must not be blank")
        return value.strip()

    @property
    def api_key_header(self) -> str:
        return f"{self.etsy_api_key}:{self.etsy_api_secret}"

    @property
    def sqlite_path(self) -> Path:
        return self.etsy_sqlite_path
```

- [ ] **Step 4: Run tests**

Run:

```bash
pytest tests/test_config.py -q
```

Expected: PASS.

### Task 3: Normalized Models

**Files:**
- Create: `src/etsy_collector/models.py`
- Create: `tests/test_models.py`

- [ ] **Step 1: Write failing model tests**

Create `tests/test_models.py`:

```python
from etsy_collector.models import ListingRecord


def test_listing_record_from_api_payload():
    payload = {
        "listing_id": 123,
        "title": "Handmade Necklace",
        "url": "https://www.etsy.com/listing/123/handmade-necklace",
        "shop_id": 456,
        "price": {"amount": 2599, "divisor": 100, "currency_code": "USD"},
        "state": "active",
        "quantity": 7,
        "tags": ["necklace", "gift"],
    }

    record = ListingRecord.from_api(payload)

    assert record.listing_id == 123
    assert record.title == "Handmade Necklace"
    assert record.price_amount == 25.99
    assert record.currency == "USD"
    assert record.tags == ["necklace", "gift"]
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
pytest tests/test_models.py -q
```

Expected: FAIL because `etsy_collector.models` does not exist.

- [ ] **Step 3: Implement models**

Create `src/etsy_collector/models.py`:

```python
from datetime import datetime, timezone
from typing import Any

from pydantic import BaseModel, Field


class ListingRecord(BaseModel):
    listing_id: int
    title: str
    url: str | None = None
    shop_id: int | None = None
    price_amount: float | None = None
    currency: str | None = None
    state: str | None = None
    quantity: int | None = None
    tags: list[str] = Field(default_factory=list)
    collected_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

    @classmethod
    def from_api(cls, payload: dict[str, Any]) -> "ListingRecord":
        price = payload.get("price") or {}
        amount = price.get("amount")
        divisor = price.get("divisor") or 100
        price_amount = None
        if amount is not None:
            price_amount = amount / divisor

        return cls(
            listing_id=int(payload["listing_id"]),
            title=str(payload.get("title") or ""),
            url=payload.get("url"),
            shop_id=payload.get("shop_id"),
            price_amount=price_amount,
            currency=price.get("currency_code"),
            state=payload.get("state"),
            quantity=payload.get("quantity"),
            tags=list(payload.get("tags") or []),
        )
```

- [ ] **Step 4: Run tests**

Run:

```bash
pytest tests/test_models.py -q
```

Expected: PASS.

### Task 4: Etsy API Client

**Files:**
- Create: `src/etsy_collector/api_client.py`
- Create: `tests/test_api_client.py`

- [ ] **Step 1: Write failing API client tests**

Create `tests/test_api_client.py`:

```python
import httpx
import pytest

from etsy_collector.api_client import EtsyApiClient, EtsyAccessBlocked
from etsy_collector.config import Settings


def make_settings() -> Settings:
    return Settings(etsy_api_key="key123", etsy_api_secret="secret456")


def test_client_sends_api_key_header(httpx_mock):
    httpx_mock.add_response(
        method="GET",
        url="https://api.etsy.com/v3/application/listings?state=active&limit=1",
        json={"results": []},
    )
    client = EtsyApiClient(make_settings())

    client.get_active_listings(limit=1)

    request = httpx_mock.get_request()
    assert request.headers["x-api-key"] == "key123:secret456"


def test_client_classifies_forbidden_as_blocked(httpx_mock):
    httpx_mock.add_response(status_code=403, json={"error": "forbidden"})
    client = EtsyApiClient(make_settings())

    with pytest.raises(EtsyAccessBlocked):
        client.get_active_listings(limit=1)
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
pytest tests/test_api_client.py -q
```

Expected: FAIL because `etsy_collector.api_client` does not exist.

- [ ] **Step 3: Implement API client**

Create `src/etsy_collector/api_client.py`:

```python
from typing import Any

import httpx

from etsy_collector.config import Settings


class EtsyApiError(RuntimeError):
    pass


class EtsyAccessBlocked(EtsyApiError):
    pass


class EtsyApiClient:
    def __init__(self, settings: Settings):
        self.settings = settings
        self.client = httpx.Client(
            base_url="https://api.etsy.com/v3/application",
            timeout=settings.etsy_request_timeout_seconds,
            headers={"x-api-key": settings.api_key_header},
        )
        if settings.etsy_oauth_access_token:
            self.client.headers["authorization"] = f"Bearer {settings.etsy_oauth_access_token}"

    def get_active_listings(self, *, limit: int = 25, offset: int = 0) -> dict[str, Any]:
        return self._get("/listings", params={"state": "active", "limit": limit, "offset": offset})

    def _get(self, path: str, params: dict[str, Any]) -> dict[str, Any]:
        response = self.client.get(path, params=params)
        if response.status_code in {401, 403}:
            raise EtsyAccessBlocked(f"Etsy API access denied: HTTP {response.status_code}")
        try:
            response.raise_for_status()
        except httpx.HTTPStatusError as exc:
            raise EtsyApiError(str(exc)) from exc
        return response.json()

    def close(self) -> None:
        self.client.close()
```

- [ ] **Step 4: Run tests**

Run:

```bash
pytest tests/test_api_client.py -q
```

Expected: PASS.

### Task 5: SQLite Storage

**Files:**
- Create: `src/etsy_collector/storage.py`
- Create: `tests/test_storage.py`

- [ ] **Step 1: Write failing storage tests**

Create `tests/test_storage.py`:

```python
import sqlite3

from etsy_collector.models import ListingRecord
from etsy_collector.storage import SqliteStore


def test_upsert_listing(tmp_path):
    db_path = tmp_path / "etsy.sqlite3"
    store = SqliteStore(db_path)
    store.initialize()

    store.upsert_listing(
        ListingRecord(
            listing_id=123,
            title="Handmade Necklace",
            url="https://www.etsy.com/listing/123/handmade-necklace",
            price_amount=25.99,
            currency="USD",
            tags=["necklace"],
        )
    )

    with sqlite3.connect(db_path) as conn:
        row = conn.execute("select listing_id, title, price_amount from listings").fetchone()

    assert row == (123, "Handmade Necklace", 25.99)
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
pytest tests/test_storage.py -q
```

Expected: FAIL because `etsy_collector.storage` does not exist.

- [ ] **Step 3: Implement storage**

Create `src/etsy_collector/storage.py`:

```python
import json
import sqlite3
from pathlib import Path

from etsy_collector.models import ListingRecord


class SqliteStore:
    def __init__(self, db_path: Path):
        self.db_path = Path(db_path)

    def initialize(self) -> None:
        self.db_path.parent.mkdir(parents=True, exist_ok=True)
        with sqlite3.connect(self.db_path) as conn:
            conn.execute(
                """
                create table if not exists listings (
                    listing_id integer primary key,
                    title text not null,
                    url text,
                    shop_id integer,
                    price_amount real,
                    currency text,
                    state text,
                    quantity integer,
                    tags_json text not null,
                    collected_at text not null
                )
                """
            )

    def upsert_listing(self, record: ListingRecord) -> None:
        with sqlite3.connect(self.db_path) as conn:
            conn.execute(
                """
                insert into listings (
                    listing_id, title, url, shop_id, price_amount, currency,
                    state, quantity, tags_json, collected_at
                )
                values (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                on conflict(listing_id) do update set
                    title=excluded.title,
                    url=excluded.url,
                    shop_id=excluded.shop_id,
                    price_amount=excluded.price_amount,
                    currency=excluded.currency,
                    state=excluded.state,
                    quantity=excluded.quantity,
                    tags_json=excluded.tags_json,
                    collected_at=excluded.collected_at
                """,
                (
                    record.listing_id,
                    record.title,
                    record.url,
                    record.shop_id,
                    record.price_amount,
                    record.currency,
                    record.state,
                    record.quantity,
                    json.dumps(record.tags, ensure_ascii=False),
                    record.collected_at.isoformat(),
                ),
            )
```

- [ ] **Step 4: Run tests**

Run:

```bash
pytest tests/test_storage.py -q
```

Expected: PASS.

### Task 6: Collector Orchestration

**Files:**
- Create: `src/etsy_collector/collector.py`
- Create: `tests/test_collector.py`

- [ ] **Step 1: Write failing collector test**

Create `tests/test_collector.py`:

```python
from etsy_collector.collector import ListingCollector


class FakeClient:
    def get_active_listings(self, *, limit: int, offset: int):
        assert limit == 2
        assert offset == 0
        return {
            "results": [
                {"listing_id": 1, "title": "A", "price": {"amount": 100, "divisor": 100, "currency_code": "USD"}},
                {"listing_id": 2, "title": "B", "price": {"amount": 200, "divisor": 100, "currency_code": "USD"}},
            ]
        }


class FakeStore:
    def __init__(self):
        self.records = []

    def upsert_listing(self, record):
        self.records.append(record)


def test_collect_active_listings_stores_normalized_records():
    store = FakeStore()
    collector = ListingCollector(FakeClient(), store)

    count = collector.collect_active_listings(limit=2)

    assert count == 2
    assert [record.listing_id for record in store.records] == [1, 2]
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
pytest tests/test_collector.py -q
```

Expected: FAIL because `etsy_collector.collector` does not exist.

- [ ] **Step 3: Implement collector**

Create `src/etsy_collector/collector.py`:

```python
from typing import Protocol

from etsy_collector.models import ListingRecord


class ListingClient(Protocol):
    def get_active_listings(self, *, limit: int, offset: int) -> dict:
        ...


class ListingStore(Protocol):
    def upsert_listing(self, record: ListingRecord) -> None:
        ...


class ListingCollector:
    def __init__(self, client: ListingClient, store: ListingStore):
        self.client = client
        self.store = store

    def collect_active_listings(self, *, limit: int = 100) -> int:
        response = self.client.get_active_listings(limit=limit, offset=0)
        count = 0
        for payload in response.get("results", []):
            self.store.upsert_listing(ListingRecord.from_api(payload))
            count += 1
        return count
```

- [ ] **Step 4: Run tests**

Run:

```bash
pytest tests/test_collector.py -q
```

Expected: PASS.

### Task 7: CLI

**Files:**
- Create: `src/etsy_collector/cli.py`
- Create: `tests/test_cli.py`

- [ ] **Step 1: Write failing CLI tests**

Create `tests/test_cli.py`:

```python
from etsy_collector.cli import build_parser


def test_collect_keyword_parser():
    parser = build_parser()
    args = parser.parse_args(["collect-keyword", "handmade necklace", "--limit", "50"])

    assert args.command == "collect-keyword"
    assert args.keyword == "handmade necklace"
    assert args.limit == 50
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
pytest tests/test_cli.py -q
```

Expected: FAIL because `etsy_collector.cli` does not exist.

- [ ] **Step 3: Implement CLI**

Create `src/etsy_collector/cli.py`:

```python
import argparse

from etsy_collector.api_client import EtsyApiClient
from etsy_collector.collector import ListingCollector
from etsy_collector.config import Settings
from etsy_collector.storage import SqliteStore


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="etsy-collector")
    subparsers = parser.add_subparsers(dest="command", required=True)

    subparsers.add_parser("smoke")

    collect_keyword = subparsers.add_parser("collect-keyword")
    collect_keyword.add_argument("keyword")
    collect_keyword.add_argument("--limit", type=int, default=100)

    return parser


def main() -> None:
    parser = build_parser()
    args = parser.parse_args()
    settings = Settings()
    store = SqliteStore(settings.sqlite_path)
    store.initialize()
    client = EtsyApiClient(settings)

    try:
        if args.command == "smoke":
            response = client.get_active_listings(limit=1)
            print(f"ok results={len(response.get('results', []))}")
        elif args.command == "collect-keyword":
            collector = ListingCollector(client, store)
            count = collector.collect_active_listings(limit=args.limit)
            print(f"stored listings={count} keyword={args.keyword}")
    finally:
        client.close()
```

- [ ] **Step 4: Run tests**

Run:

```bash
pytest tests/test_cli.py -q
```

Expected: PASS.

### Task 8: Browser Diagnostics Guardrail

**Files:**
- Create: `src/etsy_collector/browser_diagnostics.py`
- Create: `tests/test_browser_diagnostics.py`

- [ ] **Step 1: Write guardrail test**

Create `tests/test_browser_diagnostics.py`:

```python
from etsy_collector.browser_diagnostics import is_allowed_diagnostic_url


def test_allows_public_etsy_pages_only():
    assert is_allowed_diagnostic_url("https://www.etsy.com/market/handmade_necklace")
    assert is_allowed_diagnostic_url("https://www.etsy.com/listing/123/example")
    assert not is_allowed_diagnostic_url("https://www.etsy.com/signin")
    assert not is_allowed_diagnostic_url("https://example.com/listing/123")
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
pytest tests/test_browser_diagnostics.py -q
```

Expected: FAIL because diagnostics module does not exist.

- [ ] **Step 3: Implement URL guardrail**

Create `src/etsy_collector/browser_diagnostics.py`:

```python
from urllib.parse import urlparse


ALLOWED_PREFIXES = ("/market/", "/listing/", "/shop/")
DISALLOWED_PREFIXES = ("/signin", "/your/", "/cart", "/messages")


def is_allowed_diagnostic_url(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme != "https" or parsed.netloc != "www.etsy.com":
        return False
    if parsed.path.startswith(DISALLOWED_PREFIXES):
        return False
    return parsed.path.startswith(ALLOWED_PREFIXES)
```

- [ ] **Step 4: Run tests**

Run:

```bash
pytest tests/test_browser_diagnostics.py -q
```

Expected: PASS.

### Task 9: Full Verification

**Files:**
- Modify: none unless tests expose issues.

- [ ] **Step 1: Run all tests**

Run:

```bash
pytest -q
```

Expected: PASS.

- [ ] **Step 2: Run smoke without credentials**

Run:

```bash
etsy-collector smoke
```

Expected: exits with a clear validation error that Etsy API credentials are missing.

- [ ] **Step 3: Run smoke with credentials**

Run after `.env` is filled:

```bash
etsy-collector smoke
```

Expected: prints `ok results=1` or an Etsy API authorization error that names the missing approval/scope.

- [ ] **Step 4: Commit**

Run:

```bash
git add README.md .env.example pyproject.toml src tests docs/superpowers/plans/2026-06-01-etsy-data-collector.md
git commit -m "plan: add Etsy data collector implementation plan"
```

Expected: commit succeeds.

## Self-Review

- Spec coverage: covers login/access check, compliant primary data source, local persistence, tests, CLI, and diagnostics guardrails.
- Placeholder scan: no TBD/TODO/implement-later placeholders.
- Type consistency: `Settings`, `ListingRecord`, `EtsyApiClient`, `SqliteStore`, and `ListingCollector` names are consistent across tasks.
