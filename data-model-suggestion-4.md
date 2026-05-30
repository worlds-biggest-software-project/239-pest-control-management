# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Pest Control Management · Created: 2026-05-22

## Philosophy

This model combines a conventional relational core for operational CRUD (scheduling, billing, chemical tracking) with a property graph layer for relationship-intensive queries. The graph layer uses a lightweight adjacency pattern (`graph_nodes` and `graph_edges` tables in PostgreSQL) to model the complex, many-to-many, and temporal relationships that define pest control operations: technicians serve locations, locations belong to customers, customers hold contracts, contracts use service plans, chemicals are applied at locations by technicians, devices are placed at locations and inspected by technicians, vehicles carry chemicals and are assigned to technicians.

The pest control domain is more relationship-heavy than it first appears. Consider the queries that regulators, operations managers, and AI systems need to answer: "Which technicians have applied restricted-use products at locations within this school district?" requires traversing technician-to-application-to-location-to-geography relationships. "Show me every property that has had the same pest recurrence within 90 days of treatment" requires traversing location-to-job-to-chemical-application-to-pest chains. "Which technician's certifications cover the service types needed for tomorrow's route?" requires traversing technician-to-certification-to-service-type mappings. These are graph traversal problems.

The graph layer does not replace the relational tables — it augments them. Operational data (jobs, invoices, chemical applications) lives in standard relational tables for transactional integrity and compliance. The graph layer provides a parallel navigation structure that makes relationship queries fast and natural, without requiring multi-table JOINs across 5-6 tables.

**Best for:** Organisations that need to answer complex relationship questions across their data — regulatory compliance traversals, conflict-of-interest detection, supply chain traceability, and AI-driven insights that depend on understanding entity interconnections.

**Trade-offs:**
- (+) Complex relationship queries (multi-hop traversals) are fast and natural
- (+) Graph layer enables AI-powered insights: pest spread analysis, technician-location affinity, chemical efficacy networks
- (+) Adding new relationship types requires only new edge rows, not schema changes
- (+) Relational core preserves ACID guarantees for compliance-critical data
- (+) Graph queries complement, not replace, standard SQL for operational screens
- (-) Dual-write to relational and graph layers adds application complexity
- (-) Graph query language (recursive CTEs or procedural traversal) has a learning curve
- (-) Graph layer must be kept in sync with relational tables — eventual consistency risk
- (-) More storage: graph edges duplicate relationship information already implicit in foreign keys
- (-) Fewer developers are fluent in graph query patterns compared to standard SQL

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EPA FIFRA / 40 CFR 170 | Chemical application records in relational tables mirror EPA-mandated fields; graph edges link applications to technicians, locations, and products for traversal queries |
| EPA PPLS API | Pesticide products as graph nodes enable "which products share active ingredients?" traversal |
| NPMA QualityPro / GreenPro | Certification nodes linked to technician nodes with temporal edges (valid_from, valid_until) |
| ISO 3166 | Geographic hierarchy modelled as graph nodes: Country → State → County → City → ZIP, enabling "all locations in county X" traversals |
| RFC 7946 GeoJSON | Location nodes carry GeoJSON coordinates for spatial + graph hybrid queries |
| ISO 55000/55001 | Device lifecycle modelled as graph edges: placed_at → inspected_at → replaced_at → decommissioned, satisfying asset management traceability |

---

## Graph Layer Foundation

