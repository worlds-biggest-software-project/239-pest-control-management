# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Pest Control Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event stored in an append-only event store. The current state of any entity (a job, a chemical application, a trap inspection) is derived by replaying its event stream. Materialised read models (projections) provide fast query performance for operational screens, while the event store serves as the authoritative source of truth.

This approach is a natural fit for pest control management because the domain is heavily regulated. EPA FIFRA compliance requires that pesticide application records be retained for years and be auditable at any time. State licensing boards may audit a company's chemical usage history, technician assignments, and re-entry interval compliance. An event-sourced model provides an inherent, tamper-evident audit trail: you can answer "what was the state of this service location's treatment history on any given date?" by replaying events up to that point.

The CQRS (Command Query Responsibility Segregation) pattern separates write operations (commands that produce events) from read operations (queries against materialised projections). This enables specialised read models: a scheduling projection optimised for calendar display, a compliance projection optimised for regulatory reports, and an analytics projection optimised for AI-driven insights like churn prediction and re-treatment scheduling.

**Best for:** Organisations where regulatory compliance, complete audit trails, and temporal querying are primary requirements, and where AI/ML analytics on historical change patterns are a strategic priority.

**Trade-offs:**
- (+) Complete, immutable audit trail satisfying EPA/FIFRA record retention requirements
- (+) Temporal queries ("what chemicals were stored in vehicle #5 on March 15th?") are trivial
- (+) Event streams feed AI/ML pipelines directly without ETL
- (+) Can replay events to rebuild read models or create new projections without data loss
- (+) Natural fit for regulatory environments where "who changed what and when" must be provable
- (-) Higher implementation complexity — requires event store, projection engine, and read model management
- (-) Eventual consistency between event store and read models requires careful UX design
- (-) Debugging requires understanding event replay, not just reading current state
- (-) Event schema evolution (versioning) adds ongoing maintenance burden
- (-) Steeper learning curve for developers unfamiliar with event sourcing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EPA FIFRA / 40 CFR 170 | Chemical application events are immutable records that can never be deleted — exactly matching regulatory retention requirements |
| EPA PPLS API | `PesticideProductSynced` events track when reference data was fetched and what changed |
| NPMA QualityPro | `CertificationGranted`, `CertificationExpired` events create a verifiable credential timeline |
| ISO 9001:2015 | Event-sourced audit trail directly satisfies ISO 9001 requirements for documented evidence of quality management activities |
| OWASP API Security | Command/query separation enforces write-path validation independently from read-path access control |
| RFC 5545 iCalendar | Scheduling projections can export appointment events as VEVENT entries |
| RFC 7946 GeoJSON | Location events include GeoJSON coordinates for spatial projections |

---

## Event Store Foundation

```sql
-- The event store is the single source of truth.
-- All other tables are materialised projections that can be rebuilt from events.

CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
        -- Job, Customer, ServiceLocation, ChemicalApplication, Device,
        -- Appointment, Route, Invoice, TechnicianProfile, Contract, Vehicle
    stream_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
        -- e.g., JobCreated, JobScheduled, JobDispatched, JobStarted, JobCompleted,
        --       ChemicalApplied, ChemicalApplicationCorrected,
        --       DevicePlaced, DeviceInspected, DeviceDecommissioned,
        --       AppointmentScheduled, AppointmentRescheduled, AppointmentCancelled,
        --       TechnicianAssigned, TechnicianCertificationAdded,
        --       InvoiceIssued, PaymentReceived
    event_version INTEGER NOT NULL,  -- for event schema versioning
    sequence_number BIGINT NOT NULL,  -- per-stream ordering
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
        -- { "user_id": "...", "ip_address": "...", "correlation_id": "...", "causation_id": "..." }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_type, stream_id, sequence_number)
);

-- Primary query path: replay a single stream
CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, sequence_number);

-- Projection catch-up: read all events since a checkpoint
CREATE INDEX idx_events_tenant_created ON event_store(tenant_id, created_at);

-- Event type filtering for specific projections
CREATE INDEX idx_events_type ON event_store(event_type, created_at);

-- Partition by month for performance at scale
-- In production, consider range-partitioning on created_at
```

