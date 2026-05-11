# Inventory Report - Technical Documentation

**Version**: 1.24
**Date**: May 2026
**Author**: System Analysis
**Status**: Stable - Verified

**Changelog**:
1. **v1.24** — Multiple iterations rolled into one entry:
   1. **Product Type filter** added (8th filter). Operates on `service_type.type` (string values like `Air`, `Hotel`, `Visa`, `Insurance`, `Train`, `Tour`, `Cruise`, `Car_Transfer`, `Rental_Car`, `Hajj`, `Umrah`, `Misc`). Standard 4-operator set (`isNotBlank` / `isBlank` / `isEqual` / `in`). Backend reuses the existing `service_type` include — both Product Code (`id`) and Product Type (`type`) clauses live in the same `serviceTypeWhere` and AND together. Frontend dropdown deduplicates types from the loaded `serviceTypes` list.
   2. **Binding column widths** — Switched the Inventory Report's `compact-table` to `table-layout: fixed` AND added a `<colgroup>` block at the top of the table with one `<col style="width: …px">` per column. Earlier `<th>` width hints were just suggestions and wkhtmltopdf was stretching columns to fill blank space. The colgroup approach is universally honored — column widths are now exact.
   3. **Larger fonts** — PDF: `th` (header) → 14px, `td` (data) → 13px (was 12px both). Excel: column header → 14 (was 12), data + Grand Total rows → 13 (was 11).
   4. **Tightened column widths** to user-specified values. Final values:
      - Ticket Date: 55px / 10
      - Ticket Num: 60px / 10
      - Air-Code: 35px / 6
      - PNR: 50px / 8
      - Status: 35px / 6
      - Invoice Number: 60px / 10
      - P-Type: 40px / 6
      - P-Code: 35px / 6
      - Passenger Name: 130px / 18
      - Departure Date: 50px / 8
      - Arrival Date: 50px / 8
      - Itinerary: 40px / 6
      - Supplier No: 55px / 10
      - XO Number: 60px / 10
      - Publish Fare: 75px / 11
      - Tax Amount: 70px / 11
      - Commission: 70px / 11
      - WHT: 55px / 9
      - Extra Charges: 70px / 11
      - Total Cost: 80px / 12
      - IATA: 40px / 8
   5. **Grand Total label moved** from the Ticket Date column (first) to the XO Number column (right before the amount columns) — reads more naturally as a row anchor for the totals.
   6. **Truncation note**: at the new 13px data font size with these tight widths, several columns (Status, Itinerary, IATA, Supplier No, Invoice/XO/Ticket Num) will show `…` ellipsis on longer values. This is intentional per user preference — the layout prioritizes overall density over full readability of every cell.
