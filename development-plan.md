# Pest Control Management — Phased Development Plan

> Project: 239-pest-control-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan delivers an AI-native, open-source operations platform for pest control businesses. It is built on Data Model Suggestion 1 (Entity-Centric Normalised Relational) — chosen for its compliance-first orientation, clean REST/GraphQL mapping, and direct alignment with EPA FIFRA / 40 CFR Part 170 record-keeping requirements.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | Best fit for the AI-native features (computer-vision pest ID, churn prediction, weather-aware re-treatment). Mature ecosystems for FastAPI, Pydantic, Celery, PyTorch, and the QuickBooks/Twilio/Stripe SDKs. Domain workload is more ML + integration plumbing than raw throughput. |
| API framework | **FastAPI** | Async, automatic OpenAPI 3.1 generation (a standards.md requirement), Pydantic v2 validation, OAuth 2.0 security scopes built in, easy to test. Maps cleanly to the one-table-per-resource pattern of Data Model 1. |
| Database | **PostgreSQL 16 + PostGIS 3.4** | Data Model 1 mandates `GEOGRAPHY(POINT, 4326)` for route optimisation and trap placement, `JSONB` for PPLS raw payloads and audit changes, `GIN` indexes on `customers.tags`, and partial indexes on certification expiry. Only Postgres satisfies all of these. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Async support pairs with FastAPI; Alembic gives reproducible migrations the schema needs as it grows across 13 categories of tables. |
| Task queue | **Celery 5 + Redis** | Required for async workloads: PPLS sync, route optimisation, SMS/email reminders, photo OCR/CV inference, weather lookups, QuickBooks sync, review-request scheduling. Redis doubles as cache and Celery broker/backend. |
| Real-time | **Server-Sent Events (SSE) via FastAPI** | Technician location updates, dispatch board pushes. Lighter than WebSockets, works through proxies, sufficient for one-way push. |
| Frontend (office app) | **Next.js 15 (App Router) + React 19 + TypeScript** | Operations console with calendar, route map, customer records, compliance reports. SSR for SEO of customer portal subroutes, file-based routing for the multi-screen back-office. |
| UI library | **shadcn/ui + Tailwind CSS v4** | Accessible primitives, drag-and-drop friendly (`@dnd-kit`), unopinionated theming — suits a polished SMB-grade product. |
| Map / routing visualisation | **MapLibre GL JS + OpenStreetMap tiles + OSRM** | Open-source map stack — avoids Google Maps lock-in. OSRM provides the routing matrix the VRP solver needs. |
| Route optimiser | **Google OR-Tools (Python) — VRP solver** | Best-in-class open-source solver for the Vehicle Routing Problem with Time Windows (VRPTW). Handles the skill/certification-aware FieldRoutes-style use case in v1.1. |
| Mobile (technician) app | **React Native + Expo (TypeScript)** | Shared TS skills with the office app, single codebase for iOS + Android, Expo handles signing/distribution. Offline-first via WatermelonDB. |
| Mobile offline store | **WatermelonDB (SQLite-backed) + custom sync protocol** | The features.md "Must-have MVP" explicitly demands offline capability, signature capture, and in-field payment. WatermelonDB is the proven RN choice with conflict-free sync semantics. |
| LLM provider abstraction | **LiteLLM** | Routes prompts to OpenAI, Anthropic, or local providers without coupling. Used for record-generation prompts and explanations. |
| Computer vision (pest ID) | **PyTorch + torchvision (ResNet-50 fine-tune) + ONNX Runtime** | Open-source, deployable to CPU containers; ONNX makes a future on-device mobile path viable. |
| LLM / RAG vector store | **pgvector** (Postgres extension) | Single-database deployment; no new infrastructure. Used for PPLS label retrieval and historical service-report search. |
| Object storage | **S3-compatible (MinIO for self-host, AWS S3 for SaaS)** | Photos, signatures, exported compliance PDFs. Boto3 SDK works against both. |
| Payments | **Stripe (cards) + Stripe ACH** | Standards.md flags PCI DSS; Stripe absorbs PCI scope via Elements / Payment Sheet. |
| SMS / email | **Twilio (SMS) + Postmark (transactional email)** | Both have first-class Python SDKs, delivery webhooks, and proven reliability. |
| Accounting integration | **QuickBooks Online API (Intuit Python SDK)** | Required by all major incumbents; OAuth 2.0 flow already covered by our auth tooling. |
| Auth | **OAuth 2.0 + OIDC via Authlib; JWT (RS256) session tokens** | Standards.md lists RFC 6749 + RFC 7519 + OpenID Connect 1.0 as required. Supports SSO for enterprise tenants and powers our own public API. |
| Containerisation | **Docker + Docker Compose (dev/self-host) + Helm chart (k8s SaaS)** | Self-hosted deployment is in scope (README §Tech Stack); compose covers local dev and small operators, Helm covers managed SaaS. |
| CI / CD | **GitHub Actions** | Lint, type-check, test, build images, run Alembic, deploy via Helm or compose. |
| Testing — Python | **pytest + pytest-asyncio + httpx + factory-boy + pytest-postgresql** | Standard stack; `pytest-postgresql` spins up real Postgres for integration tests against actual SQL/PostGIS. |
| Testing — TS | **Vitest + Playwright** | Vitest for component/unit; Playwright for E2E across web + customer portal. |
| Testing — mobile | **Jest + Detox** | Detox for on-device automation including offline sync flows. |
| Code quality | **ruff + black + mypy --strict (Python); eslint + prettier + tsc --strict (TS)** | Industry standard; minimal config. |
| Package managers | **uv (Python); pnpm (Node); Expo CLI (mobile)** | uv is dramatically faster than pip; pnpm gives content-addressable storage for the monorepo. |
| Monorepo tooling | **uv workspaces (Python) + pnpm workspaces (TS)** | Each language stays in its native tool; no Bazel/Nx complexity. |
| Observability | **OpenTelemetry → Grafana stack (Tempo + Loki + Prometheus)** | Self-host friendly; OTel ensures vendor portability for SaaS deployment. |
| Documentation | **MkDocs Material + auto-generated OpenAPI redoc** | Public dev portal + internal architecture docs from one source. |

### Project Structure