## Event Taxonomy

```sql
-- This is not a table but a reference for the event types and their payloads.
-- Each event_data JSONB follows a documented schema.

/*
=== CUSTOMER EVENTS ===

CustomerCreated:
{
  "customer_id": "uuid",
  "first_name": "John",
  "last_name": "Smith",
  "email": "john@example.com",
  "phone": "555-0100",
  "customer_type": "residential",
  "billing_method": "monthly"
}

CustomerUpdated:
{
  "customer_id": "uuid",
  "changes": {
    "phone": { "old": "555-0100", "new": "555-0200" }
  }
}

=== SERVICE LOCATION EVENTS ===

ServiceLocationAdded:
{
  "location_id": "uuid",
  "customer_id": "uuid",
  "address_line1": "123 Main St",
  "city": "Springfield",
  "state_code": "IL",
  "postal_code": "62701",
  "coordinates": { "type": "Point", "coordinates": [-89.6501, 39.7817] },
  "property_type": "single_family",
  "square_footage": 2400
}

=== JOB EVENTS ===

JobCreated:
{
  "job_id": "uuid",
  "customer_id": "uuid",
  "location_id": "uuid",
  "service_type": "general_pest",
  "title": "Quarterly Pest Treatment",
  "estimated_duration_minutes": 45,
  "price_cents": 15000
}

JobDispatched:
{
  "job_id": "uuid",
  "technician_ids": ["uuid1", "uuid2"],
  "lead_technician_id": "uuid1"
}

JobStarted:
{
  "job_id": "uuid",
  "technician_id": "uuid",
  "actual_start": "2026-05-22T09:15:00Z",
  "gps_coordinates": { "type": "Point", "coordinates": [-89.6501, 39.7817] }
}

JobCompleted:
{
  "job_id": "uuid",
  "actual_end": "2026-05-22T10:00:00Z",
  "pests_found": ["german_cockroach", "ant_carpenter"],
  "treatment_summary": "Applied gel bait to kitchen and bathroom. Exterior perimeter spray.",
  "follow_up_required": true,
  "follow_up_date": "2026-06-22",
  "customer_signature_url": "https://..."
}

=== CHEMICAL APPLICATION EVENTS (EPA COMPLIANCE) ===

ChemicalApplied:
{
  "application_id": "uuid",
  "job_id": "uuid",
  "technician_id": "uuid",
  "location_id": "uuid",
  "epa_registration_number": "100-1066",
  "product_name": "Advion Cockroach Gel Bait",
  "active_ingredients": ["Indoxacarb 0.6%"],
  "application_date": "2026-05-22",
  "application_time": "09:30",
  "target_pests": ["german_cockroach"],
  "application_site": "interior kitchen cabinets, bathroom vanity",
  "application_method": "bait",
  "quantity_applied": 30,
  "unit_of_measure": "g",
  "dilution_rate": null,
  "re_entry_interval_hours": 0,
  "restricted_use_product": false,
  "applicator_licence_number": "IL-AG-12345",
  "temperature_f": 72,
  "wind_speed_mph": null
}

ChemicalApplicationCorrected:
{
  "application_id": "uuid",
  "correction_reason": "Incorrect quantity recorded in field",
  "corrected_by": "uuid",
  "changes": {
    "quantity_applied": { "old": 30, "new": 25 }
  }
}
-- Note: The original ChemicalApplied event is NEVER modified.
-- Corrections are modelled as new events that reference the original.

=== DEVICE/TRAP EVENTS ===

DevicePlaced:
{
  "device_id": "uuid",
  "location_id": "uuid",
  "device_type": "bait_station",
  "barcode": "BS-2026-00142",
  "placement_description": "Under kitchen sink, left side",
  "coordinates": { "type": "Point", "coordinates": [-89.6502, 39.7818] },
  "installed_date": "2026-05-22"
}

DeviceInspected:
{
  "device_id": "uuid",
  "technician_id": "uuid",
  "appointment_id": "uuid",
  "inspection_date": "2026-05-22T09:45:00Z",
  "activity_level": "moderate",
  "findings": "Signs of rodent feeding on bait block",
  "action_taken": "bait_replaced",
  "photo_urls": ["https://..."]
}

DeviceDecommissioned:
{
  "device_id": "uuid",
  "reason": "Customer renovation — kitchen being gutted",
  "decommissioned_by": "uuid"
}

=== CERTIFICATION EVENTS ===

CertificationAdded:
{
  "technician_id": "uuid",
  "certification_type": "state_applicator",
  "certification_number": "IL-AG-12345",
  "issuing_authority": "Illinois Department of Agriculture",
  "state_code": "IL",
  "category": "Category 7A - General Pest",
  "issued_date": "2024-01-15",
  "expiry_date": "2027-01-15"
}

CertificationExpired:
{
  "technician_id": "uuid",
  "certification_number": "IL-AG-12345",
  "expired_on": "2027-01-15"
}

=== SCHEDULING EVENTS ===

AppointmentScheduled:
{
  "appointment_id": "uuid",
  "job_id": "uuid",
  "scheduled_start": "2026-05-22T09:00:00Z",
  "scheduled_end": "2026-05-22T10:00:00Z",
  "arrival_window_start": "2026-05-22T09:00:00Z",
  "arrival_window_end": "2026-05-22T10:00:00Z",
  "technician_ids": ["uuid"]
}

AppointmentRescheduled:
{
  "appointment_id": "uuid",
  "old_start": "2026-05-22T09:00:00Z",
  "new_start": "2026-05-23T14:00:00Z",
  "reason": "customer_request"
}

=== BILLING EVENTS ===

InvoiceIssued:
{
  "invoice_id": "uuid",
  "customer_id": "uuid",
  "job_id": "uuid",
  "invoice_number": "INV-2026-0042",
  "line_items": [
    { "description": "Quarterly Pest Treatment", "quantity": 1, "unit_price_cents": 15000, "total_cents": 15000 }
  ],
  "subtotal_cents": 15000,
  "tax_cents": 1200,
  "total_cents": 16200,
  "due_date": "2026-06-22"
}

PaymentReceived:
{
  "invoice_id": "uuid",
  "payment_id": "uuid",
  "amount_cents": 16200,
  "payment_method": "credit_card",
  "processor_transaction_id": "ch_xxx"
}
*/
```

