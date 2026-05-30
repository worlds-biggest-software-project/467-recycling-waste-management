# Recycling & Waste Management — Phased Development Plan

> Project: 467-recycling-waste-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The data model is grounded in **Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL/PostGIS)** for the operational core, with **TimescaleDB hypertables from Suggestion 4** introduced for high-frequency telemetry in Phase 8. The hybrid model is chosen because it gives ACID integrity and referential constraints for compliance/billing data while absorbing jurisdiction-specific and integration-variable fields in JSONB without constant migrations.

---

## Product Synthesis

**What it does.** An open-source, self-hostable platform unifying collection routing, material/weighbridge tracking, jurisdiction-configurable regulatory compliance, customer/contract management, and billing for waste haulers, recycling processors (MRFs), and municipal authorities — with AI-native route optimisation, contamination image classification, and LLM-assisted compliance document generation.

**Who uses it.** Dispatchers (route planning, live edits), drivers (offline mobile execution), supervisors (fleet visibility), billing clerks (invoicing/exceptions), compliance officers (manifests, transfer notes, diversion reports), weighbridge operators, and customers (self-service portal). Multi-tenant by `organization`.

**Key differentiators.** (1) Only open-source platform in a market of proprietary SaaS; (2) pluggable, community-contributable compliance rule engine spanning US RCRA/e-Manifest, UK Digital Waste Tracking, EU CSRD/ESRS E5; (3) ML contamination detection; (4) LLM/MCP-native compliance and ops tooling; (5) affordable for 1–10 truck operators.

**Deployment model.** Self-hosted or cloud; Docker Compose for single-node, Helm-ready for clusters. Backend REST API (OpenAPI 3.1) + web console + offline-first mobile app + MCP server.

**Integration surface.** EPA e-Manifest (RCRA), UK Waste Services API, Geotab + Samsara telematics (normalised via Open Telematics API patterns), weighbridge hardware (serial RS-232 / TCP-IP Modbus adapters), QuickBooks Online, commodity price feeds, S3-compatible object storage for photos.

**Standards implemented.** OpenAPI 3.1, OAuth 2.0 (RFC 6749) + OIDC, JSON (RFC 8259), EPA e-Manifest JSON Schema, UK Waste Services API, ISO 24161 vocabulary as the semantic baseline, OWASP API Security Top 10 (2023), GDPR data-handling, ISO 14001 / CSRD ESRS E5 reporting exports, MCP specification.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | Domain is ML/LLM-heavy (route RL, contamination CV, manifest generation). Python has the strongest ecosystem for OR-Tools, PyTorch, and LLM SDKs, plus mature geospatial/scale-adapter libraries. |
| API framework | **FastAPI** | Native async (essential for telematics polling and long LLM calls), auto-generates OpenAPI 3.1 (a stated standard requirement), Pydantic v2 validation maps cleanly onto the hybrid JSONB model. |
| Database | **PostgreSQL 16 + PostGIS 3.4 + pgRouting** | Hybrid relational/JSONB model (Suggestion 3) needs typed columns + GIN-indexed JSONB + `GEOGRAPHY` types + in-DB routing. Single engine, ACID, audit-ready. |
| Time-series store | **TimescaleDB 2.x (same Postgres instance)** | GPS traces, bin fill levels, diagnostics, commodity prices are append-mostly high-volume series (Suggestion 4). Hypertables in the same instance allow joins to relational tables with no ETL. Introduced Phase 8. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic + GeoAlchemy2** | Async ORM support, explicit migration history (compliance requires auditable schema evolution), spatial column types. |
| Task queue | **Celery + Redis** | Async workloads: telematics ingestion, LLM manifest generation, contamination inference, invoice batch runs, webhook delivery. Redis doubles as read-model/cache. |
| Real-time push | **Redis Pub/Sub + WebSocket (FastAPI)** | Live route edits must propagate to driver apps and supervisor dashboards instantly. |
| Routing/optimisation | **Google OR-Tools (CP-SAT / Routing solver)** for baseline VRP; **custom RL layer (PyTorch)** later | OR-Tools is the open-source VRP standard; deterministic baseline ships in MVP, RL refinement layers on in Phase 9 without rework. |
| ML — contamination | **PyTorch + torchvision (EfficientNet/ViT fine-tune)**, served via **ONNX Runtime** | Image classification at point of collection; ONNX keeps inference portable and CPU-deployable for self-hosters. |
| LLM integration | **Provider-agnostic via LiteLLM** (Anthropic default) | Manifest/transfer-note generation and billing-exception review. LiteLLM keeps self-hosters free to plug in local or alternate models. |
| Web frontend | **TypeScript + React (Vite) + MapLibre GL + TanStack Query** | Dispatch board, map views, dashboards. MapLibre is open-source (no Mapbox token lock-in) and renders PostGIS/vector tiles. |
| Mobile (driver) | **React Native + WatermelonDB (offline) + SQLite** | Offline-first sync is a hard requirement; React Native shares typing with the web app; WatermelonDB gives a sync-friendly local store. |
| Object storage | **S3-compatible (MinIO bundled, AWS S3 in cloud)** | Contamination photos, signed manifests, weigh-ticket scans. |
| Auth | **OAuth 2.0 / OIDC via Authlib; JWT access tokens** | Standards requirement (RFC 6749, OIDC). Supports enterprise SSO and the third-party-integration token model. |
| MCP server | **Python `mcp` SDK** | Expose route status, manifest generation, compliance checks as MCP tools — the novel AI-native differentiator. |
| Containerisation | **Docker + Docker Compose; Helm chart** | Self-hosted is the primary deployment mode. |
| Testing | **pytest + pytest-asyncio + testcontainers**; **Vitest + Playwright** (web); **Jest + Detox** (mobile) | testcontainers gives a real PostGIS/Timescale instance for integration tests; Playwright/Detox for E2E. |
| Quality tools | **Ruff (lint+format), mypy (strict), pre-commit**; **ESLint + Prettier + tsc** | Enforced in CI; type safety matters for the hybrid model boundary. |
| Package mgmt | **uv (Python), pnpm (JS)** | Fast, reproducible lockfiles. |

