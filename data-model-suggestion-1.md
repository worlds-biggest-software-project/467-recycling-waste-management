# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL + PostGIS)

> Project: 467 — Recycling & Waste Management
> Model type: Fully normalized relational (3NF) with spatial extensions
> Database: PostgreSQL 16+ with PostGIS 3.4+ and pgRouting

---

## Design Philosophy

This model applies strict third-normal-form (3NF) normalization to every domain entity. Waste management is inherently a **transactional, multi-party, compliance-heavy** domain: loads must be weighed accurately, manifests must satisfy regulatory audits, and billing must reconcile to the gram. A normalized relational model delivers referential integrity, ACID guarantees, and audit-ready query paths that align directly with these requirements.

PostGIS extensions add native geospatial support for route geometry, stop locations, GPS traces, and service-area polygons. pgRouting provides shortest-path and vehicle-routing-problem (VRP) algorithms directly inside the database.

---

## Schema Definition

### Core Domain: Organizations & Users

```sql
-- Tenancy and multi-org support
CREATE TABLE organizations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    org_type            VARCHAR(50) NOT NULL CHECK (org_type IN (
                            'hauler', 'municipality', 'mrf', 'transfer_station',
                            'landfill', 'broker', 'generator'
                        )),
    parent_org_id       UUID REFERENCES organizations(id),
    tax_id              VARCHAR(50),
    epa_site_id         VARCHAR(20),       -- EPA handler ID (e.g., "TXD000000001")
    uk_environment_agency_id VARCHAR(20),
    default_currency    CHAR(3) DEFAULT 'USD',
    default_timezone    VARCHAR(50) DEFAULT 'America/New_York',
    address_line1       VARCHAR(255),
    address_line2       VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) DEFAULT 'US',
    location            GEOGRAPHY(POINT, 4326),  -- PostGIS point
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_organizations_type ON organizations(org_type);
CREATE INDEX idx_organizations_epa ON organizations(epa_site_id) WHERE epa_site_id IS NOT NULL;
CREATE INDEX idx_organizations_location ON organizations USING GIST(location);

-- User accounts and authentication
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    email               VARCHAR(255) NOT NULL UNIQUE,
    password_hash       VARCHAR(255),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    phone               VARCHAR(30),
    role                VARCHAR(50) NOT NULL CHECK (role IN (
                            'admin', 'dispatcher', 'driver', 'supervisor',
                            'billing_clerk', 'compliance_officer', 'customer_portal',
                            'weighbridge_operator', 'mechanic'
                        )),
    driver_license_number   VARCHAR(50),
    driver_license_class    VARCHAR(10),
    driver_license_expiry   DATE,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_role ON users(role);
```

### Customer & Contract Management

```sql
-- Customers (residential, commercial, industrial, municipal accounts)
CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),  -- owning hauler/org
    customer_number     VARCHAR(50) NOT NULL,
    customer_type       VARCHAR(30) NOT NULL CHECK (customer_type IN (
                            'residential', 'commercial', 'industrial',
                            'municipal', 'roll_off'
                        )),
    name                VARCHAR(255) NOT NULL,
    billing_name        VARCHAR(255),
    email               VARCHAR(255),
    phone               VARCHAR(30),
    billing_address_line1   VARCHAR(255),
    billing_address_line2   VARCHAR(255),
    billing_city            VARCHAR(100),
    billing_state           VARCHAR(100),
    billing_postal_code     VARCHAR(20),
    billing_country_code    CHAR(2) DEFAULT 'US',
    tax_exempt          BOOLEAN DEFAULT false,
    tax_exempt_id       VARCHAR(50),
    payment_terms_days  INTEGER DEFAULT 30,
    credit_limit        NUMERIC(12,2),
    account_status      VARCHAR(20) DEFAULT 'active' CHECK (account_status IN (
                            'active', 'suspended', 'closed', 'pending'
                        )),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_customers_number ON customers(organization_id, customer_number);

-- Service locations (a customer can have multiple pickup points)
CREATE TABLE service_locations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    location_name       VARCHAR(255),
    address_line1       VARCHAR(255) NOT NULL,
    address_line2       VARCHAR(255),
    city                VARCHAR(100) NOT NULL,
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20) NOT NULL,
    country_code        CHAR(2) DEFAULT 'US',
    location            GEOGRAPHY(POINT, 4326) NOT NULL,  -- GPS coordinates
    access_instructions TEXT,                               -- gate codes, dock info
    time_window_start   TIME,                               -- earliest pickup time
    time_window_end     TIME,                               -- latest pickup time
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_service_locations_customer ON service_locations(customer_id);
CREATE INDEX idx_service_locations_geo ON service_locations USING GIST(location);

-- Service agreements / contracts
CREATE TABLE service_agreements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    agreement_number    VARCHAR(50) NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE,
    billing_frequency   VARCHAR(20) NOT NULL CHECK (billing_frequency IN (
                            'per_pickup', 'weekly', 'biweekly', 'monthly',
                            'quarterly', 'annually'
                        )),
    billing_method      VARCHAR(20) NOT NULL CHECK (billing_method IN (
                            'weight_based', 'volume_based', 'flat_rate', 'hybrid'
                        )),
    auto_renew          BOOLEAN DEFAULT false,
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN (
                            'draft', 'active', 'expired', 'terminated', 'suspended'
                        )),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Service agreement line items (one per material stream per location)
CREATE TABLE service_agreement_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agreement_id        UUID NOT NULL REFERENCES service_agreements(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    material_type_id    UUID NOT NULL REFERENCES material_types(id),
    container_type_id   UUID REFERENCES container_types(id),
    container_count     INTEGER DEFAULT 1,
    collection_frequency VARCHAR(30) NOT NULL CHECK (collection_frequency IN (
                            'daily', 'twice_weekly', 'weekly', 'biweekly',
                            'monthly', 'on_call'
                        )),
    collection_days     INTEGER[] DEFAULT '{}',  -- 0=Sun, 1=Mon, ..., 6=Sat
    rate_per_unit       NUMERIC(10,4),           -- price per ton/cubic yard/pickup
    rate_unit           VARCHAR(20) CHECK (rate_unit IN (
                            'per_ton', 'per_cubic_yard', 'per_pickup', 'per_container',
                            'per_month'
                        )),
    minimum_charge      NUMERIC(10,2),
    contamination_surcharge_rate NUMERIC(10,2),
    overweight_surcharge_rate    NUMERIC(10,2),
    effective_date      DATE NOT NULL,
    end_date            DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agreement_lines_agreement ON service_agreement_lines(agreement_id);
```