## Materialised Read Models (Projections)

```sql
-- These tables are DERIVED from the event store.
-- They can be dropped and rebuilt at any time by replaying events.
-- Each projection has a checkpoint tracking its position in the event stream.

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    events_processed BIGINT NOT NULL DEFAULT 0
);

-- =============================================
-- PROJECTION: Current Customer State
-- Built from: CustomerCreated, CustomerUpdated
-- =============================================
CREATE TABLE proj_customers (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(30),
    customer_type VARCHAR(30),
    billing_method VARCHAR(30),
    tags VARCHAR(50)[],
    total_jobs_completed INTEGER NOT NULL DEFAULT 0,
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    last_service_date DATE,
    next_scheduled_date DATE,
    churn_risk_score NUMERIC(4,3),  -- AI-computed from event patterns
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_customers_tenant ON proj_customers(tenant_id);

-- =============================================
-- PROJECTION: Scheduling Board
-- Built from: AppointmentScheduled, Rescheduled, Cancelled, JobDispatched, JobStarted, JobCompleted
-- Optimised for: calendar display, dispatch board
-- =============================================
CREATE TABLE proj_schedule (
    appointment_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    job_id UUID NOT NULL,
    customer_name VARCHAR(200),
    service_location_address VARCHAR(500),
    service_location_coordinates GEOGRAPHY(POINT, 4326),
    service_type VARCHAR(50),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    arrival_window_start TIMESTAMPTZ,
    arrival_window_end TIMESTAMPTZ,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    technician_ids UUID[],
    lead_technician_name VARCHAR(200),
    status VARCHAR(30) NOT NULL,
    priority VARCHAR(20)
);

CREATE INDEX idx_proj_schedule_date ON proj_schedule(tenant_id, scheduled_start);
CREATE INDEX idx_proj_schedule_techs ON proj_schedule USING GIN(technician_ids);

-- =============================================
-- PROJECTION: Chemical Compliance Register
-- Built from: ChemicalApplied, ChemicalApplicationCorrected
-- Optimised for: regulatory reporting, EPA compliance audits
-- =============================================
CREATE TABLE proj_chemical_register (
    application_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    job_id UUID NOT NULL,
    technician_name VARCHAR(200) NOT NULL,
    applicator_licence_number VARCHAR(50),
    customer_name VARCHAR(200) NOT NULL,
    service_address VARCHAR(500) NOT NULL,
    epa_registration_number VARCHAR(30) NOT NULL,
    product_name VARCHAR(500) NOT NULL,
    active_ingredients TEXT[],
    application_date DATE NOT NULL,
    application_time TIME NOT NULL,
    target_pests TEXT[],
    application_site VARCHAR(255) NOT NULL,
    application_method VARCHAR(50) NOT NULL,
    quantity_applied NUMERIC(10,3) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL,
    dilution_rate VARCHAR(50),
    re_entry_interval_hours NUMERIC(6,1),
    restricted_use_product BOOLEAN NOT NULL,
    temperature_f NUMERIC(5,1),
    wind_speed_mph NUMERIC(4,1),
    correction_count INTEGER NOT NULL DEFAULT 0,
    last_corrected_at TIMESTAMPTZ
);

CREATE INDEX idx_proj_chem_date ON proj_chemical_register(tenant_id, application_date);
CREATE INDEX idx_proj_chem_product ON proj_chemical_register(epa_registration_number);
CREATE INDEX idx_proj_chem_technician ON proj_chemical_register(tenant_id, technician_name);

-- =============================================
-- PROJECTION: Device Status Map
-- Built from: DevicePlaced, DeviceInspected, DeviceDecommissioned
-- Optimised for: map display, inspection scheduling
-- =============================================
CREATE TABLE proj_device_status (
    device_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    location_id UUID NOT NULL,
    customer_name VARCHAR(200),
    service_address VARCHAR(500),
    device_type VARCHAR(50),
    barcode VARCHAR(100),
    placement_description VARCHAR(255),
    coordinates GEOGRAPHY(POINT, 4326),
    status VARCHAR(30) NOT NULL,
    installed_date DATE NOT NULL,
    last_inspected_at TIMESTAMPTZ,
    last_activity_level VARCHAR(20),
    inspection_count INTEGER NOT NULL DEFAULT 0,
    next_inspection_due DATE
);

CREATE INDEX idx_proj_devices_tenant ON proj_device_status(tenant_id);
CREATE INDEX idx_proj_devices_location ON proj_device_status(location_id);
CREATE INDEX idx_proj_devices_due ON proj_device_status(tenant_id, next_inspection_due) WHERE status = 'active';

-- =============================================
-- PROJECTION: Technician Dashboard
-- Built from: CertificationAdded, CertificationExpired, JobCompleted, ChemicalApplied
-- Optimised for: credential management, performance metrics
-- =============================================
CREATE TABLE proj_technician_dashboard (
    technician_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    full_name VARCHAR(200) NOT NULL,
    active_certifications JSONB NOT NULL DEFAULT '[]',
        -- [{ "type": "state_applicator", "number": "IL-AG-12345", "state": "IL", "expiry": "2027-01-15" }]
    earliest_cert_expiry DATE,
    skills TEXT[],
    jobs_completed_30d INTEGER NOT NULL DEFAULT 0,
    jobs_completed_ytd INTEGER NOT NULL DEFAULT 0,
    chemicals_applied_30d INTEGER NOT NULL DEFAULT 0,
    avg_job_duration_minutes NUMERIC(6,1),
    current_route_id UUID,
    last_updated_at TIMESTAMPTZ
);

CREATE INDEX idx_proj_tech_tenant ON proj_technician_dashboard(tenant_id);
CREATE INDEX idx_proj_tech_cert_expiry ON proj_technician_dashboard(tenant_id, earliest_cert_expiry);

-- =============================================
-- PROJECTION: Revenue & Billing
-- Built from: InvoiceIssued, PaymentReceived, InvoiceVoided
-- Optimised for: financial dashboards, QuickBooks sync
-- =============================================
CREATE TABLE proj_invoices (
    invoice_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    customer_name VARCHAR(200),
    job_id UUID,
    invoice_number VARCHAR(50),
    status VARCHAR(30) NOT NULL,
    total_cents INTEGER NOT NULL,
    amount_paid_cents INTEGER NOT NULL DEFAULT 0,
    balance_due_cents INTEGER NOT NULL,
    issued_date DATE,
    due_date DATE,
    paid_date DATE,
    quickbooks_sync_status VARCHAR(20) DEFAULT 'pending',
    last_updated_at TIMESTAMPTZ
);

CREATE INDEX idx_proj_invoices_tenant ON proj_invoices(tenant_id, status);
CREATE INDEX idx_proj_invoices_customer ON proj_invoices(customer_id);
```