```
pest-control-management/
├── README.md
├── LICENCE
├── CONTRIBUTING.md
├── docker-compose.yml                  # Postgres+PostGIS, Redis, MinIO, OSRM, app stack for local dev
├── Makefile                            # convenience targets (up, down, migrate, seed, test)
├── .github/
│   └── workflows/
│       ├── api.yml
│       ├── web.yml
│       ├── mobile.yml
│       └── release.yml
│
├── deploy/
│   ├── helm/                           # Helm chart for k8s SaaS deployment
│   └── compose/                        # production-leaning compose overrides
│
├── docs/                               # MkDocs site
│   ├── mkdocs.yml
│   └── docs/
│       ├── architecture.md
│       ├── api/                        # auto-generated from OpenAPI
│       └── compliance/
│
├── api/                                # Python backend (FastAPI)
│   ├── pyproject.toml
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── migrations/                     # Alembic revisions
│   └── src/pcm/
│       ├── __init__.py
│       ├── main.py                     # FastAPI app factory + lifespan
│       ├── config.py                   # Pydantic Settings
│       ├── db/
│       │   ├── session.py              # async engine + session factory
│       │   ├── base.py                 # DeclarativeBase
│       │   └── rls.py                  # row-level security helpers
│       ├── models/                     # SQLAlchemy models (one file per category)
│       │   ├── tenant.py
│       │   ├── customer.py
│       │   ├── scheduling.py
│       │   ├── routing.py
│       │   ├── pesticide.py
│       │   ├── device.py
│       │   ├── technician.py
│       │   ├── fleet.py
│       │   ├── billing.py
│       │   ├── service.py
│       │   ├── communication.py
│       │   └── audit.py
│       ├── schemas/                    # Pydantic v2 request/response models
│       ├── api/                        # FastAPI routers (one per resource)
│       │   ├── auth.py
│       │   ├── customers.py
│       │   ├── jobs.py
│       │   ├── appointments.py
│       │   ├── routes.py
│       │   ├── chemicals.py
│       │   ├── devices.py
│       │   ├── invoices.py
│       │   ├── service_reports.py
│       │   ├── webhooks.py
│       │   └── exports.py
│       ├── services/                   # business logic, not HTTP-aware
│       │   ├── scheduling.py
│       │   ├── route_optimiser.py
│       │   ├── chemical_compliance.py
│       │   ├── billing.py
│       │   ├── notifications.py
│       │   └── sync.py
│       ├── integrations/
│       │   ├── epa_ppls.py
│       │   ├── quickbooks.py
│       │   ├── stripe_payments.py
│       │   ├── twilio_sms.py
│       │   ├── postmark_email.py
│       │   ├── weather.py
│       │   └── osrm.py
│       ├── ai/
│       │   ├── pest_id.py              # CV inference
│       │   ├── churn.py                # churn scoring
│       │   ├── retreatment.py          # weather + history → interval
│       │   ├── record_generator.py     # LLM EPA record drafting
│       │   └── prompts/                # versioned prompt templates
│       ├── tasks/                      # Celery tasks
│       │   ├── celery_app.py
│       │   ├── ppls_sync.py
│       │   ├── reminders.py
│       │   ├── route_jobs.py
│       │   ├── photo_processing.py
│       │   ├── qb_sync.py
│       │   └── review_requests.py
│       ├── exports/
│       │   ├── chemical_register.py    # state-format CSV/PDF generators
│       │   └── icalendar.py            # RFC 5545 VCALENDAR export
│       ├── auth/
│       │   ├── oauth.py
│       │   ├── jwt.py
│       │   └── rbac.py
│       └── utils/
│           ├── geo.py                  # PostGIS helpers
│           ├── rrule.py                # RFC 5545 RRULE expansion
│           └── pagination.py
│   └── tests/
│       ├── conftest.py
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/
│
├── web/                                # Next.js office console + customer portal
│   ├── package.json
│   ├── Dockerfile
│   ├── next.config.ts
│   └── src/
│       ├── app/
│       │   ├── (office)/
│       │   │   ├── dashboard/
│       │   │   ├── schedule/
│       │   │   ├── routes/
│       │   │   ├── customers/
│       │   │   ├── jobs/
│       │   │   ├── chemicals/
│       │   │   ├── devices/
│       │   │   ├── reports/
│       │   │   ├── invoices/
│       │   │   └── settings/
│       │   └── (portal)/
│       │       └── customer/
│       ├── components/
│       │   ├── ui/                     # shadcn primitives
│       │   ├── calendar/
│       │   ├── route-map/
│       │   ├── forms/
│       │   └── compliance/
│       ├── lib/
│       │   ├── api-client.ts           # generated from OpenAPI
│       │   ├── auth.ts
│       │   └── maplibre.ts
│       └── hooks/
│   └── tests/
│       ├── unit/
│       └── e2e/
│
├── mobile/                             # React Native technician app (Expo)
│   ├── package.json
│   ├── app.json
│   └── src/
│       ├── App.tsx
│       ├── screens/
│       │   ├── DailyRoute.tsx
│       │   ├── JobDetail.tsx
│       │   ├── ChemicalLog.tsx
│       │   ├── DeviceInspection.tsx
│       │   ├── ServiceReport.tsx
│       │   └── Payment.tsx
│       ├── db/                         # WatermelonDB schemas + sync
│       ├── components/
│       └── lib/
│   └── e2e/                            # Detox tests
│
└── shared/
    ├── openapi/                        # checked-in OpenAPI 3.1 spec
    ├── json-schema/                    # canonical compliance JSON Schemas
    └── seed-data/                      # PPLS sample, demo tenant, fixtures
```

---

## Phase 1: Foundations — Repo, Infra, Tenancy, Auth

### Purpose
Stand up the monorepo, container infrastructure, Postgres+PostGIS, Alembic migrations, the first models (tenants, users), and the authentication primitives that every later phase depends on. Nothing customer-facing ships here, but every subsequent phase can assume a multi-tenant, authenticated, observable platform exists.

### Tasks

#### 1.1 — Monorepo skeleton & tooling

**What**: Initialise the directory structure above, configure `uv` workspaces (api), `pnpm` workspaces (web, mobile, shared), root `Makefile`, root `pre-commit` hooks (ruff, black, mypy, eslint, prettier, tsc), and GitHub Actions for lint/type-check.

**Design**:
- Root `pyproject.toml` declares `[tool.uv.workspace] members = ["api"]`.
- Root `pnpm-workspace.yaml` declares `web`, `mobile`, `shared/*`.
- `api/pyproject.toml` pins Python 3.12, FastAPI 0.115+, SQLAlchemy 2.0+, Alembic, Celery, Pydantic v2.
- `.pre-commit-config.yaml` runs ruff format/check, mypy --strict, eslint --max-warnings 0, prettier --check.
- GitHub Actions matrix: `lint`, `type-check`, `test-unit` for each workspace.

**Testing**:
- CI: `pre-commit run --all-files` passes on an empty repo.
- CI: `uv run pytest -q` reports "no tests collected" but exit 0.
- CI: `pnpm -r build` succeeds on placeholder packages.

#### 1.2 — Docker Compose dev stack

**What**: A single `docker-compose up` brings up Postgres+PostGIS, Redis, MinIO, OSRM, MailHog (dev SMTP), the API, the Celery worker + beat, and the Next.js dev server.

**Design**:
- Services in `docker-compose.yml`:
  - `db`: `postgis/postgis:16-3.4`, port 5432, healthcheck `pg_isready`.
  - `redis`: `redis:7-alpine`, port 6379.
  - `minio`: `minio/minio:latest`, ports 9000/9001, console at 9001.
  - `osrm`: `osrm/osrm-backend:latest` with a small Bay Area OSM extract for dev.
  - `mailhog`: `mailhog/mailhog`, port 8025.
  - `api`: builds `api/Dockerfile`, depends on db+redis, mounts `./api` for hot reload via `uvicorn --reload`.
  - `worker`: same image, command `celery -A pcm.tasks.celery_app worker -l info -B`.
  - `web`: `node:22-alpine`, mounts `./web`, runs `pnpm dev`.
- `.env.example` enumerates every required variable; `pcm/config.py` Pydantic Settings rejects boot with missing required vars.

**Testing**:
- Manual smoke: `make up && make migrate && curl http://localhost:8000/health` returns `{"status":"ok"}`.
- CI: a `compose-up` job runs `docker compose up -d db redis`, then `pg_isready` and `redis-cli ping`.

#### 1.3 — Database session, migrations baseline, RLS scaffolding

**What**: Async SQLAlchemy engine/session, Alembic environment, and a baseline migration that enables required Postgres extensions and provides the row-level security helper.

**Design**:
- `pcm/db/session.py`:
  ```python
  engine = create_async_engine(settings.database_url, pool_size=20, pool_pre_ping=True)
  AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

  async def get_db() -> AsyncIterator[AsyncSession]:
      async with AsyncSessionLocal() as session:
          yield session
  ```
- Baseline Alembic migration `0001_baseline.py` runs:
  ```sql
  CREATE EXTENSION IF NOT EXISTS postgis;
  CREATE EXTENSION IF NOT EXISTS pgcrypto;       -- gen_random_uuid()
  CREATE EXTENSION IF NOT EXISTS pg_trgm;        -- fuzzy customer search
  CREATE EXTENSION IF NOT EXISTS vector;         -- pgvector for RAG
  CREATE EXTENSION IF NOT EXISTS btree_gist;     -- exclusion constraints
  ```
- `pcm/db/rls.py` exposes `async def set_tenant_context(session, tenant_id: UUID)` that runs `SELECT set_config('app.current_tenant', :tid, true)` so RLS policies (added per table from Phase 1.4 onward) can read it.

**Testing**:
- Unit: `set_tenant_context` issues the expected SQL with the tenant UUID.
- Integration (real Postgres via pytest-postgresql): baseline migration runs cleanly, all extensions reported in `pg_extension`.
- Integration: downgrade then upgrade returns identical schema (verified with `pg_dump --schema-only`).

#### 1.4 — Tenants and users tables + RLS policies

**What**: Implement the `tenants` and `users` tables from Data Model 1, with RLS policies enforcing `tenant_id = current_setting('app.current_tenant')::uuid`.

**Design**:
- Alembic `0002_tenants_users.py` creates the two tables exactly as in Data Model 1 §"Core Identity & Multi-Tenancy".
- RLS policy:
  ```sql
  ALTER TABLE users ENABLE ROW LEVEL SECURITY;
  CREATE POLICY users_tenant_isolation ON users
      USING (tenant_id = current_setting('app.current_tenant', true)::uuid);
  ```
