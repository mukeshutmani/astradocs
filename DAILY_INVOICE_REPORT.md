# Daily Invoice Report

Lists all invoices raised within a selected date range with their sales, cost, profit/loss, and margin. Produces a PDF or Excel export scoped to the user's company.

## Location

| Layer | Path |
|---|---|
| Route | `psback/routes/report.route.js` → `POST /api/report/dailyInvoiceReport` |
| Controller | `psback/controllers/reports/dailyInvoiceReport.report.controller.js` |
| Re-exported via | `psback/controllers/reports/index.js` |
| EJS template (PDF) | `psback/views/pages/reports/daily-invoice-report.ejs` |
| Frontend page | `psfront/src/pages/Report/DailyInvoiceReport.jsx` |
| API helper | `psfront/src/api/report.js` → `getDailyInvoiceReport` |
| Permission | `Daily-Invoice-Report` |

## Filters

All filters are sent in the request body.

| Field | Type | Description |
|---|---|---|
| `invoiceDateFilter` | `"="`, `"<"`, `"<="`, `">"`, `">="`, `"<>"`, `"between"`, `"blank"` | Operator for `invoice.invoice_date` |
| `invoiceStartDate` | ISO date | Used with all date ops (day-boundary for equality) |
| `invoiceEndDate` | ISO date | Used only with `between` |
| `branchFilter` | `"isNotBlank"`, `"isBlank"`, `"isEqual"` | Applied to `service.order.branch_id` |
| `branch_id` | number | Required when `branchFilter = isEqual` |
| `customerFilter` | `"isNotBlank"`, `"isBlank"`, `"isEqual"`, `"between"` | Applied to `service.order.customer_id` (and `customer.id` include) |
| `customer_id` | number | Required for `isEqual` |
| `customer_idStart`, `customer_idEnd` | number | Required for `between` |
| `tcidFilter` | `"isNotBlank"`, `"isBlank"`, `"isEqual"` | Applied to `service.order.tcid_id` |
| `tcid_id` | number | Required when `tcidFilter = isEqual` |
| `documentStatusFilter` | `""` (default) \| `"isEqual"` \| `"in"` | Controls `invoice.status` filtering |
| `document_status` | string \| string[] | Single status for `isEqual`, array for `in`. Values: `Printed`, `Settled`, `Partially Settled`, `Raised`, `Void` |
| `type` | `"pdf"` \| `"excel"` | Output format (defaults to `pdf`) |

### Status filter behavior

- `documentStatusFilter` blank → default `status IN ('Printed','Settled','Partially Settled')` (Void and Raised excluded).
- `documentStatusFilter = 'isEqual'` with `document_status = 'X'` → `status = X`.
- `documentStatusFilter = 'in'` with `document_status = [...]` → `status IN (...)`.
- If `documentStatusFilter = 'in'` but the array is empty, the default list is used.

### Implicit filters
- Invoice must have a matching `documents` row where `document_type = 'invoice'`.
- Company scope: joined via `service → user → company` where `company.code = req.user.company_code`. This prevents cross-company leakage when the same invoice number exists in different companies.

## Data model / joins

```
invoice
  ├─ document (document_type = 'invoice', required)
  ├─ currency_code
  ├─ service (required)
  │    ├─ user → company (required; company.code = req.user.company_code)
  │    ├─ order
  │    │    ├─ customer (customerWhere applied here)
  │    │    ├─ branch → company
  │    │    └─ tcid
  │    ├─ service_type
  │    └─ cost
  ├─ invoice_tax
  └─ invoice_discount
```

## Columns

Each row in the main table comes from one invoice record.

| Column | Source / formula |
|---|---|
| Company | `service.order.branch.company.name` |
| Invoice Number | `invoice.invoice_number` or `document.document_number` |
| Status | Full status text (`Printed`, `Settled`, `Raised`, `Void`). `Partially Settled` is shortened to `PS` to fit the column. |
| Date | Effective invoice date — see *Ticket issue date* below. Formatted `DD MMM YYYY`. |
| Client Name | `service.order.customer.customer_name` |
| Currency | `currency_code.currency_code` |
| Gross | `invoice.total_price × invoice.exchange_rate` (`total_price` already includes SST and line quantity) |
| Discount | `Σ invoice_discount.discount_amount × invoice.exchange_rate` |
| Sales | `Gross − Discount` |
| Cost | See formula below |
| SST | Sales-side SST charged on the invoice, in PKR. `invoice.transaction_fee × invoice.sst / 100 × invoice.exchange_rate`. Same formula used by `invoice.controller.js` when computing `total_price`. Shown even when Cost is `N/A`. |
| Profit/Loss | `Sales − Cost − SST` (blank when Cost is `N/A`) |
| Margin | `(Profit/Loss ÷ Sales) × 100 %` based on the SST-adjusted Profit/Loss (blank when no Cost or Sales = 0) |
| TCID | `service.order.tcid.name` |

### Cost formula

When `service.cost` exists:

```
cost_per_unit = published_rate
              − (published_rate × commission / 100)
              + free_of_cost                                    ← extra charges
              + Σ cost_taxes.tax_amount
              + (published_rate × commission / 100) × (sst / 100)

Cost (PKR) = cost_per_unit × cost.quantity × cost.exchange_rate
```

