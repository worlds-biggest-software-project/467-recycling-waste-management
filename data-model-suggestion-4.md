# Data Model Suggestion 4: Time-Series + Geospatial Hybrid (TimescaleDB + PostGIS)

> Project: 467 — Recycling & Waste Management
> Model type: Time-series hypertables with geospatial extensions for sensor, telemetry, and operational data
> Database: PostgreSQL 16+ with TimescaleDB 2.x and PostGIS 3.4+

---

## Design Philosophy

Waste management and recycling operations are, at their core, **continuous physical processes that generate time-stamped, location-tagged data streams**. Trucks emit GPS coordinates every few seconds. Bin sensors report fill levels every few hours. Weighbridges record weights at each transaction. Vehicle diagnostics flow from OBD-II adapters, Geotab, and Samsara units. Carbon emissions accrue per-route and per-facility over time. Commodity prices for recovered materials fluctuate daily.

A conventional relational model treats these data streams as rows in ordinary tables, which leads to predictable problems at scale: table bloat, slow analytical queries, manual partitioning, and no built-in support for downsampling or retention policies. An event-sourced model captures state transitions but is not optimized for high-frequency numeric telemetry where you need fast aggregation across millions of readings.

This model separates the domain into two tiers:

1. **Relational reference tables** (standard PostgreSQL) for slowly changing entities: organizations, customers, contracts, vehicles, containers, material types, compliance rules, and users. These tables use conventional primary keys, foreign keys, and constraints.

2. **Time-series hypertables** (TimescaleDB) for high-volume, append-mostly data streams: GPS telemetry, bin fill-level readings, weighbridge transactions, vehicle diagnostics, environmental sensor data, route execution traces, commodity price feeds, and carbon emission calculations. These tables are automatically partitioned by time, compressed after an aging threshold, and rolled up into continuous aggregates for dashboard and reporting queries.

PostGIS extends both tiers with native geospatial types, enabling spatial queries (containers within a service zone, trucks within a geofence, nearest available vehicle) without leaving the database.

The key insight is that **the time-series tier and the relational tier live in the same PostgreSQL instance**, so a single SQL query can join a continuous aggregate of hourly vehicle telemetry against the relational vehicles and routes tables. There is no ETL pipeline, no polyglot persistence boundary, and no eventual consistency gap to manage.

---

## Schema Definition

### Part 1: Relational Reference Tables (Standard PostgreSQL)

These tables store slowly changing reference data. They are conventional PostgreSQL tables, not hypertables.

```sql
-- ============================================================
-- Extensions
-- ============================================================
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pg_trgm;        -- fuzzy text search
CREATE EXTENSION IF NOT EXISTS btree_gist;     -- exclusion constraints

-- ============================================================
-- Organizations & Multi-Tenancy
-- ============================================================
CREATE TABLE organizations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    org_type            VARCHAR(50) NOT NULL CHECK (org_type IN (
                            'hauler', 'municipality', 'mrf', 'transfer_station',
                            'landfill', 'broker', 'generator'
                        )),
    parent_org_id       UUID REFERENCES organizations(id),
    tax_id              VARCHAR(50),
    epa_site_id         VARCHAR(20),
    default_currency    CHAR(3) DEFAULT 'USD',
    default_timezone    VARCHAR(50) DEFAULT 'America/New_York',
    location            GEOGRAPHY(POINT, 4326),
    boundary            GEOGRAPHY(POLYGON, 4326),   -- service area boundary
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_location ON organizations USING GIST(location);
CREATE INDEX idx_org_boundary ON organizations USING GIST(boundary);

-- ============================================================
-- Vehicles & Equipment
-- ============================================================
CREATE TABLE vehicles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    vin                 VARCHAR(17) UNIQUE,
    license_plate       VARCHAR(20),
    vehicle_type        VARCHAR(50) NOT NULL CHECK (vehicle_type IN (
                            'front_loader', 'rear_loader', 'side_loader',
                            'roll_off', 'hook_lift', 'compactor',
                            'tanker', 'flatbed', 'supervisor'
                        )),
    make                VARCHAR(100),
    model               VARCHAR(100),
    year                SMALLINT,
    fuel_type           VARCHAR(20) CHECK (fuel_type IN (
                            'diesel', 'cng', 'lng', 'electric', 'hybrid', 'gasoline'
                        )),
    gross_vehicle_weight_kg NUMERIC(10,2),
    payload_capacity_kg    NUMERIC(10,2),
    telematics_provider VARCHAR(50),          -- 'geotab', 'samsara', 'manufacturer'
    telematics_device_id VARCHAR(100),        -- external device identifier
    home_depot          GEOGRAPHY(POINT, 4326),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicles_org ON vehicles(organization_id);
CREATE INDEX idx_vehicles_telematics ON vehicles(telematics_provider, telematics_device_id);

-- ============================================================
-- Containers (bins, dumpsters, roll-off boxes)
-- ============================================================
CREATE TABLE containers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    customer_id         UUID,                -- nullable for unassigned inventory
    container_type      VARCHAR(50) NOT NULL CHECK (container_type IN (
                            'cart_35gal', 'cart_65gal', 'cart_95gal',
                            'dumpster_2yd', 'dumpster_4yd', 'dumpster_6yd', 'dumpster_8yd',
                            'roll_off_10yd', 'roll_off_20yd', 'roll_off_30yd', 'roll_off_40yd',
                            'compactor', 'underground'
                        )),
    material_stream     VARCHAR(50) NOT NULL CHECK (material_stream IN (
                            'msw', 'recycling_single', 'recycling_dual',
                            'organics', 'yard_waste', 'construction_demolition',
                            'hazardous', 'medical', 'ewaste', 'bulky'
                        )),
    rfid_tag            VARCHAR(50) UNIQUE,
    barcode             VARCHAR(50),
    sensor_id           VARCHAR(100),        -- IoT fill-level sensor ID
    sensor_type         VARCHAR(30),         -- 'ultrasonic', 'radar', 'infrared', 'weight'
    location            GEOGRAPHY(POINT, 4326),
    placed_at           TIMESTAMPTZ,
    last_serviced_at    TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_containers_sensor ON containers(sensor_id) WHERE sensor_id IS NOT NULL;
CREATE INDEX idx_containers_rfid ON containers(rfid_tag) WHERE rfid_tag IS NOT NULL;
CREATE INDEX idx_containers_location ON containers USING GIST(location);
CREATE INDEX idx_containers_material ON containers(material_stream);

-- ============================================================
-- Service Stops (locations on a route)
-- ============================================================
CREATE TABLE service_stops (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    customer_id         UUID NOT NULL,
    address_line1       VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    location            GEOGRAPHY(POINT, 4326) NOT NULL,
    access_notes        TEXT,                -- driver instructions
    time_window_start   TIME,                -- earliest acceptable service time
    time_window_end     TIME,                -- latest acceptable service time
    service_days        SMALLINT[],          -- ISO day-of-week: 1=Mon ... 7=Sun
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stops_location ON service_stops USING GIST(location);
CREATE INDEX idx_stops_customer ON service_stops(customer_id);

-- ============================================================
-- Routes (planned route templates)
-- ============================================================
CREATE TABLE routes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    route_name          VARCHAR(100) NOT NULL,
    route_type          VARCHAR(50) NOT NULL CHECK (route_type IN (
                            'residential', 'commercial', 'roll_off', 'special'
                        )),
    material_stream     VARCHAR(50) NOT NULL,
    assigned_vehicle_id UUID REFERENCES vehicles(id),
    assigned_driver_id  UUID,
    service_day         SMALLINT,            -- ISO day-of-week
    estimated_stops     INTEGER,
    estimated_duration  INTERVAL,
    route_geometry      GEOGRAPHY(LINESTRING, 4326),  -- planned path
    service_zone        GEOGRAPHY(POLYGON, 4326),     -- zone boundary
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routes_org ON routes(organization_id);
CREATE INDEX idx_routes_geometry ON routes USING GIST(route_geometry);
CREATE INDEX idx_routes_zone ON routes USING GIST(service_zone);

-- ============================================================
-- Material Types & Commodity Pricing Reference
-- ============================================================
CREATE TABLE material_types (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(20) NOT NULL UNIQUE,  -- e.g., 'OCC', 'HDPE', 'PET', 'ALU'
    name                VARCHAR(100) NOT NULL,
    category            VARCHAR(50) NOT NULL CHECK (category IN (
                            'paper', 'plastic', 'metal', 'glass', 'organic',
                            'construction', 'hazardous', 'ewaste', 'mixed', 'residual'
                        )),
    default_unit        VARCHAR(10) DEFAULT 'kg',
    density_kg_per_m3   NUMERIC(10,2),       -- for volume-to-weight estimation
    is_recyclable       BOOLEAN NOT NULL DEFAULT true,
    epa_waste_code      VARCHAR(10),         -- e.g., 'D001' for ignitable hazmat
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- Weighbridge Scales (reference metadata)
-- ============================================================
CREATE TABLE weighbridge_scales (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    facility_id         UUID NOT NULL REFERENCES organizations(id),
    scale_name          VARCHAR(100) NOT NULL,
    manufacturer        VARCHAR(100),
    model               VARCHAR(100),
    max_capacity_kg     NUMERIC(12,2) NOT NULL,
    min_division_kg     NUMERIC(8,4) NOT NULL,       -- smallest readable increment
    protocol            VARCHAR(20) CHECK (protocol IN ('serial', 'tcp_ip', 'usb', 'api')),
    connection_string   VARCHAR(255),                 -- COM port or IP:port
    last_calibration    TIMESTAMPTZ,
    calibration_due     TIMESTAMPTZ,
    location            GEOGRAPHY(POINT, 4326),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### Part 2: Time-Series Hypertables (TimescaleDB)

These tables store high-volume, append-mostly data. Each is converted to a TimescaleDB hypertable with automatic time partitioning, compression policies, and retention rules.

#### 2.1 Vehicle GPS Telemetry

The highest-volume stream. A fleet of 100 trucks reporting every 5 seconds generates ~1.7 million rows per day.

```sql
-- ============================================================
-- Vehicle GPS Telemetry (hypertable)
-- ============================================================
CREATE TABLE vehicle_telemetry (
    time                TIMESTAMPTZ         NOT NULL,
    vehicle_id          UUID                NOT NULL,  -- FK to vehicles (not enforced on hypertable for performance)
    location            GEOGRAPHY(POINT, 4326) NOT NULL,
    speed_kmh           REAL,
    heading_degrees     REAL,                -- 0-360 compass bearing
    altitude_m          REAL,
    odometer_km         REAL,
    engine_on           BOOLEAN,
    fuel_level_pct      REAL,                -- 0-100
    engine_rpm          SMALLINT,
    engine_temp_c       REAL,
    dtc_codes           TEXT[],              -- active diagnostic trouble codes
    harsh_event         VARCHAR(20),         -- 'hard_brake', 'hard_accel', 'hard_turn', null
    source              VARCHAR(30) NOT NULL DEFAULT 'telematics',  -- 'telematics', 'mobile_app', 'manual'
    raw_payload         JSONB                -- vendor-specific fields preserved verbatim
);