- SQLAlchemy models in `pcm/models/tenant.py`:
  ```python
  class Tenant(Base):
      __tablename__ = "tenants"
      id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
      name: Mapped[str]
      slug: Mapped[str] = mapped_column(unique=True)
      subscription_plan: Mapped[str] = mapped_column(default="trial")
      timezone: Mapped[str] = mapped_column(default="America/New_York")
      settings: Mapped[dict] = mapped_column(JSONB, default=dict)
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now())
  ```
- Password hashing via `argon2-cffi`; never store plaintext.

**Testing**:
- Unit: `User.set_password` then `verify_password` round-trips for valid pw, fails for wrong.
- Integration: insert two tenants T1, T2 with one user each. Set `app.current_tenant` to T1 → `SELECT * FROM users` returns one row. Switch to T2 → returns the other. Without setting it → returns zero rows.
- Integration: attempting `INSERT INTO users (tenant_id, ...)` where `tenant_id` differs from the session's `current_tenant` is rejected by `WITH CHECK` policy.

#### 1.5 — OAuth 2.0 + JWT authentication

**What**: Implement `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/me`, and the dependency that injects the authenticated user into routes.

**Design**:
- JWT RS256 with rotating signing keys; access tokens 15 min, refresh tokens 30 days stored hashed in `auth_refresh_tokens`.
- Endpoints (FastAPI):
  - `POST /auth/register` `{tenant_slug, email, password, first_name, last_name}` → 201 `{user, access_token, refresh_token}`.
  - `POST /auth/login` `{email, password, tenant_slug}` → 200 `{user, access_token, refresh_token}`.
  - `POST /auth/refresh` `{refresh_token}` → 200 `{access_token, refresh_token}` (rotated).
  - `GET /auth/me` → 200 current `User`.
- Dependency `get_current_user` decodes JWT, loads user, calls `set_tenant_context`.
- RBAC via `pcm/auth/rbac.py`:
  ```python
  def requires(*roles: str) -> Callable: ...
  # usage: dependencies=[Depends(requires("owner", "admin"))]
  ```

**Testing**:
- Unit: JWT encode/decode happy path; expired token raises `TokenExpired`; tampered signature raises `InvalidToken`.
- Integration: register → login → call `/auth/me` succeeds; refresh rotates; old refresh token is revoked.
- Integration: `requires("admin")` on a `technician` user returns 403.
- Integration: request without token returns 401; with valid token, `current_tenant` is set in the DB session.

#### 1.6 — Audit log middleware

**What**: Implement the `audit_logs` table and FastAPI middleware that records create/update/delete actions on any registered model.

**Design**:
- Alembic migration creates `audit_logs` exactly as in Data Model 1 §"Audit Trail".
- SQLAlchemy event listeners on `before_flush` collect `(entity, action, changes)` tuples and a request-scoped buffer flushes them after the response, capturing `user_id`, `ip_address`, `user_agent` from a `contextvar`.
- Sensitive fields (`password_hash`, payment metadata) are redacted in `changes`.

**Testing**:
- Unit: redactor strips `password_hash` from a diff dict.
- Integration: creating a `Customer` (Phase 2) writes an audit row with `action='create'`, the new field values, and the authenticated `user_id`.
- Integration: updating a customer writes an audit row with only the changed fields in `changes`.

### Definition of Done
All migrations apply and roll back cleanly; `pre-commit`, `mypy --strict`, and `pytest` all green; `docker compose up` produces a working stack in under 60 s on a developer laptop; OpenAPI is published at `/docs`.

---

## Phase 2: Customers, Locations, and Service Plans

### Purpose
Deliver the CRM backbone. After this phase the platform can store customers, their service addresses with geocoded coordinates, service-plan templates, and active contracts. The customer record then anchors every later domain (jobs, chemicals, invoices, communications).

### Tasks

#### 2.1 — Customer and service location models + migrations

**What**: Tables `customers` and `service_locations` exactly per Data Model 1.

**Design**:
- Alembic migration `0003_customers_locations.py` creates both tables with all indexes (`idx_customers_tenant`, `idx_customers_name`, `idx_customers_tags` GIN, `idx_service_locations_geo` GIST).
- RLS enabled with the same `tenant_id` policy from Phase 1.
- SQLAlchemy models add `customer = relationship("Customer", back_populates="locations")` and `locations = relationship("ServiceLocation", back_populates="customer", cascade="all, delete-orphan")`.

**Testing**:
- Integration: insert customer + 3 locations; query reflects relationship.
- Integration: GIST index used (`EXPLAIN ANALYZE ST_DWithin(...)`) — assertion on plan node text.

#### 2.2 — Customer CRUD API

**What**: `/customers` resource with list, create, retrieve, update, delete, plus search-by-name (pg_trgm).

**Design**:
- Routes:
  - `GET /customers?q=&tag=&type=&cursor=&limit=` — cursor pagination; `q` triggers `last_name || ' ' || first_name || ' ' || company_name % :q ORDER BY similarity(...) DESC`.
  - `POST /customers` — Pydantic `CustomerCreate` schema; geocodes address on creation via a pluggable `Geocoder` interface (Nominatim default, optional Google).
  - `GET /customers/{id}` — returns customer with embedded `locations`.
  - `PATCH /customers/{id}` — partial update via Pydantic `CustomerUpdate`.
  - `DELETE /customers/{id}` — soft delete by default (`deleted_at IS NULL` filter); add a column in this migration.
- Pydantic schema:
  ```python
  class CustomerCreate(BaseModel):
      first_name: str | None = None
      last_name: str | None = None
      company_name: str | None = None
      email: EmailStr | None = None
      phone: str | None = None
      customer_type: Literal["residential", "commercial", "government"] = "residential"
      tags: list[str] = []
      locations: list[ServiceLocationCreate] = Field(min_length=1)

      @model_validator(mode="after")
      def name_required(self) -> Self:
          if not (self.first_name or self.last_name or self.company_name):
              raise ValueError("at least one of first_name/last_name/company_name")
          return self
  ```

**Testing**:
- Unit: Pydantic validator rejects customer with all-blank names.
- Integration (mocked Nominatim): POST returns 201 with `coordinates` set on each location.
- Integration: search "John Smith" returns the matching customer ranked first; case-insensitive.
- Integration: another tenant's user gets 404 on a customer they cannot see (RLS verified).
- E2E (Playwright stub): creating a customer through the office UI persists and re-appears in the list.

#### 2.3 — Service plans and customer contracts

**What**: `service_plans` and `customer_contracts` tables and their CRUD endpoints.

**Design**:
- Migration `0004_plans_contracts.py` creates both tables per Data Model 1.
- `POST /service-plans` and `POST /customers/{id}/contracts`.
- Business rule (in `services/contracts.py`): an active contract automatically schedules future jobs/appointments per its `service_plan.frequency` (implementation in Phase 3 — for now persist the contract and a `next_due_at` column).
- `frequency` values map to RRULE strings stored on the contract for later expansion: e.g. `quarterly` → `FREQ=MONTHLY;INTERVAL=3`.

**Testing**:
- Unit: `frequency_to_rrule("quarterly")` returns `"FREQ=MONTHLY;INTERVAL=3"`.
- Integration: creating a contract with an already-expired `end_date` returns 422.
- Integration: cancelling a contract sets `status='cancelled'` and `cancelled_at=now()`.

#### 2.4 — Customer interaction history projection

**What**: A read-only `/customers/{id}/timeline` endpoint that unions jobs, communications, payments, and audit events into a paged stream.

**Design**:
- Implemented as a `UNION ALL` SQL view created in migration `0005_customer_timeline_view.py`:
  ```sql
  CREATE VIEW customer_timeline AS
      SELECT customer_id, 'job' AS kind, id AS ref_id, created_at AS occurred_at, ... FROM jobs
      UNION ALL SELECT customer_id, 'communication', id, sent_at, ... FROM communications
      UNION ALL SELECT i.customer_id, 'payment', p.id, p.payment_date, ... FROM payments p JOIN invoices i ON i.id = p.invoice_id
      ;
  ```
- The endpoint reads the view filtered by `customer_id` and the RLS tenant context.

**Testing**:
- Integration: with seeded data, the timeline returns events sorted desc by `occurred_at`.
- Integration: pagination via `?cursor=` is stable.

### Definition of Done
CRM CRUD live; all migrations reversible; OpenAPI spec includes the new resources; >90 % branch coverage in `services/customer.py` and `api/customers.py`.

---

## Phase 3: Scheduling, Jobs, and Appointments

