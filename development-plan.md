# Utility Asset Management — Phased Development Plan

> Project: 243-utility-asset-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises the project's research artefacts (`research.md`, `features.md`, `standards.md`, `README.md`) and data model proposal #1 (IEC CIM-aligned normalised relational, with PostGIS) into a sequenced, implementable specification. The plan targets an open-source, self-hostable alternative to the proprietary GIS + EAM stacks (Esri ArcGIS Utility Network, IBM Maximo, Trimble Unity, Hexagon HxGN EAM) used by electric, gas, and water utilities. Standards alignment (IEC 61968/61970 CIM, ISO 55000, OGC API Features, GeoJSON RFC 7946, OpenAPI 3.1, NERC CIP, OAuth 2.0 / OIDC, W3C SOSA/SSN) is non-negotiable and is built in from Phase 1.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary backend language | **Python 3.12+** | Domain has heavy ML/LLM workloads (RUL scoring, NLQ, computer vision); Python has best-in-class geospatial libraries (Shapely, GeoPandas, Rasterio), AI/ML libraries (scikit-learn, PyTorch, ultralytics), and Anthropic/OpenAI SDKs. |
| API framework | **FastAPI** | Native OpenAPI 3.1 schema generation (mandated by `standards.md`); Pydantic v2 validation for GeoJSON payloads; async-first matches sensor ingestion workloads; auto-generated client SDKs. |
| ORM / DB layer | **SQLAlchemy 2.x (async) + GeoAlchemy2** | Mature PostGIS bindings (geometry types, ST_* functions in Python); Alembic for migrations; required for CIM-style class-table-inheritance. |
| Database | **PostgreSQL 16 + PostGIS 3.4 + pgvector + TimescaleDB extensions** | PostGIS is the open standard for spatial data and supports OGC API Features serving; pgvector for embedding-based NLQ; TimescaleDB hypertables replace manual partitioning for `sensor_observation`. Single database avoids polyglot persistence complexity. |
| Auth | **Authlib (server) + OAuth 2.0 / OpenID Connect** | Required by `standards.md`; integrates with Keycloak, Azure AD, Okta, Auth0 used by enterprise utilities. JWT bearer tokens for API auth, OIDC for browser login. |
| Background workers | **Arq (Redis-backed)** | Lightweight async task queue; fits AI scoring jobs, climate-risk batch overlays, PM schedule generation. Simpler than Celery for an MVP. |
| Caching / queue broker | **Redis 7** | Arq broker plus query-result caching for spatial layers and trace results. |
| Object storage | **S3-compatible (MinIO for self-host, AWS S3 / GCS / R2 in cloud)** | Inspection photos, drone imagery, document attachments. `boto3` client works against any S3-compatible store. |
| LLM provider abstraction | **LiteLLM** | Provider-agnostic; supports Claude (default), GPT-4, local Llama via Ollama. Required for the NLQ feature and image defect captioning. |
| Computer vision | **ultralytics (YOLOv8/YOLO-NAS) + OpenCV** | Open-weights object detection for corrosion, vegetation, structural defects; runs on CPU or GPU. |
| Frontend | **Next.js 15 (App Router) + React 19 + TypeScript** | Map-centric SPA with server components for asset-detail pages; widely-supported PWA capability for offline field use. |
| Mapping library | **MapLibre GL JS** | Open-source fork of Mapbox GL JS; supports vector tiles, OGC WMS, GeoJSON layers; no per-tile fees, unlike Mapbox/Google. |
| Vector tile server | **pg_tileserv (CrunchyData)** | Serves MVT directly from PostGIS tables; zero ETL for keeping map in sync with asset edits. |
| OGC API Features server | **pygeoapi** | Open-source reference implementation of OGC API Features, Records, Coverages; wraps PostGIS tables as OGC endpoints. |
| Mobile / offline field app | **Next.js PWA + Service Worker + IndexedDB + GeoPackage download** | PWA approach avoids dual native codebases; GeoPackage (OGC standard) is the field interchange format mandated by `standards.md`. |
| Testing — backend | **pytest + pytest-asyncio + pytest-postgresql + httpx.AsyncClient** | Standard FastAPI testing stack; `testcontainers-python` spins up PostGIS for integration tests. |
| Testing — frontend | **Vitest + React Testing Library + Playwright** | Vitest for unit/component, Playwright for end-to-end browser tests including map interactions. |
| Code quality | **ruff (lint+format) + mypy (strict) + pre-commit** for Python; **ESLint + Biome + tsc --strict** for TypeScript | Modern, fast toolchain; type-checking is non-negotiable for a regulated-industry platform. |
| Package management | **uv (Python) + pnpm (JS)** | `uv` for fast deterministic Python deps; `pnpm` for monorepo workspace support. |
| Containerisation | **Docker + docker-compose (dev) + Helm chart (prod)** | Single-command local bring-up; Kubernetes-native deployment for utility customers. |
| CI/CD | **GitHub Actions** | Free for open source, matrix builds across Python versions and PostGIS versions. |
| MCP server | **`mcp` Python SDK** | Exposes asset register, work orders, network trace as MCP tools — a key AI-native differentiator per `standards.md`. |
| Spec compliance test rig | **Schemathesis (OpenAPI fuzzing) + OGC API CITE test suite** | Validates that the API actually conforms to the standards it claims. |

### Project Structure