### Material Types & Container Assets

```sql
-- Canonical material type reference table
CREATE TABLE material_types (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(20) NOT NULL UNIQUE,   -- e.g., 'MSW', 'CARD', 'ALU', 'GLASS'
    name                VARCHAR(100) NOT NULL,
    category            VARCHAR(50) NOT NULL CHECK (category IN (
                            'recyclable', 'organic', 'hazardous', 'construction_demolition',
                            'electronic', 'medical', 'general_waste', 'special'
                        )),
    is_hazardous        BOOLEAN DEFAULT false,
    epa_waste_codes     VARCHAR(10)[],     -- RCRA waste codes, e.g., {'D001', 'D002'}
    ewc_code            VARCHAR(10),       -- European Waste Catalogue code
    default_unit        VARCHAR(20) DEFAULT 'tons' CHECK (default_unit IN (
                            'tons', 'kilograms', 'pounds', 'cubic_yards', 'gallons'
                        )),
    commodity_index     VARCHAR(50),       -- commodity price index reference
    density_factor      NUMERIC(8,4),      -- estimated kg per cubic meter
    requires_manifest   BOOLEAN DEFAULT false,
    description         TEXT,
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Container types (bins, carts, skips, roll-offs, compactors)
CREATE TABLE container_types (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(20) NOT NULL UNIQUE,
    name                VARCHAR(100) NOT NULL,
    category            VARCHAR(30) NOT NULL CHECK (category IN (
                            'cart', 'bin', 'dumpster', 'roll_off', 'compactor',
                            'skip', 'tank', 'drum'
                        )),
    capacity_value      NUMERIC(10,2) NOT NULL,
    capacity_unit       VARCHAR(20) NOT NULL CHECK (capacity_unit IN (
                            'gallons', 'cubic_yards', 'liters', 'cubic_meters'
                        )),
    tare_weight_kg      NUMERIC(8,2),
    max_gross_weight_kg NUMERIC(8,2),
    compatible_material_types UUID[],  -- references material_types.id
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Individual container assets (physical inventory)
CREATE TABLE containers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    container_type_id   UUID NOT NULL REFERENCES container_types(id),
    serial_number       VARCHAR(50) UNIQUE,
    rfid_tag            VARCHAR(50),
    barcode             VARCHAR(50),
    current_location_id UUID REFERENCES service_locations(id),
    current_customer_id UUID REFERENCES customers(id),
    status              VARCHAR(20) DEFAULT 'in_service' CHECK (status IN (
                            'in_service', 'in_yard', 'in_repair', 'retired',
                            'lost', 'in_transit'
                        )),
    condition           VARCHAR(20) DEFAULT 'good' CHECK (condition IN (
                            'new', 'good', 'fair', 'damaged', 'condemned'
                        )),
    last_inspected_at   TIMESTAMPTZ,
    deployed_at         DATE,
    location            GEOGRAPHY(POINT, 4326),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_containers_org ON containers(organization_id);
CREATE INDEX idx_containers_customer ON containers(current_customer_id);
CREATE INDEX idx_containers_rfid ON containers(rfid_tag) WHERE rfid_tag IS NOT NULL;
CREATE INDEX idx_containers_location ON containers USING GIST(location);
```

### Fleet & Vehicle Management