### Purpose
Ship the heart of the operations product: the dispatch board. After this phase, an office user can create jobs, attach appointments with arrival windows, assign technicians (single or crew), and view a calendar. Recurring appointments work via RFC 5545 RRULE.

### Tasks

#### 3.1 — Jobs, appointments, appointment_technicians

**What**: All three tables from Data Model 1 §"Scheduling & Dispatch".

**Design**:
- Migration `0006_jobs_appointments.py` creates the three tables with the listed indexes.
- Add an EXCLUDE constraint to prevent a technician being double-booked:
  ```sql
  ALTER TABLE appointment_technicians
    ADD CONSTRAINT no_overlap_per_technician
    EXCLUDE USING gist (
        technician_id WITH =,
        tstzrange((SELECT scheduled_start FROM appointments WHERE id = appointment_id),
                  (SELECT scheduled_end   FROM appointments WHERE id = appointment_id)) WITH &&
    );
  ```
  (Implemented in practice via a generated tstzrange column on `appointments` joined to this table — spelled out in the migration.)

**Testing**:
- Integration: assigning a technician to two overlapping appointments raises `IntegrityError` referencing `no_overlap_per_technician`.
- Integration: appointment status state machine — only legal transitions allowed (see 3.3).

#### 3.2 — Appointment status state machine

**What**: Enforce transitions in `services/scheduling.py`.

**Design**:
- Allowed transitions:
  ```
  scheduled → confirmed → en_route → arrived → in_progress → completed
  scheduled → cancelled
  scheduled → no_show
  any → cancelled (admin override only, audited)
  ```
- `change_appointment_status(appt_id, new_status, user)` validates and audits.

**Testing**:
- Unit: every valid transition allowed; every invalid one raises `IllegalTransition`.
- Unit: an admin can force `completed → cancelled`, a technician cannot.
- Integration: status change creates an `audit_logs` row with both old and new values.

#### 3.3 — Recurring appointments via RRULE

**What**: Generate occurrences for a parent appointment with `recurrence_rule` populated.

**Design**:
- `pcm/utils/rrule.py` wraps `python-dateutil`'s `rrule` and `rruleset`.
- `expand_recurrence(parent: Appointment, until: datetime) -> list[Appointment]` returns transient occurrence objects (not persisted) for calendar display.
- Persistent "materialise next N occurrences" job runs nightly via Celery beat to convert near-future occurrences into real `appointments` rows with `parent_appointment_id` set, enabling per-occurrence assignments and edits without losing the series.
- Exceptions list (`EXDATE`) supported by storing as a JSONB column `recurrence_exdates` on the parent (added in this migration).

**Testing**:
- Unit: `RRULE:FREQ=MONTHLY;INTERVAL=3;COUNT=4` produces exactly 4 occurrences on the right dates.
- Unit: an `EXDATE` removes the right occurrence.
- Integration: nightly materialiser creates the configured horizon of children without duplicates on re-run (idempotent).

#### 3.4 — Scheduling API

**What**: Endpoints to drive the dispatch board.

**Design**:
- `GET /appointments?from=&to=&technician_id=&status=` — list within a time range (cursor pagination).
- `POST /jobs` then `POST /jobs/{id}/appointments` to add bookings.
- `POST /appointments/{id}/assign` `{technician_ids: [...], lead_id}` updates `appointment_technicians`.
- `POST /appointments/{id}/reschedule` `{scheduled_start, scheduled_end}` — moves with EXCLUDE conflict check, recomputes route if assigned (Phase 4).
- `POST /appointments/{id}/status` body `{status, reason?}` calls the state machine.
- SSE channel `GET /events/dispatch` pushes `{type: "appointment.updated", payload: {...}}` so the dispatch board updates in real time without polling.

**Testing**:
- Integration: GET with `from`/`to` returns only appointments overlapping the range.
- Integration: rescheduling into a conflict returns 409 with `code=CONFLICT_TECH_OVERLAP`.
- Integration: SSE client receives an event within 2 s of an `appointment.updated`.
- E2E (Playwright): dragging an appointment in the calendar UI persists and re-renders without page reload.

#### 3.5 — iCalendar export

**What**: `GET /appointments.ics?token=<read-only-token>` returns an RFC 5545 VCALENDAR feed of the requesting user's appointments.

**Design**:
- `pcm/exports/icalendar.py` uses the `icalendar` library.
- VEVENT properties: `UID = appointment_id@<tenant_slug>.pestcontrol`, `DTSTART/DTEND` in UTC, `SUMMARY = "<service_type>: <customer_name>"`, `LOCATION = formatted address`, `RRULE` for recurrences.
- Read-only token issued per user from `GET /me/calendar-feeds` (scoped, revocable).

**Testing**:
- Unit: VEVENT output validated against `icalendar.Calendar.from_ical` round-trip.
- Integration: Google Calendar URL subscription accepts the feed (manual test documented in `docs/integrations/google-calendar.md`).

### Definition of Done
Office user can build a week's schedule end-to-end through the UI; recurring contracts auto-materialise upcoming appointments; technician conflicts are impossible at the DB layer; iCalendar feed validates against an RFC 5545 linter.

---

## Phase 4: Route Optimisation

### Purpose
Turn the day's appointments into optimised stop sequences per technician — the single biggest operational value driver in the segment, and where AI-native differentiation begins.

### Tasks

#### 4.1 — Routes and route_stops tables

**What**: Tables from Data Model 1 §"Route Optimisation" with their indexes.

**Design**: Straight migration `0007_routes.py`. Add a `routes.optimisation_input_hash` column (text) to short-circuit recomputation when nothing changed.

**Testing**: Integration: insert sample routes and stops, confirm cascade delete on route removes stops.

#### 4.2 — OSRM integration

**What**: A thin client over the OSRM HTTP API for route matrix and turn-by-turn directions.

**Design**:
- `pcm/integrations/osrm.py`:
  ```python
  class OSRMClient:
      async def table(self, sources: list[Coord], destinations: list[Coord]) -> Matrix: ...
      async def route(self, waypoints: list[Coord]) -> RouteResponse: ...
  ```
- Configurable base URL (default `http://osrm:5000`); retries with exponential backoff; circuit-breaker on persistent failure.

**Testing**:
- Unit: request payload assembly matches OSRM expected shape (fixture-based).
- Integration: against the local `osrm` compose service, distance matrix for 5 Bay Area points returns sensible values (within ±10 % of straight-line × 1.3).

#### 4.3 — VRPTW solver

**What**: `pcm/services/route_optimiser.py` produces an ordered stop sequence per technician minimising total drive time subject to time windows.

**Design**:
- Inputs: list of `appointments` (each with `arrival_window_start/end`, `service_location.coordinates`, `estimated_duration_minutes`), list of technicians (each with `start_location`, `end_location`, `max_daily_stops`).
- Uses Google OR-Tools `RoutingModel` with:
  - `AddDimension(time, slack, horizon, fix_start_cumul_to_zero=True)` for time windows.
  - `AddDimension(count, ..., max_daily_stops)` for stop count.
  - Service time as transit callback addend.
- Output: `OptimisationResult` Pydantic model with `routes: list[RoutePlan]` and `unassigned: list[UUID]` (for infeasible appointments).
- Async wrapper offloads solver to a thread pool with a configurable `time_limit_seconds` (default 30).

**Testing**:
- Unit (fixture): 10 stops + 2 technicians produces a feasible 2-route solution; total drive time < unoptimised baseline by ≥ 20 %.
- Unit: tight time windows that make assignment infeasible report the affected appointment IDs in `unassigned`.
- Integration (real OSRM): same 10-stop scenario with real OSM data produces a sensible route order.

#### 4.4 — Route optimisation API + Celery job

**What**: `POST /routes/optimise` body `{date, technician_ids, options}` enqueues a Celery job and returns `202 {task_id}`. `GET /routes/{task_id}` returns status and, when ready, the persisted `routes`.

**Design**:
- The Celery task hydrates inputs, calls the solver, persists `routes` + `route_stops`, then publishes an SSE event `route.optimised`.
- Idempotency: if `optimisation_input_hash` matches the existing route for that technician+date, return existing route without recompute.

**Testing**:
- Integration: POST then poll → result populated within 60 s with mock solver.
- Integration: re-submission with identical inputs is a no-op (verified via `optimisation_input_hash`).

#### 4.5 — Map visualisation in office UI

**What**: A `RouteMap` React component showing today's routes per technician colour-coded, with stop markers and a stop-sequence sidebar.