SELECT create_hypertable(
    'vehicle_telemetry', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Composite index for "show me this truck's path today"
CREATE INDEX idx_vt_vehicle_time ON vehicle_telemetry (vehicle_id, time DESC);

-- Spatial index for "which trucks are near this location right now"
CREATE INDEX idx_vt_location ON vehicle_telemetry USING GIST (location);

-- Partial index for harsh events (safety dashboard)
CREATE INDEX idx_vt_harsh ON vehicle_telemetry (vehicle_id, time DESC)
    WHERE harsh_event IS NOT NULL;

-- Compression: segment by vehicle, order by time
ALTER TABLE vehicle_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'vehicle_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Compress chunks older than 3 days (recent data stays uncompressed for fast writes)
SELECT add_compression_policy('vehicle_telemetry', compress_after => INTERVAL '3 days');

-- Retain raw telemetry for 90 days; older data lives in continuous aggregates
SELECT add_retention_policy('vehicle_telemetry', drop_after => INTERVAL '90 days');
```

#### 2.2 Container Fill-Level Readings

IoT sensors (ultrasonic, radar, infrared) mounted inside bins report fill percentage. Typical reporting interval is 1-6 hours. A fleet of 10,000 smart bins at 4 readings/day generates ~40,000 rows/day.

```sql
-- ============================================================
-- Container Fill-Level Sensor Readings (hypertable)
-- ============================================================
CREATE TABLE container_fill_readings (
    time                TIMESTAMPTZ         NOT NULL,
    container_id        UUID                NOT NULL,  -- references containers.id
    sensor_id           VARCHAR(100)        NOT NULL,  -- hardware sensor identifier
    fill_level_pct      REAL                NOT NULL CHECK (fill_level_pct BETWEEN 0 AND 100),
    temperature_c       REAL,                          -- internal bin temperature
    battery_level_pct   REAL,                          -- sensor battery remaining
    tilt_degrees        REAL,                          -- container tilt (knocked over?)
    signal_strength_dbm REAL,                          -- LoRaWAN/NB-IoT signal
    anomaly_flag        BOOLEAN DEFAULT false,         -- ML-detected anomaly
    location            GEOGRAPHY(POINT, 4326),        -- sensor GPS (if equipped)
    raw_payload         JSONB                          -- vendor-specific fields
);

SELECT create_hypertable(
    'container_fill_readings', 'time',
    chunk_time_interval => INTERVAL '7 days',
    if_not_exists => TRUE
);

CREATE INDEX idx_cfr_container_time ON container_fill_readings (container_id, time DESC);
CREATE INDEX idx_cfr_sensor ON container_fill_readings (sensor_id, time DESC);
CREATE INDEX idx_cfr_high_fill ON container_fill_readings (container_id, time DESC)
    WHERE fill_level_pct >= 80;  -- "nearly full" alert queries

ALTER TABLE container_fill_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'container_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('container_fill_readings', compress_after => INTERVAL '14 days');
SELECT add_retention_policy('container_fill_readings', drop_after => INTERVAL '365 days');
```

#### 2.3 Weighbridge Transactions

Each inbound/outbound weighing at a transfer station, landfill, or MRF is a time-series event. Moderate volume (hundreds to low thousands per facility per day) but high compliance value.

```sql
-- ============================================================
-- Weighbridge Transactions (hypertable)
-- ============================================================
CREATE TABLE weighbridge_transactions (
    time                TIMESTAMPTZ         NOT NULL,  -- timestamp of weight capture
    transaction_id      UUID                NOT NULL DEFAULT gen_random_uuid(),
    scale_id            UUID                NOT NULL,  -- references weighbridge_scales.id
    facility_id         UUID                NOT NULL,  -- references organizations.id
    vehicle_id          UUID,                          -- references vehicles.id (null if external)
    external_plate      VARCHAR(20),                   -- if vehicle not in system
    direction           VARCHAR(10)         NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    gross_weight_kg     NUMERIC(12,2)       NOT NULL,
    tare_weight_kg      NUMERIC(12,2),                 -- known or measured
    net_weight_kg       NUMERIC(12,2) GENERATED ALWAYS AS (gross_weight_kg - COALESCE(tare_weight_kg, 0)) STORED,
    material_type_id    UUID,                          -- references material_types.id
    material_stream     VARCHAR(50),
    quality_grade       VARCHAR(10),                   -- 'A', 'B', 'C', 'rejected'
    contamination_pct   REAL,                          -- estimated contamination percentage
    ticket_number       VARCHAR(50),                   -- printed weigh ticket reference
    manifest_id         VARCHAR(50),                   -- regulatory manifest linkage
    operator_id         UUID,                          -- references users.id
    photo_urls          TEXT[],                         -- load inspection photos
    notes               TEXT,
    weight_discrepancy  BOOLEAN DEFAULT false,         -- flagged by variance check
    raw_scale_data      JSONB                          -- raw protocol response from scale hardware
);

SELECT create_hypertable(
    'weighbridge_transactions', 'time',
    chunk_time_interval => INTERVAL '1 month',
    if_not_exists => TRUE
);

CREATE UNIQUE INDEX idx_wt_txn ON weighbridge_transactions (transaction_id, time);
CREATE INDEX idx_wt_facility_time ON weighbridge_transactions (facility_id, time DESC);
CREATE INDEX idx_wt_vehicle_time ON weighbridge_transactions (vehicle_id, time DESC)
    WHERE vehicle_id IS NOT NULL;
CREATE INDEX idx_wt_material ON weighbridge_transactions (material_type_id, time DESC);
CREATE INDEX idx_wt_discrepancy ON weighbridge_transactions (facility_id, time DESC)
    WHERE weight_discrepancy = true;

ALTER TABLE weighbridge_transactions SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'facility_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Weighbridge data has long compliance retention (7 years in many jurisdictions)
SELECT add_compression_policy('weighbridge_transactions', compress_after => INTERVAL '30 days');
-- No retention policy: compressed data retained indefinitely for compliance
```

#### 2.4 Route Execution Events

Each stop on a route generates an event when the driver arrives, services the stop, or records an exception. This table tracks the actual execution of planned routes as a time series.

```sql
-- ============================================================
-- Route Execution Events (hypertable)
-- ============================================================
CREATE TABLE route_execution_events (
    time                TIMESTAMPTZ         NOT NULL,
    event_id            UUID                NOT NULL DEFAULT gen_random_uuid(),
    route_id            UUID                NOT NULL,   -- references routes.id
    vehicle_id          UUID                NOT NULL,
    driver_id           UUID                NOT NULL,
    stop_id             UUID,                           -- references service_stops.id (null for non-stop events)
    container_id        UUID,                           -- references containers.id
    event_type          VARCHAR(50)         NOT NULL CHECK (event_type IN (
                            'route_started', 'route_completed', 'route_paused', 'route_resumed',
                            'stop_arrived', 'stop_serviced', 'stop_skipped', 'stop_exception',
                            'container_lifted', 'container_set_down',
                            'offload_started', 'offload_completed',
                            'break_started', 'break_ended',
                            'geofence_enter', 'geofence_exit'
                        )),
    location            GEOGRAPHY(POINT, 4326),
    weight_kg           NUMERIC(10,2),                  -- on-board scale reading (if equipped)
    rfid_scanned        VARCHAR(50),                    -- RFID tag read at pickup
    exception_reason    VARCHAR(100),                   -- 'blocked_access', 'contamination', 'overweight', etc.
    contamination_type  VARCHAR(50),                    -- ML classification result
    contamination_confidence REAL,                      -- model confidence 0-1
    photo_url           TEXT,                           -- exception/contamination photo
    notes               TEXT,
    duration_seconds    INTEGER,                        -- time spent at stop
    sequence_number     INTEGER,                        -- planned order on route
    actual_sequence     INTEGER,                        -- actual order driven
    synced_from_offline BOOLEAN DEFAULT false           -- true if uploaded from offline cache
);

SELECT create_hypertable(
    'route_execution_events', 'time',
    chunk_time_interval => INTERVAL '7 days',
    if_not_exists => TRUE
);

CREATE UNIQUE INDEX idx_ree_event ON route_execution_events (event_id, time);
CREATE INDEX idx_ree_route_time ON route_execution_events (route_id, time DESC);
CREATE INDEX idx_ree_vehicle_time ON route_execution_events (vehicle_id, time DESC);
CREATE INDEX idx_ree_stop ON route_execution_events (stop_id, time DESC)
    WHERE stop_id IS NOT NULL;
CREATE INDEX idx_ree_exceptions ON route_execution_events (route_id, time DESC)
    WHERE event_type IN ('stop_exception', 'stop_skipped');
CREATE INDEX idx_ree_contamination ON route_execution_events (time DESC)
    WHERE contamination_type IS NOT NULL;
CREATE INDEX idx_ree_location ON route_execution_events USING GIST (location);

ALTER TABLE route_execution_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'route_id, vehicle_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('route_execution_events', compress_after => INTERVAL '14 days');
SELECT add_retention_policy('route_execution_events', drop_after => INTERVAL '2 years');
```

#### 2.5 Vehicle Diagnostics & Maintenance Telemetry

Separate from GPS telemetry, this captures engine health, DTC codes, and predictive maintenance signals at lower frequency.

```sql
-- ============================================================
-- Vehicle Diagnostics (hypertable)
-- ============================================================
CREATE TABLE vehicle_diagnostics (
    time                TIMESTAMPTZ         NOT NULL,
    vehicle_id          UUID                NOT NULL,
    source              VARCHAR(30)         NOT NULL,   -- 'geotab', 'samsara', 'obd2', 'j1939'
    engine_hours        REAL,
    odometer_km         REAL,
    fuel_consumed_liters REAL,
    fuel_rate_lph       REAL,                           -- liters per hour
    oil_pressure_kpa    REAL,
    coolant_temp_c      REAL,
    transmission_temp_c REAL,
    battery_voltage     REAL,
    dpf_soot_load_pct   REAL,                           -- diesel particulate filter
    def_level_pct       REAL,                           -- diesel exhaust fluid
    active_dtc_codes    TEXT[],                          -- current fault codes
    pending_dtc_codes   TEXT[],                          -- pending fault codes
    brake_pressure_psi  REAL,
    tire_pressure_psi   REAL[],                          -- per-tire array
    hydraulic_pressure_psi REAL,                         -- lift arm hydraulics
    lift_cycle_count    INTEGER,                         -- daily arm lift counter
    raw_payload         JSONB
);

SELECT create_hypertable(
    'vehicle_diagnostics', 'time',
    chunk_time_interval => INTERVAL '7 days',
    if_not_exists => TRUE
);

CREATE INDEX idx_vd_vehicle_time ON vehicle_diagnostics (vehicle_id, time DESC);
CREATE INDEX idx_vd_dtc ON vehicle_diagnostics (vehicle_id, time DESC)
    WHERE array_length(active_dtc_codes, 1) > 0;

ALTER TABLE vehicle_diagnostics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'vehicle_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('vehicle_diagnostics', compress_after => INTERVAL '7 days');
SELECT add_retention_policy('vehicle_diagnostics', drop_after => INTERVAL '1 year');
```

#### 2.6 Commodity Price Feed

Recycled material prices change daily. Tracking them as a time series enables billing tied to market rates and historical trend analysis.

```sql
-- ============================================================
-- Commodity Price Feed (hypertable)
-- ============================================================
CREATE TABLE commodity_prices (
    time                TIMESTAMPTZ         NOT NULL,
    material_type_id    UUID                NOT NULL,   -- references material_types.id
    region              VARCHAR(50)         NOT NULL,   -- e.g., 'US_SOUTHEAST', 'UK_NATIONAL', 'EU_CENTRAL'
    price_per_ton       NUMERIC(10,2)       NOT NULL,
    currency            CHAR(3)             NOT NULL DEFAULT 'USD',
    source              VARCHAR(50)         NOT NULL,   -- 'recyclingmarkets', 'letsrecycle', 'manual'
    grade               VARCHAR(20),                    -- e.g., 'PS 11' (ISRI grade for OCC)
    min_price           NUMERIC(10,2),
    max_price           NUMERIC(10,2),
    notes               TEXT
);

SELECT create_hypertable(
    'commodity_prices', 'time',
    chunk_time_interval => INTERVAL '1 month',
    if_not_exists => TRUE
);

CREATE INDEX idx_cp_material_time ON commodity_prices (material_type_id, time DESC);
CREATE INDEX idx_cp_region ON commodity_prices (region, time DESC);

ALTER TABLE commodity_prices SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'material_type_id, region',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('commodity_prices', compress_after => INTERVAL '30 days');
-- No retention: historical price data retained indefinitely for billing disputes and trend analysis
```

#### 2.7 Carbon Emissions & Sustainability Metrics

Per-route and per-facility emissions calculated from fuel consumption, material diversion, and processing activity. Critical for ESG reporting and GHG Protocol compliance.

```sql
-- ============================================================
-- Carbon Emissions Tracking (hypertable)
-- ============================================================
CREATE TABLE carbon_emissions (
    time                TIMESTAMPTZ         NOT NULL,   -- period end timestamp
    organization_id     UUID                NOT NULL,
    scope               SMALLINT            NOT NULL CHECK (scope IN (1, 2, 3)),
    source_type         VARCHAR(50)         NOT NULL CHECK (source_type IN (
                            'fleet_fuel', 'facility_energy', 'landfill_methane',
                            'processing_energy', 'transport_upstream', 'material_avoided'
                        )),
    source_entity_id    UUID,                           -- vehicle, facility, or route ID
    source_entity_type  VARCHAR(30),                    -- 'vehicle', 'facility', 'route'
    co2e_kg             NUMERIC(14,4)       NOT NULL,   -- CO2 equivalent in kilograms
    fuel_liters         NUMERIC(10,2),
    electricity_kwh     NUMERIC(10,2),
    distance_km         NUMERIC(10,2),
    material_diverted_kg NUMERIC(12,2),                 -- weight diverted from landfill
    emission_factor     NUMERIC(10,6),                  -- kg CO2e per unit
    emission_factor_source VARCHAR(100),                -- 'EPA_2025', 'DEFRA_2026', 'GHG_PROTOCOL'
    calculation_method  VARCHAR(50),                    -- 'fuel_based', 'distance_based', 'spend_based'
    notes               TEXT
);

SELECT create_hypertable(
    'carbon_emissions', 'time',
    chunk_time_interval => INTERVAL '1 month',
    if_not_exists => TRUE
);

CREATE INDEX idx_ce_org_time ON carbon_emissions (organization_id, time DESC);
CREATE INDEX idx_ce_scope ON carbon_emissions (scope, time DESC);
CREATE INDEX idx_ce_source ON carbon_emissions (source_entity_id, time DESC)
    WHERE source_entity_id IS NOT NULL;

ALTER TABLE carbon_emissions SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'organization_id, scope',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('carbon_emissions', compress_after => INTERVAL '90 days');
-- No retention: emissions data retained indefinitely for ESG audit trail
```

#### 2.8 ML Model Inference Log

Contamination detection, demand forecasting, and route optimization models produce predictions that must be tracked, evaluated, and audited.

```sql
-- ============================================================
-- ML Inference Log (hypertable)
-- ============================================================
CREATE TABLE ml_inference_log (
    time                TIMESTAMPTZ         NOT NULL,
    inference_id        UUID                NOT NULL DEFAULT gen_random_uuid(),
    model_name          VARCHAR(100)        NOT NULL,   -- 'contamination_detector_v3', 'demand_forecast_v2'
    model_version       VARCHAR(50)         NOT NULL,
    input_entity_id     UUID,                           -- container, route, stop, etc.
    input_entity_type   VARCHAR(30),
    prediction          JSONB               NOT NULL,   -- structured prediction output
    confidence          REAL,                           -- 0-1 overall confidence
    ground_truth        JSONB,                          -- filled in later when label is available
    feedback            VARCHAR(20),                    -- 'confirmed', 'rejected', 'corrected', null
    latency_ms          INTEGER,                        -- inference time
    input_photo_url     TEXT,                           -- for vision models
    notes               TEXT
);

SELECT create_hypertable(
    'ml_inference_log', 'time',
    chunk_time_interval => INTERVAL '7 days',
    if_not_exists => TRUE
);

CREATE INDEX idx_ml_model ON ml_inference_log (model_name, model_version, time DESC);
CREATE INDEX idx_ml_entity ON ml_inference_log (input_entity_id, time DESC)
    WHERE input_entity_id IS NOT NULL;
CREATE INDEX idx_ml_feedback ON ml_inference_log (model_name, time DESC)
    WHERE feedback IS NOT NULL;

ALTER TABLE ml_inference_log SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'model_name',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ml_inference_log', compress_after => INTERVAL '30 days');
SELECT add_retention_policy('ml_inference_log', drop_after => INTERVAL '2 years');
```

---

### Part 3: Continuous Aggregates

Continuous aggregates are materialized views that TimescaleDB refreshes incrementally. They provide fast dashboard queries without scanning raw data. Critically, they survive retention policy drops -- you can delete raw telemetry after 90 days while keeping hourly or daily aggregates for years.

```sql
-- ============================================================
-- Vehicle Telemetry: Hourly Aggregates
-- ============================================================
CREATE MATERIALIZED VIEW vehicle_telemetry_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time)                     AS bucket,
    vehicle_id,
    AVG(speed_kmh)                                  AS avg_speed_kmh,
    MAX(speed_kmh)                                  AS max_speed_kmh,
    AVG(fuel_level_pct)                             AS avg_fuel_pct,
    MIN(fuel_level_pct)                             AS min_fuel_pct,
    COUNT(*)                                        AS reading_count,
    COUNT(*) FILTER (WHERE harsh_event IS NOT NULL) AS harsh_event_count,
    COUNT(*) FILTER (WHERE engine_on = true)        AS engine_on_readings,
    MAX(odometer_km) - MIN(odometer_km)             AS distance_km
