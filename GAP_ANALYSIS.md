# Hairsense Phase 1 — Gap Analysis

> **Date:** 2026-02-26
> **Base System:** Sorinio (Laravel 12, MariaDB 11.4)
> **Reference:** `ARCHITECTURE.md` in project root

---

## How to Read This Document

Each requirement is assessed with:

| Column | Meaning |
|--------|---------|
| **Status** | `EXISTS` — feature works as-is or with minor config · `PARTIAL` — some foundation exists but significant pieces are missing · `MISSING` — no meaningful code exists |
| **Approach** | `CONFIGURE` — enable/seed/set settings, no code changes · `EXTEND` — add fields, listeners, or views to existing modules · `BUILD NEW` — create new models, controllers, migrations, and UI from scratch |

---

## 1. POS REQUIREMENTS

| # | Feature | Status | Approach | Evidence / Notes |
|---|---------|--------|----------|------------------|
| 1.1 | **Service menu with pricing** | PARTIAL | EXTEND | `ProductServiceItem.type` supports `'service'` value (`packages/workdo/ProductService/src/Models/ProductServiceItem.php:23`). Demo seeder includes service-type items. **Gap:** No `duration` or `time_minutes` field — salon services need time-based scheduling. Migration has only `sale_price`/`purchase_price` — no service-specific pricing tiers. Need to add duration, service category (hair, nails, skin, etc.), and difficulty-level pricing. |
| 1.2 | **Product sales alongside services** | EXISTS | CONFIGURE | POS already sells `ProductServiceItem` records of both `type = 'product'` and `type = 'service'` from the same cart. `PosController::store()` at `packages/workdo/Pos/src/Http/Controllers/PosController.php:167-237` processes items regardless of type. Minor UI work to visually separate services from products in the POS interface. |
| 1.3 | **Staff-linked sales (stylist per service)** | MISSING | BUILD NEW | Neither `Pos` nor `PosItem` models have `staff_id`/`employee_id` fields. `PosItem` fillable (`packages/workdo/Pos/src/Models/PosItem.php:12-22`): only `pos_id`, `product_id`, `quantity`, `price`, `subtotal`, `tax_ids`, `tax_amount`, `total_amount`, `creator_id`, `created_by`. The `creator_id` tracks who logged the sale, NOT who performed the service. Need new `staff_id` column on `pos_items`, staff selector in POS UI, and reporting by stylist. |
| 1.4 | **Multiple payment methods** | MISSING | BUILD NEW | `pos_payments` migration (`packages/workdo/Pos/src/Database/Migrations/2025_09_30_000003_create_pos_payments_table.php`) has only: `pos_id`, `discount`, `amount`, `discount_amount`. **No `payment_method` column exists.** The frontend (`packages/workdo/Pos/src/Resources/js/Pages/Pos/Create.tsx:182`) has `paymentMethod` state defaulting to `'cash'`, but `PosController::store()` never reads it — the field is silently discarded by the backend. Need new migration adding `payment_method` enum (cash, pos_terminal, bank_transfer, mobile_money) and backend support. |
| 1.5 | **Split payments** | MISSING | BUILD NEW | `Pos.payment()` is `HasOne(PosPayment)` (`packages/workdo/Pos/src/Models/Pos.php:46-49`), allowing only a single payment record per sale. Need to: (a) change to `HasMany`, (b) create `pos_payment_lines` table with `payment_method` + `amount` per line, (c) build split-payment UI, (d) validate total across lines equals sale amount. |
| 1.6 | **Discounts & promotions engine** | PARTIAL | EXTEND | A `coupons` table exists (`app/Models/Coupon.php`) with fields: `code`, `discount`, `type` (percentage/flat/fixed), `minimum_spend`, `maximum_spend`, `limit_per_user`, `expiry_date`. However, coupons are **only integrated with plan purchases** (`app/Http/Controllers/PlanController.php`), NOT with POS. POS has only a manual flat discount field on `PosPayment.discount`. Need to: (a) integrate coupon engine with POS, (b) add salon-specific promotions (happy hour, loyalty, bundle deals, first-visit discount), (c) add automatic discount rules. |
| 1.7 | **Daily sales close / Z-reports** | MISSING | BUILD NEW | No z-report, daily close, or end-of-shift functionality exists. `PosReportController` (`packages/workdo/Pos/src/Http/Controllers/PosReportController.php`) only provides ad-hoc sales/product/customer reports with date filtering. No concept of "closing" a register, no cash drawer tracking, no shift-based settlement. Need: `pos_shifts` table, open/close register workflow, Z-report generation with expected vs actual cash, daily settlement records. |
| 1.8 | **Receipt printing / digital receipts** | PARTIAL | EXTEND | Print route exists: `GET /pos/orders/{sale}/print` renders Inertia page (`PosController::print()` at line 305). Frontend has `PrintReceipt.tsx`, `DownloadReceipt.tsx`, and `ReceiptModal.tsx` components. **Gaps:** (a) browser print only — no PDF generation or thermal printer support, (b) no email/SMS receipt — `CreatePos` event listeners are empty (`packages/workdo/Pos/src/Providers/EventServiceProvider.php` has no listeners), (c) no receipt template customization per company. Need PDF generation (e.g., DomPDF), email dispatch listener on `CreatePos`, and optional WhatsApp/SMS via Twilio (already in composer.json). |

