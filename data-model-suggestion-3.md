# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL + JSONB)

> Project: 467 — Recycling & Waste Management
> Model type: Hybrid normalized relational tables with JSONB columns for flexible/variable data
> Database: PostgreSQL 16+ with PostGIS 3.4+, leveraging native JSONB support

---

## Design Philosophy

Waste management sits at the intersection of rigid regulatory requirements and highly variable operational data. Manifest fields are strictly defined by the EPA or Environment Agency, billing rules follow fixed contractual terms, and weigh ticket measurements are precise numeric values — all of these demand normalized, typed columns with referential integrity. But the same system must also accommodate:

- **Jurisdiction-specific compliance fields** that differ between US RCRA, UK Environment Agency, and EU CSRD/ESRS E5 frameworks, and which change when regulations are amended.
- **Variable telematics payloads** from Geotab, Samsara, and manufacturer-native systems, each with different diagnostic code schemas and sensor readings.
- **ML classification results** with model-specific metadata (confidence scores, bounding boxes, feature vectors) that evolve as models are retrained.
- **Customer-specific billing configurations** that may include custom surcharge rules, special pricing tiers, or contract-specific fields that do not fit a fixed column schema.
- **Hardware-specific weighbridge data** where each scale manufacturer returns different supplementary fields beyond gross/tare weight.

The hybrid model uses **normalized columns for stable, frequently queried, and compliance-critical data** and **JSONB columns for variable, schema-evolving, or integration-specific data**. This delivers the referential integrity of a relational database where it matters most, while eliminating the constant ALTER TABLE migrations that plague a fully normalized model in a multi-jurisdiction, multi-vendor environment.

---

## Schema Definition

### Organizations & Users (Stable Schema)

```sql
CREATE TABLE organizations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    org_type            VARCHAR(50) NOT NULL CHECK (org_type IN (
                            'hauler', 'municipality', 'mrf', 'transfer_station',
                            'landfill', 'broker', 'generator'
                        )),
    parent_org_id       UUID REFERENCES organizations(id),
    -- Stable identifier fields
    tax_id              VARCHAR(50),
    epa_site_id         VARCHAR(20),
    uk_environment_agency_id VARCHAR(20),
    default_currency    CHAR(3) DEFAULT 'USD',
    default_timezone    VARCHAR(50) DEFAULT 'America/New_York',
    -- Structured address
    address_line1       VARCHAR(255),
    address_line2       VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) DEFAULT 'US',
    location            GEOGRAPHY(POINT, 4326),
    -- Flexible: regulatory registrations, certifications, per-jurisdiction IDs
    regulatory_ids      JSONB NOT NULL DEFAULT '{}',
    /*
        Example regulatory_ids:
        {
            "epa_generator_category": "LQG",
            "state_permits": [
                {"state": "TX", "permit_number": "HW-2025-1234", "expiry": "2027-01-15"}
            ],
            "uk_waste_carrier_licence": "CBDU123456",
            "eu_vat_id": "DE123456789",
            "iso_14001_certified": true,
            "iso_14001_expiry": "2026-12-31"
        }
    */
    settings            JSONB NOT NULL DEFAULT '{}',  -- org-specific configuration
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_organizations_type ON organizations(org_type);
CREATE INDEX idx_organizations_epa ON organizations(epa_site_id) WHERE epa_site_id IS NOT NULL;
CREATE INDEX idx_organizations_location ON organizations USING GIST(location);
CREATE INDEX idx_organizations_regulatory ON organizations USING GIN(regulatory_ids);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    email               VARCHAR(255) NOT NULL UNIQUE,
    password_hash       VARCHAR(255),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    phone               VARCHAR(30),
    role                VARCHAR(50) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    -- Flexible: driver credentials, certifications, preferences
    profile_data        JSONB NOT NULL DEFAULT '{}',
    /*
        Example profile_data for a driver:
        {
            "license_number": "DL-12345678",
            "license_class": "CDL-B",
            "license_expiry": "2027-06-30",
            "endorsements": ["hazmat", "tanker"],
            "medical_card_expiry": "2026-11-15",
            "certifications": [
                {"type": "OSHA_40hr", "expiry": "2027-03-01"},
                {"type": "DOT_drug_test", "last_tested": "2026-04-15", "result": "negative"}
            ],
            "preferred_vehicle_types": ["front_loader", "side_loader"],
            "languages": ["en", "es"]
        }
    */
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_profile ON users USING GIN(profile_data);
```

### Customers & Service Agreements (Hybrid)

