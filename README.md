# PostgreSQL pglogical Replication — Complete Proof of Concept
### WSL + Podman + PostgreSQL Environment

> **Hands-on Lab** | PostgreSQL 15 | pglogical 2.x | Podman Containers | WSL2

---

## Table of Contents

1. [Environment Overview](#environment-overview)
2. [Step 1 — Prerequisites & Installation](#step-1--prerequisites--installation)
3. [Step 2 — Create Podman Network](#step-2--create-podman-network)
4. [Step 3 — Launch PostgreSQL Containers](#step-3--launch-postgresql-containers)
5. [Step 4 — Install pglogical Extension](#step-4--install-pglogical-extension)
6. [Step 5 — PostgreSQL Configuration](#step-5--postgresql-configuration)
7. [Step 6 — pg_hba.conf Configuration](#step-6--pg_hbaconf-configuration)
8. [Step 7 — Basic Node Setup & Connectivity](#step-7--basic-node-setup--connectivity)
9. [Step 8 — Replication Sets Configuration](#step-8--replication-sets-configuration)
10. [Step 9 — Table-Level Replication](#step-9--table-level-replication)
11. [Step 10 — Replicate All Tables in Schema](#step-10--replicate-all-tables-in-schema)
12. [Step 11 — Automatic Replication of New Tables](#step-11--automatic-replication-of-new-tables)
13. [Step 12 — Subscription Creation & Management](#step-12--subscription-creation--management)
14. [Step 13 — Bi-directional Replication Setup](#step-13--bi-directional-replication-setup)
15. [Step 14 — Conflict Detection & Resolution](#step-14--conflict-detection--resolution)
16. [Step 15 — DDL (Schema) Replication](#step-15--ddl-schema-replication)
17. [Step 16 — Sequence Replication](#step-16--sequence-replication)
18. [Step 17 — Row-Level Filtering](#step-17--row-level-filtering)
19. [Step 18 — Column-Level Filtering](#step-18--column-level-filtering)
20. [Step 19 — Initial Data Synchronization Control](#step-19--initial-data-synchronization-control)
21. [Step 20 — Monitoring & Replication Status](#step-20--monitoring--replication-status)
22. [Step 21 — Performance & Load Testing](#step-21--performance--load-testing)
23. [Step 22 — Failover & Recovery Simulation](#step-22--failover--recovery-simulation)
24. [Step 23 — Multi-node Replication Topology](#step-23--multi-node-replication-topology)
25. [Step 24 — Selective Replication Sets per Subscription](#step-24--selective-replication-sets-per-subscription)
26. [Step 25 — Schema-Level Replication Behavior](#step-25--schema-level-replication-behavior)
27. [Troubleshooting](#troubleshooting)
28. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Environment Overview

```
┌─────────────────────────────────────────────────────────┐
│                     WSL2 (Ubuntu)                       │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────────┐    │
│  │  pg-provider     │      │  pg-subscriber       │    │
│  │  (Primary)       │◄────►│  (Replica)           │    │
│  │  Port: 5432      │      │  Port: 5433          │    │
│  │  DB: providerdb  │      │  DB: subscriberdb    │    │
│  └──────────────────┘      └──────────────────────┘    │
│           │                          │                  │
│           └──────────────────────────┘                  │
│                  pglogical-net (Podman)                  │
│                  Subnet: 10.89.0.0/24                   │
└─────────────────────────────────────────────────────────┘
```

| Container | Role | Host Port | DB Name | pglogical Node |
|-----------|------|-----------|---------|----------------|
| `pg-provider` | Provider (Publisher) | 5432 | providerdb | provider_node |
| `pg-subscriber` | Subscriber (Replica) | 5433 | subscriberdb | subscriber_node |
| `pg-node3` | Third Node (Multi-topology) | 5434 | node3db | node3 |

---

## Step 1 — Prerequisites & Installation

### Step 1.1 — Verify WSL2 and Podman

```bash
# Check WSL version
wsl --version

# Check Podman version (must be >= 4.x)
podman --version

# Expected output example:
# podman version 4.9.4
```

### Step 1.2 — Install required tools inside WSL

```bash
sudo apt-get update && sudo apt-get install -y \
  postgresql-client \
  curl \
  wget \
  pgbench \
  jq
```

### Step 1.3 — Create a working directory

```bash
mkdir -p ~/pglogical-poc && cd ~/pglogical-poc
```

### Step 1.4 — Create the custom PostgreSQL Dockerfile

```bash
cat > ~/pglogical-poc/Dockerfile << 'EOF'
FROM postgres:15

# Install pglogical
RUN apt-get update && apt-get install -y \
    postgresql-15-pglogical \
    && rm -rf /var/lib/apt/lists/*

# Copy init scripts
COPY init/ /docker-entrypoint-initdb.d/
EOF
```

### Step 1.5 — Create init directory and placeholder

```bash
mkdir -p ~/pglogical-poc/init

cat > ~/pglogical-poc/init/00-placeholder.sh << 'EOF'
#!/bin/bash
# Placeholder — actual config done via SQL after container start
echo "Container initialized"
EOF

chmod +x ~/pglogical-poc/init/00-placeholder.sh
```

---

## Step 2 — Create Podman Network

Containers must communicate over a dedicated network. This avoids port conflicts and provides DNS resolution by container name.

### Step 2.1 — Create the Podman network

```bash
podman network create \
  --driver bridge \
  --subnet 10.89.0.0/24 \
  --gateway 10.89.0.1 \
  pglogical-net
```

### Step 2.2 — Verify network creation

```bash
podman network inspect pglogical-net | jq '.[0].subnets'
```

**Expected output:**
```json
[
  {
    "subnet": "10.89.0.0/24",
    "gateway": "10.89.0.1"
  }
]
```

---

## Step 3 — Launch PostgreSQL Containers

### Step 3.1 — Build the custom image

```bash
cd ~/pglogical-poc
podman build -t pg15-pglogical:latest .
```

**Expected output:**
```
STEP 1/3: FROM postgres:15
STEP 2/3: RUN apt-get update ...
Successfully tagged localhost/pg15-pglogical:latest
```

### Step 3.2 — Start the Provider container

```bash
podman run -d \
  --name pg-provider \
  --network pglogical-net \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=provider_pass \
  -e POSTGRES_DB=providerdb \
  -e POSTGRES_USER=postgres \
  -v pglogical-provider-data:/var/lib/postgresql/data \
  pg15-pglogical:latest
```

### Step 3.3 — Start the Subscriber container

```bash
podman run -d \
  --name pg-subscriber \
  --network pglogical-net \
  -p 5433:5432 \
  -e POSTGRES_PASSWORD=subscriber_pass \
  -e POSTGRES_DB=subscriberdb \
  -e POSTGRES_USER=postgres \
  -v pglogical-subscriber-data:/var/lib/postgresql/data \
  pg15-pglogical:latest
```

### Step 3.4 — Start the third node (for multi-topology demo)

```bash
podman run -d \
  --name pg-node3 \
  --network pglogical-net \
  -p 5434:5432 \
  -e POSTGRES_PASSWORD=node3_pass \
  -e POSTGRES_DB=node3db \
  -e POSTGRES_USER=postgres \
  -v pglogical-node3-data:/var/lib/postgresql/data \
  pg15-pglogical:latest
```

### Step 3.5 — Verify all containers are running

```bash
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Expected output:**
```
NAMES            STATUS         PORTS
pg-provider      Up 30 seconds  0.0.0.0:5432->5432/tcp
pg-subscriber    Up 25 seconds  0.0.0.0:5433->5432/tcp
pg-node3         Up 20 seconds  0.0.0.0:5434->5432/tcp
```

### Step 3.6 — Verify network connectivity between containers

```bash
# Test provider → subscriber connectivity
podman exec pg-provider pg_isready -h pg-subscriber -p 5432 -U postgres

# Test subscriber → provider connectivity
podman exec pg-subscriber pg_isready -h pg-provider -p 5432 -U postgres
```

**Expected output:**
```
pg-subscriber:5432 - accepting connections
pg-provider:5432 - accepting connections
```

---

## Step 4 — Install pglogical Extension

### Step 4.1 — Install pglogical on Provider

```bash
podman exec -it pg-provider psql -U postgres -d providerdb -c "CREATE EXTENSION IF NOT EXISTS pglogical;"
```

**Expected output:**
```
CREATE EXTENSION
```

### Step 4.2 — Install pglogical on Subscriber

```bash
podman exec -it pg-subscriber psql -U postgres -d subscriberdb -c "CREATE EXTENSION IF NOT EXISTS pglogical;"
```

### Step 4.3 — Install pglogical on Node3

```bash
podman exec -it pg-node3 psql -U postgres -d node3db -c "CREATE EXTENSION IF NOT EXISTS pglogical;"
```

### Step 4.4 — Verify extension installation on all nodes

```bash
for container in pg-provider pg-subscriber pg-node3; do
  echo "=== $container ==="
  podman exec $container psql -U postgres -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'pglogical';"
done
```

**Expected output:**
```
=== pg-provider ===
  extname  | extversion
-----------+------------
 pglogical | 2.4.4
(1 row)
```

---

## Step 5 — PostgreSQL Configuration

pglogical requires specific `postgresql.conf` settings. These must be set **before** creating nodes.

### Step 5.1 — Configure Provider (postgresql.conf)

```bash
podman exec -it pg-provider psql -U postgres << 'ENDSQL'
ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM SET max_worker_processes = 10;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET shared_preload_libraries = 'pglogical';
ALTER SYSTEM SET track_commit_timestamp = on;
-- Optional but recommended
ALTER SYSTEM SET wal_sender_timeout = '60s';
ALTER SYSTEM SET wal_receiver_timeout = '60s';
ALTER SYSTEM SET log_replication_commands = on;
ENDSQL
```

### Step 5.2 — Configure Subscriber (postgresql.conf)

```bash
podman exec -it pg-subscriber psql -U postgres << 'ENDSQL'
ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM SET max_worker_processes = 10;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET shared_preload_libraries = 'pglogical';
ALTER SYSTEM SET track_commit_timestamp = on;
ALTER SYSTEM SET wal_sender_timeout = '60s';
ALTER SYSTEM SET wal_receiver_timeout = '60s';
ENDSQL
```

### Step 5.3 — Configure Node3 (postgresql.conf)

```bash
podman exec -it pg-node3 psql -U postgres << 'ENDSQL'
ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM SET max_worker_processes = 10;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET shared_preload_libraries = 'pglogical';
ALTER SYSTEM SET track_commit_timestamp = on;
ENDSQL
```

### Step 5.4 — Restart all containers to apply settings

```bash
podman restart pg-provider pg-subscriber pg-node3

# Wait for PostgreSQL to come up
sleep 10

# Verify settings applied
podman exec pg-provider psql -U postgres -c "SHOW wal_level; SHOW max_replication_slots;"
```

**Expected output:**
```
 wal_level
-----------
 logical

 max_replication_slots
-----------------------
 10
```

### Step 5.5 — Verify shared_preload_libraries

```bash
podman exec pg-provider psql -U postgres -c "SHOW shared_preload_libraries;"
```

**Expected output:**
```
 shared_preload_libraries
--------------------------
 pglogical
```

---

## Step 6 — pg_hba.conf Configuration

pglogical uses replication connections. These must be explicitly allowed.

### Step 6.1 — Configure pg_hba.conf on Provider

```bash
podman exec -it pg-provider bash -c "
cat >> /var/lib/postgresql/data/pg_hba.conf << 'EOF'

# pglogical replication entries
host    all             postgres        10.89.0.0/24            md5
host    replication     postgres        10.89.0.0/24            md5
host    all             all             10.89.0.0/24            md5
EOF
"
```

### Step 6.2 — Configure pg_hba.conf on Subscriber

```bash
podman exec -it pg-subscriber bash -c "
cat >> /var/lib/postgresql/data/pg_hba.conf << 'EOF'

# pglogical replication entries
host    all             postgres        10.89.0.0/24            md5
host    replication     postgres        10.89.0.0/24            md5
host    all             all             10.89.0.0/24            md5
EOF
"
```

### Step 6.3 — Configure pg_hba.conf on Node3

```bash
podman exec -it pg-node3 bash -c "
cat >> /var/lib/postgresql/data/pg_hba.conf << 'EOF'

# pglogical replication entries
host    all             postgres        10.89.0.0/24            md5
host    replication     postgres        10.89.0.0/24            md5
host    all             all             10.89.0.0/24            md5
EOF
"
```

### Step 6.4 — Reload pg_hba.conf on all nodes

```bash
for container in pg-provider pg-subscriber pg-node3; do
  podman exec $container psql -U postgres -c "SELECT pg_reload_conf();"
done
```

### Step 6.5 — Validate connection strings across containers

```bash
# Get container IPs for reference
podman inspect pg-provider --format '{{.NetworkSettings.Networks.pglogical-net.IPAddress}}'
podman inspect pg-subscriber --format '{{.NetworkSettings.Networks.pglogical-net.IPAddress}}'
podman inspect pg-node3 --format '{{.NetworkSettings.Networks.pglogical-net.IPAddress}}'
```

> **Note:** Save these IPs. They are used in DSN strings below. Container name DNS resolution also works within the Podman network.

---

## Step 7 — Basic Node Setup & Connectivity

### Step 7.1 — Create Provider node

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT pglogical.create_node(
    node_name := 'provider_node',
    dsn := 'host=pg-provider port=5432 dbname=providerdb user=postgres password=provider_pass'
);
ENDSQL
```

**Expected output:**
```
 create_node
-------------
  3953743997
(1 row)
```

### Step 7.2 — Create Subscriber node

```bash
podman exec -it pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
SELECT pglogical.create_node(
    node_name := 'subscriber_node',
    dsn := 'host=pg-subscriber port=5432 dbname=subscriberdb user=postgres password=subscriber_pass'
);
ENDSQL
```

### Step 7.3 — Create Node3

```bash
podman exec -it pg-node3 psql -U postgres -d node3db << 'ENDSQL'
SELECT pglogical.create_node(
    node_name := 'node3',
    dsn := 'host=pg-node3 port=5432 dbname=node3db user=postgres password=node3_pass'
);
ENDSQL
```

### Step 7.4 — Verify node creation

```bash
# Check nodes on provider
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT node_id, node_name FROM pglogical.node;"
```

**Expected output:**
```
  node_id   |  node_name
------------+-------------
 3953743997 | provider_node
```

### Step 7.5 — Verify network interfaces

```bash
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT if_name, if_dsn FROM pglogical.node_interface;"
```

**Expected output:**
```
   if_name    |                           if_dsn
--------------+-------------------------------------------------------------
 provider_node | host=pg-provider port=5432 dbname=providerdb user=postgres ...
```

---

## Step 8 — Replication Sets Configuration

Replication sets define **which tables and sequences** are replicated and **what operations** (INSERT, UPDATE, DELETE, TRUNCATE) are included.

### Step 8.1 — Create sample schema on Provider

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Create test schema
CREATE SCHEMA IF NOT EXISTS public;

-- Main orders table
CREATE TABLE public.orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    product     VARCHAR(100),
    quantity    INTEGER,
    price       NUMERIC(10,2),
    status      VARCHAR(20) DEFAULT 'pending',
    region      VARCHAR(50),
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Customers table
CREATE TABLE public.customers (
    customer_id SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    email       VARCHAR(100),
    country     VARCHAR(50),
    tier        VARCHAR(20) DEFAULT 'standard',
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Audit log table (will NOT be replicated)
CREATE TABLE public.audit_log (
    log_id      SERIAL PRIMARY KEY,
    action      VARCHAR(50),
    table_name  VARCHAR(100),
    logged_at   TIMESTAMP DEFAULT NOW()
);

-- Products table
CREATE TABLE public.products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(100),
    price        NUMERIC(10,2),
    stock        INTEGER,
    category     VARCHAR(50)
);
ENDSQL
```

### Step 8.2 — Create custom replication sets on Provider

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- 1. Create a replication set for critical transactional data
SELECT pglogical.create_replication_set(
    set_name         := 'critical_data',
    replicate_insert := true,
    replicate_update := true,
    replicate_delete := true,
    replicate_truncate := true
);

-- 2. Create a read-only / insert-only replication set
SELECT pglogical.create_replication_set(
    set_name         := 'insert_only',
    replicate_insert := true,
    replicate_update := false,
    replicate_delete := false,
    replicate_truncate := false
);

-- 3. Create a set for product catalog (inserts + updates, no deletes)
SELECT pglogical.create_replication_set(
    set_name         := 'catalog_data',
    replicate_insert := true,
    replicate_update := true,
    replicate_delete := false,
    replicate_truncate := false
);
ENDSQL
```

### Step 8.3 — Add tables to the default replication set

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Add orders to 'critical_data' set
SELECT pglogical.replication_set_add_table(
    set_name   := 'critical_data',
    relation   := 'public.orders',
    synchronize_data := true
);

-- Add customers to 'critical_data' set
SELECT pglogical.replication_set_add_table(
    set_name   := 'critical_data',
    relation   := 'public.customers',
    synchronize_data := true
);

-- Add products to 'catalog_data' set
SELECT pglogical.replication_set_add_table(
    set_name   := 'catalog_data',
    relation   := 'public.products',
    synchronize_data := true
);

-- NOTE: audit_log is intentionally NOT added to any replication set
ENDSQL
```

### Step 8.4 — Verify replication sets

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    rs.set_name,
    rs.replicate_insert,
    rs.replicate_update,
    rs.replicate_delete,
    rs.replicate_truncate
FROM pglogical.replication_set rs
ORDER BY rs.set_name;
ENDSQL
```

**Expected output:**
```
   set_name    | replicate_insert | replicate_update | replicate_delete | replicate_truncate
---------------+------------------+------------------+------------------+--------------------
 catalog_data  | t                | t                | f                | f
 critical_data | t                | t                | t                | t
 default       | t                | t                | t                | t
 default_insert_only | t          | f                | f                | f
 ddl_sql       | t                | t                | t                | t
 insert_only   | t                | f                | f                | f
```

### Step 8.5 — List tables in each replication set

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    set_name,
    nspname AS schema,
    relname AS table_name
FROM pglogical.replication_set_table rst
JOIN pglogical.replication_set rs USING (set_id)
JOIN pg_class c ON c.oid = rst.set_reloid
JOIN pg_namespace n ON n.oid = c.relnamespace
ORDER BY set_name, relname;
ENDSQL
```

---

## Step 9 — Table-Level Replication

### Step 9.1 — Create matching schema on Subscriber

The subscriber must have identical table definitions **before** subscribing.

```bash
podman exec -it pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
CREATE TABLE public.orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    product     VARCHAR(100),
    quantity    INTEGER,
    price       NUMERIC(10,2),
    status      VARCHAR(20) DEFAULT 'pending',
    region      VARCHAR(50),
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE public.customers (
    customer_id SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    email       VARCHAR(100),
    country     VARCHAR(50),
    tier        VARCHAR(20) DEFAULT 'standard',
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE public.products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(100),
    price        NUMERIC(10,2),
    stock        INTEGER,
    category     VARCHAR(50)
);
ENDSQL
```

### Step 9.2 — Insert test data on Provider

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Insert customers
INSERT INTO public.customers (name, email, country, tier) VALUES
  ('Alice Johnson', 'alice@example.com', 'USA', 'premium'),
  ('Bob Smith', 'bob@example.com', 'UK', 'standard'),
  ('Carol White', 'carol@example.com', 'Germany', 'enterprise'),
  ('David Lee', 'david@example.com', 'India', 'standard');

-- Insert products
INSERT INTO public.products (product_name, price, stock, category) VALUES
  ('Laptop Pro', 1299.99, 50, 'Electronics'),
  ('Wireless Mouse', 29.99, 200, 'Accessories'),
  ('USB-C Hub', 49.99, 150, 'Accessories'),
  ('Monitor 4K', 599.99, 30, 'Electronics');

-- Insert orders
INSERT INTO public.orders (customer_id, product, quantity, price, status, region) VALUES
  (1, 'Laptop Pro', 1, 1299.99, 'completed', 'North America'),
  (2, 'Wireless Mouse', 2, 59.98, 'pending', 'Europe'),
  (3, 'USB-C Hub', 3, 149.97, 'shipped', 'Europe'),
  (4, 'Monitor 4K', 1, 599.99, 'pending', 'Asia');

-- Verify
SELECT COUNT(*) AS customers FROM public.customers;
SELECT COUNT(*) AS orders FROM public.orders;
SELECT COUNT(*) AS products FROM public.products;
ENDSQL
```

---

## Step 10 — Replicate All Tables in Schema

### Step 10.1 — Add all tables in public schema to default replication set

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT pglogical.replication_set_add_all_tables(
    set_name    := 'default',
    schema_names := ARRAY['public'],
    synchronize_data := true
);
ENDSQL
```

**Expected output:**
```
 replication_set_add_all_tables
--------------------------------
 t
(1 row)
```

### Step 10.2 — Add all sequences in public schema

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT pglogical.replication_set_add_all_sequences(
    set_name    := 'default',
    schema_names := ARRAY['public'],
    synchronize_data := true
);
ENDSQL
```

### Step 10.3 — Verify all tables are included

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    rs.set_name,
    n.nspname AS schema,
    c.relname AS table_name,
    c.relkind
FROM pglogical.replication_set_table rst
JOIN pglogical.replication_set rs USING (set_id)
JOIN pg_class c ON c.oid = rst.set_reloid
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE rs.set_name = 'default'
ORDER BY c.relname;
ENDSQL
```

---

## Step 11 — Automatic Replication of New Tables

pglogical supports an **event trigger** mechanism so new tables are automatically added.

### Step 11.1 — Enable automatic table addition via event trigger on Provider

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Create an event trigger function
CREATE OR REPLACE FUNCTION public.auto_add_table_to_replication()
RETURNS event_trigger
LANGUAGE plpgsql AS $$
DECLARE
    obj record;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        IF obj.command_tag = 'CREATE TABLE' AND obj.schema_name = 'public' THEN
            BEGIN
                PERFORM pglogical.replication_set_add_table(
                    set_name := 'default',
                    relation := (obj.schema_name || '.' || obj.object_identity)::regclass,
                    synchronize_data := true
                );
                RAISE NOTICE 'Auto-added table % to replication set default', obj.object_identity;
            EXCEPTION WHEN others THEN
                RAISE WARNING 'Could not add table %: %', obj.object_identity, SQLERRM;
            END;
        END IF;
    END LOOP;
END;
$$;

-- Create the event trigger
CREATE EVENT TRIGGER auto_replicate_new_tables
ON ddl_command_end
WHEN TAG IN ('CREATE TABLE')
EXECUTE FUNCTION public.auto_add_table_to_replication();
ENDSQL
```

### Step 11.2 — Test automatic table addition

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Create a new table — should be auto-added to replication
CREATE TABLE public.inventory (
    inventory_id SERIAL PRIMARY KEY,
    product_id   INTEGER,
    warehouse    VARCHAR(50),
    quantity     INTEGER,
    updated_at   TIMESTAMP DEFAULT NOW()
);

-- Check if it was auto-added
SELECT 
    rs.set_name,
    c.relname AS table_name
FROM pglogical.replication_set_table rst
JOIN pglogical.replication_set rs USING (set_id)
JOIN pg_class c ON c.oid = rst.set_reloid
WHERE c.relname = 'inventory';
ENDSQL
```

**Expected output:**
```
 set_name |  table_name
----------+-------------
 default  | inventory
```

---

## Step 12 — Subscription Creation & Management

### Step 12.1 — Create the subscription on Subscriber

```bash
podman exec -it pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
SELECT pglogical.create_subscription(
    subscription_name := 'sub_from_provider',
    provider_dsn      := 'host=pg-provider port=5432 dbname=providerdb user=postgres password=provider_pass',
    replication_sets  := ARRAY['default', 'critical_data'],
    synchronize_structure := false,
    synchronize_data  := true,
    forward_origins   := ARRAY['all']
);
ENDSQL
```

**Expected output:**
```
 create_subscription
---------------------
          2975782184
(1 row)
```

### Step 12.2 — Monitor subscription status

```bash
podman exec pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
SELECT 
    sub_name,
    sub_enabled,
    sub_slot_name,
    sub_replication_sets,
    sub_forward_origins
FROM pglogical.subscription;
ENDSQL
```

### Step 12.3 — Check subscription worker status

```bash
podman exec pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
SELECT * FROM pglogical.show_subscription_status('sub_from_provider');
ENDSQL
```

**Expected output:**
```
 subscription_name  | status |  provider_node | replication_sets        | ...
--------------------+--------+----------------+-------------------------+
 sub_from_provider  | replicating | provider_node | {default,critical_data} |
```

### Step 12.4 — Validate data was synchronized

```bash
podman exec pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
SELECT 'customers' AS tbl, COUNT(*) FROM public.customers
UNION ALL
SELECT 'orders', COUNT(*) FROM public.orders
UNION ALL
SELECT 'products', COUNT(*) FROM public.products;
ENDSQL
```

**Expected output:**
```
    tbl    | count
-----------+-------
 customers |     4
 orders    |     4
 products  |     4
```

### Step 12.5 — Test live replication

```bash
# Insert on provider
podman exec -it pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO public.orders (customer_id, product, quantity, price, status, region) VALUES (1, 'Keyboard', 1, 89.99, 'new', 'North America');"

# Wait a moment for replication
sleep 2

# Check subscriber received the row
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT order_id, product, status FROM public.orders ORDER BY order_id DESC LIMIT 3;"
```

### Step 12.6 — Disable and re-enable a subscription

```bash
# Disable
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT pglogical.alter_subscription_disable('sub_from_provider', true);"

# Verify status
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT sub_name, sub_enabled FROM pglogical.subscription;"

# Re-enable
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT pglogical.alter_subscription_enable('sub_from_provider', true);"
```

### Step 12.7 — Drop and recreate a subscription

```bash
# Drop subscription
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT pglogical.drop_subscription('sub_from_provider', true);"

# Recreate it
podman exec -it pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
SELECT pglogical.create_subscription(
    subscription_name := 'sub_from_provider',
    provider_dsn      := 'host=pg-provider port=5432 dbname=providerdb user=postgres password=provider_pass',
    replication_sets  := ARRAY['default', 'critical_data'],
    synchronize_structure := false,
    synchronize_data  := true,
    forward_origins   := ARRAY['all']
);
ENDSQL
```

---

## Step 13 — Bi-directional Replication Setup

Bi-directional replication means both nodes are simultaneously providers and subscribers. This enables active-active scenarios.

> ⚠️ **Warning:** Bi-directional replication can cause infinite loops and conflicts. Always configure `forward_origins` carefully.

### Step 13.1 — On Subscriber, create a subscription back to Provider (reverse direction)

First create matching tables on both sides (already done). Now create reverse subscription.

```bash
# On Provider — subscribe to Subscriber
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Create a separate table for bidirectional demo
CREATE TABLE public.bidi_test (
    id         SERIAL PRIMARY KEY,
    node_name  VARCHAR(50),
    message    VARCHAR(200),
    created_at TIMESTAMP DEFAULT NOW()
);

SELECT pglogical.replication_set_add_table(
    set_name := 'default',
    relation := 'public.bidi_test',
    synchronize_data := true
);
ENDSQL
```

```bash
# On Subscriber — create matching table
podman exec -it pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
CREATE TABLE public.bidi_test (
    id         SERIAL PRIMARY KEY,
    node_name  VARCHAR(50),
    message    VARCHAR(200),
    created_at TIMESTAMP DEFAULT NOW()
);

SELECT pglogical.replication_set_add_table(
    set_name := 'default',
    relation := 'public.bidi_test',
    synchronize_data := true
);
ENDSQL
```

### Step 13.2 — Create reverse subscription (Subscriber → Provider direction)

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT pglogical.create_subscription(
    subscription_name := 'sub_from_subscriber',
    provider_dsn      := 'host=pg-subscriber port=5432 dbname=subscriberdb user=postgres password=subscriber_pass',
    replication_sets  := ARRAY['default'],
    synchronize_structure := false,
    synchronize_data  := false,
    -- CRITICAL: forward_origins prevents infinite loops
    forward_origins   := ARRAY[]::text[]
);
ENDSQL
```

### Step 13.3 — Test bidirectional write

```bash
# Write from Provider side
podman exec pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO public.bidi_test (node_name, message) VALUES ('provider_node', 'Hello from provider');"

# Write from Subscriber side
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "INSERT INTO public.bidi_test (node_name, message) VALUES ('subscriber_node', 'Hello from subscriber');"

# Wait for replication
sleep 3

# Check both sides
echo "=== Provider side ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT id, node_name, message FROM public.bidi_test ORDER BY id;"

echo "=== Subscriber side ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT id, node_name, message FROM public.bidi_test ORDER BY id;"
```

**Expected output (both sides should show both rows):**
```
 id |   node_name     |         message
----+-----------------+-------------------------
  1 | provider_node   | Hello from provider
  2 | subscriber_node | Hello from subscriber
```

---

## Step 14 — Conflict Detection & Resolution

Conflicts happen in bi-directional setups when the same row is modified on both sides simultaneously.

### Step 14.1 — Check current conflict resolution settings

```bash
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SHOW pglogical.conflict_resolution;"
```

Default is `error` — you can change it to `apply_remote`, `keep_local`, or `last_update_wins`.

### Step 14.2 — Set conflict resolution strategy

```bash
# Option 1: last_update_wins (recommended for most use cases)
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
ALTER SYSTEM SET pglogical.conflict_resolution = 'last_update_wins';
SELECT pg_reload_conf();
ENDSQL

podman exec pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
ALTER SYSTEM SET pglogical.conflict_resolution = 'last_update_wins';
SELECT pg_reload_conf();
ENDSQL
```

### Step 14.3 — Simulate a conflict

```bash
# Pause replication temporarily by disabling subscription
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT pglogical.alter_subscription_disable('sub_from_provider', true);"

# Update the SAME row on both sides simultaneously
podman exec pg-provider psql -U postgres -d providerdb \
  -c "UPDATE public.bidi_test SET message = 'Updated on PROVIDER at ' || NOW() WHERE id = 1;"

podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "UPDATE public.bidi_test SET message = 'Updated on SUBSCRIBER at ' || NOW() WHERE id = 1;"

# Re-enable subscription — conflict should be resolved by last_update_wins
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT pglogical.alter_subscription_enable('sub_from_provider', true);"

sleep 3

# Check the result (last write should win)
echo "=== Provider ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT id, message FROM public.bidi_test WHERE id = 1;"

echo "=== Subscriber ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT id, message FROM public.bidi_test WHERE id = 1;"
```

### Step 14.4 — Monitor conflict log

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    conflict_type,
    conflict_resolution,
    local_tuple,
    remote_tuple,
    local_commit_ts,
    remote_commit_ts
FROM pglogical.conflict_history
ORDER BY conflict_ts DESC
LIMIT 10;
ENDSQL
```

### Step 14.5 — Configure conflict logging

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
ALTER SYSTEM SET pglogical.conflict_log_level = 'LOG';
SELECT pg_reload_conf();
ENDSQL
```

---

## Step 15 — DDL (Schema) Replication

pglogical does **not** automatically replicate DDL. You must use `pglogical.replicate_ddl_command()`.

### Step 15.1 — Replicate DDL using pglogical function

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- This will execute the DDL on provider AND send it to subscribers
SELECT pglogical.replicate_ddl_command(
    $$ ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS notes TEXT; $$,
    ARRAY['default']
);
ENDSQL
```

### Step 15.2 — Verify DDL was applied on Subscriber

```bash
sleep 2
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "\d public.orders"
```

**Expected output — `notes` column should now appear in the subscriber schema.**

### Step 15.3 — Replicate a CREATE INDEX via DDL replication

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT pglogical.replicate_ddl_command(
    $$ CREATE INDEX IF NOT EXISTS idx_orders_status ON public.orders (status); $$,
    ARRAY['default']
);
ENDSQL
```

### Step 15.4 — Replicate ADD CONSTRAINT via DDL

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT pglogical.replicate_ddl_command(
    $$ ALTER TABLE public.orders ADD COLUMN IF NOT EXISTS priority INTEGER DEFAULT 0; $$,
    ARRAY['default']
);
ENDSQL
```

### Step 15.5 — Validate DDL replication history

```bash
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT query, queued_at FROM pglogical.queue WHERE message_type = 'S' ORDER BY queued_at DESC LIMIT 5;"
```

---

## Step 16 — Sequence Replication

pglogical supports sequence synchronization, but sequences are NOT replicated in real time — they are periodically synchronized.

### Step 16.1 — Add sequences to replication set

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Add specific sequences
SELECT pglogical.replication_set_add_sequence(
    set_name := 'default',
    relation := 'public.orders_order_id_seq',
    synchronize_data := true
);

SELECT pglogical.replication_set_add_sequence(
    set_name := 'default',
    relation := 'public.customers_customer_id_seq',
    synchronize_data := true
);
ENDSQL
```

### Step 16.2 — Manually synchronize sequences

```bash
# Force sync of sequences to subscriber
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT pglogical.synchronize_sequence('public.orders_order_id_seq');
SELECT pglogical.synchronize_sequence('public.customers_customer_id_seq');
ENDSQL
```

### Step 16.3 — Check sequence values on both sides

```bash
echo "=== Provider sequence values ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT last_value FROM public.orders_order_id_seq;"

echo "=== Subscriber sequence values ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT last_value FROM public.orders_order_id_seq;"
```

> **Note:** pglogical advances the sequence on the subscriber to avoid conflicts when the subscriber also writes to the table. The subscriber's sequence is set higher (e.g., 1000 higher) to prevent primary key collisions in bidirectional setups.

---

## Step 17 — Row-Level Filtering

Row filters allow you to replicate only rows matching a specific condition.

### Step 17.1 — Add a table with a row filter

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Only replicate orders where region = 'Europe'
SELECT pglogical.replication_set_add_table(
    set_name   := 'critical_data',
    relation   := 'public.orders',
    synchronize_data := false,
    row_filter := $$ region = 'Europe' $$
);
ENDSQL
```

### Step 17.2 — Test row filter

```bash
# Insert rows with different regions
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
INSERT INTO public.orders (customer_id, product, quantity, price, status, region)
VALUES
  (1, 'Tablet', 1, 399.99, 'pending', 'Europe'),       -- should replicate
  (2, 'Phone', 1, 799.99, 'pending', 'Asia'),           -- should NOT replicate
  (3, 'Watch', 1, 199.99, 'pending', 'North America');  -- should NOT replicate
ENDSQL
```

```bash
sleep 2

echo "=== Provider: All regions ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT order_id, product, region FROM public.orders ORDER BY order_id DESC LIMIT 5;"

echo "=== Subscriber: Should only see Europe ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT order_id, product, region FROM public.orders ORDER BY order_id DESC LIMIT 5;"
```

### Step 17.3 — More complex row filters

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Only replicate high-value orders (price > 500)
SELECT pglogical.replication_set_add_table(
    set_name   := 'insert_only',
    relation   := 'public.orders',
    synchronize_data := false,
    row_filter := $$ price > 500.00 AND status != 'cancelled' $$
);
ENDSQL
```

---

## Step 18 — Column-Level Filtering

Column filtering allows you to exclude sensitive columns from replication.

### Step 18.1 — Add table with column filtering

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Replicate customers but exclude the sensitive 'email' column
SELECT pglogical.replication_set_add_table(
    set_name   := 'catalog_data',
    relation   := 'public.customers',
    synchronize_data := true,
    columns    := ARRAY['customer_id', 'name', 'country', 'tier', 'created_at']
    -- 'email' is excluded
);
ENDSQL
```

### Step 18.2 — Verify column filtering on subscriber

```bash
# Insert a customer with email
podman exec pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO public.customers (name, email, country, tier) VALUES ('Eve Brown', 'eve@secret.com', 'Canada', 'premium');"

sleep 2

# Check subscriber — email should be NULL (not replicated)
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT customer_id, name, email, country FROM public.customers WHERE name = 'Eve Brown';"
```

**Expected output:**
```
 customer_id |   name    | email | country
-------------+-----------+-------+---------
           5 | Eve Brown | NULL  | Canada
```

---

## Step 19 — Initial Data Synchronization Control

### Step 19.1 — Create subscription WITHOUT initial sync

```bash
podman exec -it pg-node3 psql -U postgres -d node3db << 'ENDSQL'
-- Create tables first
CREATE TABLE public.orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    product     VARCHAR(100),
    quantity    INTEGER,
    price       NUMERIC(10,2),
    status      VARCHAR(20) DEFAULT 'pending',
    region      VARCHAR(50),
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Subscribe without initial data sync
SELECT pglogical.create_subscription(
    subscription_name := 'sub_from_provider_nosync',
    provider_dsn      := 'host=pg-provider port=5432 dbname=providerdb user=postgres password=provider_pass',
    replication_sets  := ARRAY['default'],
    synchronize_structure := false,
    synchronize_data  := false,     -- No initial sync
    forward_origins   := ARRAY['all']
);
ENDSQL
```

### Step 19.2 — Manually trigger synchronization of a specific table

```bash
podman exec -it pg-node3 psql -U postgres -d node3db << 'ENDSQL'
-- Sync only the orders table
SELECT pglogical.alter_subscription_synchronize_table(
    subscription_name := 'sub_from_provider_nosync',
    relation          := 'public.orders'
);
ENDSQL
```

### Step 19.3 — Check synchronization status

```bash
podman exec pg-node3 psql -U postgres -d node3db << 'ENDSQL'
SELECT * FROM pglogical.local_sync_status;
ENDSQL
```

**Expected output:**
```
  sync_kind  | sync_subid |    sync_nspname     | sync_relname | sync_status | sync_statuslsn
-------------+------------+---------------------+--------------+-------------+----------------
 f           | 2975782184 | public              | orders       | r           | ...
```

> Status `r` = ready (synchronized), `i` = initializing, `d` = data sync in progress.

### Step 19.4 — Force full resynchronization of subscription

```bash
podman exec pg-node3 psql -U postgres -d node3db << 'ENDSQL'
SELECT pglogical.alter_subscription_resynchronize_table(
    subscription_name := 'sub_from_provider_nosync',
    relation          := 'public.orders',
    truncate          := true
);
ENDSQL
```

---

## Step 20 — Monitoring & Replication Status

### Step 20.1 — Check replication lag

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    slot_name,
    plugin,
    slot_type,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS replication_lag,
    confirmed_flush_lsn,
    pg_current_wal_lsn() AS current_lsn
FROM pg_replication_slots
WHERE plugin = 'pglogical_output';
ENDSQL
```

### Step 20.2 — Check active replication workers

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag,
    write_lag,
    flush_lag,
    replay_lag AS replay_delay
FROM pg_stat_replication;
ENDSQL
```

### Step 20.3 — View pglogical worker processes

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    pid,
    datname,
    application_name,
    client_addr,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE application_name LIKE 'pglogical%'
   OR query LIKE '%pglogical%';
ENDSQL
```

### Step 20.4 — Show subscription table sync status

```bash
podman exec pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
SELECT 
    sync_kind,
    sync_nspname AS schema,
    sync_relname AS table_name,
    CASE sync_status
        WHEN 'r' THEN 'Ready'
        WHEN 'i' THEN 'Initializing'
        WHEN 'd' THEN 'Data sync'
        WHEN 'c' THEN 'Constraint sync'
        ELSE sync_status
    END AS sync_status
FROM pglogical.local_sync_status
ORDER BY sync_nspname, sync_relname;
ENDSQL
```

### Step 20.5 — Create a monitoring dashboard query

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Comprehensive monitoring query
SELECT 
    now() AS monitored_at,
    (SELECT COUNT(*) FROM pg_replication_slots WHERE active) AS active_slots,
    (SELECT COUNT(*) FROM pg_stat_replication) AS replication_connections,
    (SELECT pg_current_wal_lsn()) AS current_wal_lsn,
    (SELECT MAX(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn))
     FROM pg_replication_slots
     WHERE plugin = 'pglogical_output') AS max_replication_lag_bytes;
ENDSQL
```

### Step 20.6 — Check slot sizes and status

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT 
    slot_name,
    active,
    active_pid,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained,
    to_char(NOW() - pg_last_xact_replay_timestamp(), 'HH24:MI:SS') AS lag_time
FROM pg_replication_slots
WHERE slot_type = 'logical'
ORDER BY slot_name;
ENDSQL
```

---

## Step 21 — Performance & Load Testing

### Step 21.1 — Create load test script

```bash
cat > ~/pglogical-poc/load_test.sh << 'SCRIPT'
#!/bin/bash
echo "=== pglogical Load Test ==="
echo "Writing 10,000 rows to provider..."

podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
DO $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 1..10000 LOOP
        INSERT INTO public.orders (customer_id, product, quantity, price, status, region)
        VALUES (
            (i % 4) + 1,
            'LoadTest-Product-' || i,
            (i % 10) + 1,
            (i % 500)::NUMERIC + 9.99,
            CASE WHEN i % 3 = 0 THEN 'completed'
                 WHEN i % 3 = 1 THEN 'pending'
                 ELSE 'shipped' END,
            CASE WHEN i % 4 = 0 THEN 'Europe'
                 WHEN i % 4 = 1 THEN 'Asia'
                 WHEN i % 4 = 2 THEN 'North America'
                 ELSE 'Africa' END
        );
    END LOOP;
    RAISE NOTICE 'Inserted 10000 rows';
END;
$$;
ENDSQL

echo "Waiting 10 seconds for replication to catch up..."
sleep 10

echo ""
echo "=== Provider count ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT COUNT(*) AS provider_orders FROM public.orders;"

echo "=== Subscriber count ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT COUNT(*) AS subscriber_orders FROM public.orders;"

echo ""
echo "=== Replication lag after load ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag FROM pg_replication_slots WHERE plugin='pglogical_output';"
SCRIPT

chmod +x ~/pglogical-poc/load_test.sh
```

### Step 21.2 — Run the load test

```bash
cd ~/pglogical-poc && bash load_test.sh
```

### Step 21.3 — Use pgbench for transaction throughput testing

```bash
# Initialize pgbench on provider (uses default pgbench schema)
podman exec pg-provider pgbench -U postgres -d providerdb -i -s 10

# Run pgbench for 60 seconds
podman exec pg-provider pgbench -U postgres -d providerdb \
  -c 5 -j 2 -T 60 --report-latencies

# Check replication lag during load
watch -n 2 "podman exec pg-provider psql -U postgres -d providerdb \
  -c \"SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag FROM pg_replication_slots WHERE plugin='pglogical_output';\""
```

### Step 21.4 — Measure replication throughput

```bash
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Track WAL generation rate
SELECT 
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS current_lag,
    pg_current_wal_lsn(),
    NOW() AS measured_at
FROM pg_replication_slots
WHERE plugin = 'pglogical_output';
ENDSQL
```

---

## Step 22 — Failover & Recovery Simulation

### Step 22.1 — Simulate provider outage

```bash
# Record last replicated position on subscriber
echo "=== Subscriber state before failover ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT COUNT(*) AS orders_before_failover FROM public.orders;"

# Stop provider (simulate failure)
echo "Stopping provider..."
podman stop pg-provider

# Verify subscriber detects the failure
sleep 5
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT sub_name, sub_enabled FROM pglogical.subscription;"
```

### Step 22.2 — Verify subscriber is still readable during outage

```bash
# Subscriber should still serve reads
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT COUNT(*) AS orders_during_failover FROM public.orders;"
```

### Step 22.3 — Restore provider and verify recovery

```bash
# Restart the provider
podman start pg-provider
sleep 10

# Verify replication resumes automatically
echo "=== Subscription status after recovery ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT * FROM pglogical.show_subscription_status('sub_from_provider');"

# Insert rows on provider after recovery
podman exec pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO public.orders (customer_id, product, quantity, price, status, region) VALUES (1, 'Recovery Test', 1, 99.99, 'new', 'Europe');"

sleep 3

# Confirm replication resumed
echo "=== Subscriber after recovery ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT order_id, product, status FROM public.orders ORDER BY order_id DESC LIMIT 3;"
```

### Step 22.4 — Simulate network partition and recovery

```bash
# Disconnect subscriber from network (simulate network partition)
podman network disconnect pglogical-net pg-subscriber

# Write data during partition
podman exec pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO public.orders (customer_id, product, quantity, price, status, region) VALUES (2, 'Partition Test', 5, 250.00, 'pending', 'Asia');"

sleep 5

# Reconnect subscriber
podman network connect pglogical-net pg-subscriber

sleep 10

# Verify data replicated after reconnect (WAL is retained in slot)
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT order_id, product FROM public.orders WHERE product = 'Partition Test';"
```

### Step 22.5 — Check WAL retention during outage

```bash
# WAL is retained in the replication slot — check how much WAL is retained
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained FROM pg_replication_slots WHERE slot_type='logical';"
```

> ⚠️ **Warning:** WAL accumulates on the provider during subscriber outage. Monitor `wal_retained` to avoid disk exhaustion. Set `wal_max_size` appropriately.

---

## Step 23 — Multi-node Replication Topology

### Step 23.1 — Star topology: Provider → Subscriber & Node3

Both subscriber and node3 subscribe to the same provider.

```bash
# Create matching tables on Node3
podman exec -it pg-node3 psql -U postgres -d node3db << 'ENDSQL'
CREATE TABLE public.customers (
    customer_id SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    email       VARCHAR(100),
    country     VARCHAR(50),
    tier        VARCHAR(20) DEFAULT 'standard',
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE public.orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    product     VARCHAR(100),
    quantity    INTEGER,
    price       NUMERIC(10,2),
    status      VARCHAR(20) DEFAULT 'pending',
    region      VARCHAR(50),
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Subscribe to provider
SELECT pglogical.create_subscription(
    subscription_name := 'node3_from_provider',
    provider_dsn      := 'host=pg-provider port=5432 dbname=providerdb user=postgres password=provider_pass',
    replication_sets  := ARRAY['default', 'critical_data'],
    synchronize_structure := false,
    synchronize_data  := true,
    forward_origins   := ARRAY['all']
);
ENDSQL
```

### Step 23.2 — Verify star topology replication

```bash
sleep 5

# Insert on provider
podman exec pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO public.customers (name, email, country, tier) VALUES ('Star Node Test', 'star@test.com', 'Japan', 'enterprise');"

sleep 3

echo "=== Provider ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT COUNT(*) FROM public.customers;"

echo "=== Subscriber ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT COUNT(*) FROM public.customers;"

echo "=== Node3 ==="
podman exec pg-node3 psql -U postgres -d node3db \
  -c "SELECT COUNT(*) FROM public.customers;"
```

### Step 23.3 — Cascade topology: Provider → Subscriber → Node3

```bash
# On Subscriber — forward changes to Node3
podman exec -it pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
-- Make subscriber also a provider
SELECT pglogical.replication_set_add_all_tables(
    set_name     := 'default',
    schema_names := ARRAY['public'],
    synchronize_data := false
);
ENDSQL
```

```bash
# On Node3 — subscribe to Subscriber instead of Provider
podman exec pg-node3 psql -U postgres -d node3db \
  -c "SELECT pglogical.drop_subscription('node3_from_provider', true);"

podman exec -it pg-node3 psql -U postgres -d node3db << 'ENDSQL'
SELECT pglogical.create_subscription(
    subscription_name := 'node3_from_subscriber',
    provider_dsn      := 'host=pg-subscriber port=5432 dbname=subscriberdb user=postgres password=subscriber_pass',
    replication_sets  := ARRAY['default'],
    synchronize_structure := false,
    synchronize_data  := true,
    forward_origins   := ARRAY['all']
);
ENDSQL
```

### Step 23.4 — Verify cascade replication

```bash
# Write to provider
podman exec pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO public.customers (name, email, country) VALUES ('Cascade Test', 'cascade@test.com', 'Brazil');"

sleep 5

echo "Provider → Subscriber → Node3 cascade:"
for container in pg-provider pg-subscriber pg-node3; do
  echo -n "$container: "
  podman exec $container psql -U postgres -d ${container/pg-/}db \
    -c "SELECT COUNT(*) AS count FROM public.customers;" 2>/dev/null | grep -E '^[[:space:]]+[0-9]'
done
```

---

## Step 24 — Selective Replication Sets per Subscription

### Step 24.1 — Create specialized replication sets

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- European orders replication set
SELECT pglogical.create_replication_set(
    set_name := 'eu_orders',
    replicate_insert := true,
    replicate_update := true,
    replicate_delete := true,
    replicate_truncate := true
);

-- Add orders with EU row filter
SELECT pglogical.replication_set_add_table(
    set_name   := 'eu_orders',
    relation   := 'public.orders',
    synchronize_data := false,
    row_filter := $$ region IN ('Europe', 'UK') $$
);

-- Asia orders replication set
SELECT pglogical.create_replication_set(
    set_name := 'asia_orders',
    replicate_insert := true,
    replicate_update := true,
    replicate_delete := true,
    replicate_truncate := true
);

SELECT pglogical.replication_set_add_table(
    set_name   := 'asia_orders',
    relation   := 'public.orders',
    synchronize_data := false,
    row_filter := $$ region IN ('Asia', 'India', 'Japan') $$
);
ENDSQL
```

### Step 24.2 — Create subscriptions that use different sets

```bash
# Node3 subscribes ONLY to EU orders
podman exec pg-node3 psql -U postgres -d node3db \
  -c "SELECT pglogical.drop_subscription('node3_from_subscriber', true);" 2>/dev/null || true

podman exec -it pg-node3 psql -U postgres -d node3db << 'ENDSQL'
SELECT pglogical.create_subscription(
    subscription_name := 'node3_eu_only',
    provider_dsn      := 'host=pg-provider port=5432 dbname=providerdb user=postgres password=provider_pass',
    replication_sets  := ARRAY['eu_orders'],
    synchronize_structure := false,
    synchronize_data  := true,
    forward_origins   := ARRAY['all']
);
ENDSQL
```

### Step 24.3 — Validate selective replication

```bash
# Insert orders for different regions
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
INSERT INTO public.orders (customer_id, product, quantity, price, status, region)
VALUES
  (1, 'EU Product', 1, 100.00, 'new', 'Europe'),
  (2, 'Asia Product', 1, 100.00, 'new', 'Asia'),
  (3, 'US Product', 1, 100.00, 'new', 'North America');
ENDSQL

sleep 3

echo "=== Provider: All 3 orders ==="
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT product, region FROM public.orders WHERE product IN ('EU Product','Asia Product','US Product');"

echo "=== Node3 (EU only): Should only see EU Product ==="
podman exec pg-node3 psql -U postgres -d node3db \
  -c "SELECT product, region FROM public.orders WHERE product IN ('EU Product','Asia Product','US Product');"
```

---

## Step 25 — Schema-Level Replication Behavior

### Step 25.1 — Create a separate schema and replicate it

```bash
podman exec -it pg-provider psql -U postgres -d providerdb << 'ENDSQL'
-- Create analytics schema
CREATE SCHEMA IF NOT EXISTS analytics;

CREATE TABLE analytics.daily_summary (
    summary_id   SERIAL PRIMARY KEY,
    summary_date DATE DEFAULT CURRENT_DATE,
    total_orders INTEGER,
    total_revenue NUMERIC(12,2),
    created_at   TIMESTAMP DEFAULT NOW()
);

-- Add all tables in analytics schema
SELECT pglogical.replication_set_add_all_tables(
    set_name     := 'default',
    schema_names := ARRAY['analytics'],
    synchronize_data := true
);
ENDSQL
```

### Step 25.2 — Create matching schema on subscriber

```bash
podman exec -it pg-subscriber psql -U postgres -d subscriberdb << 'ENDSQL'
CREATE SCHEMA IF NOT EXISTS analytics;

CREATE TABLE analytics.daily_summary (
    summary_id   SERIAL PRIMARY KEY,
    summary_date DATE DEFAULT CURRENT_DATE,
    total_orders INTEGER,
    total_revenue NUMERIC(12,2),
    created_at   TIMESTAMP DEFAULT NOW()
);
ENDSQL
```

### Step 25.3 — Test schema-level replication

```bash
podman exec pg-provider psql -U postgres -d providerdb \
  -c "INSERT INTO analytics.daily_summary (total_orders, total_revenue) VALUES (150, 45678.90);"

sleep 2

echo "=== Subscriber analytics schema ==="
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT * FROM analytics.daily_summary;"
```

### Step 25.4 — Validate schema exclusions

```bash
# Check which schemas ARE being replicated
podman exec pg-provider psql -U postgres -d providerdb << 'ENDSQL'
SELECT DISTINCT
    n.nspname AS schema,
    COUNT(*) AS tables_in_replication
FROM pglogical.replication_set_table rst
JOIN pglogical.replication_set rs USING (set_id)
JOIN pg_class c ON c.oid = rst.set_reloid
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE rs.set_name = 'default'
GROUP BY n.nspname
ORDER BY n.nspname;
ENDSQL
```

---

## Troubleshooting

### Common Issue 1 — Extension not loading

**Symptom:** `ERROR: could not open extension control file`

```bash
# Verify pglogical is installed in the container
podman exec pg-provider bash -c "ls /usr/share/postgresql/15/extension/pglogical*"

# Verify shared_preload_libraries is set
podman exec pg-provider psql -U postgres -c "SHOW shared_preload_libraries;"

# If missing — restart container
podman restart pg-provider
```

### Common Issue 2 — Subscription stuck in 'initializing'

**Symptom:** `SELECT * FROM pglogical.show_subscription_status()` returns `initializing` indefinitely.

```bash
# Check worker process logs
podman logs pg-subscriber --tail 50

# Check replication slot on provider
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT slot_name, active, confirmed_flush_lsn FROM pg_replication_slots;"

# Drop and recreate subscription
podman exec pg-subscriber psql -U postgres -d subscriberdb \
  -c "SELECT pglogical.drop_subscription('sub_from_provider', true);"
```

### Common Issue 3 — pg_hba.conf authentication failure

**Symptom:** `FATAL: pg_hba.conf rejects connection`

```bash
# Check pg_hba.conf entries
podman exec pg-provider cat /var/lib/postgresql/data/pg_hba.conf | grep -v "^#" | grep -v "^$"

# Test connection manually
podman exec pg-subscriber psql \
  "host=pg-provider port=5432 dbname=providerdb user=postgres password=provider_pass" \
  -c "SELECT 1;"

# Reload after changes
podman exec pg-provider psql -U postgres -c "SELECT pg_reload_conf();"
```

### Common Issue 4 — Replication slot accumulating WAL

**Symptom:** Disk filling up on provider.

```bash
# Check WAL retained per slot
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained FROM pg_replication_slots ORDER BY restart_lsn;"

# If subscriber is gone permanently, drop the slot
podman exec pg-provider psql -U postgres \
  -c "SELECT pg_drop_replication_slot('slot_name_here');"
```

### Common Issue 5 — Table not found during replication

**Symptom:** `ERROR: table "orders" does not exist`

```bash
# Verify table exists on both sides with identical structure
podman exec pg-provider psql -U postgres -d providerdb -c "\d public.orders"
podman exec pg-subscriber psql -U postgres -d subscriberdb -c "\d public.orders"

# Verify the table has a PRIMARY KEY (required by pglogical)
podman exec pg-provider psql -U postgres -d providerdb \
  -c "SELECT contype, conname FROM pg_constraint WHERE conrelid = 'public.orders'::regclass;"
```

> ⚠️ **pglogical requires tables to have a PRIMARY KEY** for UPDATE and DELETE replication.

### Common Issue 6 — Conflict errors in logs

```bash
# View PostgreSQL logs for conflict details
podman logs pg-subscriber --tail 100 | grep -i conflict

# Change conflict resolution
podman exec pg-provider psql -U postgres \
  -c "ALTER SYSTEM SET pglogical.conflict_resolution = 'last_update_wins'; SELECT pg_reload_conf();"
```

### Common Issue 7 — max_replication_slots exceeded

```bash
# Check current slots
podman exec pg-provider psql -U postgres \
  -c "SELECT COUNT(*) FROM pg_replication_slots;"

# Increase limit
podman exec pg-provider psql -U postgres \
  -c "ALTER SYSTEM SET max_replication_slots = 20; SELECT pg_reload_conf();"

# Requires restart
podman restart pg-provider
```

---

## Quick Reference Cheat Sheet

```bash
# ============================================================
# NODE MANAGEMENT
# ============================================================
# Create node
SELECT pglogical.create_node(node_name := 'name', dsn := 'connection_string');

# Drop node
SELECT pglogical.drop_node('node_name', ifexists := true);

# List nodes
SELECT * FROM pglogical.node;

# ============================================================
# REPLICATION SET MANAGEMENT
# ============================================================
# Create replication set
SELECT pglogical.create_replication_set('set_name');

# Add table
SELECT pglogical.replication_set_add_table('set_name', 'schema.table', true);

# Add table with row filter
SELECT pglogical.replication_set_add_table('set_name', 'schema.table', false, $$ column = 'value' $$);

# Add table with column filter
SELECT pglogical.replication_set_add_table('set_name', 'schema.table', true, NULL, ARRAY['col1','col2']);

# Add all tables in schema
SELECT pglogical.replication_set_add_all_tables('set_name', ARRAY['schema_name'], true);

# Remove table
SELECT pglogical.replication_set_remove_table('set_name', 'schema.table');

# List tables in set
SELECT * FROM pglogical.replication_set_table;

# ============================================================
# SUBSCRIPTION MANAGEMENT
# ============================================================
# Create subscription
SELECT pglogical.create_subscription(
  subscription_name := 'sub_name',
  provider_dsn := 'host=... dbname=... user=... password=...',
  replication_sets := ARRAY['default'],
  synchronize_data := true
);

# Drop subscription
SELECT pglogical.drop_subscription('sub_name', true);

# Enable/disable
SELECT pglogical.alter_subscription_enable('sub_name', true);
SELECT pglogical.alter_subscription_disable('sub_name', true);

# Check status
SELECT * FROM pglogical.show_subscription_status('sub_name');

# ============================================================
# MONITORING
# ============================================================
# Replication lag
SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots WHERE plugin = 'pglogical_output';

# Active connections
SELECT * FROM pg_stat_replication;

# Sync status
SELECT * FROM pglogical.local_sync_status;

# ============================================================
# DDL REPLICATION
# ============================================================
SELECT pglogical.replicate_ddl_command($$ ALTER TABLE ... $$, ARRAY['default']);

# ============================================================
# CONFLICT RESOLUTION OPTIONS
# ============================================================
# error | apply_remote | keep_local | last_update_wins
ALTER SYSTEM SET pglogical.conflict_resolution = 'last_update_wins';
SELECT pg_reload_conf();

# ============================================================
# CLEANUP
# ============================================================
# Drop all containers and volumes
podman stop pg-provider pg-subscriber pg-node3
podman rm pg-provider pg-subscriber pg-node3
podman volume rm pglogical-provider-data pglogical-subscriber-data pglogical-node3-data
podman network rm pglogical-net
```

---

## Summary of Demonstrated Features

| Feature | Steps | Status |
|---------|-------|--------|
| Basic Node Setup | 7 | ✅ |
| Replication Sets | 8 | ✅ |
| Table-Level Replication | 9 | ✅ |
| Schema-Wide Replication | 10 | ✅ |
| Auto New Table Replication | 11 | ✅ |
| Subscription Management | 12 | ✅ |
| Bi-directional Replication | 13 | ✅ |
| Conflict Detection/Resolution | 14 | ✅ |
| DDL Replication | 15 | ✅ |
| Sequence Replication | 16 | ✅ |
| Row-Level Filtering | 17 | ✅ |
| Column-Level Filtering | 18 | ✅ |
| Initial Data Sync Control | 19 | ✅ |
| Monitoring & Status | 20 | ✅ |
| Load Testing | 21 | ✅ |
| Failover & Recovery | 22 | ✅ |
| Multi-node Topology | 23 | ✅ |
| Selective Replication Sets | 24 | ✅ |
| Schema-Level Replication | 25 | ✅ |

---

and expected outputs. Refer to the [Troubleshooting](#troubleshooting) section for common issues, and the [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet) for day-to-day operations.
