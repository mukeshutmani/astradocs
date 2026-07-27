# DSR 01 Report

**Status**: Stable
**Date**: July 2026

---

## Overview

**DSR 01** is an independent report derived from the **Daily Sale Report**. It shares the
same filters, business logic and calculations, but has its **own 22-column sales-focused
layout** on **A4 landscape**. It lives in a separate controller, template, frontend page,
route and permission so it can evolve **without affecting the Daily Sale Report**.

**DSR 01 is released to company `1007` (5th Pillars) only.** Users of any other company
never see the menu item and cannot call the API.

> For the shared functional/business-logic details (active cost selection, XO status rules,
> date-filter logic, currency conversion, summary table) see
> **`docs/DAILY_SALE_REPORT.md`**. Everything below documents where DSR 01 diverges.

**Not to be confused with** the separate **DSR** report (`docs/DSR_REPORT.md`), which is a
different condensed replica.

---

## What is different from Daily Sale Report

| Aspect | Daily Sale Report | DSR 01 |
|--------|-------------------|--------|
| On-screen / PDF title | `Daily Sale Report` | `DSR 01` |
| Report number prefix | `TPDS` + timestamp | `DSR01` + timestamp |
| `report_type` | `daily-sale-report` | `dsr-01-report` |
| PDF template | `daily-sale-report.ejs` | `dsr-01-report.ejs` |
| PDF renderer | shared `wkhtmltopdf` service | shared `wkhtmltopdf` service |
| PDF sizing | `zoom 0.75`, fixed 100% width | auto-fit: canvas sized per report to the data's widest cells (min `1250px`), `zoom = 1098 / canvas` → always fills the 291mm printable width, never clips |
| Main table font | cells `8px`, body `9px` | **cells `10px`, body `11px`** |
| Main table | 28 columns | **22 columns** (see below) |
| Excel sheet name | `Daily Sale Report` | `DSR 01` |
| Excel download name | `Daily_Sale_Report_<no>.xlsx` | `DSR_01_<no>.xlsx` |
| Permission | `Daily-Sale-Report` | `DSR-01-Report` |
| Backend route | `POST /api/report/dailySaleReport` | `POST /api/report/dsr01` |
| Company availability | All companies | **`1007` only** |
| Service types | All | **Cruise, Train, Car Rental excluded** |

Filters, row grouping, the per-XO supplier split, currency conversion and the summary table's
columns are unchanged.

**Removed (2026-07-10):** the bottom footer block — the
`Total Sales (X) - Total Cost (Y) - Total SST (Z) = TOTAL PROFIT/LOSS` formula line and the
standalone **TOTAL PROFIT/LOSS** figure — is gone from **both** the PDF (`.total-gross` div
and its CSS) and the Excel builder. The grand profit/loss is still visible in the summary
table's own **Total** row.

**Grid lines (2026-07-10):** table borders are `1px solid #999` (grey hairline), not
`#000`. At `zoom: 1.0` a solid-black 1px border renders too heavy. The double underline
under the summary is unaffected.

**Font (2026-07-11):** the PDF prints in **Nunito Sans**. The regular and bold TTF files
live in `psback/fonts/` (they deploy with the repo — nothing to install on the server) and
are loaded via `@font-face` with a runtime `file:///` URL (`fontBase` render local computed
with `pathToFileURL`). If the files are missing the report falls back to Arial. Other
reports are unaffected.

**Removed (2026-07-11): the Edit Report layout editor.** For one day DSR 01's main-table
layout was user-editable (System Tables → Edit Report, `report_layouts` table, permission
`Edit-Report-Layout`, and briefly a Playwright/Chromium PDF renderer). The feature was
removed the same day: end users should not hand-tune column widths, and the future
customization direction is QuickBooks-style (choose columns, rename headings) — to be built
later on the column registry that remains. One-time DB cleanup after the removal:

```sql
DELETE FROM permission_groups WHERE permission_id = 183;  -- Edit-Report-Layout grant
DELETE FROM permissions WHERE id = 183;                   -- Edit-Report-Layout
DROP TABLE report_layouts;
```

**Leftovers fixed (2026-07-13) — amounts must never truncate:**