This matches the stored `costs.total_costing` value used elsewhere in the system (and in `dailySaleReportWithRefund`).

- `free_of_cost` is the **extra charges** field (despite the legacy column name) — added after commission is subtracted.
- `cost_taxes` is loaded via a nested include on the `cost` association; without that include the tax sum silently becomes 0 and Cost is understated.

If `service.cost` is null → `Cost = "N/A"`, `Profit/Loss = null`, `Margin = null`.

## Ticket issue date (company-scoped)

When the company's `invoice_settings.ticket_issue_date_in_invoice = 1`, the report treats the invoice's **effective date** as the latest `service.ticket_issue_date` of any Air service belonging to that invoice `document_number`. All rows of the same invoice share that one date (no per-row fallback), which matches what the customer sees on the printed invoice.

- An `issueDateMap` is built after the fetch — `document_number → latest Air ticket_issue_date`.
- If an invoice has no Air service, every row falls back to `invoice.invoice_date`.
- The **Date column** displays the mapped date.
- The **Date filter** runs in JavaScript against the mapped date; SQL `invoice_date` filter is skipped when the setting is on.
- Date comparisons use Asia/Karachi (PKT) via `moment-timezone`, so UTC-stored dates are compared day-correctly against user-entered PKT dates.
- Companies with the setting disabled keep using `invoice.invoice_date` at the SQL level (unchanged).

## Row order

After filtering, rows are sorted:
1. By effective date ascending (groups rows by date).
2. By `document_number` (keeps all rows of one invoice adjacent).
3. By `invoice.id` (stable order inside each invoice).

This guarantees all rows of KHIN00000096 (or any multi-service invoice) appear together, even when the underlying invoice IDs were created at different times.

## Cost row selection

The `service.cost` include is scoped to `status != 'Void'`. If a service has both a Void and an active cost row, the active one is used. If only Void costs exist, `Cost` is shown as `N/A`.

## Currency handling

All amount columns (Gross, Discount, Sales, Cost, Profit/Loss) are **always shown in PKR**. For foreign-currency invoices the raw amount is multiplied by `invoice.exchange_rate` before display. The Currency column still shows the original invoice currency code so the source is visible.

- **Margin column**: a percentage; not converted.
- **PKR-native invoices** (`currency_code = PKR` or `exchange_rate = 1`): amounts pass through unchanged.
- Excel and PDF render the same PKR values — no two-line cells.

## Totals row

The footer row sums the PKR-converted numeric columns (`gross`, `discount`, `sales`, `cost`, `sst`, `profile_or_loss`) across all fetched invoices. The `currency` cell is labelled `"Total (PKR)"`; margin is left blank (it is not a simple sum).

## Output

### PDF

Rendered through a dedicated template `views/pages/reports/daily-invoice-report.ejs` and converted with `createPdf(html, true)` — **A4 Landscape**. Text columns (Company, Invoice Number, Status, Date, Client Name, Currency, TCID) are **left-aligned**; amount columns (Gross, Discount, Sales, Cost, SST, Profit/Loss, Margin) are right-aligned. Uploaded to MinIO under `TPDI<timestamp>.pdf`.

### Excel

Built with ExcelJS, worksheet name `"Daily Invoice Report"`. Layout:

1. Company name (row 1, bold 24pt, centered)
2. Company address
3. Report title
4. Report ID (`TPDI<timestamp>`)
5. Printed By
6. Print Date/Time
7. Blank row (frozen split at row 7)
8. Column headers (grey fill, bold) — alignment matches PDF: text columns left, numeric columns (incl. Margin) right.
9. Data rows — same alignment rules as the header. Numeric columns (Gross, Discount, Sales, Cost, SST, Profit/Loss) get thousands separators and two decimals. Margin keeps its `%` string. Empty cells render as empty (not `0.00`).
10. Total row (bold, light-grey fill) — alignment and formatting match the data rows; Margin and unused text cells stay blank.

Column widths auto-fit between 10 and 50 characters.

## Report record

Every request creates a row in `report`:

| Field | Value |
|---|---|
| `user_id` | `req.user.id` |
| `file_type` | `xlsx` when `type=excel`, else `pdf` |
| `report_number` | `TPDI<epoch_ms>` |
| `report_type` | `"daily-invoice-report"` |

The frontend history panel queries this with `getReports({ type: "daily-invoice-report" })`.

## Response shape

```json
{
  "status": 200,
  "message": "success",
  "link": "<minio object key>",
  "downloadLink": "<signed URL, excel only>",
  "report": { "id": 123, "report_number": "TPDI1712345678", ... }
}
```

## Notes

- The controller lives in its own module under `controllers/reports/` and is re-exported via `controllers/reports/index.js`. It is no longer defined in `report.controller.js`.
- The transaction (`db.sequelize.transaction`) currently wraps only the SELECT and the `report` insert; the PDF/Excel generation and MinIO upload happen inside the transaction callback but do not need transactional isolation.
- The EJS template `report1.ejs` is shared with other simple tabular reports — changes there affect more than this report.
