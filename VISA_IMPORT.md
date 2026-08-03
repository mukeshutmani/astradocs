# Visa Import Module

**Status: IMPLEMENTED (2026-07-23) — backend + frontend built per this spec. DB migration `V2026.07.23.001__add_type_to_lcc_profile_templates.sql` applied to `astra_test` via the Flyway pipeline. Staging/production migration pending Mukesh's approval. Awaiting UI verification.**

## 1. What this module does

1. Lets the user import visa sales from an Excel file, the same way Air bookings are imported today through LCC templates.
2. Each imported row becomes a complete visa order in the system: order + visa service + applicant details + cost + invoice.
3. Uses the single visa product code **71 / Visa** (`service_types.id = 25`).
4. PNR is **not** used anywhere in visa import (visa has no PNR).

## 2. Key business rules (confirmed by Mukesh)

1. **One order per customer, items grouped by passenger type.**
   1. All rows of the same Customer No in one file become **one order** (one sequential order number, `<branch prefix>SO00000001` style, same numbering rules as today).
   2. Inside that order, rows group by **PTC** (passenger type) into **service items**. Example: 3 rows for one customer (2 adults + 1 child) = 1 order with 2 items — an Adult item holding 2 pax and a Child item holding 1 pax.
   3. Each Excel row is one applicant — one pax line inside its item. The visa order detail page and visa summary show the item's passengers **pax-wise** (each applicant's own name, passport, etc.).
   4. **Grouping key = PTC only, with a consistency rule.** Inside one customer, all rows of the same PTC must have the SAME Destination, Price, and Cost. If they differ (e.g. two adults with different countries or prices), the import shows a clear error popup telling the user which fields differ, that customer is skipped, and the user fixes the Excel and re-imports. Rows are never silently merged with mixed values.
2. **Imported orders are marked.** Each created order gets `order_type = 'Visa Import'`. The order listing page already shows the Order Type column, and the order detail page already shows Order Type — so the "this order was imported" sign appears with no extra UI work (Air import does the same with `'LCC Import'`).
3. **Visa Code is skipped.** Import does not read, match, or store any visa code. The Visa Code field on the visa detail page stays **empty** for imported visas. (Import visa orders and manual visa orders are separate; they are not synced with Visa Code Maintenance.)
4. **Everything is company-scoped.** Customer lookup, supplier lookup, order numbering, and the passport duplicate check all work inside the importing user's company only.
5. **Unknown lookup values = row error.** If a PTC / Entries / Purpose / Length of Stay / Nationality / Destination value from the Excel is not found in the maintenance lists, that row fails with a clear error. The user adds the value in maintenance and re-imports. The system never auto-creates maintenance values.
6. **Length of Stay is user-maintainable.** Already true today — full add/edit/delete endpoints exist (`/data/visa-length-stay`), so no new work needed.

## 3. Excel columns

### 3.1 Required columns (row fails without them)

| # | Excel column | Goes to | Validation |
|---|-------------|---------|------------|
| 1 | Customer No | order.customer_id (lookup) | Must exist in the importing company (via branch). |
| 2 | Supplier No | services.supplier_id (lookup) | Must exist in the importing company (supp_no is per-company). |
| 3 | Passenger Name | service_visas.given_name | Shown as **Pax Name** on the visa detail page. |
| 4 | PTC | service_visas.ptc (lookup) | Must match a code in Visa PTC Maintenance. |
| 5 | Gender | service_visas.gender | male / female (M / F accepted, any letter case). |
| 6 | Nationality | service_visas.nationality (lookup) | Must match a country in Country Codes. |
| 7 | Destination | service_visas.destination (lookup) | Must match a country in Country Codes. |
| 8 | Passport Number | service_visas.passport_number | **8 to 10 characters** (not less than 8, not more than 10). Letters, numbers, and special characters allowed. Also the duplicate key — see section 5. |
| 9 | Remarks | service_visas.remarks | **Required.** Max **45 characters** (DB limit) — longer value = row error. |
| 10 | Entries | service_visas.entries (lookup) | Must match Visa No. of Entries maintenance. |
| 11 | Purpose | service_visas.purpose (lookup) | Must match Visa Purpose maintenance. |
| 12 | Length of Stay | service_visas.length_of_stay (lookup) | Must match Visa Length of Stay maintenance. |
| 13 | Price | service_visas.price + invoices.price | Per-visa selling price. Numeric. Plain number only. |
| 14 | Cost | service_visas.cost + costs.amount / published_rate | Per-visa cost. Numeric. Plain number only. |

**Note:** "Base Price (Total from all visas)" and "Total Cost (All Visas)" are NOT Excel columns. The system calculates them itself — sum of the per-visa prices/costs of the group — and they appear on the visa order pages the same way as for a manually created order.