```sql
-- Vehicles in the fleet
CREATE TABLE vehicles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    vehicle_number      VARCHAR(30) NOT NULL,
    vin                 VARCHAR(17),
    license_plate       VARCHAR(20),
    vehicle_type        VARCHAR(30) NOT NULL CHECK (vehicle_type IN (
                            'rear_loader', 'front_loader', 'side_loader',
                            'roll_off', 'grapple', 'tanker', 'flatbed',
                            'service_truck', 'supervisor'
                        )),
    make                VARCHAR(50),
    model               VARCHAR(50),
    year                INTEGER,
    capacity_cubic_yards NUMERIC(8,2),
    max_payload_kg      NUMERIC(10,2),
    fuel_type           VARCHAR(20) CHECK (fuel_type IN (
                            'diesel', 'cng', 'electric', 'hybrid', 'gasoline'
                        )),
    telematics_provider VARCHAR(30),   -- 'geotab', 'samsara', 'manufacturer'
    telematics_device_id VARCHAR(50),
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN (
                            'active', 'maintenance', 'out_of_service', 'retired'
                        )),
    last_odometer_km    NUMERIC(10,1),
    last_inspection_date DATE,
    next_inspection_due  DATE,
    insurance_expiry    DATE,
    registration_expiry DATE,
    co2_emission_factor NUMERIC(8,4),  -- kg CO2 per km
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_vehicles_number ON vehicles(organization_id, vehicle_number);

-- Pre-trip inspection records
CREATE TABLE vehicle_inspections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicles(id),
    inspector_id        UUID NOT NULL REFERENCES users(id),
    inspection_type     VARCHAR(30) NOT NULL CHECK (inspection_type IN (
                            'pre_trip', 'post_trip', 'scheduled', 'dot_annual'
                        )),
    inspection_date     TIMESTAMPTZ NOT NULL DEFAULT now(),
    odometer_km         NUMERIC(10,1),
    passed              BOOLEAN NOT NULL,
    defects_found       TEXT[],
    corrective_actions  TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_vehicle ON vehicle_inspections(vehicle_id, inspection_date DESC);

-- Vehicle maintenance records
CREATE TABLE vehicle_maintenance (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicles(id),
    maintenance_type    VARCHAR(30) NOT NULL CHECK (maintenance_type IN (
                            'preventive', 'corrective', 'emergency', 'recall'
                        )),
    description         TEXT NOT NULL,
    vendor              VARCHAR(255),
    cost                NUMERIC(10,2),
    started_at          TIMESTAMPTZ NOT NULL,
    completed_at        TIMESTAMPTZ,
    odometer_km         NUMERIC(10,1),
    next_due_date       DATE,
    next_due_odometer   NUMERIC(10,1),
    parts_replaced      TEXT[],
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_maintenance_vehicle ON vehicle_maintenance(vehicle_id);
```

### Route Planning & Collection Operations

```sql
-- Route templates (recurring route definitions)
CREATE TABLE route_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    route_code          VARCHAR(30) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    service_day         INTEGER CHECK (service_day BETWEEN 0 AND 6),  -- 0=Sun
    material_type_id    UUID REFERENCES material_types(id),
    estimated_stops     INTEGER,
    estimated_duration_minutes INTEGER,
    route_geometry      GEOGRAPHY(LINESTRING, 4326),  -- planned route path
    service_area        GEOGRAPHY(POLYGON, 4326),     -- geographic coverage
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_route_templates_org ON route_templates(organization_id);
CREATE INDEX idx_route_templates_area ON route_templates USING GIST(service_area);

-- Stops within a route template
CREATE TABLE route_template_stops (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_template_id   UUID NOT NULL REFERENCES route_templates(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    sequence_number     INTEGER NOT NULL,
    estimated_arrival   TIME,
    estimated_service_minutes INTEGER DEFAULT 5,
    special_instructions TEXT,
    UNIQUE(route_template_id, sequence_number)
);

-- Daily planned routes (instances of templates or ad-hoc)
CREATE TABLE planned_routes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_template_id   UUID REFERENCES route_templates(id),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    route_date          DATE NOT NULL,
    vehicle_id          UUID REFERENCES vehicles(id),
    driver_id           UUID REFERENCES users(id),
    status              VARCHAR(20) DEFAULT 'planned' CHECK (status IN (
                            'planned', 'dispatched', 'in_progress',
                            'completed', 'cancelled', 'partial'
                        )),
    planned_start_time  TIMESTAMPTZ,
    actual_start_time   TIMESTAMPTZ,
    actual_end_time     TIMESTAMPTZ,
    total_stops         INTEGER,
    completed_stops     INTEGER DEFAULT 0,
    missed_stops        INTEGER DEFAULT 0,
    total_distance_km   NUMERIC(10,2),
    total_weight_kg     NUMERIC(12,2),
    actual_route_geometry GEOGRAPHY(LINESTRING, 4326),  -- GPS trace
    dispatcher_notes    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_planned_routes_date ON planned_routes(route_date);
CREATE INDEX idx_planned_routes_driver ON planned_routes(driver_id, route_date);
CREATE INDEX idx_planned_routes_vehicle ON planned_routes(vehicle_id, route_date);

-- Individual collection stops within a planned route
CREATE TABLE collection_stops (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    planned_route_id    UUID NOT NULL REFERENCES planned_routes(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    container_id        UUID REFERENCES containers(id),
    sequence_number     INTEGER NOT NULL,
    status              VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
                            'pending', 'in_progress', 'completed', 'skipped',
                            'missed', 'extra', 'rescheduled'
                        )),
    scheduled_time      TIMESTAMPTZ,
    arrival_time        TIMESTAMPTZ,
    departure_time      TIMESTAMPTZ,
    arrival_location    GEOGRAPHY(POINT, 4326),  -- actual GPS at arrival
    material_type_id    UUID REFERENCES material_types(id),
    estimated_weight_kg NUMERIC(10,2),
    actual_weight_kg    NUMERIC(10,2),
    weight_source       VARCHAR(20) CHECK (weight_source IN (
                            'estimated', 'onboard_scale', 'weighbridge', 'rfid'
                        )),
    container_fullness  NUMERIC(3,2),  -- 0.00 to 1.00
    skip_reason         VARCHAR(50),
    exception_type      VARCHAR(30) CHECK (exception_type IN (
                            'none', 'contamination', 'overweight', 'blocked_access',
                            'not_out', 'damaged_container', 'extra_waste', 'other'
                        )) DEFAULT 'none',
    exception_notes     TEXT,
    driver_notes        TEXT,
    signature_data      TEXT,          -- base64 encoded signature
    synced_at           TIMESTAMPTZ,   -- for offline sync tracking
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stops_route ON collection_stops(planned_route_id, sequence_number);
CREATE INDEX idx_stops_customer ON collection_stops(customer_id);
CREATE INDEX idx_stops_date ON collection_stops(created_at);
```