```
utility-asset-management/
├── README.md
├── LICENSE                                # AGPL-3.0 (planned)
├── CONTRIBUTING.md
├── .github/
│   └── workflows/
│       ├── backend-ci.yml
│       ├── frontend-ci.yml
│       └── docker-publish.yml
├── docker-compose.yml                     # PostGIS + Redis + MinIO + Keycloak + backend + frontend + workers
├── docker-compose.override.yml.example
├── .env.example
├── deploy/
│   └── helm/
│       └── utility-asset-management/      # Helm chart for prod
├── docs/
│   ├── architecture.md
│   ├── data-model.md
│   ├── api/openapi.json                   # Generated, checked in
│   └── compliance/
│       ├── iso-55000-mapping.md
│       └── nerc-cip-mapping.md
├── backend/
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── migrations/                        # Alembic
│   │   └── versions/
│   ├── src/
│   │   └── uam/                           # Package: utility-asset-management
│   │       ├── __init__.py
│   │       ├── main.py                    # FastAPI app entry
│   │       ├── config.py                  # pydantic-settings
│   │       ├── db.py                      # Async engine, sessionmaker
│   │       ├── deps.py                    # FastAPI dependencies (auth, db, etc.)
│   │       ├── auth/
│   │       │   ├── oidc.py
│   │       │   ├── jwt.py
│   │       │   ├── rbac.py
│   │       │   └── models.py
│   │       ├── models/                    # SQLAlchemy ORM
│   │       │   ├── base.py
│   │       │   ├── organisation.py
│   │       │   ├── location.py
│   │       │   ├── asset.py
│   │       │   ├── equipment_detail.py    # transformer_info, pipe_info, etc.
│   │       │   ├── topology.py
│   │       │   ├── work_order.py
│   │       │   ├── inspection.py
│   │       │   ├── sensor.py
│   │       │   ├── inventory.py
│   │       │   ├── compliance.py
│   │       │   ├── ai_analytics.py
│   │       │   ├── document.py
│   │       │   └── service_request.py
│   │       ├── schemas/                   # Pydantic v2
│   │       │   ├── common.py              # GeoJSON, pagination, error envelopes
│   │       │   ├── asset.py
│   │       │   ├── work_order.py
│   │       │   └── ...
│   │       ├── api/
│   │       │   ├── v1/
│   │       │   │   ├── router.py
│   │       │   │   ├── assets.py
│   │       │   │   ├── work_orders.py
│   │       │   │   ├── inspections.py
│   │       │   │   ├── sensors.py
│   │       │   │   ├── topology.py        # Network trace endpoints
│   │       │   │   ├── inventory.py
│   │       │   │   ├── compliance.py
│   │       │   │   ├── analytics.py       # AI predictions, NLQ
│   │       │   │   ├── capital.py         # Capital programme & candidates
│   │       │   │   ├── climate_risk.py
│   │       │   │   ├── service_requests.py
│   │       │   │   ├── documents.py
│   │       │   │   └── audit.py
│   │       │   └── ogcapi/                # pygeoapi integration / OGC endpoints
│   │       │       └── features.py
│   │       ├── services/                  # Domain services
│   │       │   ├── asset_service.py
│   │       │   ├── work_order_service.py
│   │       │   ├── inspection_service.py
│   │       │   ├── scoring.py             # Inspection -> condition score
│   │       │   ├── trace.py               # Network trace algorithms
│   │       │   ├── pm_engine.py           # Preventive maintenance schedule -> WO
│   │       │   ├── sensor_ingest.py
│   │       │   ├── rul_model.py           # Remaining-useful-life
│   │       │   ├── cv_defect.py           # Computer vision
│   │       │   ├── nlq.py                 # Natural language query
│   │       │   ├── capital_optimiser.py
│   │       │   ├── climate_overlay.py
│   │       │   └── audit_service.py
│   │       ├── workers/                   # Arq workers
│   │       │   ├── settings.py
│   │       │   ├── pm_jobs.py
│   │       │   ├── scoring_jobs.py
│   │       │   ├── cv_jobs.py
│   │       │   └── overlay_jobs.py
│   │       ├── integrations/
│   │       │   ├── scada.py
│   │       │   ├── erp_sap.py
│   │       │   ├── geopackage_io.py       # Field sync
│   │       │   └── llm.py                 # LiteLLM wrapper
│   │       ├── mcp_server/
│   │       │   └── server.py
│   │       └── observability/
│   │           ├── logging.py
│   │           ├── tracing.py             # OpenTelemetry
│   │           └── metrics.py
│   └── tests/
│       ├── conftest.py
│       ├── fixtures/
│       │   ├── sample_network.geojson
│       │   ├── inspection_template.json
│       │   └── ...
│       ├── unit/
│       ├── integration/
│       └── e2e/
├── frontend/
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── Dockerfile
│   ├── next.config.ts
│   ├── tsconfig.json
│   ├── public/
│   │   └── manifest.webmanifest           # PWA manifest
│   ├── src/
│   │   ├── app/                           # Next.js App Router
│   │   │   ├── (auth)/login/page.tsx
│   │   │   ├── (app)/
│   │   │   │   ├── layout.tsx             # Authenticated shell
│   │   │   │   ├── map/page.tsx           # Map-centric main UI
│   │   │   │   ├── assets/[id]/page.tsx
│   │   │   │   ├── work-orders/...
│   │   │   │   ├── inspections/...
│   │   │   │   ├── sensors/...
│   │   │   │   ├── capital/...
│   │   │   │   ├── compliance/...
│   │   │   │   └── nlq/page.tsx           # Natural language query
│   │   │   └── api/                       # Next.js API routes for BFF if needed
│   │   ├── components/
│   │   │   ├── map/MapView.tsx            # MapLibre GL wrapper
│   │   │   ├── map/AssetLayer.tsx
│   │   │   ├── map/TraceOverlay.tsx
│   │   │   └── ...
│   │   ├── lib/
│   │   │   ├── api-client.ts              # Generated from OpenAPI
│   │   │   ├── auth.ts                    # OIDC client
│   │   │   └── offline/
│   │   │       ├── service-worker.ts
│   │   │       ├── geopackage.ts          # SQL.js + GeoPackage
│   │   │       └── sync.ts
│   │   └── types/
│   └── tests/
│       ├── unit/
│       └── e2e/                           # Playwright
├── packages/                              # Shared TS packages
│   └── api-types/                         # Generated from OpenAPI
└── scripts/
    ├── seed.py                            # Demo data seeder
    ├── load_climate_zones.py
    └── generate-openapi.py
```

The structure is monorepo with two deployable apps (`backend`, `frontend`) plus a shared types package. Each domain (assets, work orders, sensors, etc.) is a module under `backend/src/uam/`, organised by layer (model / schema / api / service). Phases add files to existing directories rather than restructuring.

---

## Phase 1: Foundation & Core Infrastructure

### Purpose
Establish the project skeleton, development environment, CI/CD, configuration management, database connectivity, authentication scaffolding, observability, and the OpenAPI surface. After this phase a developer can run `docker compose up` and get a working empty FastAPI service backed by PostGIS, Keycloak, Redis, and MinIO, with a passing CI build, `/healthz` and `/version` endpoints, and a generated OpenAPI 3.1 document. No domain entities yet — this is purely platform foundation.

### Tasks

#### 1.1 — Repository scaffold, tooling, and monorepo layout

**What**: Initialise the monorepo, configure tooling (`uv`, `pnpm`, `pre-commit`, `ruff`, `mypy`, `eslint`, `biome`, `tsc`), and define `pyproject.toml` / `package.json` files for backend and frontend.

**Design**:
- `pyproject.toml` declares Python 3.12 minimum, project name `uam`, dependencies grouped:
  - runtime: `fastapi>=0.115`, `uvicorn[standard]`, `sqlalchemy[asyncio]>=2.0`, `geoalchemy2>=0.15`, `asyncpg`, `alembic`, `pydantic>=2.7`, `pydantic-settings`, `authlib`, `python-jose[cryptography]`, `httpx`, `redis>=5`, `arq`, `boto3`, `litellm`, `mcp`, `shapely`, `pyproj`, `geojson-pydantic`.
  - dev: `pytest`, `pytest-asyncio`, `pytest-cov`, `httpx`, `testcontainers[postgres]`, `schemathesis`, `ruff`, `mypy`, `types-redis`, `types-boto3`.
- `ruff.toml`: target `py312`, enable `E,F,I,UP,B,SIM,RUF,N`, line length 100.
- `mypy.ini`: `strict = True`, `plugins = pydantic.mypy, sqlalchemy.ext.mypy.plugin`.
- `frontend/package.json`: Next 15.x, React 19, TypeScript 5.5, `maplibre-gl`, `@tanstack/react-query`, `zustand`, `openapi-fetch`.
- Root `package.json` defines `pnpm-workspace.yaml` listing `frontend`, `packages/*`.
- `.pre-commit-config.yaml` runs ruff, mypy, eslint, biome, and `detect-secrets`.

**Testing**:
- Unit: `ruff check .` exits 0 on the scaffold.
- Unit: `mypy backend/src` exits 0 on the scaffold.
- Unit: `pnpm -C frontend lint` exits 0.
- Integration: `pre-commit run --all-files` passes on a clean checkout.

#### 1.2 — docker-compose dev environment

**What**: Provide a one-command local environment running PostGIS, Redis, MinIO, Keycloak, the backend, the frontend, and Arq workers.

**Design**:
- `docker-compose.yml` services:
  - `postgres`: `postgis/postgis:16-3.4`, ports `5432`, volume `postgres_data`, init script installs `pgvector` and `timescaledb` extensions.
  - `redis`: `redis:7-alpine`.
  - `minio`: `minio/minio` with default bucket `uam-attachments`.
  - `keycloak`: `quay.io/keycloak/keycloak:25.0` start-dev, realm `uam` imported from `deploy/keycloak/realm-export.json`.
  - `backend`: built from `backend/Dockerfile`, depends on `postgres`, `redis`, `minio`, `keycloak`; mounts source for hot-reload via `uvicorn --reload`.
  - `worker`: same image as backend, command `arq uam.workers.settings.WorkerSettings`.
  - `frontend`: built from `frontend/Dockerfile`, `pnpm dev`.
