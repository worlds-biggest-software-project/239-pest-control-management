# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Pest Control Management · Created: 2026-05-22

## Philosophy

This model keeps core, frequently-queried, and compliance-critical fields as typed relational columns while using PostgreSQL JSONB columns for variable, jurisdiction-specific, and extensible data. The result is a schema with fewer tables than a fully normalised model, faster development velocity, and the ability to accommodate differences between states, pest types, and customer requirements without schema migrations.

Pest control management is a domain where variability is high: each US state has different pesticide applicator categories and licensing requirements; different pest types demand different treatment documentation; commercial versus residential customers have different service plan structures; and IPM (Integrated Pest Management) protocols vary by certification body. A fully normalised model would need junction tables and nullable columns for every variant, while a pure document model would sacrifice query performance on the fields used in every compliance report. The hybrid approach puts EPA-mandated fields (registration number, quantity, date, method) in typed columns for indexing and validation, and puts everything else (jurisdiction-specific fields, custom forms, IPM checklists) in JSONB.

This pattern is widely used in modern SaaS platforms. Jobber's custom fields feature (linkable to Clients, Properties, Quotes, Jobs, Invoices) and GorillaDesk's flexible device tracking both reflect a hybrid relational/flexible schema philosophy. PostgreSQL's JSONB type with GIN indexes provides excellent query performance on semi-structured data while maintaining ACID guarantees.

**Best for:** Rapid MVP development and multi-jurisdiction deployments where state-by-state regulatory variations, custom form fields, and tenant-specific configurations must be accommodated without continuous schema migrations.

