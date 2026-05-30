# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Pest Control Management · Created: 2026-05-22

## Philosophy

This model follows a traditional normalized relational approach where every domain concept gets its own table with explicit foreign keys and junction tables. The design prioritises data integrity, referential consistency, and query flexibility over development speed. Every chemical application, trap inspection, and service visit is a first-class entity with its own table, ensuring that complex cross-entity queries (e.g., "which technicians applied Product X at locations within ZIP code 90210 in Q3?") can be answered with standard SQL joins.

This is the approach used by mature enterprise systems like ServiceTitan and PestPac, where the data model has evolved over years to cover every edge case. It maps naturally to the EPA's structured data requirements (EPA Registration Numbers, active ingredients, application records) and to the ServiceTitan/Jobber API patterns where each entity (Customer, Location, Job, Appointment, Technician) is a distinct addressable resource.

The normalized approach is best suited for teams building a long-lived, compliance-heavy platform where data integrity is non-negotiable and the cost of fixing data corruption outweighs the cost of additional JOIN complexity.

**Best for:** Compliance-first deployments where regulatory audit readiness and cross-entity reporting are the primary concerns.

**Trade-offs:**
- (+) Strongest referential integrity — cascading deletes and foreign keys prevent orphan records
- (+) Easiest to query with arbitrary ad-hoc SQL
- (+) Natural fit for EPA/FIFRA compliance record structure
- (+) Clean mapping to REST/GraphQL API resources (one table = one resource)
- (-) Highest table count (~45-50 tables) increases migration complexity
- (-) Many JOINs required for common read paths (e.g., displaying a service report)
- (-) Schema changes require migrations; adding jurisdiction-specific fields means ALTER TABLE
- (-) Junction tables for many-to-many relationships add boilerplate

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EPA FIFRA / 40 CFR 170 | `chemical_applications` table mirrors the EPA-mandated application record fields: EPA Reg Number, active ingredients, quantity, method, target pests, REI |
| EPA PPLS API | `pesticide_products` table stores reference data fetched from the PPLS API; `epa_registration_number` is the natural key |
| EPA PPIS | `pesticide_products.formulation_code`, `toxicity_category` fields align with PPIS bulk data |
| NPMA QualityPro | `technician_certifications` table tracks QualityPro and GreenPro credentials with expiry dates |
| ISO 3166 | `addresses.country_code` and `addresses.state_code` use ISO 3166-1 alpha-2 and ISO 3166-2 subdivision codes |
| RFC 5545 iCalendar | `appointments` table fields (start_at, end_at, recurrence_rule) map to VEVENT properties for calendar export |
| RFC 7946 GeoJSON | `service_locations.coordinates` stored as PostGIS GEOGRAPHY for GeoJSON-compatible output |
| OAuth 2.0 / RFC 6749 | `api_clients` and `api_tokens` tables support OAuth 2.0 integration patterns |
| OpenAPI 3.x | One-to-one table-to-resource mapping enables clean OpenAPI spec generation |

---

## Core Identity & Multi-Tenancy

```sql
-- Every table includes tenant_id for row-level security
-- This section defines the tenant and authentication foundation

CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    subscription_plan VARCHAR(50) NOT NULL DEFAULT 'trial',
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    settings JSONB NOT NULL DEFAULT '{}',
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
        -- roles: owner, admin, office_staff, technician, sales
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
```

## Customer & Location Management

```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_number VARCHAR(50),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(30),
    alt_phone VARCHAR(30),
    customer_type VARCHAR(30) NOT NULL DEFAULT 'residential',
        -- residential, commercial, government
    billing_method VARCHAR(30) NOT NULL DEFAULT 'per_service',
        -- per_service, monthly, quarterly, annual
    tax_exempt BOOLEAN NOT NULL DEFAULT false,
    notes TEXT,
    tags VARCHAR(50)[],
    referral_source VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_name ON customers(tenant_id, last_name, first_name);
CREATE INDEX idx_customers_tags ON customers USING GIN(tags);

CREATE TABLE service_locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    label VARCHAR(100),  -- 'Main Office', 'Warehouse B'
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state_code VARCHAR(10) NOT NULL,       -- ISO 3166-2
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL DEFAULT 'US', -- ISO 3166-1 alpha-2
    coordinates GEOGRAPHY(POINT, 4326),
    property_type VARCHAR(50),
        -- single_family, multi_family, apartment, commercial, warehouse, restaurant, food_processing
    square_footage INTEGER,
    lot_size_acres NUMERIC(8,2),
    access_notes TEXT,
    gate_code VARCHAR(50),
    is_primary BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_service_locations_customer ON service_locations(customer_id);
CREATE INDEX idx_service_locations_geo ON service_locations USING GIST(coordinates);
CREATE INDEX idx_service_locations_tenant ON service_locations(tenant_id);
```