FROM vehicle_telemetry
GROUP BY bucket, vehicle_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'vehicle_telemetry_hourly',
    start_offset  => INTERVAL '4 hours',
    end_offset    => INTERVAL '1 hour',
    schedule_interval => INTERVAL '30 minutes'
);

-- ============================================================
-- Vehicle Telemetry: Daily Aggregates (built on hourly)
-- ============================================================
CREATE MATERIALIZED VIEW vehicle_telemetry_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket)                    AS bucket,
    vehicle_id,
    AVG(avg_speed_kmh)                              AS avg_speed_kmh,
    MAX(max_speed_kmh)                              AS max_speed_kmh,
    SUM(harsh_event_count)                          AS total_harsh_events,
    SUM(distance_km)                                AS total_distance_km,
    SUM(reading_count)                              AS total_readings,
    SUM(engine_on_readings)                         AS total_engine_on_readings
FROM vehicle_telemetry_hourly
GROUP BY time_bucket('1 day', bucket), vehicle_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'vehicle_telemetry_daily',
    start_offset  => INTERVAL '3 days',
    end_offset    => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour'
);

-- ============================================================
-- Container Fill Levels: Daily Summary
-- ============================================================
CREATE MATERIALIZED VIEW container_fill_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time)                      AS bucket,
    container_id,
    AVG(fill_level_pct)                             AS avg_fill_pct,
    MAX(fill_level_pct)                             AS max_fill_pct,
    MIN(fill_level_pct)                             AS min_fill_pct,
    MAX(fill_level_pct) - MIN(fill_level_pct)       AS fill_range_pct,
    AVG(temperature_c)                              AS avg_temp_c,
    MIN(battery_level_pct)                          AS min_battery_pct,
    COUNT(*)                                        AS reading_count,
    COUNT(*) FILTER (WHERE anomaly_flag = true)     AS anomaly_count