```sql
CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    customer_number     VARCHAR(50) NOT NULL,
    customer_type       VARCHAR(30) NOT NULL CHECK (customer_type IN (
                            'residential', 'commercial', 'industrial',
                            'municipal', 'roll_off'
                        )),
    name                VARCHAR(255) NOT NULL,
    email               VARCHAR(255),
    phone               VARCHAR(30),
    account_status      VARCHAR(20) DEFAULT 'active',
    payment_terms_days  INTEGER DEFAULT 30,
    -- Structured billing address (queried frequently)
    billing_address     JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "line1": "123 Main St",
            "line2": "Suite 100",
            "city": "Austin",
            "state": "TX",
            "postal_code": "78701",
            "country": "US"
        }
    */
    -- Flexible: customer-specific settings, portal preferences, tax info
    settings            JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "tax_exempt": true,
            "tax_exempt_id": "EX-12345",
            "credit_limit": 50000.00,
            "auto_pay_enabled": true,
            "portal_access": true,
            "notification_preferences": {
                "collection_reminders": true,
                "contamination_alerts": "email",
                "invoice_delivery": "email"
            },
            "sustainability_reporting": true,
            "custom_fields": {
                "department_code": "FAC-01",
                "property_manager": "Jane Smith"
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_customers_number ON customers(organization_id, customer_number);
CREATE INDEX idx_customers_type ON customers(customer_type);
CREATE INDEX idx_customers_settings ON customers USING GIN(settings);

CREATE TABLE service_locations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    location_name       VARCHAR(255),
    address_line1       VARCHAR(255) NOT NULL,
    city                VARCHAR(100) NOT NULL,
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20) NOT NULL,
    country_code        CHAR(2) DEFAULT 'US',
    location            GEOGRAPHY(POINT, 4326) NOT NULL,
    -- Flexible: access details, site-specific requirements
    site_details        JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "access_instructions": "Enter via loading dock B, gate code 4567",
            "time_window": {"start": "06:00", "end": "14:00"},
            "requires_escort": false,
            "dock_height_inches": 48,
            "clearance_restrictions": "Max height 12ft under awning",
            "hazmat_storage_areas": ["Building C, Room 101"],
            "site_contacts": [
                {"name": "Bob Jones", "phone": "555-0123", "role": "facility_manager"}
            ],
            "special_equipment_needed": ["forklift"],
            "photos": ["site_overview.jpg", "dock_access.jpg"]
        }
    */
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_service_locations_customer ON service_locations(customer_id);
CREATE INDEX idx_service_locations_geo ON service_locations USING GIST(location);

CREATE TABLE service_agreements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    agreement_number    VARCHAR(50) NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE,
    billing_frequency   VARCHAR(20) NOT NULL,
    billing_method      VARCHAR(20) NOT NULL,
    status              VARCHAR(20) DEFAULT 'active',
    auto_renew          BOOLEAN DEFAULT false,
    -- Flexible: pricing tiers, contract-specific surcharge rules, SLA terms
    pricing_config      JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "base_rates": [
                {
                    "location_id": "uuid-1",
                    "material_type": "MSW",
                    "container_type": "96gal_cart",
                    "container_count": 1,
                    "frequency": "weekly",
                    "days": [1, 4],
                    "rate": 45.00,
                    "rate_unit": "per_month",
                    "effective_date": "2026-01-01"
                },
                {
                    "location_id": "uuid-1",
                    "material_type": "CARD",
                    "container_type": "2yd_dumpster",
                    "container_count": 1,
                    "frequency": "weekly",
                    "days": [3],
                    "rate": 85.50,
                    "rate_unit": "per_pickup",
                    "effective_date": "2026-01-01"
                }
            ],
            "surcharge_rules": {
                "contamination": {"rate": 75.00, "per": "incident"},
                "overweight_pct_threshold": 15,
                "overweight_rate": 0.05,
                "overweight_unit": "per_lb_over",
                "fuel_surcharge_pct": 8.5,
                "environmental_fee": 3.50,
                "environmental_fee_unit": "per_pickup"
            },
            "volume_discounts": [
                {"min_tons_monthly": 50, "discount_pct": 5},
                {"min_tons_monthly": 100, "discount_pct": 10}
            ],
            "commodity_revenue_share": {
                "enabled": true,
                "share_pct": 25,
                "materials": ["ALU", "CARD", "HDPE"],
                "price_index": "RecyclingMarkets_Southeast"
            },
            "sla": {
                "collection_window_hours": 4,
                "missed_pickup_credit": 25.00,
                "response_time_hours": 24
            }
        }
    */
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agreements_customer ON service_agreements(customer_id);
CREATE INDEX idx_agreements_status ON service_agreements(status);
CREATE INDEX idx_agreements_pricing ON service_agreements USING GIN(pricing_config);
```

### Material Types & Container Assets (Stable + Flexible)

