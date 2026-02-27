# HairSense ERP

HairSense ERP is the centralized enterprise platform used by **HairSense Unisex Salon** to manage operations across multiple locations in **Nigeria, Canada, UK, and USA**.

This repository is based on ERPGo SaaS and has been customized to support HairSense’s multi-location operating model, including salon service sales, branch inventory controls, stylist workflows, and consolidated business reporting.

---

## What This Repository Contains

### Core Platform
- Laravel 12 application with Inertia-based frontend architecture.
- Modular package structure under `packages/workdo/*`.
- Multi-tenant business scoping primarily through `created_by` / `creator_id` patterns.

### HairSense Customization Layers (Phase 1)
- Branch and service domain extensions:
  - `service_categories`
  - `salon_services`
  - `salon_service_branch_prices`
  - `stylist_profiles`
  - `product_batches`
  - `branch_stocks`
- Product model extension for retail vs salon consumable behavior.
- POS backend extension for:
  - service-only sales
  - product-only sales
  - mixed sales
  - stylist attribution
  - split / multi-method payments
  - commission calculation hooks

### Documentation Added
- `ARCHITECTURE.md` – full baseline architecture investigation.
- `GAP_ANALYSIS.md` – Phase 1 requirement gap assessment.
- `update.md` – implementation progress summary.

---

## Repository Structure (High-Level)

```text
app/                         # Core Laravel application code
bootstrap/                   # App bootstrap and middleware wiring
config/                      # Framework and module configuration
database/                    # Migrations, seeders, factories
packages/workdo/             # ERP modules (POS, HRM, Account, etc.)
routes/                      # Web/API/installer/updater routes
tests/                       # Automated tests
ARCHITECTURE.md              # Architecture investigation
GAP_ANALYSIS.md              # Gap analysis against HairSense requirements
update.md                    # Progress update log
```

---

## Getting Started (Development)

### 1) Prerequisites
- PHP `^8.2`
- Composer
- MySQL/MariaDB
- Node.js + npm (for frontend assets)

### 2) Install dependencies

```bash
composer install
npm install
```

### 3) Environment setup

```bash
cp .env.example .env
php artisan key:generate
```

Configure DB credentials in `.env`.

### 4) Run migrations

```bash
php artisan migrate
```

### 5) (Optional) Seed HairSense sample data

```bash
php artisan db:seed --class=HairsenseSeeder
```

### 6) Run locally

```bash
php artisan serve
npm run dev
```

---

## Production Deployment Notes

When deploying updates that include schema changes, run:

```bash
php artisan down
php artisan migrate --force
php artisan up
```

If required for initial sample bootstrap only:

```bash
php artisan db:seed --class=HairsenseSeeder --force
```

> Do **not** run sample seeders on production unless you intentionally want demo/sample records.

---

## Current Implementation Focus

This repository is currently focused on:
- Multi-location salon operations standardization
- POS + service integration
- Staff/stylist operational accountability
- Inventory controls by branch and batch
- Financial visibility and reconciliation readiness

---

## Contributing

1. Create a feature branch from current working branch.
2. Keep customizations additive and avoid editing historical migrations.
3. Preserve tenant-scoping conventions on all new data models.
4. Add tests for new behavior where feasible.
5. Update `update.md` when delivering major milestone changes.

---

## License

This repository is derived from ERPGo SaaS/Laravel ecosystem components and follows their applicable licensing terms for upstream components.