FROM container_fill_readings
GROUP BY bucket, container_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'container_fill_daily',
    start_offset  => INTERVAL '3 days',
    end_offset    => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour'
);

-- ============================================================
-- Weighbridge Transactions: Daily Facility Summary
-- ============================================================
CREATE MATERIALIZED VIEW weighbridge_daily_summary
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time)                      AS bucket,
    facility_id,
    material_stream,
    direction,
    COUNT(*)                                        AS transaction_count,
    SUM(net_weight_kg)                              AS total_net_kg,
    AVG(net_weight_kg)                              AS avg_net_kg,
    SUM(net_weight_kg) FILTER (WHERE quality_grade = 'rejected') AS rejected_kg,
    AVG(contamination_pct)
        FILTER (WHERE contamination_pct IS NOT NULL) AS avg_contamination_pct,
    COUNT(*) FILTER (WHERE weight_discrepancy = true) AS discrepancy_count
FROM weighbridge_transactions
GROUP BY bucket, facility_id, material_stream, direction
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'weighbridge_daily_summary',
    start_offset  => INTERVAL '3 days',
    end_offset    => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour'
);

-- ============================================================
-- Carbon Emissions: Monthly Organization Summary
-- ============================================================
CREATE MATERIALIZED VIEW carbon_emissions_monthly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', time)                    AS bucket,
    organization_id,
    scope,
    source_type,
    SUM(co2e_kg)                                    AS total_co2e_kg,
    SUM(fuel_liters)                                AS total_fuel_liters,
    SUM(electricity_kwh)                            AS total_electricity_kwh,
    SUM(distance_km)                                AS total_distance_km,
    SUM(material_diverted_kg)                       AS total_diverted_kg,
    COUNT(*)                                        AS record_count