```sql
CREATE TABLE material_types (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(20) NOT NULL UNIQUE,
    name                VARCHAR(100) NOT NULL,
    category            VARCHAR(50) NOT NULL,
    is_hazardous        BOOLEAN DEFAULT false,
    default_unit        VARCHAR(20) DEFAULT 'tons',
    -- Flexible: jurisdiction-specific waste codes, regulatory classifications
    regulatory_codes    JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "epa_rcra_codes": ["D001"],
            "epa_management_methods": ["H010", "H020"],
            "ewc_code": "20 01 01",
            "uk_waste_classification": "non-hazardous",
            "california_waste_codes": ["331"],
            "dot_proper_shipping_name": "Waste flammable liquid, n.o.s.",
            "dot_hazard_class": "3",
            "dot_packing_group": "II",
            "un_number": "UN1993"
        }
    */
    -- Flexible: commodity tracking, density estimates, processing instructions
    properties          JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "commodity_index": "OCC_Southeast",
            "density_kg_per_m3": 150,
            "baling_spec": {"min_density_kg_m3": 400, "wire_count": 5},
            "sorting_instructions": "Remove film and wax-coated board",
            "contamination_threshold_pct": 5,
            "accepted_at_mrf": true,
            "color_sort_required": true
        }
    */
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_material_types_category ON material_types(category);
CREATE INDEX idx_material_types_regulatory ON material_types USING GIN(regulatory_codes);

CREATE TABLE container_types (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(20) NOT NULL UNIQUE,
    name                VARCHAR(100) NOT NULL,
    category            VARCHAR(30) NOT NULL,
    capacity_value      NUMERIC(10,2) NOT NULL,
    capacity_unit       VARCHAR(20) NOT NULL,
    tare_weight_kg      NUMERIC(8,2),
    max_gross_weight_kg NUMERIC(8,2),
    -- Flexible: dimensions, compatible vehicles, RFID specs
    specifications      JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "dimensions": {"length_in": 72, "width_in": 42, "height_in": 44},
            "compatible_vehicle_types": ["front_loader", "side_loader"],
            "lid_type": "hinged",
            "color": "blue",
            "rfid_compatible": true,
            "rfid_frequency": "UHF_860_960MHz",
            "manufacturer": "Rehrig Pacific",
            "model": "Evr-Green Cart 96",
            "warranty_years": 10,
            "material": "HDPE",
            "recycled_content_pct": 30
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE containers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    container_type_id   UUID NOT NULL REFERENCES container_types(id),
    serial_number       VARCHAR(50) UNIQUE,
    rfid_tag            VARCHAR(50),
    barcode             VARCHAR(50),
    current_location_id UUID REFERENCES service_locations(id),
    current_customer_id UUID REFERENCES customers(id),
    status              VARCHAR(20) DEFAULT 'in_service',
    condition           VARCHAR(20) DEFAULT 'good',
    location            GEOGRAPHY(POINT, 4326),
    -- Flexible: maintenance history, sensor data, custom tracking
    metadata            JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "deployed_date": "2025-06-15",
            "last_inspected": "2026-03-01",
            "inspection_result": "good",
            "repairs": [
                {"date": "2026-02-10", "type": "lid_replacement", "cost": 45.00}
            ],
            "iot_sensor": {
                "device_id": "SNS-00123",
                "provider": "Bigbelly",
                "last_fill_level_pct": 72,
                "last_reading_at": "2026-05-25T14:30:00Z",
                "battery_pct": 85
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_containers_org ON containers(organization_id);
CREATE INDEX idx_containers_rfid ON containers(rfid_tag) WHERE rfid_tag IS NOT NULL;
CREATE INDEX idx_containers_location ON containers USING GIST(location);
CREATE INDEX idx_containers_metadata ON containers USING GIN(metadata);
```

### Fleet & Vehicles (Hybrid with Telematics JSONB)

```sql
CREATE TABLE vehicles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    vehicle_number      VARCHAR(30) NOT NULL,
    vin                 VARCHAR(17),
    license_plate       VARCHAR(20),
    vehicle_type        VARCHAR(30) NOT NULL,
    make                VARCHAR(50),
    model               VARCHAR(50),
    year                INTEGER,
    capacity_cubic_yards NUMERIC(8,2),
    max_payload_kg      NUMERIC(10,2),
    fuel_type           VARCHAR(20),
    status              VARCHAR(20) DEFAULT 'active',
    co2_emission_factor NUMERIC(8,4),
    -- Flexible: telematics config, specs, maintenance schedule
    telematics_config   JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "provider": "geotab",
            "device_id": "b1234",
            "serial_number": "G9-ABC-1234",
            "firmware_version": "4.2.1",
            "reporting_interval_seconds": 10,
            "capabilities": ["gps", "engine_data", "fuel_level", "pto", "temperature"],
            "api_credentials_ref": "vault://telematics/geotab/device-b1234"
        }
    */
    vehicle_specs       JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "gvwr_kg": 15000,
            "wheelbase_mm": 4200,
            "turning_radius_m": 12.5,
            "body_manufacturer": "McNeilus",
            "body_model": "Meridian Front Loader",
            "body_serial": "MFR-2025-0456",
            "packer_capacity_cubic_yards": 31,
            "arm_type": "automated_side_loader",
            "camera_count": 4,
            "onboard_scale": {
                "manufacturer": "Loadman",
                "model": "D230",
                "max_capacity_kg": 12000,
                "last_calibration": "2026-04-01"
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_vehicles_number ON vehicles(organization_id, vehicle_number);
CREATE INDEX idx_vehicles_telematics ON vehicles USING GIN(telematics_config);

-- Telematics data points (high volume, partitioned, with flexible payload)
CREATE TABLE vehicle_telemetry (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY,
    vehicle_id          UUID NOT NULL REFERENCES vehicles(id),
    recorded_at         TIMESTAMPTZ NOT NULL,
    location            GEOGRAPHY(POINT, 4326) NOT NULL,
    speed_kmh           NUMERIC(5,1),
    heading             NUMERIC(5,1),
    -- Flexible: provider-specific telemetry data
    telemetry_data      JSONB NOT NULL DEFAULT '{}',
    /*
        Geotab example:
        {
            "odometer_km": 45230.5,
            "engine_hours": 3201.4,
            "fuel_level_pct": 62,
            "engine_rpm": 1200,
            "coolant_temp_c": 88,
            "battery_voltage": 13.8,
            "pto_active": true,
            "diagnostic_codes": [],
            "harsh_events": [],
            "zones": ["yard_depot_1"]
        }

        Samsara example:
        {
            "odometer_meters": 45230500,
            "fuel_percent": 62.0,
            "ambient_air_temp_c": 28,
            "def_level_pct": 45,
            "engine_load_pct": 35,
            "barometric_pressure_kpa": 101.3,
            "gps_accuracy_meters": 2.5,
            "driver_id_detected": "DRV-789"
        }
    */
    PRIMARY KEY (id, recorded_at)
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions
CREATE TABLE vehicle_telemetry_2026_01 PARTITION OF vehicle_telemetry
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional monthly partitions

CREATE INDEX idx_telemetry_vehicle_time ON vehicle_telemetry(vehicle_id, recorded_at DESC);
CREATE INDEX idx_telemetry_location ON vehicle_telemetry USING GIST(location);

-- Vehicle inspections with flexible checklist
CREATE TABLE vehicle_inspections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id          UUID NOT NULL REFERENCES vehicles(id),
    inspector_id        UUID NOT NULL REFERENCES users(id),
    inspection_type     VARCHAR(30) NOT NULL,
    inspection_date     TIMESTAMPTZ NOT NULL DEFAULT now(),
    odometer_km         NUMERIC(10,1),
    passed              BOOLEAN NOT NULL,
    -- Flexible: inspection checklist items vary by vehicle type and jurisdiction
    checklist_results   JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "items": [
                {"item": "brakes", "category": "safety", "result": "pass", "notes": ""},
                {"item": "lights", "category": "safety", "result": "pass", "notes": ""},
                {"item": "tires", "category": "safety", "result": "fail", "notes": "LF tire at 3/32 tread depth"},
                {"item": "hydraulic_system", "category": "operational", "result": "pass", "notes": ""},
                {"item": "packer_blade", "category": "operational", "result": "pass", "notes": ""},
                {"item": "arm_mechanism", "category": "operational", "result": "pass", "notes": "Minor play in pivot"}
            ],
            "defects_requiring_repair": ["tires"],
            "photos": ["lf_tire_wear.jpg"],
            "dot_compliance": true,
            "inspector_certification": "DOT-456-2026"
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_vehicle ON vehicle_inspections(vehicle_id, inspection_date DESC);
```