### 3.2 Optional columns

| # | Excel column | Goes to | Notes |
|---|-------------|---------|-------|
| 1 | Commission | costs.commission | Amount in file, converted to a percentage of cost (same conversion style as Air import). |
| 2 | Markup | invoices.markup | Amount. |
| 3 | Discount | invoices.discount | Amount in file, stored as percentage of base (same as Air import). |
| 4 | Transaction Fee | invoices.transaction_fee | **Override only.** See section 6.3. |
| 5 | Customer Name | — (informational) | Lookup is always by Customer No; this column is not used for matching. |

### 3.3 Column mapping is template-driven, pre-seeded by default

1. Like Air, the user maps their Excel's real column headings to these system fields once, in a template.
2. The mapping screen for a Visa template shows this visa field list instead of the flight field list.
3. **A new Visa template starts with all 14 required mappings already created** (Customer No, Supplier No, Passenger Name, PTC, Gender, Nationality, Destination, Passport Number, Remarks, Entries, Purpose, Length of Stay, Price, Cost). The user only edits headings that differ from their file. Defaults live in `psback/services/visaImport/defaultTemplate.js`.
4. **A sample Excel file is downloadable** from the visa page (`GET /visa-import/sample-template`) — its headings are identical to the default mappings, so the sample file imports with zero mapping changes. It contains a 2-adult + 1-child family of one customer to demonstrate the grouping rule.
5. **Empty visa templates show a "Load Default Mappings" button** (`POST /visa-import/template/:id/seed-defaults`) — fills the 14 defaults into a template created before seeding existed, or after the user deleted everything. Refuses if the template already has mappings.

## 4. What gets created

1. **Order** — one per customer group: `order_type = 'Visa Import'`, status Active, customer's branch / salesperson / booking type, sequential company-scoped order number.
2. **Service item** — one `services` row per PTC group inside the order: `service_type_id` = selected visa product (71/Visa), status Booked, **quantity = number of pax in the group**, supplier from Supplier No, `pnr` empty, linked to the order.
3. **Visa applicants** — one `service_visas` row **per Excel row (per pax)** under its service item: pax name (given_name), PTC, gender, nationality, destination, passport number, price, cost, entries, purpose, length of stay, remarks. `visa_code` stays empty. This is what makes the detail page and visa summary show pax-wise lines.
   1. **Order-level passenger records too** — each pax also gets a `passengers` row (name, passport number, type, first pax = Lead) + a `service_passengers` link, so the Passengers grid on the order page fills exactly like an Air-imported order. Visa PTC codes map down to the passenger types the grid supports: `ADT` → ADT, child codes (`C05` etc.) → CHD, `INF` → INF; the exact PTC stays on the visa row.
4. **Cost pricing row** — one `costs` row per service item, following the VISA TOTALS CONVENTION (same as the manual visa form: rate fields hold the TOTAL of all visas in the item; quantity = pax count is display-only and never multiplied): published_rate = Cost × pax, net_rate/total_costing = that total minus commission amount, commission stored as percent. Amounts are plain numbers; the system's currency feature applies as everywhere else.
5. **Invoice pricing row** — one `invoices` row per service item, same totals convention: price = Price × pax (markup NOT baked in — the visa price form applies it once at save time), markup/discount as item-total amounts (file values are per-visa, × pax), transaction fee once per item, total = price total + markup − discount + fee, status Raised. Per-person amounts live only on the `service_visas` pax rows.
6. **Order totals are calculated, not imported** — Base Price (Total from all visas) and Total Cost (All Visas) come from the system's own math (per-visa amount × pax, summed per group), exactly like a manually created visa order. The Excel has no total columns.
7. **NO invoice or XO documents are generated by the import.** The import only saves the order with its pricing data. The user generates the invoice (and XO) afterward with the existing buttons in the system, exactly like a manually created order. The customer's auto-invoice setting is ignored by this import.
8. **Order date = import day.** No sale date column is read from the file.

## 5. Duplicate protection