**POS Summary:** 1 EXISTS, 3 PARTIAL, 4 MISSING

---

## 2. FINANCE & ACCOUNTING REQUIREMENTS

| # | Feature | Status | Approach | Evidence / Notes |
|---|---------|--------|----------|------------------|
| 2.1 | **Chart of Accounts customized for salon + retail** | PARTIAL | CONFIGURE | Full CoA exists: `ChartOfAccount` model (`packages/workdo/Account/src/Models/ChartOfAccount.php`) with `account_code`, `account_name`, `level`, `normal_balance`, `parent_account_id`. Seeded via `DemoChartOfAccountSeeder.php`. **Gap:** Default CoA is generic business. Need to seed salon-specific accounts: service revenue (by category), product revenue, consumables expense, stylist commissions payable, tip income, etc. This is mostly a seeder/configuration task. |
| 2.2 | **Daily branch sales capture (services + products separately)** | PARTIAL | EXTEND | POS reports (`PosReportController`) group by product and by warehouse (branch). `ProductServiceItem.type` distinguishes `'product'` from `'service'`. **Gap:** Reports do NOT filter or subtotal by product type. The `type` field exists but POS report queries don't use it. Need to add service-vs-product breakdown in daily sales reports and dashboard widgets. |
| 2.3 | **Expense tracking by category** | EXISTS | CONFIGURE | Full implementation: `ExpenseCategories` model with `category_name`, `category_code`, `gl_account_id` (`packages/workdo/Account/src/Models/ExpenseCategories.php`). Expenses link to categories via FK. `ExpenseController` supports category filtering. Demo categories are seeded. Just need to seed Hairsense-specific categories: power/generator, diesel, water, consumables, staff welfare, branch maintenance, etc. |
| 2.4 | **Petty cash management per branch with approval limits** | PARTIAL | BUILD NEW | "Petty Cash" exists as a Chart of Accounts entry (account code 1005 in `AccountUtility.php:92`). Expenses can be recorded against bank accounts. **Gaps:** (a) no per-branch petty cash fund concept, (b) no petty cash approval limits (e.g., branch manager can approve up to ₦50K, anything above needs HQ), (c) no petty cash request/replenishment workflow. Need: `petty_cash_funds` table (per branch, with balance and limit), `petty_cash_requests` with multi-level approval, replenishment tracking. |
| 2.5 | **Bank reconciliation** | PARTIAL | EXTEND | Basic reconciliation exists: `BankTransactionController::markReconciled()` (`packages/workdo/Account/src/Http/Controllers/BankTransactionController.php:51-70`) toggles individual transactions. `bank_transactions` table has `reconciliation_status` (unreconciled/reconciled) and `transaction_status` (pending/cleared/cancelled). **Gaps:** (a) manual one-by-one marking only, (b) no bank statement upload/import, (c) no auto-matching algorithm, (d) no reconciliation summary/variance report, (e) no reconciliation session concept. Need statement import (CSV/OFX), auto-match engine, and reconciliation worksheet UI. |
| 2.6 | **POS terminal settlement reconciliation** | MISSING | BUILD NEW | No terminal concept exists. POS sales flow to bank transactions via `CreatePosListener` (`packages/workdo/Account/src/Listeners/CreatePosListener.php`) but with no terminal ID. Bank transactions have no `terminal_id` or `settlement_batch_id`. Need: `pos_terminals` table (per branch), terminal ID on POS sales, settlement batch processing, terminal-to-bank-deposit matching. |
| 2.7 | **VAT & tax reporting (Nigerian tax rules)** | PARTIAL | EXTEND | Generic tax support exists: `ProductServiceTax` with `tax_name` and `rate`. Tax summary report in `ReportsController::taxSummary()` aggregates tax collected (sales) vs tax paid (purchases). CoA has "Tax Receivable (VAT/GST Input)" and "VAT Payable (Sales Tax Output)" accounts. **Gaps:** (a) no Nigerian-specific VAT rate (7.5%), (b) no WHT (Withholding Tax) calculation, (c) no FIRS filing format, (d) no tax period management. Need: Nigerian tax configuration (VAT 7.5%, WHT rates by category), FIRS-compliant report generation, and tax period close workflow. |
| 2.8 | **Profit & loss per branch** | MISSING | BUILD NEW | P&L report (`packages/workdo/DoubleEntry/src/Services/ProfitLossService.php`) filters only by `created_by` (company level) and date range. No `warehouse_id` or `branch_id` parameter accepted. Same limitation in `ProfitLossController` — only `from_date` and `to_date` parameters. Journal entries DO have a linkable `reference_type`/`reference_id` but no direct branch tagging. Need: `branch_id`/`cost_center_id` on journal entry items, branch parameter on all financial reports. |
| 2.9 | **Consolidated group financials** | MISSING | BUILD NEW | All accounting is scoped to single company via `created_by = creatorId()`. No multi-entity concept, no inter-company elimination, no group-level reporting. Each "company" user is an isolated tenant. Need: group/parent company concept, inter-company transaction handling, consolidation engine with elimination entries. |
| 2.10 | **Budget creation and variance analysis** | EXISTS | CONFIGURE | Full BudgetPlanner module: `Budget`, `BudgetAllocation`, `BudgetPeriod`, `BudgetMonitoring` models (`packages/workdo/BudgetPlanner/src/`). Variance tracking with `variance_amount` and `variance_percentage`. `BudgetService::calculateActualSpending()` pulls actual from journal entries. Budget approval workflow (draft → approved → active → closed). Just need to activate the module and create Hairsense budget structure. |
| 2.11 | **Multi-level approval workflows for expenditure** | PARTIAL | BUILD NEW | Single-level approval exists: expenses have `status` (draft → approved → posted) with `approved_by` field (`packages/workdo/Account/src/Database/Migrations/2025_11_11_092118_create_expenses_table.php:22-24`). Same pattern for revenues, budgets, debit/credit notes. **Gap:** Only one approval level. No approval chains, no amount-based routing (e.g., <₦100K → branch manager, <₦500K → area manager, >₦500K → director). Need: `approval_workflows` table defining chains by document type and amount threshold, `approval_steps` tracking each step. |