**Design**:
- `web/src/components/route-map/RouteMap.tsx` uses MapLibre GL JS.
- Sources: a FeatureCollection per technician (one LineString from `OSRM /route` + Point features for stops).
- Side panel lists stops with drag-to-reorder; reorder POSTs to `PATCH /routes/{id}/stops` `{stops: [{id, stop_order}, ...]}` which re-optimises only the affected route or accepts the manual order (toggle).

**Testing**:
- Unit (Vitest): given a FeatureCollection prop, the component renders the expected layer IDs.
- E2E (Playwright): on the routes page, two technician routes render; dragging a stop re-orders persistently.

### Definition of Done
Daily optimisation produces routes within 30 s for 50 stops × 5 technicians; manual reordering is possible; map renders in <2 s; route data is exportable to GeoJSON via `GET /routes/{id}.geojson`.

---

## Phase 5: Chemical & Pesticide Compliance (EPA / FIFRA)

### Purpose
Ship the regulatory backbone that differentiates this platform from generic FSM tools. The MVP must let technicians log chemical applications that satisfy EPA FIFRA / 40 CFR Part 170 record-keeping requirements, with EPA Registration Numbers cross-checked against the PPLS API.

### Tasks

#### 5.1 — Pesticide products reference catalogue

**What**: `pesticide_products` table populated initially from EPA PPIS bulk-download files and kept current via PPLS API on-demand fetches.

**Design**:
- Migration `0008_pesticide_products.py` creates the table per Data Model 1 with the three indexes.
- `pcm/integrations/epa_ppls.py`:
  ```python
  class PPLSClient:
      BASE = "https://ordspub.epa.gov/ords/pesticides/apex/api/ppls/v1"
      async def get_product(self, epa_reg_number: str) -> PPLSProduct: ...
      async def search(self, q: str) -> list[PPLSProductSummary]: ...
  ```
- A `seed_pesticides.py` script ingests a checked-in `shared/seed-data/ppis_sample.csv` (≈ 1000 most common US products) on first boot so demo deployments work without a PPLS round-trip.
- Celery beat task `refresh_pesticide_products` runs weekly: for each product where `last_synced_at < now() - 30 days`, fetch fresh PPLS data and update.

**Testing**:
- Unit: PPLS JSON response parser handles the documented payload shape, including products with no `active_ingredients`.
- Integration (mocked HTTP): fetch a sample product → row created with `epa_registration_number`, `restricted_use`, `active_ingredients`, `re_entry_interval_hours`.
- Integration: re-fetch updates `last_synced_at` and only changed fields (verified via audit log).
- Fixture: seed script imports the 1000-row CSV in <30 s with all required fields non-null.

#### 5.2 — Chemical inventory

**What**: `chemical_inventory` table and CRUD endpoints for office staff to record purchases, lots, and on-hand quantities.

**Design**:
- Migration `0009_chemical_inventory.py` per Data Model 1.
- `POST /inventory` `{pesticide_product_id, lot_number, quantity, unit_of_measure, expiration_date, purchase_cost_cents}`.
- `services/inventory.py` exposes `deduct(product_id, quantity, lot=None)` using FIFO across lots (oldest non-expired first).
- Low-stock alerts: a daily Celery task scans for `quantity_on_hand < reorder_threshold` (stored in `tenants.settings`) and emits a notification.

**Testing**:
- Unit: FIFO selection prefers earliest `purchase_date` among non-expired lots; falls back to next when first runs out.
- Integration: applying 5 oz of a product with two 3-oz lots empties the first and deducts 2 from the second.
- Integration: expired lots are skipped and surfaced in `GET /inventory/expired`.

#### 5.3 — Chemical applications (the EPA record)

**What**: The `chemical_applications` table and the application-recording flow used both from the technician app and the office.

**Design**:
- Migration `0010_chemical_applications.py` per Data Model 1, with `idx_chem_apps_date` and `idx_chem_apps_technician`.
- `POST /jobs/{job_id}/chemical-applications` body validated by:
  ```python
  class ChemicalApplicationCreate(BaseModel):
      pesticide_product_id: UUID
      application_date: date
      application_time: time
      target_pests: list[str] = Field(min_length=1)
      application_site: str
      application_method: Literal["spray", "bait", "dust", "fog", "fumigation", "granular", "trap"]
      quantity_applied: Decimal = Field(gt=0)
      unit_of_measure: Literal["oz", "fl_oz", "lb", "gal", "ml", "L", "kg", "g"]
      dilution_rate: str | None = None
      concentration_pct: Decimal | None = None
      wind_speed_mph: Decimal | None = None
      temperature_f: Decimal | None = None
      notes: str | None = None
  ```
- On create, the service:
  1. Loads the linked `pesticide_product`; copies `epa_registration_number`, `re_entry_interval_hours`, `restricted_use` onto the application row (frozen snapshot for audit even if the catalogue later changes).
  2. Verifies the technician holds a non-expired certification covering the product's category (raises 403 with `code=TECH_NOT_CERTIFIED` if not).
  3. Deducts inventory via `services/inventory.deduct`.
  4. If `restricted_use_product`, requires `applicator_licence_number` be populated; rejects otherwise.
  5. Schedules a customer notification with the REI in plain English (Phase 7).

**Testing**:
- Unit: validation fails when `quantity_applied <= 0`.
- Unit: snapshot copy of EPA fields occurs even when catalogue values change mid-test.
- Integration: applying a restricted-use product without a licence returns 403.
- Integration: application without sufficient inventory returns 422 with `code=INSUFFICIENT_INVENTORY`.
- Integration: successful application increments `audit_logs` and produces a `chemical_applications` row that passes the §170.311 field requirements (verified by a property test that all required fields are non-null).

#### 5.4 — Vehicle chemical loads + DOT placarding

**What**: `vehicles` and `vehicle_chemical_loads` tables and the DOT hazmat placard flag.

**Design**:
- Migration `0011_vehicles.py` per Data Model 1.
- `POST /vehicles/{id}/loads` to record chemicals loaded on a vehicle for a day; `services/dot.py` derives the required placard class from the union of products' `dot_hazmat_class`.
- Daily Celery task warns owners when a vehicle without `dot_hazmat_placard=true` carries a restricted-use product.

**Testing**:
- Unit: placard derivation given a list of products returns the correct hazmat class set.
- Integration: warning is emitted when a non-placarded vehicle has a restricted-use load.

#### 5.5 — State-format compliance reports

**What**: `GET /reports/chemical-register?from=&to=&state=` produces a PDF (default) or CSV in the format required by the requested state's pesticide enforcement agency.

**Design**:
- `pcm/exports/chemical_register.py` defines a `RegisterFormat` interface with concrete implementations: `CaliforniaPUR`, `FloridaACR`, `TexasSPCS`, `NPMAStandard`, etc. (10 states for v1, extensible).
- PDFs rendered via WeasyPrint from Jinja templates that present the rows in the agency-specified column order.
- Outputs uploaded to S3/MinIO; endpoint returns a signed URL.

**Testing**:
- Unit per format: given the same fixture of 5 applications, the output column order and headers match the state's published template.
- Integration: report generation for 1000 applications completes in <5 s.

### Definition of Done
A technician can log a chemical application from the mobile app offline; once synced the record passes EPA field-completeness checks; the office can export a CA PUR-format register for any month; restricted-use safeguards block uncertified or unlicensed applications.

---

## Phase 6: Mobile Technician App (Offline-First)

### Purpose
Deliver the field experience: a React Native app where technicians see their route, complete jobs, log chemicals, inspect traps, capture signatures, and collect payment — all functioning in zero-connectivity environments.

### Tasks

#### 6.1 — Mobile app scaffolding and auth

**What**: Expo TypeScript app with OAuth 2.0 login, biometric unlock, and secure token storage (Expo SecureStore).

**Design**:
- Login screen calls `/auth/login`; access token stored in SecureStore; refresh handled in an Axios interceptor.
- Biometric prompt on app open re-unlocks the cached refresh token (FaceID/TouchID/Android Biometric).

**Testing**:
- Unit (Jest): refresh interceptor retries the original request with a fresh token on 401.
- Detox: login → biometric → daily route screen path works on iOS + Android simulators.

#### 6.2 — WatermelonDB schema + sync protocol

**What**: A local SQLite schema mirroring the subset of server tables needed in the field (`jobs`, `appointments`, `service_locations`, `customers`, `chemical_inventory`, `chemical_applications`, `device_inspections`, `service_reports`, `service_photos`, `payments`).