## Temporal Query Examples

```sql
-- "What chemicals were applied at 123 Main St between Jan and March 2026?"
-- Answer by querying the event store directly — no projection needed.

SELECT
    e.event_data->>'product_name' AS product,
    e.event_data->>'epa_registration_number' AS epa_number,
    e.event_data->>'quantity_applied' AS quantity,
    e.event_data->>'unit_of_measure' AS unit,
    e.event_data->>'application_date' AS applied_on,
    e.event_data->>'technician_id' AS technician,
    e.created_at AS recorded_at
FROM event_store e
WHERE e.tenant_id = $1
  AND e.event_type = 'ChemicalApplied'
  AND e.event_data->>'location_id' = $2
  AND e.created_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY e.created_at;


-- "What was the status of device BS-2026-00142 on February 15th?"
-- Replay device events up to that date.

SELECT
    e.event_type,
    e.event_data,
    e.created_at
FROM event_store e
WHERE e.stream_type = 'Device'
  AND e.stream_id = $device_id
  AND e.created_at <= '2026-02-15T23:59:59Z'
ORDER BY e.sequence_number;
-- The application replays these events in order to reconstruct device state at that point in time.


-- "Show the complete audit history of a corrected chemical application"

SELECT
    e.event_type,
    e.event_data,
    e.metadata->>'user_id' AS changed_by,
    e.created_at
FROM event_store e
WHERE e.stream_type = 'ChemicalApplication'
  AND e.stream_id = $application_id
ORDER BY e.sequence_number;
-- Returns: ChemicalApplied (original), then ChemicalApplicationCorrected (each correction)
-- The full chain is immutable — corrections reference but never overwrite the original.
```