**Finance Summary:** 2 EXISTS, 5 PARTIAL, 4 MISSING

---

## 3. INVENTORY REQUIREMENTS

| # | Feature | Status | Approach | Evidence / Notes |
|---|---------|--------|----------|------------------|
| 3.1 | **Dual catalog: salon-use consumables vs retail products** | PARTIAL | EXTEND | `ProductServiceItem` has `type` (product/service) and `category_id` → `ProductServiceCategory`. Categories have `category_name` and can be user-defined. **Gap:** No `usage_type` field to distinguish "internal salon use" (e.g., shampoo used during service) vs "retail for sale" (same shampoo sold to customer). A product could be both. Need: `usage_type` enum (salon_consumable, retail, both) on product items, with separate stock deduction logic for services-consumed vs sold-to-customer. |
| 3.2 | **Stock levels tracked per branch** | EXISTS | CONFIGURE | Multi-warehouse stock tracking works: `WarehouseStock` (`packages/workdo/ProductService/src/Models/WarehouseStock.php`) has `product_id`, `warehouse_id`, `quantity`. One record per product-warehouse combo. Warehouses map to branches. Stock is modified by sales, purchases, transfers, and returns. Just need to create a warehouse per Hairsense branch. |
| 3.3 | **Batch & expiry tracking** | MISSING | BUILD NEW | No `batch_number`, `expiry_date`, `lot_number`, or `serial_number` fields on any product or stock table. `WarehouseStock` only tracks aggregate `quantity`. Search across entire `packages/workdo/ProductService/` confirms zero matches. Need: `product_batches` table (batch_number, expiry_date, manufacture_date, quantity, warehouse_id, product_id), modify stock operations to be batch-aware, expiry alert notification system, FEFO (First Expiry, First Out) logic for stock deduction. |
| 3.4 | **Reorder level alerts** | MISSING | BUILD NEW | No `reorder_level`, `minimum_stock`, or `safety_stock` field on `ProductServiceItem` or `WarehouseStock`. No alert/notification system for low stock. Search confirms zero matches in ProductService module. Need: `reorder_level` and `reorder_quantity` fields on product items (or per-warehouse), scheduled job to check levels, notification dispatch (email + in-app) when stock falls below threshold. |
| 3.5 | **Supplier management with price history** | PARTIAL | EXTEND | Vendors exist in Account module: `Vendor` model (`packages/workdo/Account/src/Models/Vendor.php`) with company details, credit limit, etc. `PurchaseInvoice` links to vendor. **Gap:** No `vendor_product_prices` or `price_history` table. Purchase invoice items record the price at time of purchase, but there's no dedicated price comparison or historical price tracking per vendor per product. Need: `supplier_price_lists` table (vendor_id, product_id, unit_price, effective_date), price history view, best-price comparison when creating POs. |
| 3.6 | **Purchase orders & goods receipt notes** | PARTIAL | EXTEND | `PurchaseInvoice` model (`app/Models/PurchaseInvoice.php`) handles purchasing with status flow (draft → posted → partial → paid). Items, returns, and vendor payments exist. **Gaps:** (a) no separate PO model — system is invoice-first, (b) no Goods Receipt Note (GRN) — stock is updated on invoice posting, not on physical receipt, (c) no three-way matching (PO → GRN → Invoice). Need: `purchase_orders` table with approval workflow, `goods_receipt_notes` table for physical receipt, quantity matching between PO/GRN/Invoice. |
| 3.7 | **Inter-branch stock transfer with approval** | PARTIAL | EXTEND | Transfer model exists (`app/Models/Transfer.php`) with `from_warehouse`, `to_warehouse`, `product_id`, `quantity`, `date`. `TransferController::store()` immediately deducts from source and adds to destination. **Gap:** No `status` field — transfer is instant with no approval step. No transfer request workflow. Need: `status` enum (requested → approved → in_transit → received → cancelled), `requested_by`, `approved_by` fields, approval notification, two-step receive confirmation. |
| 3.8 | **Stock variance & shrinkage reports** | MISSING | BUILD NEW | No stock take, physical count, variance, or shrinkage functionality. No `stock_movements` audit table — `WarehouseStock` only stores current quantity with no history. Search confirms zero matches for variance/shrinkage/stocktake across the codebase. Need: `stock_takes` table (scheduled counts), `stock_take_items` (expected vs counted), variance calculation, shrinkage reports, `stock_movements` audit log tracking every quantity change with reason codes. |
| 3.9 | **Inventory valuation (FIFO and weighted average)** | MISSING | BUILD NEW | No FIFO, weighted average, or any cost-method logic. `ProductServiceItem` has a single `purchase_price` — no per-batch or per-receipt cost tracking. `WarehouseStock` stores only `quantity` with no cost component. Search confirms zero matches for valuation/FIFO/weighted average in application code. Need: cost tracking per stock receipt (batch-level purchase cost), valuation method setting per product or global, FIFO/weighted average cost calculation engine, inventory valuation report. |