**Trade-offs:**
- (+) Fewer tables (~20) — faster to build, easier to understand
- (+) JSONB columns absorb jurisdiction-specific and tenant-specific variation without migrations
- (+) Custom fields per tenant without schema changes — mirrors Jobber's custom fields model
- (+) GIN indexes on JSONB provide fast containment queries
- (+) Easier to evolve incrementally — promote JSONB fields to typed columns when patterns stabilise
- (-) JSONB fields lack database-enforced type safety and foreign key constraints
- (-) JSONB query syntax is less familiar to many developers
- (-) Reporting tools may struggle with JSONB columns without custom extraction
- (-) Risk of schema drift if JSONB structures are not documented and validated at the application layer
- (-) Partial updates to JSONB require rewriting the entire JSONB value (no in-place field update)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EPA FIFRA / 40 CFR 170 | Core compliance fields (EPA Reg Number, quantity, date, method, target pests, REI) are typed relational columns; jurisdiction-specific extras live in `state_compliance_data` JSONB |
| EPA PPLS API | `pesticide_products` reference table with PPLS response cached in `ppls_raw` JSONB column |
| NPMA QualityPro / GreenPro | Certification details stored in `certifications` JSONB array on technician profile — flexible enough for any certification body |
| ISO 3166 | `state_code` and `country_code` as typed columns; sub-jurisdiction details in JSONB for counties/municipalities |
| RFC 5545 iCalendar | `recurrence_rule` as typed VARCHAR column on appointments |
| RFC 7946 GeoJSON | Coordinates stored as PostGIS GEOGRAPHY; JSONB `geofence` field for complex service area boundaries |
| JSON Schema | Application layer validates JSONB columns against JSON Schema definitions per entity type |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    subscription_plan VARCHAR(50) NOT NULL DEFAULT 'trial',
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    -- Tenant-specific configuration in JSONB avoids separate settings tables
    config JSONB NOT NULL DEFAULT '{}',
    /*
    config example:
    {
      "branding": { "logo_url": "...", "primary_color": "#2563eb" },
      "compliance": {
        "default_state": "IL",
        "require_customer_signature": true,
        "chemical_record_retention_years": 7,
        "require_weather_on_application": false
      },
      "scheduling": {
        "default_duration_minutes": 60,
        "arrival_window_minutes": 120,
        "max_daily_stops": 14
      },
      "billing": {
        "tax_rate_pct": 8.25,
        "auto_invoice_on_completion": true,
        "payment_terms_days": 30
      },
      "custom_field_definitions": {
        "customers": [
          { "key": "referral_source", "label": "Referral Source", "type": "select", "options": ["Google", "Yelp", "Referral", "Door-to-door"] },
          { "key": "pets_on_property", "label": "Pets on Property", "type": "text" }
        ],
        "jobs": [
          { "key": "ipm_checklist_completed", "label": "IPM Checklist Completed", "type": "boolean" }
        ]
      }
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(30),
    role VARCHAR(50) NOT NULL DEFAULT 'technician',
    is_active BOOLEAN NOT NULL DEFAULT true,
    -- Technician-specific profile data in JSONB (avoids separate technician_profiles table)
    profile JSONB NOT NULL DEFAULT '{}',
    /*
    profile example (for technicians):
    {
      "employee_number": "T-042",
      "hire_date": "2024-03-15",
      "hourly_rate_cents": 2500,
      "commission_rate_pct": 5.0,
      "vehicle_id": "uuid",
      "skills": ["general_pest", "termite", "rodent", "bed_bug"],
      "max_daily_stops": 12,
      "certifications": [
        {
          "type": "state_applicator",
          "number": "IL-AG-12345",
          "issuing_authority": "Illinois Department of Agriculture",
          "state_code": "IL",
          "categories": ["7A - General Pest", "7B - Termite"],
          "issued_date": "2024-01-15",
          "expiry_date": "2027-01-15",
          "status": "active",
          "document_url": "https://..."
        },
        {
          "type": "qualitypro",
          "number": "QP-2025-0042",
          "issuing_authority": "NPMA",
          "issued_date": "2025-06-01",
          "expiry_date": "2026-06-01",
          "status": "active"
        }
      ]
    }
    */
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_certifications ON users USING GIN(profile jsonb_path_ops);
-- Example query: Find technicians with expiring certifications
-- SELECT * FROM users WHERE profile @> '{"certifications": [{"status": "active"}]}'
--   AND (profile->'certifications'->0->>'expiry_date')::date < now() + interval '90 days';
```

## Customers & Service Locations

```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_number VARCHAR(50),
    -- Core fields that every customer has and that are always queried
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(30),
    customer_type VARCHAR(30) NOT NULL DEFAULT 'residential',
    billing_method VARCHAR(30) NOT NULL DEFAULT 'per_service',
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Flexible fields for tenant-specific and variable data
    extra JSONB NOT NULL DEFAULT '{}',
    /*
    extra example:
    {
      "alt_phone": "555-0200",
      "referral_source": "Google",
      "pets_on_property": "2 dogs, 1 cat",
      "preferred_contact_method": "sms",
      "tax_exempt": true,
      "tax_exempt_id": "IL-EX-12345",
      "tags": ["vip", "commercial-monthly"],
      "notes": "Gate code: 1234. Dog is friendly.",
      "quickbooks_id": "QBO-123456",
      "churn_risk_score": 0.23,
      "lifetime_revenue_cents": 456000,
      "last_review_rating": 5
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_name ON customers(tenant_id, last_name, first_name);
CREATE INDEX idx_customers_extra ON customers USING GIN(extra jsonb_path_ops);

CREATE TABLE service_locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    -- Core address fields (always queried, always present)
    address_line1 VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state_code VARCHAR(10) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    coordinates GEOGRAPHY(POINT, 4326),
    -- Flexible property details
    details JSONB NOT NULL DEFAULT '{}',
    /*
    details example:
    {
      "label": "Main Office",
      "address_line2": "Suite 400",
      "property_type": "commercial",
      "square_footage": 15000,
      "lot_size_acres": 0.5,
      "floors": 3,
      "basement": true,
      "crawl_space": true,
      "access_notes": "Use loading dock entrance on Elm St",
      "gate_code": "4567",
      "alarm_code_location": "Check customer notes",
      "is_primary": true,
      "pest_history": ["german_cockroach", "ant_carpenter", "mouse"],
      "last_inspection_date": "2026-04-15",
      "ipm_zone_map_url": "https://..."
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_customer ON service_locations(customer_id);
CREATE INDEX idx_locations_geo ON service_locations USING GIST(coordinates);
CREATE INDEX idx_locations_details ON service_locations USING GIN(details jsonb_path_ops);
```

## Jobs & Appointments (Combined)

```sql
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    job_number VARCHAR(50),
    service_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    -- Scheduling (embedded — avoids separate appointments table for simple cases)
    scheduled_start TIMESTAMPTZ,
    scheduled_end TIMESTAMPTZ,
    arrival_window_start TIMESTAMPTZ,
    arrival_window_end TIMESTAMPTZ,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    recurrence_rule VARCHAR(255),  -- RFC 5545 RRULE
    parent_job_id UUID REFERENCES jobs(id),
    -- Assignment
    technician_ids UUID[] NOT NULL DEFAULT '{}',
    lead_technician_id UUID REFERENCES users(id),
    -- Pricing
    price_cents INTEGER,
    -- Service report (embedded — avoids separate service_reports table)
    service_report JSONB,
    /*
    service_report example:
    {
      "pests_found": ["german_cockroach", "ant_odorous"],
      "conditions_observed": "Moisture under kitchen sink, crumbs in pantry",
      "treatment_summary": "Applied gel bait in kitchen, exterior perimeter spray",
      "recommendations": "Seal gap under garage door. Fix leaking pipe under kitchen sink.",
      "follow_up_required": true,
      "follow_up_date": "2026-06-22",
      "ipm_approach_used": true,
      "ipm_checklist": {
        "inspection_completed": true,
        "prevention_recommendations_given": true,
        "minimum_chemical_approach": true,
        "monitoring_devices_checked": true
      },
      "customer_signature_url": "https://...",
      "customer_signed_at": "2026-05-22T10:05:00Z",
      "photos": [
        { "url": "https://...", "caption": "Evidence under sink", "type": "evidence" },
        { "url": "https://...", "caption": "After treatment", "type": "after" }
      ]
    }
    */
    -- Flexible job metadata
    extra JSONB NOT NULL DEFAULT '{}',
    /*
    extra example:
    {
      "contract_id": "uuid",
      "notes": "Customer prefers back door entry",
      "custom_fields": {
        "ipm_checklist_completed": true,
        "property_condition_rating": 3
      }
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_tenant_status ON jobs(tenant_id, status);
CREATE INDEX idx_jobs_schedule ON jobs(tenant_id, scheduled_start);
CREATE INDEX idx_jobs_customer ON jobs(customer_id);
CREATE INDEX idx_jobs_location ON jobs(service_location_id);
CREATE INDEX idx_jobs_technicians ON jobs USING GIN(technician_ids);
CREATE INDEX idx_jobs_service_report ON jobs USING GIN(service_report jsonb_path_ops);
```

## Chemical Applications (Compliance-Critical — Fully Typed)

```sql
-- This table has NO JSONB flexibility on compliance-critical fields.
-- EPA-mandated fields are all typed columns with NOT NULL constraints.
-- Only supplementary/jurisdiction-specific data uses JSONB.

CREATE TABLE pesticide_products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epa_registration_number VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(500) NOT NULL,
    registrant_name VARCHAR(255),
    signal_word VARCHAR(20),
    restricted_use BOOLEAN NOT NULL DEFAULT false,
    active_ingredients TEXT[],
    -- Everything else from PPLS goes into JSONB
    ppls_data JSONB NOT NULL DEFAULT '{}',
    /*
    ppls_data example:
    {
      "formulation_code": "EC",
      "toxicity_category": "III",
      "target_pests": ["cockroach", "ant", "spider"],
      "use_sites": ["structural", "perimeter"],
      "re_entry_interval_hours": 2,
      "dot_hazmat_class": null,
      "label_url": "https://...",
      "last_synced_at": "2026-05-22T00:00:00Z"
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_epa ON pesticide_products(epa_registration_number);

CREATE TABLE chemical_applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    job_id UUID NOT NULL REFERENCES jobs(id),
    pesticide_product_id UUID NOT NULL REFERENCES pesticide_products(id),
    technician_id UUID NOT NULL REFERENCES users(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    -- All EPA-mandated fields as typed columns (NOT NULL where required)
    epa_registration_number VARCHAR(30) NOT NULL,
    application_date DATE NOT NULL,
    application_time TIME NOT NULL,
    target_pests TEXT[] NOT NULL,
    application_site VARCHAR(255) NOT NULL,
    application_method VARCHAR(50) NOT NULL,
    quantity_applied NUMERIC(10,3) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL,
    dilution_rate VARCHAR(50),
    concentration_pct NUMERIC(5,2),
    re_entry_interval_hours NUMERIC(6,1),
    restricted_use_product BOOLEAN NOT NULL DEFAULT false,
    applicator_licence_number VARCHAR(50),
    -- State-specific and supplementary data in JSONB
    state_compliance_data JSONB NOT NULL DEFAULT '{}',
    /*
    state_compliance_data examples by state:

    Illinois:
    {
      "state": "IL",
      "applicator_category": "7A",
      "business_licence_number": "IL-BUS-9876",
      "wind_speed_mph": 5.2,
      "temperature_f": 72,
      "notification_required": false
    }

    California:
    {
      "state": "CA",
      "county": "Los Angeles",
      "dpr_permit_number": "CA-LA-2026-0042",
      "use_report_filed": false,
      "use_report_due_date": "2026-06-10",
      "fumigation_management_plan": null,
      "wind_speed_mph": 3.1,
      "temperature_f": 78,
      "notification_distance_ft": 100,
      "neighbors_notified": true
    }

    New York:
    {
      "state": "NY",
      "dec_registration_number": "NY-DEC-2026-0015",
      "school_proximity_flag": false,
      "48hr_notice_posted": true,
      "wind_speed_mph": 8.0,
      "temperature_f": 65
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chem_apps_job ON chemical_applications(job_id);
CREATE INDEX idx_chem_apps_date ON chemical_applications(tenant_id, application_date);
CREATE INDEX idx_chem_apps_product ON chemical_applications(pesticide_product_id);
CREATE INDEX idx_chem_apps_technician ON chemical_applications(technician_id);
CREATE INDEX idx_chem_apps_state ON chemical_applications USING GIN(state_compliance_data jsonb_path_ops);
```

## Devices & Traps

```sql
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    -- Core fields always present
    barcode VARCHAR(100),
    device_category VARCHAR(50) NOT NULL,
        -- bait_station, snap_trap, glue_board, pheromone_trap, electronic_trap, monitoring_station
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    installed_date DATE NOT NULL,
    -- Flexible device details (manufacturer, model, placement vary widely)
    details JSONB NOT NULL DEFAULT '{}',
    /*
    details example:
    {
      "device_type_name": "Protecta LP Rat Station",
      "manufacturer": "Bell Laboratories",
      "model_number": "LP-100",
      "placement_description": "Exterior - NE corner of building",
      "coordinates": { "type": "Point", "coordinates": [-89.6502, 39.7818] },
      "bait_type": "Contrac Blox",
      "bait_epa_number": "12455-86",
      "check_interval_days": 30,
      "last_checked_date": "2026-05-01",
      "next_check_date": "2026-05-31",
      "iot_sensor_id": "SENS-2026-042",
      "iot_last_reading": { "timestamp": "2026-05-22T03:00:00Z", "triggered": false, "battery_pct": 87 }
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_location ON devices(service_location_id);
CREATE INDEX idx_devices_barcode ON devices(tenant_id, barcode);
CREATE INDEX idx_devices_details ON devices USING GIN(details jsonb_path_ops);

CREATE TABLE device_inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    device_id UUID NOT NULL REFERENCES devices(id),
    job_id UUID REFERENCES jobs(id),
    technician_id UUID NOT NULL REFERENCES users(id),
    inspection_date TIMESTAMPTZ NOT NULL,
    activity_level VARCHAR(20) NOT NULL,
    action_taken VARCHAR(50),
    -- Flexible inspection data
    findings JSONB NOT NULL DEFAULT '{}',
    /*
    findings example:
    {
      "notes": "Fresh droppings near station, bait consumption approximately 30%",
      "bait_consumption_pct": 30,
      "pest_evidence": ["droppings", "gnaw_marks"],
      "photos": ["https://...", "https://..."],
      "bait_replaced": true,
      "new_bait_type": "Contrac Blox",
      "device_condition": "good",
      "tamper_evident_intact": true
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_device ON device_inspections(device_id);
CREATE INDEX idx_inspections_date ON device_inspections(tenant_id, inspection_date);
```

## Routes

```sql
CREATE TABLE routes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    technician_id UUID NOT NULL REFERENCES users(id),
    route_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'planned',
    -- Route-level metrics
    total_distance_meters INTEGER,
    total_drive_time_seconds INTEGER,
    optimised_at TIMESTAMPTZ,
    -- Stops embedded as JSONB array (avoids separate route_stops table)
    stops JSONB NOT NULL DEFAULT '[]',
    /*
    stops example:
    [
      {
        "stop_order": 1,
        "job_id": "uuid",
        "customer_name": "Smith, John",
        "address": "123 Main St, Springfield, IL 62701",
        "coordinates": { "type": "Point", "coordinates": [-89.6501, 39.7817] },
        "estimated_arrival": "2026-05-22T08:30:00Z",
        "estimated_departure": "2026-05-22T09:15:00Z",
        "actual_arrival": null,
        "actual_departure": null,
        "drive_distance_meters": 5200,
        "drive_time_seconds": 480
      },
      {
        "stop_order": 2,
        "job_id": "uuid",
        "customer_name": "Garcia, Maria",
        "address": "456 Oak Ave, Springfield, IL 62702",
        "coordinates": { "type": "Point", "coordinates": [-89.6550, 39.7830] },
        "estimated_arrival": "2026-05-22T09:25:00Z",
        "estimated_departure": "2026-05-22T10:10:00Z",
        "actual_arrival": null,
        "actual_departure": null,
        "drive_distance_meters": 1800,
        "drive_time_seconds": 240
      }
    ]
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routes_date ON routes(tenant_id, route_date);
CREATE INDEX idx_routes_technician ON routes(technician_id, route_date);
```

## Billing

```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    job_id UUID REFERENCES jobs(id),
    invoice_number VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    amount_paid_cents INTEGER NOT NULL DEFAULT 0,
    issued_date DATE,
    due_date DATE,
    -- Line items and payments embedded as JSONB
    line_items JSONB NOT NULL DEFAULT '[]',
    /*
    line_items example:
    [
      { "description": "Quarterly Pest Treatment", "quantity": 1, "unit_price_cents": 15000, "total_cents": 15000 },
      { "description": "Additional bait station installation (x2)", "quantity": 2, "unit_price_cents": 2500, "total_cents": 5000 }
    ]
    */
    payments JSONB NOT NULL DEFAULT '[]',
    /*
    payments example:
    [
      {
        "payment_id": "uuid",
        "amount_cents": 20000,
        "payment_method": "credit_card",
        "processor": "stripe",
        "processor_transaction_id": "ch_xxx",
        "payment_date": "2026-05-22T14:30:00Z",
        "status": "completed"
      }
    ]
    */
    extra JSONB NOT NULL DEFAULT '{}',
    /*
    extra example:
    {
      "quickbooks_id": "QBO-INV-456",
      "quickbooks_sync_status": "synced",
      "discount_cents": 0,
      "discount_reason": null,
      "notes": "Thank you for your business!"
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(tenant_id, status);
```

## Communications & Audit

```sql
CREATE TABLE communications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    channel VARCHAR(20) NOT NULL,
    direction VARCHAR(10) NOT NULL,
    subject VARCHAR(255),
    body TEXT,
    sent_at TIMESTAMPTZ,
    -- Flexible metadata
    meta JSONB NOT NULL DEFAULT '{}',
    /*
    meta example:
    {
      "template_name": "appointment_reminder",
      "related_job_id": "uuid",
      "delivery_status": "delivered",
      "opened_at": "2026-05-21T10:00:00Z",
      "clicked_at": null,
      "review_platform": "google",
      "review_rating": 5
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comms_customer ON communications(customer_id);
CREATE INDEX idx_comms_date ON communications(tenant_id, sent_at);

CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL,
    changes JSONB,  -- { "field": { "old": "...", "new": "..." } }
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_date ON audit_logs(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id);
```

## Fleet (Lightweight)

```sql
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    vehicle_number VARCHAR(50),
    make VARCHAR(100),
    model VARCHAR(100),
    year INTEGER,
    licence_plate VARCHAR(20),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    -- All other vehicle data in JSONB
    details JSONB NOT NULL DEFAULT '{}',
    /*
    details example:
    {
      "vin": "1HGBH41JXMN109186",
      "dot_hazmat_placard": false,
      "current_odometer": 45230,
      "insurance_expiry": "2027-01-15",
      "inspection_expiry": "2026-12-01",
      "assigned_technician_id": "uuid",
      "chemical_load": [
        { "product": "Advion Cockroach Gel", "epa_number": "100-1066", "quantity": 5, "unit": "tubes" },
        { "product": "Suspend SC", "epa_number": "432-763", "quantity": 1, "unit": "gal" }
      ]
    }
    */
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicles_tenant ON vehicles(tenant_id);
```

## JSONB Query Examples

```sql
-- Find all customers tagged as "vip"
SELECT * FROM customers
WHERE tenant_id = $1
  AND extra @> '{"tags": ["vip"]}';

