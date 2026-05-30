# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: 467 — Recycling & Waste Management
> Model type: Event Sourcing with Command Query Responsibility Segregation (CQRS)
> Technologies: PostgreSQL (event store), Apache Kafka (event bus), Redis (read model cache), PostgreSQL (read model projections)

---

## Design Philosophy

Waste management operations produce a natural stream of domain events: a truck departs the yard, arrives at a stop, records a weight, detects contamination, completes a pickup. Compliance auditing demands a complete, tamper-evident history of every state change — when a manifest was created, who signed it, when weights were corrected, why a load was rejected. An event-sourced architecture captures every fact as an immutable event, making the system its own audit trail by construction rather than by afterthought.

CQRS separates the write path (accepting commands that produce events) from the read path (projections optimized for queries). This separation allows the collection operations write model to handle high-throughput mobile device syncs while the billing and compliance read models are optimized for their specific query patterns. The architecture also supports replaying events to rebuild read models, correct projections, or generate new reports retroactively — a powerful capability when regulatory requirements change.

---

## Event Store Schema

The event store is the single source of truth. All state is derived from events.

```sql
-- Core event store table (append-only, immutable)
CREATE TABLE event_store (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id           UUID NOT NULL,          -- aggregate root ID
    stream_type         VARCHAR(50) NOT NULL,    -- aggregate type name
    event_type          VARCHAR(100) NOT NULL,   -- fully qualified event name
    event_version       INTEGER NOT NULL,        -- position in stream (monotonically increasing)
    event_data          JSONB NOT NULL,           -- event payload
    event_metadata      JSONB NOT NULL DEFAULT '{}', -- correlation IDs, causation, user info
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID,                    -- user who triggered the command
    schema_version      INTEGER NOT NULL DEFAULT 1,  -- for event upcasting
    UNIQUE(stream_id, event_version)             -- optimistic concurrency control
);

-- Append-only enforcement: no updates or deletes allowed
-- Enforced via database permissions (REVOKE UPDATE, DELETE on event_store)

-- Indexes for efficient stream reading and global ordering
CREATE INDEX idx_event_store_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_store_type ON event_store(stream_type, event_type);
CREATE INDEX idx_event_store_created ON event_store(created_at);
CREATE INDEX idx_event_store_correlation ON event_store((event_metadata->>'correlation_id'))
    WHERE event_metadata->>'correlation_id' IS NOT NULL;

-- Partition by month for long-term storage efficiency
-- (events are immutable, so old partitions can be compressed or archived)

-- Snapshot store for aggregate rehydration optimization
CREATE TABLE event_snapshots (
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(50) NOT NULL,
    snapshot_version    INTEGER NOT NULL,        -- event_version at snapshot time
    snapshot_data       JSONB NOT NULL,           -- serialized aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Projection checkpoints (track which events each projection has consumed)
CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_event_id       UUID NOT NULL REFERENCES event_store(event_id),
    last_event_version  BIGINT NOT NULL,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status              VARCHAR(20) DEFAULT 'running' CHECK (status IN (
                            'running', 'paused', 'rebuilding', 'error'
                        )),
    error_message       TEXT
);

-- Idempotency tracking for command handlers
CREATE TABLE processed_commands (
    command_id          UUID PRIMARY KEY,
    stream_id           UUID NOT NULL,
    command_type        VARCHAR(100) NOT NULL,
    processed_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    result_event_ids    UUID[] NOT NULL
);

CREATE INDEX idx_processed_commands_stream ON processed_commands(stream_id);
```

---

## Aggregate Roots & Event Types

### 1. Collection Route Aggregate

The route aggregate manages the lifecycle of a daily collection route from planning through completion.

```
Stream Type: CollectionRoute
Stream ID: {route_id}
```

**Events:**