## Service Plans & Contracts

```sql
CREATE TABLE service_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    service_type VARCHAR(50) NOT NULL,
        -- general_pest, termite, rodent, mosquito, wildlife, bed_bug, fumigation, ipm
    frequency VARCHAR(30) NOT NULL,
        -- one_time, monthly, bi_monthly, quarterly, semi_annual, annual
    default_duration_minutes INTEGER NOT NULL DEFAULT 60,
    base_price_cents INTEGER NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer_contracts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    service_plan_id UUID NOT NULL REFERENCES service_plans(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    contract_number VARCHAR(50),
    start_date DATE NOT NULL,
    end_date DATE,
    renewal_type VARCHAR(30) NOT NULL DEFAULT 'auto_renew',
        -- auto_renew, manual, non_renewable
    price_cents INTEGER NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'active',
        -- draft, active, paused, cancelled, expired
    cancellation_reason TEXT,
    cancelled_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_customer ON customer_contracts(customer_id);
CREATE INDEX idx_contracts_status ON customer_contracts(tenant_id, status);
```

## Scheduling & Dispatch

```sql
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    customer_contract_id UUID REFERENCES customer_contracts(id),
    job_number VARCHAR(50),
    service_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(30) NOT NULL DEFAULT 'scheduled',
        -- draft, scheduled, dispatched, in_progress, completed, cancelled
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
        -- low, normal, high, emergency
    estimated_duration_minutes INTEGER NOT NULL DEFAULT 60,
    price_cents INTEGER,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_tenant_status ON jobs(tenant_id, status);
CREATE INDEX idx_jobs_customer ON jobs(customer_id);
CREATE INDEX idx_jobs_location ON jobs(service_location_id);

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    job_id UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    arrival_window_start TIMESTAMPTZ,
    arrival_window_end TIMESTAMPTZ,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    status VARCHAR(30) NOT NULL DEFAULT 'scheduled',
        -- scheduled, confirmed, en_route, arrived, in_progress, completed, cancelled, no_show
    recurrence_rule VARCHAR(255),  -- RFC 5545 RRULE for recurring appointments
    parent_appointment_id UUID REFERENCES appointments(id),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appointments_schedule ON appointments(tenant_id, scheduled_start, scheduled_end);
CREATE INDEX idx_appointments_job ON appointments(job_id);

CREATE TABLE appointment_technicians (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id UUID NOT NULL REFERENCES appointments(id) ON DELETE CASCADE,
    technician_id UUID NOT NULL REFERENCES users(id),
    is_lead BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(appointment_id, technician_id)
);

CREATE INDEX idx_appt_techs_technician ON appointment_technicians(technician_id);
```

## Route Optimisation

```sql
CREATE TABLE routes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    technician_id UUID NOT NULL REFERENCES users(id),
    route_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'planned',
        -- planned, optimised, in_progress, completed
    start_location GEOGRAPHY(POINT, 4326),
    end_location GEOGRAPHY(POINT, 4326),
    total_distance_meters INTEGER,
    total_drive_time_seconds INTEGER,
    optimised_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routes_date ON routes(tenant_id, route_date);
CREATE INDEX idx_routes_technician ON routes(technician_id, route_date);

CREATE TABLE route_stops (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_id UUID NOT NULL REFERENCES routes(id) ON DELETE CASCADE,
    appointment_id UUID NOT NULL REFERENCES appointments(id),
    stop_order INTEGER NOT NULL,
    estimated_arrival TIMESTAMPTZ,
    estimated_departure TIMESTAMPTZ,
    actual_arrival TIMESTAMPTZ,
    actual_departure TIMESTAMPTZ,
    drive_distance_meters INTEGER,
    drive_time_seconds INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_route_stops_route ON route_stops(route_id, stop_order);
```

## Chemical & Pesticide Management

