# DoD Contract Announcement Parser

![Python](https://img.shields.io/badge/python-3.12%2B-3776AB?logo=python&logoColor=white)
![Scrapy](https://img.shields.io/badge/scrapy-2.x-60A839)
![PostgreSQL](https://img.shields.io/badge/postgresql-16-4169E1?logo=postgresql&logoColor=white)
![uv](https://img.shields.io/badge/managed%20with-uv-DE5FE9)

A Scrapy pipeline that collects the Department of War's (formerly the Department of Defense) daily contract announcements, extracts structured fields from their free-text paragraphs, and measures its own accuracy against federal spending records from USAspending.gov.

Every business day at 5 p.m. ET, the Department publishes every contract action worth $7.5 million or more. Those announcements are written for journalists: one paragraph per award, with the company, dollar figure, contract type, work locations, completion date, and contract number embedded in prose that varies by branch, by author, and by decade. This project turns that prose into a clean, queryable dataset and backs every field with an accuracy measurement.

---

## Motivation

From 2015 to 2019 I served in the U.S. Army as a Joint Fire Support Specialist (13F). Part of that service meant supporting civilian contractors working alongside the military in Jordan, Kuwait, Germany, and Italy. Working next to them made me want to understand the other half of their story: which companies win defense work, what it costs, where it gets done, and which offices award it.

That record is public, but it arrives as daily prose that is hard to search or total up. I built this project to make it usable, and to make it trustworthy: every extracted field is checked against a hand-labeled test set and reconciled with a second government source.

---

## Features

- **Archive backfill and daily collection.** Crawls historical announcement pages, then picks up each new day from the contracts RSS feed.
- **Structured extraction.** Turns each award paragraph into a typed record: company, amount and what kind of amount it is, branch, contract type, work locations with percentage splits, completion date, contract number (PIID), contracting activity, award date, small-business flag, and Foreign Military Sales countries.
- **Handles real-world complexity.** Multiple-award contracts with dozens of awardees, modifications to existing contracts, undefinitized actions with not-to-exceed values, late announcements, and two decades of format drift.
- **Measured accuracy.** Per-field precision and recall against a hand-labeled gold set.
- **Cross-source validation.** Matches each award to its USAspending.gov record and reports where the two sources agree and why they differ.
- **Reproducible and idempotent.** Raw HTML is kept, so any day can be re-parsed with improved rules without re-crawling. Re-running a day never creates duplicate rows.
- **Polite by design.** Obeys `robots.txt`, throttles automatically, identifies itself with a contact address, and never routes around bot protection.

---

## How it works

```mermaid
flowchart LR
    A[war.gov RSS / archive pages] -->|Scrapy spider| B[(Raw HTML store)]
    B --> C[Page splitter<br/>branch sections → paragraphs]
    C --> D[Field extractors<br/>rules + Pydantic validation]
    D --> E[(PostgreSQL<br/>pages, paragraphs, awards)]
    F[USAspending API] -->|httpx + tenacity| G[(PostgreSQL<br/>usaspending_awards)]
    E --> H[Matcher<br/>PIID + fuzzy name + date window]
    G --> H
    H --> I[(validation_results)]
    I --> J[Quality report]
    K[Daily scheduler] -.-> A
    K -.90-day lag.-> F
```

1. **Crawl.** A Scrapy spider fetches announcement pages and stores the raw HTML on disk, one file per page.
2. **Split.** Each page is divided into branch sections (Army, Navy, Air Force, Space Force, Defense Logistics Agency, and other agencies) and then into award paragraphs.
3. **Extract.** One small, independently tested rule module per field pulls values from each paragraph. Every value records which rule produced it, and every record passes through a Pydantic model that rejects impossible values.
4. **Load.** Records are upserted into PostgreSQL, keyed on article ID and paragraph position.
5. **Validate.** Awards older than 90 days are matched to USAspending records and compared field by field.
6. **Report.** Accuracy and agreement metrics are written to a Markdown report.

### Why validation runs 90 days behind

DoD contract actions are not published in FPDS, and therefore not in USAspending, until 90 days after signing. The pipeline marks newer awards `not_yet_available` and re-checks each one automatically once it crosses that mark.

### Why matching on the contract number alone isn't enough

A PIID is not unique by itself. Delivery order `0001`, for example, exists under many different parent contracts. The matcher combines the normalized PIID with company-name similarity and a date window, then stores USAspending's `generated_internal_id` as the durable link.

---

## Extracted fields

| Field | Example | Challenge |
|-------|---------|-----------|
| `company` | "L3Harris Technologies Integrated Systems L.P., doing business as Aeromet" | Legal suffixes, "doing business as", joint ventures, lists of 20+ awardees |
| `company_location` | "Tulsa, Oklahoma" / "Kongsberg, Norway" | AP state abbreviations on older pages; foreign awardees |
| `small_business` | Trailing `*` after the company name | Easy to misattribute in multi-award lists |
| `amount`, `amount_kind` | "not-to-exceed $708,939,863" | Several dollar figures per paragraph: face value, obligations, options, FMS funds, cumulative value |
| `branch` | ARMY, NAVY, AIR FORCE | Comes from page structure, not paragraph text |
| `contract_type` | "firm-fixed-price, indefinite-delivery/indefinite-quantity" | Compound types; one contract can be several at once |
| `work_locations` | "Andover, Massachusetts (30%); San Diego, California (10%)…" | Percentage splits, "various locations", "determined with each order" |
| `completion_date` | "Aug. 4, 2032" / "March 2030" | Day- vs. month-level precision; option-extended dates |
| `piid` | `W911S0-26-D-A008` / `FA853226DB028` | Inconsistent dashes; parent PIID on modifications |
| `contracting_activity` | "Army Contracting Command, Newark, New Jersey" | Position in the paragraph varies |
| `award_date` | "(Awarded July 30, 2026)" | Present only when it differs from the announcement date |
| `fms_countries` | "Foreign Military Sales to Belgium, Canada, Finland…" | Optional, variable-length lists |

---

## Data model

| Table | One row per | Key columns |
|-------|-------------|-------------|
| `announcement_pages` | Daily announcement page | `article_id` (PK), `announcement_date`, `url`, `stated_threshold`, `fetched_at`, `raw_html_path`, `parse_version` |
| `award_paragraphs` | Award paragraph | `page_id`, `branch`, `paragraph_index`, `text` |
| `awards` | Awardee on a contract | `paragraph_id`, `company`, `piid`, `amount`, `amount_kind`, `contract_types`, `completion_date`, `completion_precision`, `small_business`, `extraction_flags` |
| `work_locations` | Location on an award | `award_id`, `city`, `state_or_country`, `percent` |
| `usaspending_awards` | USAspending record | `generated_internal_id` (PK), `piid`, `recipient_name`, `recipient_uei`, `base_and_all_options`, `total_obligation`, `potential_end_date`, `fetched_at` |
| `validation_results` | Award × field check | `award_id`, `usaspending_id`, `field`, `outcome`, `detail` |
| `pipeline_runs` | Pipeline run | `started_at`, `finished_at`, `pages_fetched`, `awards_parsed`, `errors` |

Schema changes are managed with Alembic migrations.

---

## Tech stack

| Tool | Role |
|------|------|
| Python 3.12+, [uv](https://docs.astral.sh/uv/) | Runtime and dependency management |
| [Scrapy](https://scrapy.org/) | Crawling, throttling, `robots.txt` compliance, HTTP caching |
| [parsel](https://parsel.readthedocs.io/) | CSS/XPath selectors for page splitting |
| [Pydantic](https://docs.pydantic.dev/) | Typed, validated award records |
| [RapidFuzz](https://rapidfuzz.github.io/RapidFuzz/) | Company-name similarity for record matching |
| [httpx](https://www.python-httpx.org/) + [tenacity](https://tenacity.readthedocs.io/) | USAspending API client with retry and backoff |
| PostgreSQL 16, [SQLAlchemy](https://www.sqlalchemy.org/), [Alembic](https://alembic.sqlalchemy.org/) | Storage, ORM, migrations |
| [pytest](https://docs.pytest.org/) | Extractor unit tests and gold-set evaluation |
| Docker Compose | Local database and reproducible runs |

---

## Getting started

### Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/)
- Docker and Docker Compose

### Install

```bash
git clone https://github.com/<your-username>/dod-contracts.git
cd dod-contracts
uv sync
cp .env.example .env          # then set CONTACT_EMAIL
docker compose up -d db
uv run alembic upgrade head
```

### Run

```bash
# Backfill a date range
uv run dod-contracts crawl --since 2025-10-01 --until 2026-09-30
uv run dod-contracts parse
uv run dod-contracts load

# Validate against USAspending and build the report
uv run dod-contracts validate
uv run dod-contracts report --out reports/quality.md

# Daily run: RSS → crawl → parse → load → lagged validation
uv run dod-contracts run
```

### Command reference

| Command | Description |
|---------|-------------|
| `crawl` | Fetch announcement pages (RSS by default, or `--since`/`--until` for a backfill) and store raw HTML |
| `parse` | Split and extract stored pages; `--reparse` re-runs every page with the current rules |
| `load` | Upsert parsed records into PostgreSQL |
| `validate` | Match eligible awards to USAspending and record per-field outcomes |
| `report` | Write accuracy and agreement metrics to Markdown |
| `evaluate` | Score the extractors against the gold set |
| `run` | The full daily pipeline, intended for cron or a systemd timer |

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `postgresql+psycopg://dod:dod@localhost:5432/dod_contracts` | Database connection |
| `CONTACT_EMAIL` | *(required)* | Included in the crawler's User-Agent so site operators can reach you |
| `DATA_DIR` | `./data` | Location of raw HTML and cached API responses |
| `VALIDATION_LAG_DAYS` | `90` | Minimum award age before USAspending validation |
| `HTTPCACHE_ENABLED` | `false` | Enable Scrapy's HTTP cache during development |

---

## Testing

```bash
uv run pytest                     # unit tests for every extractor
uv run dod-contracts evaluate     # precision and recall on the gold set
```

The gold set lives in `tests/gold/` alongside `GUIDELINES.md`, which defines how each field was labeled (for example, which dollar figure counts as the announced amount, and how month-only dates are recorded).

---

## Project layout

```
dod-contracts/
├── pyproject.toml
├── scrapy.cfg
├── alembic/                    # migrations
├── docker-compose.yml
├── src/dod_contracts/
│   ├── cli.py                  # command entry points
│   ├── settings.py             # Scrapy settings
│   ├── spiders/                # announcement and RSS spiders
│   ├── splitting.py            # page → branch sections → paragraphs
│   ├── extractors/             # one module per field
│   ├── models.py               # Pydantic records
│   ├── db/                     # SQLAlchemy models and loaders
│   └── validation/             # USAspending client, matcher, report
└── tests/
    ├── fixtures/               # saved announcement pages
    └── gold/                   # hand-labeled paragraphs
```

---

## Responsible crawling

The announcement site sits behind bot detection. This project respects that boundary:

- `ROBOTSTXT_OBEY` is always on, and the User-Agent names the project and a contact address.
- One request at a time per domain, with Scrapy's AutoThrottle enabled.
- Daily collection reads the RSS feed and fetches only pages not already stored.
- Raw HTML is kept, so improving the parser never requires re-crawling.
- No proxy rotation, header spoofing, or headless-browser evasion. If a source blocks automated access, the pipeline stops and logs it.

USAspending requests are paginated until `hasNext` is false, retried with backoff on 429 and 5xx responses, and cached locally.

---

## Limitations

- Validation can't run on awards younger than 90 days, because of the DoD publication delay.
- USAspending's search "Award Amount" reflects obligated dollars, which rarely equals an announced IDIQ ceiling. Amount comparisons depend on `amount_kind` and use a tolerance.
- Older announcements use a $5 million threshold, AP-style abbreviations, and different HTML. Extraction accuracy is reported per era rather than as one number.
- Work locations are compared on the primary place of performance only.
- Only prime awards are covered; subcontracts are out of scope.

---

## Data sources and attribution

- **Contract announcements:** U.S. Department of War (formerly Department of Defense), [war.gov/News/Contracts](https://www.war.gov/News/Contracts/)
- **Federal award data:** [USAspending.gov](https://www.usaspending.gov/), U.S. Department of the Treasury

Both are U.S. Government works. This is an independent project, not affiliated with or endorsed by the Department of War or the Department of the Treasury.