```sql
-- Lightweight property graph implemented in PostgreSQL.
-- Every domain entity gets a node; every relationship gets an edge.
-- Graph queries use recursive CTEs or application-layer traversal.

CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    node_type VARCHAR(50) NOT NULL,
        -- Customer, ServiceLocation, Technician, Job, ChemicalApplication,
        -- PesticideProduct, Device, Vehicle, Certification, ServicePlan,
        -- PestType, GeographicArea, Contract
    entity_id UUID NOT NULL,  -- FK to the corresponding relational table row
    label VARCHAR(255),  -- human-readable label for display
    properties JSONB NOT NULL DEFAULT '{}',
        -- cached/denormalised properties for graph-only queries
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(node_type, entity_id)
);

CREATE INDEX idx_gnodes_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_gnodes_type ON graph_nodes(node_type, tenant_id);
CREATE INDEX idx_gnodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_gnodes_properties ON graph_nodes USING GIN(properties jsonb_path_ops);

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type VARCHAR(50) NOT NULL,
        -- OWNS_LOCATION, SERVICED_AT, APPLIED_CHEMICAL, ASSIGNED_TO,
        -- HAS_CERTIFICATION, DRIVES_VEHICLE, PLACED_DEVICE, INSPECTED_DEVICE,
        -- SUBSCRIBES_TO, TARGETS_PEST, CONTAINS_INGREDIENT, LOCATED_IN,
        -- INVOICED_FOR, FOLLOWS_UP
    properties JSONB NOT NULL DEFAULT '{}',
        -- edge-specific data: dates, quantities, roles, etc.
    valid_from TIMESTAMPTZ,  -- temporal edges: when did this relationship start?
    valid_until TIMESTAMPTZ, -- temporal edges: when did it end? NULL = current
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_gedges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_gedges_type ON graph_edges(edge_type, tenant_id);
CREATE INDEX idx_gedges_temporal ON graph_edges(valid_from, valid_until)
    WHERE valid_until IS NULL;
CREATE INDEX idx_gedges_properties ON graph_edges USING GIN(properties jsonb_path_ops);
```

## Graph Edge Taxonomy

```
The following edge types define the pest control domain graph:

=== CUSTOMER RELATIONSHIPS ===
Customer --OWNS_LOCATION--> ServiceLocation
  properties: { "is_primary": true }
Customer --SUBSCRIBES_TO--> ServicePlan
  properties: { "contract_id": "uuid", "start_date": "...", "price_cents": 15000 }

=== SERVICE DELIVERY ===
Technician --SERVICED_AT--> ServiceLocation
  properties: { "job_id": "uuid", "service_date": "2026-05-22", "service_type": "general_pest" }
  valid_from: job start time, valid_until: job end time
Technician --ASSIGNED_TO--> Job
  properties: { "is_lead": true }
Job --PERFORMED_AT--> ServiceLocation
  properties: { "scheduled_start": "...", "actual_start": "..." }

=== CHEMICAL TRACKING ===
ChemicalApplication --USED_PRODUCT--> PesticideProduct
  properties: { "quantity": 30, "unit": "g", "method": "bait" }
ChemicalApplication --APPLIED_BY--> Technician
  properties: { "licence_number": "IL-AG-12345" }
ChemicalApplication --APPLIED_AT--> ServiceLocation
  properties: { "application_site": "interior kitchen", "date": "2026-05-22" }
ChemicalApplication --TARGETS--> PestType
  properties: { "treatment_method": "bait" }
PesticideProduct --CONTAINS_INGREDIENT--> ActiveIngredient
  properties: { "concentration_pct": 0.6 }

=== DEVICE MANAGEMENT ===
Device --PLACED_AT--> ServiceLocation
  properties: { "placement": "exterior NE corner", "installed_date": "2026-05-22" }
  valid_from: installed_date, valid_until: decommission_date
Technician --INSPECTED_DEVICE--> Device
  properties: { "activity_level": "moderate", "action": "bait_replaced", "date": "2026-05-22" }
Device --MONITORS_FOR--> PestType
  properties: { "bait_type": "Contrac Blox" }

=== CREDENTIALS ===
Technician --HAS_CERTIFICATION--> Certification
  properties: { "number": "IL-AG-12345", "category": "7A" }
  valid_from: issued_date, valid_until: expiry_date
Certification --ISSUED_BY--> CertifyingBody
  properties: {}

=== FLEET ===
Technician --DRIVES_VEHICLE--> Vehicle
  properties: { "assigned_date": "2026-03-01" }
  valid_from: assignment start, valid_until: assignment end
Vehicle --CARRIES_CHEMICAL--> PesticideProduct
  properties: { "quantity": 1, "unit": "gal", "loaded_date": "2026-05-22" }

=== GEOGRAPHIC HIERARCHY ===
ServiceLocation --LOCATED_IN--> GeographicArea (ZIP)
GeographicArea (ZIP) --PART_OF--> GeographicArea (City)
GeographicArea (City) --PART_OF--> GeographicArea (County)
GeographicArea (County) --PART_OF--> GeographicArea (State)

=== PEST ECOLOGY ===
PestType --COMMONLY_FOUND_AT--> PropertyType
  properties: { "risk_level": "high", "season": "summer" }
PestType --TREATED_BY--> PesticideProduct
  properties: { "efficacy_rating": 0.92, "source": "field_data" }
```