## Command Handlers (Application Layer Reference)

```
-- These are NOT database objects but document the write-side architecture.
-- Each command validates business rules before appending events to the store.

ApplyChemical command:
  1. Validate technician has active certification for the product's category
  2. Validate product EPA registration is current (check pesticide_products reference)
  3. Validate restricted-use products are applied by appropriately certified applicator
  4. Validate quantity does not exceed label-specified maximum application rate
  5. If all validations pass → append ChemicalApplied event
  6. If restricted-use → also append RestrictedUseProductApplied event for compliance flagging

ScheduleAppointment command:
  1. Validate technician availability (no overlapping appointments in schedule projection)
  2. Validate technician certifications cover the service type
  3. Validate service location exists and is active
  4. Append AppointmentScheduled event
  5. Projection engine updates proj_schedule within milliseconds

CompleteJob command:
  1. Validate all required chemical applications have been recorded
  2. Validate customer signature captured (if required by tenant settings)
  3. Append JobCompleted event
  4. Triggers: update proj_customers (last_service_date, total_jobs), 
              update proj_technician_dashboard (jobs_completed),
              create InvoiceIssued event if auto-invoice enabled
```

## Snapshot Strategy

```sql
-- For streams with many events (e.g., a customer with 500+ service visits),
-- maintain periodic snapshots to avoid replaying the full event history.

CREATE TABLE stream_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type VARCHAR(50) NOT NULL,
    stream_id UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,  -- sequence_number this snapshot reflects
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_type, stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON stream_snapshots(stream_type, stream_id, snapshot_version DESC);

-- To rebuild current state:
-- 1. Load latest snapshot for the stream
-- 2. Replay only events with sequence_number > snapshot_version
-- 3. Snapshots are created every 100 events per stream (configurable)
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only event table (partitioned by month in production) |
| Projection Infrastructure | 1 | projection_checkpoints |
| Snapshots | 1 | stream_snapshots for long-lived streams |
| Customer Projection | 1 | proj_customers |
| Scheduling Projection | 1 | proj_schedule |
| Chemical Compliance Projection | 1 | proj_chemical_register |
| Device Status Projection | 1 | proj_device_status |
| Technician Projection | 1 | proj_technician_dashboard |
| Billing Projection | 1 | proj_invoices |
| Reference Data | 0 | pesticide_products could be a separate reference table or fetched on demand |
| **Total** | **9** | Plus reference data tables as needed |

---

## Key Design Decisions

1. **Single event store table** — all domain events across all entity types go into one table, partitioned by `created_at`. This simplifies the write path and makes cross-entity correlation queries possible (e.g., "what happened across all entities for tenant X on date Y?").

2. **Immutable chemical application records** — `ChemicalApplied` events can never be modified or deleted. Corrections are modelled as separate `ChemicalApplicationCorrected` events that reference the original. This directly satisfies EPA/FIFRA audit requirements and provides a tamper-evident record chain.

3. **JSONB event payloads with versioned schemas** — `event_version` on each event enables schema evolution. When the shape of a `ChemicalApplied` event changes (e.g., adding a new required field), the version number increments and the projection engine handles both old and new versions during replay.

4. **Purpose-built projections per read concern** — each projection is optimised for a specific use case. The scheduling projection denormalises customer name and address into each row for fast calendar rendering without JOINs. The chemical register projection flattens everything needed for a compliance report. This eliminates the multi-JOIN problem of the normalised model.

5. **Projection rebuild capability** — projections can be dropped and rebuilt from the event store at any time. This enables adding new projections (e.g., a franchise benchmarking dashboard) without schema migrations affecting existing data.

6. **Metadata on every event** — `user_id`, `ip_address`, `correlation_id`, and `causation_id` in the metadata JSONB enable full traceability. Correlation IDs link events triggered by the same user action; causation IDs link events triggered by other events (e.g., `JobCompleted` causing `InvoiceIssued`).

7. **Snapshots for performance** — long-lived streams (customers with years of service history) get periodic snapshots to bound replay time. The snapshot + tail-event pattern keeps read performance constant regardless of stream length.

8. **AI/ML pipeline integration** — the event store is a natural source for ML feature engineering. Churn prediction models can consume `JobCompleted`, `AppointmentCancelled`, and `PaymentReceived` event streams directly. Re-treatment interval prediction can analyse `ChemicalApplied` event sequences per location. No separate ETL pipeline is needed.

9. **Stream-per-aggregate** — each aggregate (Customer, Job, Device, etc.) has its own stream identified by `stream_type` + `stream_id`. This enables optimistic concurrency control: commands check the expected `sequence_number` before appending, preventing lost-update conflicts.

10. **Separate reference data management** — EPA pesticide product data (from PPLS API) is not event-sourced. It is either stored in a conventional reference table or fetched on demand, since it is externally maintained and does not need audit trail tracking within the application.