```typescript
// Route lifecycle events
RouteCreated {
    route_id: UUID,
    organization_id: UUID,
    route_template_id: UUID | null,
    route_date: Date,
    vehicle_id: UUID,
    driver_id: UUID,
    planned_stops: [{
        stop_id: UUID,
        sequence: number,
        service_location_id: UUID,
        customer_id: UUID,
        material_type_id: UUID,
        estimated_time: Timestamp,
        container_id: UUID | null
    }],
    planned_start_time: Timestamp
}

RouteDispatched {
    route_id: UUID,
    dispatched_by: UUID,
    dispatched_at: Timestamp
}

RouteStarted {
    route_id: UUID,
    actual_start_time: Timestamp,
    driver_location: { lat: number, lng: number },
    odometer_km: number
}

StopArrived {
    route_id: UUID,
    stop_id: UUID,
    arrived_at: Timestamp,
    arrival_location: { lat: number, lng: number },
    sequence_number: number
}

StopCompleted {
    route_id: UUID,
    stop_id: UUID,
    completed_at: Timestamp,
    weight_kg: number | null,
    weight_source: 'estimated' | 'onboard_scale' | 'rfid',
    container_fullness: number,       // 0.0 to 1.0
    material_type_id: UUID,
    driver_notes: string | null,
    photo_urls: string[],
    synced_from_offline: boolean
}

StopSkipped {
    route_id: UUID,
    stop_id: UUID,
    skipped_at: Timestamp,
    reason: 'not_out' | 'blocked_access' | 'damaged_container' | 'customer_request' | 'safety',
    driver_notes: string,
    photo_urls: string[]
}

StopResequenced {
    route_id: UUID,
    stop_id: UUID,
    old_sequence: number,
    new_sequence: number,
    reason: string,
    resequenced_by: UUID     // driver or dispatcher
}

ExtraStopAdded {
    route_id: UUID,
    stop_id: UUID,
    service_location_id: UUID,
    customer_id: UUID,
    added_at: Timestamp,
    added_by: UUID,
    reason: string
}

RouteCompleted {
    route_id: UUID,
    completed_at: Timestamp,
    total_stops_completed: number,
    total_stops_skipped: number,
    total_weight_kg: number,
    total_distance_km: number,
    final_odometer_km: number,
    route_geometry: GeoJSON
}

RouteCancelled {
    route_id: UUID,
    cancelled_at: Timestamp,
    cancelled_by: UUID,
    reason: string,
    stops_completed_before_cancel: number
}
```

### 2. Weigh Ticket Aggregate

```
Stream Type: WeighTicket
Stream ID: {ticket_id}
```

**Events:**

```typescript
GrossWeighRecorded {
    ticket_id: UUID,
    ticket_number: string,
    weighbridge_id: UUID,
    vehicle_id: UUID,
    driver_id: UUID,
    customer_id: UUID | null,
    direction: 'inbound' | 'outbound',
    gross_weight_kg: number,
    weighed_at: Timestamp,
    operator_id: UUID,
    auto_captured: boolean        // true if from hardware integration
}

TareWeighRecorded {
    ticket_id: UUID,
    tare_weight_kg: number,
    weighed_at: Timestamp,
    operator_id: UUID
}

MaterialClassified {
    ticket_id: UUID,
    material_type_id: UUID,
    quality_grade: 'premium' | 'standard' | 'below_standard' | 'contaminated' | 'rejected',
    classified_by: UUID
}

TicketFinalized {
    ticket_id: UUID,
    net_weight_kg: number,
    finalized_at: Timestamp,
    finalized_by: UUID
}

WeightDiscrepancyDetected {
    ticket_id: UUID,
    expected_weight_range_min: number,
    expected_weight_range_max: number,
    actual_weight_kg: number,
    deviation_pct: number,
    auto_detected: boolean
}

WeightCorrected {
    ticket_id: UUID,
    original_gross_kg: number,
    corrected_gross_kg: number,
    original_tare_kg: number,
    corrected_tare_kg: number,
    correction_reason: string,
    corrected_by: UUID,
    approved_by: UUID
}

TicketVoided {
    ticket_id: UUID,
    voided_at: Timestamp,
    voided_by: UUID,
    reason: string
}
```

### 3. Compliance Manifest Aggregate

The manifest aggregate is particularly well-suited to event sourcing because regulatory compliance demands a complete, tamper-evident chain of custody.

```
Stream Type: ComplianceManifest
Stream ID: {manifest_id}
```

**Events:**