**Inventory Summary:** 1 EXISTS, 4 PARTIAL, 4 MISSING

---

## 4. PAYROLL REQUIREMENTS

| # | Feature | Status | Approach | Evidence / Notes |
|---|---------|--------|----------|------------------|
| 4.1 | **Salary structures by role** | PARTIAL | EXTEND | Employee model has `basic_salary`, `hours_per_day`, `days_per_week`, `rate_per_hour` (`packages/workdo/Hrm/src/Models/Employee.php:14-38`). Allowances and deductions are assigned per employee. **Gap:** No salary structure template per role/designation. Each employee's pay components are set individually via "Set Salary" feature. Need: `salary_structures` table (linked to designation/role) with predefined allowance/deduction components that auto-apply when employee is assigned to a role. |
| 4.2 | **Overtime calculation rules** | PARTIAL | EXTEND | Overtime model exists (`packages/workdo/Hrm/src/Models/Overtime.php`) with `title`, `employee_id`, `total_days`, `hours`, `rate` (flat rate, not multiplier). Overtime is manually created per employee. PayrollController includes overtime in payroll calculation. **Gap:** `rate` is a flat monetary amount, not a multiplier (e.g., 1.5x or 2x base rate). No automatic overtime calculation from attendance clock-in/out times. No configurable overtime rules (weekday vs weekend vs holiday rates). Need: overtime rule definitions (type → multiplier), auto-calculation from attendance data, Nigerian labor law compliance (standard 1.5x weekday, 2x weekend). |
| 4.3 | **Commission on services rendered (per stylist)** | MISSING | BUILD NEW | No commission model or calculation in HRM/Payroll module. `PayrollController::runPayroll()` processes allowances, deductions, overtimes, and loans — but NO commission. Search for "commission" in `packages/workdo/Hrm/` returns zero results. A `SalesAgentCommission` concept exists in the Account module (`packages/workdo/Account/src/Listeners/ApproveSalesAgentCommissionAdjustmentLis.php`) but it's for invoice-based sales agents, not POS/service-linked stylists. Need: `commission_rules` table (service_category → percentage/fixed per stylist), link POS service sales to stylist via `staff_id`, commission calculation in payroll run, commission statement per stylist. |
| 4.4 | **Commission on product sales (per staff)** | MISSING | BUILD NEW | Same as 4.3 — no POS-to-payroll commission link. POS items have no `staff_id`, so there's no data trail to calculate who sold what product. Need: `staff_id` on `pos_items` (shared with 1.3), product commission rules (flat or %), commission aggregation per pay period. |
| 4.5 | **Bonus scheme support** | PARTIAL | CONFIGURE | "Performance Bonus" exists as a seeded `AllowanceType` (`packages/workdo/Hrm/src/Database/Seeders/DemoAllowanceTypeSeeder.php:41-43`). Bonuses can be created as allowances per employee with fixed or percentage amounts. **Gap:** No dedicated bonus scheme engine — bonuses are just manual allowance entries. No target-based triggers (e.g., if stylist hits ₦X revenue, auto-award bonus). No team/branch performance bonuses. For Phase 1, treating bonuses as allowances is workable; a full bonus engine can come later. |
| 4.6 | **Deductions: lateness, absences, loan repayments** | PARTIAL | EXTEND | Deductions model exists (`packages/workdo/Hrm/src/Models/Deduction.php`) with type (fixed/percentage). "Late Coming Fine" is a seeded deduction type (`DemoDeductionTypeSeeder.php:37`). Loans exist with payroll integration (`Loan` model). PayrollEntry tracks: `absent_day_deduction`, `half_day_deduction`, `unpaid_leave_deduction`. **Gap:** Lateness deductions are manual — attendance tracks clock-in time but payroll doesn't auto-calculate late-coming fines. Need: auto-lateness detection from attendance (grace period config), automatic deduction calculation in payroll run. |
| 4.7 | **Payslip generation (PDF)** | EXISTS | EXTEND | Payslip route exists: `GET /hrm/payroll-entries/{id}/print` → `PayrollController::printPayslip()` at line 518. Renders `Hrm/Payrolls/payslip/Payslip.tsx` which uses **html2pdf.js** for client-side PDF generation (`?download=pdf` parameter triggers auto-download, lines 62-91). Full template with earnings/deductions breakdown, company branding. **Minor gaps:** (a) no bulk payslip generation for entire payroll run, (b) no email dispatch of payslips to employees, (c) no server-side PDF for archival. Existing implementation is functional for Phase 1. |
| 4.8 | **Pension & statutory deductions (Nigerian: pension, PAYE, NHF)** | MISSING | BUILD NEW | Employee model has generic `tax_payer_id` field (`packages/workdo/Hrm/src/Models/Employee.php:37`). Deduction types can be named "Pension" or "PAYE" but there's **no calculation logic** for Nigerian statutory deductions. No pension contribution calculation (employee 8% + employer 10% under PRA 2014), no PAYE graduated tax table, no NHF (2.5% of basic), no NHIS. Need: Nigerian tax configuration module with PAYE tax brackets, pension calculation engine (RSA contributions), NHF deduction, statutory deduction auto-calculation in payroll run, filing report generation for PFA/FIRS. |
| 4.9 | **Bank payment file generation** | MISSING | BUILD NEW | No bank file export in payroll. `PayrollEntry` has a `PaySalary` event dispatch (`PayrollController::paySalary()` at line 531-539) but this only marks the entry as "paid". No CSV/bank format export. Need: bank payment file generation supporting Nigerian bank formats (NIBSS, individual bank templates), bulk payment file for all employees in a payroll run, NUBAN account validation. |