### Project Structure

```
recycling-waste-management/
├── docker-compose.yml                # postgres+postgis+timescale, redis, minio, api, worker, web
├── docker-compose.dev.yml
├── Dockerfile.api
├── Dockerfile.worker
├── pyproject.toml                    # uv-managed
├── alembic.ini
├── deploy/
│   └── helm/                         # production Kubernetes chart
├── docs/
│   └── openapi.json                  # generated, committed for reference
├── backend/
│   ├── src/wms/
│   │   ├── main.py                   # FastAPI app factory, router mounting
│   │   ├── config.py                 # pydantic-settings, env vars
│   │   ├── db/
│   │   │   ├── base.py               # SQLAlchemy async engine/session
│   │   │   ├── models/               # ORM models grouped by domain
│   │   │   │   ├── org.py            # organizations, users
│   │   │   │   ├── customer.py       # customers, service_locations, agreements
│   │   │   │   ├── catalog.py        # material_types, container_types, containers
│   │   │   │   ├── fleet.py          # vehicles, drivers
│   │   │   │   ├── routing.py        # routes, route_stops, route_executions
│   │   │   │   ├── materials.py      # collection_events, weigh_tickets, contamination
│   │   │   │   ├── compliance.py     # manifests, transfer_notes, compliance_rules
│   │   │   │   └── billing.py        # invoices, invoice_lines, payments, price_index
│   │   │   └── migrations/           # Alembic versions
│   │   ├── schemas/                  # Pydantic request/response models per domain
│   │   ├── api/
│   │   │   ├── deps.py               # auth, db session, tenancy dependencies
│   │   │   └── v1/                   # routers per domain
│   │   ├── services/                 # business logic (pure, testable)
│   │   │   ├── routing/              # OR-Tools VRP, RL refinement
│   │   │   ├── billing/             # rating engine
│   │   │   ├── compliance/          # rule engine + jurisdiction plugins
│   │   │   ├── contamination/       # ML inference client
│   │   │   └── llm/                 # manifest/exception generation
│   │   ├── integrations/
│   │   │   ├── telematics/          # geotab.py, samsara.py, base.py (normaliser)
│   │   │   ├── weighbridge/         # serial.py, tcp_modbus.py, base.py
│   │   │   ├── epa_emanifest/
│   │   │   ├── uk_waste/
│   │   │   └── quickbooks/
│   │   ├── tasks/                    # Celery tasks
│   │   ├── realtime/                 # WebSocket + Redis pub/sub
│   │   ├── mcp/                      # MCP server tool definitions
│   │   └── auth/                     # OAuth2/OIDC, JWT, RBAC
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/                 # sample diffs, manifests, telematics payloads
├── web/                              # React + Vite dispatch console
│   ├── src/{routes,components,api,maps,state}/
│   └── tests/
├── mobile/                           # React Native driver app
│   └── src/{screens,sync,db,components}/
└── ml/                               # contamination model training pipeline
    ├── train.py
    ├── export_onnx.py
    └── datasets/
```

---

## Phase 1: Foundation — Project Skeleton, Tenancy, Auth, Core Schema

### Purpose
Establish the runnable backbone: containerised PostGIS + Redis + MinIO, FastAPI app factory, configuration, multi-tenant data model for organisations and users, OAuth2/JWT auth with RBAC, and CI. After this phase a developer can register an organisation, create users, log in, and call an authenticated health endpoint. Everything later builds on the tenancy and auth primitives created here.

### Tasks

#### 1.1 — Project scaffolding & containers
**What**: Create the repo skeleton, `pyproject.toml` (uv), Dockerfiles, and `docker-compose` bringing up PostGIS, Redis, and MinIO.

**Design**:
- `docker-compose.yml` services: `db` (`postgis/postgis:16-3.4` with TimescaleDB extension preloaded), `redis:7`, `minio`, `api`, `worker`, `web`.
- `config.py` using `pydantic-settings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str; s3_access_key: str; s3_secret_key: str; s3_bucket: str = "wms"
    jwt_secret: str; jwt_algorithm: str = "HS256"; access_token_ttl_min: int = 30
    llm_provider: str = "anthropic"; llm_api_key: str | None = None
    environment: Literal["dev", "test", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="WMS_", env_file=".env")
```
- `main.py`: app factory `create_app() -> FastAPI`, mounts `/api/v1`, `/health`, CORS, exception handlers, OpenAPI metadata (`title`, `version`, `openapi_version="3.1.0"`).
- CI: GitHub Actions running Ruff, mypy, pytest with a Postgres service.

**Testing**:
- `Unit: create_app() returns FastAPI with /health route registered`
- `Integration: GET /health → 200 {"status":"ok","db":"ok","redis":"ok"} when services up`
- `Integration (testcontainers): app boots against real PostGIS, PostGIS + TimescaleDB extensions present`
- `CI smoke: docker compose up → /health reachable`

#### 1.2 — Core schema: organizations & users (Alembic migration 0001)
**What**: Implement the `organizations` and `users` tables from data-model Suggestion 3 as SQLAlchemy models + first Alembic migration.

