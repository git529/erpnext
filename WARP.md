# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Overview

**ERPNext** (version 15.82.2) is a 100% open-source ERP system built on the Frappe Framework. It's a comprehensive business management platform covering accounting, inventory, manufacturing, CRM, projects, support, HR, and more.

- **Version**: 15.82.2 (version-15 branch)
- **Framework**: Frappe Framework (Python/JavaScript full-stack framework)
- **License**: GNU General Public License v3
- **Python**: >= 3.10
- **Database**: MariaDB/PostgreSQL

## Development Environment

This is a **Frappe Bench application**. The directory structure is:
```
frappe-bench/
├── apps/
│   ├── frappe/        # Frappe Framework (dependency)
│   └── erpnext/       # This repository (version-15 branch)
├── sites/             # Site-specific data and configurations
├── env/               # Python virtual environment
├── config/            # Bench configuration files
└── logs/              # Application logs
```

### Key Paths
- **Application code**: `erpnext/`
- **Controllers**: `erpnext/controllers/` (base classes for inheritance)
- **Modules**: `erpnext/accounts/`, `erpnext/stock/`, `erpnext/selling/`, etc.
- **Tests**: `erpnext/tests/` and alongside each module/doctype
- **Regional customizations**: `erpnext/regional/`

## Common Commands

All commands should be run from the `frappe-bench` directory (two levels up). Use `cd ../../` to navigate there.

### Development Server

```bash
# Start development server
bench start

# Or from this directory:
cd ../../ && bench start
```

### Building Assets

```bash
# Build assets for production
bench build --app erpnext

# Build in development mode (faster, with source maps)
bench build --app erpnext --dev

# Watch mode for development
bench watch
```

### Testing

```bash
# Run all tests for ERPNext
bench --site [sitename] run-tests --app erpnext

# Run a specific test module
bench --site [sitename] run-tests --app erpnext --module erpnext.accounts.doctype.sales_invoice.test_sales_invoice

# Run specific test class/method
bench --site [sitename] run-tests --app erpnext --module erpnext.stock.doctype.item.test_item --test TestItem.test_item_defaults

# Run tests for a specific DocType
bench --site [sitename] run-tests --app erpnext --doctype "Sales Invoice"

# Run tests with coverage
bench --site [sitename] run-tests --app erpnext --coverage

# Run tests in parallel (faster)
bench --site [sitename] run-parallel-tests --app erpnext
```

### Code Quality

```bash
# Lint Python code (uses ruff)
ruff check erpnext/

# Fix auto-fixable linting issues
ruff check --fix erpnext/

# Format Python code
ruff format erpnext/

# Check import sorting
ruff check --select=I erpnext/

# Run pre-commit hooks manually
pre-commit run --all-files

# Run on specific files
pre-commit run --files erpnext/accounts/doctype/sales_invoice/sales_invoice.py
```

### Database Operations

```bash
# Run database migrations
bench --site [sitename] migrate

# Clear cache
bench --site [sitename] clear-cache

# Reinstall app (⚠️ Destructive - deletes all data)
bench --site [sitename] reinstall --yes

# Python console with Frappe/ERPNext context
bench --site [sitename] console

# Database console
bench --site [sitename] mariadb
# or for PostgreSQL:
bench --site [sitename] postgres
```

### Site Management

```bash
# Create a new site
bench new-site [sitename]

# Install ERPNext on a site
bench --site [sitename] install-app erpnext

# Backup site
bench --site [sitename] backup

# Restore from backup
bench --site [sitename] restore [backup-file]
```

### Translation

```bash
# Extract translatable strings
bench get-translations

# Update POT file
bench update-translations
```

## Architecture Overview

### High-Level Structure

ERPNext follows the Frappe Framework's DocType-centric architecture. The codebase is organized by business domains (modules), each containing:
- **DocTypes**: Database schema + business logic (Python controllers)
- **Reports**: Financial and operational reports (Query Report, Script Report)
- **Pages**: Custom web pages
- **Print Formats**: Document printing templates
- **Dashboards**: Analytics and KPI views

### Module Organization

```
erpnext/
├── accounts/          # Financial accounting, invoicing, payments, GL entries
├── assets/            # Asset management, depreciation
├── buying/            # Purchase orders, suppliers, procurement
├── selling/           # Sales orders, customers, quotations
├── stock/             # Inventory, warehouses, stock movements, serial/batch
├── manufacturing/     # BOM, work orders, production planning, job cards
├── projects/          # Project management, tasks, timesheets
├── crm/              # Leads, opportunities, campaigns
├── support/          # Issues, service level agreements, warranty
├── setup/            # Company, employees, global settings
├── quality_management/ # Quality procedures, goals, reviews
├── subcontracting/   # Subcontracting workflows
├── controllers/      # Base controller classes (inheritance hierarchy)
├── regional/         # Country-specific customizations
├── utilities/        # Shared utilities and helpers
├── public/           # Frontend assets (JS, CSS, images)
├── templates/        # Jinja templates
└── tests/            # Test fixtures and utilities
```