FROM carbon_emissions
GROUP BY bucket, organization_id, scope, source_type
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'carbon_emissions_monthly',
    start_offset  => INTERVAL '2 months',
    end_offset    => INTERVAL '1 day',
    schedule_interval => INTERVAL '6 hours'
);

-- ============================================================
-- Route Execution: Daily Route Performance
-- ============================================================
CREATE MATERIALIZED VIEW route_performance_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time)                      AS bucket,
    route_id,
    vehicle_id,
    COUNT(*) FILTER (WHERE event_type = 'stop_serviced')    AS stops_completed,
    COUNT(*) FILTER (WHERE event_type = 'stop_skipped')     AS stops_skipped,
    COUNT(*) FILTER (WHERE event_type = 'stop_exception')   AS stops_exception,
    COUNT(*) FILTER (WHERE contamination_type IS NOT NULL)  AS contamination_events,
    AVG(duration_seconds) FILTER (WHERE event_type = 'stop_serviced') AS avg_stop_duration_s,
    SUM(weight_kg) FILTER (WHERE weight_kg IS NOT NULL)     AS total_weight_kg,
    COUNT(*)                                                AS total_events,
    MAX(actual_sequence)                                    AS max_sequence
FROM route_execution_events
GROUP BY bucket, route_id, vehicle_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'route_performance_daily',
    start_offset  => INTERVAL '3 days',
    end_offset    => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour'
);
```

---

### Part 4: Key Analytical Queries

These examples demonstrate the power of combining time-series hypertables with relational reference data and geospatial functions in a single query.

#### 4.1 Real-Time Fleet Map with Last Known Position

```sql
-- Last known position for every active vehicle (for dispatcher dashboard)
SELECT DISTINCT ON (v.id)
    v.id AS vehicle_id,
    v.license_plate,
    v.vehicle_type,
    vt.time AS last_seen,
    ST_Y(vt.location::geometry) AS latitude,
    ST_X(vt.location::geometry) AS longitude,
    vt.speed_kmh,
    vt.engine_on,
    vt.fuel_level_pct,
    r.route_name AS current_route
FROM vehicles v
JOIN vehicle_telemetry vt ON vt.vehicle_id = v.id
LEFT JOIN routes r ON r.assigned_vehicle_id = v.id AND r.is_active
WHERE v.is_active
  AND vt.time > now() - INTERVAL '15 minutes'
ORDER BY v.id, vt.time DESC;
```

#### 4.2 Containers That Need Collection (Dynamic Routing Input)

```sql
-- Containers above 80% fill, ordered by urgency and proximity to a given depot
WITH latest_fill AS (
    SELECT DISTINCT ON (container_id)
        container_id,
        fill_level_pct,
        temperature_c,
        time AS last_reading
    FROM container_fill_readings
    WHERE time > now() - INTERVAL '24 hours'
    ORDER BY container_id, time DESC
)
SELECT
    c.id,
    c.container_type,
    c.material_stream,
    c.rfid_tag,
    lf.fill_level_pct,
    lf.temperature_c,
    lf.last_reading,
    ST_Y(c.location::geometry) AS latitude,
    ST_X(c.location::geometry) AS longitude,
    ST_Distance(c.location, depot.location) / 1000.0 AS distance_from_depot_km