1. Duplicate key = **passport number, scoped per company** (case-insensitive compare). A passport repeated **inside the same file** is also caught — the second occurrence is skipped as a duplicate.
2. If any visa service in the same company already has this passport number, that pax row is skipped and reported in the "duplicates" list (same UX as Air's duplicate tickets). The customer's other pax rows still import normally.
3. If **all** pax rows of a customer group are duplicates, no order is created for that customer (mirrors Air's whole-group duplicate skip).
4. The same passport number CAN exist in two different companies (e.g. company 1001 and 9876) — that is not a duplicate.
5. **Known limitation (accepted):** a repeat customer applying for a second visa later with the same passport will be flagged as duplicate and must be entered manually.

## 6. Financial rules

### 6.1 Base amounts
1. Per-visa Price and Cost fill each pax's visa row.
2. Each service item's invoice/cost total = per-visa amount × pax count in that item.
3. Base Price (Total) and Total Cost (All Visas) are calculated by the system from those amounts — they are not in the Excel (see 4.6).

### 6.2 Optional charges
1. Commission, Markup, Discount only apply when the columns are mapped and have values.

### 6.3 Transaction Fee — auto with file override
1. **Default (no column mapped / cell empty):** the system auto-calculates the transaction fee from the customer's Fee Group rules, exactly like the manual flow — using product code 71/Visa, no airline, no routing. The existing fee resolver already supports this; it needs no modification.
2. **Override:** if the import file has a Transaction Fee column mapped and the cell has a value, that value is used instead of the auto calculation (and the invoice is marked as manual transaction fee so recalculation doesn't overwrite it).

## 7. Technical design

### 7.1 Principles
1. **Air import code is not touched.** No edits to `lccImport.controller.js` or its route. The visa import is a parallel module.
2. **SOLID structure** — small single-purpose files instead of one giant controller:

| File (new) | Responsibility |
|-----------|----------------|
| `psback/routes/visaImport.route.js` | Routes: `/visa-import/preview`, `/visa-import/import-sse`, `/visa-import/template/:id`. JWT auth, Excel-only multer (same limits as Air). |
| `psback/controllers/visaImport.controller.js` | Thin HTTP/SSE layer only — parses request, loads the visa template (company-scoped, type = visa), streams progress, delegates. |
| `psback/services/visaImport/excelParser.js` | Reads the Excel buffer into rows (visa module's own copy — Air's parser is private to its controller and stays untouched). |
| `psback/services/visaImport/rowValidator.js` | All row validations from section 3 (required fields, passport 8–10, remarks ≤ 45, gender normalize, numerics) + the PTC-group consistency check. |
| `psback/services/visaImport/lookupResolver.js` | Company-scoped customer/supplier lookup + global PTC / country / entries / purpose / length-of-stay lookups, with per-import caching. Also resolves the company base currency. |
| `psback/services/visaImport/orderBuilder.js` | Order number allocation (company-scoped gap-fill, same as Air), duplicate passport check, and creation of order + service + service_visas + cost + invoice rows. |
| `psback/services/visaImport/importService.js` | Orchestrator: one shared analysis pass (validate → resolve → group → check) used by BOTH preview and import, then one DB transaction per customer order. |
| `psback/services/visaImport/defaultTemplate.js` | Default mappings seeded into every new Visa template + the sample Excel generator (`/visa-import/sample-template`). |

3. **Reused as-is (imported, not modified):**
   - `utils/iurFeeResolver.js` → `loadFeeCustomerById` + `computeFeeGroupCharges` for the auto transaction fee.
   - Existing template tables (`lcc_profile_templates` + `templates`) — see 7.2.

### 7.2 Template type (one schema change)

1. `type` column added to `lcc_profile_templates`: `'air'` (default) or `'visa'`.
2. All existing templates automatically stay `'air'` — zero impact on current Air imports.
3. Migration `V2026.07.23.001__add_type_to_lcc_profile_templates.sql` in the Flyway repo (`astra_migration`) — **applied to `astra_test` 2026-07-23 via the pipeline; staging/production pending Mukesh's approval**.
4. `psback/models/lcc_profile_template.js` carries the new field; `data.controller.js` `createTemplate` accepts `type` ('visa' only when explicitly sent, otherwise 'air').

### 7.3 Transaction / error behavior

1. One DB transaction **per customer order**. If anything in a customer's group fails, only that customer's order rolls back; other customers still import.
2. If ANY row of a customer has a validation error, that whole customer is skipped (partial families would be confusing) — every bad row is reported with its Excel row number and a plain-English reason (e.g. `Row 7: "XX" not found in Visa PTC Maintenance`).
3. Progress streams live to the UI (SSE), same event names and experience as Air import.

### 7.4 Frontend changes

1. **Template list page** (`/iur/upload-lcc`, `UploadLcc.jsx`): "Add Template" dialog has a Type choice (Air (LCC) / Visa); visa templates show a purple "Visa" badge. Clicking an Air template goes to the existing page (unchanged); clicking a Visa template goes to `/iur/upload-visa/:id`.
2. **New Visa template + upload page** (`psfront/src/pages/VisaImport/VisaImportTemplate.jsx`): two tabs — Field Mapping (visa field list from section 3.3, warns when required mappings are missing) and Upload & Import (product dropdown filtered to Visa products with 71/Visa auto-selected, Preview with planned-orders table, SSE import with progress modal). The 2,100-line Air page is never edited.
3. The Upload tab shows the import rules inline (remarks ≤ 45 chars, passport 8–10 chars, grouping and consistency rules) so users see them before importing.
4. **No changes** to the order listing / order detail pages — the Visa Import tag rides on the existing Order Type display.

## 8. Open points (answer during doc review)

1. ~~Sale Date~~ — ANSWERED: no sale date column; order date = import day; no invoice/XO documents created by import (user generates them manually with existing buttons).
2. ~~Commission conversion~~ — ANSWERED: commission is optional; when present it is an AMOUNT and is converted to a percentage internally, same as Air import.
3. ~~Currency~~ — ANSWERED: import reads plain numbers only; the system's existing currency feature handles conversion; no currency logic in the import.
4. ~~Base Price vs calculated total mismatch~~ — ANSWERED: not applicable. The Excel has no total columns; the system calculates Base Price (Total) and Total Cost (All Visas) itself from the per-visa Price and Cost, so no mismatch is possible.
5. ~~Item grouping key~~ — ANSWERED: PTC only, with the consistency rule in section 2.1.4 (mismatched Destination/Price/Cost inside one PTC group = error popup, customer skipped).

## 9. Review fixes (2026-07-28, after venom senior review)

1. **Clear error for company-less users** — import/preview now reject up front with "Your user account has no company assigned" instead of a raw DB error (`visaImport.controller.js parseRequest`).
2. **Concurrent duplicate race closed** — each pax's passport is re-checked INSIDE the save transaction with a `SELECT ... FOR UPDATE` lock; a clash rolls the customer back with a plain message (`orderBuilder.js`).
3. **Order-number clash auto-retry** — a concurrent import taking the same order number now triggers up to 3 re-allocations instead of failing the customer (`orderBuilder.js`).
4. **Excel heading spaces trimmed** — `"Customer No "` now matches `"Customer No"` (`excelParser.js`).
5. **Blank = 0 for money columns** in the PTC-group consistency check; supplier/destination stay strictly compared (`rowValidator.js`).
6. **Oversized commission caught at validation** — commission ≥ ~1000× cost (DB percent-field overflow) now fails the row with a readable message, visible in Preview too (`rowValidator.js`).
7. **Passport duplicate lookup batched** — one query per file instead of one per row; the per-pax locked re-check (fix 2) remains (`orderBuilder.js` + `importService.js`).
8. **"First sheet only" noted on the visa page** info box.
9. **Shared mapping endpoints company-scoped** — `/data/profile*` (used by BOTH Air and Visa mapping screens since the Air era) now verify the template belongs to the user's company (or is global) before read/write/delete; also fixed a crash when deleting a non-existent mapping id (`data.controller.js templateAccessibleByUser`). **Air template screen needs one re-test after this.**
10. **Accepted as-is**: in-file duplicate marking may reference a row from a customer that itself got skipped (message stays truthful; re-import after fixing resolves it), and Preview showing only the first 20 rows (totals/errors still cover the whole file).

Round 2 (same day):

11. **Order-number retry made effective** — retries now re-scan with a `FOR UPDATE` locking read so they see the concurrently committed order instead of the transaction's stale snapshot re-producing the same clashing number (`orderBuilder.js allocateOrderNumber`).
12. **Branch prefix escaped in the number-parsing regex** (`orderBuilder.js escapeRegex`) — a prefix with a regex-special character can no longer distort numbering.
13. **Parked (needs DB approval)**: Flyway migration adding an index on `service_visas.passport_number` to keep the locked duplicate re-check narrow as data grows.

Round 3 (2026-07-31, found by Mukesh during UI testing):

14. **Cost/invoice rows switched to the visa TOTALS convention** — the import was storing PER-PERSON amounts in `published_rate`/`net_rate`/`price` where the system (manual visa form, XO generation, supplier reports) expects the item TOTAL with quantity as display-only. Stored totals now match what the visa costing screen recalculates, so XO/supplier figures can no longer read half the real amount. Orders imported BEFORE this fix carry per-person stored rates — delete and re-import test orders rather than patching them.
15. **Per-visa × quantity display on the visa screens** (Mukesh's request, display-only — stored values and all math stay on totals): when every visa in an item shares one price/cost, the Cost tab shows "Cost per Visa" with "× N passengers = total", and the Price summary shows "Base Price per Visa" + "Quantity × N" rows above the All-Visas line. Mixed per-visa amounts (possible on manual orders) automatically fall back to the aggregated display so no misleading average is shown. Touched: `VisaCost.jsx`, `VisaPrice.jsx`, `AddService.jsx` (shared with manual visa orders — re-test one manual visa order).

## 10. Out of scope (this phase)

1. Visa Code matching or syncing with Visa Code Maintenance.
2. Merging different customers into one order (grouping is per customer within one file).
3. Urgency, Validity, Process Days, passport type, mobile, email, address — not in the import file; stay empty.
4. Editing/updating existing visa orders via re-import (import only creates new orders).