- `.env.example` enumerates `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `LLM_PROVIDER`, `LLM_API_KEY`, `SECRET_KEY`.

**Testing**:
- Integration: `docker compose up -d && curl -fsS http://localhost:8000/healthz` returns `200 {"status":"ok"}` within 30 s.
- Integration: connecting to PostGIS shows `postgis`, `pgvector`, `timescaledb` extensions present.
- Integration: Keycloak realm import succeeds (assertion on `/realms/uam/.well-known/openid-configuration` returning 200).

#### 1.3 — Configuration, logging, and observability

**What**: Implement the `Settings` class, structured logging, request-id middleware, and OpenTelemetry tracing/metrics scaffolding.

**Design**:
```python
# uam/config.py
from pydantic import PostgresDsn, RedisDsn, AnyHttpUrl
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "Utility Asset Management"
    environment: Literal["dev","test","staging","prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str
    s3_access_key: str
    s3_secret_key: str
    s3_bucket: str = "uam-attachments"
    oidc_issuer: AnyHttpUrl
    oidc_client_id: str
    oidc_client_secret: str
    oidc_audience: str = "uam-api"
    secret_key: str
    llm_provider: str = "anthropic"
    llm_model: str = "claude-opus-4-5"
    llm_api_key: str | None = None
    otel_endpoint: AnyHttpUrl | None = None

    model_config = {"env_file": ".env"}
```
- `uam/observability/logging.py`: `structlog` JSON logger with timestamp, level, logger, event, request_id, user_id.
- Request-ID middleware sets `X-Request-Id` header; bound into `structlog` context.
- OpenTelemetry instrumentation via `opentelemetry-instrumentation-fastapi` and `opentelemetry-instrumentation-sqlalchemy` exporting OTLP when `OTEL_ENDPOINT` set.

**Testing**:
- Unit: missing required env var raises `pydantic.ValidationError` on `Settings()`.
- Unit: log record contains `request_id` matching response header.
- Integration: hitting `/healthz` produces a single OTLP span with attribute `http.route=/healthz`.

#### 1.4 — Database engine, Alembic, and base ORM

**What**: Async SQLAlchemy engine, session dependency, Alembic configured for autogenerate with PostGIS types, base ORM class with shared fields.

**Design**:
- `uam/db.py`:
```python
engine = create_async_engine(settings.database_url, echo=False, pool_pre_ping=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionLocal() as session:
        yield session
```
- `uam/models/base.py`:
```python
class Base(DeclarativeBase):
    metadata = MetaData(schema="uam")

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now(), nullable=False)
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(),
                                                 onupdate=func.now(), nullable=False)

class UUIDPK:
    id: Mapped[UUID] = mapped_column(PG_UUID(as_uuid=True),
                                     primary_key=True, server_default=func.gen_random_uuid())
```
- `alembic/env.py` configured for async engine; `compare_type=True`, `include_object` filter excludes PostGIS internal tables; `geoalchemy2` types registered.
- Initial migration `0001_init.py` enables extensions: `CREATE EXTENSION IF NOT EXISTS postgis; pgvector; timescaledb; pgcrypto;` and creates the `uam` schema.

**Testing**:
- Unit: `Base.metadata.tables` contains only `uam`-schema tables.
- Integration (testcontainers): `alembic upgrade head` on empty DB succeeds; extensions are present.
- Integration: `get_session()` fixture yields a session that can `SELECT 1`.

#### 1.5 — Auth scaffolding (OAuth 2.0 / OIDC)

**What**: Dependency that validates JWT bearer tokens from Keycloak, extracts user identity and roles, and provides current-user object to endpoints. No user CRUD yet — users come from Keycloak.

**Design**:
- `uam/auth/jwt.py`: caches Keycloak JWKS for 10 minutes, validates `iss`, `aud`, `exp`; uses `python-jose`.
- `uam/auth/models.py`:
```python
class CurrentUser(BaseModel):
    subject: UUID            # Keycloak sub
    email: EmailStr
    display_name: str
    roles: set[str]
    organisation_id: UUID | None  # from token claim 'organisation'
```
- `uam/deps.py`:
```python
async def current_user(token: str = Depends(oauth2_scheme)) -> CurrentUser: ...
async def require_role(role: str) -> Callable: ...
```
- `permissions` table seeded in migration; RBAC is "user has role; role has permissions" — the role names come from token, permissions are checked DB-side.

**Testing**:
- Unit: valid signed JWT → returns `CurrentUser` with correct claims.
- Unit: expired JWT → `HTTPException(401)`.
- Unit: missing audience → `HTTPException(401)`.
- Integration: `GET /me` with valid Keycloak token returns user; without token returns 401.

#### 1.6 — OpenAPI 3.1 surface and Schemathesis spec gate

**What**: Configure FastAPI to emit OpenAPI 3.1 with full info, security schemes, error envelopes; commit a baseline `openapi.json`; wire Schemathesis to fuzz against the spec in CI.

**Design**:
- `FastAPI(openapi_version="3.1.0", title=..., contact={...}, license_info={"name":"AGPL-3.0"})`.
- Common error schema:
```python
class ErrorEnvelope(BaseModel):
    type: str           # RFC 7807-ish, e.g. "https://uam/errors/not-found"
    title: str
    status: int
    detail: str | None = None
    instance: str | None = None
    request_id: str
```
- Security scheme `oauth2_keycloak` with authorization-code + refresh flow and PKCE.
- `scripts/generate-openapi.py` writes `docs/api/openapi.json` from the app; CI fails if checked-in spec is stale.
- Schemathesis job: `schemathesis run --checks all docs/api/openapi.json --base-url=http://localhost:8000`.

**Testing**:
- Unit: `app.openapi()["openapi"] == "3.1.0"`.
- Unit: generated spec file matches checked-in file (exit 1 otherwise).
- Integration: Schemathesis run against `/healthz`, `/version`, `/me` passes.

#### 1.7 — Frontend shell and OIDC login

**What**: Next.js 15 App Router scaffold with Keycloak login, authenticated layout, OpenAPI-generated typed client, and an empty map page.

**Design**:
- `@auth/keycloak-provider` (NextAuth v5) for browser OIDC.
- Generated client via `openapi-typescript` reading `docs/api/openapi.json` → `packages/api-types/`.
- `MapView` component renders MapLibre with an OSM raster fallback basemap (no API key) and a placeholder vector source pointing at `tiles/{z}/{x}/{y}.pbf` (404 acceptable in Phase 1).

**Testing**:
- Unit (Vitest): `MapView` renders a `<div>` with id `map`.
- E2E (Playwright): visiting `/` while unauthenticated redirects to Keycloak login; after login returns to `/map` and renders the basemap.

---

## Phase 2: Identity, Organisation, and Spatial Foundations

### Purpose
Build the identity / organisation / RBAC tables and the spatial location model that everything else depends on. After this phase the platform can store organisations, jurisdictions, users (synced from Keycloak), roles/permissions, and locations with PostGIS geometry. The map can show locations.

### Tasks

#### 2.1 — Organisation, jurisdiction, app_user, role, permission tables

**What**: Implement migrations and ORM models for the Core Identity & Organisation group from data-model-suggestion-1.

**Design**:
- Tables: `organisation`, `jurisdiction`, `app_user`, `role`, `user_role`, `permission`, `role_permission` (DDL as in suggestion-1 §Core Identity).
- Seed migration loads ISO 3166-1 alpha-2 country codes into `jurisdiction` from a CSV in `migrations/data/iso_3166.csv`.
- Default roles seeded: `admin`, `asset_manager`, `gis_analyst`, `field_supervisor`, `field_crew`, `inspector`, `auditor`, `read_only`.
- Default permissions seeded as `(resource, action)` tuples for every domain entity × `{read, write, delete, approve}`.