### Route Planning & Collection (Hybrid)

```sql
CREATE TABLE route_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    route_code          VARCHAR(30) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    service_day         INTEGER CHECK (service_day BETWEEN 0 AND 6),
    material_type_id    UUID REFERENCES material_types(id),
    estimated_stops     INTEGER,
    estimated_duration_minutes INTEGER,
    route_geometry      GEOGRAPHY(LINESTRING, 4326),
    service_area        GEOGRAPHY(POLYGON, 4326),
    -- Flexible: optimization parameters, constraints
    optimization_config JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "max_route_duration_hours": 8,
            "max_distance_km": 150,
            "vehicle_type_required": "side_loader",
            "avoid_zones": ["school_zone_during_hours"],
            "time_windows_strict": false,
            "lunch_break": {"after_stop": 100, "duration_minutes": 30},
            "depot_return_required": true,
            "road_restrictions": ["no_left_turns_arterial"],
            "seasonal_adjustments": {
                "holiday_skip_dates": ["2026-12-25", "2027-01-01"],
                "summer_early_start": {"months": [6,7,8], "start_time": "05:30"}
            }
        }
    */
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_route_templates_org ON route_templates(organization_id);
CREATE INDEX idx_route_templates_area ON route_templates USING GIST(service_area);

CREATE TABLE planned_routes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_template_id   UUID REFERENCES route_templates(id),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    route_date          DATE NOT NULL,
    vehicle_id          UUID REFERENCES vehicles(id),
    driver_id           UUID REFERENCES users(id),
    status              VARCHAR(20) DEFAULT 'planned',
    planned_start_time  TIMESTAMPTZ,
    actual_start_time   TIMESTAMPTZ,
    actual_end_time     TIMESTAMPTZ,
    total_stops         INTEGER,
    completed_stops     INTEGER DEFAULT 0,
    missed_stops        INTEGER DEFAULT 0,
    total_distance_km   NUMERIC(10,2),
    total_weight_kg     NUMERIC(12,2),
    actual_route_geometry GEOGRAPHY(LINESTRING, 4326),
    -- Flexible: AI optimization results, performance metrics
    route_analytics     JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "optimization_algorithm": "reinforcement_learning_v2",
            "optimization_score": 0.87,
            "estimated_vs_actual": {
                "duration_minutes": {"estimated": 420, "actual": 445},
                "distance_km": {"estimated": 85.2, "actual": 88.7},
                "fuel_liters": {"estimated": 42, "actual": 44.1}
            },
            "traffic_conditions": "moderate",
            "weather": "clear",
            "co2_emissions_kg": 115.6,
            "idle_time_minutes": 35,
            "stops_per_hour": 22.5,
            "efficiency_score": 0.82
        }
    */
    dispatcher_notes    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_planned_routes_date ON planned_routes(route_date);
CREATE INDEX idx_planned_routes_driver ON planned_routes(driver_id, route_date);

CREATE TABLE collection_stops (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    planned_route_id    UUID NOT NULL REFERENCES planned_routes(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    container_id        UUID REFERENCES containers(id),
    sequence_number     INTEGER NOT NULL,
    status              VARCHAR(20) DEFAULT 'pending',
    material_type_id    UUID REFERENCES material_types(id),
    -- Core measurement fields (stable, frequently aggregated)
    scheduled_time      TIMESTAMPTZ,
    arrival_time        TIMESTAMPTZ,
    departure_time      TIMESTAMPTZ,
    arrival_location    GEOGRAPHY(POINT, 4326),
    estimated_weight_kg NUMERIC(10,2),
    actual_weight_kg    NUMERIC(10,2),
    weight_source       VARCHAR(20),
    -- Flexible: exception details, driver observations, ML results
    stop_details        JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "container_fullness": 0.85,
            "container_condition": "good",
            "lid_open": false,
            "extra_bags_count": 2,
            "contamination": {
                "detected": true,
                "method": "ml_classification",
                "type": "food_waste_in_recycling",
                "severity": "moderate",
                "confidence": 0.91,
                "model_version": "contam-detect-v3.2",
                "bounding_boxes": [
                    {"x": 120, "y": 80, "w": 200, "h": 150, "label": "food_waste", "conf": 0.93}
                ],
                "photo_urls": ["stop_789_photo1.jpg", "stop_789_photo2.jpg"]
            },
            "skip_reason": null,
            "driver_notes": "Customer left note requesting larger container",
            "signature": "base64_encoded_signature_data",
            "synced_from_offline": false,
            "sync_timestamp": "2026-05-26T08:35:00Z",
            "rfid_read": {
                "tag_id": "RFID-00456",
                "read_strength": -45,
                "read_time_ms": 12
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stops_route ON collection_stops(planned_route_id, sequence_number);
CREATE INDEX idx_stops_customer ON collection_stops(customer_id);
CREATE INDEX idx_stops_date ON collection_stops(created_at);
CREATE INDEX idx_stops_details ON collection_stops USING GIN(stop_details);
-- Partial index for contamination filtering
CREATE INDEX idx_stops_contaminated ON collection_stops USING GIN((stop_details->'contamination'))
    WHERE stop_details->'contamination' IS NOT NULL
    AND (stop_details->'contamination'->>'detected')::boolean = true;
```

