# DSR Report

**Status**: Stable
**Date**: June 2026

---

## Overview

**DSR** is an independent, full replica of the **Daily Sale Report**. It has the exact
same filters, business logic, calculations, PDF layout, and Excel export. It was created
as a separate report (separate controller, template, frontend page, route, and permission)
so it can evolve on its own **without affecting the Daily Sale Report**.

> For all functional/business-logic details (calculation logic, active cost selection,
> XO status rules, date-filter logic, currency conversion, service-type mapping, summary
> table, footer, etc.) see **`docs/DAILY_SALE_REPORT.md`** — DSR behaves identically.

---

## What is different from Daily Sale Report

DSR is a byte-faithful copy of the `getDailySaleReport` logic with only these differences:

| Aspect | Daily Sale Report | DSR |
|--------|-------------------|-----|
| On-screen / PDF title | `Daily Sale Report` | `DSR` |
| Report number prefix | `TPDS` + timestamp | `DSRR` + timestamp |
| `report_type` | `daily-sale-report` | `dsr-report` |
| PDF template | `daily-sale-report.ejs` | `dsr-report.ejs` |
| Permission | `Daily-Sale-Report` | `DSR-Report` |
| Backend route | `POST /api/report/dailySaleReport` | `POST /api/report/dsr` |

Everything else (filters, grouping, summary, profit/loss, currency conversion,
Excel format, landscape PDF) is identical — **except the main-table column layout**,
which is intentionally simplified for DSR (see next section).

---

## Main-table column layout (DSR-specific)

DSR's main table is a condensed version of the Daily Sale Report's. The underlying
calculations are unchanged — only which columns are shown and how some are merged.

**Hidden in DSR** (present in Daily Sale Report, removed here):
- **XO Number** and the XO **Status** column (supplier side)
- **SST** column (supplier side)
- **S-ID** (Sales ID) column

> Note: the **Sales ID filter** still works — only the S-ID *output column* is removed.
> XO number / status and per-row SST are still computed internally (and still drive the
> supplier per-XO row split and the profit/loss math); they are just not displayed.

**Merged columns** (applied on BOTH the Receivable/customer side and the Payable/supplier side):
- **Transport** = Car + Cruise + Train
- **Others** = Insurance + Tour + Misc (Misc already folds in Hajj + Umrah)

**Resulting DSR main-table columns:**

| Section | Columns |
|---------|---------|
| Invoice Info | Invoice No., Inv. Date, PNR, Status, Client Name |
| Receivable from Customer | Air, Visa, Hotel, **Transport**, **Others**, Total Sales |
| Payable to Supplier | Supp Name, Air, Visa, Hotel, **Transport**, **Others**, Total Cost |
| Profit/Loss | Sales − Cost |
| Pax | Passenger names |

The **TOTAL** row and the **Excel** export mirror this exact layout.

**Summary breakdown table (DSR-specific):** the bottom table lists each individual
service category (Air, Visa, Hotel, Insurance, Car Rental, Cruise, Tour, Train, Hajj,
Umrah, Miscellaneous) with **Total Sales, Total Cost, and Profit/Loss** columns. The
**SST column has been removed** from this table (2026-06-09). Profit/Loss is still
calculated as `Sales − Cost − SST` internally, so the hidden SST is the reason
`Sales − Cost` does not visually equal Profit/Loss. The grand Profit/Loss appears in
this table's own **Total** row.

The bottom block (the `Total Sales − Total Cost − Total SST = TOTAL PROFIT/LOSS`
formula line and the standalone **TOTAL PROFIT/LOSS** figure) has been **removed**
(2026-06-09) from both the PDF (`dsr-report.ejs`, the `.total-gross` div) and the
Excel builder in `dsr.report.controller.js`.

The summary table uses `page-break-inside: avoid` so it is never split across two PDF
pages — it stays together on a single page.

### Heading date label (DSR-specific)

The report heading reflects the **chosen date operator** instead of always printing a
`From … To …` range (2026-06-09). Built in `dsr.report.controller.js` (`header.dateRange`):
- `between` → `From <start> To <end>`
- `=` → `Date: <date>`
- `>=` → `From <date>`, `>` → `After <date>`, `<=` → `Up To <date>`, `<` → `Before <date>`, `<>` → `Not <date>`
- `isBlank` / `isNotBlank` / no date → empty heading

Previously the heading was hardcoded to `From <start> To <end>` with `end` defaulting to
today, so an `=` filter wrongly showed a range up to the current date.

Implemented in `psback/views/pages/reports/dsr-report.ejs` (PDF) and the Excel builder
inside `psback/controllers/reports/dsr.report.controller.js`. The Daily Sale Report
template/controller are untouched.

### Readability formatting (DSR-specific)

Because the condensed 20-column layout leaves free space on the landscape page, DSR uses
larger fonts than Daily Sale Report for easier reading:
- Main table text cells/headers `10px` (vs `8px`); **number/amount cells (`.amount`) `12px`**
  (2026-06-09) so values read clearly; body `11px`, report title `18px`, summary table `11px`.
- Cell padding tightened to `2px 3px` (2026-06-09) to free horizontal space so the larger
  `12px` numbers still fit on one landscape page without triggering auto-shrink.
- Client/PNR/Pax text columns widened.
- PDF rendered at `zoom: 1.0` (Daily Sale Report uses `0.75`); smart shrinking stays on as a
  safety net so a wide row auto-fits the landscape width instead of clipping.
- The bottom **TOTAL PROFIT/LOSS** block is left-aligned (`.total-gross { text-align: left }`)
  so it sits under the left-side summary table.

> Note: the report-number prefix is `DSRR` (not `DSR`) because `DSR` is already used
> internally by the Daily Settlement report. The on-screen name is still **DSR**.

---

## Files

### New files (the replica)
| File | Purpose |
|------|---------|
| `psback/controllers/reports/dsr.report.controller.js` | Backend controller `getDSRReport` (copy of `getDailySaleReport`) |
| `psback/views/pages/reports/dsr-report.ejs` | PDF template (copy of `daily-sale-report.ejs`, title = DSR) |
| `psfront/src/pages/Report/DSR.jsx` | Frontend filter page (copy of `DailySaleReport.jsx`) |

### Existing files — additive wiring only (Daily Sale Report code untouched)
| File | Change |
|------|--------|
| `psback/controllers/reports/index.js` | require + export `getDSRReport` |
| `psback/routes/report.route.js` | import `getDSRReport`; add `POST /dsr` (permission `DSR-Report`) |
| `psfront/src/api/report.js` | add `getDSRReport()` API client + export |
| `psfront/src/App.jsx` | import `DSR`; add `<Route path="dsr">` |
| `psfront/src/pages/Report/Report.jsx` | add **DSR** menu item under History (gated by `DSR-Report`) |

---

## Permission setup (one-time)

DSR is gated by a dedicated permission `DSR-Report`. Add it to the master list, then
assign it to the relevant user/role:

```sql
INSERT INTO permissions (permission_name, model_name) VALUES ('DSR-Report', 'Report');
```

After that, grant `DSR-Report` to the user/role through the normal permissions screen
(writes to `user_permissions`). The DSR menu item and route only appear/work for users
who have this permission.

---

## Report Metadata

```javascript
{
  report_number: "DSRR" + timestamp,
  report_type: "dsr-report",
  file_type: "xlsx" | "pdf"
}
```

---

**Document End**