**Design**:
- Sync endpoints on the server:
  - `POST /sync/pull` body `{last_pulled_at}` → `{changes: {<table>: {created: [], updated: [], deleted: []}}, timestamp}` filtered to the technician's assigned data only.
  - `POST /sync/push` body `{changes}` → applies upserts in a single transaction, returning conflict details if any (last-write-wins with audit, except for `chemical_applications` which are append-only and immutable once synced).
- Client uses `synchronize()` from `@nozbe/watermelondb/sync`.
- File assets (photos, signatures) sync separately: queued for background upload to S3 via pre-signed URLs (`POST /uploads/presign`).

**Testing**:
- Unit (Jest): conflict-resolution rule for `chemical_applications` rejects modifications to already-synced records.
- Integration (mock server): a 100-record pull populates the DB in <1 s.
- Detox: airplane-mode → complete a job → re-enable connectivity → record appears on the server within 5 s.

#### 6.3 — Daily route screen and job detail

**What**: The home screen lists today's stops in optimised order; tapping opens the full job detail with customer, location, chemical history at this property, and previous service report.

**Design**:
- Route map snippet at top (MapLibre RN), stop list below grouped by appointment status.
- Pull-to-refresh triggers a sync pull.

**Testing**:
- Detox: stops render in `route_stops.stop_order`; tapping a stop opens detail.

#### 6.4 — Chemical application entry (offline)

**What**: The in-field form for chemical application aligned with the `ChemicalApplicationCreate` schema.

**Design**:
- Product picker shows the technician's vehicle's loaded products first, then the wider catalogue (offline-searchable).
- Quantity stepper enforces unit-of-measure rules per product.
- Validation runs client-side using a shared JSON Schema (`shared/json-schema/chemical-application.json`) generated from the Pydantic model and bundled into the app at build time.
- Submission writes to WatermelonDB synchronously; pushes on next sync.

**Testing**:
- Unit (Jest): client validator catches the same errors as the server Pydantic model (parity test using shared JSON Schema).
- Detox: airplane-mode → fill form → save → record visible in local list with `pending_sync` flag.

#### 6.5 — Signature capture and payment collection

**What**: Customer signature on completion; in-field payment via Stripe Terminal SDK.

**Design**:
- `react-native-signature-canvas` for signatures; PNG uploaded on next sync.
- Stripe Terminal SDK for tap/insert; payment intent created via `POST /payments/intents` which proxies to Stripe — the connection token is fetched from `/payments/terminal/connection-token`.
- Offline payments not supported; clearly indicated when no connection.

**Testing**:
- Detox: signature drawn, file present in WatermelonDB, syncs to S3.
- Integration (mocked Stripe): payment intent → confirm → invoice marked paid.

### Definition of Done
A full offline workday — login, view route, complete 5 jobs with chemical logs, signatures, and one online payment — works without crashes; sync resolves cleanly on reconnection; Detox flow runs in CI on both iOS and Android simulators.

---

## Phase 7: Billing, Invoicing, Payments, and QuickBooks

### Purpose
Close the operations loop: a completed appointment produces an invoice; the customer can pay online or in-field; financials sync to QuickBooks. Required for any SMB to actually operate the system.

### Tasks

#### 7.1 — Invoices, line items, payments tables

**What**: Tables per Data Model 1 §"Billing & Payments".

**Design**: Migration `0012_billing.py` with the listed indexes.

**Testing**: Integration: an invoice with multiple line items totals correctly; deleting an invoice cascades to its line items.

#### 7.2 — Invoice generation on job completion

**What**: When a job transitions to `completed`, `services/billing.generate_invoice(job)` produces an invoice draft.

**Design**:
- Pricing rules: use `customer_contracts.price_cents` if linked; else `service_plans.base_price_cents`; else `jobs.price_cents`.
- Tax: per-tenant default rate from `tenants.settings.default_tax_rate_pct`, overridable per location for jurisdictions with local sales tax.
- Line items: base service + each chemical application (when tenant setting `bill_chemicals_separately=true`) + any device replacements from `device_inspections`.

**Testing**:
- Unit: pricing precedence respects contract → plan → job override.
- Integration: completing a job creates a draft invoice with the expected lines and totals.

#### 7.3 — Stripe payment integration

**What**: Cards via Stripe Checkout for the customer portal, Terminal SDK for in-field.

**Design**:
- `POST /invoices/{id}/checkout-session` → `{url}` for portal redirect.
- Stripe webhook handler `POST /webhooks/stripe` (signature-verified using `STRIPE_WEBHOOK_SECRET`) handles `payment_intent.succeeded`, `charge.refunded`, etc., updating `payments` and `invoices.status` atomically.

**Testing**:
- Integration: signed webhook → invoice paid; unsigned → 401 ignored.
- Integration: refund webhook flips status to `refunded` and records a negative payment.

#### 7.4 — QuickBooks Online sync

**What**: Bidirectional invoice + customer + payment sync.

**Design**:
- OAuth 2.0 connect at `/integrations/quickbooks/connect`; tokens stored encrypted (Fernet) in `integration_credentials` table (new in this migration).
- `pcm/integrations/quickbooks.py` wraps the Intuit Python SDK.
- Outbound: on invoice `sent` status change, push to QBO via a Celery task; on payment, push payment.
- Inbound: nightly task pulls QBO customers/payments and reconciles by `quickbooks_id`.
- Idempotency via the `quickbooks_id` column already in Data Model 1.

**Testing**:
- Integration (mocked Intuit): sending an invoice creates a QBO Invoice; `quickbooks_id` populated.
- Integration: replaying the same task is a no-op (idempotent).
- Integration: connection failure surfaces in the admin dashboard with a retry button.

#### 7.5 — Customer portal payment

**What**: A Next.js `/customer` route where a customer logs in (magic-link via Postmark) and pays open invoices.

**Design**:
- Magic-link endpoint `POST /auth/portal/request` sends a signed JWT to the customer's email; clicking it logs them into a restricted scope (`customer_portal:read`, `customer_portal:pay`).
- Portal screens: invoices, service history, upcoming appointments, document downloads (signed service reports, exported invoices).

**Testing**:
- E2E (Playwright): request magic link → click email link (intercepted from MailHog) → portal loads → pay invoice.

### Definition of Done
End-to-end: complete a job → invoice draft → send → customer pays via portal OR technician collects in-field → QuickBooks reflects the invoice and payment.

---

## Phase 8: Traps, Devices, and Inspections (IPM Backbone)

### Purpose
Cover the device-tracking capability that GorillaDesk uses as a moat, and lay the asset-management foundation for GreenPro IPM certification and (Phase 11) IoT smart-trap integration.

### Tasks

#### 8.1 — Device types, devices, device inspections

**What**: Tables per Data Model 1 §"Trap & Device Management".

**Design**: Migration `0013_devices.py` with all indexes including the partial `next_check_date` index on active devices.

**Testing**: Integration: querying `WHERE next_check_date <= today AND status='active'` uses the partial index.

#### 8.2 — Device placement workflow (barcode + GPS)

**What**: Office assigns device types to a service location; technician scans a barcode on first placement and the device's `placement_coordinates` are captured from the phone GPS.

**Design**:
- Mobile screen `DevicePlacement.tsx` opens the camera, decodes EAN-13 / Code-128 via `expo-barcode-scanner`, prompts for placement description, records GPS, creates a `device` row offline.
- A device-type catalogue is seeded with common bait stations and traps.

**Testing**:
- Detox: barcode scan path creates a device with the scanned code and current GPS.

#### 8.3 — Inspection workflow

**What**: Quick inspection form: activity level, action taken, optional photos.

**Design**:
- `POST /devices/{id}/inspections` body `{inspection_date, activity_level, findings, action_taken, photo_urls}`.
- On submit, the device's `last_checked_date` and `next_check_date` (calculated from the device type's `default_check_interval_days`) are updated.
- "Bait replaced" or "device replaced" actions auto-deduct from inventory.

**Testing**:
- Unit: `next_check_date = inspection_date + interval` calculation is correct across DST boundaries.
- Integration: an inspection that replaces bait deducts the configured bait product quantity.

#### 8.4 — Activity dashboard

**What**: `GET /devices/activity?location_id=&from=&to=` and a corresponding office UI heat-map.

**Design**: SQL aggregation over `device_inspections` grouped by week and activity level; map rendered with MapLibre heat-map layer.