FROM containers c
JOIN latest_fill lf ON lf.container_id = c.id
CROSS JOIN (SELECT location FROM organizations WHERE id = $1) depot  -- $1 = depot org ID
WHERE lf.fill_level_pct >= 80
  AND c.is_active
ORDER BY lf.fill_level_pct DESC, distance_from_depot_km ASC;
```

#### 4.3 Weekly Landfill Diversion Rate

```sql
-- Diversion rate = (total recycled + composted) / (total recycled + composted + landfilled)
SELECT
    wds.bucket,
    org.name AS facility_name,
    SUM(wds.total_net_kg) FILTER (WHERE wds.material_stream NOT IN ('msw', 'residual'))
        AS diverted_kg,
    SUM(wds.total_net_kg) FILTER (WHERE wds.material_stream IN ('msw', 'residual'))
        AS landfilled_kg,
    ROUND(
        100.0 * SUM(wds.total_net_kg) FILTER (WHERE wds.material_stream NOT IN ('msw', 'residual'))
        / NULLIF(SUM(wds.total_net_kg), 0),
        2
    ) AS diversion_rate_pct
FROM weighbridge_daily_summary wds
JOIN organizations org ON org.id = wds.facility_id
WHERE wds.bucket >= now() - INTERVAL '7 days'
  AND wds.direction = 'inbound'
GROUP BY wds.bucket, org.name
ORDER BY wds.bucket, org.name;
```

#### 4.4 Route Efficiency Trend (30-Day Comparison)

```sql
-- Compare stops/hour and kg/stop over the last 30 days vs. previous 30 days
WITH current_period AS (
    SELECT
        route_id,
        AVG(stops_completed) AS avg_stops,
        AVG(avg_stop_duration_s) AS avg_duration_s,
        AVG(total_weight_kg / NULLIF(stops_completed, 0)) AS avg_kg_per_stop
    FROM route_performance_daily
    WHERE bucket >= now() - INTERVAL '30 days'
    GROUP BY route_id
),
previous_period AS (
    SELECT
        route_id,
        AVG(stops_completed) AS avg_stops,
        AVG(avg_stop_duration_s) AS avg_duration_s,
        AVG(total_weight_kg / NULLIF(stops_completed, 0)) AS avg_kg_per_stop
    FROM route_performance_daily
    WHERE bucket >= now() - INTERVAL '60 days'
      AND bucket < now() - INTERVAL '30 days'
    GROUP BY route_id
)
SELECT
    r.route_name,
    cp.avg_stops AS current_avg_stops,
    pp.avg_stops AS previous_avg_stops,
    ROUND(100.0 * (cp.avg_stops - pp.avg_stops) / NULLIF(pp.avg_stops, 0), 1) AS stops_change_pct,
    cp.avg_kg_per_stop AS current_kg_per_stop,
    pp.avg_kg_per_stop AS previous_kg_per_stop
FROM current_period cp
JOIN previous_period pp ON pp.route_id = cp.route_id
JOIN routes r ON r.id = cp.route_id
ORDER BY stops_change_pct DESC;
```

#### 4.5 ESG Dashboard: Monthly Scope 1 Emissions by Source

```sql
SELECT
    bucket AS month,
    source_type,
    SUM(total_co2e_kg) / 1000.0 AS total_co2e_tonnes,
    SUM(total_fuel_liters) AS total_fuel_liters,
    SUM(total_distance_km) AS total_distance_km
FROM carbon_emissions_monthly
WHERE organization_id = $1
  AND scope = 1
  AND bucket >= now() - INTERVAL '12 months'