-- Find technicians with certifications expiring within 90 days
SELECT id, first_name, last_name,
       jsonb_array_elements(profile->'certifications') AS cert
FROM users
WHERE tenant_id = $1
  AND role = 'technician'
  AND EXISTS (
    SELECT 1 FROM jsonb_array_elements(profile->'certifications') AS c
    WHERE (c->>'expiry_date')::date < now() + interval '90 days'
      AND c->>'status' = 'active'
  );

-- Find chemical applications in California that haven't filed use reports
SELECT ca.*, ca.state_compliance_data->>'dpr_permit_number' AS permit
FROM chemical_applications ca
WHERE ca.tenant_id = $1
  AND ca.state_compliance_data->>'state' = 'CA'
  AND (ca.state_compliance_data->>'use_report_filed')::boolean = false;

-- Find all devices at a location with IoT sensors
SELECT d.*, d.details->>'iot_sensor_id' AS sensor_id
FROM devices d
WHERE d.service_location_id = $1
  AND d.details ? 'iot_sensor_id';

-- Get route stops for a specific date with actual arrival times
SELECT r.route_date,
       stop->>'customer_name' AS customer,
       stop->>'address' AS address,
       stop->>'actual_arrival' AS arrived_at,
       stop->>'stop_order' AS stop_num