### Weighbridge & Material Tracking (Hybrid)

```sql
CREATE TABLE weighbridges (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    facility_name       VARCHAR(255) NOT NULL,
    location            GEOGRAPHY(POINT, 4326),
    max_capacity_kg     NUMERIC(10,2),
    precision_kg        NUMERIC(6,3),
    -- Flexible: hardware config, calibration history
    hardware_config     JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "manufacturer": "Avery Weigh-Tronix",
            "model": "BridgeMont ZM303",
            "serial_number": "AWX-2025-0789",
            "connection": {
                "protocol": "tcp_ip_modbus",
                "ip_address": "192.168.1.50",
                "port": 502,
                "register_address": 40001,
                "data_format": "32bit_float_big_endian"
            },
            "calibration_history": [
                {"date": "2026-04-01", "technician": "ScaleTech Inc.", "certificate": "CAL-2026-0456", "result": "pass"},
                {"date": "2025-10-15", "technician": "ScaleTech Inc.", "certificate": "CAL-2025-1234", "result": "pass"}
            ],
            "last_calibration": "2026-04-01",
            "next_calibration_due": "2026-10-01",
            "legal_for_trade": true
        }
    */
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    -- Core weight measurements (stable, numeric, frequently aggregated)
    gross_weight_kg     NUMERIC(12,2) NOT NULL,
    tare_weight_kg      NUMERIC(12,2) NOT NULL,
    net_weight_kg       NUMERIC(12,2) GENERATED ALWAYS AS (gross_weight_kg - tare_weight_kg) STORED,
    gross_weigh_time    TIMESTAMPTZ NOT NULL,
    tare_weigh_time     TIMESTAMPTZ,
    quality_grade       VARCHAR(20),
    -- Flexible: raw hardware data, quality details, discrepancy analysis
    ticket_details      JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "raw_scale_readings": {
                "gross": [15234.5, 15234.8, 15234.6],
                "tare": [5120.1, 5120.0, 5120.2],
                "stability_check": "stable",
                "motion_detected": false
            },
            "source_location": "MRF Processing Line 2",
            "destination": "Greenville Landfill",
            "quality_assessment": {
                "grade": "standard",
                "moisture_pct": 8.5,
                "contamination_pct": 2.1,
                "notes": "Minor plastic contamination in cardboard bale"
            },
            "discrepancy": {
                "flagged": true,
                "expected_range_kg": {"min": 8000, "max": 12000},
                "actual_net_kg": 10114.4,
                "deviation_pct": 0,
                "auto_detected": true,
                "resolved": true,
                "resolution": "Within acceptable range after review"
            },
            "photos": ["ticket_12345_gross.jpg"],
            "operator_notes": "Load appeared clean, minor moisture",
            "linked_collection_stops": ["stop-uuid-1", "stop-uuid-2"]
        }
    */
    manifest_id         UUID REFERENCES compliance_manifests(id),
    voided              BOOLEAN DEFAULT false,
    voided_reason       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_weigh_tickets_number ON weigh_tickets(organization_id, ticket_number);
CREATE INDEX idx_weigh_tickets_date ON weigh_tickets(gross_weigh_time);
CREATE INDEX idx_weigh_tickets_material ON weigh_tickets(material_type_id);
CREATE INDEX idx_weigh_tickets_details ON weigh_tickets USING GIN(ticket_details);

-- Commodity market prices
CREATE TABLE commodity_prices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    material_type_id    UUID NOT NULL REFERENCES material_types(id),
    region              VARCHAR(50) NOT NULL,
    price_per_ton       NUMERIC(10,2) NOT NULL,
    currency            CHAR(3) DEFAULT 'USD',
    effective_date      DATE NOT NULL,
    source              VARCHAR(100),
    -- Flexible: index details, price history metadata
    price_metadata      JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "index_name": "OCC Southeast Domestic",
            "grade_specification": "PS-11 OCC",
            "price_range": {"low": 85.00, "high": 95.00, "mid": 90.00},
            "trend": "rising",
            "week_over_week_change_pct": 2.5,
            "market_notes": "Strong demand from domestic mills"
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commodity_prices_material ON commodity_prices(material_type_id, effective_date DESC);
```