GROUP BY bucket, source_type
ORDER BY bucket, source_type;
```

---

## Data Retention & Tiered Storage Strategy

The time-series model enables a principled multi-tier data lifecycle that is impossible to achieve cleanly with a conventional relational model:

| Data Stream | Hot (uncompressed) | Warm (compressed) | Continuous Aggregate | Raw Retention |
|---|---|---|---|---|
| Vehicle GPS telemetry | 3 days | 3-90 days | Hourly: 2 years, Daily: indefinite | 90 days |
| Container fill readings | 14 days | 14-365 days | Daily: indefinite | 1 year |
| Weighbridge transactions | 30 days | 30 days-indefinite | Daily: indefinite | Indefinite (compliance) |
| Route execution events | 14 days | 14 days-2 years | Daily: indefinite | 2 years |
| Vehicle diagnostics | 7 days | 7-365 days | -- | 1 year |
| Commodity prices | 30 days | 30 days-indefinite | -- | Indefinite |
| Carbon emissions | 90 days | 90 days-indefinite | Monthly: indefinite | Indefinite (ESG audit) |
| ML inference log | 30 days | 30 days-2 years | -- | 2 years |

This tiered approach means a hauler with 100 trucks, 10,000 smart bins, and 3 facilities can retain years of operational intelligence in approximately 50-100 GB of compressed storage, whereas uncompressed raw telemetry alone would exceed 1 TB per year.

---

## Pros and Cons

### Advantages

1. **Natural fit for operational data streams.** Waste management is defined by continuous physical operations -- trucks moving, bins filling, scales weighing, emissions accruing. Time-series storage is designed precisely for these append-mostly, time-ordered workloads, delivering 10-20x compression and purpose-built aggregation that a conventional relational schema cannot match.

2. **Single-database architecture.** TimescaleDB is a PostgreSQL extension, not a separate system. Relational reference tables (organizations, contracts, vehicles) and time-series hypertables live in the same database instance. Queries can join across both tiers with standard SQL. There is no polyglot persistence layer, no ETL pipeline, and no eventual consistency gap to debug.

3. **Built-in data lifecycle management.** Compression policies, retention policies, and continuous aggregates are declared once and maintained automatically by the database. A hauler does not need a data engineering team to manage partitioning, archival, and rollup pipelines -- TimescaleDB handles this natively.

4. **PostGIS integration for spatial operations.** Route optimization, geofencing, nearest-vehicle queries, and service-zone containment tests are native SQL queries, not application-layer computations. The spatial index works seamlessly on both reference tables and hypertables.

5. **Compliance-friendly retention separation.** Weighbridge data can be retained indefinitely (7-year regulatory requirement in many jurisdictions) while GPS telemetry is compressed after 3 days and dropped after 90 days. Continuous aggregates preserve summary statistics long after raw data is purged, satisfying reporting requirements without unbounded storage growth.

6. **High-throughput ingestion.** TimescaleDB handles 50,000+ inserts per second on modest hardware when batching. This comfortably supports a fleet of 500+ trucks reporting GPS every 5 seconds, thousands of smart bin sensors, and multiple weighbridge scales -- all ingesting concurrently.

7. **Familiar PostgreSQL ecosystem.** The application layer uses standard PostgreSQL drivers, connection poolers (PgBouncer, pgpool), backup tools (pg_dump, pgBackRest), and monitoring (pg_stat_statements). No specialized client libraries, no proprietary query language, no vendor lock-in.

8. **Continuous aggregates replace reporting infrastructure.** Instead of building a separate analytics pipeline (Airflow + Spark + data warehouse), the database materializes hourly, daily, and monthly rollups incrementally. Dashboards query these materialized views directly, reducing infrastructure complexity.

9. **ML pipeline support.** The inference log hypertable enables tracking model performance over time, comparing contamination detection accuracy across model versions, and identifying drift -- all with standard SQL time-series queries.

### Disadvantages

1. **No referential integrity on hypertables.** TimescaleDB hypertables do not support foreign key constraints (a limitation inherited from PostgreSQL's partitioned tables). Vehicle IDs, container IDs, and facility IDs in hypertables are not enforced by the database. Application-layer validation and data pipeline checks must compensate for this gap. Stale references to deleted entities will not be caught automatically.

2. **Schema evolution is harder for hypertables.** Adding or modifying columns on a hypertable with billions of compressed rows requires careful migration. Compressed chunks must be decompressed, altered, and recompressed. Unlike a JSONB approach (Suggestion 3), new fields cannot be added without a DDL operation. Planning ahead for likely schema extensions (using the `raw_payload JSONB` columns) mitigates this.

3. **Compression constrains update patterns.** Compressed chunks are read-only. If a weighbridge transaction needs correction after compression, the workflow is: decompress the chunk, update the row, recompress. For corrections to recent data (within the hot window), this is not an issue. For corrections to month-old data, the operational overhead is noticeable. An append-only correction pattern (write a new "adjustment" row rather than updating the original) is the recommended workaround but adds application complexity.

4. **Chunk management overhead at scale.** With multiple hypertables, each with daily or weekly chunks, the total number of chunks can grow into the thousands. PostgreSQL's planner must consider all chunks during query planning, and very large chunk counts can slow planning time. Tuning chunk intervals (wider intervals for low-volume tables, narrower for high-volume) and maintaining chunk exclusion via time-range filters in queries is essential.

5. **Continuous aggregate limitations.** Continuous aggregates cannot reference joins (they aggregate from a single hypertable). Dashboard queries that combine aggregated telemetry with relational metadata must join the continuous aggregate view with reference tables at query time. Nested continuous aggregates (daily built on hourly) add additional refresh latency. Not all PostgreSQL aggregate functions are supported in continuous aggregates.

6. **Geospatial index overhead on hypertables.** Maintaining a GiST spatial index on the `vehicle_telemetry` hypertable (which receives millions of rows per day) adds measurable write overhead. For deployments where spatial queries on raw telemetry are infrequent, the spatial index should be omitted and replaced with a spatial index on the hourly continuous aggregate instead.

7. **Vendor concentration risk.** Although TimescaleDB is open source (Apache 2.0 for Community Edition), some features (multi-node, certain continuous aggregate capabilities) are restricted to the proprietary Enterprise edition. A deployment that grows beyond a single node may face licensing costs or need to adopt the community fork (which lacks distributed hypertables). The fallback is native PostgreSQL declarative partitioning with manual aggregate maintenance.

8. **Backup and restore complexity.** pg_dump of a database with hundreds of compressed chunks and continuous aggregates produces large dump files. Point-in-time recovery (PITR) via WAL archiving works, but restoring a subset of data (e.g., only one facility's data) is not straightforward. pgBackRest with incremental backups is the recommended approach but adds operational knowledge requirements.

---

## Technology Recommendations

### Core Stack

| Component | Technology | Rationale |
|---|---|---|
| Primary database | PostgreSQL 16+ with TimescaleDB 2.x OSS + PostGIS 3.4+ | Unified time-series, geospatial, and relational in one engine |
| Connection pooling | PgBouncer | Essential for high-connection-count telemetry ingestion |
| Telemetry ingestion | MQTT broker (EMQX or Mosquitto) with bridge to PostgreSQL | Industry standard for IoT device communication |
| Message queue | Apache Kafka or NATS | Buffer between MQTT broker and database for backpressure handling |
| Mobile app sync | REST API with offline queue | Drivers upload route execution events when connectivity is restored |
| Dashboards | Grafana with PostgreSQL/TimescaleDB data source | Native support for time-series panels, geospatial maps, and continuous aggregates |
| Backup | pgBackRest with incremental + WAL archiving | Handles compressed chunk files efficiently |

### Infrastructure Sizing Guidelines

**Small operator (1-10 trucks, 500 bins, 1 facility):**
- Single PostgreSQL instance: 4 vCPU, 16 GB RAM, 500 GB SSD
- Expected data volume: ~2 GB/month raw, ~200 MB/month compressed
- TimescaleDB Community Edition is sufficient

**Medium operator (10-100 trucks, 5,000 bins, 3 facilities):**
- Single PostgreSQL instance: 8 vCPU, 32 GB RAM, 2 TB SSD
- Expected data volume: ~20 GB/month raw, ~2 GB/month compressed
- Dedicated PgBouncer for connection pooling
- Grafana instance for operations dashboards

**Large operator (100-500 trucks, 50,000 bins, 10+ facilities):**
- Primary PostgreSQL instance: 16 vCPU, 64 GB RAM, 4 TB NVMe SSD
- Read replica for dashboards and reporting queries
- Expected data volume: ~100 GB/month raw, ~10 GB/month compressed
- Kafka cluster for telemetry ingestion buffering
- Consider TimescaleDB Enterprise for multi-node if query latency on the primary becomes a bottleneck

### Recommended PostgreSQL Configuration

```ini
# postgresql.conf tuning for time-series + geospatial workloads
shared_buffers = '8GB'                  # 25% of RAM
effective_cache_size = '24GB'           # 75% of RAM
work_mem = '64MB'                       # per-sort/hash operation
maintenance_work_mem = '2GB'            # for compression jobs and index builds
max_connections = 200                   # rely on PgBouncer for pooling
wal_level = 'replica'                   # for streaming replication and PITR
max_wal_size = '4GB'
min_wal_size = '1GB'
checkpoint_completion_target = 0.9

# TimescaleDB-specific
timescaledb.max_background_workers = 8  # for compression and aggregate refresh
timescaledb.telemetry_level = 'off'     # opt out of usage telemetry