### Contamination Tracking

```sql
-- Contamination incidents recorded at collection stops
CREATE TABLE contamination_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_stop_id  UUID NOT NULL REFERENCES collection_stops(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    detected_by         UUID NOT NULL REFERENCES users(id),   -- driver or inspector
    detection_method    VARCHAR(30) NOT NULL CHECK (detection_method IN (
                            'visual', 'ml_classification', 'manual_inspection',
                            'weighbridge_anomaly'
                        )),
    contamination_type  VARCHAR(50) NOT NULL,   -- 'food_waste_in_recycling', 'hazardous', etc.
    severity            VARCHAR(20) NOT NULL CHECK (severity IN (
                            'minor', 'moderate', 'severe', 'rejected'
                        )),
    description         TEXT,
    ml_confidence_score NUMERIC(5,4),  -- 0.0000 to 1.0000
    photo_urls          TEXT[],
    customer_notified   BOOLEAN DEFAULT false,
    notification_sent_at TIMESTAMPTZ,
    surcharge_applied   BOOLEAN DEFAULT false,
    surcharge_amount    NUMERIC(10,2),
    resolution_status   VARCHAR(20) DEFAULT 'open' CHECK (resolution_status IN (
                            'open', 'notified', 'acknowledged', 'resolved', 'disputed'
                        )),
    resolved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contamination_customer ON contamination_records(customer_id);
CREATE INDEX idx_contamination_date ON contamination_records(created_at);
```

### Weighbridge & Material Tracking

```sql
-- Weighbridge / scale installations
CREATE TABLE weighbridges (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    facility_name       VARCHAR(255) NOT NULL,
    location            GEOGRAPHY(POINT, 4326),
    scale_manufacturer  VARCHAR(100),
    scale_model         VARCHAR(100),
    connection_protocol VARCHAR(20) CHECK (connection_protocol IN (
                            'serial_rs232', 'tcp_ip_modbus', 'tcp_ip_custom',
                            'usb', 'api'
                        )),
    max_capacity_kg     NUMERIC(10,2),
    precision_kg        NUMERIC(6,3),
    last_calibration_date DATE,
    next_calibration_due  DATE,
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Weigh tickets (inbound and outbound at facilities)
CREATE TABLE weigh_tickets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_number       VARCHAR(30) NOT NULL,
    weighbridge_id      UUID NOT NULL REFERENCES weighbridges(id),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    vehicle_id          UUID REFERENCES vehicles(id),
    driver_id           UUID REFERENCES users(id),
    customer_id         UUID REFERENCES customers(id),
    direction           VARCHAR(10) NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    material_type_id    UUID NOT NULL REFERENCES material_types(id),
    gross_weight_kg     NUMERIC(12,2) NOT NULL,
    tare_weight_kg      NUMERIC(12,2) NOT NULL,
    net_weight_kg       NUMERIC(12,2) GENERATED ALWAYS AS (gross_weight_kg - tare_weight_kg) STORED,
    gross_weigh_time    TIMESTAMPTZ NOT NULL,
    tare_weigh_time     TIMESTAMPTZ,
    quality_grade       VARCHAR(20) CHECK (quality_grade IN (
                            'premium', 'standard', 'below_standard', 'contaminated', 'rejected'
                        )),
    source_location     VARCHAR(255),
    destination         VARCHAR(255),
    manifest_id         UUID REFERENCES compliance_manifests(id),
    discrepancy_flag    BOOLEAN DEFAULT false,
    discrepancy_reason  TEXT,
    operator_id         UUID REFERENCES users(id),  -- weighbridge operator
    notes               TEXT,
    voided              BOOLEAN DEFAULT false,
    voided_reason       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_weigh_tickets_number ON weigh_tickets(organization_id, ticket_number);
CREATE INDEX idx_weigh_tickets_date ON weigh_tickets(gross_weigh_time);
CREATE INDEX idx_weigh_tickets_vehicle ON weigh_tickets(vehicle_id);
CREATE INDEX idx_weigh_tickets_material ON weigh_tickets(material_type_id);

-- Material commodity pricing (for dynamic billing and reporting)
CREATE TABLE commodity_prices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    material_type_id    UUID NOT NULL REFERENCES material_types(id),
    region              VARCHAR(50) NOT NULL,
    price_per_ton       NUMERIC(10,2) NOT NULL,
    currency            CHAR(3) DEFAULT 'USD',
    effective_date      DATE NOT NULL,
    source              VARCHAR(100),   -- 'RecyclingMarkets.net', 'OCC Index', etc.
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commodity_prices_material ON commodity_prices(material_type_id, effective_date DESC);
```