**Payroll Summary:** 1 EXISTS, 3 PARTIAL, 5 MISSING

---

## 5. SUMMARY COUNTS

### By Status

| Status | POS | Finance | Inventory | Payroll | **Total** |
|--------|-----|---------|-----------|---------|-----------|
| EXISTS | 1 | 2 | 1 | 1 | **5** |
| PARTIAL | 3 | 5 | 4 | 3 | **15** |
| MISSING | 4 | 4 | 4 | 5 | **17** |
| **Total** | **8** | **11** | **9** | **9** | **37** |

### By Approach

| Approach | POS | Finance | Inventory | Payroll | **Total** |
|----------|-----|---------|-----------|---------|-----------|
| CONFIGURE | 1 | 3 | 1 | 1 | **6** |
| EXTEND | 3 | 3 | 4 | 4 | **14** |
| BUILD NEW | 4 | 5 | 4 | 4 | **17** |
| **Total** | **8** | **11** | **9** | **9** | **37** |

### Effort Distribution

```
CONFIGURE  (6 items) ███░░░░░░░░░░░░░░░░░  16%  — Seeder/settings changes only
EXTEND    (14 items) ████████░░░░░░░░░░░░  38%  — Add to existing modules
BUILD NEW (17 items) █████████░░░░░░░░░░░  46%  — New models + migrations + UI
```