**Design**:
- ORM models mirror Suggestion 3 DDL. Key columns:
```python
class Organization(Base):
    __tablename__ = "organizations"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    name: Mapped[str]
    org_type: Mapped[str]  # CHECK: hauler|municipality|mrf|transfer_station|landfill|broker|generator
    parent_org_id: Mapped[UUID | None] = mapped_column(ForeignKey("organizations.id"))
    epa_site_id: Mapped[str | None]
    default_currency: Mapped[str] = mapped_column(default="USD")
    default_timezone: Mapped[str] = mapped_column(default="America/New_York")
    location: Mapped[WKBElement | None] = mapped_column(Geography("POINT", 4326))
    regulatory_ids: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at / updated_at / deleted_at (soft delete)

class User(Base):
    id, organization_id (FK), email (unique), password_hash
    first_name, last_name, phone
    role: Mapped[str]  # CHECK enum: admin|dispatcher|driver|supervisor|billing_clerk|compliance_officer|customer_portal|weighbridge_operator|mechanic
    profile_data: Mapped[dict] = mapped_column(JSONB, default=dict)  # license, certs, endorsements
    is_active, last_login_at, created_at, updated_at
```
- Indexes: `org_type`, partial index on `epa_site_id`, GIST on `location`, GIN on `profile_data`, `users(organization_id)`, `users(role)`.
- Migration `0001_core` creates extensions (`postgis`, `pg_trgm`, `btree_gist`), then tables.

**Testing**:
- `Unit: Organization model validates org_type against allowed set`
- `Integration (testcontainers): migration 0001 applies and downgrades cleanly`
- `Integration: insert org with location POINT → round-trips via GeoAlchemy2`
- `Integration: duplicate user email → IntegrityError`

#### 1.3 — Auth: registration, OAuth2 password+refresh, JWT, RBAC
**What**: Authentication endpoints and an RBAC dependency enforcing role + tenancy isolation.

**Design**:
- Endpoints:
  - `POST /api/v1/auth/register-org` → creates org + first admin user. Body: `{org: {...}, admin: {email, password, first_name, last_name}}`.
  - `POST /api/v1/auth/token` (OAuth2 password grant) → `{access_token, refresh_token, token_type:"bearer", expires_in}`.
  - `POST /api/v1/auth/refresh` → new access token.
  - `GET /api/v1/auth/me` → current user.
- Password hashing: `argon2`. JWT claims: `sub` (user id), `org` (organization id), `role`, `exp`.
- `deps.py`:
```python
async def current_user(token=Depends(oauth2_scheme), db=Depends(get_db)) -> User: ...
def require_roles(*roles: str) -> Callable: ...   # raises 403 if user.role not in roles
async def tenant_scope(user=Depends(current_user)) -> UUID: return user.organization_id
```
- All domain queries MUST filter by `organization_id == tenant_scope` (OWASP API3:2023 — broken object-level authz). Enforced via a `TenantQuery` helper.
- Error model: RFC 7807 `application/problem+json` envelope `{type, title, status, detail, instance}`.

**Testing**:
- `Unit: argon2 hash/verify round-trip`
- `Unit: JWT encode/decode includes org and role claims; expired token → 401`
- `Integration: register-org creates org+admin, returns 201`
- `Integration: token grant with valid creds → 200 with tokens; bad password → 401`
- `Integration: require_roles("admin") on dispatcher token → 403`
- `Integration (tenancy): user from org A cannot read org B object → 404 (not 403, to avoid existence leak)`
- `Integration: refresh with rotated/blacklisted token → 401`

---

## Phase 2: Customers, Service Locations, Contracts & Catalog

### Purpose
Model the commercial and physical reality: customers, their service locations (geocoded), service agreements with flexible pricing config, and the reference catalog of material types, container types, and container assets. This phase produces the master data that routing, materials, and billing all reference. After it, an operator can fully configure who they serve, where, for what materials, with what containers and rates.

### Tasks

#### 2.1 — Customers, service locations, service agreements (migration 0002)
**What**: CRUD for customers, geocoded service locations, and service agreements with JSONB pricing config.

**Design**:
- Tables exactly per Suggestion 3 (`customers`, `service_locations`, `service_agreements`). Notable JSONB:
  - `customers.settings` — tax exemption, credit limit, auto-pay, notification prefs, custom fields.
  - `service_locations.site_details` — access instructions, time windows, clearance, site contacts, photos.
  - `service_agreements.pricing_config` — `base_rates[]`, `surcharge_rules`, `volume_discounts[]`, `commodity_revenue_share`, `sla` (see Suggestion 3 §Customers).
- Pydantic schemas validate the JSONB sub-structures (e.g., `PricingConfig`, `BaseRate`, `SurchargeRules` models) so the API rejects malformed pricing even though storage is JSONB.
- Endpoints under `/api/v1/customers`, `/customers/{id}/locations`, `/customers/{id}/agreements`. Standard list (paginated, filterable by `customer_type`, `account_status`), get, create, update, soft-delete.
- Geocoding: `service_locations.location` accepts `{lat, lng}`; optional integration with a geocoder is deferred — MVP requires explicit coordinates.

**Testing**:
- `Unit: PricingConfig schema rejects base_rate missing 'rate'; accepts full example`
- `Integration: create customer → create location with POINT → create agreement → all linked`
- `Integration: list customers filters by customer_type and is tenant-scoped`
- `Integration: GIN index used for settings.tax_exempt query (EXPLAIN contains index scan)`
- `Integration: soft-delete customer hides from default list but agreement history retained`

#### 2.2 — Catalog: material types, container types, containers (migration 0003)
**What**: Reference catalog and container asset inventory with location/condition tracking.

**Design**:
- `material_types` (code, name, category, is_hazardous, `regulatory_codes` JSONB incl. EPA RCRA codes, EWC code, DOT classification; `properties` JSONB incl. commodity_index, density, contamination_threshold).
- `container_types` (capacity, tare/max gross weight kg, `specifications` JSONB).
- `containers` (serial, rfid_tag, barcode, `current_location_id`, `current_customer_id`, status, condition, `location` GEOGRAPHY, `metadata` JSONB). Status enum: `in_service|in_yard|in_repair|retired|lost`.
- Seed migration loads a default `material_types` set keyed by ISO 24161 vocabulary and common EWC/RCRA codes (MSW, OCC/cardboard, ALU, HDPE, glass, organics, e-waste, D001 hazardous).
- Container assignment endpoint: `POST /containers/{id}/assign {customer_id, location_id}` records assignment and updates `current_*` fields.

