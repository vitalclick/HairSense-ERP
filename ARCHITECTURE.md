# ERPGo SaaS - Architecture Documentation

> **Investigation Date:** 2026-02-26
> **Framework:** Laravel 12.0 (PHP 8.2+)
> **Frontend:** Inertia.js + Vue/React
> **Database:** MariaDB 11.4.10 (single database: `ondoatco_erpsaas`)
> **SQL Dump:** `ondoatco_erpsaas.sql` (1.4 MB, 13,697 lines, ~243 tables)

---

## 1. TENANCY MODEL

### 1.1 Tenancy Package

**No third-party tenancy package is used.** The application implements **custom application-level multi-tenancy** using user-based row-level scoping.

- **No reference to:** `stancl/tenancy`, `spatie/laravel-multitenancy`, `hyn/multi-tenant`
- **Evidence:** `composer.json` (root) contains no tenancy packages

### 1.2 Tenant Creation & Management

Tenants are represented by **User records with `type = 'company'`**. The `User` model serves double duty as both authentication entity and tenant root.

**User types (tenant hierarchy):**
```
superadmin  → System-wide administrator (single instance)
company     → Tenant root / workspace owner
staff       → Employee under a company
client      → Customer under a company
vendor      → Supplier under a company
doctor      → Healthcare module user
student     → Education module user
parent      → Education module user
```

**Registration flow** (`app/Http/Controllers/Auth/RegisteredUserController.php`):
1. New user registers → created with `type = 'company'`
2. `User::CompanySetting($user->id)` initializes company settings
3. `User::MakeRole($user->id)` creates default roles (staff, client, vendor) scoped to that company
4. `assignPlan()` helper activates modules based on chosen plan

**No artisan commands for tenant management** - tenants are created via web UI only.

### 1.3 Database Strategy

**Single shared database with row-level filtering via `created_by` column.**

- **File:** `config/database.php` - single connection, no dynamic tenant databases
- **Every business table** includes `created_by` (bigint FK → users.id) for tenant isolation
- **Every business table** includes `creator_id` (bigint FK → users.id) for ownership tracking
- **No global query scopes** - filtering is manual in every controller query

**Key helper functions** (`app/Helpers/Helper.php`):
```php
// Returns the company/workspace ID for the current user
function creatorId() {
    if (Auth::user()->type == 'superadmin' || Auth::user()->type == 'company') {
        return Auth::user()->id;    // This user IS the tenant
    } else {
        return Auth::user()->created_by;  // User belongs to this tenant
    }
}

// Returns the company user object
function creatorUser() { ... }
```

**Controller query pattern** (found in every controller):
```php
$records = Model::where('created_by', creatorId())->paginate();
```

### 1.4 Tenant Middleware

**File:** `bootstrap/app.php`

Middleware stack (no `Kernel.php` - Laravel 12 style):
```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        App\Http\Middleware\CheckInstallation::class,      // Installation check
        App\Http\Middleware\HandleInertiaRequests::class,   // Inertia shared data
        App\Http\Middleware\DemoModeMiddleware::class,      // Demo mode restrictions
        Illuminate\Http\Middleware\AddLinkHeadersForPreloadedAssets::class,
        App\Http\Middleware\UpdateUserActiveStatus::class,  // User activity tracking
    ]);
    $middleware->alias([
        'PlanModuleCheck' => App\Http\Middleware\PlanModuleCheck::class,
        'api.json' => App\Http\Middleware\ApiForceJson::class,
    ]);
})
```

**`PlanModuleCheck` middleware** (`app/Http/Middleware/PlanModuleCheck.php`):
- Checks if company plan is expired → redirects to plans page
- For sub-users: checks if creator's plan is expired → logs out
- If `$moduleName` parameter provided: checks if module is active for user's plan
- Superadmin bypasses all checks

### 1.5 Tenant Context Resolution

**Method: Authentication session only.** No subdomain, no path prefix, no header-based resolution.

- User logs in → `Auth::user()` available
- `creatorId()` determines which company's data to query
- All routes use standard paths (no `/tenant-slug/` prefix)
- Impersonation supported via session storage (`app/Http/Controllers/UserController.php`)

### 1.6 Flags / Concerns

- **No global query scopes:** Every controller manually adds `where('created_by', creatorId())`. This is error-prone - a missing filter exposes cross-tenant data.
- **No middleware-level tenant isolation:** Tenant context is resolved per-query, not per-request.
- **Settings are tenant-scoped** via `settings` table with `created_by` column.

---

## 2. DATABASE SCHEMA

### 2.1 Core Migration Files

**Location:** `database/migrations/`