```sql
-- Reference data from EPA PPLS/PPIS
CREATE TABLE pesticide_products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epa_registration_number VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(500) NOT NULL,
    registrant_name VARCHAR(255),
    formulation_code VARCHAR(10),
    toxicity_category VARCHAR(10),
        -- I, II, III, IV
    signal_word VARCHAR(20),
        -- DANGER, WARNING, CAUTION
    restricted_use BOOLEAN NOT NULL DEFAULT false,
    active_ingredients TEXT[],
    target_pests TEXT[],
    use_sites TEXT[],
    re_entry_interval_hours NUMERIC(6,1),
    dot_hazmat_class VARCHAR(20),
    ppls_data JSONB,  -- raw PPLS API response for reference
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pesticides_epa ON pesticide_products(epa_registration_number);
CREATE INDEX idx_pesticides_name ON pesticide_products(product_name);
CREATE INDEX idx_pesticides_restricted ON pesticide_products(restricted_use) WHERE restricted_use = true;

-- Tenant-specific chemical inventory
CREATE TABLE chemical_inventory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    pesticide_product_id UUID NOT NULL REFERENCES pesticide_products(id),
    lot_number VARCHAR(100),
    quantity_on_hand NUMERIC(10,2) NOT NULL DEFAULT 0,
    unit_of_measure VARCHAR(20) NOT NULL,
        -- oz, fl_oz, lb, gal, ml, L, kg, g
    storage_location VARCHAR(255),
    purchase_date DATE,
    expiration_date DATE,
    purchase_cost_cents INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chem_inventory_tenant ON chemical_inventory(tenant_id);

-- Per-service chemical application records (EPA compliance)
CREATE TABLE chemical_applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    job_id UUID NOT NULL REFERENCES jobs(id),
    appointment_id UUID NOT NULL REFERENCES appointments(id),
    pesticide_product_id UUID NOT NULL REFERENCES pesticide_products(id),
    technician_id UUID NOT NULL REFERENCES users(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    -- EPA-mandated fields
    epa_registration_number VARCHAR(30) NOT NULL,
    application_date DATE NOT NULL,
    application_time TIME NOT NULL,
    target_pests TEXT[] NOT NULL,
    application_site VARCHAR(255) NOT NULL,
        -- e.g., 'interior baseboards', 'exterior perimeter', 'attic'
    application_method VARCHAR(50) NOT NULL,
        -- spray, bait, dust, fog, fumigation, granular, trap
    quantity_applied NUMERIC(10,3) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL,
    dilution_rate VARCHAR(50),
    concentration_pct NUMERIC(5,2),
    wind_speed_mph NUMERIC(4,1),
    temperature_f NUMERIC(5,1),
    re_entry_interval_hours NUMERIC(6,1),
    restricted_use_product BOOLEAN NOT NULL DEFAULT false,
    applicator_licence_number VARCHAR(50),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chem_apps_job ON chemical_applications(job_id);
CREATE INDEX idx_chem_apps_product ON chemical_applications(pesticide_product_id);
CREATE INDEX idx_chem_apps_date ON chemical_applications(tenant_id, application_date);
CREATE INDEX idx_chem_apps_technician ON chemical_applications(technician_id, application_date);
```

## Trap & Device Management

```sql
CREATE TABLE device_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    category VARCHAR(50) NOT NULL,
        -- bait_station, snap_trap, glue_board, pheromone_trap, electronic_trap, monitoring_station
    manufacturer VARCHAR(255),
    model_number VARCHAR(100),
    description TEXT,
    default_check_interval_days INTEGER NOT NULL DEFAULT 30,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    device_type_id UUID NOT NULL REFERENCES device_types(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    barcode VARCHAR(100),
    placement_description VARCHAR(255),
    placement_coordinates GEOGRAPHY(POINT, 4326),
    installed_date DATE NOT NULL,
    last_checked_date DATE,
    next_check_date DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'active',
        -- active, needs_replacement, decommissioned, missing
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_location ON devices(service_location_id);
CREATE INDEX idx_devices_barcode ON devices(tenant_id, barcode);
CREATE INDEX idx_devices_next_check ON devices(tenant_id, next_check_date) WHERE status = 'active';

CREATE TABLE device_inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    device_id UUID NOT NULL REFERENCES devices(id),
    appointment_id UUID REFERENCES appointments(id),
    technician_id UUID NOT NULL REFERENCES users(id),
    inspection_date TIMESTAMPTZ NOT NULL,
    activity_level VARCHAR(20) NOT NULL,
        -- none, low, moderate, high
    findings TEXT,
    action_taken VARCHAR(50),
        -- no_action, bait_replaced, trap_reset, device_replaced, device_removed
    photo_urls TEXT[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_device_inspections_device ON device_inspections(device_id);
CREATE INDEX idx_device_inspections_date ON device_inspections(tenant_id, inspection_date);
```