**Testing**:
- Unit: organisation tree with 3 levels round-trips through ORM.
- Unit: granting a role to a user without `organisation_id` raises `IntegrityError`.
- Integration: after migration, `SELECT count(*) FROM jurisdiction WHERE country_code='US'` == 1.
- Integration: `SELECT count(*) FROM role` == 8.

#### 2.2 — User provisioning service (Keycloak → app_user)

**What**: On first authenticated request from a Keycloak user, upsert the corresponding `app_user` row with email, display_name, and `organisation_id` derived from token claim or default organisation.

**Design**:
- `services/user_provisioning.py::ensure_app_user(claims) -> AppUser`.
- Resolves `organisation_id` from claim `organisation`; if missing, assigns to a `default` organisation seeded at startup.
- Maps Keycloak realm roles → app roles by name; missing roles get added with empty permissions.

**Testing**:
- Unit: new subject → new `app_user` row inserted.
- Unit: existing subject with changed display_name → row updated.
- Integration: hitting `/me` twice with same token results in exactly one `app_user` row.

#### 2.3 — Location + linear_reference_system

**What**: Implement Location & Spatial Model tables and CRUD endpoints serving and accepting GeoJSON.

**Design**:
- Tables per data-model-suggestion-1 §Location.
- Pydantic schemas use `geojson-pydantic`:
```python
class LocationCreate(BaseModel):
    name: str | None
    location_type: LocationType
    address_line1: str | None
    geom: Feature[Geometry, dict] | None
```
- Endpoints (all under `/api/v1`):
  - `POST /locations` — create
  - `GET /locations/{id}` — read; returns GeoJSON Feature
  - `GET /locations?bbox=&type=&jurisdiction_id=&page=` — paginated; bbox uses `ST_Intersects` against `geom`.
  - `PATCH /locations/{id}`
  - `DELETE /locations/{id}`
- All write endpoints require permission `location:write`.
- LRS endpoints under `/linear-reference-systems`.

**Testing**:
- Unit: creating location with `geom` outside SRID 4326 raises validation error.
- Unit: `LocationCreate` with `location_type='unknown'` → ValidationError.
- Integration: `GET /locations?bbox=-122.5,37.7,-122.4,37.8` returns only locations whose geom intersects.
- Integration: response Content-Type is `application/geo+json` when client sends `Accept: application/geo+json`.
- E2E: round-trip create → fetch → patch → delete via HTTP, with assertions on returned GeoJSON.

#### 2.4 — Audit log infrastructure (write-only)

**What**: Implement `audit_log` partitioned table and a SQLAlchemy event hook that captures before/after JSON for every INSERT/UPDATE/DELETE on listed tables, plus an `audit_service` for explicit business events (approve, login, export).

**Design**:
- `audit_log` per suggestion-1 §Compliance & Audit, `PARTITION BY RANGE (created_at)`; create 12 future monthly partitions in initial migration; cron job in Phase 9 maintains.
- `audit_service.record(action, entity_type, entity_id, old, new, request_id, user_id, ip)`.
- ORM-level listener `_audit_listener(mapper, connection, target)` registered for `Asset`, `WorkOrder`, `Inspection`, `Location`, `User`, `Role`, `Permission`.
- Tamper-evidence: each audit row gets a `prev_hash` and `row_hash` (SHA-256 chain over `(prev_hash, payload)`) appended via trigger; integrity verifiable.

**Testing**:
- Unit: updating a `Location` produces one audit row with `action='update'`, populated `old_values` and `new_values`.
- Unit: `hash_chain` verifier rejects a row with mutated payload.
- Integration: 1000 concurrent inserts produce monotonically non-decreasing audit rowcount.

---

## Phase 3: Asset Register

### Purpose
Build the heart of the platform — the asset register. After this phase users can create, search, classify, hierarchise, and locate assets; can attach typed attributes; can record equipment-class detail (transformer, pipe, valve, etc.); and can model linear assets with section-level attributes. The map shows assets coloured by type and condition.

### Tasks

#### 3.1 — Asset type catalogue and CIM mapping seeds

**What**: Implement `asset_type`, `asset_type_material` tables and seed them with CIM-aligned defaults for electric, gas, water utility types.

**Design**:
- Per suggestion-1 §Asset Register.
- Seed file `migrations/data/asset_types.yaml` lists ~80 types covering electric (`PowerTransformer`, `ACLineSegment`, `LoadBreakSwitch`, `Recloser`, `Pole`, `Capacitor`, `Regulator`), water (`Pipe`, `Hydrant`, `Valve`, `PressureReducingValve`, `PumpStation`, `Reservoir`), gas (`Regulator`, `MeterStation`, `Pipe`, `Valve`).
- Each entry maps `cim_class`, `utility_type`, `category`, `expected_life_years`, `is_linear`.
- `GET /asset-types` and `GET /asset-types/{id}` endpoints; `POST/PATCH/DELETE` admin-only.

**Testing**:
- Unit: YAML loader rejects entry missing `utility_type`.
- Integration: after seed, `count(*) where utility_type='electric'` ≥ 25.

#### 3.2 — Asset CRUD with spatial, hierarchy, attribute support

**What**: Implement `asset` and `asset_attribute` tables, ORM, schemas, and the full REST CRUD with hierarchy traversal endpoints.

**Design**:
- Table per suggestion-1 §Asset Register.
- Pydantic schemas:
```python
class AssetCreate(BaseModel):
    asset_code: str
    asset_type_id: UUID
    parent_id: UUID | None = None
    organisation_id: UUID
    location_id: UUID | None = None
    name: str
    description: str | None = None
    serial_number: str | None = None
    manufacturer: str | None = None
    model_number: str | None = None
    install_date: date | None = None
    commission_date: date | None = None
    expected_disposal: date | None = None
    status: AssetStatus = AssetStatus.PLANNED
    criticality: Criticality | None = None
    replacement_cost: Decimal | None = None
    currency_code: str = "USD"
    geom: Feature[Geometry, dict] | None = None
    attributes: dict[str, str] = {}
```
- Endpoints:
  - `POST /assets`
  - `GET /assets/{id}` — includes nested `attributes` and `equipment_detail` (populated by type-specific join)
  - `GET /assets?bbox=&type=&status=&q=&page=` — `q` does ILIKE on `asset_code` / `name` / `serial_number`.
  - `PATCH /assets/{id}`
  - `DELETE /assets/{id}` — soft-deletes by setting `status='disposed'` unless `?hard=true` and user has `asset:delete`.
  - `GET /assets/{id}/descendants?depth=N` — recursive CTE per suggestion-1.
  - `GET /assets/{id}/ancestors` — walk up `parent_id`.
- Asset state machine enforced in service: allowed transitions `planned→procurement→installed→active→{degraded,out_of_service,decommissioned}→disposed`.

**Testing**:
- Unit: invalid transition `active→procurement` → `DomainError`.
- Unit: duplicate `asset_code` → 409 Conflict.
- Integration: descendants endpoint on 4-level tree returns correct nodes in correct order.
- Integration: bbox query returns only assets whose geom intersects.
- Fixture-based: import 200 assets from `tests/fixtures/sample_network.geojson`, verify counts and geometry types.

#### 3.3 — Equipment-class detail tables and polymorphic detail resolution

**What**: Implement `transformer_info`, `conductor_info`, `switch_info`, `pipe_info`, `valve_info`, `hydrant_info`, `gas_regulator_info` per suggestion-1.

**Design**:
- ORM uses joined-table inheritance keyed by `asset_type.cim_class`.
- Service `asset_service.resolve_detail(asset)` returns the appropriate Pydantic union model.
- `PATCH /assets/{id}/detail` accepts a discriminated union determined by the asset's `cim_class`.

