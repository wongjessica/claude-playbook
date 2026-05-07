# CLAUDE.md for a Python Data Pipeline

## Project shape

- Backend data pipeline: Python 3.12, Snowflake as the warehouse, Airflow for orchestration, S3 for object storage
- Multiple integrations with partner APIs (data vendors, ad platforms)
- Mostly batch (hourly/daily DAGs), some near-real-time
- Heavy SQL, both as `.sql` files and embedded in Python
- Tests use pytest with fixtures for mocked Snowflake connections

This shape is common in ad-tech, fintech, analytics platforms, and anywhere "the data team" owns ETL.

## The CLAUDE.md

```markdown
# Project Context

This is the data engineering pipeline for [Company]. We move data from partner APIs and our application database into Snowflake, transform it for downstream products (dashboards, ML, ad platform exports), and push results back out to partners.

## Stack

- Python 3.12 (do not use 3.13, we hit a `distutils` removal incompat with one of our libraries)
- Snowflake (primary warehouse)
- Airflow on Astronomer (Astro CLI for local dev)
- AWS: S3, Secrets Manager, occasionally Lambda
- Databricks for some heavy transforms (we are migrating off it)
- Partner APIs: LiveRamp, The Trade Desk, Index Exchange, OpenX, DV360, Meta

## Code conventions

- Black formatter, line length 100
- Type hints on every function signature, no exceptions
- Docstrings only when the function is non-obvious. Avoid noise like "Returns the result."
- Use `pathlib.Path`, never raw string paths
- Use `httpx` for HTTP, never `requests`
- Use `pydantic` v2 for any data shape that crosses a boundary (API response, file format, queue message)
- Use `structlog` for logging, with key/value pairs, never f-strings inside log messages

## SQL conventions

- Snowflake SQL files live in `sql/`, organized by domain
- Use uppercase keywords (`SELECT`, `FROM`, `WHERE`)
- Use lowercase for identifiers (column names, table names)
- Always qualify tables with database and schema: `INTERNAL_DB.MAPPINGS.METRO_CROSSWALK`
- Prefer `QUALIFY ROW_NUMBER() OVER (...)` over correlated subqueries for "latest record per group"
- Never use positional `GROUP BY` (e.g. `GROUP BY 1, 2, 3`). Always name the columns.
- Watch for `LUID_TO_RAMP_MAPPING_VIA_HEMS` style tables that have ~62x duplication; filter to latest data load via the staging view before joining

## Airflow conventions

- DAG files in `dags/`, one file per DAG, filename matches `dag_id`
- Use `@dag` and `@task` decorators (TaskFlow), not the old operator API, for new DAGs
- Default args go in a shared module, not duplicated per DAG
- All connections and variables resolved via Airflow connections / env, never hardcoded
- Use `params` for backfillable arguments (date ranges, client IDs)
- For DAG Python errors locally: import `timedelta` directly, use `datetime.strptime` (not `pd.to_datetime`) for parsing strings

## Snowflake gotchas (real ones we have hit)

- Permissions differ between dev and prod. Before merging to main, run any new query as the prod role to confirm access. We had a production break from `INTERNAL_DB.CHALICE_APPS.CLIENT_CONFIG` being readable in dev but not prod.
- Dynamic tables with incremental refresh can miss new partitions for some flight/run keys. If a dashboard shows missing data and the query looks right, suspect the dynamic table and try converting to a regular view.
- Year fields sometimes arrive malformed (e.g. `0025` instead of `2025`). Use `DATEADD(year, 2000, ...)` with a `YEAR() < 100` condition rather than string-replacing.

## Testing

- pytest, with fixtures for Snowflake mocks
- Run tests: `make test` (which is `pytest -xvs tests/`)
- We don't unit test SQL itself, but we do test the Python that builds queries
- Integration tests live in `tests/integration/` and require a Snowflake connection; they're skipped in CI

## How to run things

- Local Airflow: `astro dev start`
- Single DAG run: `astro dev run dags test <dag_id> <execution_date>`
- Snowflake CLI: we use `snowsql` configured via `~/.snowsql/config`, role `DATA_ENGINEER` for dev work
- Container model deployments: see `docs/container-deploys.md` (OpenX always uses `'US'` region; Index uses `'AMS'` for US deployments and `'EMEA'` for European)

## What not to do

- Do not write new code that imports `pandas` for ETL. We are moving toward `polars` for transforms and pure SQL for everything that fits in the warehouse.
- Do not add new Databricks notebooks. We are migrating off Databricks.
- Do not commit credentials, even to a `.env`. Use AWS Secrets Manager via `get_secret(name)` from `utils/secrets.py`.
- Do not use `print()` for diagnostics. Use `structlog`.
- Do not write your own retry logic for HTTP. Use `httpx.AsyncClient` with our shared retry transport from `utils/http.py`.

## When in doubt

- Ask before adding a new top-level dependency. Look in `pyproject.toml` first to see if something similar exists.
- For any new partner integration, check `integrations/README.md` for the standard pattern (auth, rate limiting, error handling).
- For any new Snowflake table or view, follow the naming convention: `<DOMAIN>_<SUBJECT>_<GRAIN>` (e.g. `EVENTS_PAGEVIEW_DAILY`).
```

## Decisions and reasoning

**The "Stack" section is short, but every line has saved time.** "Do not use 3.13" is there because it's a bug we already hit. Without it, Claude would happily suggest 3.13 because it's the latest. Same energy for `requests` vs `httpx`: it's a real friction point, so it's in the doc.

**SQL conventions get their own section.** In data-heavy codebases, SQL is half the surface area. Conventions for SQL are as important as conventions for Python, but most CLAUDE.md examples online ignore them.

**The "Snowflake gotchas" section is the differentiator.** These are bugs we hit, with the diagnostic insight folded in. Claude can now apply that knowledge to new bugs that look similar. This is where you put the lessons that would otherwise live in tribal knowledge or one engineer's head.

**The "What not to do" section is in negative form for a reason.** Claude defaults to common patterns (`pandas`, `print`, hand-rolled retries). Listing what we *don't* want is more efficient than listing every preferred alternative.

**The "When in doubt" section is short.** It tells Claude where to look (`pyproject.toml`, integration READMEs) instead of guessing. This routes Claude's curiosity at runtime instead of trying to pre-load everything.

## What's deliberately not included

- **A full directory structure tree.** Claude can read the directory itself. Pre-loading it bloats context and goes stale.
- **Detailed deploy instructions.** Those live in `docs/`. The CLAUDE.md points at them rather than reproducing them. Source-of-truth in one place.
- **Code examples.** No "here's a sample DAG." Claude can read existing DAGs from the repo. Including examples in CLAUDE.md doubles the maintenance burden and gets stale fast.
- **Library version pins.** Those live in `pyproject.toml`. CLAUDE.md mentions Python 3.12 because that's a version Claude can't infer from `pyproject.toml` quickly enough.
- **A glossary.** Tempting in domain-heavy codebases (especially ad-tech with its endless acronyms), but the glossary should live in `docs/glossary.md` and be referenced from CLAUDE.md, not embedded.

## How to adapt this to your project

- If you're not in ad-tech, strip the partner integrations section and replace with your domain
- Keep the structure: Project Context → Stack → Conventions → Gotchas → How To Run → What Not To Do → When In Doubt
- The "gotchas" section is the most valuable one to fill in honestly. It's empty for new projects. Add to it every time you correct Claude on the same thing twice.
- For greenfield projects, this CLAUDE.md will be 30 lines. That's correct. The gotchas accumulate over time.