FROM routes r,
     jsonb_array_elements(r.stops) AS stop
WHERE r.tenant_id = $1
  AND r.route_date = '2026-05-22';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenants (with config JSONB), users (with profile JSONB) |
| Customer & Location | 2 | customers (with extra JSONB), service_locations (with details JSONB) |
| Jobs & Scheduling | 1 | jobs (with embedded service report, scheduling, and assignment) |
| Chemical & Pesticide | 2 | pesticide_products, chemical_applications (with state_compliance_data JSONB) |
| Devices & Traps | 2 | devices (with details JSONB), device_inspections (with findings JSONB) |
| Routes | 1 | routes (with stops JSONB array) |
| Billing | 1 | invoices (with line_items and payments JSONB arrays) |
| Communications | 1 | communications (with meta JSONB) |
| Fleet | 1 | vehicles (with details JSONB) |
| Audit | 1 | audit_logs |
| **Total** | **14** | |

---

## Key Design Decisions

1. **Typed columns for compliance, JSONB for everything else** — the chemical_applications table is the most strictly typed in the schema because its fields are legally mandated by EPA/FIFRA. Every other entity uses JSONB for variable data. This is a deliberate asymmetry: compliance data gets database-enforced constraints; operational data gets flexibility.

2. **Tenant configuration in JSONB, not separate tables** — custom field definitions, billing settings, scheduling defaults, and branding are all stored in `tenants.config`. This eliminates 3-5 settings tables and makes tenant onboarding a single INSERT with a JSON payload.

