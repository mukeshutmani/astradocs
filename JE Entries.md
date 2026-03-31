# Journal Entries Documentation

## Overview

Journal Entries is the core accounting module in PowerSuite that records all financial transactions using the double-entry bookkeeping system. Every financial activity (invoices, payments, receipts, costs) automatically generates corresponding journal entries to maintain accurate financial records.

---

## Table of Contents

1. [What are Journal Entries?](#what-are-journal-entries)
2. [Types of Journal Entries](#types-of-journal-entries)
3. [Key Concepts](#key-concepts)
4. [Database Structure](#database-structure)
5. [Workflow](#workflow)
6. [Automatic Journal Entry Generation](#automatic-journal-entry-generation)
7. [Manual Journal Entries](#manual-journal-entries)
8. [API Endpoints](#api-endpoints)
9. [Frontend Components](#frontend-components)
10. [Examples](#examples)

---

## What are Journal Entries?

Journal Entries are the fundamental records of all financial transactions in a business. They follow the **double-entry accounting** principle where every transaction affects at least two accounts:

- **Debit**: Money going out or assets increasing
- **Credit**: Money coming in or liabilities increasing

**Rule**: Total Debits must always equal Total Credits.

---

## Types of Journal Entries

| Type | Description | Editable |
|------|-------------|----------|
| **System JE** | Automatically generated from documents (invoices, receipts, payments, etc.) | No (Read-only) |
| **Manual JE** | Created manually by users for adjustments, corrections, or special entries | Yes |

---

## Key Concepts

### 1. Journal Batch
A container that groups related journal entries together, organized by:
- **Period**: Monthly fiscal period (e.g., 122025 for December 2025)
- **Branch**: Business branch the entries belong to
- **Batch Number**: Auto-generated unique identifier (e.g., KHJB00000001)

### 2. Journal Entry
Individual debit/credit records within a batch containing:
- Transaction date
- Chart of Account (GL Account)
- Sub-account (optional)
- GL Entity ID (optional reference)
- Debit amount
- Credit amount
- Description
- Analysis codes (for linking to source documents)

### 3. Journal Period
Fiscal periods that define when entries can be posted:
- Grouped by fiscal year (format: MMYYYY, e.g., 072025 for fiscal year starting July 2025)
- Each fiscal year contains 12 monthly periods
- Status: Boolean (`true` = Open, `false` = Closed)
- Only open periods accept new entries
- Scoped to company via `company_code`
- Periods include a `report_date` for reporting purposes
- Period date ranges cannot overlap across fiscal years

### 4. Journal Entry Types
Lookup codes that classify journal entries (e.g., Invoice, Receipt, Payment). Used by posting rules to determine which GL accounts to debit/credit.

### 5. Posting Rules
Configuration rules that define how documents automatically generate journal entries:
- Linked to a `journal_entry_type` (not a free-text document type)
- Specify a single `chart_of_account_id` with a `debit_credit` indicator ('debit' or 'credit')
- Can be filtered by service type, customer type, and branch
- Support `sub_account_id` for sub-ledger entries
- Have a `record_type` of 'Default' or 'Additional'
- Include `prefix_code` for batch numbering
- Scoped to company via `company_code`

### 6. Posting Number Types
Lookup codes used by posting rules to determine batch number prefixes and sequencing.

---

## Database Structure

### Tables

```
┌─────────────────────────────────────────────────────────────────┐
│                      journal_batches                            │
├─────────────────────────────────────────────────────────────────┤
│ id                  - Primary key (auto-increment)              │
│ batch_no            - Unique batch number (e.g., KHJB00000001) │
│ journal_entry_period- Period in MMYYYY format                   │
│ branch_id           - Foreign key to branches (nullable)        │
│ batch_type          - ENUM: 'Manual JE' or 'System JE'         │
│ narratives          - Description/notes (TEXT)                  │
│ dated               - Batch date (defaults to CURRENT_TIMESTAMP)│
│ status              - ENUM: 'Open','Drafted','Posted','Void'   │
│ created_by          - Foreign key to users                      │
│ updated_by          - Foreign key to users                      │
│ created_at          - Creation timestamp                        │
│ updated_at          - Last update timestamp                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      journal_entries                            │
├─────────────────────────────────────────────────────────────────┤
│ id                  - Primary key (auto-increment)              │
│ entry_no            - Entry number within batch (default: 1)    │
│ journal_batch_id    - Foreign key to journal_batches (CASCADE)  │
│ transaction_date    - Date of transaction                       │
│ chart_of_account_id - Foreign key to chart_of_accounts          │
│ sub_account_id      - Foreign key to sub_accounts (nullable)    │
│ debit               - Debit amount (DECIMAL 10,2)               │
│ credit              - Credit amount (DECIMAL 10,2)              │
│ gl_entity_id        - GL entity reference (STRING, nullable)    │
│ description         - Entry description (TEXT)                  │
│ analysis_code1      - Analysis/reference code (STRING 160)      │
│ analysis_code2      - Analysis/reference code (STRING 160)      │
│ analysis_code3      - Analysis/reference code (STRING 160)      │
│ analysis_code4      - Analysis/reference code (STRING 160)      │
│ analysis_code5      - Analysis/reference code (STRING 160)      │
│ analysis_code6      - Analysis/reference code (STRING 160)      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      journal_periods                            │
├─────────────────────────────────────────────────────────────────┤
│ id                  - Primary key (auto-increment)              │
│ company_code        - Foreign key to companies                  │
│ fiscal_year         - Fiscal year code (STRING, e.g., 072025)   │
│ month               - Month number (1-12)                       │
│ year                - Calendar year                             │
│ status              - BOOLEAN (true=Open, false=Closed)         │
│ start_date          - Period start date                         │
│ end_date            - Period end date                           │
│ report_date         - Report date                               │
│ created_at          - Creation timestamp                        │
│ updated_at          - Last update timestamp                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    journal_entry_types                          │
├─────────────────────────────────────────────────────────────────┤
│ id                  - Primary key (auto-increment)              │
│ code                - Type code (STRING)                        │
│ description         - Type description (STRING)                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      posting_rules                              │
├─────────────────────────────────────────────────────────────────┤
│ id                  - Primary key (auto-increment)              │
│ company_code        - Foreign key to companies                  │
│ journal_entry_type_id - Foreign key to journal_entry_types      │
│ posting_number_type_id - Foreign key to posting_number_types    │
│ customer_type_id    - Foreign key to customer_types (nullable)  │
│ branch_id           - Foreign key to branches (nullable)        │
│ debit_credit        - ENUM: 'debit' or 'credit'                │
│ service_type_id     - Foreign key to service_types (nullable)   │
│ sub_account_id      - Foreign key to sub_accounts (nullable)    │
│ chart_of_account_id - Foreign key to chart_of_accounts (nullable)│
│ prefix_code         - Batch number prefix (STRING, nullable)    │
│ record_type         - ENUM: 'Default' or 'Additional'          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   posting_number_types                          │
├─────────────────────────────────────────────────────────────────┤
│ id                  - Primary key (auto-increment)              │
│ code                - Type code (STRING)                        │
│ description         - Type description (STRING)                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  posting_number_prefixes                        │
├─────────────────────────────────────────────────────────────────┤
│ id                  - Primary key (auto-increment)              │
│ posting_number_type_id - Foreign key to posting_number_types    │
│ code                - Prefix code (STRING)                      │
│ description         - Prefix description (STRING)               │
└─────────────────────────────────────────────────────────────────┘
```

### Relationships

```
journal_batches (1) ──────── (Many) journal_entries
       │                                │
       ├── branch_id ──── branches      ├── chart_of_account_id ──── chart_of_accounts
       └── created_by ──── users        └── sub_account_id ──── sub_accounts

journal_periods ──── company_code ──── companies

posting_rules ──── journal_entry_type_id ──── journal_entry_types
       │
       ├── posting_number_type_id ──── posting_number_types
       ├── customer_type_id ──── customer_types
       ├── branch_id ──── branches
       ├── service_type_id ──── service_types
       ├── sub_account_id ──── sub_accounts
       └── chart_of_account_id ──── chart_of_accounts

posting_number_prefixes ──── posting_number_type_id ──── posting_number_types
```

---

## Workflow

### System Journal Entry Flow

```
Document Created (Invoice/Receipt/Payment/Cost)
                    │
                    ▼
        ┌───────────────────┐
        │  Posting Rules    │
        │  Engine Checks    │
        └─────────┬─────────┘
                  │
                  ▼
        ┌───────────────────┐
        │  Determine GL     │
        │  Accounts         │
        └─────────┬─────────┘
                  │
                  ▼
        ┌───────────────────┐
        │  Create Journal   │
        │  Entry            │
        └─────────┬─────────┘
                  │
                  ▼
        ┌───────────────────┐
        │  Add to Batch     │
        │  (by Period)      │
        └─────────┬─────────┘
                  │
                  ▼
        ┌───────────────────┐
        │  Post to General  │
        │  Ledger           │
        └───────────────────┘
```

### Batch Status Flow

```
Open ──► Drafted ──► Posted ──► Void
  │                     │
  └── Can edit entries  └── Cannot edit (locked)
```

---

## Automatic Journal Entry Generation

The system automatically generates journal entries when these documents are created:

### 1. Invoice (Sales)
```
When invoice is created for PKR 100,000:

Debit:  Accounts Receivable (Customer)    100,000
Credit: Sales Revenue                     100,000
```

### 2. Receipt (Customer Payment)
```
When customer pays PKR 100,000:

Debit:  Bank/Cash Account                 100,000
Credit: Accounts Receivable (Customer)    100,000
```

### 3. Customer Deposit (Advance Payment)
```
When customer pays advance PKR 50,000:

Debit:  Bank/Cash Account                 50,000
Credit: Customer Deposits (Liability)     50,000
```

### 4. Cost Entry (Supplier Cost)
```
When cost is recorded for PKR 80,000:

Debit:  Cost of Sales / Expense           80,000
Credit: Accounts Payable (Supplier)       80,000
```

### 5. Supplier Payment
```
When paying supplier PKR 80,000:

Debit:  Accounts Payable (Supplier)       80,000
Credit: Bank/Cash Account                 80,000
```

### 6. Credit Note (Customer Refund)
```
When credit note issued for PKR 10,000:

Debit:  Sales Revenue (or Returns)        10,000
Credit: Accounts Receivable (Customer)    10,000
```

### 7. Debit Note
```
When debit note issued:

Debit:  Accounts Payable (Supplier)       Amount
Credit: Purchase Returns                  Amount
```

### 8. Refund
```
When refund processed:

Debit:  Refund Expense                    Amount
Credit: Bank/Cash Account                 Amount
```

---

## Manual Journal Entries

### When to Use Manual JE

1. **Adjusting Entries**: Month-end or year-end adjustments
2. **Corrections**: Fix errors in previous entries
3. **Non-standard Transactions**: Transactions not covered by automatic rules
4. **Depreciation**: Asset depreciation entries
5. **Accruals**: Accrued expenses or income
6. **Transfers**: Inter-account or inter-branch transfers

### Creating Manual JE

1. Navigate to General Entries > Journal Entry Batches
2. Click "Add New Batch"
3. Select:
   - Period (MMYYYY)
   - Branch
   - Batch Type: "Manual JE"
4. Add entries with:
   - Date
   - GL Account
   - Debit/Credit amount
   - Description
5. Ensure total Debits = total Credits
6. Save and Post

---

## API Endpoints

### Journal Entry Batches & Entries

**Base URL**: `/api/journal-entry`

| Method | Endpoint | Controller Function | Description |
|--------|----------|-------------------|-------------|
| GET | `/` | `journalEntryBatches` | Get all journal batches |
| GET | `/periods` | `getPeriodInfo` | Get available journal periods |
| GET | `/resolveDoc/:docNumber` | `resolveDocumentLink` | Resolve a document number to its source link |
| GET | `/:batch_no` | `journalEntryBatch` | Get single batch details by batch number |
| GET | `/generate/:branch_id` | `generateJournalEntries` | Generate system journal entries for a branch |
| GET | `/entryNumbers/:batch_id` | `entryNumberList` | Get all entry numbers in a batch |
| GET | `/entries/:batch_id/:entry_no` | `entriesListByBatchAndNumber` | Get entries for a specific entry number |
| POST | `/` | `createJournalEntryBatch` | Create new batch |
| POST | `/entries` | `createJournalEntries` | Create journal entries |
| POST | `/regenerate` | `regenerateBatches` | Regenerate batches |
| PUT | `/:batch_id` | `updateJournalEntryBatch` | Update batch |
| PUT | `/entries/:batch_id` | `updateJournalEntries` | Update entries in a batch |
| PUT | `/void/:batch_no` | `voidBatch` | Void a batch by batch number |

### Journal Entry Types

**Base URL**: `/api/journal-data`

| Method | Endpoint | Controller Function | Description |
|--------|----------|-------------------|-------------|
| GET | `/journal_entry_types` | `listJournalEntryTypes` | List all journal entry types |
| GET | `/journal_entry_types/:id` | `journalEntryTypeDetails` | Get single entry type details |
| POST | `/journal_entry_types` | `createJournalEntryType` | Create new entry type |
| PUT | `/journal_entry_types/:id` | `updateJournalEntryType` | Update entry type |
| DELETE | `/journal_entry_types/:id` | `deleteJournalEntryType` | Delete entry type |

### Posting Rules

**Base URL**: `/api/posting-rule`

| Method | Endpoint | Controller Function | Description |
|--------|----------|-------------------|-------------|
| GET | `/` | `postingRules` | Get all posting rules |
| GET | `/posting_number_types` | `postingNumberTypes` | Get posting number types |
| POST | `/additional_posting_rule/:id` | `additionalPostingRule` | Copy/add additional posting rule |
| PUT | `/update_posting_rule` | `updatePostingRules` | Update a posting rule |
| DELETE | `/delete_posting_rule/:id` | `deletePostingRule` | Delete a posting rule |

### Journal Periods

**Base URL**: `/api/jvPeriod`

| Method | Endpoint | Controller Function | Description |
|--------|----------|-------------------|-------------|
| GET | `/` | `getAllJvPeriods` | Get all journal periods (filter by `?fiscalYear=`) |
| GET | `/fishalYears` | `getFiscalYears` | Get list of distinct fiscal years |
| GET | `/check-deletion/:fiscalYear` | `checkFiscalYearDeletion` | Check if a fiscal year can be deleted |
| POST | `/` | `createJvPeriodsForYear` | Generate 12 monthly periods for a fiscal year |
| POST | `/update` | `updateJvPeriods` | Update multiple journal periods |
| DELETE | `/:fiscalYear` | `deleteFiscalYear` | Delete all periods for a fiscal year |

### Request/Response Examples

#### Create Batch
```json
POST /api/journal-entry
{
  "journal_entry_period": "122025",
  "branch_id": 1,
  "batch_type": "Manual JE",
  "narratives": "December adjustments"
}
```

#### Create Entries
```json
POST /api/journal-entry/entries
{
  "batch_id": 1,
  "entries": [
    {
      "entry_no": 1,
      "transaction_date": "2025-12-25",
      "chart_of_account_id": 101,
      "debit": 50000,
      "credit": 0,
      "description": "Rent expense"
    },
    {
      "entry_no": 1,
      "transaction_date": "2025-12-25",
      "chart_of_account_id": 201,
      "debit": 0,
      "credit": 50000,
      "description": "Rent expense"
    }
  ]
}
```

#### Void a Batch
```json
PUT /api/journal-entry/void/KHJB00000001
```

#### Generate Fiscal Year Periods
```json
POST /api/jvPeriod
{
  "month": 7,
  "year": 2025
}
```

---

## Frontend Components

### Journal Entry Batches & Entries

**Location**: `psfront/src/pages/GeneralEntries/JournalEntryBatch/`

| Component | Description |
|-----------|-------------|
| `JournalEntryBatches.jsx` | List all batches with filters |
| `JournalEntries.jsx` | View entries within a batch |
| `JournalEntriesImproved.jsx` | Enhanced entries view |
| `AddJournalEntryBatch.jsx` | Create new batch form |
| `AddJournalEntryBatchImproved.jsx` | Enhanced batch creation |
| `AddSystemJE.jsx` | Generate system entries |

### Journal Periods

**Location**: `psfront/src/pages/GeneralEntries/`

| Component | Description |
|-----------|-------------|
| `JournalPeriod.jsx` | Manage fiscal years and journal periods |
| `PostingRule.jsx` | Manage posting rules configuration |

### Journal Entry Types

**Location**: `psfront/src/pages/JournalEntryTypes/`

| Component | Description |
|-----------|-------------|
| `JournalEntryType.jsx` | List and manage entry types |
| `AddJournalEntryType.jsx` | Create/edit entry type form |

### Frontend API Modules

#### `psfront/src/api/journal_entry.js`

```javascript
getJournalEntryBatches(params)          // GET /journalEntry
getSingleJournalEntryBatch(batch_no)    // GET /journalEntry/:batch_no
createJournalEntryBatch(data)           // POST /journalEntry
updateEntryBatch(batch_id, data)        // PUT /journalEntry/:batch_id
listOfEntryNumbers(batch_id)            // GET /journalEntry/entryNumbers/:batch_id
getEntriesListByBatchAndNumber(batch_id, entry_no) // GET /journalEntry/entries/:batch_id/:entry_no
createOrUpdateJournalEntries(batch_id, data)       // PUT /journalEntry/entries/:batch_id
generateSystemJournalEntry(branch_id, data)        // GET /journalEntry/generate/:branch_id
regenerateJournalBatches(batchIds)      // POST /journalEntry/regenerate
getPeriodInfo()                         // GET /journalEntry/periods
resolveDocumentLink(docNumber)          // GET /journalEntry/resolveDoc/:docNumber
voidJournalBatch(batch_no)             // PUT /journalEntry/void/:batch_no
```

#### `psfront/src/api/journal_data.js`

```javascript
getJournalEntryType(query, page, pageLimit, search) // GET /journal-data/journal_entry_types
getSingleJournalEntryType(id)           // GET /journal-data/journal_entry_types/:id
createJournalEntryType(data)            // POST /journal-data/journal_entry_types
updateJournalEntryType(id, data)        // PUT /journal-data/journal_entry_types/:id
deleteJournalEntryType(id)              // DELETE /journal-data/journal_entry_types/:id
```

#### `psfront/src/api/journal_perios.js`

```javascript
getJournalPeriods(fiscalYear)           // GET /jvPeriod?fiscalYear=
getFiscalYears()                        // GET /jvPeriod/fishalYears
generateJournalPeriods(data)            // POST /jvPeriod
updateJournalPeriods(data)              // POST /jvPeriod/update
checkFiscalYearDeletion(fiscalYear)     // GET /jvPeriod/check-deletion/:fiscalYear
deleteFiscalYear(fiscalYear)            // DELETE /jvPeriod/:fiscalYear
```

#### `psfront/src/api/posting.js`

```javascript
getPostingNumberTypes()                 // GET /posting-rule/posting_number_types
getPostingRules({ query })              // GET /posting-rule
copyPostingRule(id)                     // POST /posting-rule/additional_posting_rule/:id
deletePostingRule(id)                   // DELETE /posting-rule/delete_posting_rule/:id
updatePostingRule(data)                 // PUT /posting-rule/update_posting_rule
```

---

## Backend Service Layer

### `psback/services/journal.js`

| Function | Description |
|----------|-------------|
| `generateJournal` | Core function that generates journal entries from documents (invoices, receipts, payments, etc.) |
| `regenerateBatches` | Regenerates existing journal batches |
| `generateVoidDepositJE` | Generates immediate void reversal journal entries for customer deposits |
| `generateVoidInvoiceJE` | Generates immediate void reversal journal entries for invoices when voided |

---

## Examples

### Example 1: Complete Ticket Sale Cycle

**Scenario**: Customer buys a ticket for PKR 100,000. Cost from airline is PKR 85,000.

#### Step 1: Invoice Created
```
Entry #1:
Debit:  Accounts Receivable - Customer    100,000
Credit: Sales Revenue - Tickets           100,000
```

#### Step 2: Cost Recorded
```
Entry #2:
Debit:  Cost of Sales - Tickets           85,000
Credit: Accounts Payable - Airline        85,000
```

#### Step 3: Customer Pays
```
Entry #3:
Debit:  Bank Account                      100,000
Credit: Accounts Receivable - Customer    100,000
```

#### Step 4: Pay Airline
```
Entry #4:
Debit:  Accounts Payable - Airline        85,000
Credit: Bank Account                      85,000
```

#### Result: Profit = 100,000 - 85,000 = PKR 15,000

---

### Example 2: Customer Deposit Flow

**Scenario**: Customer pays PKR 50,000 advance. Later uses PKR 30,000 against invoice.

#### Step 1: Deposit Received
```
Debit:  Bank Account                      50,000
Credit: Customer Deposits (Liability)     50,000
```

#### Step 2: Invoice Created (PKR 30,000)
```
Debit:  Accounts Receivable               30,000
Credit: Sales Revenue                     30,000
```

#### Step 3: Deposit Applied to Invoice
```
Debit:  Customer Deposits                 30,000
Credit: Accounts Receivable               30,000
```

#### Result: Remaining Deposit = PKR 20,000

---

## Analysis Codes

Journal entries support 6 analysis codes for cross-referencing:

| Code | Typical Usage |
|------|---------------|
| `analysis_code1` | Invoice/Document Number |
| `analysis_code2` | Customer/Supplier ID |
| `analysis_code3` | Bank ID / Service ID |
| `analysis_code4` | Branch ID / Product Code |
| `analysis_code5` | Check Number / Branch Code |
| `analysis_code6` | Card Number / Custom Reference |

These codes enable:
- Drill-down from GL to source documents
- Report filtering and grouping
- Audit trail maintenance

---

## Best Practices

1. **Always balance entries**: Ensure Debits = Credits
2. **Use correct periods**: Post entries to appropriate fiscal periods
3. **Add descriptions**: Include meaningful descriptions for audit trail
4. **Use analysis codes**: Link entries to source documents
5. **Review before posting**: Check entries before changing status to Posted
6. **Don't delete**: Void entries instead of deleting for audit trail
7. **Regular reconciliation**: Reconcile GL balances with sub-ledgers

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Entries not balancing | Check total debits = total credits |
| Cannot post to period | Ensure period status is open (true) |
| Missing GL account | Add account to Chart of Accounts |
| System JE not generating | Check posting rules configuration |
| Batch locked | Posted batches cannot be edited; void and recreate if needed |
| Period overlap error | Ensure period date ranges don't overlap across fiscal years |

---

## Related Modules

- **Chart of Accounts**: GL account master data
- **Sub Accounts**: Customer/Supplier sub-ledgers
- **Posting Rules**: Automatic entry generation rules
- **Journal Entry Types**: Entry type classifications
- **Posting Number Types**: Batch numbering configuration
- **Trial Balance**: Summary of all GL balances
- **General Ledger Report**: Detailed transaction listing

---

## File Locations

```
Backend:
├── controllers/
│   ├── journal_entry.controller.js     # Batch & entry CRUD, generate, void
│   ├── journal_data.controller.js      # Journal entry types CRUD
│   ├── posting.controller.js           # Posting rules management
│   └── jvPeriod.controller.js          # Journal periods management
├── models/general/
│   ├── journal_batch.js
│   ├── journal_entry.js
│   ├── journal_period.js
│   └── posting_rule.js
├── models/system_models/
│   ├── journal_entry_type.js
│   ├── posting_number_type.js
│   └── posting_number_prefix.js
├── services/
│   └── journal.js                      # generateJournal, regenerateBatches, generateVoidDepositJE
└── routes/
    ├── journal_entry.route.js          # /api/journal-entry
    ├── journal_data.route.js           # /api/journal-data
    ├── posting.route.js                # /api/posting-rule
    └── jvperiod.route.js              # /api/jvPeriod

Frontend:
├── pages/GeneralEntries/
│   ├── JournalEntryBatch/
│   │   ├── JournalEntryBatches.jsx
│   │   ├── JournalEntries.jsx
│   │   ├── JournalEntriesImproved.jsx
│   │   ├── AddJournalEntryBatch.jsx
│   │   ├── AddJournalEntryBatchImproved.jsx
│   │   └── AddSystemJE.jsx
│   ├── JournalPeriod.jsx
│   └── PostingRule.jsx
├── pages/JournalEntryTypes/
│   ├── JournalEntryType.jsx
│   └── AddJournalEntryType.jsx
└── api/
    ├── journal_entry.js                # Batch & entry operations
    ├── journal_data.js                 # Journal entry types
    ├── journal_perios.js               # Journal periods & fiscal years
    └── posting.js                      # Posting rules
```

---

*Document Version: 2.0*
*Last Updated: February 2026*