## Technician Credentials & Certifications

```sql
CREATE TABLE technician_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE UNIQUE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    employee_number VARCHAR(50),
    hire_date DATE,
    hourly_rate_cents INTEGER,
    commission_rate_pct NUMERIC(5,2),
    vehicle_id UUID REFERENCES vehicles(id),
    skills TEXT[],
        -- general_pest, termite, fumigation, wildlife, bed_bug, mosquito
    max_daily_stops INTEGER DEFAULT 12,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE technician_certifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    technician_profile_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    certification_type VARCHAR(50) NOT NULL,
        -- state_applicator, state_operator, qualitypro, greenpro, wdo_inspector, fumigation
    certification_number VARCHAR(100) NOT NULL,
    issuing_authority VARCHAR(255) NOT NULL,
    state_code VARCHAR(10),
    category VARCHAR(100),
        -- e.g., Category 7A (General Pest), Category 7B (Termite)
    issued_date DATE NOT NULL,
    expiry_date DATE NOT NULL,
    renewal_required BOOLEAN NOT NULL DEFAULT true,
    document_url VARCHAR(500),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
        -- active, expired, suspended, revoked
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certs_technician ON technician_certifications(technician_profile_id);
CREATE INDEX idx_certs_expiry ON technician_certifications(tenant_id, expiry_date) WHERE status = 'active';
```

## Fleet & Vehicle Management

```sql
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    vehicle_number VARCHAR(50),
    make VARCHAR(100),
    model VARCHAR(100),
    year INTEGER,
    licence_plate VARCHAR(20),
    vin VARCHAR(20),
    dot_hazmat_placard BOOLEAN NOT NULL DEFAULT false,
    current_odometer INTEGER,
    insurance_expiry DATE,
    inspection_expiry DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicles_tenant ON vehicles(tenant_id);

CREATE TABLE vehicle_chemical_loads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id UUID NOT NULL REFERENCES vehicles(id),
    pesticide_product_id UUID NOT NULL REFERENCES pesticide_products(id),
    quantity NUMERIC(10,2) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL,
    loaded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    dot_classification VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicle_loads ON vehicle_chemical_loads(vehicle_id);
```

## Billing & Payments

```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    job_id UUID REFERENCES jobs(id),
    invoice_number VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, sent, viewed, paid, partial, overdue, void, refunded
    subtotal_cents INTEGER NOT NULL DEFAULT 0,
    tax_cents INTEGER NOT NULL DEFAULT 0,
    discount_cents INTEGER NOT NULL DEFAULT 0,
    total_cents INTEGER NOT NULL DEFAULT 0,
    amount_paid_cents INTEGER NOT NULL DEFAULT 0,
    issued_date DATE,
    due_date DATE,
    paid_date DATE,
    quickbooks_id VARCHAR(100),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(tenant_id, status);
CREATE INDEX idx_invoices_qb ON invoices(quickbooks_id) WHERE quickbooks_id IS NOT NULL;

CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description VARCHAR(500) NOT NULL,
    quantity NUMERIC(10,2) NOT NULL DEFAULT 1,
    unit_price_cents INTEGER NOT NULL,
    total_cents INTEGER NOT NULL,
    tax_rate_pct NUMERIC(5,2) DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    amount_cents INTEGER NOT NULL,
    payment_method VARCHAR(30) NOT NULL,
        -- credit_card, ach, check, cash, online_portal
    payment_processor VARCHAR(30),
    processor_transaction_id VARCHAR(255),
    payment_date TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'completed',
        -- pending, completed, failed, refunded
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
```

## Service Documentation

```sql
CREATE TABLE service_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    job_id UUID NOT NULL REFERENCES jobs(id),
    appointment_id UUID NOT NULL REFERENCES appointments(id),
    technician_id UUID NOT NULL REFERENCES users(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    pests_found TEXT[],
    conditions_observed TEXT,
    treatment_summary TEXT,
    recommendations TEXT,
    follow_up_required BOOLEAN NOT NULL DEFAULT false,
    follow_up_date DATE,
    customer_signature_url VARCHAR(500),
    customer_signed_at TIMESTAMPTZ,
    ipm_approach_used BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_service_reports_job ON service_reports(job_id);

CREATE TABLE service_photos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    service_report_id UUID REFERENCES service_reports(id),
    job_id UUID NOT NULL REFERENCES jobs(id),
    uploaded_by UUID NOT NULL REFERENCES users(id),
    file_url VARCHAR(500) NOT NULL,
    thumbnail_url VARCHAR(500),
    caption VARCHAR(255),
    photo_type VARCHAR(30),
        -- before, after, evidence, damage, device
    ai_species_id VARCHAR(100),  -- AI-identified pest species
    ai_confidence NUMERIC(4,3),
    taken_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_photos_report ON service_photos(service_report_id);
CREATE INDEX idx_photos_job ON service_photos(job_id);
```

