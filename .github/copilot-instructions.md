# Copilot Instructions for sql-analytics-portfolio

## Project Overview
This is a SQL analytics portfolio project featuring a PostgreSQL database with a sample retail analytics schema (customers, employees, products, orders, sales). The entire stack runs in Docker with pgAdmin for query interface.

## Architecture & Data Model

### Core Tables (see `init/01_schema.sql`)
- **employees** → **sales**: One employee can have multiple sales (nullable FK)
- **customers** → **sales**: One customer has multiple sales (required FK)  
- **products** → **sales**: One product appears in multiple sales (required FK)
- **orders** → **sales**: One order links to multiple sales (required FK)
- **orders**: Pre-computed `year`, `quarter`, `month` stored as columns (denormalized for analytics queries)

### Key Pattern
Sales is the fact table with four dimensions. Foreign keys use `ON DELETE RESTRICT` except employees (SET NULL) to prevent accidental data loss.

## Development Workflow

### Local Setup
1. Create `.env` file with `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `PGADMIN_EMAIL`, `PGADMIN_PASSWORD`
2. `docker-compose up -d` — starts PostgreSQL (port 5432) and pgAdmin (port 5050)
3. Container mounts `./init` scripts auto-run on first startup via `docker-entrypoint-initdb.d`
4. Access pgAdmin at `http://localhost:5050`

### Database Initialization
- **01_schema.sql**: Creates all 5 tables with indexes and constraints
- **02_etl.sql**: COPY commands load 5 CSV files from `./data` into containers via Docker volume mount
- Scripts run sequentially in filename order (PostgreSQL convention)
- CSV mount path inside container: `/docker-entrypoint-initdb.d/data/` (read-only)

### Resetting Database
Edit `01_schema.sql` to uncomment DROP TABLE statements, then:
```bash
docker-compose down -v  # -v removes volumes
docker-compose up -d    # Re-initializes
```

## Key Conventions

### Naming
- Table names: singular (`sales`, `orders`, not `sales_fact`)
- ID columns: `{table}_id` (e.g., `employee_id`, `transaction_id`)
- Foreign keys: same as referenced table's PK name
- Indexes: `idx_{table}_{column}` pattern (`idx_sales_order_id`)

### Temporal Data
Orders table denormalizes time dimensions (`year`, `quarter`, `month`) as stored columns rather than computed. This pattern avoids repeated extraction in analytics queries—ETL scripts pre-compute these from `order_date`.

### Constraints
- Primary keys: SERIAL or INTEGER
- Foreign keys: Restrict deletion by default (except employees → SET NULL)
- NULL handling: `employee_id` nullable (employee can be deleted); others NOT NULL

## Important Files

| File | Purpose |
|------|---------|
| `docker-compose.yaml` | Database (PostgreSQL 17) + pgAdmin services, volume mounts, healthchecks |
| `init/01_schema.sql` | Schema: tables, PKs, FKs, indexes |
| `init/02_etl.sql` | COPY commands to load 5 CSVs (note: file named `02_etl.sql` but contents say `03_seed_from_csv`) |
| `data/*.csv` | Source data for employees, customers, products, orders, sales |
| `.env` | Environment variables for Docker services (user-created, not in repo) |

## Common Tasks

**Query sales by customer**: Join `sales` → `customers` on `customer_id`
**Analyze orders over time**: Use pre-computed `year`, `quarter`, `month` columns (no need for date functions in WHERE/GROUP BY)
**Add new data**: Update CSVs in `data/` or insert directly via pgAdmin; container volume is live
**Inspect running DB**: Connect pgAdmin to container host `db:5432` (service name in docker-compose)

## Notes for AI Agents
- Docker volume paths in SQL scripts use container-side paths (`/docker-entrypoint-initdb.d/data/`), not host paths
- Filename comment mismatch in `02_etl.sql` (says `03_seed_from_csv.sql`) — scripts run by filename order, not comment
- On database reset, old `postgres_data/` directory must be removed; `docker-compose down -v` handles this