### Compliance & Regulatory

```sql
-- Jurisdiction-specific compliance rule sets
CREATE TABLE jurisdictions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    jurisdiction_level  VARCHAR(20) NOT NULL CHECK (jurisdiction_level IN (
                            'federal', 'state', 'county', 'city', 'country'
                        )),
    parent_jurisdiction_id UUID REFERENCES jurisdictions(id),
    country_code        CHAR(2) NOT NULL,
    state_code          VARCHAR(10),
    geographic_boundary GEOGRAPHY(MULTIPOLYGON, 4326),
    regulatory_framework VARCHAR(30) CHECK (regulatory_framework IN (
                            'epa_rcra', 'uk_environment_agency', 'eu_csrd',
                            'state_specific', 'provincial'
                        )),
    diversion_target_pct NUMERIC(5,2),    -- mandated diversion rate
    reporting_frequency  VARCHAR(20),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdictions_boundary ON jurisdictions USING GIST(geographic_boundary);

-- Compliance rules per jurisdiction
CREATE TABLE compliance_rules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id     UUID NOT NULL REFERENCES jurisdictions(id),
    rule_code           VARCHAR(50) NOT NULL,
    rule_name           VARCHAR(255) NOT NULL,
    description         TEXT,
    applies_to_material_types UUID[],  -- which material types this rule covers
    requires_manifest   BOOLEAN DEFAULT false,
    manifest_type       VARCHAR(30),   -- 'epa_emanifest', 'uk_waste_transfer_note', etc.
    requires_photo      BOOLEAN DEFAULT false,
    max_storage_days    INTEGER,       -- max days waste can be stored on-site
    reporting_template  TEXT,          -- template for regulatory report format
    effective_date      DATE NOT NULL,
    expiry_date         DATE,
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_rules_jurisdiction ON compliance_rules(jurisdiction_id);

-- Compliance manifests / waste transfer notes
CREATE TABLE compliance_manifests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    manifest_type       VARCHAR(30) NOT NULL CHECK (manifest_type IN (
                            'epa_emanifest', 'uk_waste_transfer_note',
                            'uk_hazardous_consignment_note', 'state_manifest',
                            'international_movement', 'internal'
                        )),
    manifest_tracking_number VARCHAR(30),  -- EPA MTN format: 9 digits + 3 letters
    status              VARCHAR(30) DEFAULT 'draft' CHECK (status IN (
                            'draft', 'pending', 'scheduled', 'in_transit',
                            'received', 'corrected', 'rejected', 'voided'
                        )),
    jurisdiction_id     UUID REFERENCES jurisdictions(id),

    -- Generator (source)
    generator_org_id    UUID REFERENCES organizations(id),
    generator_site_name VARCHAR(255),
    generator_epa_id    VARCHAR(20),
    generator_address   TEXT,
    generator_contact   VARCHAR(255),
    generator_phone     VARCHAR(30),
    generator_signed_at TIMESTAMPTZ,
    generator_signed_by UUID REFERENCES users(id),

    -- Transporter(s) handled via manifest_transporters junction table

    -- Designated Facility (destination)
    facility_org_id     UUID REFERENCES organizations(id),
    facility_name       VARCHAR(255),
    facility_epa_id     VARCHAR(20),
    facility_address    TEXT,
    facility_received_at TIMESTAMPTZ,
    facility_signed_by  UUID REFERENCES users(id),

    -- Shipment details
    shipped_date        DATE,
    received_date       DATE,
    total_weight_kg     NUMERIC(12,2),
    total_containers    INTEGER,
    special_handling    TEXT,
    emergency_info      TEXT,

    -- Discrepancy / rejection
    discrepancy_type    VARCHAR(30),
    discrepancy_description TEXT,
    rejection_reason    TEXT,

    -- EPA e-Manifest specific
    epa_submission_id   VARCHAR(50),
    epa_submitted_at    TIMESTAMPTZ,
    epa_response_status VARCHAR(30),

    pdf_document_url    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_manifests_org ON compliance_manifests(organization_id);
CREATE INDEX idx_manifests_mtn ON compliance_manifests(manifest_tracking_number);
CREATE INDEX idx_manifests_status ON compliance_manifests(status);

-- Manifest waste line items
CREATE TABLE manifest_waste_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manifest_id         UUID NOT NULL REFERENCES compliance_manifests(id),
    line_number         INTEGER NOT NULL,
    material_type_id    UUID NOT NULL REFERENCES material_types(id),
    waste_description   TEXT NOT NULL,
    dot_hazardous       BOOLEAN DEFAULT false,
    dot_id_number       VARCHAR(20),
    dot_shipping_name   VARCHAR(255),
    dot_hazard_class    VARCHAR(10),
    epa_waste_codes     VARCHAR(10)[],  -- federal RCRA codes
    state_waste_codes   VARCHAR(10)[],
    container_count     INTEGER NOT NULL,
    container_type      VARCHAR(30),     -- 'drum', 'tank', 'box', etc.
    quantity            NUMERIC(12,4) NOT NULL,
    unit_of_measure     VARCHAR(20) NOT NULL,
    management_method   VARCHAR(10),     -- EPA management method code
    pcb                 BOOLEAN DEFAULT false,
    handling_instructions TEXT,
    UNIQUE(manifest_id, line_number)
);

CREATE INDEX idx_manifest_lines_manifest ON manifest_waste_lines(manifest_id);

-- Manifest transporters (ordered list)
CREATE TABLE manifest_transporters (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manifest_id         UUID NOT NULL REFERENCES compliance_manifests(id),
    transporter_org_id  UUID NOT NULL REFERENCES organizations(id),
    transport_order     INTEGER NOT NULL,
    transporter_epa_id  VARCHAR(20),
    signed_at           TIMESTAMPTZ,
    signed_by           UUID REFERENCES users(id),
    UNIQUE(manifest_id, transport_order)
);
```