## Relational Core — Operational Tables

```sql
-- The relational tables handle transactional operations.
-- They are the source of truth; graph nodes/edges are derived from them.

CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    subscription_plan VARCHAR(50) NOT NULL DEFAULT 'trial',
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_number VARCHAR(50),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(30),
    customer_type VARCHAR(30) NOT NULL DEFAULT 'residential',
    billing_method VARCHAR(30) NOT NULL DEFAULT 'per_service',
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);

CREATE TABLE service_locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    address_line1 VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state_code VARCHAR(10) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    coordinates GEOGRAPHY(POINT, 4326),
    property_type VARCHAR(50),
    details JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_customer ON service_locations(customer_id);
CREATE INDEX idx_locations_geo ON service_locations USING GIST(coordinates);

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
    profile JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

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
    scheduled_start TIMESTAMPTZ,
    scheduled_end TIMESTAMPTZ,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    technician_ids UUID[] NOT NULL DEFAULT '{}',
    lead_technician_id UUID REFERENCES users(id),
    price_cents INTEGER,
    service_report JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_tenant_status ON jobs(tenant_id, status);
CREATE INDEX idx_jobs_schedule ON jobs(tenant_id, scheduled_start);

CREATE TABLE pesticide_products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epa_registration_number VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(500) NOT NULL,
    registrant_name VARCHAR(255),
    signal_word VARCHAR(20),
    restricted_use BOOLEAN NOT NULL DEFAULT false,
    active_ingredients TEXT[],
    ppls_data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chemical_applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    job_id UUID NOT NULL REFERENCES jobs(id),
    pesticide_product_id UUID NOT NULL REFERENCES pesticide_products(id),
    technician_id UUID NOT NULL REFERENCES users(id),
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    epa_registration_number VARCHAR(30) NOT NULL,
    application_date DATE NOT NULL,
    application_time TIME NOT NULL,
    target_pests TEXT[] NOT NULL,
    application_site VARCHAR(255) NOT NULL,
    application_method VARCHAR(50) NOT NULL,
    quantity_applied NUMERIC(10,3) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL,
    dilution_rate VARCHAR(50),
    re_entry_interval_hours NUMERIC(6,1),
    restricted_use_product BOOLEAN NOT NULL DEFAULT false,
    applicator_licence_number VARCHAR(50),
    state_compliance_data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chem_apps_date ON chemical_applications(tenant_id, application_date);
CREATE INDEX idx_chem_apps_job ON chemical_applications(job_id);

CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    service_location_id UUID NOT NULL REFERENCES service_locations(id),
    barcode VARCHAR(100),
    device_category VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    installed_date DATE NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_location ON devices(service_location_id);
CREATE INDEX idx_devices_barcode ON devices(tenant_id, barcode);

CREATE TABLE device_inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    device_id UUID NOT NULL REFERENCES devices(id),
    job_id UUID REFERENCES jobs(id),
    technician_id UUID NOT NULL REFERENCES users(id),
    inspection_date TIMESTAMPTZ NOT NULL,
    activity_level VARCHAR(20) NOT NULL,
    action_taken VARCHAR(50),
    findings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_device ON device_inspections(device_id);

CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id),
    job_id UUID REFERENCES jobs(id),
    invoice_number VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    total_cents INTEGER NOT NULL DEFAULT 0,
    amount_paid_cents INTEGER NOT NULL DEFAULT 0,
    issued_date DATE,
    due_date DATE,
    line_items JSONB NOT NULL DEFAULT '[]',
    payments JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(tenant_id, status);

-- Pest type reference data (used as graph nodes)
CREATE TABLE pest_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    common_name VARCHAR(100) NOT NULL,
    scientific_name VARCHAR(200),
    category VARCHAR(50) NOT NULL,
        -- insect, rodent, wildlife, bird, arachnid, other
    reportable BOOLEAN NOT NULL DEFAULT false,
    description TEXT,
    treatment_protocols JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL,
    changes JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_date ON audit_logs(tenant_id, created_at);
```