### Compliance & Regulatory (Hybrid with Jurisdiction-Specific JSONB)

```sql
CREATE TABLE jurisdictions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    jurisdiction_level  VARCHAR(20) NOT NULL,
    parent_jurisdiction_id UUID REFERENCES jurisdictions(id),
    country_code        CHAR(2) NOT NULL,
    state_code          VARCHAR(10),
    geographic_boundary GEOGRAPHY(MULTIPOLYGON, 4326),
    -- Flexible: jurisdiction-specific regulatory parameters
    regulatory_config   JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "framework": "epa_rcra",
            "diversion_targets": [
                {"year": 2026, "target_pct": 50},
                {"year": 2030, "target_pct": 75}
            ],
            "reporting": {
                "frequency": "quarterly",
                "due_day_of_quarter": 30,
                "submission_portal": "https://rcrainfo.epa.gov",
                "required_reports": ["diversion_rate", "tonnage_by_stream", "hazardous_manifest_summary"]
            },
            "manifest_requirements": {
                "manifest_type": "epa_emanifest",
                "required_for": ["hazardous"],
                "state_waste_codes_required": true,
                "additional_state_codes": ["TX-HW-01", "TX-HW-02"]
            },
            "container_regulations": {
                "max_storage_days_hazardous": 90,
                "labeling_required": true,
                "color_coding": {"recyclable": "blue", "organic": "green", "waste": "black"}
            },
            "fees": {
                "manifest_fee_per_shipment": 20.00,
                "annual_generator_fee": 500.00,
                "tipping_fee_per_ton": 45.00
            }
        }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdictions_boundary ON jurisdictions USING GIST(geographic_boundary);
CREATE INDEX idx_jurisdictions_config ON jurisdictions USING GIN(regulatory_config);

CREATE TABLE compliance_manifests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    manifest_type       VARCHAR(30) NOT NULL,
    manifest_tracking_number VARCHAR(30),
    status              VARCHAR(30) DEFAULT 'draft',
    jurisdiction_id     UUID REFERENCES jurisdictions(id),
    -- Core chain-of-custody fields (stable, audit-critical)
    generator_org_id    UUID REFERENCES organizations(id),
    generator_epa_id    VARCHAR(20),
    facility_org_id     UUID REFERENCES organizations(id),
    facility_epa_id     VARCHAR(20),
    shipped_date        DATE,
    received_date       DATE,
    total_weight_kg     NUMERIC(12,2),
    total_containers    INTEGER,
    -- Flexible: jurisdiction-specific manifest data
    manifest_data       JSONB NOT NULL DEFAULT '{}',
    /*
        EPA e-Manifest example:
        {
            "generator": {
                "name": "ABC Chemical Corp",
                "site_address": {"line1": "100 Industrial Blvd", "city": "Houston", "state": "TX", "zip": "77001"},
                "mailing_address": {"line1": "PO Box 1234", "city": "Houston", "state": "TX", "zip": "77001"},
                "contact": {"name": "John Smith", "phone": "713-555-0100", "email": "jsmith@abc.com"},
                "emergency_phone": "800-555-0199",
                "generator_category": "LQG",
                "signed_at": "2026-05-20T10:00:00Z",
                "signature_type": "electronic"
            },
            "transporters": [
                {
                    "order": 1,
                    "org_id": "uuid-transporter",
                    "epa_id": "TXD000000002",
                    "name": "SafeHaul Trucking",
                    "signed_at": "2026-05-20T10:30:00Z"
                }
            ],
            "facility": {
                "name": "CleanTech TSDF",
                "epa_id": "TXD000000003",
                "address": {"line1": "500 Treatment Rd", "city": "Baytown", "state": "TX"},
                "received_at": "2026-05-20T14:00:00Z",
                "received_weight_kg": 4520.5,
                "signed_by": "uuid-facility-mgr"
            },
            "waste_lines": [
                {
                    "line_number": 1,
                    "waste_description": "Waste flammable liquid, n.o.s.",
                    "dot_hazardous": true,
                    "dot_info": {"id": "UN1993", "shipping_name": "Flammable liquid, n.o.s.", "class": "3", "packing_group": "II"},
                    "epa_waste_codes": ["D001", "D018"],
                    "state_waste_codes": ["TX-HW-01"],
                    "containers": {"count": 10, "type": "DM", "description": "55-gal drums"},
                    "quantity": {"value": 2500, "unit": "K", "description": "Kilograms"},
                    "management_method": "H040",
                    "pcb": false,
                    "handling_instructions": "Keep away from heat sources"
                }
            ],
            "special_handling": "Temperature controlled transport required",
            "epa_submission": {
                "submission_id": "EPA-2026-MTN-123456789ABC",
                "submitted_at": "2026-05-20T15:00:00Z",
                "response_status": "accepted"
            }
        }

        UK Waste Transfer Note example:
        {
            "transferor": {
                "name": "London Waste Services Ltd",
                "address": "45 Green Lane, London E1 4AB",
                "carrier_licence": "CBDU123456",
                "sic_code": "38.11"
            },
            "transferee": {
                "name": "Southern Recycling PLC",
                "address": "Unit 7, Greenfield Industrial Estate, Kent ME2 3BD",
                "permit_number": "EPR/AB1234CD",
                "sic_code": "38.32"
            },
            "waste_details": {
                "ewc_code": "20 01 01",
                "description": "Paper and cardboard",
                "quantity_tonnes": 18.5,
                "hazardous": false,
                "container_type": "Loose in skip"
            },
            "duty_of_care_declaration": true,
            "transfer_date": "2026-05-20",
            "season_ticket": false,
            "season_ticket_expiry": null
        }
    */
    pdf_document_url    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_manifests_org ON compliance_manifests(organization_id);
CREATE INDEX idx_manifests_mtn ON compliance_manifests(manifest_tracking_number);
CREATE INDEX idx_manifests_status ON compliance_manifests(status);
CREATE INDEX idx_manifests_data ON compliance_manifests USING GIN(manifest_data);
```