**Testing**: Integration: aggregation matches a known fixture's expected counts.

### Definition of Done
Devices placeable and inspectable from the mobile app; office sees overdue checks and activity heat-map; an export-ready device-history report is downloadable per location.

---

## Phase 9: Notifications & Communications (SMS / Email / Reviews)

### Purpose
Automate the customer touchpoints that drive retention and reviews: appointment reminders, en-route notifications, post-service summaries, and review requests.

### Tasks

#### 9.1 — Communications table + templates

**What**: `communications` table per Data Model 1 plus a `notification_templates` table (new) for tenant-customisable messages.

**Design**:
- Migration `0014_communications.py`.
- `notification_templates` columns: `id, tenant_id, key, channel, subject, body, locale, is_active`.
- Templates use Jinja2 with a whitelisted variable set (`{{customer.first_name}}`, `{{appointment.scheduled_start}}`, `{{technician.first_name}}`, `{{rei_hours}}`, etc.).

**Testing**: Unit: template renderer rejects undefined variables in strict mode.

#### 9.2 — Twilio + Postmark integration

**What**: Send SMS and email through configured providers with delivery webhooks.

**Design**:
- `pcm/integrations/twilio_sms.py`, `pcm/integrations/postmark_email.py`.
- `POST /webhooks/twilio` updates `communications.delivered_at` / failure reason.
- All sends are queued via Celery (`tasks/notifications.py`) with retries on transient failure.

**Testing**:
- Integration (mocked): sending an SMS records `communications` with `sent_at`; webhook callback updates `delivered_at`.

#### 9.3 — Appointment lifecycle notifications

**What**: Pre-defined templates auto-fire on key events.

**Design**:
- Events:
  - 24h before: SMS reminder.
  - 1h before: SMS "your technician is on the way today".
  - Status `en_route`: SMS "<Tech> is on the way, ETA <time>".
  - Status `completed`: email service summary (PDF attached).
  - 1d post-completion: SMS REI reminder if applicable.
- Per-tenant toggles to disable any individual notification.

**Testing**:
- Integration: scheduled tasks fire at the right time (clock-driven test with `freezegun`).
- Integration: tenant disabling the 24h reminder suppresses send.

#### 9.4 — Review request automation

**What**: `review_requests` table per Data Model 1 + automation.

**Design**:
- Three days after a `completed` job (configurable) and only if the customer has not been asked in the past 90 days, send a review request via SMS or email per tenant preference, with a single-link to the configured Google or Yelp review URL.
- `GET /r/{token}` records `clicked_at` and 302-redirects to the review URL.

**Testing**:
- Integration: throttling rule blocks a second request within 90 days.

### Definition of Done
A new customer through a complete service cycle receives the right messages at the right times in their preferred channel; office can edit templates; deliverability metrics visible on a dashboard.

---

## Phase 10: Public API, Webhooks, and Developer Portal

### Purpose
Expose a first-class, OAuth 2.0-secured public API so the platform can compete with Jobber's developer-friendly posture and become the integration substrate other open-source projects can build on.

### Tasks

#### 10.1 — OAuth 2.0 application registration

**What**: `oauth_apps` and `oauth_access_grants` tables; UI for tenants to create apps and rotate secrets.

**Design**:
- Tables based on standard OAuth 2.0 patterns (no exact equivalent in Data Model 1 — added here):
  ```sql
  CREATE TABLE oauth_apps (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
      name VARCHAR(255) NOT NULL,
      client_id VARCHAR(64) NOT NULL UNIQUE,
      client_secret_hash VARCHAR(255) NOT NULL,
      redirect_uris TEXT[] NOT NULL,
      scopes TEXT[] NOT NULL,
      created_at TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  ```
- Authorisation code + PKCE supported per RFC 7636.
- Scopes: `customers:read`, `customers:write`, `jobs:read`, `jobs:write`, `chemicals:read`, `chemicals:write`, `webhooks:manage`.

**Testing**:
- Integration: full authorisation_code+PKCE flow yields an access token bound to the requested scopes.
- Integration: a token issued for `customers:read` is rejected by a `customers:write` endpoint.

#### 10.2 — OpenAPI 3.1 spec polishing + dev portal

**What**: All endpoints carry tags, summaries, examples, and error schemas. The dev portal at `/developers` serves Redoc.

**Design**:
- `pcm/api/_common.py` defines `ErrorResponse` Pydantic schema; every endpoint registers `responses={400: {...}, 401: {...}, 403: {...}, 404: {...}, 409: {...}, 422: {...}}` consistently.
- Spec checked into `shared/openapi/openapi.json` and validated in CI by `redocly lint`.

**Testing**:
- CI: `redocly lint shared/openapi/openapi.json` exits 0.
- CI: a generated TypeScript client (`openapi-typescript-codegen`) builds against the spec — guarantees the spec is consumable.

#### 10.3 — Webhooks (outbound)

**What**: Tenants register HTTPS webhook endpoints that receive signed events for `customer.created`, `job.completed`, `chemical_application.created`, `invoice.paid`, etc.

**Design**:
- `webhook_endpoints` table: `id, tenant_id, url, secret, events[], is_active, last_status, last_delivered_at`.
- Celery task delivers events with HMAC-SHA256 signature header `X-PCM-Signature: t=<ts>,v1=<hex>`.
- Exponential retry with jitter on non-2xx; dead-letter after 10 attempts.

**Testing**:
- Integration: emitting an event delivers to a local test webhook receiver with a valid signature.
- Integration: receiver returning 500 triggers retries; receiver returning 410 stops retries and disables the endpoint.

#### 10.4 — Rate limiting + abuse protection

**What**: Per-token sliding-window rate limits.

**Design**: Redis-backed (`limits` library) — default 10 req/s, 1000 req/min, 50000 req/day per access token; overrideable per app.

**Testing**: Integration: 11th request inside one second returns 429 with `Retry-After`.

### Definition of Done
Independent OSS contributors can `pip install pcm-client` (generated from OpenAPI) and call the API after registering an app; webhooks deliver and retry; rate limits hold.

---

## Phase 11: AI-Native Features

### Purpose
Ship the differentiators that no incumbent has: photo-based pest ID, weather-aware re-treatment, churn prediction, and LLM-assisted compliance record generation. This is what justifies the project's existence beyond "another open-source FSM".

### Tasks

#### 11.1 — Computer-vision pest identification

**What**: Technicians snap a photo of a pest; the system returns top-3 likely species with confidence and recommends a treatment protocol.

**Design**:
- Model: ResNet-50 fine-tuned on a curated dataset of ~50 common US pests (cockroach species, ant species, termites, rodents, bed bugs, etc.) — assembled from public datasets (iNaturalist research-grade) and labelled in-house.
- Training pipeline (`api/src/pcm/ai/training/`) produces an ONNX model checkpoint.
- Inference: `POST /ai/pest-id` accepts an image (`multipart/form-data` or S3 key), runs CPU ONNX Runtime, returns:
  ```json
  {
    "predictions": [
      {"species": "Blattella germanica", "common_name": "German cockroach", "confidence": 0.92},
      {"species": "Periplaneta americana", "common_name": "American cockroach", "confidence": 0.05},
      {"species": "Supella longipalpa", "common_name": "Brown-banded cockroach", "confidence": 0.02}
    ],
    "recommended_protocol_id": "uuid-of-protocol"
  }
  ```
- Predictions written to `service_photos.ai_species_id` / `ai_confidence`.
- Recommended protocols come from a tenant-editable `treatment_protocols` table (added here) seeded with NPMA best-practice templates.

**Testing**:
- Unit: ONNX inference on a fixture image returns a stable top-1 species.
- Integration: uploading via API returns predictions; the photo record is updated.
- Bench: latency p95 < 400 ms on a 4 vCPU container.

#### 11.2 — Weather-aware re-treatment recommendations

**What**: Predict the optimal re-treatment interval per location given history, pest type, and forecast.

**Design**:
- `pcm/integrations/weather.py` wraps Open-Meteo (free, no API key) — daily temp/precipitation/humidity forecasts.
- `services/retreatment.py.recommend(location_id, pest)` features: prior intervals where pest re-appeared, current/forecast temperature trend, season, treatment history at this address.
- v1 model: gradient-boosted regressor (LightGBM) trained on synthesised + early-customer data.
- Output: `{recommended_next_visit: date, confidence: float, rationale: str}` where `rationale` is generated by a small LLM call summarising the feature contributions.

