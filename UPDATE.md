# Hairsense ERP Customisation — Progress Update

**Date:** 27 February 2026
**Branch:** `claude/investigate-erpgo-tenancy-oYotz`
**Base System:** ERPGo SaaS (Laravel 12, PHP 8.4, MariaDB 11.4)

---

## Executive Summary

We have completed the investigation, planning, and first two implementation phases of the Hairsense salon ERP customisation. The ERPGo SaaS platform has been thoroughly analysed, a gap analysis has been produced, and the foundational database structures and POS backend for salon service sales are now in place — all with passing automated tests.

---

## What Has Been Delivered

### Phase 0 — System Investigation & Documentation

**Status: Complete**

- Produced `ARCHITECTURE.md` — a comprehensive reference document covering ERPGo's multi-tenancy model, database strategy, user hierarchy, module system, and key helpers.
- Key finding: ERPGo uses **custom application-level tenancy** (every table has a `created_by` column) rather than database-per-tenant. All Hairsense customisations follow this pattern.

### Phase 0.5 — Gap Analysis

**Status: Complete**

- Produced `GAP_ANALYSIS.md` assessing **37 features** across four domains against Hairsense requirements:

| Domain             | Exists | Partial | Missing | Total |
|--------------------|--------|---------|---------|-------|
| POS & Sales        | 1      | 4       | 3       | 8     |
| Finance/Accounting | 2      | 4       | 5       | 11    |
| Inventory          | 1      | 4       | 4       | 9     |
| Payroll/HR         | 1      | 3       | 5       | 9     |
| **Totals**         | **5**  | **15**  | **17**  | **37**|

- Each feature includes a recommended approach: **Configure** (use as-is), **Extend** (add fields/logic), or **Build New**.

### Phase 1A — Database Foundation & Branch Model

**Status: Complete**

Created the foundational data structures for multi-branch salon operations:

| What | Detail |
|------|--------|
| **8 migrations** | Extended `branches` table with address/status/manager fields; linked warehouses to branches; created `service_categories`, `salon_services`, `stylist_profiles`, `product_batches`, `branch_stocks` tables; extended `product_service_items` with salon product types |
| **5 Eloquent models** | `ServiceCategory`, `SalonService`, `StylistProfile`, `ProductBatch`, `BranchStock` — each with tenant scoping, relationships, and helper methods |
| **1 seeder** | `HairsenseSeeder` populating 3 Nigerian branches (Lekki, Victoria Island, Ikeja), 3 service categories, 10 salon services, 5 stylists, and 15 products with stock and batch records |

Key design decisions:
- Extended the existing HRM `branches` table rather than creating a duplicate, preserving ERPGo compatibility.
- Branch-specific pricing stored as JSON on `salon_services` for flexibility.
- Commission support at both service level (default) and stylist level (override).

### Phase 1B — POS Backend for Salon Service Sales

**Status: Complete (10 tests passing, 55 assertions)**

Extended the existing POS module to support salon-specific sales:

| Component | Files | What It Does |
|-----------|-------|-------------|
| **3 migrations** | Extend `pos`, extend `pos_items`, create `sale_payments` | Add branch/stylist/sale-type tracking to sales; add service/commission fields to line items; enable split payments |
| **SalePayment model** | `app/Models/SalePayment.php` | Tracks individual payment legs (cash, POS terminal, bank transfer) per sale |
| **CommissionCalculator** | `app/Services/CommissionCalculator.php` | Calculates commissions with priority: stylist override rate > service default rate. Supports both percentage and fixed amounts |
| **SalonSaleService** | `app/Services/SalonSaleService.php` | Core sale creation logic wrapped in a DB transaction — creates Pos record, line items with tax & commission, validates and records split payments, decrements branch stock, fires existing accounting events |
| **SalonPosController** | `app/Http/Controllers/Api/SalonPosController.php` | Three API endpoints with validation and permission checks |
| **Feature tests** | `tests/Feature/SalonSaleTest.php` | 10 test methods covering service sales, product sales, mixed sales, split payments, commission calculation, stock management, and all API endpoints |

**API Endpoints delivered:**

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/pos/salon-sale` | Create a salon sale (services, products, or mixed) with split payments |
| GET | `/api/pos/salon-sale/{id}` | Retrieve full sale details with items, payments, and stylist info |
| GET | `/api/pos/daily-summary/{branch_id}/{date}` | Daily branch report: revenue breakdown, commission totals, payment method split |

**Backward compatibility preserved:**
- Existing `PosPayment` record still created for every sale so the existing stock-decrement and accounting journal-entry event listeners continue to work untouched.
- The `CreatePos` event is dispatched as before, maintaining integration with ProductService and Account modules.

---

## File Inventory

| Category | Count | Location |
|----------|-------|----------|
| Migrations | 11 | `database/migrations/2026_02_27_*` |
| Models | 6 | `app/Models/` |
| Services | 2 | `app/Services/` |
| Controllers | 1 | `app/Http/Controllers/Api/SalonPosController.php` |
| Seeders | 1 | `database/seeders/HairsenseSeeder.php` |
| Tests | 1 | `tests/Feature/SalonSaleTest.php` |
| Route changes | 1 | `routes/api.php` (3 new routes added) |
| Documentation | 2 | `ARCHITECTURE.md`, `GAP_ANALYSIS.md` |
| **Total** | **25** | |

---

## Technical Notes

1. **No existing ERPGo migrations or core files were modified** in Phase 1A. Phase 1B required minimal, additive changes to two package models (`Pos.php`, `PosItem.php`) — only adding new fillable fields and relationships, nothing removed or altered.

2. **Testing uses SQLite in-memory** (`phpunit.xml` configured) since the development environment does not have a dedicated test MySQL instance. All migrations are compatible with both SQLite and MariaDB.

3. **Known issue:** The existing `app/Models/warehouse.php` file has a lowercase filename that breaks PSR-4 autoloading. We work around this where needed (using `DB::table()` in the seeder, excluding the `warehouse` eager-load in the API). This is a pre-existing issue in the ERPGo codebase.

---

## What Comes Next

Based on the gap analysis, the recommended next phases are:

| Phase | Scope | Key Deliverables |
|-------|-------|-----------------|
| **1C** | POS Frontend | React/Inertia salon POS interface — service selection, stylist assignment, split payment UI |
| **2A** | Appointments & Scheduling | Booking system with stylist availability, calendar views, walk-in vs appointment tracking |
| **2B** | Inventory Enhancements | Inter-branch stock transfers, batch/expiry management UI, low-stock alerts |
| **3A** | Payroll & Commission | Commission reports per stylist, salary + commission payslip generation, branch-level P&L |
| **3B** | Finance Extensions | Petty cash per branch, daily cash reconciliation, multi-branch consolidated reporting |

---

*This document will be updated as each phase is completed.*