**Testing**:
- Unit: storing transformer detail with `phase_count=2` → CHECK constraint violation.
- Unit: detail resolver returns `TransformerInfo` for asset with `cim_class='PowerTransformer'`.
- Integration: PATCH detail for asset whose `cim_class` doesn't match payload → 422.

#### 3.4 — Linear asset sections

**What**: Implement `linear_asset_section` table, validation that `from_measure_m < to_measure_m`, and a service to derive `section_geom` from the parent LRS geometry using `ST_LineSubstring`.

**Design**:
- Endpoints under `/assets/{id}/linear-sections` (POST/GET/PATCH/DELETE).
- Server computes `section_geom` if not provided.

**Testing**:
- Unit: from > to → ValidationError.
- Integration: creating a section without `section_geom` results in geometry computed from LRS route.
- Integration: GET sections returns ordered by `from_measure_m`.

#### 3.5 — Vector tile endpoint for assets

**What**: Run `pg_tileserv` as a sidecar container; expose a Next.js-side proxy at `/tiles/assets/{z}/{x}/{y}.pbf` with auth-checked access.

**Design**:
- `pg_tileserv` config publishes view `mvt_asset` (joins asset + asset_type + condition_score + geom).
- Backend issues short-lived signed JWT for tile requests; pg_tileserv runs in private network.
- Map page renders MVT layer; clicking an asset opens a side panel calling `/assets/{id}`.

**Testing**:
- Integration: tile request for known bbox returns non-empty MVT (`Content-Type: application/x-protobuf`, body length > 0).
- E2E (Playwright): clicking a feature on the map opens the asset detail panel with expected name.

---

## Phase 4: Work Order & Inspection Engines

### Purpose
Deliver the operational workhorse: corrective, preventive, and condition-based work orders, plus a configurable inspection engine that computes condition scores from weighted question responses and auto-creates follow-up work orders for defects. After this phase a field supervisor can plan and assign work, and inspectors can complete templated inspections that feed asset condition.

### Tasks

#### 4.1 — Work order model, state machine, REST API

**What**: Implement `work_order_type`, `work_order`, `work_order_task`, `work_order_labour` tables; build the work order REST API and state machine.

**Design**:
- Tables per suggestion-1 §Work Order Management.
- State machine: `draft → pending_approval → approved → scheduled → in_progress → {on_hold ↔ in_progress} → completed → closed`; `cancelled` reachable from any pre-`completed` state.
- Service enforces:
  - `approved` requires permission `work_order:approve`.
  - `completed` requires all required tasks have `status='done'`.
  - `actual_cost` auto-computed from labour + material lines at `completed`.
- Endpoints under `/work-orders`:
  - CRUD as above.
  - `POST /work-orders/{id}/transition {to: "approved", note: "..."}`.
  - `POST /work-orders/{id}/tasks`, `PATCH /work-orders/{id}/tasks/{taskId}`.
  - `POST /work-orders/{id}/labour` (logs time).
- WO numbering: `format = 'WO-{yy}-{nnnnnn}'` from a Postgres sequence.

**Testing**:
- Unit: transition `draft → in_progress` raises `InvalidTransition`.
- Unit: completing a WO without finishing required tasks → 422.
- Unit: cost calculation: 3 labour rows @ 1h $50 + 2 material rows of $20 → `actual_cost == 190`.
- Integration: concurrent POST /work-orders generate distinct, sequential numbers.

#### 4.2 — Preventive maintenance engine

**What**: Implement `pm_schedule` table and Arq job that wakes nightly to generate WOs for schedules where `next_due <= now + lead_time_days`.

**Design**:
- Trigger types: `time` (interval_days), `meter` (sensor threshold), `condition` (asset condition_score below threshold).
- `services/pm_engine.py::evaluate_schedules(now)` returns list of `(schedule, target_asset)` to create WOs for; idempotent — does not re-create a WO if an open WO for that schedule+asset exists.
- Arq cron `pm_evaluator` runs `0 2 * * *`.

**Testing**:
- Unit: schedule with `interval_days=30` and `last_triggered=30 days ago` triggers; with `last_triggered=29 days ago` does not.
- Unit: meter-based schedule with `meter_threshold=10000` and sensor latest reading `9500` → no trigger.
- Integration: running evaluator twice in a row generates WOs only the first time.

#### 4.3 — Inspection templates and inspections

**What**: Implement template/template-item/inspection/inspection-response/inspection-attachment tables, a scoring service, and endpoints for template management and inspection execution.

**Design**:
- Tables per suggestion-1 §Inspection.
- Scoring algorithm (`services/scoring.py`):
```python
def compute_score(responses: list[InspectionResponse], items: list[TemplateItem]) -> tuple[Decimal, str]:
    total_weight = sum(i.weight for i in items if i.is_required)
    earned = sum(r.score_contribution * i.weight for r,i in pairs)
    score = (earned / total_weight) * 100
    grade = "A" if score>=90 else "B" if score>=80 else "C" if score>=70 else "D" if score>=60 else "F"
    return Decimal(score), grade
```
- Each response's `score_contribution` is computed from response type:
  - boolean: True→1, False→0
  - numeric: 1 if within defect_threshold else 0
  - choice: lookup table per item
- On `inspection.status='completed'`: propagate `overall_score` to `asset.condition_score`; for every defect response, enqueue work order creation (corrective WO, severity → priority mapping).
- Attachment upload: presigned PUT URL to MinIO, callback `POST /inspections/{id}/attachments/{id}/finalise` records metadata.
- Endpoints under `/inspection-templates` and `/inspections`.

**Testing**:
- Unit: scoring with all-correct responses → 100, grade A.
- Unit: scoring with one critical defect (weight 5) out of 10 weighted items → score 50, grade F.
- Unit: completing inspection updates `asset.condition_score`.
- Integration: completing inspection with 2 defects creates 2 corrective WOs linked to the asset.
- Integration: presigned URL upload succeeds for ≤ 50 MB JPG.

#### 4.4 — Inventory + materials (MRO)

**What**: Implement storeroom / material / inventory_balance / work_order_material / asset_type_material tables and endpoints.

**Design**:
- Endpoints under `/storerooms`, `/materials`, `/inventory`, `/work-orders/{id}/materials`.
- Issuing material from storeroom debits `inventory_balance.quantity` atomically (row lock).
- Reorder service: nightly job flags materials where `quantity < reorder_point` and emits a `MaterialReorderNeeded` domain event (consumed by integrations in Phase 9).

**Testing**:
- Unit: issuing 5 of a material with quantity 3 → `InsufficientStock` error.
- Integration: concurrent issues of same material total quantity decremented correctly under serializable isolation.

---

## Phase 5: Network Topology and Trace

### Purpose
Differentiator feature: implement the connectivity-node / terminal topology, subnetworks, and the network trace algorithms (upstream, downstream, isolation, shortest path) that lift the platform from a flat CMMS into a true utility network management system.

### Tasks

#### 5.1 — Connectivity-node, terminal, subnetwork tables

**What**: Implement tables per suggestion-1 §Network Topology and CRUD endpoints for graph editing.

**Design**:
- Endpoints:
  - `POST /connectivity-nodes`, `GET /connectivity-nodes?bbox=`.
  - `POST /assets/{id}/terminals` (creates terminal at given connectivity_node).
  - `POST /subnetworks` with `source_asset_id`; `GET /subnetworks/{id}/members`.
- Edge: terminal is the join row that participates in graph topology.

**Testing**:
- Unit: creating two terminals on the same asset with identical `sequence_number` → IntegrityError.
- Integration: deleting an asset cascades terminals.

#### 5.2 — Network trace service

**What**: Implement `services/trace.py` with `downstream(asset_id, *, respect_open_switches=True, stop_at_open_devices=True)`, `upstream`, `isolation_trace(asset_id)` (find isolating switches to drop service to a fault), `shortest_path(a,b)`.

