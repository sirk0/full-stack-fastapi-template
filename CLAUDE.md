# Agent Instructions

## Architecture

This project runs as a set of Docker Compose services defined in `compose.yml` (production config) and `compose.override.yml` (dev overrides applied automatically).

| Service | Description | Port(s) |
|---------|-------------|---------|
| `db` | PostgreSQL 18 database | 5432 |
| `adminer` | Database web UI | 8080 |
| `prestart` | Runs DB migrations and seeds initial data; exits when done | — |
| `backend` | FastAPI app; health-checked at `/api/v1/utils/health-check/` | 8000 |
| `frontend` | React + TypeScript (Vite) | 5173 |
| `mailcatcher` | Catches all outbound SMTP; replaces real email in dev | SMTP 1025, UI 1080 |
| `playwright` | Playwright E2E test runner container (dev override only) | 9323 |
| `proxy` | Traefik reverse proxy (dev override only) | UI 8090 |

`prestart` always runs before `backend` starts. It waits for `db` to be healthy, applies Alembic migrations, and creates the initial superuser.

## Setting Up the Dev Environment

Prerequisites:
- Docker with Compose plugin
- `uv` (Python package manager, used to run backend tooling outside containers)
- `bun` (JS runtime, used to run frontend tooling outside containers)

The `.env` file at the project root contains all configuration. It is already present with working defaults:
- `FIRST_SUPERUSER=admin@example.com`
- `FIRST_SUPERUSER_PASSWORD=changethis`
- `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` for database credentials

## Starting Services

Always start services using Docker only. Exclude `playwright` (a one-shot test runner with no default command) from the startup command:

```bash
docker compose up --build --wait backend frontend adminer mailcatcher
```

`--build` rebuilds images if source has changed. `--wait` blocks until all health checks pass (backend, db, frontend, adminer). On first run this takes ~30 seconds while `prestart` runs migrations automatically.

## Stopping Services

```bash
# Stop containers, keep the database volume
docker compose down

# Stop containers and wipe the database volume (full reset)
docker compose down -v --remove-orphans
```

## Checking Service Logs

```bash
# All services
docker compose logs

# Single service
docker compose logs backend

# Follow live output
docker compose logs -f backend
```

## Database Access

**Adminer web UI** — http://localhost:8080
- System: `PostgreSQL`
- Server: `db`
- Username / Password / Database: values from `.env` (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`)

**Direct psql connection** (while stack is running):
```bash
docker compose exec db psql -U $POSTGRES_USER $POSTGRES_DB
```

**Host connection** — `localhost:5432` with credentials from `.env` (exposed by `compose.override.yml`).

## OpenAPI Specification

The backend exposes a full OpenAPI 3.1 spec. Three ways to access it:

**Interactive Swagger UI** (stack must be running):
```
http://localhost:8000/docs
```

**ReDoc** (alternative read-only viewer):
```
http://localhost:8000/redoc
```

**Raw JSON spec** (stack must be running):
```bash
curl http://localhost:8000/openapi.json
```

**Committed static copy** — `frontend/openapi.json` is checked into the repo and used to generate the TypeScript client. It may lag behind the live backend by one `generate-client` run:
```bash
# Regenerate the TypeScript client from the live spec (stack must be running)
bash scripts/generate-client.sh
```

## Running Backend Tests

Mirrors the CI workflow in `.github/workflows/test-backend.yml`.

```bash
# 1. Start only the services the tests need
docker compose up --build --wait db mailcatcher

# 2. Apply migrations and seed initial data
cd backend && uv run bash scripts/prestart.sh

# 3. Run the test suite (pytest + coverage)
cd backend && uv run bash scripts/tests-start.sh "Coverage"

# 4. Tear down after tests
docker compose down -v --remove-orphans
```

### Proof Tests Passed

After step 3 completes, generate the coverage gate report:

```bash
cd backend && uv run coverage report --fail-under=90
```

Passing output lists per-module coverage and ends with a `TOTAL` line showing ≥ 90%. Exit code 0 means pass. The HTML report is written to `backend/htmlcov/index.html`.

## Running Playwright E2E Tests

The full stack must be running. Playwright tests run inside Docker using the `playwright` service defined in `compose.override.yml`.

```bash
# 1. Start the long-running services (exclude playwright — it's a one-shot test runner)
docker compose up --build --wait backend frontend mailcatcher

# 2. Run E2E tests via the playwright container
docker compose run --rm playwright bun run test
```

The `playwright` service connects to `backend` and `mailcatcher` over the internal Docker network.

### Proof Tests Passed

Playwright prints a summary table at the end of the run. All rows show `passed` and the process exits with code 0. Test artifacts are written to mounted volumes:
- `frontend/test-results/` — per-test traces and screenshots
- `frontend/blob-report/` — blob report for CI merging

To open the HTML report locally after a run:

```bash
cd frontend && bunx playwright show-report test-results
```