**Testing**:
- `Unit: material_type with is_hazardous=true requires regulatory_codes.epa_rcra_codes (schema rule)`
- `Integration: seed migration loads ≥8 default material types`
- `Integration: assign container updates current_customer_id and current_location_id`
- `Integration: container serial uniqueness enforced`
- `Integration: query containers within 5km of a point uses GIST index`

---

## Phase 3: Fleet, Collection Events & Manual Weight Recording

### Purpose
Introduce the operational record at the heart of the product: vehicles, the linkage of materials to customers via collection events, and weigh-ticket recording (manual entry first; hardware in Phase 6). This is the data that compliance and billing consume. After this phase, an operator can record that a given vehicle collected a given material of a given weight at a given stop — the atomic transaction of the whole business.

### Tasks

#### 3.1 — Vehicles & drivers (migration 0004)
**What**: Vehicle inventory with maintenance/inspection metadata and driver assignment.

**Design**:
- `vehicles` (id, organization_id, registration/plate, vehicle_type enum `front_loader|side_loader|rear_loader|roll_off|grapple`, capacity_kg, capacity_volume_m3, telematics_device_id, status `available|on_route|maintenance|out_of_service`, `location` GEOGRAPHY, `metadata` JSONB for maintenance schedule, last_inspection, odometer).
- Drivers are `users` with `role='driver'`; `profile_data` holds license class/expiry/endorsements.
- `vehicle_assignments` (vehicle_id, driver_id, date) for daily assignment.

**Testing**:
- `Unit: vehicle_type enum validation`
- `Integration: assign driver to vehicle for a date; conflicting double-assignment → 409`
- `Integration: list vehicles by status, tenant-scoped`

#### 3.2 — Collection events & weigh tickets (migration 0005)
**What**: Core transactional records linking a collection to customer, location, material, vehicle, and weight.

**Design**:
```python
class CollectionEvent(Base):
    id, organization_id, service_location_id (FK), customer_id (FK)
    vehicle_id (FK), driver_id (FK), material_type_id (FK), container_id (FK nullable)
    route_stop_id: Mapped[UUID | None]   # linked in Phase 5
    collected_at: Mapped[datetime]
    status: Mapped[str]  # collected|missed|skipped|exception
    location: Mapped[WKBElement | None] = mapped_column(Geography("POINT", 4326))
    quantity_value: Mapped[Decimal | None]; quantity_unit: Mapped[str | None]
    quality_grade: Mapped[str | None]
    event_data: Mapped[dict] = mapped_column(JSONB, default=dict)  # photos[], notes, exception_reason

class WeighTicket(Base):
    id, organization_id, collection_event_id (FK nullable), facility_org_id (FK)
    vehicle_id (FK), material_type_id (FK)
    gross_weight_kg: Mapped[Decimal]; tare_weight_kg: Mapped[Decimal]
    net_weight_kg: Mapped[Decimal]   # generated column = gross - tare
    weighed_at: Mapped[datetime]
    source: Mapped[str]  # manual|weighbridge|estimated
    ticket_number: Mapped[str]
    ticket_data: Mapped[dict] = mapped_column(JSONB)  # raw scale fields, operator, scan url
    discrepancy_flag: Mapped[bool] = mapped_column(default=False)
```
- `net_weight_kg` is a Postgres generated column.
- Discrepancy detection service: if `net_weight_kg` deviates from material/container expected range (from `container_types.max_gross_weight_kg` or contract expected weight) beyond a threshold → `discrepancy_flag=true` and emit alert.
- Endpoints: `POST /collection-events`, `POST /weigh-tickets`, list/filter by date range, customer, material, vehicle.

**Testing**:
- `Unit: net_weight computed correctly; tare > gross → ValidationError`
- `Unit: discrepancy detection flags load 20% over container max`
- `Integration: create collection event → create weigh ticket linked → net weight persisted`
- `Integration: list collection events by date range + material, tenant-scoped`
- `Integration: weigh ticket without collection_event (gate weighing) allowed`

---

## Phase 4: Billing & Invoicing Engine

### Purpose
Turn operational records into revenue. The rating engine reads `service_agreements.pricing_config` and the collection/weigh-ticket records to generate invoices supporting weight-based, volume-based, and flat-rate billing, plus surcharges (contamination, overweight, fuel, environmental). After this phase the system closes the loop from collection to invoice — the second half of the core value proposition.

### Tasks

#### 4.1 — Rating engine (pure service)
**What**: A deterministic, side-effect-free engine that computes charge lines from agreements + events over a billing period.

**Design**:
```python
@dataclass
class ChargeLine:
    description: str
    quantity: Decimal
    unit: str
    unit_rate: Decimal
    amount: Decimal
    material_type_id: UUID | None
    source_event_ids: list[UUID]
    line_type: str  # base|weight|surcharge|discount|commodity_share|credit

def rate_period(agreement: ServiceAgreement,
                events: list[CollectionEvent],
                tickets: list[WeighTicket],
                period: DateRange,
                price_index: PriceIndexLookup) -> list[ChargeLine]: ...
```
- Algorithm:
  1. Emit base rates per `pricing_config.base_rates` (per_month prorated; per_pickup × count of matching events).
  2. Weight-based lines: sum `net_weight_kg` per material × rate; apply `overweight` surcharge when load exceeds `overweight_pct_threshold`.
  3. Apply `surcharge_rules`: contamination (per incident from contamination records — Phase 7), fuel_surcharge_pct, environmental_fee.
  4. Apply `volume_discounts` by monthly tonnage tier.
  5. Apply `commodity_revenue_share` credits using `price_index` lookup for shared materials.
  6. Apply missed-pickup SLA credits.
- All money is `Decimal`; rounding `ROUND_HALF_UP` to currency minor units. No floats anywhere in billing.