```typescript
ManifestCreated {
    manifest_id: UUID,
    manifest_type: 'epa_emanifest' | 'uk_waste_transfer_note' | 'state_manifest',
    organization_id: UUID,
    jurisdiction_id: UUID,
    generator: {
        org_id: UUID,
        epa_site_id: string,
        name: string,
        address: Address,
        contact: Contact
    }
}

WasteLineAdded {
    manifest_id: UUID,
    line_number: number,
    material_type_id: UUID,
    waste_description: string,
    dot_hazardous: boolean,
    dot_info: { id_number: string, shipping_name: string, hazard_class: string } | null,
    epa_waste_codes: string[],
    state_waste_codes: string[],
    container_count: number,
    container_type: string,
    quantity: number,
    unit_of_measure: string,
    management_method: string,
    handling_instructions: string | null
}

WasteLineRemoved {
    manifest_id: UUID,
    line_number: number,
    reason: string,
    removed_by: UUID
}

TransporterAssigned {
    manifest_id: UUID,
    transport_order: number,
    transporter_org_id: UUID,
    transporter_epa_id: string,
    assigned_at: Timestamp
}

GeneratorSigned {
    manifest_id: UUID,
    signed_by: UUID,
    signed_at: Timestamp,
    signature_type: 'electronic' | 'wet_ink',
    certification_text: string
}

TransporterSigned {
    manifest_id: UUID,
    transport_order: number,
    signed_by: UUID,
    signed_at: Timestamp,
    signature_type: 'electronic' | 'wet_ink'
}

ManifestShipped {
    manifest_id: UUID,
    shipped_date: Date,
    manifest_tracking_number: string    // MTN assigned
}

FacilityReceived {
    manifest_id: UUID,
    facility_org_id: UUID,
    facility_epa_id: string,
    received_at: Timestamp,
    received_weight_kg: number,
    signed_by: UUID,
    discrepancy_noted: boolean
}

ManifestDiscrepancyRecorded {
    manifest_id: UUID,
    discrepancy_type: 'quantity' | 'waste_type' | 'container' | 'partial_rejection' | 'full_rejection',
    description: string,
    original_values: Record<string, any>,
    actual_values: Record<string, any>,
    recorded_by: UUID
}

ManifestSubmittedToEPA {
    manifest_id: UUID,
    epa_submission_id: string,
    submitted_at: Timestamp,
    submission_type: 'original' | 'correction' | 'resubmission'
}

EPASubmissionAcknowledged {
    manifest_id: UUID,
    epa_submission_id: string,
    epa_response_status: 'accepted' | 'rejected' | 'pending_review',
    epa_response_message: string | null,
    acknowledged_at: Timestamp
}

ManifestCorrected {
    manifest_id: UUID,
    corrected_fields: Record<string, { old_value: any, new_value: any }>,
    correction_reason: string,
    corrected_by: UUID,
    requires_epa_resubmission: boolean
}
```

### 4. Customer Account Aggregate

```
Stream Type: CustomerAccount
Stream ID: {customer_id}
```

**Events:**

```typescript
CustomerCreated {
    customer_id: UUID,
    organization_id: UUID,
    customer_number: string,
    customer_type: 'residential' | 'commercial' | 'industrial' | 'municipal' | 'roll_off',
    name: string,
    billing_address: Address,
    contact: Contact,
    payment_terms_days: number
}

CustomerUpdated {
    customer_id: UUID,
    changed_fields: Record<string, { old_value: any, new_value: any }>
}

ServiceLocationAdded {
    customer_id: UUID,
    location_id: UUID,
    address: Address,
    coordinates: { lat: number, lng: number },
    access_instructions: string | null,
    time_window: { start: Time, end: Time } | null
}

ServiceAgreementCreated {
    customer_id: UUID,
    agreement_id: UUID,
    agreement_number: string,
    start_date: Date,
    end_date: Date | null,
    billing_frequency: string,
    billing_method: string,
    lines: [{
        line_id: UUID,
        location_id: UUID,
        material_type_id: UUID,
        container_type_id: UUID,
        collection_frequency: string,
        collection_days: number[],
        rate_per_unit: number,
        rate_unit: string
    }]
}

ServiceAgreementAmended {
    customer_id: UUID,
    agreement_id: UUID,
    amendments: Record<string, { old_value: any, new_value: any }>,
    effective_date: Date,
    reason: string
}

ContainerDeployed {
    customer_id: UUID,
    container_id: UUID,
    location_id: UUID,
    deployed_at: Timestamp,
    deployed_by: UUID
}

ContainerRetrieved {
    customer_id: UUID,
    container_id: UUID,
    location_id: UUID,
    retrieved_at: Timestamp,
    retrieved_by: UUID,
    reason: string
}

AccountSuspended {
    customer_id: UUID,
    suspended_at: Timestamp,
    reason: string,
    suspended_by: UUID
}

AccountReactivated {
    customer_id: UUID,
    reactivated_at: Timestamp,
    reactivated_by: UUID
}
```

### 5. Invoice Aggregate

```
Stream Type: Invoice
Stream ID: {invoice_id}
```