## Graph Synchronisation Triggers

```sql
-- Triggers automatically create/update graph nodes and edges when relational data changes.
-- This keeps the graph in sync without manual dual-writes in application code.

CREATE OR REPLACE FUNCTION sync_customer_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_nodes (tenant_id, node_type, entity_id, label, properties)
        VALUES (
            NEW.tenant_id, 'Customer', NEW.id,
            COALESCE(NEW.company_name, NEW.first_name || ' ' || NEW.last_name),
            jsonb_build_object(
                'customer_type', NEW.customer_type,
                'billing_method', NEW.billing_method,
                'status', NEW.status
            )
        );
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE graph_nodes SET
            label = COALESCE(NEW.company_name, NEW.first_name || ' ' || NEW.last_name),
            properties = jsonb_build_object(
                'customer_type', NEW.customer_type,
                'billing_method', NEW.billing_method,
                'status', NEW.status
            )
        WHERE node_type = 'Customer' AND entity_id = NEW.id;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM graph_nodes WHERE node_type = 'Customer' AND entity_id = OLD.id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_customer_graph
    AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW EXECUTE FUNCTION sync_customer_to_graph();

-- Similar triggers for: service_locations, users, jobs, chemical_applications,
-- devices, device_inspections, pesticide_products, invoices

CREATE OR REPLACE FUNCTION sync_chemical_application_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    app_node_id UUID;
    tech_node_id UUID;
    loc_node_id UUID;
    product_node_id UUID;
BEGIN
    IF TG_OP = 'INSERT' THEN
        -- Create ChemicalApplication node
        INSERT INTO graph_nodes (tenant_id, node_type, entity_id, label, properties)
        VALUES (
            NEW.tenant_id, 'ChemicalApplication', NEW.id,
            NEW.epa_registration_number || ' @ ' || NEW.application_date,
            jsonb_build_object(
                'date', NEW.application_date,
                'method', NEW.application_method,
                'quantity', NEW.quantity_applied,
                'unit', NEW.unit_of_measure,
                'restricted_use', NEW.restricted_use_product
            )
        )
        RETURNING id INTO app_node_id;

        -- Find related nodes
        SELECT id INTO tech_node_id FROM graph_nodes
            WHERE node_type = 'Technician' AND entity_id = NEW.technician_id;
        SELECT id INTO loc_node_id FROM graph_nodes
            WHERE node_type = 'ServiceLocation' AND entity_id = NEW.service_location_id;
        SELECT id INTO product_node_id FROM graph_nodes
            WHERE node_type = 'PesticideProduct' AND entity_id = NEW.pesticide_product_id;

        -- Create edges
        IF tech_node_id IS NOT NULL THEN
            INSERT INTO graph_edges (tenant_id, source_node_id, target_node_id, edge_type, properties, valid_from)
            VALUES (NEW.tenant_id, app_node_id, tech_node_id, 'APPLIED_BY',
                    jsonb_build_object('licence_number', NEW.applicator_licence_number),
                    (NEW.application_date || ' ' || NEW.application_time)::timestamptz);
        END IF;

        IF loc_node_id IS NOT NULL THEN
            INSERT INTO graph_edges (tenant_id, source_node_id, target_node_id, edge_type, properties, valid_from)
            VALUES (NEW.tenant_id, app_node_id, loc_node_id, 'APPLIED_AT',
                    jsonb_build_object('site', NEW.application_site),
                    (NEW.application_date || ' ' || NEW.application_time)::timestamptz);
        END IF;

        IF product_node_id IS NOT NULL THEN
            INSERT INTO graph_edges (tenant_id, source_node_id, target_node_id, edge_type, properties, valid_from)
            VALUES (NEW.tenant_id, app_node_id, product_node_id, 'USED_PRODUCT',
                    jsonb_build_object('quantity', NEW.quantity_applied, 'unit', NEW.unit_of_measure, 'method', NEW.application_method),
                    (NEW.application_date || ' ' || NEW.application_time)::timestamptz);
        END IF;

        -- Create edges for each target pest
        DECLARE
            pest_name TEXT;
            pest_node_id UUID;
        BEGIN
            FOREACH pest_name IN ARRAY NEW.target_pests LOOP
                SELECT gn.id INTO pest_node_id FROM graph_nodes gn
                    JOIN pest_types pt ON pt.id = gn.entity_id
                    WHERE gn.node_type = 'PestType' AND pt.common_name = pest_name;
                IF pest_node_id IS NOT NULL THEN
                    INSERT INTO graph_edges (tenant_id, source_node_id, target_node_id, edge_type, properties, valid_from)
                    VALUES (NEW.tenant_id, app_node_id, pest_node_id, 'TARGETS',
                            jsonb_build_object('method', NEW.application_method),
                            (NEW.application_date || ' ' || NEW.application_time)::timestamptz);
                END IF;
            END LOOP;
        END;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_chemical_app_graph
    AFTER INSERT ON chemical_applications
    FOR EACH ROW EXECUTE FUNCTION sync_chemical_application_to_graph();
```

