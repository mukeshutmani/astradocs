# Daily Sale Report with Refund - Technical Documentation

**Version**: 1.0
**Date**: April 2026
**Author**: System Analysis
**Status**: Stable - Initial Release

---

## Overview

The **Daily Sale Report with Refund** is a fully independent duplicate of the existing Daily Sale Report. It provides the same comprehensive daily view of sales and costs broken down by service type (Air, Visa, Hotel, Insurance, Car Rental, Cruise, Tour, Train, Miscellaneous) **AND** retains the full **Refund Section** (Credit Notes & Debit Notes) below the summary table.

This report is functionally identical to the original Daily Sale Report, including the Refund Section. The plain `Daily Sale Report` will later have its refund section removed, while this report will continue to display refunds. The two reports are completely independent: separate controller, EJS template, route, frontend page, API helper, permission key, and menu entry.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/DailySaleReportWithRefund.jsx`
**Route**: `/reports/dailySaleReportWithRefund`
**Menu Permission**: `Daily-Sale-Report-With-Refund`

**Filters** (identical to Daily Sale Report):
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
  - **default (none)** — shows valid statuses only (Printed, Settled, Partially Settled). Void and Raised are excluded unless explicitly selected.

  The **status column** displays the invoice status (`Partially Settled` displays as `PS`).

**Output**: PDF or Excel (landscape orientation). Excel filename: `Daily_Sale_Report_With_Refund_${report_number}.xlsx`.

**Filter retention & inline preview**:
- Selected filters are retained after generating (module-level `cachedFilter`); returning to the page keeps prior selections. A full page refresh resets to `DEFAULT_FILTER` (with `startDate`/`endDate` recomputed to today on mount).
- PDF is rendered inline below the filters (iframe → `{VITE_API_URL}/document/view/{report_number}`) with a **Preview** button (opens the full PDF in a new tab). No navigation away from the page. Excel still downloads and clears the inline PDF.

### Backend Controller

**File**: `psback/controllers/reports/dailySaleReportWithRefund.report.controller.js`
**Function**: `exports.getDailySaleReportWithRefund`
**Re-exported from**: `psback/controllers/reports/index.js`

### API Endpoint

`POST /api/report/dailySaleReportWithRefund`

Protected by `authenticate` and `permission("Daily-Sale-Report-With-Refund")`.

### PDF Template

**File**: `psback/views/pages/reports/daily-sale-report-with-refund.ejs`
**Orientation**: Landscape (`@page { size: landscape; }`)

### Frontend API Helper

**File**: `psfront/src/api/report.js`
**Function**: `getDailySaleReportWithRefund(data)`

---

## Report Layout

### Main Table Columns

| Section | Columns |
|---------|---------|
| Invoice Info | Invoice No., Inv. Date, PNR, Status, Client Name |
| Receivable from Customer | Air, Visa, Hotel, Ins, Car, Cruise, Tour, Train, Misc, Total Sales |
| Payable to Supplier | XO Number, Supp Name, Air, Visa, Hotel, Ins, Car, Cruise, Tour, Train, Misc, Total Cost, SST |
| Profit/Loss | Sales - Cost - SST |
| Pax | Passenger names (comma-separated) |

**Note:** Hajj and Umrah receivable/payable amounts are folded into the **Misc** column in the main table; they still appear as dedicated rows in the Summary table.

**Row grouping — one row per invoice (with XO-based supplier split):**
Per-service rows are grouped by `invoice_no`. The main sales table now emits:
- **One primary row per invoice** carrying the **full customer-side sales** — every service's amount is placed in its matching column (Air→Air, Hotel→Hotel, Hajj/Umrah folded into Misc), plus the invoice's **Total Sales** and full **Profit/Loss**. All unique pax names from every service are shown on this row.
- **Supplier side** is split by distinct XO:
  - If all services share **one XO** (or no valid XO) → single row only.
  - If **multiple XOs** → additional **secondary rows** are emitted, one per extra XO. Each secondary row shows only that XO's cost breakdown (per service column), XO Number, and Supp Name.
- **On secondary rows**: Invoice No., Inv. Date, PNR, Status, Client Name, all customer-side sales columns, Profit/Loss, and Sales ID are **blank** — they belong to the primary row. The Pax column is left as-is (display-only; CSS/wrap unchanged per user preference).
- Services with **no valid XO** still contribute their sales to the primary row; no supplier row is created for them. Their P/L share is included in the primary row's total Profit/Loss.

Summary table totals are computed from the pre-grouping per-service data, so grand totals are unchanged. **The Refund Section has its own merging rules (CN + DN merged by `refund_id`) and is not affected by this main-table grouping.**

**Main table totals row:** The last row shows column-wise totals.

**Status display:** `Partially Settled` is displayed as **PS**.

### Summary Table

Breakdown by service category with columns: **Total Sales, Total Credit, Net Sales, Total Cost, Total Debit, Net Cost, SST, Profit/Loss**, followed by a grand total row.

- **Total Credit** = sum of CN amounts (from `refundSummary[type].credit`) allocated to that service type.
- **Total Debit** = sum of DN amounts (from `refundSummary[type].debit`) allocated to that service type.
- **Net Sales** = `Total Sales − Total Credit` (per row and at grand total).
- **Net Cost** = `Total Cost − Total Debit` (per row and at grand total).
- **SST** = actual SST accumulated per service type from each invoice's SST.

**Per-category Profit/Loss** formula:

```
Category P/L = (Category Sales − Category Credit) − (Category Cost − Category Debit) − Category SST
```

**Grand total (TOTAL PROFIT/LOSS footer)** uses the same formula at the totals level:

```
Net Profit = (Total Sales − Total Credit) − (Total Cost − Total Debit) − Total SST
```

The calculation breakdown shown above the final value displays the components so users can trace the number.

### Section Order (PDF + Excel)

Both outputs now render sections in this order:
1. **Sales Section** — titled header + main sales table (one row per invoice + service type — see "Row grouping" above)
2. **Refund Section (Credit Notes & Debit Notes)** — titled header + refund table
3. **Summary Section** — titled header + category-wise totals table
4. **TOTAL PROFIT/LOSS** footer

### Refund Section (Credit & Debit Notes) — RETAINED (PDF + Excel)

The refund section has the same columns and TOTAL row in both PDF and Excel outputs.

**Doc columns in the refund section:**

- **First column (receivable side) — "Credit Note No."**: shows the Credit Note reference (e.g., `KHCN00000012`). Blank for standalone Debit Note rows.
- **Doc column (payable side) — "Debit Note No."**: shows the Debit Note reference (e.g., `KHDN00000008`). Blank for standalone Credit Note rows.

**CN + DN merging:** When a Credit Note and a Debit Note share the **same `refund_id`**, they are merged into a single row so the customer-side refund and supplier-side recovery for the same refund appear on one line. Matching by `refund_id` is more reliable than invoice/XO pairing because a refund document is the natural link between a customer credit and a supplier debit.

- **Credit Note No.** column: the CN reference
- **Debit Note No.** column: the DN reference
- **Receivable (Customer) side**: CN amount in the matching service-type column, Total Credit
- **Payable (Supplier) side**: DN amount in the matching service-type column, Total Debit
- **SST**: from the CN
- **Profit/Loss** on the merged row: `Total Debit − Total Credit`
- **Pax / S-ID / Client Name / PNR**: copied from the CN (fall back to DN if missing)

A CN with no matching DN (or a DN with no matching CN) renders as its own standalone row.



A separate table is rendered below the summary listing each Credit Note and Debit Note document attached to the invoices in scope. It mirrors the **full main table column structure** — same columns including SST, Profit/Loss, Pax and S-ID, with these naming differences:

- **"Total Sales" → "Total Credit"** (sum of credit-note billing amount for the row)
- **"Total Cost" → "Total Debit"** (sum of debit-note amount for the row)

**Row rules:**
- One row per document.
- **Credit Notes** populate the receivable side (matching service-type column based on the linked service). SST = `cn.sst_amount`. Pax / S-ID copied from the linked service.
  - Service type is resolved by trying `cn.refund_id → refund.service_id → service` **first** (refund-related CNs are anchored to the refund's actual service, even when the CN is settled against an unrelated invoice for ledger purposes). If the CN has no `refund_id`, it falls back to `cn.settled_invoice_number → invoice.service`. Only CNs with no linkage at all default to **Misc**. This keeps CN allocation consistent with DN allocation (DN already uses `refund.service_id`).
- **Debit Notes** populate the payable side (service type from the linked refund's service, plus Debit Note No. and Supp Name from the active cost). SST = 0. Pax / S-ID copied from the refund's linked service.

**Profit/Loss formula (refund section): `Total Debit − Total Credit`.**
- Credit Note row → `0 − cnAmount` (negative)
- Debit Note row → `dnAmount − 0` (positive)
- Merged row → `dnAmount − cnAmount`

Grand total refund P/L = sum of per-row P/L = `totalDebit − totalCredit`.

The payable side's first column is **Doc Number**:
- **Credit Note rows** display the **invoice number** (`cn.settled_invoice_number`) only.
- **Debit Note rows** display the **XO number** (from the linked refund's service active cost) only.

**Date filtering for the refund section:** CNs and DNs are filtered by their **own `doc_date`** (with `created_at` as fallback when `doc_date` is null), not by the linked invoice's date. This means a refund issued in the report's date range always appears, even if the original invoice is outside that range. Both queries are scoped to the current company via `branch_id` belonging to the company's branches and exclude `Void` documents.

**Other filters applied to refund rows:** After loading, each refund row is checked against the same filter criteria used by the main sales table — **Supplier, Customer (including `between`), Branch, Product Code, Pax Name, PNR, and Sales ID** — using the linked service's data. A refund row is dropped if the linked service doesn't match. For CN rows the linked service comes from `cn.refund_id → refund.service` (or `cn.settled_invoice_number → invoice.service` when no `refund_id`). For DN rows it comes from `dn.refund_id → refund.service`.

**Document Status filter on refund docs:** The same `documentStatus` rules used for invoices are now applied to CN and DN queries via their own `doc_status` column:
- **isEqual** — refund docs must have `doc_status` exactly equal to the chosen status.
- **in** — refund docs whose `doc_status` is in the chosen array.
- **default (none)** — refund docs in `Printed`, `Settled`, or `Partially Settled` (excludes `Raised` and `Void`).

Void refund docs are always excluded.

If a CN's `settled_invoice_number` or a DN's linked refund's `service_id` references an invoice/service that wasn't already loaded for the main table, the controller fetches the missing invoice/service info on demand so the refund row still has correct service type, customer, supplier, XO, pax, and sales ID.

**Service resolution for refund rows** is done via a **service-id keyed lookup** (not the invoice-number keyed lookup). An invoice number can map to multiple invoice rows (multi-service invoices), so keying the refund match by invoice number would drop entries on collision and cause debit notes for (e.g.) Air to land in Misc. Keying by `service.id` avoids this and guarantees each refund row is allocated to the exact service type of its linked service.

A bold **TOTAL** row at the bottom sums each column. Hajj/Umrah amounts are folded into the Misc column the same way as the main table.

### Footer

A one-line calculation breakdown is shown above the final amount, in **both the PDF and the Excel exports**:

```
(Total Sales X - Total Credit Y1) - (Total Cost Y2 - Total Debit Y3) - Total SST (Z) = TOTAL PROFIT/LOSS
```

The **TOTAL PROFIT/LOSS** value is then displayed prominently below the formula.

---

## Calculation Logic (identical to DSR)

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
  + sum(cost_taxes.tax_amount)
  + (commissionAmount * cost.sst% / 100)

totalCost = costPerUnit * cost.quantity
```