### Billing & Invoicing (Stable + Flexible)

```sql
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
    status              VARCHAR(20) DEFAULT 'draft',
    issued_date         DATE,
    due_date            DATE,
    paid_date           DATE,
    -- Flexible: accounting integration details, payment info
    billing_metadata    JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "payment_method": "ach",
            "payment_reference": "ACH-2026-05-30-12345",
            "quickbooks": {
                "invoice_id": "QB-INV-5678",
                "synced_at": "2026-05-30T10:00:00Z",
                "sync_status": "success"
            },
            "applied_discounts": [
                {"type": "volume", "description": "50+ tons monthly discount", "amount": -125.00}
            ],
            "commodity_revenue_credits": [
                {"material": "ALU", "tons": 2.5, "price_per_ton": 1200.00, "share_pct": 25, "credit": -750.00}
            ],
            "tax_breakdown": {
                "state_tax_rate": 0.0625,
                "county_tax_rate": 0.01,
                "state_tax": 312.50,
                "county_tax": 50.00
            }
        }
    */
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_invoices_number ON invoices(organization_id, invoice_number);
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(status);

CREATE TABLE invoice_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id),
    line_number         INTEGER NOT NULL,
    description         TEXT NOT NULL,
    line_type           VARCHAR(30) NOT NULL,
    material_type_id    UUID REFERENCES material_types(id),
    quantity            NUMERIC(12,4),
    unit                VARCHAR(20),
    rate                NUMERIC(10,4),
    amount              NUMERIC(12,2) NOT NULL,
    tax_rate            NUMERIC(5,4) DEFAULT 0,
    tax_amount          NUMERIC(10,2) DEFAULT 0,
    -- Flexible: source traceability, adjustments
    line_metadata       JSONB NOT NULL DEFAULT '{}',
    /*
        {
            "source_stops": ["stop-uuid-1", "stop-uuid-2"],
            "source_weigh_tickets": ["ticket-uuid-1"],
            "agreement_line_id": "agreement-line-uuid",
            "commodity_price_at_time": 90.00,
            "rate_calculation": {
                "method": "weight_based",
                "base_rate": 45.00,
                "fuel_surcharge_pct": 8.5,
                "environmental_fee": 3.50,
                "total_rate": 52.33
            }
        }
    */
    UNIQUE(invoice_id, line_number)
);

CREATE INDEX idx_invoice_lines_invoice ON invoice_lines(invoice_id);
```

### Audit Trail (Compact with JSONB)

```sql
CREATE TABLE audit_log (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID NOT NULL,
    action              VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_by          UUID REFERENCES users(id),
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    changes             JSONB NOT NULL,  -- old/new values in a single document
    /*
        {
            "old": {"status": "draft", "total_weight_kg": null},
            "new": {"status": "in_transit", "total_weight_kg": 4520.5},
            "context": {
                "ip_address": "10.0.1.50",
                "user_agent": "WasteApp-Mobile/3.2.1 iOS",
                "session_id": "sess-abc-123"
            }
        }
    */
    ip_address          INET
);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_changed_at ON audit_log(changed_at);
CREATE INDEX idx_audit_changes ON audit_log USING GIN(changes);
```

---

## Query Patterns Enabled by JSONB

### Find all stops with ML-detected contamination above a confidence threshold

```sql
SELECT cs.id, cs.customer_id, c.name AS customer_name,
       cs.stop_details->'contamination'->>'type' AS contamination_type,
       (cs.stop_details->'contamination'->>'confidence')::numeric AS confidence,
       cs.stop_details->'contamination'->>'model_version' AS model_version
FROM collection_stops cs
JOIN customers c ON c.id = cs.customer_id
WHERE cs.stop_details @> '{"contamination": {"detected": true}}'
  AND (cs.stop_details->'contamination'->>'confidence')::numeric > 0.85
  AND cs.created_at >= '2026-05-01'
ORDER BY confidence DESC;
```

### Query vehicles with specific telematics capability

```sql
SELECT v.vehicle_number, v.vehicle_type,
       v.telematics_config->>'provider' AS telematics_provider,
       v.vehicle_specs->'onboard_scale'->>'manufacturer' AS scale_manufacturer
FROM vehicles v
WHERE v.telematics_config->'capabilities' ? 'pto'
  AND v.vehicle_specs->'onboard_scale' IS NOT NULL;
```

### Find pricing rules for a specific customer and material

```sql
SELECT sa.agreement_number,
       rate_line->>'material_type' AS material,
       rate_line->>'rate' AS rate,
       rate_line->>'rate_unit' AS unit,
       sa.pricing_config->'surcharge_rules' AS surcharges
FROM service_agreements sa,
     jsonb_array_elements(sa.pricing_config->'base_rates') AS rate_line
WHERE sa.customer_id = 'customer-uuid'
  AND sa.status = 'active'
  AND rate_line->>'material_type' = 'CARD';
```

---