| Migration | Table | Purpose |
|-----------|-------|---------|
| `0001_01_01_000000_create_users_table.php` | `users`, `password_reset_tokens`, `sessions` | Core auth |
| `0001_01_01_000001_create_cache_table.php` | `cache`, `cache_locks` | Cache storage |
| `0001_01_01_000002_create_jobs_table.php` | `jobs`, `job_batches`, `failed_jobs` | Queue system |
| `0001_01_01_000003_create_personal_access_tokens_table.php` | `personal_access_tokens` | Sanctum API tokens |
| `2025_08_12_105132_create_permission_tables.php` | `permissions`, `roles`, `model_has_*` | Spatie RBAC |
| `2025_08_12_105136_create_warehouses_table.php` | `warehouses` | Warehouse locations |
| `2025_08_18_104001_create_settings_table.php` | `settings` | Key-value settings |
| `2025_08_25_032702_create_media_directories_table.php` | `media_directories` | Media organization |
| `2025_08_25_032809_create_media_table.php` | `media` | File/media storage |
| `2025_08_28_083657_create_add_ons_table.php` | `add_ons` | Module registry |
| `2025_08_28_083708_create_user_active_modules_table.php` | `user_active_modules` | Per-user module activation |
| `2025_09_01_000001_create_plans_table.php` | `plans` | Subscription plans |
| `2025_09_02_043140_create_email_templates_table.php` | `email_templates` | Email templates |
| `2025_09_03_055622_create_orders_table.php` | `orders` | Plan purchase orders |
| `2025_09_03_055623_create_coupons_table.php` | `coupons` | Discount coupons |
| `2025_09_09_072036_create_login_histories_table.php` | `login_histories` | Login audit trail |
| `2025_09_13_000002_create_transfers_table.php` | `transfers` | Stock transfers |
| `2025_09_16_112923_create_ch_messages_table.php` | `ch_messages` | Chat messages |
| `2025_09_26_102328_create_purchase_invoices_table.php` | `purchase_invoices` | Purchase invoicing |
| `2025_09_26_102340_create_sales_invoices_table.php` | `sales_invoices` | Sales invoicing |
| `2025_11_10_120000_create_sales_proposals_table.php` | `sales_proposals` | Sales proposals |

**Package migrations** are loaded via each module's ServiceProvider from `packages/workdo/{Module}/src/Database/Migrations/`.

### 2.2 Core Models & Relationships

**Location:** `app/Models/`

#### User (`app/Models/User.php`)
- **Table:** `users`
- **Traits:** `HasRoles` (Spatie), `HasFactory`, `Notifiable`, `HasApiTokens` (Sanctum)
- **Key fields:** `type`, `created_by`, `creator_id`, `active_plan`, `plan_expire_date`, `total_user`, `storage_limit`, `is_disable`, `is_enable_login`, `slug`
- **Relationships:**
  - `createdBy()` → BelongsTo User (via `created_by`)
- **Scopes:**
  - `scopeEmp()` → Filters to employee-type users only

#### Plan (`app/Models/Plan.php`)
- **Table:** `plans`
- **Key fields:** `name`, `modules` (JSON array), `number_of_users`, `storage_limit`, `package_price_monthly`, `package_price_yearly`, `free_plan`, `trial`, `trial_days`
- **Relationships:**
  - `creator()` → BelongsTo User
  - `orders()` → HasMany Order
- **Scopes:** `scopeActive()`

#### SalesInvoice (`app/Models/SalesInvoice.php`)
- **Table:** `sales_invoices`
- **Key fields:** `invoice_number` (auto: SI-YYYY-MM-nnn), `customer_id`, `warehouse_id`, `status` (draft/posted/partial/paid/overdue), financial amounts
- **Relationships:**
  - `items()` → HasMany SalesInvoiceItem
  - `customer()` → BelongsTo User
  - `warehouse()` → BelongsTo Warehouse
  - `customerDetails()` → BelongsTo Customer (Account module)
  - `paymentAllocations()` → HasMany CustomerPaymentAllocation
  - `salesReturns()` → HasMany SalesInvoiceReturn

#### PurchaseInvoice (`app/Models/PurchaseInvoice.php`)
- **Table:** `purchase_invoices`
- **Key fields:** `invoice_number` (auto: PI-YYYY-MM-nnn), `vendor_id`, `warehouse_id`, `status`, financial amounts
- **Relationships:**
  - `items()` → HasMany PurchaseInvoiceItem
  - `vendor()` → BelongsTo User
  - `warehouse()` → BelongsTo Warehouse
  - `vendorDetails()` → BelongsTo Vendor (Account module)
  - `paymentAllocations()` → HasMany VendorPaymentAllocation
  - `purchaseReturns()` → HasMany PurchaseReturn

#### Warehouse (`app/Models/warehouse.php`)
- **Table:** `warehouses`
- **Key fields:** `name`, `address`, `city`, `zip_code`, `phone`, `email`, `is_active`

#### Setting (`app/Models/Setting.php`)
- **Table:** `settings`
- **Key fields:** `key`, `value`, `is_public`, `created_by`
- **Pattern:** Key-value store scoped per company

### 2.3 Tenant-Scoped vs Global Tables

**Tenant-scoped (have `created_by` column):**
All business data tables including: `users`, `settings`, `warehouses`, `transfers`, `sales_invoices`, `purchase_invoices`, `sales_proposals`, `coupons`, `media`, `media_directories`, `helpdesk_tickets`, `login_histories`, and all module-specific tables.

**Global tables (no tenant scoping):**
- `add_ons` - Module registry (system-wide)
- `permissions` - Permission definitions (shared)
- `cache`, `cache_locks` - Cache storage
- `jobs`, `job_batches`, `failed_jobs` - Queue
- `sessions` - Session storage
- `password_reset_tokens` - Password resets
- `personal_access_tokens` - API tokens
- `migrations` - Migration tracking

**Note:** `roles` table has a custom `created_by` column added by the app (not standard Spatie) to scope roles per company.

### 2.4 SQL Dump Analysis (`ondoatco_erpsaas.sql`)