**Design**:
- Uses recursive CTE pattern from suggestion-1 §Example Queries adapted for switch states and stop conditions.
- Returns `TraceResult { assets: list[AssetSummary], edges: list[Edge], stopped_at: list[AssetSummary] }`.
- Caches results in Redis with key `trace:{type}:{root}:{state_version}` where `state_version` is a counter bumped on any switch state change; TTL 1 h.

**Testing**:
- Fixture-based: build a sample feeder of 1 source, 2 switches, 10 line segments, 3 transformers; assert downstream from source returns all when switches closed; closing switch 1 reduces set correctly.
- Unit: isolation trace on midline fault returns the two nearest sectionalising switches.
- Integration: 10k-node synthetic network traces in < 500 ms (warm cache < 50 ms).

#### 5.3 — Trace endpoints and map overlay

**What**: REST endpoints + frontend overlay that highlights traced assets on the map.

**Design**:
- `GET /topology/trace?type=downstream&asset_id=X&respect_open=true` returns `TraceResult` plus a GeoJSON FeatureCollection for direct map rendering.
- Frontend toggles a "Trace" mode; right-clicking an asset opens a trace menu; results highlighted in colour.

**Testing**:
- Integration: API returns valid GeoJSON.
- E2E (Playwright): right-click on a feeder breaker → choose "Trace downstream" → verify highlight layer renders ≥ N expected features.

#### 5.4 — Subnetwork auto-discovery job

**What**: Background job that scans the connectivity graph and updates `subnetwork_member` rows.

**Design**:
- Triggered on topology edits (debounced 30 s) and nightly.
- For each `subnetwork`, runs `downstream(source_asset_id, respect_open_switches=True)`; replaces member set in a single transaction.

**Testing**:
- Integration: changing a switch state and waiting 60 s results in updated `subnetwork_member` reflecting new partitions.

---

## Phase 6: AI-Native Capabilities (RUL, NLQ, CV, Capital Planning)

### Purpose
Ship the AI features that differentiate this platform from incumbents. After this phase the system computes remaining-useful-life scores, ranks capital replacement candidates, answers natural-language asset questions, and tags defects in inspection imagery.

### Tasks

#### 6.1 — Condition prediction and RUL model

**What**: Train (or initialise with hand-tuned heuristics) a per-asset-class RUL/failure-probability model; expose predictions via `/analytics/predictions/{asset_id}`.

**Design**:
- `services/rul_model.py`:
  - Feature extraction from asset, equipment_detail, inspections (last 5 years), failure WOs (`work_order` where `failure_class is not null`), sensor anomalies, climate risk overlay.
  - MVP model: `sklearn.ensemble.GradientBoostingRegressor` per `cim_class`, target = days-to-failure derived from historical failure dates; models stored in `models/rul/{cim_class}.joblib`.
  - Fallback heuristic when training data sparse: `RUL_days = expected_life_days * (condition_score/100) * (1 - climate_risk_multiplier)`.
- Writes to `condition_prediction` table per suggestion-1 §AI & Analytics.
- Arq job `score_all_assets` weekly; on-demand `POST /analytics/score-asset/{id}`.
- Persists `model_name`, `model_version`, `input_features` JSONB for reproducibility.

**Testing**:
- Unit: heuristic produces RUL between 0 and `expected_life_days`.
- Unit: model predict returns finite probability in [0,1].
- Integration: scoring 1000 assets stores 1000 rows; subsequent scoring inserts new rows (history retained), not overwrites.

#### 6.2 — Capital replacement prioritisation

**What**: Implement `capital_programme` and `capital_replacement_candidate` tables, a knapsack-style optimiser that ranks assets to fit within a budget.

**Design**:
- `services/capital_optimiser.py::rank(programme_id, budget, *, weights: dict)`:
  - Score = `w_risk * risk_score + w_consequence * criticality_numeric + w_age * age_ratio - w_cost_normaliser * (cost/budget)`.
  - Greedy knapsack pass; outputs candidate rows with `priority_rank` set.
- Endpoints:
  - `POST /capital-programmes` (create).
  - `POST /capital-programmes/{id}/optimise` body `{ budget, weights, candidate_asset_filter }`.
  - `GET /capital-programmes/{id}/candidates?status=`.
- Result dashboard in frontend: ranked list, total cost, risk reduction estimate.

**Testing**:
- Unit: optimiser with budget exactly equal to sum of top-N candidates includes exactly those N.
- Unit: changing `w_risk` weight reorders results.

#### 6.3 — Natural language query (LLM-backed)

**What**: Endpoint `/analytics/nlq` accepts plain English question, returns structured answer + map layer + SQL trace.

**Design**:
- `services/nlq.py`:
  - System prompt template loaded from `prompts/nlq.system.md`. Includes (a) the schema (auto-generated DDL trimmed to columns of `asset`, `asset_type`, `work_order`, `inspection`, `sensor_observation`, `condition_prediction`, `climate_risk_zone`), (b) PostGIS function cheatsheet, (c) safety rules ("never write DDL/DML, only SELECT").
  - Tool-calling via LiteLLM with two tools:
    - `run_sql(query: str)` — executed in a per-request session with `SET ROLE uam_nlq_readonly` (READ-only role), 30 s statement timeout, max 10k rows.
    - `render_layer(geojson)` — attaches result GeoJSON to the response.
  - Hard rule: any non-SELECT or use of `pg_*` catalogs → reject before exec.
  - Response shape:
    ```python
    class NlqResponse(BaseModel):
        question: str
        answer_markdown: str
        sql_executed: list[str]
        rows: list[dict]
        layer_geojson: FeatureCollection | None
        latency_ms: int
    ```
- DB role `uam_nlq_readonly`: SELECT on the curated views in schema `uam_nlq_views` (no PII, no audit_log).

**Testing**:
- Unit (mocked LLM): "list 5 oldest transformers" → produces SELECT against `asset` joined `asset_type`, results sorted by `install_date`.
- Unit: LLM tries DDL → rejected, error returned.
- Unit: query that returns >10k rows → truncated with marker in `answer_markdown`.
- Integration (real LLM, optional, gated): "which water mains over 60 years old have had two or more main breaks in the past five years" returns ≥0 rows and valid GeoJSON.

#### 6.4 — Computer vision defect detection on inspection photos

**What**: Background job analyses inspection attachments with a YOLO model and writes findings to `inspection_attachment.ai_analysis`.

**Design**:
- `services/cv_defect.py`:
  - Loads YOLO weights from `models/cv/defects-v1.pt` (pretrained on COCO + fine-tuned on a placeholder corrosion/vegetation dataset; ships with an "untrained" warning flag in `model_version`).
  - On new attachment, downloads from S3, runs inference, writes JSON: `{ "detections": [{ "class": "corrosion", "confidence": 0.84, "bbox": [...] }, ...], "model_version": "v0.1-stub" }`.
  - High-confidence "critical" defects (corrosion ≥ 0.8) auto-create corrective WOs.
- Arq job `analyse_attachment(attachment_id)` enqueued on upload finalise.

**Testing**:
- Unit (mocked model): 1 detection returned → JSON written; high-confidence → WO created.
- Integration: upload a known fixture image → analysis row appears within 10 s.

#### 6.5 — MCP server exposing read-only asset tools

**What**: Run an MCP server (stdio + SSE) that exposes asset search, asset detail, work order list, network trace, and NLQ as tools — enabling Claude/GPT clients to interact with the platform.

**Design**:
- `mcp_server/server.py` registers tools:
  - `search_assets(query: str, bbox: list[float] | None, limit: int = 25)`
  - `get_asset(asset_id: str)`
  - `list_work_orders(asset_id: str | None, status: str | None, limit: int = 50)`
  - `trace(type: Literal["upstream","downstream","isolation"], asset_id: str)`
  - `ask(question: str)`  # delegates to NLQ service