**Testing**:
- `Unit: flat per_month rate prorated for mid-period start`
- `Unit: weight-based line sums multiple tickets correctly`
- `Unit: overweight surcharge applied only above threshold`
- `Unit: volume discount tier selection (49.9t → 0%, 50t → 5%, 100t → 10%)`
- `Unit: commodity revenue share credit = net_weight × index_price × share_pct`
- `Unit: missed pickup → SLA credit line`
- `Unit: zero events → only base lines`

#### 4.2 — Invoices, lines, payments (migration 0006) & invoice API
**What**: Persist invoices, expose generation and listing endpoints.

**Design**:
- `invoices` (id, organization_id, customer_id, invoice_number, period_start, period_end, status `draft|issued|paid|void|overdue`, subtotal, tax_total, total, currency, due_date, `metadata` JSONB).
- `invoice_lines` (invoice_id, description, quantity, unit, unit_rate, amount, line_type, material_type_id, `source` JSONB with `source_event_ids`).
- `payments` (invoice_id, amount, method, paid_at, reference).
- `POST /customers/{id}/invoices/generate {period_start, period_end}` → runs rating engine, persists a `draft` invoice. `POST /invoices/{id}/issue`, `POST /invoices/{id}/payments`.
- Invoice numbering: per-org monotonic sequence.

**Testing**:
- `Integration: generate invoice for customer over period → draft with correct line breakdown`
- `Integration: issue invoice → status issued, due_date set from payment_terms_days`
- `Integration: record payment ≥ total → status paid; partial → still issued`
- `Integration: regenerate over same period replaces draft, refuses if already issued (409)`
- `Integration: invoice totals reconcile to sum of lines (property test)`

---

## Phase 5: Route Planning, Dispatch & Real-Time Execution

### Purpose
Deliver collection routing — the most operationally visible feature. Build deterministic VRP optimisation (OR-Tools), a dispatch model linking stops to service locations and collection events, live edits propagated via WebSocket, and the supervisor view. After this phase dispatchers can generate and edit routes and watch execution in real time. This is the engine the AI layer (Phase 9) later refines.

### Tasks

#### 5.1 — Route & stop schema, VRP optimiser (migration 0007)
**What**: Route/stop tables and an OR-Tools-based optimiser producing sequenced stops respecting capacity, time windows, and frequency.

**Design**:
```python
class Route(Base):
    id, organization_id, name, service_date, vehicle_id, driver_id
    status: str  # planned|dispatched|in_progress|completed|cancelled
    geometry: Mapped[WKBElement | None] = mapped_column(Geography("LINESTRING", 4326))
    planned_distance_m, planned_duration_s, metadata: JSONB

class RouteStop(Base):
    id, route_id, service_location_id, sequence: int
    material_type_id, container_id
    planned_arrival, time_window_start, time_window_end
    status: str  # pending|completed|missed|skipped
    actual_arrival, collection_event_id (set on completion)
```
- Optimiser service:
```python
def optimize_route(stops: list[StopInput], vehicle: VehicleCaps,
                   distance_matrix: Matrix,
                   constraints: RouteConstraints) -> OptimizedRoute: ...
```
  Uses OR-Tools Routing model: capacity dimension (vehicle_kg/m3), time dimension (service durations + windows), distance objective. Distance matrix from pgRouting (road network) with haversine fallback.
- Endpoint `POST /routes/optimize {service_date, vehicle_id, location_ids|service_area, constraints}` → persisted `planned` route with sequenced stops.

**Testing**:
- `Unit: optimizer sequences 10 stops respecting a capacity limit (split rejected loads)`
- `Unit: time-window violation makes a stop infeasible → reported, not silently dropped`
- `Integration: optimize over real service locations → route geometry + ordered stops persisted`
- `Integration: optimizer deterministic given fixed seed and matrix`
- `Fixture: 50-stop benchmark completes < 5s`

#### 5.2 — Live dispatch, edits & WebSocket propagation
**What**: Real-time route edits and execution status pushed to drivers and supervisors.

**Design**:
- WebSocket endpoints: `/ws/routes/{route_id}` (driver/supervisor), `/ws/fleet` (supervisor org-wide). Redis pub/sub channel per route + per org.
- Edit operations (each emits an event): reorder stops, add/remove stop, reassign vehicle, mark stop completed/missed. All produce an append-only `route_events` audit log (echoing event-sourcing rationale from Suggestion 2 for the audit-critical routing domain).
- Completing a stop creates a `CollectionEvent` and links `route_stop_id`.
- Supervisor view query: routes with live progress % = completed_stops / total_stops.

**Testing**:
- `Integration (ws): two clients subscribe to route; reorder by client A → client B receives event`
- `Integration: mark stop completed → CollectionEvent created, progress updates, audit row written`
- `Integration: add stop mid-execution → resequenced, drivers notified`
- `Integration: audit log is append-only (no update/delete path)`
- `E2E: optimize → dispatch → complete all stops → route status completed`

---

## Phase 6: Weighbridge & Telematics Integration

### Purpose
Connect the physical hardware: weighbridge scales (electronic weigh tickets) and telematics (GPS/diagnostics) from heterogeneous vendors, normalised behind common interfaces. After this phase weights flow in automatically and live vehicle positions populate the fleet map — replacing manual entry and giving supervisors true real-time visibility.

### Tasks

#### 6.1 — Weighbridge adapters (serial & TCP/IP)
**What**: Hardware-agnostic adapter layer producing `WeighTicket` records from scale hardware.

**Design**:
```python
class WeighbridgeAdapter(Protocol):
    async def read_weight(self) -> ScaleReading: ...   # {gross_kg, stable, raw}

class SerialAdapter(WeighbridgeAdapter): ...   # pyserial, RS-232, vendor parse profiles
class TcpModbusAdapter(WeighbridgeAdapter): ... # pymodbus, register map per vendor
```
- Vendor parse profiles (Toledo, Avery Weigh-Tronix) as config-driven regex/register maps in JSONB `weighbridge_configs`.
- Capture flow: operator selects vehicle+material in console → reads scale → creates `WeighTicket(source='weighbridge')` with raw fields in `ticket_data`. Stability gating: only accept a reading flagged stable.
- No open standard exists (noted in standards.md) → adapters isolated behind the Protocol so new vendors are additive.