**Events:**

```typescript
InvoiceGenerated {
    invoice_id: UUID,
    invoice_number: string,
    organization_id: UUID,
    customer_id: UUID,
    billing_period: { start: Date, end: Date },
    lines: [{
        line_number: number,
        description: string,
        line_type: string,
        material_type_id: UUID | null,
        quantity: number,
        unit: string,
        rate: number,
        amount: number,
        source_event_ids: UUID[]     // traceability to collection/weigh events
    }],
    subtotal: number,
    tax_amount: number,
    surcharges: number,
    total_amount: number,
    currency: string
}

InvoiceIssued {
    invoice_id: UUID,
    issued_date: Date,
    due_date: Date,
    delivery_method: 'email' | 'mail' | 'portal'
}

InvoiceLineAdjusted {
    invoice_id: UUID,
    line_number: number,
    original_amount: number,
    adjusted_amount: number,
    adjustment_reason: string,
    adjusted_by: UUID,
    supporting_event_ids: UUID[]      // billing exception events
}

PaymentReceived {
    invoice_id: UUID,
    payment_amount: number,
    payment_method: string,
    payment_reference: string,
    received_at: Timestamp
}

InvoiceDisputed {
    invoice_id: UUID,
    disputed_lines: number[],
    dispute_reason: string,
    disputed_by: UUID,
    disputed_at: Timestamp
}

InvoiceVoided {
    invoice_id: UUID,
    voided_at: Timestamp,
    voided_by: UUID,
    reason: string,
    credit_note_id: UUID | null
}

InvoiceExportedToAccounting {
    invoice_id: UUID,
    external_system: 'quickbooks' | 'xero' | 'sage',
    external_id: string,
    exported_at: Timestamp
}
```

### 6. Contamination Incident Aggregate

```
Stream Type: ContaminationIncident
Stream ID: {incident_id}
```

**Events:**

```typescript
ContaminationDetected {
    incident_id: UUID,
    collection_stop_id: UUID,
    route_id: UUID,
    customer_id: UUID,
    detected_by: UUID,
    detection_method: 'visual' | 'ml_classification' | 'manual_inspection',
    contamination_type: string,
    severity: 'minor' | 'moderate' | 'severe' | 'rejected',
    description: string,
    photo_urls: string[],
    ml_confidence_score: number | null,
    detected_at: Timestamp
}

MLClassificationOverridden {
    incident_id: UUID,
    original_classification: string,
    corrected_classification: string,
    original_severity: string,
    corrected_severity: string,
    overridden_by: UUID,
    reason: string
}

CustomerNotified {
    incident_id: UUID,
    customer_id: UUID,
    notification_method: 'email' | 'sms' | 'app_push' | 'letter',
    notification_content: string,
    sent_at: Timestamp
}

SurchargeApplied {
    incident_id: UUID,
    customer_id: UUID,
    surcharge_amount: number,
    rate_source: string,
    invoice_line_reference: UUID | null
}

IncidentResolved {
    incident_id: UUID,
    resolution: 'acknowledged' | 'corrective_action' | 'no_action' | 'disputed_overturned',
    resolution_notes: string,
    resolved_by: UUID,
    resolved_at: Timestamp
}
```

---

## Read Model Projections

Read models are materialized views of the event stream, optimized for specific query patterns. Each projection subscribes to relevant events and maintains its own denormalized data store.

### Projection 1: Active Routes Dashboard

Optimized for dispatchers monitoring live route progress.

```sql
-- Read model: real-time route status
CREATE TABLE rm_active_routes (
    route_id            UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    route_date          DATE NOT NULL,
    route_code          VARCHAR(30),
    vehicle_number      VARCHAR(30),
    vehicle_type        VARCHAR(30),
    driver_name         VARCHAR(200),
    driver_phone        VARCHAR(30),
    status              VARCHAR(20) NOT NULL,
    started_at          TIMESTAMPTZ,
    total_stops         INTEGER NOT NULL,
    completed_stops     INTEGER NOT NULL DEFAULT 0,
    skipped_stops       INTEGER NOT NULL DEFAULT 0,
    remaining_stops     INTEGER NOT NULL,
    completion_pct      NUMERIC(5,2) GENERATED ALWAYS AS (
                            CASE WHEN total_stops > 0
                                THEN (completed_stops::NUMERIC / total_stops) * 100
                                ELSE 0
                            END
                        ) STORED,
    current_stop_id     UUID,
    current_stop_address VARCHAR(255),
    last_known_location GEOGRAPHY(POINT, 4326),
    last_location_time  TIMESTAMPTZ,
    total_weight_kg     NUMERIC(12,2) DEFAULT 0,
    estimated_finish    TIMESTAMPTZ,
    exceptions_count    INTEGER DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_active_routes_org ON rm_active_routes(organization_id, route_date);

-- Consumes: RouteCreated, RouteDispatched, RouteStarted,
--           StopArrived, StopCompleted, StopSkipped,
--           RouteCompleted, RouteCancelled
```