## Graph Query Examples

```sql
-- =============================================================
-- QUERY 1: Which technicians have applied restricted-use products
--          at locations within a specific ZIP code?
-- =============================================================
WITH restricted_apps AS (
    SELECT DISTINCT ge_tech.target_node_id AS tech_node_id
    FROM graph_nodes gn_app
    JOIN graph_edges ge_tech
        ON ge_tech.source_node_id = gn_app.id AND ge_tech.edge_type = 'APPLIED_BY'
    JOIN graph_edges ge_loc
        ON ge_loc.source_node_id = gn_app.id AND ge_loc.edge_type = 'APPLIED_AT'
    JOIN graph_edges ge_geo
        ON ge_geo.source_node_id = ge_loc.target_node_id AND ge_geo.edge_type = 'LOCATED_IN'
    JOIN graph_nodes gn_zip
        ON gn_zip.id = ge_geo.target_node_id AND gn_zip.properties->>'code' = '62701'
    WHERE gn_app.node_type = 'ChemicalApplication'
      AND (gn_app.properties->>'restricted_use')::boolean = true
      AND gn_app.tenant_id = $1
)
SELECT gn.label AS technician_name, gn.entity_id AS technician_id
FROM restricted_apps ra
JOIN graph_nodes gn ON gn.id = ra.tech_node_id;


-- =============================================================
-- QUERY 2: For a given location, show the full treatment history
--          as a graph traversal (products, technicians, pests)
-- =============================================================
SELECT
    gn_app.properties->>'date' AS treatment_date,
    gn_tech.label AS technician,
    gn_prod.label AS product,
    gn_pest.label AS target_pest,
    ge_prod.properties->>'quantity' AS quantity,
    ge_prod.properties->>'unit' AS unit,
    ge_prod.properties->>'method' AS method
FROM graph_nodes gn_loc
JOIN graph_edges ge_app ON ge_app.target_node_id = gn_loc.id AND ge_app.edge_type = 'APPLIED_AT'
JOIN graph_nodes gn_app ON gn_app.id = ge_app.source_node_id
JOIN graph_edges ge_tech ON ge_tech.source_node_id = gn_app.id AND ge_tech.edge_type = 'APPLIED_BY'
JOIN graph_nodes gn_tech ON gn_tech.id = ge_tech.target_node_id
JOIN graph_edges ge_prod ON ge_prod.source_node_id = gn_app.id AND ge_prod.edge_type = 'USED_PRODUCT'
JOIN graph_nodes gn_prod ON gn_prod.id = ge_prod.target_node_id
LEFT JOIN graph_edges ge_pest ON ge_pest.source_node_id = gn_app.id AND ge_pest.edge_type = 'TARGETS'
LEFT JOIN graph_nodes gn_pest ON gn_pest.id = ge_pest.target_node_id
WHERE gn_loc.entity_id = $location_id
ORDER BY gn_app.properties->>'date' DESC;


-- =============================================================
-- QUERY 3: Which products share active ingredients with a
--          product that was effective at a given location?
--          (Useful for suggesting alternative treatments)
-- =============================================================
WITH effective_products AS (
    SELECT DISTINCT ge_prod.target_node_id AS product_node_id
    FROM graph_nodes gn_loc
    JOIN graph_edges ge_app ON ge_app.target_node_id = gn_loc.id AND ge_app.edge_type = 'APPLIED_AT'
    JOIN graph_nodes gn_app ON gn_app.id = ge_app.source_node_id
    JOIN graph_edges ge_prod ON ge_prod.source_node_id = gn_app.id AND ge_prod.edge_type = 'USED_PRODUCT'
    WHERE gn_loc.entity_id = $location_id
),
effective_ingredients AS (
    SELECT DISTINCT ge_ingr.target_node_id AS ingredient_node_id
    FROM effective_products ep
    JOIN graph_edges ge_ingr ON ge_ingr.source_node_id = ep.product_node_id
        AND ge_ingr.edge_type = 'CONTAINS_INGREDIENT'
)
SELECT DISTINCT gn_alt.label AS alternative_product, gn_alt.entity_id AS product_id
FROM effective_ingredients ei
JOIN graph_edges ge_alt ON ge_alt.target_node_id = ei.ingredient_node_id
    AND ge_alt.edge_type = 'CONTAINS_INGREDIENT'
JOIN graph_nodes gn_alt ON gn_alt.id = ge_alt.source_node_id
WHERE gn_alt.id NOT IN (SELECT product_node_id FROM effective_products);


-- =============================================================
-- QUERY 4: Pest spread analysis — find all locations where a
--          specific pest was found, clustered by geographic area
--          (recursive geographic hierarchy traversal)
-- =============================================================
WITH RECURSIVE geo_hierarchy AS (
    -- Start from locations where the pest was targeted
    SELECT DISTINCT
        ge_geo.target_node_id AS area_node_id,
        gn_area.label AS area_name,
        gn_area.properties->>'level' AS area_level,
        1 AS depth
    FROM graph_nodes gn_pest
    JOIN graph_edges ge_target ON ge_target.target_node_id = gn_pest.id AND ge_target.edge_type = 'TARGETS'
    JOIN graph_nodes gn_app ON gn_app.id = ge_target.source_node_id
    JOIN graph_edges ge_loc ON ge_loc.source_node_id = gn_app.id AND ge_loc.edge_type = 'APPLIED_AT'
    JOIN graph_edges ge_geo ON ge_geo.source_node_id = ge_loc.target_node_id AND ge_geo.edge_type = 'LOCATED_IN'
    JOIN graph_nodes gn_area ON gn_area.id = ge_geo.target_node_id
    WHERE gn_pest.label = 'German Cockroach'
      AND gn_pest.tenant_id = $1

    UNION ALL

    -- Walk up the geographic hierarchy
    SELECT
        ge_parent.target_node_id,
        gn_parent.label,
        gn_parent.properties->>'level',
        gh.depth + 1
    FROM geo_hierarchy gh
    JOIN graph_edges ge_parent ON ge_parent.source_node_id = gh.area_node_id
        AND ge_parent.edge_type = 'PART_OF'
    JOIN graph_nodes gn_parent ON gn_parent.id = ge_parent.target_node_id
    WHERE gh.depth < 4  -- Stop at state level
)
SELECT area_level, area_name, COUNT(*) AS affected_locations
FROM geo_hierarchy
GROUP BY area_level, area_name
ORDER BY area_level, affected_locations DESC;


-- =============================================================
-- QUERY 5: Technician certification coverage check — does this
--          technician have valid certifications for all service
--          types on tomorrow's route?
-- =============================================================
SELECT
    j.service_type,
    EXISTS (
        SELECT 1
        FROM graph_edges ge_cert
        JOIN graph_nodes gn_cert ON gn_cert.id = ge_cert.target_node_id
        WHERE ge_cert.source_node_id = gn_tech.id
          AND ge_cert.edge_type = 'HAS_CERTIFICATION'
          AND ge_cert.valid_until IS NULL  -- currently valid
          AND gn_cert.properties->>'covers_service_type' = j.service_type
    ) AS has_valid_cert
FROM jobs j
JOIN graph_nodes gn_tech ON gn_tech.node_type = 'Technician' AND gn_tech.entity_id = $technician_id
WHERE j.tenant_id = $1
  AND j.scheduled_start::date = CURRENT_DATE + 1
  AND $technician_id = ANY(j.technician_ids);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Identity & Multi-Tenancy | 2 | tenants, users |
| Customer & Location | 2 | customers, service_locations |
| Jobs & Scheduling | 1 | jobs (with embedded service report) |
| Chemical & Pesticide | 2 | pesticide_products, chemical_applications |
| Devices & Traps | 2 | devices, device_inspections |
| Billing | 1 | invoices |
| Reference Data | 1 | pest_types |
| Audit | 1 | audit_logs |
| **Total** | **14** | Plus graph data (nodes/edges scale with data volume) |

---

## Key Design Decisions

1. **PostgreSQL-native graph, not a separate graph database** — using `graph_nodes` and `graph_edges` tables in the same PostgreSQL database avoids the operational complexity of a separate Neo4j or similar graph database. Recursive CTEs provide adequate traversal performance for pest control data volumes (typically thousands to tens of thousands of nodes per tenant, not millions).

2. **Graph as derived data, relational as source of truth** — database triggers automatically populate and update graph nodes/edges when relational data changes. The graph can be rebuilt from relational tables if needed. This avoids the dual-write consistency problem in application code.

3. **Temporal edges for credential and assignment tracking** — `valid_from` and `valid_until` on edges enable time-scoped queries: "Which technicians were certified to apply restricted-use products on March 15th?" This is essential for regulatory audits that investigate past compliance.

4. **Geographic hierarchy as graph nodes** — modelling Country/State/County/City/ZIP as a graph hierarchy enables multi-level geographic aggregation queries without hardcoded geographic lookups. The same traversal pattern works for franchise territory analysis.

5. **Pest ecology graph** — connecting pest types to property types, products, and locations creates a knowledge graph that AI can query for treatment recommendations. "What products have been most effective against German cockroach at commercial kitchens in this ZIP code?" becomes a graph traversal rather than a complex multi-JOIN query.

6. **Relational core follows the Hybrid JSONB pattern** — the operational tables use the same JSONB-augmented approach as Data Model Suggestion 3. The graph layer is additive, not a replacement for the relational tables.

7. **Edge properties as JSONB** — each edge can carry arbitrary metadata (quantities, dates, roles) without requiring separate relationship tables. This keeps the graph schema simple while accommodating diverse relationship attributes.

8. **Chemical application tracking in both layers** — chemical applications have full relational records for EPA compliance and graph edges for traversal queries. The relational record is the legal document; the graph edges enable analytics. Both are created atomically via the trigger.

9. **Graph query patterns for AI features** — the graph layer directly supports several AI-native features from the project's roadmap: pest spread analysis (geographic traversal), treatment efficacy analysis (product-to-pest-to-location traversal), and technician-location affinity (service history traversal). These would require 5-6 table JOINs in a purely relational model.

10. **Scalability boundary** — this approach works well up to approximately 10 million graph edges per tenant. Beyond that, consider migrating the graph layer to a dedicated graph database (Neo4j, Amazon Neptune) while keeping the relational core unchanged.