**Testing**:
- `Unit: serial frame parser extracts gross weight from Toledo sample frame`
- `Unit: modbus register map → kg conversion`
- `Unit: unstable reading rejected`
- `Integration (mock serial/tcp): full capture → WeighTicket persisted with raw fields`

#### 6.2 — Telematics normalisation (Geotab, Samsara)
**What**: Pull/stream vehicle position + diagnostics from multiple vendors into a unified model.

**Design**:
```python
class TelematicsProvider(Protocol):
    async def fetch_positions(self, since: datetime) -> list[VehiclePosition]: ...
    async def fetch_diagnostics(self, since: datetime) -> list[DiagnosticReading]: ...
```
- `GeotabProvider` (MyGeotab SDK, OAuth2/session token), `SamsaraProvider` (Bearer API key). Both map to normalised `VehiclePosition{device_id, ts, lat, lng, speed, heading}` following Open Telematics API patterns.
- Celery beat task polls each configured provider every N seconds; positions published to `/ws/fleet` and (Phase 8) written to a Timescale hypertable. Pre-Phase-8 storage: `vehicles.location` updated to latest.
- `telematics_configs` table stores per-org credentials (encrypted) and device→vehicle mapping.

**Testing**:
- `Integration (mocked Geotab): fetch_positions → normalised VehiclePosition list`
- `Integration (mocked Samsara): bearer auth header sent; diagnostics mapped`
- `Integration: poll task updates vehicle.location and publishes to fleet ws`
- `Integration: device with no mapped vehicle → skipped, warning logged`

---

## Phase 7: Compliance Engine & Contamination Recording

### Purpose
Deliver the configurable, jurisdiction-aware compliance layer and contamination capture — two of the platform's strongest differentiators. Build a pluggable rule engine, EPA e-Manifest and UK Waste Services integrations, waste-transfer-note and diversion-report generation, and contamination incident recording with photo capture (ML classification added in Phase 9). After this phase operators can produce statutory documents and log contamination tied to billing surcharges.

### Tasks

#### 7.1 — Pluggable compliance rule engine (migration 0008)
**What**: A jurisdiction-keyed rule engine deciding which documents/reports apply and validating their data.

**Design**:
- `compliance_rules` (id, jurisdiction_code e.g. `US-RCRA`, `UK-DWT`, `EU-ESRS-E5`, rule_type, `definition` JSONB, version, effective_from). Rules are data, contributable by the community.
- Plugin interface:
```python
class JurisdictionPlugin(Protocol):
    code: str
    def applicable_documents(self, ctx: ComplianceContext) -> list[DocType]: ...
    def validate(self, doc: DocPayload) -> list[ValidationError]: ...
    def required_fields(self, doc_type: DocType) -> list[Field]: ...
```
- Engine selects plugin by `service_location` jurisdiction (derived from country/state of the location). MVP ships `USRcraPlugin` and `UKDwtPlugin`; ESRS-E5 export stub.

**Testing**:
- `Unit: location in TX, hazardous material → US-RCRA manifest required`
- `Unit: UK location → transfer note via UK-DWT plugin`
- `Unit: missing required manifest field → ValidationError naming the field`
- `Integration: load community rule from JSONB and apply`

#### 7.2 — EPA e-Manifest & UK Waste Services integration
**What**: Generate/submit hazardous-waste manifests (US) and align transfer notes with UK standard.

**Design**:
- Internal `Manifest` model aligned to the **EPA e-Manifest JSON Schema** (`generator`, `transporters[]`, `designatedFacility`, `wasteLineItems[]` with RCRA waste codes, `manifestTrackingNumber`). `manifests` table stores typed key fields + full payload JSONB validated against the schema.
- `epa_emanifest` client: auth via RCRAInfo API key (CDX); endpoints submit/retrieve/correct; preprod base URL configurable. All calls through Celery with retry + idempotency key.
- UK: produce transfer-note payload conforming to the UK Waste Services API standard; export endpoint and (optional) submission client.
- Transfer-note / diversion-report generation endpoints: `POST /compliance/transfer-notes`, `GET /compliance/diversion-report?period=...` (computes diversion rate = recycled / total tonnage from weigh tickets, ISO 14001-aligned).

**Testing**:
- `Unit: internal manifest serialises to EPA schema; validates against committed emanifest.json fixture`
- `Integration (mocked EPA API): submit manifest → tracking number stored; 4xx → flagged, retried per policy`
- `Unit: diversion report computes correct rate from fixture weigh tickets`
- `Integration: UK transfer note matches UK Waste Services API shape`

#### 7.3 — Contamination incident recording (migration 0009)
**What**: Capture contamination at collection with photos, severity, and billing linkage (manual classification now; ML in Phase 9).

**Design**:
```python
class ContaminationIncident(Base):
    id, organization_id, collection_event_id (FK), service_location_id, customer_id
    material_type_id, detected_at, severity: str  # minor|major|reject
    classification: str  # manual entry now; ML-populated later
    confidence: Decimal | None
    photo_urls: Mapped[list] = mapped_column(JSONB)  # S3 keys
    status: str  # recorded|notified|disputed|resolved|billed
    incident_data: JSONB  # ml metadata: bounding boxes, model version, feature notes
```
- Photo upload: presigned S3 PUT via `POST /contamination/photos:presign`.
- On `severity in (major, reject)` → triggers customer notification (email/portal) and creates a contamination surcharge eligible for the next invoice (links to rating engine §4.1).

**Testing**:
- `Integration: record incident with 3 photos → S3 keys stored, status recorded`
- `Integration: major incident → notification queued + surcharge eligible`
- `Integration: contamination surcharge appears on next generated invoice`
- `Unit: presign returns time-limited PUT URL scoped to org prefix`

---

## Phase 8: Telemetry Time-Series, Analytics & Sustainability Dashboards