### Projection 2: Customer Account View

Optimized for customer service and account management.

```sql
-- Read model: denormalized customer view
CREATE TABLE rm_customer_accounts (
    customer_id         UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    customer_number     VARCHAR(50) NOT NULL,
    customer_type       VARCHAR(30) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    email               VARCHAR(255),
    phone               VARCHAR(30),
    billing_address     JSONB,
    account_status      VARCHAR(20) NOT NULL,
    -- Embedded service locations
    service_locations   JSONB NOT NULL DEFAULT '[]',
    -- Active agreements summary
    active_agreements   JSONB NOT NULL DEFAULT '[]',
    -- Container inventory
    deployed_containers JSONB NOT NULL DEFAULT '[]',
    -- Recent activity summary
    last_collection_date DATE,
    collections_this_month INTEGER DEFAULT 0,
    weight_this_month_kg NUMERIC(12,2) DEFAULT 0,
    -- Contamination history
    contamination_incidents_ytd INTEGER DEFAULT 0,
    last_contamination_date DATE,
    -- Billing summary
    current_balance     NUMERIC(12,2) DEFAULT 0,
    last_invoice_date   DATE,
    last_payment_date   DATE,
    payment_status      VARCHAR(20) DEFAULT 'current',
    -- Metadata
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_customers_org ON rm_customer_accounts(organization_id);
CREATE INDEX idx_rm_customers_status ON rm_customer_accounts(account_status);

-- Consumes: CustomerCreated, CustomerUpdated, ServiceLocationAdded,
--           ServiceAgreementCreated, ContainerDeployed, ContainerRetrieved,
--           StopCompleted, ContaminationDetected, InvoiceGenerated, PaymentReceived
```

### Projection 3: Compliance Manifest Register

Optimized for compliance officers tracking manifest status.

```sql
-- Read model: manifest compliance register
CREATE TABLE rm_manifest_register (
    manifest_id         UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    manifest_type       VARCHAR(30) NOT NULL,
    manifest_tracking_number VARCHAR(30),
    status              VARCHAR(30) NOT NULL,
    -- Generator details (denormalized)
    generator_name      VARCHAR(255),
    generator_epa_id    VARCHAR(20),
    generator_address   TEXT,
    -- Facility details (denormalized)
    facility_name       VARCHAR(255),
    facility_epa_id     VARCHAR(20),
    -- Transporters (denormalized array)
    transporters        JSONB NOT NULL DEFAULT '[]',
    -- Waste summary
    waste_line_count    INTEGER DEFAULT 0,
    total_weight_kg     NUMERIC(12,2),
    contains_hazardous  BOOLEAN DEFAULT false,
    epa_waste_codes     VARCHAR(10)[] DEFAULT '{}',
    -- Signature tracking
    generator_signed    BOOLEAN DEFAULT false,
    generator_signed_at TIMESTAMPTZ,
    all_transporters_signed BOOLEAN DEFAULT false,
    facility_signed     BOOLEAN DEFAULT false,
    facility_received_at TIMESTAMPTZ,
    -- EPA submission
    epa_submitted       BOOLEAN DEFAULT false,
    epa_submission_id   VARCHAR(50),
    epa_status          VARCHAR(30),
    -- Discrepancies
    has_discrepancy     BOOLEAN DEFAULT false,
    discrepancy_type    VARCHAR(30),
    -- Dates
    shipped_date        DATE,
    received_date       DATE,
    created_at          TIMESTAMPTZ NOT NULL,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_manifests_org ON rm_manifest_register(organization_id);
CREATE INDEX idx_rm_manifests_status ON rm_manifest_register(status);
CREATE INDEX idx_rm_manifests_mtn ON rm_manifest_register(manifest_tracking_number);
CREATE INDEX idx_rm_manifests_dates ON rm_manifest_register(shipped_date, received_date);

-- Consumes: ManifestCreated, WasteLineAdded, TransporterAssigned,
--           GeneratorSigned, TransporterSigned, ManifestShipped,
--           FacilityReceived, ManifestDiscrepancyRecorded,
--           ManifestSubmittedToEPA, EPASubmissionAcknowledged,
--           ManifestCorrected
```