---

## 6. CRITICAL PATH ITEMS

These items have the highest business impact and should be prioritized:

### Must-Have for Salon POS (Day-1 Blockers)

1. **Staff-linked sales** (1.3) — Core salon requirement. Every service must be attributed to a stylist for commission calculation and performance tracking.
2. **Multiple payment methods** (1.4) — Nigerian salons handle cash, POS terminals, and bank transfers daily. Cannot operate without this.
3. **Split payments** (1.5) — Closely tied to 1.4. Customers regularly pay with a mix of cash + transfer.
4. **Service duration field** (1.1) — Needed for appointment scheduling and capacity planning (future phase, but data model should include it now).

### Must-Have for Financial Control

5. **Profit & loss per branch** (2.8) — Multi-branch operations need branch-level financial visibility.
6. **Nigerian statutory deductions** (4.8) — Legal compliance. Cannot run payroll without PAYE, pension, NHF.
7. **Commission calculation** (4.3, 4.4) — Primary compensation model for salon staff.
8. **Multi-level approval** (2.11) — Financial control across branches requires tiered approvals.

### Must-Have for Inventory Integrity

9. **Batch & expiry tracking** (3.3) — Salon chemicals have shelf life. Regulatory and safety requirement.
10. **Stock movement audit trail** (3.8) — Current system has no stock history. Cannot investigate discrepancies.