### Billing & Invoicing

```sql
-- Generated invoices
CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    invoice_number      VARCHAR(30) NOT NULL,
    billing_period_start DATE NOT NULL,
    billing_period_end  DATE NOT NULL,
    subtotal            NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_amount          NUMERIC(12,2) NOT NULL DEFAULT 0,
    surcharges          NUMERIC(12,2) NOT NULL DEFAULT 0,
    credits             NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount        NUMERIC(12,2) NOT NULL DEFAULT 0,
    currency            CHAR(3) DEFAULT 'USD',
    status              VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
                            'draft', 'pending', 'sent', 'paid',
                            'partial', 'overdue', 'voided', 'disputed'
                        )),
    issued_date         DATE,
    due_date            DATE,
    paid_date           DATE,
    payment_method      VARCHAR(30),
    external_invoice_id VARCHAR(50),  -- QuickBooks/accounting system ID
    exported_at         TIMESTAMPTZ,  -- when synced to accounting system
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_invoices_number ON invoices(organization_id, invoice_number);
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(status);

-- Invoice line items
CREATE TABLE invoice_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id),
    line_number         INTEGER NOT NULL,
    description         TEXT NOT NULL,
    line_type           VARCHAR(30) NOT NULL CHECK (line_type IN (
                            'collection', 'disposal', 'recycling_credit',
                            'container_rental', 'delivery', 'removal',
                            'surcharge_contamination', 'surcharge_overweight',
                            'surcharge_extra_pickup', 'fuel_surcharge',
                            'environmental_fee', 'credit', 'adjustment'
                        )),
    service_agreement_line_id UUID REFERENCES service_agreement_lines(id),
    material_type_id    UUID REFERENCES material_types(id),
    quantity            NUMERIC(12,4),
    unit                VARCHAR(20),
    rate                NUMERIC(10,4),
    amount              NUMERIC(12,2) NOT NULL,
    tax_rate            NUMERIC(5,4) DEFAULT 0,
    tax_amount          NUMERIC(10,2) DEFAULT 0,
    collection_stop_id  UUID REFERENCES collection_stops(id),
    weigh_ticket_id     UUID REFERENCES weigh_tickets(id),
    notes               TEXT,
    UNIQUE(invoice_id, line_number)
);

CREATE INDEX idx_invoice_lines_invoice ON invoice_lines(invoice_id);

-- Billing exception queue (AI-assisted review)
CREATE TABLE billing_exceptions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    exception_type      VARCHAR(30) NOT NULL CHECK (exception_type IN (
                            'overweight', 'underweight', 'contamination',
                            'extra_pickup', 'missed_pickup', 'price_variance',
                            'duplicate_ticket'
                        )),
    source_type         VARCHAR(20) NOT NULL,  -- 'collection_stop' or 'weigh_ticket'
    source_id           UUID NOT NULL,
    customer_id         UUID NOT NULL REFERENCES customers(id),
    detected_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    expected_value      NUMERIC(12,4),
    actual_value        NUMERIC(12,4),
    variance_pct        NUMERIC(5,2),
    ai_recommendation   TEXT,
    ai_confidence       NUMERIC(5,4),
    review_status       VARCHAR(20) DEFAULT 'pending' CHECK (review_status IN (
                            'pending', 'approved', 'rejected', 'adjusted', 'auto_approved'
                        )),
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    adjustment_amount   NUMERIC(10,2),
    invoice_id          UUID REFERENCES invoices(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_billing_exceptions_status ON billing_exceptions(review_status);
CREATE INDEX idx_billing_exceptions_customer ON billing_exceptions(customer_id);
```