The dump contains **~243 tables** organized across these functional areas:
- **Core System:** 19 tables (users, settings, permissions, sessions, etc.)
- **Accounting & Finance:** 44 tables (chart_of_accounts, journal_entries, bank_accounts, etc.)
- **Sales Management:** 36 tables (sales_invoices, pos, credit_notes, etc.)
- **Purchase Management:** 13 tables
- **Product/Service:** 8 tables
- **HR/Payroll:** 54 tables (employees, payrolls, attendances, leaves, etc.)
- **CRM/Lead:** 17 tables (leads, deals, pipelines, etc.)
- **Project Management:** 11 tables
- **Recruitment:** 11 tables
- **Performance:** 11 tables
- **Support/Tickets:** 11 tables
- **Contracts:** 9 tables
- **Training:** 8 tables
- **Communication:** 3 tables (chat)
- **Configuration:** 11 tables (add_ons, plans, etc.)

---

## 3. MODULE SYSTEM

### 3.1 Module Organization

**Custom package-based architecture** - NOT using `nwidart/laravel-modules`.

All modules reside in `packages/workdo/{ModuleName}/` and follow a Laravel package structure. They are discovered and registered dynamically at boot time.

**Module directory convention:**
```
packages/workdo/{ModuleName}/
├── composer.json          # PSR-4 autoloading + service provider registration
├── module.json            # Module metadata (name, alias, priority, dependencies)
├── favicon.png            # Module icon
└── src/
    ├── Database/
    │   ├── Migrations/
    │   └── Seeders/
    ├── Events/
    ├── Helpers/
    ├── Http/
    │   ├── Controllers/
    │   └── Requests/
    ├── Listeners/
    ├── Models/
    ├── Providers/
    │   ├── {Module}ServiceProvider.php
    │   └── EventServiceProvider.php
    ├── Routes/
    │   ├── web.php
    │   └── api.php (optional)
    └── Services/
```

### 3.2 Module Registration

**`PackageServiceProvider`** (`app/Providers/PackageServiceProvider.php`):
```php
public function register(): void
{
    $loader = require base_path('vendor/autoload.php');
    $packageDirectories = glob(base_path('packages/workdo/*'), GLOB_ONLYDIR);
    foreach ($packageDirectories as $packageDir) {
        $composerFile = $packageDir . '/composer.json';
        if (file_exists($composerFile)) {
            // 1. Register PSR-4 autoloading
            // 2. Register service providers from extra.laravel.providers
        }
    }
}
```

Each module's **ServiceProvider** loads routes and migrations:
```php
public function boot(): void
{
    $this->loadRoutesFrom(__DIR__.'/../Routes/web.php');
    $this->loadMigrationsFrom(__DIR__.'/../Database/Migrations');
}
```

Module routes use the **PlanModuleCheck** middleware with module name:
```php
Route::middleware(['web', 'auth', 'verified', 'PlanModuleCheck:Hrm'])->group(function () { ... });
```

### 3.3 All Modules (28 packages)

| Module | Alias | Priority | Parent | Path |
|--------|-------|----------|--------|------|
| **ProductService** | Product & Service | 0 | - | `packages/workdo/ProductService/` |
| **Taskly** | Project | 10 | - | `packages/workdo/Taskly/` |
| **Account** | Accounting | 20 | - | `packages/workdo/Account/` |
| **LandingPage** | CMS | 20 | - (admin-only) | `packages/workdo/LandingPage/` |
| **Hrm** | HRM | 30 | - | `packages/workdo/Hrm/` |
| **Lead** | CRM | 40 | - | `packages/workdo/Lead/` |
| **Pos** | POS | 50 | - | `packages/workdo/Pos/` |
| **SupportTicket** | Support Ticket | 130 | - | `packages/workdo/SupportTicket/` |
| **DoubleEntry** | Double Entry | 220 | Account | `packages/workdo/DoubleEntry/` |
| **Goal** | Financial Goal | 230 | Account | `packages/workdo/Goal/` |
| **BudgetPlanner** | Budget Planner | 245 | Account | `packages/workdo/BudgetPlanner/` |
| **Training** | Training | 250 | Hrm | `packages/workdo/Training/` |
| **Performance** | Performance | 260 | Hrm | `packages/workdo/Performance/` |
| **Recruitment** | Recruitment | 270 | - | `packages/workdo/Recruitment/` |
| **FormBuilder** | Form Builder | 280 | - | `packages/workdo/FormBuilder/` |
| **Contract** | Contract | 300 | - | `packages/workdo/Contract/` |
| **Timesheet** | Timesheet | 310 | - | `packages/workdo/Timesheet/` |
| **Quotation** | Quotation | 320 | - | `packages/workdo/Quotation/` |
| **AIAssistant** | AI Assistant | 400 | - | `packages/workdo/AIAssistant/` |
| **Slack** | Slack | 750 | - | `packages/workdo/Slack/` |
| **Telegram** | Telegram | 760 | - | `packages/workdo/Telegram/` |
| **Twilio** | Twilio | 770 | - | `packages/workdo/Twilio/` |
| **Calendar** | Calendar | 870 | - | `packages/workdo/Calendar/` |
| **GoogleCaptcha** | Google Captcha | 880 | - (admin-only) | `packages/workdo/GoogleCaptcha/` |
| **ZoomMeeting** | Zoom Meeting | 900 | - | `packages/workdo/ZoomMeeting/` |
| **Webhook** | Webhook | 930 | - | `packages/workdo/Webhook/` |
| **Stripe** | Stripe | 1500 | - | `packages/workdo/Stripe/` |
| **Paypal** | Paypal | 1510 | - | `packages/workdo/Paypal/` |

### 3.4 Module Activation/Deactivation

**Two-tier activation system:**