2. **v1.23** — `Total Cost` now **includes** Extra Charges (`free_of_cost`). Reverses the v1.1 design decision that kept Extra Charges as a standalone column only. Now `costPerUnit = nettRate + regularTax + whtAmount + extraCharges`, matching the formula used by Airline Sales and Daily Sale reports. Behavior change: previously generated reports on S3 won't reflect this; only newly generated reports do.
2. **v1.21** — Three combined fixes: (a) Bumped `.compact-table` font to 12px in BOTH the screen rule (was 8px) AND the `@media print` rule (was 11px). The earlier 11px-only print fix didn't show up in the PDF because wkhtmltopdf wasn't reliably applying `@media print`, so the 8px screen rule was leaking through. Setting both rules to the same size guarantees consistent output regardless of media mode. Excel data + Grand Total fonts also bumped 10pt → 11pt. (b) Added `supp_no` to the width maps (`75px → 95px` after the 12px-font rebalance) so Supplier No isn't oversized. (c) Widened all explicit-width columns to fit 12px content cleanly (dates 65→75, ticket_no 70→80, invoice/supp/xo IDs 70-75 → 90-95, etc.) — no truncation expected at the new font size.
2. **v1.20** — Added `supp_no: 75px / 14` plus widened narrow columns slightly (dates 55→65px, PNR/Status 55→60px, P-Type/P-Code 35→40px, IATA 60→65px, Itinerary 50→55px) and bumped print font 7px → 9px. *(Note: the print-font bump silently failed to take effect — see v1.21 above for the actual fix.)*
3. **v1.19** — Tightened more columns. `issue_date` 60px → 55px. Added `dep_date` (55px), `arr_date` (55px), `invoice_no` (70px), `xo_number` (70px) to the width maps. Excel widths mirrored. Now 13 of the 21 columns have explicit widths; the remaining 8 share the leftover A3 landscape width.
2. **v1.18** — Two readability tweaks: (a) bumped the `.compact-table` print font from 7px → 9px and padding from 1px/2px → 2px/3px so the PDF is easier to read. (b) Extended `columnWidths` / `excelColumnWidths` with six more narrow columns (PNR, Status, P-Type, P-Code, Itinerary, IATA) so they don't waste horizontal space.
2. **v1.17** — Narrowed three columns that were taking too much horizontal room: Ticket Date (60px), Ticket Num (70px), Air-Code (40px). Implemented via a new `columnWidths` map passed to the EJS template (the template's existing per-column width logic picks it up) and a parallel `excelColumnWidths` override that runs after the Excel auto-fit loop so the explicit widths win. Pure cosmetic — no data logic touched.
2. **v1.16** — Two cosmetic tweaks: (a) shortened six column header labels — `Ticket Issue Date → Ticket Date`, `Ticket Number → Ticket Num`, `Airline Code → Air-Code`, `Ticket Status → Status`, `Product Type → P-Type`, `Product Code → P-Code`. (b) Added `iata_no` to `rightAlignKeys` so the IATA column data is right-aligned in both PDF and Excel. No data logic, key names, filters, or includes touched.
2. **v1.15** — Added a per-key right-align override. The 6 amount columns — `publish_fare`, `tax_amount`, `commision`, `wht`, `extra_charges`, `cost` — now render right-aligned in both PDF and Excel even though `alignAllLeft: true` is still in effect for everything else. Implemented via a new `rightAlignKeys` list that the EJS template and Excel writer both consult before falling back to the global alignment setting. Other reports sharing `report1.ejs` are unaffected (they don't pass `rightAlignKeys`).
2. **v1.14** — Reordered the 21 columns to match the user's preferred layout (Ticket Issue Date first, IATA last) and refreshed every header label via the existing `columnHeaders` override map. No data logic, no key renames, no Sequelize changes — purely a cosmetic reorder + relabel. The new column order is documented in the Columns table below.
2. **v1.13** — Right-side columns were still clipped because A3 wasn't actually being applied. `psback/services/pdf.js` hardcodes `pageSize: 'A4'` in the wkhtmltopdf options, and wkhtmltopdf ignores the `@page size` CSS rule when a `pageSize` option is set. Fixed by passing an explicit `{ pageSize: 'A3' }` as the third argument to `createPdf(html, true, { pageSize: 'A3' })`. The `pageSize` EJS flag I added in v1.8 is now effectively redundant but harmless — kept for documentation/intent.
2. **v1.12** — Right-side columns (Commission and Cost) were still being clipped at A3 landscape because the default 9px font + 2px/4px padding made each cell wider than needed. Added a new `compactTable: true` flag plus a `.compact-table` CSS class that drops the table to 8px on screen / 7px in print and tightens padding to 1px/2px. All 21 columns now fit on a single A3 landscape page without truncation. Other reports sharing `report1.ejs` are unaffected (they don't pass the flag).
2. **v1.11** — Status column now shows just `"Ticketed"` or `"Refunded"`. The refund document number is no longer appended in parentheses (user preference — diverges from the Inventory tab, which still shows `Refunded (TTRF…)`). The XO Number column is the place to look up the costing document number; refund document numbers are not surfaced anywhere on this report.
2. **v1.10** — Three layout/data fixes:
   1. Supplier No column was empty: code was reading `service?.Supplier` (capital S) but the Sequelize association registered in `models/index.js` uses lowercase `supplier`, so the accessor returned `undefined`. Fixed to read `service?.supplier?.supp_no` (with a defensive fallback to capital). Also renamed the column key from `supplier_no` to `supp_no` so the header reads **"Supp No"**.
   2. Renamed `product_code` to `p_code` and added a header override of **"P-Code"** (the auto-capitalizer would otherwise emit "P Code"). Same override mechanism now drives `PNR`, `WHT`, `IATA No`, `XO Number`, `Supp No` — all rendered in correct casing in both PDF and Excel.
   3. XO Number column was leaking invoice numbers (e.g. `KHIN…`) for some rows because the join `documents.document_id = costs.id` matched any document type when the FK happened to coincide. Made the include explicitly `where: { document_type: 'costing' }, required: false` so only XO/costing document numbers appear; rows with no costing document show empty.
2. **v1.8** — Layout polish: (a) renamed `issue_issue_date` key to `issue_date` so the header now reads "Issue Date" instead of "Issue Issue Date"; (b) all PDF and Excel cells now left-aligned; (c) PDF page size switched to A3 landscape so all 21 columns fit without right-side clipping; (d) report title rendered with a new `prominent` class so it's visually larger above the table. All template changes are additive — other reports sharing `report1.ejs` are unaffected because they don't pass the new `pageSize`, `alignAllLeft`, or `prominentTitle` flags.
2. **v1.7** — Fixed Ticket Status post-filter aliasing bug. `let filtered = data1` followed by `data1.length = 0; data1.push(...filtered)` emptied both arrays whenever no filter actually ran (because `filtered` was the same reference as `data1`). Switched to a predicate-based approach that only mutates `data1` when a filter genuinely applies. This was the root cause of empty reports under default filter values, regardless of the date range.
2. **v1.6** — Fixed company scoping. The previous path was `service.user → company.code` with `required: true`, but `services.user_id` is nullable, so the inner join was excluding most ticketed rows and the report came back empty. Switched to `service.Order.user.company_code` (the same path the Inventory tab uses), which is reliable for ticketed services.
2. **v1.5** — Default filter operators changed: Ticket Issue Date defaults to `between`, all other filters default to `isEqual`. Hardened the date `between` branch to skip safely when only one date (or none) is entered. Documented the "operator selected but no value → filter skipped" semantics.
2. **v1.4** — `status` column now matches the Inventory tab: values are `"Ticketed"` or `"Refunded (<refund_no>)"`, computed from the service's printed refunds (not from `invoice.status` — that was a wrong mapping introduced in v1.2). Ticket Status filter is now a post-fetch filter on the two real values (`Ticketed`, `Refunded`). Added `refund` include on service. Grand Total now recomputed after the status filter so totals match the rows shown.
2. **v1.3** — Removed two always-noise columns: `special_fare_code` (always null) and `fare_amount` (duplicate of `publish_fare`). Column count dropped from 23 to 21.
2. **v1.2** — Expanded the filter set from 2 filters to 7 (Product Code, Ticket Issue Date, Airline, IATA No, Supplier No., Ticket Status, Branch). Product Code, Airline, IATA No, Supplier No., and Ticket Status all support `isNotBlank / isBlank / isEqual / in` operators. The `status` column in the report body now reflects the actual invoice status from DB (was previously hardcoded to `"ticketed"`).
3. **v1.1** — Added 6 new columns (Supplier No., PNR, Extra Charges, WHT, IATA No, XO Number) inserted right after Issue Issue Date. Added a Grand Total row at the end of both PDF and Excel output. Cost formula unchanged.
4. **v1.0** — Initial documentation.

---

## Overview

The Inventory Report lists every ticketed passenger in the system, one row per `service_passenger` record that has a non-null `ticket_number`. For each ticket it shows the airline, product, invoice, passenger, flight itinerary, and the published/commission/tax/cost breakdown for that single ticket.

It answers the question: *"What tickets have been issued (inventory) and what do they look like financially per passenger?"*

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/InventoryReport.jsx`

**Filters**:

1. **Product Code** — `isNotBlank`, `isBlank`, `isEqual`, `in` (applied to `service_type.id` via the `service_type` include).
2. **Product Type** — `isNotBlank`, `isBlank`, `isEqual`, `in` (applied to `service_type.type` — string values like `Air`, `Hotel`, `Visa`, `Insurance`, `Train`, `Tour`, `Cruise`, `Car_Transfer`, `Rental_Car`, `Hajj`, `Umrah`, `Misc`).
3. **Ticket Issue Date** — `=`, `<`, `<=`, `>`, `>=`, `<>`, `between` (applied to `service.ticket_issue_date`).
4. **Airline** — `isNotBlank`, `isBlank`, `isEqual`, `in` (applied to `service.airline_form`).
5. **IATA No** — `isNotBlank`, `isBlank`, `isEqual`, `in` (applied to `service.Order.branch.iata_number`).
6. **Supplier No.** — `isNotBlank`, `isBlank`, `isEqual`, `in` (applied to `service.supplier_id`).
7. **Ticket Status** — `isNotBlank`, `isBlank`, `isEqual`, `in`. Operates on the **computed** status (`Ticketed` / `Refunded`), not on any DB column. Applied as a post-fetch JS filter after the SQL result is built (see "Filter Logic → Ticket Status" below).
8. **Branch** — `isNotBlank`, `isBlank`, `isEqual` (applied to `Service.Order.branch_id`).

**Output**: PDF or Excel (user picks from the Generate dropdown).

**History table**: Lists previously generated Inventory Reports from the `report` table where `report_type = 'inventory-report'`, with per-row PDF/Excel download links.

### Frontend API

**File**: `psfront/src/api/report.js`
**Function**: `getInventoryReport(data)` → `POST /report/inventoryReport`

### Backend Route

**File**: `psback/routes/report.route.js`
**Endpoint**: `POST /api/report/inventoryReport`
**Middleware**: `authenticate`, `permission("Inventory-Report")`

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getInventoryReport` (line ~1664)

### PDF Template

**File**: `psback/views/pages/reports/report1.ejs` (generic report template — used by multiple reports, renders from `data1[]` and `header`).

### Report Persistence

- Document number: `"TPIR" + Date.now()` (e.g. `TPIR1714044823051`).
- Stored in `report` table with `report_type = 'inventory-report'` and `file_type = 'pdf' | 'xlsx'`.
- File uploaded to S3/MinIO via `uploadFile()`.

---

## Data Source

Root query: `db.service_passenger.findAll(...)` with a hard filter of `ticket_number IS NOT NULL`.

**Included models (eager-loaded)**:

1. `passenger` — passenger name.
2. `service` (`required: true` — INNER JOIN) →
   1. `airline_code` — for `airline_ticket_prefix`.
   2. `service_flight` (has many) → `city_code as city_from_code`, `city_code as city_to_code`.
   3. `invoice` → `invoice_tax`.
   4. `refund` (alias `refunds`) — used to compute the Ticket Status column.
   5. `cost` → `cost_tax`, `document` (the costing document — XO).
   6. `service_type` — for `product_code` and `type`.
   7. `supplier` — for `supp_no` (Supplier No. column).
   8. `order` (`required: true`) →
      1. `user` (`required: true`, `where: { company_code: req.user.company_code }`) — **company isolation lives here**.
      2. `branch` — used by the Branch filter and for `branch.iata_number` (IATA No. column).

**Why company scoping is on `Order.user`, not `Service.user`**: `services.user_id` is nullable, so requiring `service.user.company.code = X` would exclude every service whose `user_id` is null (i.e. most of them). `orders.user_id` is reliably set for ticketed services. The Inventory tab uses the same path.

---

## Filter Logic

All non-date filters share the same operator set: `isNotBlank`, `isBlank`, `isEqual`, `in`. The controller uses two helpers:

1. `toArray(value)` — coerces a single id, an array, or a comma-separated string into a deduplicated non-empty array (used by the `in` operator).
2. `applySimpleFilter(target, field, op, value)` — attaches the right `Sequelize.Op` clause to the `where` object for the four standard operators.

All filters AND together. Empty/unset filters are skipped.

### Frontend defaults

When the page first loads:

1. **Ticket Issue Date** → `between` (with both dates empty).
2. All other filters (Branch, Product Code, Airline, IATA No, Supplier No., Ticket Status) → `isEqual` (with no value selected).

### "Operator selected but no value" semantics

Every filter is **bypassed** at the controller when the chosen operator needs a value but no value/range is supplied:

1. `isEqual` with empty/null value → filter skipped.
2. `in` with empty list → filter skipped.
3. `between` (date) with only one or zero dates → filter skipped.

This means a fresh page submission (no values touched) returns the full dataset, rather than returning zero rows or throwing.

### Product Code & Product Type

Both filters share the same `service_type` include — the controller builds a single `serviceTypeWhere` object with clauses for `id` (Product Code) and `type` (Product Type) and lets Sequelize AND them together.

**Product Code** (operates on `service_type.id`):

1. `isNotBlank` → `service_type.id IS NOT NULL` with `required: true` (acts as INNER JOIN).
2. `isBlank` → no-op (every service has a service_type; filter is skipped).
3. `isEqual` → `service_type.id = :product_code` with `required: true`.
4. `in` → `service_type.id IN (...ids)` with `required: true` (ids from comma-separated string).

**Product Type** (operates on `service_type.type`, the string column):

1. `isNotBlank` → `service_type.type IS NOT NULL` with `required: true`.
2. `isBlank` → `service_type.type IS NULL` (LEFT JOIN — service_types with explicit null type are rare but supported).
3. `isEqual` → `service_type.type = :product_type` with `required: true`. Frontend dropdown shows the deduplicated list of types (Air, Hotel, Visa, …) loaded from the existing `serviceTypes` API.
4. `in` → `service_type.type IN (...types)` with `required: true` (comma-separated strings).

When both filters are active, only services matching BOTH the id AND type conditions pass — useful for narrowing within a product category (e.g., "Air" type AND product_code 12).

### Ticket Issue Date

Applied to `service.ticket_issue_date`:

1. Operators `=`, `<`, `<=`, `>`, `>=`, `<>` → compared against `new Date(startDate)`.
2. Operator `between` → compared against `[new Date(startDate), new Date(endDate)]`.
3. `dateFilter === "blank"` (UI default) is skipped.

### Airline

Applied to `service.airline_form` (the same field Airline Sales Report filters on). Standard 4-operator set.

### IATA No

Applied to `service.Order.branch.iata_number` (matches literal IATA-number strings, not branch IDs). Standard 4-operator set.

### Supplier No.

Applied to `service.supplier_id`. Standard 4-operator set. The UI dropdown lets users pick by supplier, and the ID is sent to the backend.

### Ticket Status

Ticket Status is **computed**, not stored. It mirrors the Inventory tab's logic from `psback/controllers/inventory.controller.js`:

1. Eager-load `service.refunds` (`Service.hasMany(refund, { as: 'refunds' })`).
2. For each ticket row, look at the service's refunds and find one matching ALL of:
   1. `refund.status === 'Printed'`.
   2. EITHER `refund.selected_tickets` is null/empty (whole service refunded), OR the array contains this ticket — matching by string equality, or by `ticket_number` / `ticketNumber` / `ticket_no` field on object entries, or by `service_passenger.id`.
3. If a printed refund matches → both display and plain values are `"Refunded"` (the refund document number is intentionally not appended).
4. Otherwise → both display and plain are `"Ticketed"`.

The display value goes into the `status` column. The plain value is stashed on a temporary `_ticketStatusPlain` field used only for the filter, then deleted before rendering. (Display and plain are now identical — the field still exists separately so the filter never matches against a parenthesized variant in case the format changes again later.)

**Filter (post-fetch JS)** — runs after the Sequelize result is built into `data1`:

1. `isNotBlank` → keeps rows whose `_ticketStatusPlain` is non-empty (effectively all rows).
2. `isBlank` → keeps rows whose `_ticketStatusPlain` is empty (rare, only if a ticket has neither status — practically unused).
3. `isEqual` → case-insensitive match against the single value (`Ticketed` or `Refunded`).
4. `in` → case-insensitive match against any value in the comma-separated list.

The Grand Total is recomputed **after** this filter so the total reflects only the visible rows.

**Why post-fetch and not SQL?** The match against `refund.selected_tickets` is JSON-array element matching against multiple possible field names — practically very awkward in raw SQL. Inventory tab uses the same JS pattern. Consistency wins over a marginal perf gain.

### Branch

Applied to `Service.Order.branch_id`. Standard 3-operator set (no `in` currently). The default UI value is `isNotBlank` (all branches with any assigned branch).

---

## Row Level

One row per `service_passenger` record with a `ticket_number`. If a service has N passengers, that service produces N rows; the financial fields on each row are computed from the service-level `cost` record and are **repeated per passenger row** (no division by passenger count).

---

## Columns

Rendered left-to-right in this order (21 columns total). Display labels are set explicitly via the `columnHeaders` override map — the keys themselves stay in snake_case.

| # | Display label | Key | Source | Description |
|---|---------------|-----|--------|-------------|
| 1 | Ticket Date | `issue_date` | `service.ticket_issue_date` | Ticket issue date, formatted `DD MMM YYYY`. |
| 2 | Ticket Num | `ticket_no` | `service_passenger.ticket_number` | Ticket number of the passenger. |
| 3 | Air-Code | `airline` | `service.airline_code.airline_ticket_prefix` | Airline ticket prefix (e.g. `PK`, `EK`). |
| 4 | PNR | `pnr` | `service.pnr` | GDS PNR (Passenger Name Record) — typically a 6-character alphanumeric code. |
| 5 | Status | `status` | computed from `service.refunds` | `"Ticketed"` if no printed refund covers this ticket, otherwise `"Refunded"` (no refund document number is appended). See "Filter Logic → Ticket Status" for the detection rules. |
| 6 | Invoice Number | `invoice_no` | `service.Invoice.invoice_number` (falls back to `service.Invoices[0].invoice_number`) | Invoice number that this ticket is billed on. |
| 7 | P-Type | `product_type` | `service.service_type.type` | Product type description. |
| 8 | P-Code | `p_code` | `service.service_type.product_code` | Product code (e.g. 12, 13 for air products). |
| 9 | Passenger Name | `passenger_name` | `passenger.passenger_name` | Passenger's full name. |
| 10 | Departure Date | `dep_date` | `service_flights[0].departure_date` | Departure date of the first flight leg, `DD MMM YYYY`. |
| 11 | Arrival Date | `arr_date` | `service_flights[last].arrival_date` | Arrival date of the last flight leg, `DD MMM YYYY`. |
| 12 | Itinerary | `itinerary` | `service_flights[0].city_from_code.code` + `/` + `service_flights[last].city_to_code.code` | Origin → final destination city codes (e.g. `KHI/DXB`). |
| 13 | Supplier No | `supp_no` | `service.supplier.supp_no` | Supplier code for the ticket issuer (e.g. BSP supplier). Empty string if service has no supplier. |
| 14 | XO Number | `xo_number` | `cost.Document.document_number` with explicit `where: { document_type: 'costing' }` on the include | XO (costing) document number ONLY. Empty if the cost has no costing document. The explicit `where` prevents invoice numbers from leaking when an invoice happens to share the cost's id. |
| 15 | Publish Fare | `publish_fare` | `cost.published_rate` | Published (gross) rate per unit. |
| 16 | Tax Amount | `tax_amount` | Σ `cost.cost_taxes[i].tax_amount` | Sum of regular cost taxes (excludes WHT). |
| 17 | Commission | `commision` | `cost.published_rate × cost.commission / 100` | Commission amount (from agent's cost). The key has a typo (`commision`) preserved for compatibility; the override map forces the header to display "Commission". |
| 18 | WHT | `wht` | `commissionAmount × cost.sst / 100` | Withholding tax — WHT percent (stored in `cost.sst`) applied to the commission amount. |
| 19 | Extra Charges | `extra_charges` | `cost.free_of_cost` | Extra charges on the cost (the `free_of_cost` field is repurposed for extra charges). Shown as its own column AND folded into Total Cost (as of v1.23). |
| 20 | Total Cost | `cost` | See formula below | Unit cost after commission deduction, plus taxes and WHT. |
| 21 | IATA | `iata_no` | `service.Order.branch.iata_number` | IATA number of the issuing branch (used for BSP settlement). |

All numeric columns are rounded to 2 decimal places (`.toFixed(2)`) before rendering.

### ID-like columns (no currency formatting)

These keys are added to `report1.ejs`'s `noFormatKeys` list so they render verbatim and are not auto-formatted as currency:

1. `supp_no` (and the legacy `supplier_no` for backward compatibility)
2. `pnr`
3. `iata_no`
4. `xo_number`
5. `p_code`

This is additive-only — other reports that share `report1.ejs` are unaffected (the extra keys are simply unused by them).

### Column-header override map

The controller passes a `columnHeaders` map to the EJS template (and the same map drives the Excel header writer). When a key has an entry, that string is used as the header verbatim instead of the auto-capitalized snake_case → "Title Case" conversion.

| Key | Override |
|-----|----------|
| `issue_date` | `Ticket Date` |
| `ticket_no` | `Ticket Num` |
| `airline` | `Air-Code` |
| `pnr` | `PNR` |
| `status` | `Status` |
| `invoice_no` | `Invoice Number` |
| `product_type` | `P-Type` |
| `p_code` | `P-Code` |
| `passenger_name` | `Passenger Name` |
| `dep_date` | `Departure Date` |
| `arr_date` | `Arrival Date` |
| `itinerary` | `Itinerary` |
| `supp_no` | `Supplier No` |
| `xo_number` | `XO Number` |
| `publish_fare` | `Publish Fare` |
| `tax_amount` | `Tax Amount` |
| `commision` | `Commission` |
| `wht` | `WHT` |
| `extra_charges` | `Extra Charges` |
| `cost` | `Total Cost` |
| `iata_no` | `IATA` |

Keys not in the map fall through to the default capitalize-each-word behavior. The map is also additive — other reports don't pass it, so they keep their existing headers.

---

## Cost Calculation (column `cost`)

For each row:

1. `publishedRate = cost.published_rate`
2. `commissionPercent = cost.commission` *(percentage, e.g. 7 for 7%)*
3. `commissionAmount = publishedRate × commissionPercent / 100`
4. `nettRate = publishedRate − commissionAmount`
5. `regularTax = Σ cost_taxes[i].tax_amount`
6. `whtPercent = cost.sst` *(WHT percentage stored in the `sst` field)*
7. `whtAmount = commissionAmount × whtPercent / 100`
8. `extraCharges = cost.free_of_cost` *(exposed as its own `extra_charges` column)*
9. `costPerUnit = nettRate + regularTax + whtAmount + extraCharges` *(Extra Charges included as of v1.23)*
10. `cost = costPerUnit × 1` *(quantity is always 1 at the passenger-row level)*

**Plain-language**: Start with the published rate, strip out the commission the agent earns, add back the taxes that are passed through, add withholding tax on the commission portion, and add Extra Charges. Now matches Airline Sales / Daily Sale reports.

---

## Grand Total Row

A single **Grand Total** row is appended at the end of the report (both PDF and Excel).

1. Label `"Grand Total"` in the first column (`ticket_no`).
2. Numeric columns summed across all data rows: `extra_charges`, `wht`, `publish_fare`, `tax_amount`, `commision`, `cost`.
3. Non-numeric / ID columns left blank in the total row.
4. In PDF: rendered by the shared `report1.ejs` template via the `data1Total` variable (bold, styled with the `.total-row` class).
5. In Excel: rendered by the controller as a bold row with a light-grey fill (`#EFEFEF`) immediately after the last data row.
6. Per-page subtotals are **not** implemented — only one grand total at the bottom.

---

## Header / Filter Display

The PDF/Excel header shows:

1. Company name and address (from `req.user.company`).
2. Title: *"Inventory Report"*.
3. Report ID (the `TPIR...` document number).
4. Printed By (current username).
5. Print Date/Time (server local time).
6. Applied filter summary:
   1. *Ticket Issue Date* — if only `startDate` is set, shown as `[startDate, startDate]`; if both set, `[startDate, endDate]`. *(Operator is not shown — only the dates are surfaced.)*
   2. *Branch* — shown using `branch.code` when `branch_id` is selected.

---

## Excel Output

1. Workbook with one sheet named *"Inventory Report"*.
2. First 7 rows frozen (company header, address, title, report id, printed by, print date/time, blank spacer).
3. Column headers derived automatically from the keys of `data1[0]` — `_` replaced with space, each word capitalized.
4. Amount-like columns (matched by regex on the key name: `_amount`, `_fare`, `_cost`, `_tax`, `_commision`, `_rate`, `_nights`, `_rooms`) are still formatted with `toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })`, **but every cell is left-aligned** (matches the user's preference established in v1.8).
5. All cells bordered; column-header row has a grey (`#D9D9D9`) fill.
6. **Grand Total row** appended after the last data row — bold font, light-grey (`#EFEFEF`) fill, same border style as data rows.
7. Column widths auto-fit between 10 and 50 characters.
8. File name: `<TPIR-number>.xlsx`.

## PDF Output

1. Rendered from `pages/reports/report1.ejs` (shared generic template).
2. Variables passed: `data1`, `data1Total`, `header`, **plus seven Inventory-Report-specific flags**:
   1. `pageSize: 'A3'` — switches `@page size` from the default `A4 landscape` to `A3 landscape` (more width for 21 columns). Note: actual page size is also explicitly forced via `createPdf(html, true, { pageSize: 'A3' })` because wkhtmltopdf ignores CSS `@page` when its own `pageSize` option is set.
   2. `alignAllLeft: true` — every header and data cell renders `text-align: left` by default.
   3. `rightAlignKeys: ['publish_fare', 'tax_amount', 'commision', 'wht', 'extra_charges', 'cost', 'iata_no']` — per-key override that right-aligns the 6 amount columns plus IATA even though `alignAllLeft` is true. The template's `cellAlign` helper checks this list first.
   4. `prominentTitle: true` — adds a `.prominent` class to the `.page-subtitle` div, bumping the font size to 16px so the report title stands out.
   5. `compactTable: true` — adds a `.compact-table` class to the `<table>` element. As of v1.24 this class also enables `table-layout: fixed`, sets `th` font-size to **14px** and `td` font-size to **13px**, and tightens padding to 2px/4px. Combined with the `<colgroup>` (see flag 7) this gives binding column widths so the table doesn't stretch into blank space.
   6. `columnHeaders` — per-key header override map for the 21 final display labels.
   7. `columnWidths` — drives BOTH the inline `style="width: …"` on each `<th>` element AND the `<col style="width: …">` entries in the `<colgroup>` block at the top of the table. The colgroup is what makes widths binding under `table-layout: fixed`. Currently set for all 20 non-passenger columns; passenger_name has its own explicit 130px entry. Final values (PDF / Excel chars):
      - Ticket Date 55 / 10
      - Ticket Num 60 / 10
      - Air-Code 35 / 6
      - PNR 50 / 8
      - Status 35 / 6
      - Invoice Number 60 / 10
      - P-Type 40 / 6
      - P-Code 35 / 6
      - Passenger Name 130 / 18
      - Departure Date 50 / 8
      - Arrival Date 50 / 8
      - Itinerary 40 / 6
      - Supplier No 55 / 10
      - XO Number 60 / 10
      - Publish Fare 75 / 11
      - Tax Amount 70 / 11
      - Commission 70 / 11
      - WHT 55 / 9
      - Extra Charges 70 / 11
      - Total Cost 80 / 12
      - IATA 40 / 8
3. All seven flags are **additive** in `report1.ejs` — other reports that don't pass them keep their original behavior.
4. The template's `data1Total` block renders the Grand Total as a bold `.total-row` at the end of the table.
5. Generated via `createPdf(html, true)` (wkhtmltopdf pipeline).
6. File name: `<TPIR-number>.pdf`.

---

## Known Quirks / Gotchas

1. ~~**Key typo**: `issue_issue_date`~~ — *fixed in v1.8.* The key is now `issue_date` and renders as "Issue Date".
2. **Status column is computed, not from DB**: as of v1.4 `status` is derived from the service's printed refunds (same logic as the Inventory tab) — values are `"Ticketed"` or `"Refunded (<refund_no>)"`. It is NOT `invoice.status`. The hard filter `ticket_number IS NOT NULL` on `service_passenger` is still applied at SQL level, so only issued tickets appear.
3. **Tax split**: `tax_amount` is regular tax only (sum of `cost_taxes`). Withholding tax (WHT, stored in `cost.sst`) is **not** included in `tax_amount` — it is folded into the final `cost` column instead.
4. **Invoice fallback**: The controller reads `service.Invoice || service.Invoices[0]`, so if a service has multiple invoices only the first is shown.
5. **Per-passenger duplication**: Cost figures are the full service-level unit cost; they are **not divided among passengers**. Summing the `cost` column across rows will overstate true cost when a service has more than one passenger.
6. **Filter display does not echo operator**: The PDF/Excel header shows only the date range, not whether the user picked `<=`, `>`, etc.

---

## Permission

Route is gated by `permission("Inventory-Report")`. The permission record must exist for the user's role in the `permissions` table.

---

## Related Files

1. Frontend page: `psfront/src/pages/Report/InventoryReport.jsx`
2. Frontend API: `psfront/src/api/report.js` → `getInventoryReport`
3. Backend route: `psback/routes/report.route.js` → `POST /inventoryReport`
4. Backend controller: `psback/controllers/report.controller.js` → `getInventoryReport` (line ~1664)
5. PDF template: `psback/views/pages/reports/report1.ejs`
6. Report registry row: `report` table, `report_type = 'inventory-report'`, `document_number` prefix `TPIR`.