### Purpose
Introduce TimescaleDB hypertables (Suggestion 4) for high-volume telemetry and build the analytics layer: diversion/recovery rates, route efficiency, and per-event carbon accounting with customer-facing ESG/sustainability dashboards. After this phase the platform answers the "how are we doing" questions for operators, customers, and regulators.

### Tasks

#### 8.1 — TimescaleDB hypertables for telemetry (migration 0010)
**What**: Convert/introduce telemetry tables as hypertables with compression, retention, and continuous aggregates.

**Design**:
- Hypertables: `gps_telemetry(time, vehicle_id, location GEOGRAPHY, speed, heading)`, `vehicle_diagnostics(time, vehicle_id, code, value)`, `bin_fill_readings(time, container_id, fill_pct)`, `commodity_prices(time, index_code, material_type_id, price, currency)`.
- `create_hypertable(...)`, `add_compression_policy` after 7d, `add_retention_policy` (configurable; default 13 months for GPS).
- Continuous aggregates: `route_telemetry_hourly` (distance, idle time per vehicle), used by route-efficiency analytics. Telematics ingestion (6.2) now writes here.

**Testing**:
- `Integration (testcontainers timescale): hypertable created; insert + time-range query`
- `Integration: continuous aggregate refreshes and returns hourly rollups`
- `Integration: compression policy applies to old chunk`

#### 8.2 — Analytics & carbon accounting
**What**: Metrics endpoints and per-route/per-stop carbon footprint.

**Design**:
- `GET /analytics/diversion?period=&customer_id=` → diversion rate, recovery rate, tonnage by stream.
- `GET /analytics/route-efficiency` → stops/route, km/stop, idle %, from continuous aggregates.
- Carbon engine: per route, `CO2e = distance_km × vehicle_emission_factor + processing_factors`. Emission factors in a config table; per-stop allocation = route CO2e / stop count, plus material-specific avoided-emissions credit for recycled tonnage. ESRS-E5 / ISO 14001 export builder.
- `GET /analytics/sustainability?customer_id=` powers the customer ESG dashboard.

**Testing**:
- `Unit: diversion rate = recycled / (recycled + disposed) from fixtures`
- `Unit: carbon per stop = route CO2e / stop count; avoided emissions credited for recycled mass`
- `Integration: route-efficiency endpoint reads continuous aggregate`
- `Integration: ESRS-E5 export contains required waste-by-stream disclosures`

---

## Phase 9: AI-Native Layer — RL Routing, Contamination ML, LLM Compliance & MCP

### Purpose
Layer on the AI-native differentiators atop the now-complete deterministic system: reinforcement-learning route refinement, ML contamination classification, LLM-assisted compliance-document and billing-exception generation, and an MCP server exposing these capabilities to LLM agents. Each is additive — the deterministic baselines from earlier phases remain the fallback.

### Tasks

#### 9.1 — Contamination image classification model
**What**: Train, export, and serve an image classifier that populates `ContaminationIncident.classification/confidence`.

**Design**:
- `ml/train.py` fine-tunes EfficientNet/ViT on labelled bin photos (dataset spec + augmentation in `ml/datasets/`). `export_onnx.py` produces an ONNX model. Served via ONNX Runtime in `services/contamination/inference.py`.
- On photo upload (7.3), a Celery task runs inference, writes `classification`, `confidence`, and bounding boxes into `incident_data`; if confidence ≥ threshold and severe → auto-notify; else queue for human review.
- Model version recorded for auditability; deterministic baseline = "manual" remains valid.

**Testing**:
- `Unit: inference client returns label+confidence for fixture image`
- `Integration: photo upload → inference task → incident classification populated`
- `Integration: low-confidence result → status remains 'recorded' (human review), no auto-notify`
- `ML eval: held-out accuracy/precision/recall reported (gate ≥ target in CI artifact)`

#### 9.2 — RL route refinement
**What**: A learning layer that refines OR-Tools sequences using historical stop durations and traffic from telemetry.

**Design**:
- Reward = negative (actual route duration + missed-window penalty). Policy refines OR-Tools output (re-sequencing within feasibility) trained on `route_telemetry_hourly` + actual vs planned arrival deltas.
- `services/routing/rl.py` exposes `refine(route: OptimizedRoute, history: RouteHistory) -> OptimizedRoute`; falls back to OR-Tools output if model unavailable or produces infeasible result.
- Flag `WMS_ROUTING_RL_ENABLED`; A/B comparison metric stored per route (planned vs actual duration).

**Testing**:
- `Unit: refine() never returns infeasible route (capacity/time-window preserved)`
- `Unit: RL disabled → returns OR-Tools baseline unchanged`
- `Integration: refined route logged with A/B duration comparison`

#### 9.3 — LLM compliance & billing-exception generation
**What**: LLM-assisted drafting of transfer notes/manifests from structured data and review of billing exceptions.

**Design**:
- `services/llm/manifest.py`: builds a structured prompt from `CollectionEvent`/`WeighTicket`/`material_type` data; output validated against the compliance plugin's `required_fields` and EPA/UK schemas before persistence. LLM never bypasses schema validation — it drafts, the engine validates.
- System prompt template (stored in repo) instructs strict adherence to jurisdiction fields and prohibits fabricated codes.
- `services/llm/exception.py`: reviews overweight/discrepancy/contamination billing exceptions, proposes an adjustment with cited supporting `source_event_ids`; a human approves before the invoice line is created.
- Provider-agnostic via LiteLLM; all calls async + cached + cost-logged.

**Testing**:
- `Unit: generated manifest passes plugin.validate(); invalid LLM output → rejected, error surfaced`
- `Unit: exception reviewer cites real source_event_ids only (no fabrication)`
- `Integration (mocked LLM): draft transfer note → validated → persisted as draft`
- `Integration: human approval required before adjustment line added to invoice`

#### 9.4 — MCP server
**What**: Expose route status, manifest generation, and compliance checks as MCP tools for LLM agents.