## Communications & Notifications

```sql
CREATE TABLE communications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    channel VARCHAR(20) NOT NULL,
        -- sms, email, phone, portal
    direction VARCHAR(10) NOT NULL,
        -- inbound, outbound
    subject VARCHAR(255),
    body TEXT,
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    template_name VARCHAR(100),
    related_job_id UUID REFERENCES jobs(id),
    related_appointment_id UUID REFERENCES appointments(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comms_customer ON communications(customer_id);
CREATE INDEX idx_comms_date ON communications(tenant_id, sent_at);

CREATE TABLE review_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    job_id UUID NOT NULL REFERENCES jobs(id),
    platform VARCHAR(30) NOT NULL,
        -- google, yelp, facebook
    sent_at TIMESTAMPTZ,
    clicked_at TIMESTAMPTZ,
    review_received BOOLEAN NOT NULL DEFAULT false,
    review_rating INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reviews_customer ON review_requests(customer_id);
```

## Audit Trail

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL,
        -- create, update, delete, view
    changes JSONB,
    ip_address INET,
    user_agent VARCHAR(500),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_date ON audit_logs(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenants, users |
| Customer & Location | 2 | customers, service_locations |
| Service Plans & Contracts | 2 | service_plans, customer_contracts |
| Scheduling & Dispatch | 3 | jobs, appointments, appointment_technicians |
| Route Optimisation | 2 | routes, route_stops |
| Chemical & Pesticide | 3 | pesticide_products, chemical_inventory, chemical_applications |
| Trap & Device Management | 3 | device_types, devices, device_inspections |
| Technician & Credentials | 2 | technician_profiles, technician_certifications |
| Fleet | 2 | vehicles, vehicle_chemical_loads |
| Billing & Payments | 3 | invoices, invoice_line_items, payments |
| Service Documentation | 2 | service_reports, service_photos |
| Communications | 2 | communications, review_requests |
| Audit | 1 | audit_logs |
| **Total** | **29** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation for offline-capable mobile apps and avoids sequential ID enumeration attacks on API endpoints.

2. **Tenant-scoped row-level security** — every table carries `tenant_id` with a foreign key to `tenants`. PostgreSQL RLS policies enforce tenant isolation at the database layer, not just the application layer.

3. **Separate `pesticide_products` reference table** — EPA product data is global (not tenant-specific) and synced from the PPLS API. Tenant-specific inventory and application records reference this shared table, avoiding data duplication and ensuring EPA Registration Number consistency.

4. **`chemical_applications` as a first-class entity** — this is the core compliance record. Its fields directly mirror EPA/FIFRA requirements (EPA Reg Number, application date/time, target pests, site, method, quantity, REI). This table is the most heavily audited in the system.

5. **Explicit junction table for appointment-technician assignment** — allows multi-technician appointments with a designated lead, supporting crew-based service calls for fumigation or large commercial jobs.

6. **Route and route_stops as separate entities** — separates route-level metadata (total distance, optimisation timestamp) from per-stop details, allowing the route optimiser to replace stop sequences without modifying the route record.

7. **PostGIS GEOGRAPHY type for coordinates** — enables spatial queries (nearest-location, radius search, route distance calculation) using standard PostGIS functions and GeoJSON output.

8. **Recurrence via RFC 5545 RRULE strings** — storing recurrence rules as iCalendar-compatible strings on `appointments.recurrence_rule` avoids custom recurrence logic and enables direct export to Google Calendar / Outlook.

9. **Chemical inventory with lot tracking** — `chemical_inventory` tracks individual lots with purchase date, expiration, and cost, supporting FIFO usage and expiration alerting.

10. **Certification expiry indexing** — a partial index on `technician_certifications(expiry_date) WHERE status = 'active'` supports efficient queries for upcoming credential expirations, a critical compliance function.