### Projection 4: Billing Pipeline

Optimized for the billing engine that generates invoices from operational events.

```sql
-- Read model: unbilled collection events awaiting invoicing
CREATE TABLE rm_billing_pipeline (
    event_id            UUID PRIMARY KEY,  -- source event ID
    organization_id     UUID NOT NULL,
    customer_id         UUID NOT NULL,
    event_type          VARCHAR(50) NOT NULL,  -- 'collection' | 'contamination_surcharge' | etc.
    event_date          DATE NOT NULL,
    -- Service details
    service_location_id UUID,
    material_type_id    UUID,
    material_code       VARCHAR(20),
    -- Quantity and pricing
    weight_kg           NUMERIC(12,2),
    volume_cubic_yards  NUMERIC(8,2),
    applicable_rate     NUMERIC(10,4),
    rate_unit           VARCHAR(20),
    calculated_amount   NUMERIC(12,2),
    -- Pricing source
    agreement_line_id   UUID,
    commodity_price_per_ton NUMERIC(10,2),
    -- Status
    billing_status      VARCHAR(20) DEFAULT 'unbilled' CHECK (billing_status IN (
                            'unbilled', 'invoiced', 'adjusted', 'credited', 'exception'
                        )),
    invoice_id          UUID,
    exception_id        UUID,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_billing_unbilled ON rm_billing_pipeline(organization_id, customer_id)
    WHERE billing_status = 'unbilled';
CREATE INDEX idx_rm_billing_date ON rm_billing_pipeline(event_date);

-- Consumes: StopCompleted, TicketFinalized, ContaminationDetected,
--           SurchargeApplied, InvoiceGenerated, InvoiceLineAdjusted
```

### Projection 5: Sustainability & Diversion Metrics

Optimized for ESG dashboards and regulatory diversion reporting.

```sql
-- Read model: material flow aggregation
CREATE TABLE rm_material_flows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    period_date         DATE NOT NULL,
    period_type         VARCHAR(10) NOT NULL,  -- 'daily', 'monthly', etc.
    jurisdiction_id     UUID,
    customer_id         UUID,                  -- NULL for org-wide aggregation
    material_type_id    UUID NOT NULL,
    material_code       VARCHAR(20) NOT NULL,
    material_category   VARCHAR(50) NOT NULL,
    -- Quantities by destination
    collected_kg        NUMERIC(14,2) DEFAULT 0,
    recycled_kg         NUMERIC(14,2) DEFAULT 0,
    composted_kg        NUMERIC(14,2) DEFAULT 0,
    landfilled_kg       NUMERIC(14,2) DEFAULT 0,
    incinerated_kg      NUMERIC(14,2) DEFAULT 0,
    reused_kg           NUMERIC(14,2) DEFAULT 0,
    -- Quality metrics
    contamination_incidents INTEGER DEFAULT 0,
    rejected_loads      INTEGER DEFAULT 0,
    -- Carbon
    collection_co2_kg   NUMERIC(10,2) DEFAULT 0,
    avoided_co2_kg      NUMERIC(10,2) DEFAULT 0,  -- from recycling vs landfill
    -- Revenue
    commodity_revenue   NUMERIC(12,2) DEFAULT 0,
    disposal_cost       NUMERIC(12,2) DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_material_flows_org ON rm_material_flows(organization_id, period_date);
CREATE INDEX idx_rm_material_flows_material ON rm_material_flows(material_type_id, period_date);

-- Consumes: StopCompleted, TicketFinalized, MaterialClassified,
--           ContaminationDetected, RouteCompleted (for CO2 calculations)
```

### Projection 6: Fleet Operations View