1. **Global (system-wide):** `add_ons` table → `is_enable` flag
   - Managed by superadmin via `ModuleController`
   - Managed by `Module` class (`app/Classes/Module.php`): `enable()` / `disable()`

2. **Per-user (plan-based):** `user_active_modules` table → links `user_id` to `module` name
   - Set when a plan is assigned via `assignPlan()` helper
   - Modules included in plan are activated for the company user
   - `ActivatedModule($user_id)` helper returns active modules for a user

**Checking flow** (`Module_is_active()` in `app/Helpers/Helper.php`):
```
1. Is module globally enabled in add_ons? → No → return false
2. Is user superadmin? → Yes → return true
3. Is module in user's activated modules? → Yes/No
```

---

## 4. AUTHENTICATION & AUTHORIZATION

### 4.1 Auth System

- **Web Auth:** Laravel Breeze (session-based)
- **API Auth:** Laravel Sanctum (token-based)
- **File:** `config/auth.php` → web guard (session), api guard (sanctum)
- **User Model:** `App\Models\User` implements `MustVerifyEmail`

**Auth controllers:**
- `app/Http/Controllers/Auth/AuthenticatedSessionController.php` - Login (rate-limited, 5 attempts)
- `app/Http/Controllers/Auth/RegisteredUserController.php` - Registration (creates `type='company'`)
- `app/Http/Controllers/Api/AuthApiController.php` - API token auth

**Features:**
- Email verification (optional, controlled by `enableEmailVerification` setting)
- Login history tracking with IP, browser, location
- Account disable via `is_enable_login` field
- Impersonation (superadmin can login as company users)

### 4.2 Roles & Permissions

**Package:** `spatie/laravel-permission` v6.21
- **Config:** `config/permission.php` → teams disabled, wildcard disabled
- **Migration:** `database/migrations/2025_08_12_105132_create_permission_tables.php`
- **Custom columns on `roles`:** `label`, `editable`, `created_by` (for per-company scoping)
- **Custom columns on `permissions`:** `label`, `module`, `add_on`

### 4.3 Defined Roles

| Role | Type | Editable | Created By | Scope |
|------|------|----------|------------|-------|
| **superadmin** | System | No | NULL | System-wide |
| **company** | System | No | superadmin | Company-level |
| **staff** | Per-company | No | company user | Company staff |
| **client** | Per-company | No | company user | Company clients |
| **vendor** | Per-company | No | company user | Company vendors |
| *Custom roles* | Per-company | Yes | company user | Custom |

**Role creation:** `User::MakeRole($userId)` in `app/Models/User.php` (lines 168-300) creates staff/client/vendor roles with default permissions when a company registers.

### 4.4 Permission Categories (~70+ core permissions)

Defined in `database/seeders/PermissionRoleSeeder.php`:

- **Dashboard:** `manage-dashboard`
- **Users:** `manage-users`, `manage-any-users`, `manage-own-users`, `create-users`, `edit-users`, `delete-users`, `change-password-users`, `toggle-status-users`, `impersonate-users`, `view-login-history`
- **Roles:** `manage-roles`, `create-roles`, `edit-roles`, `delete-roles`
- **Warehouses:** `manage-warehouses`, `manage-any-warehouses`, `manage-own-warehouses`, `create-warehouses`, `edit-warehouses`, `delete-warehouses`
- **Transfers:** `manage-transfers`, `manage-any-transfers`, etc.
- **Settings:** `manage-brand-settings`, `manage-company-settings`, `manage-system-settings`, `manage-currency-settings`, `manage-email-settings`, etc. (~24 permissions)
- **Media:** `manage-media`, `manage-any-media`, `manage-own-media`, `create-media`, `download-media`, `delete-media`, etc. (~10 permissions)
- **Sales Invoices:** `manage-sales-invoices`, `manage-any-sales-invoices`, `manage-own-sales-invoices`, `view-sales-invoices`, `create-sales-invoices`, `edit-sales-invoices`, `delete-sales-invoices`, `post-sales-invoices`, `print-sales-invoices`
- **Purchase Invoices:** Similar pattern (~9 permissions)
- **Sales Returns:** `manage-sales-return-invoices`, `approve-sales-returns-invoices`, `complete-sales-returns-invoices`, etc. (~8 permissions)
- **Purchase Returns:** Similar pattern (~8 permissions)
- **Sales Proposals:** `manage-sales-proposals`, `sent-sales-proposals`, `accept-sales-proposals`, `convert-sales-proposals`, etc. (~11 permissions)
- **Plans, Coupons, Orders, Helpdesk, Messenger, Templates:** Each with own permission set

**Each module adds its own permissions** via module-specific seeders in `packages/workdo/{Module}/src/Database/Seeders/PermissionTableSeeder.php`.

### 4.5 Permission Enforcement

**Pattern 1 - Inline permission check (most common):**
```php
public function index() {
    if (Auth::user()->can('manage-sales-invoices')) {
        $query = SalesInvoice::where('created_by', creatorId());
        // ...
    } else {
        return back()->with('error', __('Permission denied'));
    }
}
```

**Pattern 2 - manage-any vs manage-own (data scoping):**
```php
$query->where(function($q) {
    if (Auth::user()->can('manage-any-warehouses')) {
        $q->where('created_by', creatorId());     // All company data
    } elseif (Auth::user()->can('manage-own-warehouses')) {
        $q->where('creator_id', Auth::id());      // Only own data
    } else {
        $q->whereRaw('1 = 0');                    // No data
    }
});
```