# PostGIS
postgis.gdal_enabled_drivers = 'ENABLE_ALL'
```

---

## Migration Considerations

### Migrating from a Conventional Relational Schema

If the project starts with Data Model Suggestion 1 (normalized relational) and later needs to adopt this time-series model, the migration path is incremental:

1. **Install TimescaleDB extension** on the existing PostgreSQL instance. This requires no data migration -- TimescaleDB is an extension, not a separate database.

2. **Identify time-series tables.** Tables that have grown large and are primarily queried by time range -- telemetry logs, weighbridge history, sensor readings -- are candidates for conversion.

3. **Convert tables to hypertables** using `create_hypertable()`. For non-empty tables, TimescaleDB provides a `migrate_data => true` option that moves existing rows into the chunk structure. This is a one-time operation that can be performed during a maintenance window.

   ```sql
   -- Example: convert existing telemetry table
   SELECT create_hypertable(
       'vehicle_telemetry', 'time',
       chunk_time_interval => INTERVAL '1 day',
       migrate_data => true,
       if_not_exists => true
   );
   ```

4. **Add compression and retention policies** incrementally. Start with the highest-volume table (GPS telemetry), monitor for 1-2 weeks, then add policies to additional tables.

5. **Create continuous aggregates** after verifying that hypertable performance is stable. Aggregates can be backfilled from historical data.

6. **Remove foreign key constraints** that reference hypertable columns. Replace with application-layer validation. This is the most disruptive change and should be accompanied by data integrity checks in the ingestion pipeline.

### Migrating from Event Sourcing (Suggestion 2)

If the project starts with event sourcing and needs time-series analytics:

1. **Keep the event store** as the write-side source of truth.
2. **Add TimescaleDB hypertables as read-side projections** that consume events and materialize them into time-series format.
3. This hybrid (event sourcing for writes + TimescaleDB projections for analytics) combines the audit trail benefits of Suggestion 2 with the analytical performance of this model.

### Migrating from Hybrid JSONB (Suggestion 3)

1. **Extract time-series JSONB data** into dedicated hypertables. For example, if telemetry data was stored as JSONB arrays in a `vehicle_events` table, write a migration that unnests the JSONB into the `vehicle_telemetry` hypertable.
2. **Keep JSONB columns** for the `raw_payload` fields in hypertables, preserving the flexibility for vendor-specific data that the hybrid model provides.

---

## Scaling Considerations

### Vertical Scaling

TimescaleDB on a single PostgreSQL node scales well for most waste management deployments. A single 16-core, 64 GB instance can handle:
- 100,000 telemetry inserts per second (batched)
- 100 million+ rows across hypertables
- 50+ concurrent dashboard queries against continuous aggregates
- PostGIS spatial queries across millions of location points

The compression ratio (typically 10-20x for telemetry data) means that effective storage capacity is 10-20x the raw disk size.

### Horizontal Scaling

For deployments that exceed single-node capacity:

1. **Read replicas.** Streaming replication to one or more read-only replicas offloads dashboard, reporting, and analytics queries from the primary. This is the simplest and most effective scaling step.

2. **Functional partitioning.** Separate the telemetry ingestion workload (high write volume, low read complexity) from the operational/billing workload (moderate writes, complex joins) onto different database instances. Telemetry data is consumed by dashboards and ML pipelines but rarely joined with billing tables in real time.

3. **TimescaleDB multi-node (Enterprise).** For very large deployments (1,000+ trucks across multiple regions), TimescaleDB Enterprise supports distributed hypertables that shard data across multiple PostgreSQL nodes. Queries are automatically federated across nodes.

4. **Cold storage offload.** For data beyond the retention window that must be preserved for compliance (e.g., weighbridge transactions older than 7 years), export compressed Parquet files to object storage (S3, GCS) and query with DuckDB or Trino when needed. This pattern keeps the live database lean while satisfying long-term audit requirements.

### High Availability

| Requirement | Solution |
|---|---|
| Zero-downtime failover | Patroni + etcd + HAProxy for automated PostgreSQL failover |
| Point-in-time recovery | pgBackRest with continuous WAL archiving to S3 |
| Read scaling | Streaming replication to 1-3 read replicas |
| Geo-distributed access | Read replicas in each operational region; single primary for writes |
| Disaster recovery | Cross-region WAL shipping with 15-minute RPO |

### Monitoring

Critical metrics to track in production:

- **Chunk count per hypertable** -- excessive chunk proliferation degrades planner performance
- **Compression ratio** -- sudden drops indicate schema changes or data quality issues
- **Continuous aggregate refresh lag** -- dashboard staleness indicator
- **Insert throughput** -- telemetry ingestion rate vs. capacity
- **WAL generation rate** -- storage and replication bandwidth indicator
- **Disk usage by hypertable** -- early warning for capacity planning
- **Slow query log** -- queries that miss chunk exclusion or spatial index

```sql
-- Monitor chunk count and compression ratio per hypertable
SELECT
    hypertable_name,
    total_chunks,
    compressed_chunks,
    ROUND(100.0 * compressed_chunks / NULLIF(total_chunks, 0), 1) AS compression_pct,
    pg_size_pretty(before_compression_total_bytes) AS uncompressed_size,
    pg_size_pretty(after_compression_total_bytes) AS compressed_size,
    ROUND(
        before_compression_total_bytes::numeric / NULLIF(after_compression_total_bytes, 0),
        1
    ) AS compression_ratio
FROM timescaledb_information.hypertable_compression_stats;

-- Monitor continuous aggregate refresh status
SELECT
    view_name,
    materialization_hypertable_schema,
    materialization_hypertable_name,
    view_definition
FROM timescaledb_information.continuous_aggregates;
```

---

## Comparison with Other Specialty Approaches

### Why Time-Series over Graph Database?

A graph database (Neo4j, Amazon Neptune) would model the material flow network -- waste generators, haulers, transfer stations, MRFs, landfills, and the material moving between them -- as nodes and edges. This is conceptually elegant for chain-of-custody tracking and regulatory lineage queries ("trace this load of hazardous waste from generator to final disposal"). However:

- The **dominant query pattern** in waste management operations is temporal, not graph traversal. Dispatchers ask "what happened on Route 7 today?", not "find all paths from generator X to landfill Y."
- **Volumetric data** (GPS telemetry, sensor readings, weighbridge transactions) does not benefit from graph storage. These are numeric time-series, not relationship-dense networks.
- Graph databases lack native time-series features (compression, retention policies, continuous aggregates) and native geospatial support at the level PostGIS provides.
- Chain-of-custody lineage is adequately modeled through the relational `weighbridge_transactions` table joined with `route_execution_events`, which together form a temporal audit trail from collection point to disposal facility.

### Why Time-Series over Pure Document Store?

A document database (MongoDB) would store each route execution, weighbridge session, or sensor reading batch as a self-contained document. This offers schema flexibility but:

- Lacks native time-partitioning, compression, and continuous aggregates.
- Multi-document joins (e.g., "show me today's weighbridge totals by material type with current commodity prices") require application-layer aggregation or MongoDB's aggregation pipeline, which is less capable than SQL for complex analytical queries.
- Does not natively support PostGIS-level geospatial queries.

### Why Time-Series over Dedicated IoT Platform?

Purpose-built IoT platforms (InfluxDB, Apache IoTDB) offer excellent time-series performance but:

- Require a separate relational database for customer, billing, compliance, and contract data, creating a polyglot persistence architecture with associated operational complexity.
- Have limited or no geospatial support.
- Use proprietary query languages (Flux for InfluxDB, SQL-like but limited for IoTDB) instead of full SQL.
- Cannot join telemetry data with relational reference data in a single query.

TimescaleDB eliminates the need for a separate IoT database by bringing time-series capabilities into the same PostgreSQL instance that stores all other operational data.

---

## Summary

This model treats waste management as what it fundamentally is: a **continuous physical process that generates time-stamped, geo-located data streams**. By using TimescaleDB hypertables for operational telemetry and PostGIS for spatial operations -- both running as extensions inside a single PostgreSQL instance alongside conventional relational tables -- the architecture delivers purpose-built storage for the highest-volume data streams without introducing a separate database engine, a custom query language, or an ETL pipeline.

The automatic compression (10-20x), continuous aggregates (eliminating the need for a separate analytics pipeline), and declarative retention policies (compliance-appropriate data lifecycle) collectively reduce both infrastructure cost and operational complexity compared to a pure relational approach that must reinvent these capabilities through application code.

For a recycling and waste management platform serving small-to-mid-size operators who cannot afford dedicated data engineering teams, this single-database architecture provides enterprise-grade time-series and geospatial capabilities with the operational simplicity of managing one PostgreSQL instance.