## Pros and Cons

### Pros

1. **Single database engine for everything.** PostgreSQL handles both the relational and document workloads. No separate document database, no synchronization layer, no additional operational burden. This is a substantial advantage for small operators who need simplicity.

2. **Schema evolution without migrations for variable data.** When a new telematics provider adds a custom field, or a jurisdiction introduces a new compliance requirement, the JSONB column accommodates it immediately. No ALTER TABLE, no deployment, no downtime.

3. **Referential integrity where it matters.** Foreign keys on customer_id, vehicle_id, material_type_id, and manifest_id enforce data consistency for the core domain relationships. The JSONB flexibility exists within the boundaries of the relational structure, not as a replacement for it.

4. **GIN indexes on JSONB enable efficient queries.** Queries like "find all stops with contamination detected" or "find vehicles with onboard scales" use GIN indexes and perform well even across large datasets. The `@>` containment operator is optimized for this pattern.

5. **Self-documenting through examples.** Each JSONB column includes inline documentation (the comments in the schema above) showing the expected structure. Combined with application-level JSON Schema validation, this provides practical schema enforcement without database-level rigidity.

6. **Gradual normalization path.** If a JSONB field proves to be queried constantly and needs stronger constraints, it can be promoted to a dedicated column with a standard migration. The reverse is also true: rarely-used columns can be folded into JSONB to simplify the schema.

7. **Excellent fit for multi-jurisdiction compliance.** The same `compliance_manifests` table stores EPA e-Manifest data and UK Waste Transfer Notes using the same manifest_data JSONB column, with jurisdiction-specific structure. Adding support for EU CSRD or Canadian provincial manifests requires no schema change.

### Cons

1. **Data integrity in JSONB is application-enforced.** If a bug writes `"severity": "extreme"` instead of `"severity": "severe"` into the contamination JSONB, the database will not catch it. Application-level JSON Schema validation mitigates this, but it is less reliable than database CHECK constraints.

2. **Reporting complexity.** Aggregating data buried in JSONB requires `jsonb_array_elements`, casts, and extraction operators that are slower and harder to read than simple column references. BI tools (Metabase, Tableau) have limited support for querying inside JSONB columns.

3. **Storage overhead.** JSONB stores field names with every row. A JSONB column storing `{"contamination": {"detected": true, "type": "food_waste"}}` on a million rows repeats those key strings a million times. This is less storage-efficient than normalized columns.

4. **No native JSONB foreign keys.** A UUID stored inside JSONB (`"source_stops": ["stop-uuid-1"]`) cannot have a foreign key constraint. If the referenced stop is deleted, the JSONB reference becomes a dangling pointer. Application code must enforce these relationships.

5. **JSONB indexing has limits.** While GIN indexes support containment and existence queries, they do not efficiently support range queries on values inside JSONB (e.g., "find all tickets where raw scale reading > 10000"). These queries require functional indexes or extraction into columns.

6. **Developer discipline required.** The flexibility of JSONB is also its risk. Without clear documentation and code review practices, JSONB columns can become dumping grounds for unstructured data that is difficult to query, validate, or migrate later.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with PostGIS 3.4+ |
| **JSONB validation** | Application-level JSON Schema validation (ajv for Node.js, jsonschema for Python) |
| **JSONB documentation** | Maintain JSON Schema definitions alongside the SQL schema in version control |
| **ORM** | Prisma (Node.js), SQLAlchemy (Python), or Diesel (Rust) — all support JSONB columns |
| **BI/Reporting** | Metabase or Apache Superset with custom SQL for JSONB extraction |
| **Migration tool** | Flyway or golang-migrate (JSONB content migrations handled in application code) |
| **Monitoring** | Track JSONB column sizes and GIN index bloat with pg_stat_user_tables |

---

## Migration & Scaling Considerations

### JSONB Content Evolution Strategy
1. **Version marker in JSONB.** Include `"schema_version": 1` in every JSONB document. Application code reads the version and applies transformations as needed.
2. **Lazy migration.** When reading an old-format JSONB document, transform it to the new format and write it back. Over time, all documents converge to the latest version.
3. **Batch migration scripts.** For critical fields, run UPDATE queries to transform JSONB structure across all rows during maintenance windows.
4. **Promotion to columns.** When a JSONB field is queried in >80% of table queries, promote it to a dedicated column with ALTER TABLE.

### Scaling Path
- **Phase 1 (1-10 trucks):** Single PostgreSQL instance handles everything. JSONB columns keep the schema simple and evolution fast.
- **Phase 2 (10-100 trucks):** Add read replica for reporting. Create materialized views that extract and aggregate JSONB data for dashboards.
- **Phase 3 (100+ trucks):** Consider Citus for horizontal sharding by organization_id. JSONB columns shard transparently. Move vehicle_telemetry to TimescaleDB hypertable.

### Integration with External Systems
- **Weighbridge hardware:** Raw scale readings stored in `ticket_details` JSONB, accommodating any hardware protocol without schema changes.
- **Telematics APIs:** Provider-specific payloads stored in `telemetry_data` JSONB. A normalization layer in the application extracts common fields (location, speed, odometer) while preserving raw data.
- **EPA e-Manifest API:** The `manifest_data` JSONB can store the raw EPA JSON schema payload alongside the normalized manifest fields, enabling round-trip fidelity with the federal API.
- **Accounting systems:** Invoice export metadata in `billing_metadata` JSONB stores QuickBooks/Xero/Sage-specific IDs and sync status without adding columns per accounting system.
