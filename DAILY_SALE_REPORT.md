# Daily Sale Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Verified

---

## Overview

The Daily Sale Report provides a comprehensive daily view of sales and costs broken down by service type (Air, Visa, Hotel, Insurance, Car Rental, Cruise, Tour, Train, Miscellaneous). Each row represents one invoice, showing receivable (sales) and payable (cost) amounts categorized by service type, along with profit/loss and passenger names.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/DailySaleReport.jsx`

**Filters**:
- Date Range: Invoice date or ticket issue date for Air (=, <, <=, >, >=, <>, between)
- Branch: isNotBlank, isBlank, isEqual
- Customer: isNotBlank, isBlank, isEqual, between
- Supplier: isNotBlank, isBlank, isEqual
- Product Code: isNotBlank, isBlank, isEqual
- Pax Name: Text search (LIKE)
- PNR: isNotBlank, isBlank, isEqual, contains
- Document Status: isEqual (Printed, Settled, Partially Settled) or default all valid

**Output**: PDF or Excel (landscape orientation)

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getDailySaleReport` (line ~18647)

### PDF Template

**File**: `psback/views/pages/reports/daily-sale-report.ejs`
**Orientation**: Landscape (`@page { size: landscape; }`)

---

## Report Layout

### Main Table Columns

| Section | Columns |
|---------|---------|
| Invoice Info | Invoice No., Inv. Date, PNR, Status, Client Name |
| Receivable from Customer | Air, Visa, Hotel, Ins, Car, Cruise, Tour, Train, Misc, Total Sales |
| Payable to Supplier | Air, Visa, Hotel, Ins, Car, Cruise, Tour, Train, Misc, Total Cost, SST |
| Profit/Loss | Sales - Cost |
| Pax | Passenger names (comma-separated) |

### Summary Table

Breakdown by service category showing Total Sales, Total Cost, and Profit/Loss per category, with a grand total row.

### Footer

**TOTAL PROFIT/LOSS** displayed prominently.

---

## Calculation Logic

### Invoice Total (Sales)

```
invoiceTotal = invoice.total_price
// total_price already includes SST
// Currency conversion applied if non-PKR
```

### Cost Calculation

```
commissionAmount = cost.published_rate * (cost.commission% / 100)
netRate = cost.published_rate - commissionAmount
netRateWithExtra = netRate + cost.free_of_cost

costPerUnit = netRateWithExtra
  + sum(cost_taxes.tax_amount)        // All cost taxes
  + (commissionAmount * cost.sst% / 100)  // WHT on commission

totalCost = costPerUnit * cost.quantity
```

**Note**: The `quantity` field on the cost record determines the multiplier (typically matches passenger count for Air services).

### SST (Service Sales Tax)

```
transactionFee = invoice.transaction_fee (converted to PKR if foreign currency)
sstPercent = invoice.sst
sstAmount = (transactionFee * sstPercent) / 100
```

Displayed in the Payable to Supplier section after Total Cost. The SST value is for display only — `invoice.total_price` already includes SST.

### Profit/Loss

```
profitLoss = invoiceTotal - totalCost - sstAmount
```

### Active Cost Selection

When multiple costs exist for a service:
1. Filter out Void costs
2. Sort by `created_at` descending (most recent first)
3. Use the first (most recent) non-Void cost

---

## Data Filtering

### Invoice Status

Default: `['Printed', 'Settled', 'Partially Settled']` (excludes Void and Raised)

### Date Filtering Logic

- **Air services with ticket_issue_date setting enabled**: Uses `service.ticket_issue_date`
- **All other services**: Uses `invoice.invoice_date`
- Setting checked via `invoice_setting.ticket_issue_date_in_invoice` per company

### Cost Filtering

- Costs with `status = 'Void'` are excluded from the query

### Branch & Customer Filtering

- Branch and Customer filters are nested inside the Order include
- Order `required` is set to `true` when ANY of its nested filters have conditions (branch, customer, or salesId)
- This ensures that branch/customer filtering properly excludes non-matching invoices

---

## Credit Notes and Refunds

The report also fetches:
- **Credit Notes**: Matched by `settled_invoice_number`, excludes Void
- **Debit Notes**: Linked through refunds via `refund_id`, excludes Void
- **Refunds**: Matched by `service_id`, excludes Void

These are used for adjusted calculations but the main Profit/Loss column uses simple `invoiceTotal - totalCost`.

---

## Currency Conversion

- Invoices in foreign currency (not PKR/110) are converted to PKR using the `exchange_rate` stored on the invoice at document creation time
- Cost conversion uses the `exchange_rate` stored on the cost record
- This ensures the report reflects the exact rate used when each document was generated, not the current rate from the exchange currencies table
- Transaction fee is also converted for foreign currency invoices

## Number Formatting

- All monetary values are displayed with exactly 2 decimal places (e.g., 245.79, not 246)
- PDF: Uses `fmt()` helper — `parseFloat(val).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })`
- Excel: Data rows use `#,##0.00` number format; summary table also uses `#,##0.00`

---

## Service Type Mapping

| Service Type | Receivable Column | Payable Column |
|-------------|-------------------|----------------|
| Air | Air | Air |
| Visa | Visa | Visa |
| Hotel | Hotel | Hotel |
| Insurance | Insurance | Insurance |
| Car / Car_Transfer | Car | Car |
| Cruise | Cruise | Cruise |
| Tour | Tour | Tour |
| Train | Train | Train |
| Miscellaneous / Other | Misc | Misc |

---

## Company Scoping

Scoped via `service → user → where: { company_code: req.user.company_code }`.

---

## Report Metadata

```javascript
{
  report_number: "TPDS" + timestamp,
  report_type: "daily-sale-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Verification Results (March 2026)

Verified with 5 invoices across 2 customers (RM0011, QKCOMP), covering Air and Hotel service types. All sales, cost, and profit/loss values matched database calculations exactly. Summary table totals verified correct. No bugs found.

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/DailySaleReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` | Controller logic (~line 18647) |
| `psback/views/pages/reports/daily-sale-report.ejs` | PDF template (landscape) |
| `psfront/src/api/report.js` | API client |

---

**Document End**
