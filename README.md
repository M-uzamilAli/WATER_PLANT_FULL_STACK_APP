# 💧 RO Water Plant Management System

A full-stack, offline-first management system for an RO (Reverse Osmosis) water plant — built to handle customer records, daily sales/dispatch, billing, and business stats on a shop's local network, with no external cloud dependency required.

---

## 🧩 Tech Stack

| Layer      | Technology                                              |
|------------|-----------------------------------------------------------|
| Backend    | FastAPI (Python), `psycopg2` for raw SQL access            |
| Frontend   | React 19 + Vite, `recharts` for charts, `sonner` for toasts |
| Database   | PostgreSQL — business logic lives in stored procedures/functions, organized by schema |
| Auth       | Username/password login, `bcrypt` password hashing via `passlib` |
| Logging    | `loguru`, with rotating file logs under `backend/logs/`    |

---

## 🧠 What It Does

- **Customer management** — add, update, search, filter (by name, ID, active status), and view stats for RO water customers.
- **Sales & dispatch** — record daily sales (account / COD / vendor), batch-dispatch the day's regular customers, track per-sale status (`pending` → `paid`), and filter/search sales history.
- **Billing** — generate a monthly bill per customer from their account sales, mark bills paid/unpaid, and protect cash-settled sales from being altered by the billing cycle (see [Billing logic](#-billing-logic) below).
- **Stats dashboard** — a "pulse" of the current month (revenue split, bottles, litres), a VIP customer leaderboard, and a 6-month sales trend.

---

## 🗂️ Project Structure

```
WATER_PLANT_FULL_STACK_APP/
├── backend/
│   ├── main.py                 # FastAPI app, CORS, exception handlers, /login, router mounts
│   ├── auth.py                 # Login/password verification (bcrypt)
│   ├── database.py             # DB connection config + GET_DB() cursor dependency
│   ├── validation.py           # Pydantic request/response models
│   ├── logger_config.py        # loguru setup
│   ├── customer_module/
│   │   └── customer.py         # /customer/* routes
│   ├── sales_module/
│   │   └── sales.py            # /sales/* routes
│   ├── billings_module/
│   │   └── billings.py         # /billings/* routes
│   ├── stats_module/
│   │   └── stats.py            # /stats/* routes
│   ├── seed_demo_data.py       # Seeds demo customers/sales
│   ├── seed_users.py           # Seeds login users
│   ├── dump_all_customer_from_excel_to_database.py  # One-off Excel → DB import
│   └── requirement.txt
├── database_migrations/
│   └── migration_pipelines/
│       ├── schemas/            # Table/schema definitions (customers, sales, billings, stats, users)
│       ├── modules/            # Stored procedures & functions, grouped per module
│       ├── utils.txt           # Dispatch helper functions
│       └── audit_triggers.txt  # Audit trail triggers
└── frontend/
    └── src/
        ├── App.jsx              # Tab layout: Customers / Sales / Dispatch / Stats / Billings
        └── components/
            ├── CustomerSection/, AllCustomersDetailsSection/, CustomerDetailsCard/, CustomerStats/
            ├── SalesSection/, AllSalesDetailsSection/, SalesDetailsCard/, SalesStats/
            ├── DailyDispatchSection/
            ├── BillingsSection/
            ├── StatsSection/
            ├── SideBar/
            └── login/
```

---

## 🚀 Running Locally

### 1. Prerequisites

- Python ≥ 3.11
- Node.js ≥ 20
- A running PostgreSQL instance

### 2. Set up the database

Run the schema and stored-procedure scripts under `database_migrations/migration_pipelines/` against your Postgres instance, in this order: `schemas/` (creates the schemas, tables, enums) → `modules/` (creates the stored procedures/functions each route calls) → `utils.txt` and `audit_triggers.txt`. Create a database named to match `database.py` (see [Configuration](#-configuration) below), then seed it:

```bash
cd backend
python seed_users.py          # creates login user(s)
python seed_demo_data.py      # optional — demo customers/sales
```

### 3. Install dependencies

```bash
# Backend
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1    # Windows PowerShell (use source .venv/bin/activate on macOS/Linux)
pip install -r requirement.txt

# Frontend
cd ../frontend
npm install
```

### 4. Run

```bash
# Backend (from backend/)
fastapi dev main.py
# → http://127.0.0.1:8000

# Frontend (from frontend/, separate terminal)
npm run dev
# → http://localhost:5173
```

> The top-level README previously referenced a `run_script.py` to start both servers together and a backend port of `8001` — if that script isn't present in your checkout, run the two commands above in separate terminals instead.

### 5. Configuration

Database credentials currently live directly in `backend/database.py`:

```python
DB_CONFIG = {
    "host": "localhost",
    "port": "5432",
    "database": "tulip-db",
    "user": "postgres",
    "password": "newpassword123",
}
```

**Before deploying anywhere beyond your own machine, move these into environment variables** (e.g. via `python-dotenv`) rather than committing real credentials. This is the single most important pre-deployment change.

---

## 🔌 API Reference

All responses follow a consistent envelope:

```json
{ "status": true, "data": ..., "message": "..." }
```

On error, FastAPI's exception handlers return:

```json
{ "success": false, "error": { "code": 422, "message": "INVALID DATA." } }
```

### Auth

| Method | Route     | Body                          | Description                          |
|--------|-----------|--------------------------------|---------------------------------------|
| POST   | `/login`  | `{ username, password }`       | Verifies credentials, returns `user_id` on success |

### Customers — `/customer`

| Method | Route              | Description                                              |
|--------|--------------------|------------------------------------------------------------|
| GET    | `/`                | List all customers                                          |
| GET    | `/stats`           | Aggregate customer stats                                    |
| GET    | `/filter?q=`        | Filter: `id-asc` (default) · `name-asc` · `name-desc` · `active` |
| GET    | `/search?q=`        | Search customers by name/phone/etc.                         |
| GET    | `/{id}`             | Get a single customer                                       |
| POST   | `/add`              | Add a customer (body: `User`)                                |
| PUT    | `/update/{id}`      | Partial update of a customer (body: `User`)                  |

### Sales — `/sales`

| Method | Route                    | Description                                                         |
|--------|--------------------------|------------------------------------------------------------------------|
| GET    | `/dispatch_customers`     | List regular customers due for today's dispatch                        |
| GET    | `/dispatch_today`         | Whether today's batch has already been dispatched                      |
| POST   | `/dispatch`               | Dispatch a batch: marks the day dispatched + bulk-inserts a `Sales` list |
| POST   | `/add`                    | Add a single sale (body: `Sales`)                                       |
| PUT    | `/update`                 | Update a sale's status (body: `Update_Sales`)                           |
| PUT    | `/filter?q=`              | Filter (body: `DateRange`): `id-asc` (default) · `price-desc` · `status` |
| PUT    | `/all`                    | All sales in a date range (body: `DateRange`)                           |
| GET    | `/stats`                  | Aggregate sales stats                                                    |
| GET    | `/search?q=`              | Search sales                                                             |
| GET    | `/search_customer/{q}`    | Search sales by customer name                                            |
| GET    | `/{id}`                   | Get a single sale                                                        |
| GET    | `/flush`                  | ⚠️ Deletes **all** sales rows — dev/testing only                          |
| GET    | `/flushCust`              | ⚠️ Deletes **all** customer rows — dev/testing only                       |

### Billings — `/billings`

| Method | Route                     | Description                                                       |
|--------|---------------------------|-----------------------------------------------------------------------|
| GET    | `/summary?month=&year=`    | Billing summary for a month                                          |
| GET    | `/customer/{cust_id}?month=&year=` | A single customer's bill for a month                        |
| POST   | `/mark_paid`               | Mark a customer's bill paid (body: `{cust_id, month, year, user_id?}`) |
| POST   | `/mark_unpaid`             | Mark a customer's bill unpaid (same body)                            |
| POST   | `/send_all`                | Generate bills for every customer with pending account sales for a month (idempotent — body: `{month, year, user_id?}`) |

### Stats — `/stats`

| Method | Route     | Description                                             |
|--------|-----------|-----------------------------------------------------------|
| GET    | `/pulse`   | Current month: revenue split, total bottles, total litres  |
| GET    | `/vip`     | Top 7 customers this month                                 |
| GET    | `/trend`   | Last 6 months of sales, oldest first, split by sale type    |

---

## 🗄️ Database Design

Business logic is deliberately kept in PostgreSQL, not the application layer — routes call stored procedures/functions (`schema_x.some_function()`), so the API stays thin and validation/constraints live next to the data.

**Schemas**: `schema_customers`, `schema_sales`, `schema_billings`, `schema_stats`, `schema_users`, plus a `utils` schema for dispatch helpers.

Key tables:

- **`schema_customers.customers`** — name, phone (`03XXXXXXXXX` format, enforced by a `CHECK`), address, `unit_price`, `advance_money`, `regular_bottles`, `is_active`, and an audit trail (`modified_by`/`modified_at`).
- **`schema_sales.sales`** — `sales_type` enum (`account` / `cod` / `vendor`), `sales_status` enum (`pending` / `paid` / `deleted`), and a `billing_locked` flag.
- **`schema_billings.bills`** — one row per `(customer, month, year)` (enforced by a unique constraint), storing a point-in-time `amount` snapshot and a `paid`/`unpaid` status.
- **`schema_users.users`** — `user_name`, bcrypt `password` hash, `role`.

### 💳 Billing logic

The billing model is designed so that paying a bill for one month can never silently affect another month's numbers, and so a sale paid in cash at the counter can't be re-opened by a batch billing run:

1. Each `(customer, month, year)` gets exactly one `bills` row. `bills.amount` is a **snapshot** taken at generation time — it does not re-derive from `sales` on every read.
2. `schema_sales.sales.billing_locked` protects cash-paid sales:
   - A sale inserted as `paid` is auto-locked via an `INSERT` trigger.
   - Flipping a sale's status in the Sales UI auto-flips the lock via an `UPDATE` trigger.
   - `mark_paid` / `mark_unpaid` in the billing module only ever touch sales where `billing_locked = FALSE`, so cash-settled entries can't be corrupted by a billing pass. The billing module sets a session flag to intentionally bypass the auto-lock trigger when it needs to update status itself.
3. `send_all` (`POST /billings/send_all`) is **idempotent** — re-running it for a month that's already fully billed generates zero new rows.

---

## 🖥️ Frontend

A single-page tabbed dashboard (`App.jsx`) with five sections — **Customers**, **Sales**, **Dispatch**, **Stats**, **Billings** — each its own component tree, mounted together and toggled by visibility rather than routed, so state (e.g. selected customer) persists across tab switches. Customer and sales data are cross-refreshed: updating a customer's dispatch on the Sales tab refreshes the Customers list, and vice versa.

- **Login** — `components/login/` handles the credential form and calls `POST /login`.
- **Toasts** — `sonner`, themed via custom `success-toast` / `error-toast` / `info-toast` classes.
- **Charts** — `recharts`, used in `StatsSection` for the sales trend.

---

## 📦 Deployment Notes

- Move `DB_CONFIG` (and any other secrets) into environment variables before deploying anywhere shared.
- Frontend → Vercel (or any static host, after `npm run build`).
- Backend → Render, Fly.io, or any host that runs a long-lived FastAPI/Uvicorn process.
- Database → Supabase, Heroku Postgres, or any managed Postgres instance — the schemas/functions under `database_migrations/` need to be applied to it first.
- `origins = ["*"]` (open CORS) in `main.py` should be tightened to the actual frontend origin(s) before going public.

---