```sql
-- Read model: vehicle and driver status
CREATE TABLE rm_fleet_status (
    vehicle_id          UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    vehicle_number      VARCHAR(30) NOT NULL,
    vehicle_type        VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    -- Current assignment
    current_route_id    UUID,
    current_driver_id   UUID,
    current_driver_name VARCHAR(200),
    -- Location
    last_known_location GEOGRAPHY(POINT, 4326),
    last_location_time  TIMESTAMPTZ,
    speed_kmh           NUMERIC(5,1),
    heading             NUMERIC(5,1),
    engine_status       VARCHAR(10),
    -- Today's statistics
    stops_completed_today INTEGER DEFAULT 0,
    weight_collected_today_kg NUMERIC(12,2) DEFAULT 0,
    distance_today_km   NUMERIC(8,2) DEFAULT 0,
    -- Maintenance
    last_inspection_passed BOOLEAN,
    last_inspection_date DATE,
    next_maintenance_due DATE,
    active_diagnostic_codes VARCHAR(10)[],
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_fleet_org ON rm_fleet_status(organization_id);
CREATE INDEX idx_rm_fleet_location ON rm_fleet_status USING GIST(last_known_location);
```

---

## Command Handlers

Commands are validated, business rules are enforced, and resulting events are appended to the event store.

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Mobile App    │────▶│ Command Handler  │────▶│ Event Store │
│   Web Client    │     │                  │     │ (PostgreSQL)│
│   API Gateway   │     │ 1. Load aggregate│     └──────┬──────┘
└─────────────────┘     │ 2. Validate      │            │
                        │ 3. Apply rules   │            ▼
                        │ 4. Emit events   │     ┌─────────────┐
                        └──────────────────┘     │ Event Bus   │
                                                 │ (Kafka)     │
                                                 └──────┬──────┘
                                                        │
                        ┌───────────────────────────────┼───────────────────┐
                        ▼                               ▼                   ▼
                 ┌──────────────┐             ┌──────────────┐     ┌──────────────┐
                 │ Route        │             │ Billing      │     │ Compliance   │
                 │ Projection   │             │ Projection   │     │ Projection   │
                 └──────────────┘             └──────────────┘     └──────────────┘
```

### Sample Command Flow: Complete a Collection Stop

```
1. Driver app sends CompleteStop command:
   {
     command_id: "cmd-123",
     route_id: "route-456",
     stop_id: "stop-789",
     weight_kg: 145.2,
     weight_source: "onboard_scale",
     container_fullness: 0.85,
     photos: ["url1.jpg"],
     timestamp: "2026-05-26T08:30:00Z"
   }

2. Command handler:
   a. Load CollectionRoute aggregate from event store (stream_id = route-456)
   b. Replay events to reconstruct current state
   c. Validate: Is the stop in this route? Is it still pending? Is the route in progress?
   d. Apply business rules: Is weight within expected range? Flag exception if >20% deviation
   e. Emit events:
      - StopCompleted { ... weight data ... }
      - WeightDiscrepancyDetected { ... } (if applicable)
      - ContaminationDetected { ... } (if ML model flags photos)

3. Events appended to event_store within a single transaction

4. Events published to Kafka topics:
   - waste.routes.stop-completed
   - waste.billing.collection-event
   - waste.compliance.material-movement (if hazardous)
```

---

## Offline Sync Strategy

Event sourcing naturally supports offline-capable mobile apps:

```
1. Driver app operates offline, recording local events with timestamps
2. Events are queued in local SQLite database on the device
3. When connectivity is restored, queued events are submitted as commands
4. Command handlers detect `synced_from_offline: true` and apply relaxed
   temporal ordering (accept events with timestamps older than current state)
5. Conflict resolution:
   a. If dispatcher modified the route while driver was offline,
      both event streams are preserved (no data loss)
   b. Stop-level conflicts resolved by "last writer wins" on status,
      but all events remain in the store for auditing
6. Read model projections reconcile automatically as events are processed
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design.** Every state change — a weight correction, a manifest amendment, a billing adjustment — is captured as an immutable event. Regulatory auditors can replay the exact sequence of actions on any entity. This exceeds the capabilities of a trigger-based audit log because events carry business semantics ("WeightCorrected with reason"), not just column diffs.

2. **Temporal queries are native.** "What was the status of manifest MTN-123456789ABC at 3:00 PM on March 15?" is answered by replaying events up to that timestamp. This is essential for compliance investigations and dispute resolution.

3. **Offline sync is structurally sound.** Events from offline drivers merge into the event stream without conflicting updates. The append-only model means no data is ever overwritten, so offline and online events coexist.

4. **Read models are independently optimized.** The billing pipeline projection is structured completely differently from the compliance register, yet both derive from the same event stream. New reporting requirements (e.g., a new ESG disclosure standard) are implemented by adding a new projection that replays historical events — no data migration needed.

5. **Event replay enables retroactive analysis.** When a municipality introduces a new diversion metric, the sustainability projection can be rebuilt from historical events to populate data retroactively.

