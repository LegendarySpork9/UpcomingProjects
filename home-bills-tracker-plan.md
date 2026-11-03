# HomeBillsTracker — Complete Project Plan

## Overview

A self-hosted household management web application for tracking food shopping and household bills. One installation per property, hosted on a Raspberry Pi 5. Residents have accounts with role-based access. The system tracks shopping prices over time, forecasts bills using historical and seasonal data, and exposes a REST API for integration with external projects (initially a C# virtual assistant).

**Core principles:**
- Single property per installation, fully isolated
- Collaborative — all residents contribute to shared data
- Analytics-driven — every data entry feeds dashboards and forecasting
- Secure by default — HTTPS, authentication on everything, encrypted file storage
- Mobile-first responsive design

---

## Tech Stack

- **Frontend:** React (mobile-first responsive SPA, Tailwind CSS, Recharts for charting)
- **Backend:** Python / FastAPI
- **Database:** SQLite (WAL mode for concurrent read access)
- **ORM:** SQLAlchemy with Alembic for migrations
- **OCR:** Tesseract with Python bindings
- **Forecasting:** Facebook Prophet / statsmodels for seasonal time-series
- **API docs:** Auto-generated OpenAPI (built into FastAPI)
- **Deployment:** Docker Compose
- **Reverse proxy:** Nginx with SSL termination
- **SSL:** WinACME with domain-attached certificate
- **Testing:** pytest (backend only)

**Why this stack:** The project is fundamentally data-driven — price tracking, trend analysis, seasonal forecasting, OCR processing. Python has the strongest ecosystem for all of these. FastAPI auto-generates OpenAPI docs, giving the virtual assistant a self-documenting API for free. SQLite eliminates the overhead of a separate database server process, which is ideal for a single-property household app with a small number of concurrent users.

**SQLite notes:** SQLite serialises writes but handles concurrent reads well. WAL (Write-Ahead Logging) mode further improves concurrency. For a household app with a handful of residents, write contention is negligible. Backup is as simple as copying a single file.

---

## Repository Structure

Monorepo — single repository, single Docker Compose, everything versioned together.

```
HomeBillsTracker/
  ├── frontend/          (React app)
  ├── backend/           (FastAPI app)
  ├── alembic/           (database migrations)
  ├── docker-compose.yml
  ├── deploy.sh          (deployment script)
  ├── nginx/             (Nginx config)
  └── README.md
```

---

## Pi 5 Hardware Recommendations

With SQLite (no separate database server process), RAM requirements drop significantly.

### Minimum (comfortable for this project)
- Pi 5 **4GB RAM**
- 32GB microSD card
- Basic case with passive cooling
- Standard PSU

### Recommended
- Pi 5 **4GB RAM**
- **256GB NVMe SSD** via Pi 5 M.2 HAT
- Active cooling case (e.g. Argon ONE or official active cooler)
- Official 27W PSU
- UPS HAT for power protection

### Optional (future-proofing)
- Pi 5 **8GB RAM** — only needed if you plan to run additional services on the same Pi

### Why NVMe still matters
SQLite writes frequently to a single database file. SD cards degrade under sustained write workloads and have slower random I/O. An NVMe SSD via the M.2 HAT gives better durability and performance. Boot from SD, store data on NVMe — or boot entirely from NVMe.

---

## System Architecture

```
┌─────────────────────────────────────────────┐
│                  Nginx                       │
│       (Reverse Proxy + SSL via WinACME)      │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────┐    ┌───────────────────┐  │
│  │  React SPA   │    │   FastAPI Backend  │  │
│  │  (Frontend)  │◄──►│                   │  │
│  │              │    │  ├─ Auth (JWT)     │  │
│  │  - Dashboard │    │  ├─ Shopping API   │  │
│  │  - Shopping  │    │  ├─ Bills API      │  │
│  │  - Products  │    │  ├─ Analytics API  │  │
│  │  - Bills     │    │  ├─ OCR Service    │  │
│  │  - Alerts    │    │  ├─ Forecast Eng.  │  │
│  │  - Admin     │    │  ├─ Alert Engine   │  │
│  └──────────────┘    │  ├─ Smart Meter    │  │
│                      │  │   Adapters      │  │
│                      │  └─ Price Source   │  │
│                      │      Adapters      │  │
│                      └─────────┬─────────┘  │
│                                │             │
│                      ┌─────────▼─────────┐  │
│                      │   SQLite (WAL)     │  │
│                      │                   │  │
│                      │  + File Storage   │  │
│                      │    (bill PDFs,    │  │
│                      │     receipt imgs) │  │
│                      └───────────────────┘  │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  External REST API (authenticated)   │   │
│  │  JSON responses for any consumer     │   │
│  │  → Virtual Assistant (C#)            │   │
│  │  → Future integrations               │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

---

## Navigation Structure

Collapsible sidebar on desktop, hamburger menu on mobile.

```
📊 Dashboard
🛒 Shopping
   ├── Shopping Lists
   └── New List
📦 Products
💡 Bills
   ├── Electricity
   ├── Gas
   ├── Water
   ├── Council Tax
   ├── Broadband
   ├── Rent/Mortgage
   └── Insurance
🔔 Alerts (badge count)
⚙️ Admin (admin only)
   ├── Users
   ├── Stores
   ├── Categories
   ├── API Keys
   └── Settings
```

Bill type sub-pages only appear for types enabled during the first-run wizard. Additional types added later appear automatically.

---

## Data Model

Products belong to a store — "Tesco Pack of Burgers" and "ASDA Pack of Burgers" are separate product records. The product name reflects the package as sold (e.g. "Pack of Burgers", "Semi-Skimmed Milk 2L"). Unit price is calculated as total price / quantity purchased.

```
Properties
  ├── Users (roles: admin, resident)
  └── Settings
        ├── currency (GBP, USD, EUR, etc. — set once during setup)
        ├── backup_location (configurable path)
        └── other site-wide config

Stores (pre-seeded: ASDA, Aldi, Tesco, Lidl, Sainsbury's, Morrisons,
        Waitrose, Co-op, M&S Food, Iceland — user-addable)
  └── Products (name, category, current_price)
        └── PriceHistory (price, date_logged)
              — new entry each time a receipt shows a different REGULAR price
              — product.current_price always reflects the latest regular price
              — sale prices do NOT create entries or update current_price

Categories (pre-defined: dairy, meat, bakery, frozen, fruit & veg,
            drinks, household, etc. — admin can extend)

ShoppingLists
  ├── store, status (draft → active → purchased → archived)
  ├── created_by, last_modified_by
  ├── list_organisation (grouped_by_category | manual | ungrouped)
  └── ShoppingListItems
        └── product, quantity, estimated_price (from current_price), checked_off

FavouriteItems (per user)
  └── product, typical_quantity
      — powers "build from my regulars" and "repeat last shop"

Receipts
  ├── store, date, total
  ├── shopping_list_id (nullable)
  ├── ReceiptImages (one-to-many — supports multiple photos, long images, or PDFs)
  │     └── image_path, upload_order
  └── ReceiptItems
        ├── product, quantity, total_price
        ├── unit_price (calculated: total_price / quantity)
        ├── is_unplanned (true if not on linked shopping list)
        ├── is_sale (if true, price does NOT update product.current_price
        │            or create a PriceHistory entry)
        └── is_consolidated (true if auto-merged from duplicate receipt lines)

BillTypes (electricity, gas, water, council_tax, broadband, rent, insurance, etc.)

Bills
  ├── bill_type, amount, date, period_start, period_end
  ├── owned_by (user or "shared")
  ├── attachment_path (encrypted storage)
  └── type-specific fields (e.g. kWh, unit rate, standing charge for electricity)

MeterReadings
  ├── bill_type, reading_value, reading_date
  └── submitted_by

SmartMeterConfigs (future — architecture designed now)
  ├── provider (n3rgy, glow_hildebrand)
  ├── credentials (encrypted)
  └── meter_type, polling_interval

Alerts
  ├── type (price_increase, bill_projection, spending_anomaly, contract_renewal)
  ├── message, severity, read/unread
  └── created_at

AlertConfigurations
  └── thresholds and preferences per alert type

ApiKeys
  ├── key_hash, name, linked_user_id (bot user)
  ├── scope (read_only, read_write, full_access)
  ├── created_at, last_used_at
  └── is_active

ExternalPriceSources (future)
  ├── source_type (trolley, open_food_facts)
  ├── enabled, credentials (if needed)
  └── last_synced
```

**Unit pricing example:** You buy 2x "Pack of Burgers" at £5 each. The receipt line is: product = "Pack of Burgers", quantity = 2, total_price = £10, unit_price = £5. The product name *is* the unit description.

**Sale price handling:** When entering a receipt, each line item has a sale/offer toggle. If flagged as sale, the price is recorded on the receipt for accuracy but does not update the product's current_price or create a PriceHistory entry. This prevents temporary deals from polluting price trends, forecasting, and estimated shopping list costs. Analytics can separately track savings from sale purchases.

**Cross-store comparison** works by matching products across stores by name and category. Searching "Pack of Burgers" returns results from all stores with their respective price histories.

---

## Implementation Phases

### Phase 1 — Foundation
- Project scaffolding (React + FastAPI + SQLite)
- Docker Compose setup for Pi deployment (single `docker compose up`)
- `deploy.sh` script (git pull → pytest → docker compose build → docker compose up)
- Database migrations framework (Alembic)
- SQLite configured in WAL mode
- **Automated backups:** scheduled Wednesday and Sunday at midnight, backup location configurable in site settings
- User authentication (JWT with refresh tokens)
- **First-run wizard:**
  1. Create admin account (username, email, password)
  2. Property setup (name, type)
  3. Select currency
  4. Pre-seed stores (checkboxes for common UK supermarkets)
  5. Select applicable bill types (determines which bill pages appear)
  6. Configure backup location
  7. Summary and go to dashboard
- **Site settings page** (admin): currency, backup location, general configuration
- Admin panel: user CRUD, role assignment
- Role-based access control middleware
- Responsive layout with **collapsible sidebar** (hamburger menu on mobile)
- Tailwind CSS for mobile-first styling
- **Health endpoint** (`/health`) — app status, database accessible, disk space, last backup status
- Health status card on admin page
- Rotating log files

### Phase 2 — Shopping Core
- Store management (CRUD, pre-seeded UK supermarkets, user-addable)
- Product database with categories (products belong to stores)
- Category management (pre-defined set, admin can extend via admin panel)
- Shopping list builder:
  - Select a store
  - Add items from that store's product database with quantities
  - Estimated cost calculated from current prices
  - Save as draft, edit later
  - Collaborative: all residents can add/edit items on any list
  - **List organisation:** user chooses grouped by category, manual order, or ungrouped (default: grouped by category)
- **Favourites / regulars:**
  - Mark products as favourites (per user) with typical quantities
  - "Build from my regulars" — pre-populate a new list with favourite items
  - "Repeat last shop" — clone a previous shopping list as a new draft
  - User can alter the generated list before finalising
- Shopping list lifecycle: **draft → active → purchased → archived**
- **Shopping mode** (mobile-optimised):
  - Full-screen view, sidebar hidden
  - Tap to check off items (strikethrough, moves to "Done" section)
  - Large tap targets for one-handed use
  - Running total of estimated cost as items are checked off
  - Screen stays awake while active
- **Offline support** via Service Worker:
  - Shopping list cached locally when opened
  - View and check off items while offline
  - Syncs checked-off state when connection returns
  - No editing or creating lists offline

### Phase 3 — Receipt Entry, Price Tracking & Product Explorer
- **Manual receipt entry form:**
  - Select store and date
  - Add line items with autocomplete from that store's products
  - Enter quantity + total price → system calculates unit price
  - **Sale/offer toggle** per line item — if flagged, price does not update product's current_price or create PriceHistory
  - Option to link to an existing shopping list
- **Receipt image upload + Tesseract OCR processing:**
  - Accepts **multiple photos, a single long image, or a PDF** per receipt
  - Multiple photos: take 2-3 photos covering the full receipt, upload them all
  - OCR processes each uploaded file and combines results into one item list
  - **Overlap detection** — if the same line appears in two photos, deduplicate
  - **Auto-consolidation** — identical product lines (same name + same price) merged with combined quantity (e.g. "BURGERS 4PK × 2 = £10.00, unit price: £5.00")
  - Consolidated items marked with a "×2", "×3" indicator in the review screen
  - User can split consolidated items back apart if they were different products with similar names
  - User reviews/corrects parsed results before confirming
  - Sale items must be flagged manually after OCR (OCR can't reliably detect offers)
- **Shopping list linkage:**
  - Items not on the list flagged as **unplanned purchases**
  - List status set to **purchased**
- **Price tracking:**
  - If receipt price differs from product's current_price AND item is not on sale → new PriceHistory entry with date logged
  - Product's current_price updated to the new regular price
  - Sale prices stored on the receipt item only — no impact on product or history
- New products auto-added to the store's catalogue when first entered via receipt
- **Product Explorer page:**
  - Search/filter products across all stores by name, category, or store
  - Product detail view showing:
    - Price history over time (line chart) — regular prices only
    - Cross-store price comparison (bar chart — same product name across stores)
    - Current price, highest recorded price, lowest recorded price
    - Percentage change over selected time periods (week, month, quarter, year)
    - Last purchased date and frequency
    - Sale purchase history shown separately (how often on sale, average saving)
  - Store-level view: browse all products at a given store with sortable columns (name, price, last price change, category)
  - **Infinite scroll** pagination on all list views

### Phase 4 — Bills & Forecasting
- Bill type configuration (purpose-built pages for each):
  - **Electricity** — amount, kWh usage, unit rate, standing charge, period
  - **Gas** — amount, kWh usage, unit rate, standing charge, period
  - **Water** — amount, usage (if metered), period
  - **Council Tax** — band, monthly amount, annual amount
  - **Broadband** — monthly cost, provider, contract end date
  - **Rent/Mortgage** — monthly amount, type
  - **Insurance** — type (home/contents), premium, renewal date
  - Additional types require new purpose-built pages (future dev)
- Bill entry forms with ownership (shared or specific resident)
- Meter reading entry (manual)
- **Smart meter adapter architecture:**
  - Plugin-based design with handlers for each provider
  - **n3rgy handler** — Consumer Access to Energy Data API
  - **Glow/Hildebrand handler** — IHD and CAD data access
  - Adapter interfaces designed now, implementation as a future phase
- Secure file upload for bill PDFs/attachments (encrypted at rest)
- Historical bill tracking and comparison
- **Forecasting engine:**
  - Seasonal time-series forecasting (Facebook Prophet)
  - Predicts next bill amount based on historical data + seasonal patterns
  - Usage-based prediction where meter data available
  - Projected annual costs
- **Infinite scroll** pagination on bill history views

### Phase 5 — Dashboard & Analytics
- **Household overview:**
  - Total monthly/annual spend (all categories)
  - Bill summaries with next-due dates
  - Shopping spend trends
- **Per-resident breakdown:**
  - Individual vs shared expenses
  - Contribution tracking
- **Shopping analytics:**
  - Price history per item per store (line charts)
  - Percentage price changes over time
  - Projected annual food cost based on weekly spend (with seasonal adjustment)
  - Store comparison (where you get the best price for specific items)
  - Most/least purchased items
  - Unplanned purchase frequency and cost
  - Sale savings tracking (total saved by buying on offer)
- **Bill analytics:**
  - Usage trends (line charts)
  - Cost forecasting with confidence intervals
  - Year-on-year comparison (bar charts)
  - Category breakdown (pie charts)
- **CSV export** on all analytics views
- **In-memory caching** for expensive analytics and forecasting queries, invalidated when new data is entered
- Interactive charts using Recharts

### Phase 6 — Alerts
- Dedicated alerts page with **badge count** in navigation for unread alerts
- **In-app notifications only** (push and email can be layered on in future)
- Alert types:
  - **Price increase** — "Pack of Burgers at Tesco has gone up 15% since last month"
  - **Bill projection** — "Your electricity bill is projected to be 20% higher than last quarter"
  - **Spending anomaly** — "This week's shop was £30 above your 4-week average"
  - **Contract/renewal reminders** — "Broadband contract ends in 30 days"
  - **Budget threshold** — configurable spending limits with warnings
- Mark as read/dismiss
- Alert configuration page (admin can set thresholds)

### Phase 7 — API & External Integration
- Authenticated REST API
- **Authentication: API key linked to a bot user account**
  - API keys generated and managed in admin panel
  - Each key linked to a dedicated bot user (e.g. "Virtual Assistant")
  - Configurable scope: read-only, read-write, or full access
  - Actions logged against the bot user for audit trail
  - Keys can be revoked independently of the user account
- Auto-generated OpenAPI documentation (built into FastAPI)
- All responses in **JSON format**, designed for any consumer
- **Dedicated endpoint handlers:**

  **Shopping handlers:**
  - `GET /api/shopping/summary` → weekly/monthly spend summary
    ```json
    {
      "period": "2026-W44",
      "total_spend": 85.40,
      "currency": "GBP",
      "shop_count": 2,
      "item_count": 34,
      "unplanned_count": 5,
      "sale_savings": 4.20,
      "comparison": {
        "vs_last_week": "+3.2%",
        "vs_4wk_avg": "-1.1%"
      }
    }
    ```
  - `GET /api/products/{id}/price-history` → price history for a product
    ```json
    {
      "product": "Pack of Burgers",
      "store": "Tesco",
      "current_price": 5.00,
      "currency": "GBP",
      "history": [
        { "price": 4.50, "date": "2026-08-15" },
        { "price": 5.00, "date": "2026-10-01" }
      ],
      "change_pct": 11.1
    }
    ```
  - `GET /api/products/search?q={query}` → cross-store product search
  - `GET /api/shopping/lists` → active shopping lists
  - `GET /api/shopping/lists/{id}/items` → items on a specific shopping list
  - `POST /api/shopping/lists/{id}/items` → add item to a shopping list (for virtual assistant "add X to the list" commands)
    ```json
    {
      "product_id": 42,
      "quantity": 1
    }
    ```
  - `POST /api/receipts` → quick receipt entry (for virtual assistant "log a spend" commands)

  **Bills handlers:**
  - `GET /api/bills/summary` → all bills overview
  - `GET /api/bills/{type}/forecast` → forecast for a specific bill type
    ```json
    {
      "bill_type": "electricity",
      "next_predicted": 142.30,
      "currency": "GBP",
      "confidence": { "low": 128.00, "high": 156.60 },
      "due_date": "2026-11-15",
      "vs_last": { "amount": 135.20, "change_pct": "+5.3%" },
      "seasonal_note": "Winter increase expected"
    }
    ```
  - `GET /api/bills/{type}/history` → historical bills
  - `GET /api/meter-readings/{type}` → meter reading history

  **Analytics handlers:**
  - `GET /api/analytics/spending` → spending trends and projections
  - `GET /api/analytics/price-changes` → recent price changes across all products
  - `GET /api/analytics/yearly-projection` → projected annual costs

  **Alerts handlers:**
  - `GET /api/alerts` → current unread alerts
  - `PATCH /api/alerts/{id}/read` → mark alert as read

  **Health handler:**
  - `GET /api/health` → system status (virtual assistant can poll this)

- Rate limiting on all endpoints

### Future Phase — External Price Sources
- **Adapter pattern** for plugging in external price data:
  - **Trolley.co.uk adapter** — UK grocery price comparison data
  - **Open Food Facts adapter** — open product database with barcode cross-referencing
  - **Barcode capture** — during receipt entry, optionally capture EAN barcodes for product matching across external databases
- External prices shown alongside your own receipt data in the Product Explorer, clearly labelled as external
- Designed as plugins: each adapter implements a standard interface, can be enabled/disabled in admin settings

### Future Phase — Smart Meter Integration
- Implement n3rgy and Glow/Hildebrand handlers using the adapter interfaces from Phase 4
- Automatic meter reading polling
- Real-time usage dashboards

### Future Phase — Additional Notification Channels
- Browser push notifications via Service Worker (already in place for offline support)
- Email notifications (requires SMTP configuration)
- Per-alert-type channel configuration

---

## Security

- **HTTPS** via WinACME with domain-attached certificate
- **JWT authentication** with short-lived access tokens + refresh tokens
- **bcrypt** password hashing (minimum 12 rounds)
- **API key authentication** for external integrations (hashed storage, linked to bot user, scoped permissions)
- **File encryption** at rest for bill attachments and receipt images (AES-256)
- **Smart meter credentials** encrypted at rest (future)
- **Input sanitisation** on all endpoints (Pydantic models enforce this in FastAPI)
- **CORS** locked to known origins
- **Rate limiting** on all API endpoints
- **SQL injection prevention** via ORM (SQLAlchemy) — parameterised queries only
- **File upload validation** — type checking, size limits
- **Automated database backups** — Wednesday and Sunday at midnight, backup location configurable in site settings
- **Firewall** — only ports 80/443 exposed
- **Rotating log files** — prevent disk fill from logging

---

## Deployment

**Initial setup:**
```bash
git clone <repo>
docker compose up -d
```

**Updates via `deploy.sh`:**
```bash
#!/bin/bash
git pull
docker compose run --rm backend pytest
docker compose build
docker compose up -d
```

Tests run before rebuild — deployment aborts if tests fail.

---

## Data Retention

All data kept indefinitely. On a 256GB NVMe, storage is not a concern for years. If performance ever becomes an issue, indexing or archiving can be added later.