### Analytics & Sustainability

```sql
-- Diversion rate tracking (aggregated periodically)
CREATE TABLE diversion_metrics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    period_type         VARCHAR(10) NOT NULL CHECK (period_type IN (
                            'daily', 'weekly', 'monthly', 'quarterly', 'annual'
                        )),
    jurisdiction_id     UUID REFERENCES jurisdictions(id),
    customer_id         UUID REFERENCES customers(id),  -- NULL = org-wide
    total_collected_kg  NUMERIC(14,2) NOT NULL DEFAULT 0,
    landfill_kg         NUMERIC(14,2) NOT NULL DEFAULT 0,
    recycled_kg         NUMERIC(14,2) NOT NULL DEFAULT 0,
    composted_kg        NUMERIC(14,2) NOT NULL DEFAULT 0,
    incinerated_kg      NUMERIC(14,2) NOT NULL DEFAULT 0,
    reused_kg           NUMERIC(14,2) NOT NULL DEFAULT 0,
    diversion_rate_pct  NUMERIC(5,2) GENERATED ALWAYS AS (
                            CASE WHEN total_collected_kg > 0
                                THEN ((recycled_kg + composted_kg + reused_kg) / total_collected_kg) * 100
                                ELSE 0
                            END
                        ) STORED,
    contamination_rate_pct NUMERIC(5,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_diversion_org_period ON diversion_metrics(organization_id, period_start);

-- Carbon footprint per route
CREATE TABLE carbon_emissions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    planned_route_id    UUID REFERENCES planned_routes(id),
    vehicle_id          UUID NOT NULL REFERENCES vehicles(id),
    route_date          DATE NOT NULL,
    distance_km         NUMERIC(10,2) NOT NULL,
    fuel_consumed_liters NUMERIC(8,2),
    co2_emissions_kg    NUMERIC(10,2) NOT NULL,
    calculation_method  VARCHAR(30) NOT NULL CHECK (calculation_method IN (
                            'odometer_factor', 'fuel_actual', 'telematics', 'estimated'
                        )),
    emission_factor_source VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carbon_vehicle ON carbon_emissions(vehicle_id, route_date);

-- GPS telemetry data (high-volume, consider partitioning)
CREATE TABLE vehicle_telemetry (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY,
    vehicle_id          UUID NOT NULL REFERENCES vehicles(id),
    recorded_at         TIMESTAMPTZ NOT NULL,
    location            GEOGRAPHY(POINT, 4326) NOT NULL,
    speed_kmh           NUMERIC(5,1),
    heading             NUMERIC(5,1),
    odometer_km         NUMERIC(10,1),
    engine_hours        NUMERIC(8,1),
    fuel_level_pct      NUMERIC(5,2),
    engine_status       VARCHAR(10),  -- 'running', 'idle', 'off'
    diagnostic_codes    VARCHAR(10)[],
    PRIMARY KEY (id, recorded_at)
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions
CREATE TABLE vehicle_telemetry_2026_01 PARTITION OF vehicle_telemetry
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE vehicle_telemetry_2026_02 PARTITION OF vehicle_telemetry
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional monthly partitions created by automated job

CREATE INDEX idx_telemetry_vehicle_time ON vehicle_telemetry(vehicle_id, recorded_at DESC);
CREATE INDEX idx_telemetry_location ON vehicle_telemetry USING GIST(location);
```

### Audit Trail

```sql
-- Generic audit log for compliance-critical operations
CREATE TABLE audit_log (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID NOT NULL,
    action              VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_by          UUID REFERENCES users(id),
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    old_values          JSONB,
    new_values          JSONB,
    ip_address          INET,
    user_agent          TEXT
);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_changed_at ON audit_log(changed_at);
```

---

## Entity Relationship Summary

```
organizations ──┬── users
                ├── customers ── service_locations
                ├── vehicles ──┬── vehicle_inspections
                │              ├── vehicle_maintenance
                │              └── vehicle_telemetry (partitioned)
                ├── containers
                ├── weighbridges ── weigh_tickets
                ├── route_templates ── route_template_stops
                ├── planned_routes ── collection_stops ── contamination_records
                ├── compliance_manifests ──┬── manifest_waste_lines
                │                         └── manifest_transporters
                ├── invoices ── invoice_lines
                ├── billing_exceptions
                ├── diversion_metrics
                └── carbon_emissions

service_agreements ── service_agreement_lines
jurisdictions ── compliance_rules
material_types (referenced by many)
container_types (referenced by containers, agreement_lines)
commodity_prices (by material_type)
```

---