### Controller Inheritance Hierarchy

ERPNext uses a sophisticated multi-level inheritance structure for transaction controllers:

```
frappe.model.document.Document (Frappe Framework base)
    ↓
erpnext.controllers.accounts_controller.AccountsController
    ├── Currency handling & conversion
    ├── Tax calculations (item-wise, total)
    ├── Multi-currency support
    ├── Price list logic
    ├── Discounts and margins
    └── Payment terms
    ↓
erpnext.controllers.stock_controller.StockController
    ├── Inventory validation
    ├── Serial/batch number handling
    ├── Quality inspection
    ├── Stock ledger entries (SLE)
    ├── GL entries for perpetual inventory
    └── Warehouse validation
    ↓
    ├── erpnext.controllers.buying_controller.BuyingController
    │       ├── Supplier-specific logic
    │       ├── Purchase taxes and landed costs
    │       ├── Purchase analytics
    │       └── Used by: Purchase Order, Purchase Invoice, Purchase Receipt
    │
    └── erpnext.controllers.selling_controller.SellingController
            ├── Customer-specific logic
            ├── Sales taxes and commissions
            ├── Sales analytics
            └── Used by: Sales Order, Sales Invoice, Delivery Note, Quotation
```

**Key Controllers:**
- `AccountsController`: Base for all transactions with financial implications
- `StockController`: Base for transactions involving inventory
- `BuyingController`: Specialized for purchase-side transactions
- `SellingController`: Specialized for sales-side transactions
- `StatusUpdater`: Mixin for documents with percentage-based status (e.g., % delivered, % billed)
- `SubcontractingController`: Handles subcontracting-specific workflows

### Important Patterns

#### 1. Stock Ledger Entry (SLE)

All inventory movements create Stock Ledger Entries that maintain:
- Real-time stock balance per item-warehouse combination
- Valuation rate calculations (FIFO, Moving Average, LIFO)
- Serial number and batch tracking
- Reserved quantity tracking

```python
# Stock Ledger Entry is automatically created by StockController
# When a document posts inventory transactions
class MyStockDocument(StockController):
    def on_submit(self):
        # StockController handles SLE creation
        super().on_submit()
```

#### 2. Serial and Batch Bundle

Modern approach (v14+) to handle serial/batch numbers:
- Centralized tracking via `Serial and Batch Bundle` DocType
- Handles both serial numbers and batches in one unified system
- Maintains complete transaction history
- Validates quantity against serial/batch availability

#### 3. General Ledger (GL) Entry

All financial transactions create GL Entries:
- Double-entry bookkeeping
- Account-wise ledger
- Party ledger (Customer/Supplier)
- Cost center and project dimension tracking

#### 4. Perpetual Inventory

When enabled, stock transactions also create GL entries:
- Stock In Hand account (Asset)
- Stock Received But Not Billed (Liability)
- Cost of Goods Sold (Expense)

#### 5. Taxes and Totals

Complex tax calculation system (`taxes_and_totals.py`):
- Item-wise tax application
- Compound taxes
- Tax templates
- Discount distribution

### DocType Structure

Each DocType typically has:
```
doctype_name/
├── doctype_name.json          # Schema definition (fields, permissions)
├── doctype_name.py            # Business logic (controller class)
├── test_doctype_name.py       # Unit tests
├── doctype_name.js            # Client-side logic (form scripts)
├── doctype_name_list.js       # List view customization (optional)
└── doctype_name_dashboard.py # Dashboard config (optional)
```

### Hooks System

The `hooks.py` file is the central configuration:
- DocType lifecycle events (`doc_events`)
- Scheduled jobs (`scheduler_events`)
- Permission overrides (`permission_query_conditions`, `has_permission`)
- Regional overrides for country-specific features
- Jinja methods and global functions
- Website routes

## Testing Guidelines

### Test Structure

- Tests inherit from `frappe.tests.utils.FrappeTestCase`
- Test fixtures in `erpnext/tests/` provide reusable test data
- Each module has its own test data setup
- Tests use `before_tests` hook to set up company and defaults

### Writing Tests

```python
from frappe.tests.utils import FrappeTestCase
import frappe

class TestSalesInvoice(FrappeTestCase):
    def test_sales_invoice_creation(self):
        si = frappe.get_doc({
            "doctype": "Sales Invoice",
            "customer": "_Test Customer",
            "items": [{
                "item_code": "_Test Item",
                "qty": 1,
                "rate": 100
            }]
        })
        si.insert()
        si.submit()
        
        self.assertEqual(si.grand_total, 100)
        self.assertEqual(si.docstatus, 1)
```

### Test Fixtures

Common test fixtures are in `erpnext/tests/`:
- `_Test Company`
- `_Test Customer`
- `_Test Supplier`
- `_Test Item`
- `_Test Warehouse`

## Key Configuration Files