---

## 7. CROSS-CUTTING CONCERNS

These architectural items affect multiple requirements and should be addressed early:

| Concern | Impact | Recommendation |
|---------|--------|----------------|
| **Branch concept** | POS, Finance, Inventory, Payroll all reference "branch" but system only has `warehouses` | Create a `branches` table (or extend `warehouses`) with branch-specific config: approval limits, petty cash fund, POS terminals, manager assignment |
| **Staff ↔ Employee link** | POS needs `staff_id` on items, Payroll needs commission from POS | Ensure `PosItem.staff_id` → `employees.id` → `users.id` chain is established for commission aggregation |
| **Stock audit trail** | Batch tracking, variance reports, valuation all need movement history | Create `stock_movements` table early — all stock-changing operations should write to it |
| **PDF generation** | Payslips, receipts, reports, tax filings all need PDF output | Add DomPDF or Snappy once — reuse across all modules |
| **Nigerian localization** | Tax rules, bank formats, currency, date formats | Create a `NigerianCompliance` service class that encapsulates PAYE brackets, pension rates, NHF rates, VAT rate (7.5%), and NUBAN validation |

---

## 8. FILE REFERENCES

Key files that will be modified or extended during implementation:

| Area | Files |
|------|-------|
| **POS Models** | `packages/workdo/Pos/src/Models/Pos.php`, `PosItem.php`, `PosPayment.php` |
| **POS Controller** | `packages/workdo/Pos/src/Http/Controllers/PosController.php` |
| **POS Migrations** | `packages/workdo/Pos/src/Database/Migrations/2025_09_30_000001_create_pos_table.php` through `000003` |
| **Product Model** | `packages/workdo/ProductService/src/Models/ProductServiceItem.php` |
| **Account Models** | `packages/workdo/Account/src/Models/Expense.php`, `ChartOfAccount.php`, `JournalEntry.php` |
| **Account Reports** | `packages/workdo/Account/src/Http/Controllers/ReportsController.php` |
| **DoubleEntry Reports** | `packages/workdo/DoubleEntry/src/Services/ProfitLossService.php`, `BalanceSheetService.php` |
| **Bank Transactions** | `packages/workdo/Account/src/Http/Controllers/BankTransactionController.php` |
| **Transfer Model** | `app/Models/Transfer.php`, `app/Http/Controllers/TransferController.php` |
| **Employee Model** | `packages/workdo/Hrm/src/Models/Employee.php` |
| **Payroll Controller** | `packages/workdo/Hrm/src/Http/Controllers/PayrollController.php` |
| **Payroll Models** | `packages/workdo/Hrm/src/Models/Payroll.php`, `PayrollEntry.php`, `Overtime.php` |
| **Warehouse Stock** | `packages/workdo/ProductService/src/Models/WarehouseStock.php` |
| **Budget Planner** | `packages/workdo/BudgetPlanner/src/` (all files — existing, needs activation) |