## Pros and Cons

### Pros

1. **Regulatory audit readiness.** Every compliance-critical entity (manifests, weigh tickets, billing) has full referential integrity and an audit trail. Inspectors can trace any manifest to its waste lines, generator, transporters, and receiving facility through clean foreign-key joins.

2. **Transactional accuracy.** ACID transactions ensure that a weigh ticket, its associated billing exception, and the resulting invoice line are all consistent. No orphaned records, no partial writes.

3. **PostGIS/pgRouting integration.** Geospatial queries (find stops within a polygon, compute route distances, identify which jurisdiction covers a GPS point) run inside the database engine with spatial indexes, avoiding round-trips to external services.

4. **Mature ecosystem.** PostgreSQL has decades of tooling for backup, replication, monitoring, and migration. Drivers exist for every programming language. The operational knowledge base is vast.

5. **Cost efficiency.** Fully open source. No per-node or per-query licensing. Self-hosted or managed (RDS, Cloud SQL, Supabase).

6. **Strong typing catches errors early.** CHECK constraints, ENUM-style validation, and generated columns (like net_weight_kg) prevent invalid data at the database level, which is essential when integrating with unreliable weighbridge hardware.

### Cons

1. **Schema rigidity.** Adding a new material attribute, a jurisdiction-specific compliance field, or a telematics provider's custom diagnostic code requires an ALTER TABLE migration. In a domain where regulatory requirements shift by jurisdiction, this creates ongoing migration overhead.

2. **Telemetry volume stress.** A fleet of 50 trucks reporting GPS every 10 seconds generates ~432,000 rows per day. While partitioning helps, PostgreSQL is not purpose-built for high-frequency time-series ingestion. Performance degrades as partitions accumulate.

3. **Complex query patterns.** Generating a single invoice requires joining across service_agreement_lines, collection_stops, weigh_tickets, contamination_records, and commodity_prices. These multi-join queries can become slow without careful indexing and query optimization.

4. **No native event replay.** If the business needs to answer "what happened to manifest X over time?" the audit_log table provides a basic history, but it lacks the event-replay semantics of a true event-sourced system. Reconstructing past state requires scanning audit records.

5. **Offline sync complexity.** The normalized schema does not inherently solve the offline-first mobile requirement. Conflict resolution when a driver syncs 200 collection stops after hours of offline operation requires application-level logic, not just database inserts.

6. **Reporting on high-cardinality data.** Diversion metrics across thousands of customers, dozens of material types, and multiple jurisdictions require pre-aggregation (the diversion_metrics table) because real-time analytical queries over raw operational data are expensive.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with PostGIS 3.4+ and pgRouting 3.6+ |
| **Connection pooling** | PgBouncer or Supavisor for high-concurrency mobile app connections |
| **Schema migrations** | Flyway or golang-migrate for versioned, auditable schema changes |
| **Spatial indexing** | GiST indexes on all GEOGRAPHY columns; SP-GiST for point-only columns |
| **Partitioning** | Range partitioning by month for vehicle_telemetry and audit_log |
| **Full-text search** | PostgreSQL tsvector for searching customer names, addresses, notes |
| **Backup** | pg_basebackup with WAL archiving for point-in-time recovery |
| **Replication** | Streaming replication with read replicas for reporting workloads |
| **Monitoring** | pg_stat_statements, pgBadger, or Datadog PostgreSQL integration |

---

## Migration & Scaling Considerations

### Initial Deployment (1-10 trucks)
- Single PostgreSQL instance (4 vCPU, 16 GB RAM, 500 GB SSD) handles all workloads
- No partitioning needed except for vehicle_telemetry
- Estimated database size: 5-20 GB in first year

### Growth Phase (10-100 trucks)
- Add read replica for reporting/analytics queries
- Partition weigh_tickets and collection_stops by month
- Introduce PgBouncer for mobile connection management
- Implement materialized views for diversion dashboards
- Estimated database size: 50-200 GB per year

### Scale Phase (100+ trucks, multi-region)
- Consider Citus extension for horizontal sharding by organization_id
- Move vehicle_telemetry to TimescaleDB hypertable (still PostgreSQL-compatible)
- Implement logical replication for cross-region deployment
- Archive historical data (>2 years) to cold storage with partman
- Estimated database size: 500 GB - 2 TB per year

### Data Retention Strategy
- **Operational data** (routes, stops, tickets): retain online for 2 years, archive to S3/GCS
- **Compliance data** (manifests, waste lines): retain online for 7 years (RCRA requirement) or jurisdiction-specific period
- **Telemetry data**: retain raw data for 90 days, aggregated data for 2 years
- **Audit log**: retain for compliance period (typically 7 years), then archive
- **Financial data** (invoices): retain for tax retention period (typically 7 years)

### Migration Path from Legacy Systems
1. Map legacy data to the normalized schema using staging tables
2. Use PostgreSQL COPY command for bulk loading historical data
3. Validate referential integrity post-migration with constraint checks
4. Run parallel operation (old + new) for one billing cycle before cutover
5. Export migration verification reports comparing totals between systems