**No Blade directives, no Policy classes, no Gate definitions found.** All authorization is inline in controllers.

---

## 5. POS MODULE

### 5.1 Overview

**Location:** `packages/workdo/Pos/`
**Route prefix:** `/pos`
**Middleware:** `PlanModuleCheck:Pos`

### 5.2 Models

**Pos** (`packages/workdo/Pos/src/Models/Pos.php`):
- **Table:** `pos`
- **Fields:** `sale_number` (auto: #POS00001), `customer_id`, `warehouse_id`, `pos_date`, `status`, `creator_id`, `created_by`
- **Relationships:**
  - `customer()` → BelongsTo User
  - `warehouse()` → BelongsTo Warehouse
  - `items()` → HasMany PosItem
  - `payment()` → HasOne PosPayment

**PosItem** (`packages/workdo/Pos/src/Models/PosItem.php`):
- **Fields:** `pos_id`, `product_id`, `quantity`, `price`, `tax_ids` (JSON), `subtotal`, `tax_amount`, `total_amount`

**PosPayment** (`packages/workdo/Pos/src/Models/PosPayment.php`):
- **Fields:** `pos_id`, `discount`, `amount`, `discount_amount`
- **Note:** Only one payment record per POS sale (HasOne)

### 5.3 POS Functionality

**Controllers:**
- `PosController` - Create POS sale, list orders, product search, barcode, print
- `DashboardController` - POS dashboard
- `PosReportController` - Sales, product, customer reports

**Sales flow** (`PosController::store()`):
1. Generate sale number
2. Create `Pos` record with customer + warehouse
3. Loop items: create `PosItem` records with tax calculations
4. Create single `PosPayment` record with discount
5. Dispatch `CreatePos` event (for module integrations)

### 5.4 Payment Methods

**Currently only supports a single, generic payment.** No payment method field exists on `PosPayment`. The model stores discount, amount, and discount_amount only.

**FLAG:** No cash/card/split payment support. No payment method tracking.

### 5.5 POS-to-Accounting Link

The `CreatePos` event is dispatched after POS sale creation. The DoubleEntry module can listen to this event to create journal entries, but **no direct journal entry creation exists in the POS module itself**.

**FLAG:** POS-to-accounting integration relies entirely on the event system. If DoubleEntry listeners aren't configured, POS sales won't appear in the general ledger.

### 5.6 POS Routes

```
GET  /pos                    → Dashboard
GET  /pos/create             → Create POS sale
POST /pos/store              → Store POS sale
GET  /pos/orders             → List orders
GET  /pos/orders/{sale}      → View order
GET  /pos/orders/{sale}/print → Print receipt
GET  /pos/products           → AJAX product search
GET  /pos/pos-number         → Get next POS number
GET  /pos/barcode            → Barcode management
GET  /pos/barcode/{sale}     → Print barcode
GET  /pos/reports/sales      → Sales report
GET  /pos/reports/products   → Product report
GET  /pos/reports/customers  → Customer report
```

---

## 6. ACCOUNTING MODULE

### 6.1 Overview

**Location:** `packages/workdo/Account/`
**Route prefix:** `/account`
**Child modules:** Goal, BudgetPlanner, DoubleEntry

### 6.2 Chart of Accounts

**ChartOfAccount** (`packages/workdo/Account/src/Models/ChartOfAccount.php`):
- **Fields:** `account_code`, `account_name`, `level`, `normal_balance`, `opening_balance`, `current_balance`, `is_active`, `is_system_account`, `description`, `account_type_id`, `parent_account_id`
- **Relationships:**
  - `accountType()` → BelongsTo AccountType
  - `parent_account()` → BelongsTo ChartOfAccount (self-referential)
  - `journalEntryItems()` → HasMany JournalEntryItem

**AccountType** (`packages/workdo/Account/src/Models/AccountType.php`):
- Categorizes accounts (Assets, Liabilities, Equity, Revenue, Expenses)

**AccountCategory** (`packages/workdo/Account/src/Models/AccountCategory.php`):
- Higher-level grouping for account types

**Default chart of accounts** seeded via `packages/workdo/Account/src/Database/Seeders/DemoChartOfAccountSeeder.php`

### 6.3 Journal Entry System

**JournalEntry** (`packages/workdo/Account/src/Models/JournalEntry.php`):
- **Fields:** `journal_number` (auto: JE-YYYY-nnn), `journal_date`, `entry_type`, `reference_type`, `reference_id`, `description`, `total_debit`, `total_credit`, `status` (draft/posted/reversed)
- **Methods:** `isBalanced()`, `isPosted()`, `isDraft()`, `isReversed()`

**JournalEntryItem** (`packages/workdo/Account/src/Models/JournalEntryItem.php`):
- **Fields:** `journal_entry_id`, `account_id`, `description`, `debit_amount`, `credit_amount`
- **Methods:** `isDebit()`, `isCredit()`, `getAmount()`, `getType()`
- **Relationship:** `account()` → BelongsTo ChartOfAccount

**Entry creation:** Journal entries support polymorphic references (`reference_type` + `reference_id`) linking to source transactions (invoices, payments, etc.).

### 6.4 Financial Reports

**Account module reports** (`packages/workdo/Account/src/Http/Controllers/ReportsController.php`):
- Invoice Aging (with print)
- Bill Aging (with print)
- Tax Summary (with print)
- Customer Balance (with print)
- Vendor Balance (with print)
- Customer Detail
- Vendor Detail

**DoubleEntry module reports** (`packages/workdo/DoubleEntry/src/Routes/web.php`):
- **Balance Sheet** (with comparisons, year-end close)
- **Profit & Loss**
- **Trial Balance**
- **General Ledger**
- **Account Statement**
- **Journal Entry Report**
- **Account Balance**
- **Cash Flow**
- **Expense Report**
- **Ledger Summary**

All reports support print/PDF views via dedicated Inertia print components.

### 6.5 Tax Handling

**ProductServiceTax** (`packages/workdo/ProductService/src/Models/ProductServiceTax.php`):
- **Fields:** `tax_name`, `rate` (percentage)
- Products link to taxes via `tax_ids` JSON array
- Tax calculations happen at line-item level in invoices and POS sales

**Per-item tax tracking tables:** `sales_invoice_item_taxes`, `purchase_invoice_item_taxes`, `credit_note_item_taxes`, `debit_note_item_taxes`

### 6.6 Additional Accounting Features

- **Bank Accounts** - Track multiple bank accounts
- **Bank Transactions** - Record and reconcile transactions
- **Bank Transfers** - Inter-account transfers with approval workflow
- **Customer Payments** - Record and allocate to invoices
- **Vendor Payments** - Record and allocate to bills
- **Credit Notes** - Customer credits (linked to sales returns)
- **Debit Notes** - Vendor credits (linked to purchase returns)
- **Revenue/Expense Categories** - Categorization for income/expenses
- **Revenue/Expense Records** - Direct revenue/expense entries with approval + posting workflow
- **Opening Balances** - Initial account balances

---

## 7. INVENTORY MODULE

### 7.1 Product/Item Structure

**ProductServiceItem** (`packages/workdo/ProductService/src/Models/ProductServiceItem.php`):
- **Fields:** `name`, `sku`, `category_id`, `description`, `sale_price`, `purchase_price`, `unit`, `type` (product/service), `is_active`, `tax_ids` (JSON), `image`, `images` (JSON)
- **Relationships:**
  - `category()` → BelongsTo ProductServiceCategory
  - `unitRelation()` → BelongsTo ProductServiceUnit
  - `warehouseStocks()` → HasMany WarehouseStock
  - `getTaxesAttribute()` → Accessor for tax collection

**ProductServiceCategory** - Categories with name and color
**ProductServiceUnit** - Units of measurement (pieces, kg, liters, etc.)
**ProductServiceTax** - Tax definitions with name and rate

### 7.2 Stock Tracking

**WarehouseStock** (`packages/workdo/ProductService/src/Models/WarehouseStock.php`):
- **Fields:** `product_id`, `warehouse_id`, `quantity`
- **Pattern:** One record per product-warehouse combination
- **No stock movement history table** - only current quantity is tracked

**Stock modification points:**
1. **Purchase Invoices** → Increase warehouse stock (via events)
2. **Sales Invoices** → Decrease warehouse stock (via events)
3. **POS Sales** → Decrease warehouse stock (via events)
4. **Transfers** → Decrease source, increase destination
5. **Purchase Returns** → Decrease stock (returned to supplier)
6. **Sales Returns** → Increase stock (returned by customer)

**FLAG:** No `stock_movements` or `stock_history` table exists. There is no audit trail for stock changes beyond the transactions that cause them.

### 7.3 Warehouse/Location Support

**Multi-warehouse:** Yes. Each product can have stock in multiple warehouses via `warehouse_stocks` junction table.

**Warehouse model** (`app/Models/warehouse.php`):
- **Fields:** `name`, `address`, `city`, `zip_code`, `phone`, `email`, `is_active`

### 7.4 Purchase Order Workflow

**Purchase Invoice** (`app/Models/PurchaseInvoice.php`):
```
Draft → Posted → Partial → Paid
                         ↘ Overdue (computed)
```

**Status transitions:**
1. **Draft** - Created, editable
2. **Posted** - Confirmed, stock updated, accounting entries created
3. **Partial** - Partially paid
4. **Paid** - Fully paid
5. **Overdue** - Computed when `due_date < now()` and not paid

**Returns:** `PurchaseReturn` linked to `original_invoice_id` with reasons: defective, wrong_item, damaged, excess_quantity, other

**FLAG:** No dedicated Purchase Order (PO) model exists. The system uses Purchase Invoices directly (invoice-first, not PO-first workflow).

### 7.5 Stock Transfer System

**Transfer** (`app/Models/Transfer.php`):
- **Fields:** `from_warehouse`, `to_warehouse`, `product_id`, `quantity`, `date`
- **Controller logic** (`app/Http/Controllers/TransferController.php`):
  - Store: Decrease source stock, increase destination stock
  - Delete: Reverse the stock movement

---

## 8. PAYROLL MODULE

### 8.1 Overview

Payroll is part of the **HRM module** (`packages/workdo/Hrm/`).

### 8.2 Salary Structure

**Employee** (`packages/workdo/Hrm/src/Models/Employee.php`):
- **Fields:** `employee_id` (auto: EMPYYYYnnnn), `basic_salary`, `hours_per_day`, `days_per_week`, `rate_per_hour`, `date_of_joining`, `employment_type`
- **Relationships:** `user()`, `branch()`, `department()`, `designation()`, `shift()`
- **Hierarchy:** Employee → Department → Branch

**Organizational structure:**
- **Branch** - Physical location (`branch_name`)
- **Department** - Functional unit (`department_name`, linked to branch)
- **Designation** - Job title (`designation_name`, linked to branch + department)
- **Shift** - Work schedule (`shift_name`, `start_time`, `end_time`, `break_start_time`, `break_end_time`, `is_night_shift`)

### 8.3 Deductions & Allowances

**Allowance** (`packages/workdo/Hrm/src/Models/Allowance.php`):
- **Fields:** `employee_id`, `allowance_type_id`, `type` (fixed/percentage), `amount`
- **Relationship:** `allowanceType()` → BelongsTo AllowanceType

**Deduction** (`packages/workdo/Hrm/src/Models/Deduction.php`):
- **Fields:** `employee_id`, `deduction_type_id`, `type` (fixed/percentage), `amount`
- **Relationship:** `deductionType()` → BelongsTo DeductionType

**Loan** (`packages/workdo/Hrm/src/Models/Loan.php`):
- **Fields:** `title`, `employee_id`, `loan_type_id`, `type` (fixed/percentage), `amount`, `start_date`, `end_date`, `reason`

**Overtime** (`packages/workdo/Hrm/src/Models/Overtime.php`):
- **Fields:** `title`, `employee_id`, `total_days`, `hours`, `rate`, `start_date`, `end_date`, `status` (active/expired)

### 8.4 Payslip Generation

**Payroll** (`packages/workdo/Hrm/src/Models/Payroll.php`):
- **Fields:** `title`, `payroll_frequency` (weekly/biweekly/monthly), `pay_period_start`, `pay_period_end`, `pay_date`, `status` (draft/processing/completed/cancelled), `is_payroll_paid` (paid/unpaid), `total_gross_pay`, `total_deductions`, `total_net_pay`, `employee_count`, `bank_account_id`

**PayrollEntry** (`packages/workdo/Hrm/src/Models/PayrollEntry.php`) - Individual payslip:
- **Monetary:** `basic_salary`, `total_allowances`, `total_deductions`, `total_loans`, `total_manual_overtimes`, `gross_pay`, `net_pay`, `per_day_salary`
- **Attendance:** `working_days`, `present_days`, `half_days`, `absent_days`, `paid_leave_days`, `unpaid_leave_days`
- **Overtime:** `manual_overtime_hours`, `attendance_overtime_hours`, `overtime_hours`, `attendance_overtime_amount`
- **Leave deductions:** `unpaid_leave_deduction`, `half_day_deduction`, `absent_day_deduction`
- **Breakdowns (JSON):** `allowances_breakdown`, `deductions_breakdown`, `manual_overtimes_breakdown`, `loans_breakdown`
- **Status:** `status` (paid/unpaid)

**Run Payroll process** (`PayrollController::runPayroll()`):
1. Get working days configuration from company settings
2. Calculate working days in pay period
3. For each employee:
   - Calculate allowances (fixed or percentage of basic)
   - Calculate deductions (fixed or percentage of basic)
   - Calculate manual overtimes
   - Calculate loan deductions
   - Pull attendance data (present/absent/half-day/overtime)
   - Pull leave data (paid/unpaid)
   - Compute: `gross_pay = basic + allowances + overtimes - leave_deductions + attendance_overtime`
   - Compute: `net_pay = gross_pay - deductions - loans`
4. Create PayrollEntry for each employee
5. Update Payroll totals

### 8.5 Pay Period Management

- **Frequencies:** Weekly, Bi-weekly, Monthly
- **Period boundaries:** Explicitly set via `pay_period_start` and `pay_period_end`
- **Pay date:** Separate from period end date
- Working days calculated dynamically from company `working_days` setting

### 8.6 Payroll Routes

```
GET    /hrm/payrolls              → List payrolls
POST   /hrm/payrolls              → Create payroll
GET    /hrm/payrolls/{id}         → View payroll (with entries)
GET    /hrm/payrolls/{id}/edit    → Edit payroll
PUT    /hrm/payrolls/{id}         → Update payroll
DELETE /hrm/payrolls/{id}         → Delete payroll
POST   /hrm/payrolls/{id}/run    → Process payroll (generate payslips)
DELETE /hrm/payroll-entries/{id}  → Delete individual payslip
GET    /hrm/payroll-entries/{id}/print → Print payslip
PATCH  /hrm/payroll-entries/{id}/pay   → Mark payslip as paid
```

### 8.7 Related HRM Features

- **Attendance:** Clock in/out, overtime tracking, shift-based
- **Leave Management:** Leave types (paid/unpaid), applications, approval workflow, balance tracking
- **Set Salary:** Configure employee base salary and components
- **Working Days:** Company-level working day configuration

---

## 9. ADDITIONAL MODULES SUMMARY

### CRM/Lead Module (`packages/workdo/Lead/`)
- Lead and Deal management with pipeline stages
- Activity logs, calls, emails, discussions, files, tasks per lead/deal
- Source and label tracking

### Project Management / Taskly (`packages/workdo/Taskly/`)
- Projects with milestones, tasks, subtasks, bugs
- Task stages and comments
- Project files and activity logs
- Team member assignments

### Recruitment (`packages/workdo/Recruitment/`)
- Job postings, categories, locations, types
- Candidate management with assessments and onboarding
- Interview scheduling with rounds and feedback

### Performance (`packages/workdo/Performance/`)
- Review cycles and employee reviews
- Goal setting and types
- Performance indicators and KPI categories

### Training (`packages/workdo/Training/`)
- Training programs with types and tasks
- Trainer management
- Training feedback collection

### Contract (`packages/workdo/Contract/`)
- Contract management with types
- Attachments, comments, notes, signatures
- Renewal tracking

### Support Ticket (`packages/workdo/SupportTicket/`)
- Ticket system with custom pages and FAQs
- Knowledge base articles
- Quick links

### Budget Planner (`packages/workdo/BudgetPlanner/`)
- Budget definitions with periods and allocations
- Budget monitoring and tracking

### Goal (`packages/workdo/Goal/`)
- Financial goals with categories
- Contributions, milestones, and tracking

### DoubleEntry (`packages/workdo/DoubleEntry/`)
- Balance sheets with comparisons and year-end close
- Profit & Loss statements
- Trial Balance
- General Ledger, Account Statements
- Cash Flow reports

---

## 10. TECHNOLOGY STACK

### Backend Dependencies (`composer.json`)

| Package | Version | Purpose |
|---------|---------|---------|
| `laravel/framework` | ^12.0 | Core framework |
| `laravel/sanctum` | ^4.2 | API authentication |
| `laravel/socialite` | ^5.23 | Social login |
| `laravel/breeze` | ^2.0 (dev) | Auth scaffolding |
| `inertiajs/inertia-laravel` | ^2.0 | SPA bridge |
| `spatie/laravel-permission` | ^6.21 | RBAC |
| `spatie/laravel-medialibrary` | ^11.14 | Media management |
| `spatie/laravel-google-calendar` | ^3.8 | Google Calendar |
| `stripe/stripe-php` | ^17.6 | Stripe payments |
| `srmklive/paypal` | ^3.0 | PayPal payments |
| `pusher/pusher-php-server` | ^7.2 | Realtime events |
| `twilio/sdk` | ^8.8 | SMS notifications |
| `phpoffice/phpspreadsheet` | ^5.1 | Excel export |
| `pragmarx/google2fa-laravel` | ^2.3 | 2FA |
| `salla/zatca` | ^3.0 | Saudi e-invoicing |
| `google/apiclient` | ^2.18 | Google API |
| `microsoft/microsoft-graph` | ^2.49 | Microsoft 365 |
| `webklex/laravel-imap` | ^6.2 | Email IMAP |
| `mailchimp/marketing` | ^3.0 | Mailchimp |
| `tightenco/ziggy` | ^2.0 | Route sharing with JS |

### Frontend
- **Inertia.js** v2.0 with Vue/React components
- **Vite** for asset bundling (`vite.config.js`)
- **Ziggy** for JavaScript route resolution

---

## 11. FLAGS & CONCERNS

### Critical Issues

1. **No global query scopes for tenant isolation.** Every query manually adds `where('created_by', creatorId())`. A single missing filter creates a cross-tenant data leak. Consider implementing a `BelongsToTenant` trait with a global scope.

2. **No stock movement audit trail.** `warehouse_stocks` only tracks current quantity with no history. Any discrepancy is undetectable.

3. **POS has no payment method tracking.** `PosPayment` stores only amount/discount - no cash/card/split payment support.

4. **No dedicated Purchase Order model.** System uses Purchase Invoices directly, skipping the PO → GRN → Invoice workflow common in ERP systems.

### Architecture Concerns

5. **Authorization is purely inline** - no Policies, no Gates, no middleware-level permission checks. Permission logic is duplicated across controllers.

6. **POS-to-accounting link is event-based only.** If event listeners aren't properly configured, POS sales won't generate journal entries.

7. **Module routes always load.** Service providers load routes for all installed packages regardless of whether the module is active. The `PlanModuleCheck` middleware blocks access but routes are still registered.

8. **No database-level foreign key constraints** on `created_by` for many package tables - relying on application-level consistency.

### Documentation Gaps

9. **No API documentation.** API routes exist in some modules but are undocumented.

10. **Event listeners are spread across modules** with no central registry of what events each module listens to.

11. **Module dependencies** (parent_module/child_module in module.json) are informational only - not enforced at activation time.

---

## 12. FILE REFERENCE INDEX

| Category | Key Files |
|----------|-----------|
| **Entry Point** | `bootstrap/app.php` |
| **Routes** | `routes/web.php`, `routes/auth.php`, `routes/api.php` |
| **Middleware** | `app/Http/Middleware/PlanModuleCheck.php`, `app/Http/Middleware/HandleInertiaRequests.php`, `app/Http/Middleware/CheckInstallation.php` |
| **Helpers** | `app/Helpers/Helper.php` (auto-loaded via composer) |
| **Module System** | `app/Classes/Module.php`, `app/Providers/PackageServiceProvider.php` |
| **User Model** | `app/Models/User.php` |
| **Core Models** | `app/Models/SalesInvoice.php`, `app/Models/PurchaseInvoice.php`, `app/Models/Plan.php`, `app/Models/Setting.php`, `app/Models/warehouse.php` |
| **Auth Controllers** | `app/Http/Controllers/Auth/` |
| **Core Seeders** | `database/seeders/PermissionRoleSeeder.php`, `database/seeders/DatabaseSeeder.php` |
| **POS Module** | `packages/workdo/Pos/src/` |
| **Account Module** | `packages/workdo/Account/src/` |
| **DoubleEntry Module** | `packages/workdo/DoubleEntry/src/` |
| **HRM Module** | `packages/workdo/Hrm/src/` |
| **Product Module** | `packages/workdo/ProductService/src/` |
| **SQL Dump** | `ondoatco_erpsaas.sql` |
| **Module Metadata** | `packages/workdo/{Module}/module.json` |