**Design**:
- `mcp/server.py` (Python `mcp` SDK) tools: `get_route_status(route_id)`, `generate_manifest(collection_event_id)`, `check_compliance(service_location_id, material_type_id)`, `list_open_exceptions(org_id)`. Auth via scoped service token mapped to an org; all tools enforce tenancy.
- Tools are thin wrappers over existing services — no business logic duplicated.

**Testing**:
- `Integration: MCP client lists tools; get_route_status returns progress for valid route`
- `Integration: tenancy enforced — token for org A cannot query org B route`
- `Integration: generate_manifest tool routes through schema validation`

---

## Phase 10: Web Console, Driver Mobile App & QuickBooks Integration

### Purpose
Deliver the human-facing surfaces and the most-requested accounting integration. The React dispatch console (map, dispatch board, dashboards), the offline-first React Native driver app, and QuickBooks Online billing sync. After this phase the platform is usable end-to-end by non-developers and back-office adoption is frictionless.

### Tasks

#### 10.1 — Web dispatch console
**What**: React app: dispatch board (drag-and-drop), MapLibre route/fleet map, customer/contract admin, invoices, compliance, analytics dashboards.

**Design**:
- TanStack Query against the OpenAPI client (generated from `docs/openapi.json`). MapLibre renders PostGIS-served vector tiles for routes, stops, live vehicle positions (WebSocket `/ws/fleet`).
- Dispatch board: list/map/board views (mirroring incumbent UX patterns) with live edits calling route-edit endpoints; optimistic updates reconciled via WS events.
- Role-based navigation per RBAC.

**Testing**:
- `Component (Vitest): dispatch board renders stops, reorder fires API call`
- `E2E (Playwright): login → optimize route → reorder stop → see WS update → mark complete`
- `E2E: generate + issue invoice from customer page`
- `Accessibility: key flows pass axe checks`

#### 10.2 — Offline-first driver mobile app
**What**: React Native app for route execution with offline capture and sync.

**Design**:
- WatermelonDB local store mirrors today's route + stops; actions (complete/miss stop, capture weight/photo, record contamination) queued offline and synced via a `/api/v1/sync` batch endpoint with conflict resolution (last-write-wins on stop status, append for events).
- Sync endpoint: `POST /sync {since, mutations[]}` → `{applied[], conflicts[], server_changes[]}`.
- Camera capture uploads to S3 via presigned URLs when online; queued otherwise.

**Testing**:
- `Unit: mutation queue persists across app restart`
- `Integration: offline complete-stop → reconnect → synced, server CollectionEvent created`
- `Integration: conflicting stop update → resolved per policy, reported`
- `E2E (Detox): execute a route fully offline → sync on reconnect`

#### 10.3 — QuickBooks Online integration
**What**: Push issued invoices, customers, and payments to QuickBooks Online; reconcile.

**Design**:
- OAuth 2.0 connection per org (`quickbooks_configs`, encrypted tokens, refresh handling). On invoice issue → Celery task maps `Invoice`/`InvoiceLine` to QBO Invoice entity; customer upsert by external id; payment sync back.
- Idempotent via stored QBO entity ids; retry with backoff; reconciliation report endpoint.

**Testing**:
- `Integration (mocked QBO): issue invoice → QBO invoice created, external id stored`
- `Integration: customer upsert maps to QBO customer`
- `Integration: token expiry → refresh flow; permanent failure → flagged for review`
- `Integration: duplicate sync is idempotent (no double invoice)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (tenancy, auth, core schema)        ── required by everything
    │
Phase 2: Customers / Contracts / Catalog                ── requires P1
    │
Phase 3: Fleet, Collection Events, Manual Weights       ── requires P2
    ├── Phase 4: Billing & Invoicing                     ── requires P3 (can parallel P5)
    └── Phase 5: Routing, Dispatch, Real-Time            ── requires P3 (can parallel P4)
            │
            ├── Phase 6: Weighbridge & Telematics         ── requires P3,P5
            └── Phase 7: Compliance & Contamination       ── requires P3,P4 (can parallel P6)
                    │
                    Phase 8: Time-Series & Analytics      ── requires P6,P7
                            │
                            Phase 9: AI-Native Layer       ── requires P5,P7,P8
                                    │
            Phase 10: Web / Mobile / QuickBooks            ── web needs P5; mobile needs P5,P3;
                                                              QBO needs P4 (mostly parallelisable once deps met)
```

**Parallelism opportunities**
- Phases 4 and 5 can be built concurrently after Phase 3.
- Phases 6 and 7 can be built concurrently after Phases 3–5.
- Phase 10 sub-tasks (web 10.1, mobile 10.2, QBO 10.3) can each begin as soon as their backend dependencies land, overlapping Phases 8–9.
- The ML model training (9.1) and RL work (9.2) can proceed in `ml/` independently once telemetry (Phase 8) produces training data.

---

## Definition of Done (per phase)

A phase is complete only when:

1. All tasks implemented and merged behind passing CI.
2. All unit and integration tests pass; integration tests run against real PostGIS/Timescale via testcontainers.
3. Ruff (lint + format) and mypy (strict) pass for backend; ESLint + Prettier + `tsc` pass for web/mobile.
4. New Alembic migration(s) created, apply and downgrade cleanly, and are reversible.
5. `docker compose up` builds and boots all affected services; `/health` green.
6. The feature works end-to-end (E2E test or documented manual walkthrough for UI-bearing phases).
7. New/changed API endpoints appear in regenerated `docs/openapi.json` (OpenAPI 3.1) with request/response schemas.
8. Tenancy isolation verified for every new endpoint (OWASP API3:2023 — no cross-org access).
9. New config options documented in `.env.example` and README.
10. For integration-bearing phases: external calls are idempotent, retried, and covered by mocked integration tests; no real third-party credentials required to run the suite.
11. Personal/customer data handling reviewed against GDPR (consent, deletion path) where applicable.
```