- Auth: MCP transport carries an OAuth bearer token; tools call backend HTTP endpoints with that token (no privilege escalation).

**Testing**:
- Unit: tool schema validates against MCP spec.
- Integration: Claude Desktop config snippet provided in docs; `search_assets` returns expected results when called via stdio harness in tests.

---

## Phase 7: Field & Mobile (Offline-Capable PWA + GeoPackage Sync)

### Purpose
Equip field crews and inspectors with an offline-capable workflow. After this phase a crew downloads a work zone (GeoPackage + WOs + inspection templates) before leaving connectivity, completes WOs and inspections in the field, and syncs on reconnect with conflict resolution.

### Tasks

#### 7.1 — GeoPackage export & import

**What**: Server endpoint to package a region (bbox or subnetwork) into an OGC GeoPackage including assets, locations, open WOs, and inspection templates; companion import endpoint to ingest a GeoPackage written by the field client.

**Design**:
- `integrations/geopackage_io.py`:
  - `build_geopackage(bbox, *, include_open_wos=True, max_size_mb=200) -> Path` using `fiona` / `pyogrio`.
  - `ingest_geopackage(path, *, user_id) -> IngestReport` validates and merges.
- Conflict policy: `last-write-wins by updated_at`, except for `work_order.status` transitions which use the state-machine merge (e.g., if server says `completed` and field says `in_progress`, keep `completed`).
- Endpoints:
  - `POST /field/packages` body `{ bbox, subnetwork_id?, ttl_minutes }` → returns signed download URL.
  - `POST /field/packages/{id}/sync` (multipart upload of modified GeoPackage) → returns `IngestReport`.

**Testing**:
- Integration: download GP for bbox with 50 assets → file opens in QGIS; layers present: `assets`, `work_orders`, `inspections`, `locations`.
- Integration: import GP with edit to one WO status → server reflects change, audit log records `system_field_sync`.
- Unit: conflict resolver chooses `completed` over `in_progress`.

#### 7.2 — PWA service worker and IndexedDB cache

**What**: Frontend service worker caches map tiles, asset data, templates; IndexedDB stores pending edits; status bar shows sync state.

**Design**:
- Workbox-based service worker; cache strategies:
  - `tiles/*` → CacheFirst (max 500 MB, eviction LRU)
  - `/api/v1/work-orders?assigned=me` → StaleWhileRevalidate
  - WO/inspection edits → Background Sync queue
- `lib/offline/sync.ts` queues mutations, retries with exponential backoff.

**Testing**:
- E2E (Playwright with `--offline=true`): start online, load WO list, go offline, edit WO status, go online, assert sync completed and server has change.

#### 7.3 — Inspection capture UX with GPS, photo, signature

**What**: Mobile-first inspection page: render template items, capture GPS, photos via `<input capture>`, sketch signature; submit as a single API call.

**Design**:
- Photos uploaded to MinIO via presigned URL while online; queued if offline.
- GPS captured on each inspection-response save (HTML5 geolocation).

**Testing**:
- E2E: complete a 10-item inspection on a mocked mobile viewport, submit, verify `inspection` row, responses, attachments.

---

## Phase 8: IoT Telemetry & Real-Time Condition Monitoring

### Purpose
Ingest sensor data, evaluate threshold and rate-of-change alerts, surface live asset health, and auto-create work orders when thresholds are breached. After this phase the platform is a live operational dashboard, not just a register of static assets.

### Tasks

#### 8.1 — Sensor registry and observation ingest

**What**: Implement `sensor` and `sensor_observation` (TimescaleDB hypertable) and an ingestion endpoint optimised for bulk pushes.

**Design**:
- Convert `sensor_observation` to TimescaleDB hypertable on `observed_at`, chunk interval 1 day, retention policy default 365 days (configurable per sensor).
- Endpoints:
  - `POST /sensors` admin-only register.
  - `POST /sensors/{id}/observations` batch body `[{ "observed_at": ..., "value_numeric": ..., "quality_flag": "good" }, ...]` (max 5000 per call).
  - `GET /sensors/{id}/observations?since=&until=&downsample=1m|5m|1h` uses TimescaleDB `time_bucket`.
- MQTT bridge `integrations/mqtt_bridge.py` subscribes to broker, translates to ingest endpoint.

**Testing**:
- Integration: 100k observation batch insert completes in < 5 s.
- Integration: downsampled query returns expected bucket count.

#### 8.2 — Alert evaluator and auto-WO creation

**What**: Stream processor that evaluates min/max threshold and rate-of-change rules and writes `sensor_alert` and optionally `work_order` rows.

**Design**:
- Arq job triggered after each ingest batch: `evaluate_alerts(sensor_id, latest_window)`.
- Rule types: `high_threshold`, `low_threshold`, `rate_of_change`, `anomaly` (z-score over rolling 24 h window).
- Auto-WO policy: critical alert on asset with `criticality in ('critical','high')` → corrective WO at priority 1.
- De-duplicates: don't create another WO for the same sensor while existing WO is open.

**Testing**:
- Unit: 5 readings above threshold → 1 alert (not 5), `severity='warning'`.
- Unit: critical alert on critical asset → WO created.
- Integration: ingesting a spike + waiting 2 s → unack'd alert visible via `GET /sensors/alerts?ack=false`.

#### 8.3 — Sensor dashboard frontend

**What**: Frontend page per asset showing live charts (Recharts or uPlot), alert list, ack button.

**Design**: Polls `/observations?since=now-24h&downsample=1m` every 5 s; uses Server-Sent Events for new alerts (`/sensors/alerts/stream`).

**Testing**: E2E: ingest a synthetic spike during test, observe chart updates and alert appears in UI within 10 s.

---

## Phase 9: Compliance, Capital, Climate Risk, Reporting & Integrations

### Purpose
Close the regulated-utility loop with NERC CIP, ISO 55000, AWWA compliance artefacts; climate-risk overlay analysis; canned report templates; and SCADA/ERP integration shims. After this phase a regulated utility can demonstrate audit readiness.

### Tasks

#### 9.1 — BES cyber asset & compliance obligation tracking

**What**: Implement `bes_cyber_asset` and `compliance_obligation` tables, plus a recurring review job that flags overdue obligations.

**Design**:
- Endpoints `/compliance/bes-assets`, `/compliance/obligations`.
- Seed obligations from `migrations/data/compliance_templates/nerc_cip.yaml` and `iso_55000.yaml` (clauses, frequencies, evidence checklists).
- Arq daily job `compliance_status_sweeper` updates `status` to `'overdue'` when `next_due < now`.

**Testing**:
- Integration: obligation seeded with `frequency='annual'`, `last_completed = 13 months ago` → `status='overdue'` after sweep.

#### 9.2 — Climate risk overlay

**What**: Implement `climate_risk_zone` table, scripts to ingest FEMA flood zones / USDA wildfire maps / NOAA hurricane probability layers, and an endpoint that returns assets in a risk zone.

**Design**:
- `scripts/load_climate_zones.py --source fema|usda|noaa --file path.gpkg` loads MultiPolygon features.
- Endpoint `GET /climate-risk/assets?hazard=flood&min_risk=high` → list of assets per suggestion-1 §Example Query.
- Periodic job intersects asset geom with risk zones and stores result for fast lookup.

**Testing**:
- Integration: load a small FEMA fixture (3 polygons), assert N assets intersect; map shows hatched layer.

#### 9.3 — Compliance report templates

**What**: PDF/HTML report generator for ISO 55001 evidence pack, NERC CIP-002 inventory, AWWA M5 condition assessment summary.

**Design**:
- `services/reporting.py` uses Jinja2 + WeasyPrint.
- Templates in `templates/reports/{iso55001,nerc_cip_002,awwa_m5}.html.j2`.
- Endpoint `POST /reports/generate { template_key, parameters }` returns `report_id`; status pollable; download URL when ready.

