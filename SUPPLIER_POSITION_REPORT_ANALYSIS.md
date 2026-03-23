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
| Add Sale Invoices | cost_invoice | Period cost total |
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

**Status filter**: Only `Paid`, `Partially Paid`, `Printed` (exclude Void, Raised)
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

**Last Updated**: March 2026 — Initial creation, includes frozen exchange rate logic
