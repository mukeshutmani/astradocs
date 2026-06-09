# Supplier Position Report - Comprehensive Analysis

## Overview

The Supplier Position Report provides a **summary view** of each supplier's financial position (payable balance). It shows one row per supplier with: opening balance, costs incurred, refunds, payments, advance payments, JV adjustments, and net balance.

**Key Difference from Customer Position Report**: Suppliers are liabilities — JV Credit INCREASES payable, JV Debit DECREASES payable (opposite of customer accounts).

---

## 1. API Endpoint

```
POST /api/report/getSupplierPositionReport
Authentication: JWT cookie
Permission: "Supplier-Position-Report"
Timeout: 5 minutes (300000ms)
```

### Request Body

```javascript
{
  supplierFilter: "isNotBlank" | "isBlank" | "isEqual" | "between",
  supplier_id: number,           // Required if supplierFilter == "isEqual"
  supplierStart: number,         // Required if supplierFilter == "between"
  supplierEnd: number,           // Required if supplierFilter == "between"
  branchFilter: "isNotBlank" | "isBlank" | "isEqual" | "between",
  branch_id: number,             // Required if branchFilter == "isEqual"
  branchStart: number,           // Required if branchFilter == "between"
  branchEnd: number,             // Required if branchFilter == "between"
  dateFilter: "blank" | "=" | "<" | "<=" | ">" | ">=" | "<>" | "between",
  startDate: "YYYY-MM-DD",      // Required unless dateFilter == "blank"
  endDate: "YYYY-MM-DD",        // Required if dateFilter == "between"
  adjustmentDateMode: boolean,   // true = use adjustment_date for posted docs
  includeRaised: boolean,        // true = also include un-printed "Raised" XOs/costs
  type: "pdf" | "excel"         // Default: "pdf"
}
```

### Response

```javascript
{
  status: 200,
  message: "success",
  link: string,               // S3/MinIO URL
  downloadLink: string,       // Signed URL for downloads
  report: {
    id, user_id, file_type,
    report_number: "TPSP{timestamp}",
    report_type: "supplier-position-report",
    created_at, updated_at
  }
}
```

---

## 2. Report Columns (One Row Per Supplier)

| Column | Key | Description |
|--------|-----|-------------|
| Supplier No. | supplier_no | Supplier number |
| Supplier Name | supplier_name | Supplier name |
| Opening Balance B/F | opening_balance | Balance before report period |
| Add Sale Invoices | cost_invoice | Period cost total (excludes opening XOs) |
| Add Opening XO | opening_xo | Period imported opening XO total |
| Less Refund Invoices | refunds | Period refunds (debit notes) |
| Less Payments | payments | Period payments to supplier |
| Less Advance Payments | advance_payments | Period supplier deposits |
| Add JV Credit | jv_credit | Manual JE credit entries |
| Less JV Debit | jv_debit | Manual JE debit entries |
| Net Balance | net_balance | Final calculated balance |

Grand Total row sums all numeric columns.

---

## 3. Calculation Formulas

### 3.1 Cost Invoice (Sale Invoices)

```javascript
publishedRate = cost.published_rate
commissionAmount = (publishedRate * cost.commission%) / 100
netRate = publishedRate - commissionAmount
taxes = SUM(cost_taxes[].tax_amount)
sstAmount = (commissionAmount * cost.sst%) / 100
freeOfCost = cost.free_of_cost  // extra charges
costPerUnit = netRate + taxes + sstAmount + freeOfCost
totalCost = costPerUnit * cost.quantity

// Currency conversion
IF foreignCurrency:
  exchangeRate = FROZEN_RATE_FROM_COST || LIVE_RATE || 1
  totalAmount = ROUND(totalCost * exchangeRate)
```