1. The geometry work had locked the **summary table** to a fixed `90mm` width, truncating
   summary amounts to `10,736,354...`. Reverted to the Daily Sale Report's auto-sized
   `width: 100%`.
2. The **main table** did the same via `table-layout: fixed` + per-column percent widths +
   global `text-overflow: ellipsis`. Now it works like the Daily Sale Report: columns
   auto-size to content, so amount and date cells always show in full; only the text-heavy
   columns (Client Name, Invoice Description, Supp Name, Pax via `max-width` caps) ellipsize.
   Invoice No. and XO Number stay on **one line** (`.wrap-narrow` is nowrap since 2026-07-13,
   despite its inherited name). PNR cells (`max-width: 72px`) fit one full 10-digit ticket per
   line; Ticket Number has its own `.ticket-cell` (`max-width: 92px`, 2026-07-20) so one full
   **prefixed** ticket (`214-5518213255`) fits per line; multi-ticket invoices wrap at the
   commas, never inside a number.
3. **Data-driven page fit (2026-07-14).** A CSS table's `width` is a *minimum* — when the
   data's widest cells need more, the table silently outgrows it, and at a fixed zoom the
   extra pixels fall off the paper (that clipped the Pax column on 07-14). So the controller
   now estimates the widest cell of every column (generous per-glyph widths — digits/letters
   6.0px, thin punctuation 3.4px, +6px per cell, +3% slack — so estimation error only ever
   makes the print slightly smaller, never clips), sizes the canvas to
   `max(1250px, estimate)`, renders the table at that width, and sets
   `zoom = 1098 / canvas` so the canvas maps exactly onto the 291mm printable width.
   Narrow reports keep the largest font (`1250px` floor ≈ zoom 0.88); wide reports shrink
   just enough to fit. Smart shrinking stays **disabled** — it overshoots its canvas in 25%
   steps, which rendered the table at ~60% scale and left a right-side gap (diagnosed
   2026-07-13 from the PDF's transform matrix). `min-width` floors on the wrap columns stop
   them from collapsing to one character per line (which is what happens at `width: 100%`).
4. The column registry no longer carries widths — sizing is fully CSS/content-driven.

---

## Main-table column layout (DSR 01-specific)

The main table is rendered from the column registry
`psback/services/reportLayout/dsr01.columns.js` — the single source of truth for column
order, headings, widths (fixed % of table width) and value/total extractors. Changing the
layout means editing that one file; the EJS template just loops over it.

22 columns, in this order:

| # | Column | Source |
|---|--------|--------|
| 1 | Invoice No. | `documents.document_number` |
| 2 | Inv. Date | invoice issue date (Air `ticket_issue_date` rule applies) |
| 3 | PNR | `services.pnr`, de-duplicated per invoice |
| 4 | Ticket Number | `service_passengers.ticket_number`, de-duplicated per invoice. **Air tickets carry the airline's 3-digit code** (2026-07-20), e.g. `214-5518213255` — same source as the invoice document: first flight leg's `airline_codes.airline_ticket_prefix`, fallback to the service's own airline (`services.airline_form`). Non-Air services, services with no airline, and tickets already containing a `-` stay bare. Applied when the row is built, so **PDF and Excel both** show it |
| 5 | Status | `invoices.status` (`Partially Settled` → `PS`) |
| 6 | Client Name | `customer.customer_name` |
| 7 | Invoice Description | `services.description`, de-duplicated per invoice |
| 8 | Air | receivable |
| 9 | Visa | receivable |
| 10 | Hotel | receivable |
| 11 | **Transport** | receivable — **Car Transfer only** |
| 12 | Insurance | receivable |
| 13 | **Others** | receivable — **Tour + Miscellaneous + Hajj + Umrah** |
| 14 | Supp Name | `service.Supplier.supp_name` |
| 15 | XO Number | cost's `costing` document number |
| 16 | Basic Fare | `invoices.price × invoices.quantity` |
| 17 | Taxes | `SUM(invoice_taxes.tax_amount) × invoices.quantity` |
| 18 | Trans Fees | `invoices.transaction_fee` (NULL → 0) |
| 19 | SST | `(transaction_fee × invoices.sst%) / 100` |
| 20 | **Total Cost** | the row's XO cost (`payable.total`) — added 2026-07-13 |
| 21 | Total Sales | `invoices.total_price` |
| 22 | Pax | passenger names |

**Optional Reference column (2026-07-21):** a **Reference** checkbox on the filter page
(`showReference`, default OFF) adds a **Reference** column right after **Invoice
Description** (PDF + Excel), showing the order's **booking-level required-field value(s)** —
the Required Fields dialog on the order page. Same pattern as the Customer Account
Statement / invoice document: `required_field_values` (`level='booking'`, `data_id` = order
id) + `customer_required_data` field names, filtered to fields the customer marked
**Print** (booking level) in `customer_required_data_settings`, sorted by the customer's
`display_order`, values comma-joined. Implementation: the registry carries the column with
`optional: true` and `resolveColumns({ showReference })` filters it, so the PDF template,
group headers, TOTAL span and the auto-fit width estimator (cap `92px`, class
`client-cell`) all follow automatically; rows carry `order_id` / `customer_number` and the
controller stamps `reference` after the per-invoice grouping. Excel inserts the
header/cell/TOTAL-blank conditionally and widens the title merges `U → V`. Secondary
(multi-XO) rows show it blank (`blankOnSecondary`). Checkbox OFF → report unchanged, no
extra query.

**Total Cost (added 2026-07-13):** per-XO cost, so it stays filled on secondary
(additional-XO) rows like Supp Name / XO Number, and shows `0.00` when an invoice has no
valid XO. Its TOTAL-row figure equals the summary table's **Total Cost** exactly. PDF only —
the Excel export columns are unchanged.

**Removed** versus Daily Sale Report: all payable per-service columns, XO Status,
Profit/Loss and S-ID. (The Sales ID *filter* still works — only the output column is gone.)
Profit is still computed internally; it feeds the summary table.

### The four money columns always sum to Total Sales

```
Basic Fare + Taxes + Transaction Fees + SST = Total Sales
```

Two details make this hold:

1. **`invoice_taxes.tax_amount` is stored PER UNIT.** Both Basic Fare and Taxes are therefore
   multiplied by `invoices.quantity`. Treating tax as a flat per-invoice figure breaks every
   multi-passenger invoice (verified on `KHIN00000025`, qty 2).
2. **The SST column uses the invoice's own SST**, not the XO-gated `sstAmountForRow` that the
   Daily Sale Report shows on its supplier side. `sstAmountForRow` is zeroed whenever an
   invoice has no valid XO, which would break the identity on those rows. The summary table
   still uses the XO-gated value, so the summary is unchanged.

Accumulated into `summary.totalBasicFare`, `totalTaxes`, `totalTransactionFee` and
`totalInvoiceSst` for the main table's TOTAL row. Note `totalInvoiceSst` (main table) and
`totalSst` (summary table) are deliberately different numbers.

Verified 2026-07-11 against company 1007's full invoice history: 33 report rows, 0
mismatches, TOTAL row reconciles to `10,736,354.83` on both the row-sum and the column-sum.

---

## Excluded service types

`Cruise`, `Train` and `Car` (Car Rental) are filtered **out of the query itself** via
`EXCLUDED_SERVICE_TYPES` in `dsr01.report.controller.js`:

```js
serviceTypeWhere.type = { [Sequelize.Op.notIn]: EXCLUDED_SERVICE_TYPES };
```

They are excluded rather than merely hidden. If they were only hidden, their sales would still
land in `invoices.total_price` (and so in Total Sales and the summary grand total) with no
column to display them in, and the category columns would silently stop adding up.

`Car_Transfer` is **kept** and reported under **Transport**.

> **Side effect:** the Product Code filter dropdown on `DSR01.jsx` is populated from all
> `service_types`, so it still lists Cruise / Train / Car Rental codes. Selecting one returns
> an empty report. Filtering that dropdown is not currently implemented.

---

## Summary table (DSR 01-specific)

Same columns as the Daily Sale Report (Total Sales, Total Cost, SST, Profit/Loss). The
footer block below it has been removed. Category rows are:

`Air, Visa, Hotel, Insurance, Transport, Tour, Hajj, Umrah, Miscellaneous`

- The `Car Rental` row was **renamed to `Transport`** and now holds Car Transfer only
  (`summary.carRental` → `summary.transport`).
- The `Cruise` and `Train` rows were **removed** along with their summary buckets.

Per-category Profit/Loss remains `Category Sales − Category Cost − Category SST`.

---

## Company restriction (1007 only)

The `permissions` table is a **global catalogue** — a permission row is not owned by a
company, and `permission.controller.js` lists every row on every company's group-permission
screen. So the permission alone would not keep DSR 01 inside company 1007: another
company's admin could tick `DSR-01-Report` on their own group.

The restriction is therefore enforced in **two** places, and the hard company check is what
actually guarantees it:

1. **Backend (authoritative)** — `psback/controllers/reports/dsr01.report.controller.js`
   returns **403** when `req.user.company_code !== '1007'`, before any query runs:

   ```js
   const DSR01_COMPANY_CODE = '1007';

   if (req?.user?.company_code !== DSR01_COMPANY_CODE) {
       return res.status(403).json({
           status: 403,
           message: 'Forbidden: DSR 01 is not available for this company'
       });
   }
   ```

2. **Frontend (cosmetic)** — `psfront/src/pages/Report/Report.jsx` only renders the
   **DSR 01** menu item when the user holds `DSR-01-Report` **and**
   `user.company_code === '1007'`.

> The `/report/dsr01` React route is not itself guarded (no report route in this app is).
> A non-1007 user who types the URL sees the filter page, but generating a report fails with
> the backend 403. This matches how every other report behaves.

**To release DSR 01 to another company later**, change `DSR01_COMPANY_CODE` in the
controller and the `'1007'` literal in `Report.jsx`, then grant the permission to that
company's group.

---

## Files

### New files (the replica)
| File | Purpose |
|------|---------|
| `psback/controllers/reports/dsr01.report.controller.js` | Backend controller `getDSR01Report` (copy of `getDailySaleReport` + company guard) |
| `psback/views/pages/reports/dsr-01-report.ejs` | PDF template (A4 landscape, renders columns from the registry) |
| `psback/services/reportLayout/dsr01.columns.js` | Column registry (order, headings, fixed widths, values, totals) |
| `psback/fonts/NunitoSans-Regular.ttf`, `NunitoSans-Bold.ttf` | Report font, embedded via `@font-face` |
| `psfront/src/pages/Report/DSR01.jsx` | Frontend filter page (copy of `DailySaleReport.jsx`) |
| `psback/migrations/add_dsr_01_report_permission.sql` | Creates the permission and grants it to company 1007 |

### Existing files — additive wiring only (Daily Sale Report code untouched)
| File | Change |
|------|--------|
| `psback/controllers/reports/index.js` | require + export `getDSR01Report` |
| `psback/routes/report.route.js` | import `getDSR01Report`; add `POST /dsr01` (permission `DSR-01-Report`) |
| `psfront/src/api/report.js` | add `getDSR01Report()` API client + export |
| `psfront/src/App.jsx` | import `DSR01`; add `<Route path="dsr01">` |
| `psfront/src/pages/Report/Report.jsx` | add **DSR 01** menu item under History (gated by permission **and** company 1007) |

---

## Permission setup (one-time)

Run `psback/migrations/add_dsr_01_report_permission.sql` (idempotent):

1. Creates the global permission row `DSR-01-Report` (`model_name = 'Report'`).
2. Grants it to every user group of company `1007` that already holds `Daily-Sale-Report`.

At the time of writing that resolves to exactly one group — `Admin-Agt` (id 28) — which
covers all 5 users of company 1007. Company 1007's empty `Finance` group is deliberately
left untouched; grant it through the normal permissions screen if needed.

Users must **log out and log back in** for the new permission to appear, because
`permissions` is baked into the JWT/session at login
(`auth.controller.js`: `user.user_group.permission_groups.flatMap(...)`).

---

## Report Metadata

```javascript
{
  report_number: "DSR01" + timestamp,
  report_type: "dsr-01-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/DSR01.jsx` | Frontend UI |
| `psback/controllers/reports/dsr01.report.controller.js` | Controller logic |
| `psback/views/pages/reports/dsr-01-report.ejs` | PDF template (landscape) |
| `psback/services/reportLayout/dsr01.columns.js` | Column registry |
| `psfront/src/api/report.js` | API client (`getDSR01Report`) |

---

**Document End**