- **pyproject.toml**: Python dependencies, ruff config, build system
- **package.json**: Node.js dependencies (if any frontend build)
- **.pre-commit-config.yaml**: Pre-commit hooks for linting
- **erpnext/hooks.py**: App-level configuration and event hooks
- **patches.txt**: Database migration patches (sequential, one per line)
- **modules.txt**: List of modules in the app

## Important ERPNext Concepts

### 1. Company

All transactions are company-specific:
- Multiple companies supported
- Each company has its own chart of accounts
- Each company can have different settings (perpetual inventory, fiscal year, etc.)

### 2. Fiscal Year

Financial reporting periods:
- Defines the accounting year
- Used for period-based reports
- Multiple fiscal years can exist

### 3. Cost Center

Profit center tracking:
- Hierarchical structure (tree)
- Used for departmental accounting
- Can be mandatory or optional per company

### 4. Accounting Dimensions

Customizable dimension tracking:
- Project, Cost Center (built-in)
- Custom dimensions can be added
- Automatically added to all accounting transactions

### 5. Item Variants

Product variations:
- Template Item with variant attributes
- Auto-generated item codes
- Variant-specific pricing and properties

## Code Style

### Python

- **Line Length**: 110 characters max
- **Indentation**: Tabs (not spaces)
- **Formatter**: Ruff
- **Type Hints**: Encouraged for new code
- **Docstrings**: Required for public methods
- **Naming**: snake_case for functions/variables, PascalCase for classes

### JavaScript

- **Style**: Follow Frappe's ESLint config
- **Formatter**: Prettier
- **Modern JS**: Use ES6+ features

### Commits

- Follow conventional commits format
- Enforced by commitlint in pre-commit hooks
- Format: `type(scope): subject`
- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

## Domain-Specific Notes

### Accounts Module

- **GL Entry**: Every financial transaction creates GL entries
- **Payment Entry**: Handles payments against invoices
- **Journal Entry**: Manual accounting entries
- **Period Closing**: Restricts backdated entries after closing
- **Multi-currency**: Exchange rate management and revaluation

### Stock Module

- **Stock Entry**: Generic inventory transaction (Material Transfer, Manufacture, Repack)
- **Serial and Batch**: Track-able items with unique identifiers
- **Warehouse**: Tree structure for location hierarchy
- **Stock Reconciliation**: Adjust stock to match physical count
- **Landed Cost Voucher**: Distribute additional costs to items

### Manufacturing Module

- **BOM (Bill of Materials)**: Defines raw materials needed
- **Work Order**: Production order tracking
- **Job Card**: Shop floor operation tracking
- **Material Request**: Planning and procurement
- **Production Plan**: MRP-style planning tool

### Buying Module

- **Purchase Order**: Commitment to buy
- **Purchase Receipt**: Goods received
- **Purchase Invoice**: Accounting entry
- **Supplier Quotation**: Quote from supplier

### Selling Module

- **Quotation**: Sales quote to customer
- **Sales Order**: Commitment to sell
- **Delivery Note**: Goods shipped
- **Sales Invoice**: Accounting entry

### Regional Customizations

Country-specific features in `erpnext/regional/`:
- Tax compliance (GST for India, VAT for various countries)
- E-invoicing integrations
- Address formats
- Banking and payment integrations

## Debugging

### Enable Developer Mode

```bash
bench --site [sitename] set-config developer_mode 1
bench --site [sitename] clear-cache
```

In developer mode:
- DocType schema changes allowed via Desk
- Error pages show full tracebacks
- JavaScript files not minified
- Missing translations highlighted

### View Logs

```bash
# Application error logs
tail -f ../../logs/[sitename]/error.log

# Web request logs
tail -f ../../logs/[sitename]/web.log

# Bench logs
tail -f ../../bench.log
```

### SQL Query Logging

Add to `sites/[sitename]/site_config.json`:
```json
{
  "db_query_log": 1
}
```

### Request Recorder

Frappe's built-in performance profiler:
```bash
bench --site [sitename] set-config recorder_enabled 1
```
View recordings in Desk: Setup > Recorder

## Important Notes

- **Never modify stock balance directly** - always use Stock Entry or Stock Reconciliation
- **GL Entries are immutable** - cancel and create new entries instead of modifying
- **Perpetual Inventory**: When enabled, stock transactions create GL entries automatically
- **Company is mandatory** for all transactions
- **Cost Center** may be mandatory depending on company settings
- **Whitelisted methods only**: Only `@frappe.whitelist()` methods are callable via API
- **Test with test data**: Always use `_Test` prefixed fixtures in tests

## Documentation and Resources

- **Official Docs**: https://docs.erpnext.com/
- **User Manual**: https://docs.erpnext.com/docs/user/manual/en
- **Developer Docs**: https://frappeframework.com/docs
- **API Reference**: https://frappeframework.com/docs/v15/api
- **Frappe School**: https://frappe.school
- **Discussion Forum**: https://discuss.erpnext.com/
- **GitHub Issues**: https://github.com/frappe/erpnext/issues
- **Translations**: https://translate.erpnext.com