3. **Certifications as JSONB array on user profile** — avoids a separate `technician_certifications` table. Each certification is a JSON object in the array. Trade-off: no foreign key referential integrity on certifications, but they are self-contained data that does not reference other entities.

4. **Service report embedded in jobs** — the service report (pests found, treatment summary, photos, customer signature) is stored as a JSONB column on the jobs table rather than a separate table. This eliminates a JOIN for the most common read path (viewing a completed job's report) and keeps all job data in one row.

5. **Route stops as JSONB array** — a single route rarely has more than 15 stops. Storing them as a JSONB array on the route row avoids a separate route_stops table and eliminates N+1 queries when loading a route. The route optimiser can replace the entire stops array atomically.

6. **Invoice line items and payments as JSONB** — invoices rarely have more than 10 line items or 2-3 payments. Embedding them avoids two separate tables and simplifies the common read path (display a full invoice). The trade-off is that aggregate queries across line items (e.g., "total revenue by service type") require JSONB extraction.

7. **State-specific compliance data in JSONB** — the `state_compliance_data` column on chemical_applications handles the significant variation between US state requirements. California requires DPR permit numbers and use-report filing; New York requires DEC registration and school-proximity flags; most other states have simpler requirements. JSONB absorbs this variation without nullable columns.

8. **GIN indexes on all JSONB columns** — every JSONB column has a `jsonb_path_ops` GIN index to support fast containment queries (`@>` operator). This provides near-relational query performance for JSONB data.

9. **14 tables total** — roughly half the count of the normalised model. This reduces migration complexity, simplifies the ORM layer, and makes the schema easier for new developers to learn. The trade-off is that some queries require JSONB extraction functions rather than simple column references.

10. **JSON Schema validation at the application layer** — since JSONB columns lack database-enforced type constraints, the application layer validates JSONB payloads against JSON Schema definitions before writing. This provides type safety without database-level enforcement.
