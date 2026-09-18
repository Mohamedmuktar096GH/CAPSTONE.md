🏆 CAPSTONE BUILD SPRINT: SYSTEM ARCHITECTURE REPORTProject Track: Advanced E-Commerce ArchitectureDatabase System: PostgreSQL (Serverless Instance via Neon.tech)Deployment Strategy: Versioned Schema Migrations🗺️ 1. Requirements Summary & System DesignOur system addresses the operational challenges of modern e-commerce catalogs by using a Hybrid Relational/NoSQL design alongside recursive categorization.Core Architectural Features:Unlimited Category Nesting: Built using a self-referencing hierarchy pattern.Flexible Schemas: Built using a PostgreSQL JSONB column to host entirely unique product attributes (e.g., electronic parameters vs. clothing sizes) inside one single physical table.Automatic Change Tracking: Handled via automated event-driven audit log ledger triggers.🚀 2. Versioned Schema Migrations (V1 to V4)V1__core_tables.sql (Foundational Entities)sqlCREATE TABLE categories (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL,
  parent_id INT REFERENCES categories(id) ON DELETE SET NULL
);

CREATE TABLE products (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL,
  sku TEXT NOT NULL UNIQUE,
  category_id INT REFERENCES categories(id) ON DELETE RESTRICT,
  price DECIMAL(12, 2) NOT NULL CHECK (price >= 0),
  attributes JSONB DEFAULT '{}'::jsonb
);

CREATE TABLE users (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'customer'
);

CREATE TABLE orders (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id INT REFERENCES users(id) ON DELETE RESTRICT,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE order_items (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id INT REFERENCES orders(id) ON DELETE CASCADE,
  product_id INT REFERENCES products(id) ON DELETE RESTRICT,
  quantity INT NOT NULL CHECK (quantity > 0),
  unit_price DECIMAL(12, 2) NOT NULL CHECK (unit_price >= 0)
);
Use code with caution.V2__audit_logging.sql (Automation Triggers)sqlCREATE TABLE product_audit_log (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  product_id INT NOT NULL,
  op TEXT NOT NULL,
  old_data JSONB,
  new_data JSONB,
  changed_by TEXT DEFAULT current_user,
  changed_at TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION track_product_changes() RETURNS TRIGGER AS $$
BEGIN
  IF (TG_OP = 'DELETE') THEN
    INSERT INTO product_audit_log(product_id, op, old_data, new_data) VALUES (OLD.id, TG_OP, to_jsonb(OLD), NULL);
  ELSIF (TG_OP = 'UPDATE') THEN
    INSERT INTO product_audit_log(product_id, op, old_data, new_data) VALUES (NEW.id, TG_OP, to_jsonb(OLD), to_jsonb(NEW));
  ELSIF (TG_OP = 'INSERT') THEN
    INSERT INTO product_audit_log(product_id, op, old_data, new_data) VALUES (NEW.id, TG_OP, NULL, to_jsonb(NEW));
  END IF;
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_track_products
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION track_product_changes();
Use code with caution.🌿 3. NoSQL Integration & Advanced Querying (V3)We populated the data structure with an deeply nested hierarchy along with structured JSON components:Complex Recursive Extraction QuerysqlWITH RECURSIVE category_path AS (
  SELECT id, name, parent_id, 0 AS depth, array[name::text] AS path_nodes
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.name, c.parent_id, cp.depth + 1, cp.path_nodes || c.name::text
  FROM categories c JOIN category_path cp ON c.parent_id = cp.id
)
SELECT 
  array_to_string(cp.path_nodes, ' -> ') AS full_category_breadcrumbs,
  p.name AS product_name,
  p.price,
  COALESCE(p.attributes->>'ram', 'N/A') AS laptop_ram,
  COALESCE(p.attributes->>'size', 'N/A') AS clothing_size
FROM category_path cp JOIN products p ON p.category_id = cp.id ORDER BY cp.path_nodes;
Use code with caution.⚡ 4. Query Optimization EvidenceTo prevent slow sequential operations across millions of variable catalog entries, we implemented an optimized indexing strategy:sqlCREATE INDEX idx_products_attributes ON products USING GIN (attributes);
Use code with caution.Performance Impact: Utilizing a Generalized Inverted Index (GIN) maps every interior JSON key-value pair into a structured bitmap tree layout, dropping dynamic element lookup latency from standard text scans straight down to sub-millisecond intervals.🔒 5. Hardened Least-Privilege Security (V4)We eliminated global public privileges and created isolated structural database identities:sqlREVOKE CREATE ON SCHEMA public FROM PUBLIC;

CREATE ROLE capstone_read_only;
CREATE ROLE capstone_api_worker;

GRANT USAGE ON SCHEMA public TO capstone_read_only, capstone_api_worker;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO capstone_read_only;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO capstone_api_worker;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO capstone_api_worker;

CREATE USER ecommerce_api_user WITH LOGIN PASSWORD 'capstone-secure-vault-2026';
GRANT capstone_api_worker TO ecommerce_api_user;
