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
- Document Status:
  - **isEqual** — single status (Printed, Settled, Partially Settled, Raised, or Void)
  - **in** — multi-select; pick any combination of (Printed, Settled, Partially Settled, Raised, Void). Sent as an array; backend uses `Op.in`.
  - **default (none)** — shows valid statuses only (Printed, Settled, Partially Settled). Void and Raised are excluded unless the user explicitly selects them.

  The **status column** in the report shows whatever status the invoice has (including Void/Raised when the user opted in). `Partially Settled` is displayed as `PS`; all other statuses display as-is.

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
| Payable to Supplier | XO Number, Supp Name, Air, Visa, Hotel, Ins, Car, Cruise, Tour, Train, Misc, Total Cost, SST |
| Profit/Loss | Sales - Cost |
| Pax | Passenger names (comma-separated) |

**Note:** Hajj and Umrah receivable/payable amounts are **folded into the Misc column** in the main table (so Misc shows `misc + hajj + umrah`). Hajj and Umrah are still tracked separately and appear as dedicated rows in the Summary table below.

**Row grouping — one row per (invoice + service type):**
A single logical service can be split across multiple rows in the `invoices`/`services` tables (e.g., a Hajj booking stored as 3 service records — one per passenger group — all sharing the same PNR and invoice number). To avoid showing the same service as multiple rows, the main table merges all rows sharing the same `invoice_no + service_type` into one row:
- **Sales / Cost / SST / Profit-Loss** — summed across merged rows.
- **PNR, XO Number, Supp Name** — if identical across merged rows, shown once; if they differ, unique values are concatenated with `, `.
- **Pax Names** — unique pax names from all merged services are concatenated.
- **Invoice Date, Status, Client Name, Sales ID** — taken from the first row (identical for all rows of the same invoice).

Service types remain distinct during grouping — an invoice with both Hajj and Tour produces **two rows** (one Hajj, one Tour), not one merged Misc row. Summary table totals are computed from the pre-merge per-service data, so grand totals are unchanged by the grouping.

**Main table totals row:** The last row of the main table displays column-wise totals (Air, Visa, Hotel, ..., Misc, Total Sales, ..., Total Cost, SST, Profit/Loss). The Misc total on this row equals `misc + hajj + umrah` to stay consistent with the folded column display. The XO Number and Supp Name columns are empty on the totals row.

**Payable side columns:**
- **XO Number** — `cost.Document.document_number` where `document_type = 'costing'` (the cost order / XO linked to the active cost record)
- **Supp Name** — `service.Supplier.supp_name` (supplier linked to the service)

**Status display:** `Partially Settled` is displayed as **PS** (abbreviated) in both PDF and Excel.

### Summary Table

Breakdown by service category showing Total Sales, Total Cost, **SST**, and Profit/Loss per category, with a grand total row.

**SST per category:** Each invoice row is tied to exactly one service (via `invoices.service_id`), and SST is computed per invoice row as `(transaction_fee × sst%)/100`. That per-row SST is accumulated into the corresponding service type's bucket in `summary[type].sst`. Per-category Profit/Loss is:

```
Category Profit/Loss = Category Sales − Category Cost − Category SST
```

Per-category values sum exactly to the grand total (`Total Sales − Total Cost − Total SST`).

> **Note:** The Refund Section (Credit & Debit Notes) was removed from this report as of 2026-04-14. It now lives only in the separate **Daily Sale Report with Refund** — see `docs/DAILY_SALE_REPORT_WITH_REFUND.md`.


### Footer

A one-line calculation breakdown is shown above the final amount so users can see how the total was derived. This appears in **both the PDF and the Excel exports**:

```
Total Sales (X) - Total Cost (Y) - Total SST (Z) = TOTAL PROFIT/LOSS
```

The **TOTAL PROFIT/LOSS** value is then displayed prominently below the formula.

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
1. Filter out costs with status `Void` or `Raised` (Void = cancelled, Raised = draft/not finalized)
2. Sort by `created_at` descending (most recent first)
3. Use the first (most recent) eligible cost

**When no valid cost exists** (all costs are Void/Raised, or the service has no cost record), the **entire supplier side is blanked** for that row:
- XO Number → empty
- Supp Name → empty
- All payable per-service-type amounts → 0
- Total Cost → 0
- SST → 0
- Profit/Loss → equals Total Sales (since there's no recorded cost)

The row still appears in the report (because the customer-side sale exists) — only the supplier-side columns go blank.

---

## Data Filtering

### Invoice Status

Default: `['Printed', 'Settled', 'Partially Settled']` (excludes Void and Raised)

### Date Filtering Logic

Company setting `invoice_setting.ticket_issue_date_in_invoice` controls behavior:

**Setting ON (invoice-level issue date — applied company-wise):**
- One "Invoice Issue Date" is computed per `document_number` and used for **all** service lines of that invoice:
  1. Latest Air service's `ticket_issue_date` (if the invoice has an Air service), OR
  2. Fallback to `invoice_date` (for non-Air-only invoices).
- The entire invoice is included/excluded based on this single date — no more partial-invoice results where only non-Air rows match due to divergent per-row `invoice_date` values.
- The same issue date is also shown in the "Inv. Date" column for every row of that invoice.

**Setting OFF:**
- Each service line filters on its own `invoice.invoice_date` (per-row behavior, unchanged).

**Why this matters:** A single invoice in the `invoices` table is stored as one row per service, and each row carries its own `invoice_date`. Historically those dates could diverge (e.g., Air row = 01-April, Hotel/Tour/etc. rows = 13-April for the same invoice number). With the setting ON, the report now treats the invoice as a single document with one issue date, matching how the invoice UI displays the issue date.

**Company scoping for the issue date map:** Invoice numbers can be reused across different companies. The `issueDateMap` query is scoped to the current user's `company_code` via `service → user`, so Air rows belonging to another company's invoice with the same number don't contaminate the current company's issue date.

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

Data is tracked per service type internally, but the main table only exposes these columns:

| Service Type | Main Table Column | Summary Table Row |
|-------------|-------------------|-------------------|
| Air | Air | Air |
| Visa | Visa | Visa |
| Hotel | Hotel | Hotel |
| Insurance | Insurance | Insurance |
| Car / Car_Transfer | Car | Car Rental |
| Cruise | Cruise | Cruise |
| Tour | Tour | Tour |
| Train | Train | Train |
| Hajj | **Misc** (folded) | Hajj |
| Umrah | **Misc** (folded) | Umrah |
| Miscellaneous / Other | Misc | Miscellaneous |

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