### SST

```
transactionFee = invoice.transaction_fee (converted to PKR if foreign currency)
sstPercent = invoice.sst
sstAmount = (transactionFee * sstPercent) / 100
```

### Profit/Loss

```
profitLoss = invoiceTotal - totalCost - sstAmount
```

### Active Cost Selection

When multiple costs exist for a service:
1. Filter out costs with status `Void` or `Raised`
2. Sort by `created_at` descending
3. Use the first (most recent) eligible cost

When no valid cost exists, the entire supplier side is blanked for that row and Profit/Loss equals Total Sales.

---

## Data Filtering

### Invoice Status

Default: `['Printed', 'Settled', 'Partially Settled']` (excludes Void and Raised)

### Date Filtering Logic

Company setting `invoice_setting.ticket_issue_date_in_invoice` controls behavior — same as Daily Sale Report.

### Cost Filtering

- Costs with `status = 'Void'` are excluded.

### Branch & Customer Filtering

- Nested inside the Order include with `required = true` when conditions exist.

---

## Service Type Mapping

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
  report_number: "TPDSR" + timestamp,
  report_type: "daily-sale-report-with-refund",
  file_type: "xlsx" | "pdf"
}
```

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/DailySaleReportWithRefund.jsx` | Frontend UI |
| `psback/controllers/reports/dailySaleReportWithRefund.report.controller.js` | Controller logic |
| `psback/controllers/reports/index.js` | Re-export |
| `psback/views/pages/reports/daily-sale-report-with-refund.ejs` | PDF template (landscape) |
| `psback/routes/report.route.js` | Route registration |
| `psfront/src/api/report.js` | API client (`getDailySaleReportWithRefund`) |
| `psfront/src/App.jsx` | Frontend route registration |
| `psfront/src/pages/Report/Report.jsx` | Menu entry |

---

## Relationship to Daily Sale Report

- This report was created as a clone of the Daily Sale Report on 2026-04-13.
- It is intentionally independent: changes to one will not affect the other.
- The plain Daily Sale Report (`/report/dailySaleReport`) will eventually have its **Refund Section removed**.
- This report (`/report/dailySaleReportWithRefund`) is the long-term home for the refund-inclusive version.

---

**Document End**