**Testing**:
- Integration: generating ISO 55001 evidence pack against seed data produces PDF, contains expected clause headings.

#### 9.4 — SCADA / ERP integration adapters (interface only)

**What**: Define abstract adapter interfaces and reference adapters for OPC-UA (`asyncua`), MQTT (`asyncio-mqtt`), SAP S/4HANA Plant Maintenance (OData), and a generic webhook outbox.

**Design**:
- `integrations/scada.py` interface:
```python
class ScadaAdapter(Protocol):
    async def read_point(self, point_id: str) -> ScadaReading: ...
    async def subscribe(self, point_ids: list[str], handler: Callable[[ScadaReading], Awaitable[None]]) -> None: ...
```
- `integrations/erp_sap.py` reference adapter posts WOs to S/4 PM via OData (`/sap/opu/odata/sap/API_MAINTNOTIFICATION/...`).
- Outbox pattern: domain events (`WorkOrderCompleted`, `AssetCondemned`) persisted to `outbox` table; relay worker delivers to configured webhook URLs with HMAC signature.

**Testing**:
- Unit (mocked OPC-UA): `read_point` returns parsed reading.
- Integration: domain event written → outbox row → webhook called within 5 s; failure retried with exponential backoff up to 24 h.

#### 9.5 — Service request public intake

**What**: Implement `service_request` table and an unauthenticated public endpoint that captures the request, geocodes the address, and triages.

**Design**:
- `POST /public/service-requests` no-auth, rate-limited via Redis (5 req/min/IP).
- Captcha via Cloudflare Turnstile (optional).
- On insert: auto-creates a corresponding WO (priority by category mapping) and assigns to default crew.

**Testing**:
- Integration: 6 requests in 60 s from same IP → 6th returns 429.
- Integration: valid request creates SR and linked WO.

---

## Phase 10: Hardening, Performance, Security, Release

### Purpose
Make the platform production-ready: security review, performance tuning, OGC and OpenAPI conformance, packaging (Helm chart), documentation, and v1.0 release.

### Tasks

#### 10.1 — OGC API Features conformance via pygeoapi

**What**: Configure `pygeoapi` with PostGIS provider to publish `/ogcapi/collections/{assets,locations,sensors,subnetworks}` and pass the OGC CITE test suite.

**Design**:
- `pygeoapi-config.yml` declares each collection with `provider: PostgreSQL`, CRS 4326 + 3857.
- Reverse-proxied at `/ogcapi/...`.

**Testing**:
- Integration: OGC API Features Core conformance class passes.
- Integration: `GET /ogcapi/collections/assets/items?bbox=...&limit=10` returns GeoJSON FeatureCollection.

#### 10.2 — Security hardening per OWASP API Top 10

**What**: Implement rate limiting, IDOR protection, field-level redaction, request-size limits; run Bandit + pip-audit + Trivy in CI.

**Design**:
- Rate limit: SlowAPI or custom Redis-token-bucket, default 600 req/min/user.
- IDOR: every `GET /<resource>/{id}` calls `assert_can_access(current_user, resource)` which enforces RBAC + organisation scoping.
- Audit-log access restricted to `auditor` role.
- CI runs `bandit -r backend/src`, `pip-audit`, `trivy image`, `npm audit --omit=dev`.

**Testing**:
- Unit: user from org A requesting asset owned by org B → 403, audit row written.
- Integration: 700 requests/minute → some 429s.
- CI: any HIGH/CRITICAL vuln fails the build.

#### 10.3 — Performance benchmarking

**What**: Load tests with Locust covering map tile fetch, asset search, work order creation, sensor ingest, trace; tune indexes and queries to meet targets.

**Design**:
- Targets:
  - p95 map tile < 200 ms
  - p95 asset search (1M assets) < 300 ms
  - p95 trace on 100k-node network < 1 s warm
  - Sensor ingest 50k observations/sec sustained
- Synthetic dataset script `scripts/load_synthetic.py --assets 1000000 --observations-per-sensor 1000000`.

**Testing**:
- CI nightly job runs Locust against staging; fails if SLOs missed.

#### 10.4 — Documentation, Helm chart, release

**What**: Author user/admin/developer docs (mkdocs-material), Helm chart, demo seed script, release process.

**Design**:
- `docs/` with sections: Getting Started, Concepts, Admin, Developer, Compliance, API Reference (generated from OpenAPI).
- `deploy/helm/utility-asset-management/` chart with values for: postgres external/internal, redis external/internal, OIDC settings, autoscaling.
- Release pipeline: tag `v1.0.0` → builds + signs Docker images, publishes Helm chart, pushes docs.

**Testing**:
- Integration: `helm install uam ./deploy/helm/utility-asset-management --set ...` against kind cluster brings up the stack; smoke tests pass.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Infrastructure        ── required by everything
    │
    ▼
Phase 2: Identity, Organisation, Spatial    ── requires Phase 1
    │
    ▼
Phase 3: Asset Register                     ── requires Phase 2
    │
    ▼
Phase 4: Work Orders & Inspections          ── requires Phase 3
    │
    ├──► Phase 5: Network Topology & Trace  ── requires Phase 3 (parallel with 4 after 3)
    │
    ├──► Phase 6: AI-Native Capabilities    ── requires Phase 4 + Phase 5 (RUL uses inspection + trace)
    │
    ├──► Phase 7: Field/Mobile/Offline      ── requires Phase 4 (can parallel with Phase 5 & 6)
    │
    ├──► Phase 8: IoT Telemetry             ── requires Phase 3 (can parallel with Phases 4–7)
    │
    ▼
Phase 9: Compliance, Capital, Climate, Integrations  ── requires Phases 4, 6, 8
    │
    ▼
Phase 10: Hardening, Perf, Security, Release ── requires all preceding
```

**Parallelism opportunities once the foundation and asset register are complete (Phases 1–3):**

- After Phase 3, a team can split: one stream takes Phase 4 (work orders & inspections), another takes Phase 5 (topology & trace), another takes Phase 8 (IoT) — they touch disjoint modules.
- Frontend work on the field PWA (Phase 7) can begin in parallel with backend Phase 4 once the WO REST contract is stable.
- AI work (Phase 6) is gated on Phases 4 and 5 because it consumes inspections, work orders, and network context.

---

## Definition of Done (per phase)

Every phase MUST satisfy all of the following before being marked complete:

1. All tasks in the phase are implemented and merged to `main`.
2. All unit tests pass (`pytest backend/tests/unit`, `pnpm -C frontend test:unit`).
3. All integration tests pass (`pytest backend/tests/integration` against testcontainers PostGIS).
4. End-to-end tests for the phase's user-visible behaviour pass (`pnpm -C frontend test:e2e`).
5. `ruff check .`, `ruff format --check .`, `mypy backend/src`, `eslint frontend/src`, `tsc --noEmit` all pass.
6. Docker image builds and `docker compose up` produces a working stack.
7. Database migrations applied cleanly via `alembic upgrade head` and reversible via `alembic downgrade -1`.
8. New API endpoints appear in `docs/api/openapi.json` (regenerated and checked in).
9. Schemathesis run against the new endpoints succeeds (no spec-violating responses, no 500s on fuzzed inputs).
10. New config options documented in `.env.example` and `docs/admin/configuration.md`.
11. New permissions seeded into the `permission` and `role_permission` tables via migration.
12. CHANGELOG.md updated with a Keep-a-Changelog entry for the phase.
13. For phases that touch user-facing UI: at least one screenshot or short Loom in `docs/screenshots/`.
14. For phases with security-sensitive surface area (Phases 1, 2, 9, 10): Bandit + npm audit + Trivy report attached, no HIGH/CRITICAL findings.
15. For phases that introduce standards claims (Phases 2, 3, 5, 10): conformance test or mapping document updated (`docs/compliance/`).