6. **Natural fit for integration.** Events published to Kafka enable downstream systems (accounting, telematics, regulatory submission APIs) to consume relevant events independently. Adding a new integration means adding a new consumer, not modifying the core system.

### Cons

1. **Operational complexity is substantially higher.** The system requires PostgreSQL (event store), Kafka (event bus), and potentially Redis (read model caching). Each component needs monitoring, backup, and failure recovery. This is a significant burden for a small hauler with limited IT staff.

2. **Event schema evolution is difficult.** When business requirements change the structure of an event (e.g., adding a field to StopCompleted), existing events must be "upcasted" during replay. Version management across potentially millions of events requires careful design and testing.

3. **Eventual consistency between write and read models.** After a driver completes a stop, there is a delay (typically milliseconds to seconds, but potentially longer under load) before the dispatcher dashboard reflects the change. For real-time fleet monitoring, this latency must be managed.

4. **Debugging is harder.** Understanding the current state of a route requires replaying potentially hundreds of events. Without proper tooling (event store browsers, projection debuggers), developers and support staff struggle to diagnose issues.

5. **Storage growth.** Events are never deleted. A fleet of 100 trucks generating 20+ events per stop across 200 stops per day produces ~4,000+ events daily. Over years, the event store grows to billions of rows. While events compress well, the storage and indexing costs are non-trivial.

6. **Learning curve.** Event sourcing and CQRS are unfamiliar patterns for many developers. Recruiting and onboarding is harder compared to a conventional CRUD application with an ORM.

7. **Invoice generation complexity.** Generating a correct invoice requires the billing projection to correlate events across multiple aggregates (stops, weigh tickets, contamination incidents, service agreements). The projection logic becomes complex and must handle late-arriving events (e.g., a weigh ticket corrected after the billing period closes).

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event store** | PostgreSQL 16+ (proven, transactional, supports JSONB events) |
| **Event bus** | Apache Kafka or Redpanda (for event distribution to projections and integrations) |
| **Read model store** | PostgreSQL (for projections requiring SQL queries) |
| **Read model cache** | Redis or Valkey (for hot read models like fleet status) |
| **Event store library** | Marten (C#/.NET), EventStoreDB (standalone), or custom on PostgreSQL |
| **Projection framework** | Custom projection runners with checkpoint management |
| **Serialization** | JSON with schema registry (Confluent Schema Registry or custom) |
| **Mobile offline store** | SQLite with event queue for offline command buffering |
| **Monitoring** | Kafka lag monitoring (Burrow), PostgreSQL stats, projection lag alerting |

---

## Migration & Scaling Considerations

### Initial Deployment
- PostgreSQL handles both event store and read models on a single instance
- Kafka can be replaced by PostgreSQL LISTEN/NOTIFY for low-volume deployments
- Read model projections run as background workers in the application process
- Estimated event volume: ~1,000 events/day for a 10-truck operation

### Growth Phase (50+ trucks)
- Separate PostgreSQL instances for event store and read models
- Deploy Kafka cluster (3 brokers) for reliable event distribution
- Partition event_store by stream_type for query performance
- Implement event snapshots for aggregates with long histories (routes with 200+ stops)
- Add Redis for the fleet status read model (sub-second update requirements)
- Estimated event volume: ~10,000 events/day

### Scale Phase (500+ trucks, multi-region)
- Event store partitioned by organization_id using Citus
- Kafka with multi-region replication for geographic distribution
- Independent projection workers per read model, each scaling horizontally
- Event archival: move events older than 2 years to cold storage (S3 + Athena for ad-hoc queries)
- Consider EventStoreDB as a purpose-built event store if PostgreSQL becomes a bottleneck
- Estimated event volume: ~100,000+ events/day

### Event Schema Migration Strategy
1. **Never modify existing event schemas** — always create new versions
2. **Upcasters** transform old event versions to new versions during replay
3. **Schema registry** validates event payloads before appending to the store
4. **Projection rebuilds** are tested against production event snapshots before deployment
5. **Blue-green projection deployment** — new projection runs alongside old, switched after validation

### Transitioning from Legacy Systems
1. Import historical data as "synthetic" events with clear metadata markers
2. Use a migration aggregate that emits `LegacyDataImported` events
3. Build projections from synthetic events to populate initial read models
4. Validate read model data against legacy system reports
5. Run parallel billing cycles comparing event-sourced invoices against legacy