**Status filter**: Only `Paid`, `Partially Paid`, `Printed` (exclude Void, Raised). When the **Include Raised XOs** option (`includeRaised`) is ON, `Raised` is added to this list — see [Include Raised XOs](#104-include-raised-xos-2026-06-08).
**Exclude**: costs where `total_costing <= 0`

### 3.2 Payment Calculation

```javascript
payments = SUM(payment_settlement_payments.amount)
// WHERE payment_settlement.status NOT IN ['Void', 'Raised']
```

### 3.3 Refund Calculation

Two sources:
1. **Service refunds**: Must have `debit_note_generated` or `debit_note_id`, debit note not Void. Uses `supplier_refund_amount_base` (PKR) preferred.
2. **Standalone debit notes**: Direct on supplier, not Void.

```javascript
totalRefunds = refundsFromServices + standaloneDebitNotes
```

### 3.4 Advance Payments

```javascript
advancePayments = SUM(supplier_deposits.amount)
// WHERE status != 'Void'
```

### 3.5 Journal Vouchers

```javascript
// Only batch_type = 'Manual JE', status IN ['Open', 'Posted']
// JV Credit INCREASES payable (add)
// JV Debit DECREASES payable (subtract)
```

### 3.6 Opening Balance

```javascript
opening_balance = historicalCosts
                - historicalPayments
                - historicalRefunds
                + historicalDebitNotes
                + historicalJvCredit
                - historicalJvDebit
                - historicalAdvancePayments
```

Only calculated when `startDate` is provided. If no date filter, opening balance = 0.

### 3.7 Net Balance (Final Formula)

```javascript
net_balance = opening_balance
            + cost_invoice        // ADD costs incurred
            + opening_xo          // ADD imported opening XOs
            - totalRefunds        // LESS refunds
            - paymentAmount       // LESS payments
            - totalAdvancePayments // LESS advance payments
            + jvCreditTotal       // ADD JV credit
            - jvDebitTotal        // LESS JV debit
```

**Positive** = Amount owed TO supplier (payable)
**Negative** = Supplier owes company (receivable)

---

## 4. Currency Conversion (Frozen Exchange Rate)

Exchange rate is **frozen at XO print time** — stored on `cost.exchange_rate`. Reports use stored rate first, falling back to live currency table rate for legacy costs:

```javascript
const storedCostRate = parseFloat(cost.exchange_rate || 0);
const exchangeRate = (storedCostRate > 1)
  ? storedCostRate                                          // Frozen rate (preferred)
  : (cost.currency_code?.currencies?.[0]?.exchange_rate || 1); // Live rate (fallback)
```

This ensures changing the exchange rate later does NOT retroactively affect already-printed XO documents.

---

## 5. Date Filtering Logic

### Cost/Service Date

- **Air services**: Use `service.ticket_issue_date` (fallback: `cost.created_at`)
- **Other services**: Use `cost.created_at`

### Adjustment Date Mode (Payments & Deposits)

When `adjustmentDateMode = true` (Posted to Ledger):
- If `adjustment_date` NULL (unposted): use `created_at`
- If `adjustment_date` exists (posted): use `adjustment_date`

When `adjustmentDateMode = false`:
- Always use `created_at`

---

## 6. Database Models Used

| Model | Purpose |
|-------|---------|
| supplier | Supplier master data |
| cost | XO/costing documents with pricing |
| cost_tax | Tax entries per cost |
| service | Service linking costs to orders |
| service_type | Service type (Air, Hotel, etc.) |
| payment_settlement | Payment records to supplier |
| payment_settlement_payment | Individual payment amounts |
| payment_settlement_cost | Links payments to costs |
| refund | Service refunds |
| debit_note | Debit notes (linked + standalone) |
| supplier_deposit | Advance payments to supplier |
| journal_entry | Manual JE entries |
| journal_batch | JE batch metadata |
| currency_code & currency | Exchange rate lookup |
| report | Report generation record |

---

## 7. Output Formats

### PDF
- **Template**: `psback/views/pages/reports/report2.ejs` (shared)
- **Page Size**: A3 Landscape, 5mm margins
- **Font**: Times New Roman, 8px (screen) / 7px (print)

### Excel
- **Library**: ExcelJS
- **Frozen Panes**: First 7 rows
- **Number Format**: `#,##0.00` (2 decimal places with comma separators)
- **Column Widths**: Auto-fit (min 10, max 50)

---

## 8. Special Handling & Edge Cases

1. **Frozen Exchange Rate**: `cost.exchange_rate` (saved at print time) preferred over live rate. Legacy costs without stored rate fall back to currency table
2. **Air Service Date**: Uses `ticket_issue_date` instead of `created_at` for Air/Flight/Ticket services
3. **Void Records Excluded**: All void/voided records skipped across costs, payments, debit notes, deposits
4. **Negative Costs Excluded**: Costs with `total_costing <= 0` are skipped
5. **Quantity Multiplication**: All cost components multiplied by `cost.quantity`
6. **Standalone Debit Notes**: Debit notes directly on supplier (not linked to refunds) are included in refund total
7. **Refund PKR Amount**: Uses `supplier_refund_amount_base` (PKR) if available, else `supplier_refund_amount`
8. **Company Isolation**: All data filtered by user's `company_code`
9. **No Date Filter = No Opening Balance**: Opening balance is 0 when no date filter applied
10. **JV for Suppliers**: Credit increases payable, Debit decreases (opposite of customer accounts)
11. **Payment Settlement Status**: Excludes both `Void` and `Raised` statuses

---

## 9. Code File References

| Component | File | Lines |
|-----------|------|-------|
| Backend Controller | `psback/controllers/report.controller.js` | 15774-16832 |
| Frontend Component | `psfront/src/pages/Report/SupplierPositionReport.jsx` | Full file |
| PDF Template | `psback/views/pages/reports/report2.ejs` | Shared template |
| API Route | `psback/routes/report.route.js` | Line 64 |
| API Client | `psfront/src/api/report.js` | Lines 286-294 |

---

## 10. Recent Updates and Fixes

### 10.1 Duplicate Debit Notes Per Refund — Opening Balance Fix (2026-04-28)

**Problem**: Opening Balance B/F in the next-period Supplier Position report did not match the previous-period Net Balance. Same root cause as the Supplier Account Statement bug fixed in `SUPPLIER_ACCOUNT_STATEMENT_ANALYSIS.md` section 16.5.

**Root cause**:
1. A refund (e.g. refund_id 434) had **multiple `debit_notes` rows** linked via `debit_note.refund_id`: one with `doc_status='Printed'` (the valid one) and one with `doc_status='Void'` (a stale/cancelled one).
2. The Sequelize association `refund.hasOne(debit_note, { foreignKey: 'refund_id' })` is non-deterministic when multiple rows match — without an `ORDER BY`, MySQL may return either row first.
3. In one query Sequelize picked the Printed DN (refund counted), in another query it picked the Void DN (refund skipped) → opening balance and current period totals diverged.

**Solution Implemented**:
1. Added `where: { doc_status: { [Op.notIn]: ['Void', 'Voided'] } }` to both `debit_note` includes (main query and historical query). With `required: false`, Sequelize emits a `LEFT JOIN ... ON dn.refund_id = r.id AND dn.doc_status NOT IN ('Void','Voided')`, so only valid debit notes are joined and `refund.debit_note` is always either the Printed one or `null`.
2. Added a guard in both refund loops: `if (refund.debit_note_generated && !refund.debit_note) <skip>;` — skips refunds where a DN was generated but no valid (non-Void) DN exists. Preserves the prior behavior of skipping all-Void refunds without falling through to the date-only path.

**Files Modified**:
- `psback/controllers/report.controller.js`
  - Main query refund→debit_note include (~line 16240): added `where` filter
  - Historical query refund→debit_note include (~line 16454): added `where` filter
  - Historical refund loop (~line 16545): added `!refund.debit_note` guard (uses `continue`)
  - Current period refund reduce (~line 16737): added `!refund.debit_note` guard (uses `return sum`)

**Edge cases handled**:
- Refund with one Printed + one Void DN → joins only Printed → counted ✓
- Refund with all Void DNs → join yields null → skipped ✓
- Refund with no DN (debit_note_generated=0) → caught by existing first guard ✓
- Refund with one Printed DN only → unchanged behavior ✓

---

### 10.2 Historical Payments Date Filter + Shared-Settlement Double-Counting Fix (2026-04-28)

**Problem**: Opening Balance B/F in the Supplier Position Report did not match the previous-period Net Balance for the same supplier. Example: Mar 1-30 Net = 38,130, but Mar 31-Apr 30 Opening = 26,130 (off by 12,000). The Supplier Account Statement showed the correct opening for the same supplier and dates — only the Position Report was wrong.

**Root cause** (two distinct bugs in the historical payment loop):

1. **Missing date filter**: The historical payment loop iterated `cost.payment_settlement_costs → settlement → payment_settlement_payments` for every historical service and summed every payment's amount. Unlike the Account Statement equivalent (which gates each settlement on `settlement_date < startDate`), this loop had **no date check** — so payments dated *after* `startDate` (which only become reachable because the cost they are linked to is historical) were being subtracted from the opening balance.

2. **Shared-settlement double-counting**: When a single payment settles multiple costs of the same supplier, the same `payment_settlement_payment` row is reachable through each cost's `payment_settlement_costs`. The loop visited it once per cost and summed it each time, inflating the payment total.

**How the 12,000 discrepancy was produced (TESTSUPPLIER, supplier_id 591)**:
- Historical costs before Mar 31: 5111 (20,000) + 5430 (6,930) + 5686 (11,200) = **38,130**
- Payment 792 (TTPY00000029, Rs 6,000, dated **April 6**) settled both cost 5111 and cost 5430 via settlement 877
- The loop visited it twice (once per cost) and added 6,000 + 6,000 = **12,000** to historical payments — even though the payment was *after* the report's startDate
- Resulting opening: 38,130 − 12,000 = **26,130** (matched the bug exactly)

The same shared-settlement double-counting also inflated the **current period** "Less Payments" (28,000 shown vs. 22,000 actual = 6,000 + 16,000), so the dedup fix was applied to the current-period loop as well.

**Solution Implemented**:

1. **Historical payment loop (~line 16523)**: Added the same `settlementDate < startDate` gate that Account Statement uses, respecting `adjustmentDateMode` (use `adjustment_date` when posted, `created_at` otherwise). Also added a `Set<payment.id>` so each `payment_settlement_payment` is summed at most once per supplier even if it appears under multiple costs.

2. **Current-period payment loop (~line 16692)**: Added the same `Set<payment.id>` dedup. Date filtering was already correct; only the dedup was missing.

**Files Modified**:
- `psback/controllers/report.controller.js`
  - Historical payment loop (~line 16523): added date filter + dedup Set
  - Current-period payment loop (~line 16692): added dedup Set

**Edge cases handled**:
- One payment settles multiple costs → counted once ✓
- Payment after startDate but cost before startDate → not counted as historical ✓
- `adjustmentDateMode` true with NULL adjustment_date → falls back to `created_at` for the date check ✓
- Payment with NULL `id` (defensive) → still counted, since dedup only triggers on non-null ids ✓

**Note**: The Account Statement opening-balance and current-period payment loops have the **same shared-settlement double-counting vulnerability** — they just hadn't triggered for the suppliers tested so far (no shared settlements in those data sets). If similar inflation appears in Account Statement payments for a supplier with shared settlements, the same dedup `Set` should be applied at:
- `report.controller.js` ~line 10688 (Account Statement opening balance payment loop)
- `report.controller.js` ~line 11229 (Account Statement current period voucher loop)

---

### 10.3 Add Opening XO Column (2026-05-19)

**Goal**: Show imported **opening XOs** as their own **"Add Opening XO"** column, so the Supplier Position Report lines up column-by-column with the Supplier Account Statement.

**What it was before**:
1. The supplier query has **no costing-document filter**, so opening XO costs (`is_opening = 1`, no `documents` row) already flowed into the report.
2. In-period opening XOs were silently bundled inside **"Add Sale Invoices"**.
3. Pre-period opening XOs were already counted inside **"Opening Balance B/F"** via the historical loop.
4. So the Net Balance was already correct — but "Add Sale Invoices" was inflated and disagreed with the Account Statement's figure.

**Solution Implemented**:
1. Reuses `getOpeningCostsForSuppliers(companyCode, supplierIds)` from `psback/services/openingXo.service.js`; opening XOs fetched once and grouped by supplier id.
2. Opening XO costs are **excluded from "Add Sale Invoices"** — `is_opening` costs skipped in the current-period cost loop.
3. Opening XO costs are **excluded from the historical opening-balance loop** — `is_opening` costs skipped (otherwise pre-period XOs would count twice).
4. A per-supplier opening XO total is built with the report date filter (`dayKey`): pre-period XOs fold into Opening Balance B/F, in-period XOs go to the new column. PKR = `total_costing × exchange_rate`.
5. New `opening_xo` ("Add Opening XO") column added to the supplier row, right after "Add Sale Invoices"; included in the `net_balance` formula.
6. Excel: `_xo` added to the amount-column regex so the column is right-aligned and number-formatted. PDF (`report2.ejs`) renders the column automatically (dynamic template — no template edit).

**Result**:
- **Net Balance is unchanged** — opening XOs simply move into their own column instead of hiding inside "Add Sale Invoices" / "Opening Balance B/F".
- "Add Sale Invoices" now matches the Supplier Account Statement.

**Files Modified**:
- `psback/controllers/report.controller.js` (`getSupplierPositionReport`) — opening-XO fetch + grouping, `is_opening` skip in current cost loop and historical loop, per-supplier opening XO total, `opening_xo` column, `net_balance` formula, Excel amount-column regex.

---

**Last Updated**: 2026-05-19 — Section 10.3 (Add Opening XO column, mirroring the Supplier Account Statement)