**Testing**:
- Unit: features assembled correctly from a fixture customer history.
- Integration: recommendation called with insufficient history (<3 visits) falls back to the service plan's default frequency with `confidence < 0.4`.

#### 11.3 — LLM-assisted EPA record generation

**What**: When a technician completes a job in the field, an LLM drafts the human-readable narrative parts of the service report from the structured event log.

**Design**:
- Trigger: appointment status → `completed`.
- Inputs gathered: chemical applications, device inspections, pests observed, conditions noted (short voice-to-text or quick checklist on mobile).
- Prompt template `ai/prompts/service_report.j2`:
  ```
  System: You are a pest control compliance writer. Produce a concise, professional service report
  summary from the structured data below. Do not invent facts. Output JSON matching the schema.

  User: {{ structured_data | tojson }}
  ```
- Output: `{treatment_summary, conditions_observed, recommendations, follow_up_required, follow_up_date}` validated against the `ServiceReport` Pydantic model.
- All prompts versioned; outputs persisted with `model`, `prompt_version`, `latency_ms`, `cost_cents`.
- Cost tracking aggregated daily in `ai_usage_log` (new table) so tenants see what AI is costing them.

**Testing**:
- Unit (mocked LLM): prompt assembly includes the right structured fields; output validated against schema.
- Integration: a complete fixture job yields a parseable, schema-conformant report.

#### 11.4 — Churn prediction

**What**: Nightly batch job scores active customers' risk of cancellation.

**Design**:
- Features: months since last service, count of complaints in last 90 days, payment-on-time ratio, recent re-treatment frequency vs plan, NPS responses (if collected).
- Model: logistic regression (interpretable for v1; sklearn).
- Output: `customer_churn_scores` table `{customer_id, score, top_factors: jsonb, computed_at}`.
- Surfaced in office UI as a retention queue with suggested outreach actions.

**Testing**:
- Unit: features extracted correctly from a fixture customer.
- Integration: nightly task processes 10k customers in <60 s.

### Definition of Done
At least one model per task is callable and returning predictions on real data; AI-generated outputs are clearly marked in the UI; all AI usage is logged and metered per tenant.

---

## Phase 12: Hardening, Compliance & Launch

### Purpose
Take the platform from "working" to "deployable in production": security, observability, performance, and the docs / packaging needed for both self-host and SaaS launch.

### Tasks

#### 12.1 — OWASP API Top 10 audit

**What**: Apply the OWASP API Security Top 10 checklist to every endpoint.

**Design**:
- Document mapping in `docs/security/owasp-api.md`: which control addresses which risk (BOLA via RLS + per-resource ownership checks; broken auth via JWT rotation; etc.).
- Add a CI security job: `bandit`, `pip-audit`, `npm audit`, `trivy image scan`, `semgrep --config p/owasp-top-10`.

**Testing**:
- CI: security job blocks merges on `HIGH`+ findings.
- Manual: BOLA test — user from tenant A cannot read tenant B's customer by guessing the UUID (RLS prevents it).

#### 12.2 — Observability

**What**: OpenTelemetry traces, structured logs, RED + USE metrics.

**Design**:
- `opentelemetry-instrumentation-fastapi` + `-sqlalchemy` + `-celery` enabled in `pcm/main.py`.
- Logs JSON-formatted (`structlog`) with `tenant_id`, `request_id`, `user_id` always present.
- `/metrics` Prometheus endpoint; Grafana dashboards committed in `deploy/grafana/`.

**Testing**: Manual: load test (Locust, 100 RPS) — traces visible in Tempo; latency histograms populated.

#### 12.3 — Backup & disaster recovery

**What**: Documented and tested DB + object-storage backup procedures.

**Design**:
- `pg_dump` daily to S3 with 30-day retention; PITR via WAL archiving.
- `restic` for MinIO bucket snapshots.
- Documented restore drill in `docs/operations/disaster-recovery.md`; CI runs the restore drill weekly on synthetic data.

**Testing**: Drill: restore from yesterday's backup to a fresh DB, run app smoke tests, exit clean.

#### 12.4 — Self-host installer & SaaS Helm chart

**What**: Single-command install for both modes.

**Design**:
- Self-host: `curl -sSL get.pcm.example.com | bash` generates a `.env`, pulls images, runs compose.
- SaaS: `helm install pcm ./deploy/helm/pcm --values prod.yaml` with sub-charts for Postgres (CloudNativePG operator), Redis (Bitnami), and the app.

**Testing**:
- CI: install script lints (`shellcheck`).
- CI: `helm template` + `kubeconform` validates manifests.

#### 12.5 — Documentation & launch site

**What**: MkDocs site covering architecture, deployment, API, compliance, AI features, and a 10-minute getting-started.

**Design**:
- `docs/` builds with MkDocs Material; deployed to GitHub Pages on `main` push.
- API reference auto-generated from `shared/openapi/openapi.json`.
- "Compliance" section maps each FIFRA / 40 CFR 170 / WPS requirement to the corresponding feature.

**Testing**: CI: `mkdocs build --strict` exits 0; broken links fail the build via `mkdocs-htmlproofer-plugin`.

### Definition of Done
First external operator can install in <30 minutes; security scan clean; backup/restore drill passes; v1.0 tagged and announced.

---

## Phase Summary & Dependencies

```
Phase 1: Foundations (repo, infra, tenancy, auth, audit)
    │
Phase 2: Customers, Locations, Service Plans
    │
Phase 3: Scheduling, Jobs, Appointments
    │
    ├─→ Phase 4: Route Optimisation
    │       (requires Phases 2-3)
    │
    ├─→ Phase 5: Chemical & Pesticide Compliance
    │       (requires Phase 3; independent of Phase 4)
    │
    └─→ Phase 8: Traps, Devices, Inspections
            (requires Phases 2-3; independent of 4-5)

Phase 6: Mobile Technician App
    (requires Phases 3, 4, 5, 8 — the workflows it covers must exist server-side)

Phase 7: Billing, QuickBooks
    (requires Phase 3; can be parallelised with Phase 5 and Phase 8)

Phase 9: Notifications & Communications
    (requires Phases 3, 7)

Phase 10: Public API & Webhooks
    (requires all of Phases 2-9 for full surface; can start at the end of Phase 5)

Phase 11: AI-Native Features
    (requires Phases 3, 5, 6 for data; itself independent of Phases 7-10)

Phase 12: Hardening, Compliance & Launch
    (requires all prior phases)
```

**Parallelism opportunities:**
- After Phase 3 ships, Phases 4, 5, and 8 can be developed by separate contributors concurrently.
- Phase 7 can be developed in parallel with Phase 5 once Phase 3 is complete.
- Phase 6 (mobile) integrates the server work from 4/5/7/8 — it should start once those server APIs stabilise.
- Phase 11 (AI) can have model-training work begin during earlier phases since it only depends on data shape, not running services.

---

## Definition of Done (per phase)

Each phase is complete only when **all** of the following hold:

1. **Functionality** — every task listed in the phase is implemented and demonstrable end-to-end against the dev compose stack.
2. **Migrations** — Alembic upgrade and downgrade succeed on a fresh database; the schema after upgrade matches the model definitions (verified by `alembic check`).
3. **Tests** — every test listed in each task's Testing section is implemented and passing in CI; overall branch coverage on phase-touched packages ≥ 85 %.
4. **Lint & types** — `ruff`, `black --check`, `mypy --strict`, `eslint --max-warnings 0`, `tsc --noEmit`, `prettier --check` all pass.
5. **Security** — `bandit`, `pip-audit`, `npm audit`, `semgrep`, and `trivy image scan` produce no `HIGH` or `CRITICAL` findings.
6. **OpenAPI** — all new endpoints appear in `shared/openapi/openapi.json` with summaries, tags, request/response examples, and the standard error responses; `redocly lint` exits 0; generated TypeScript client builds.
7. **Docker** — `docker compose up --build` succeeds on a clean machine; all health checks report healthy within 60 s.
8. **Compliance trace** — any phase introducing regulated functionality (5, 6, 8, 9) adds an entry to `docs/compliance/standards-map.md` showing which FIFRA / 40 CFR Part 170 / WPS / NPMA / ISO requirement each new feature satisfies.
9. **Documentation** — phase user-facing changes documented under `docs/` and linked from the navigation; release notes drafted in `CHANGELOG.md`.
10. **Demo seed** — `make seed` produces a working demo tenant exercising the new functionality so reviewers can click through it without bespoke data.
