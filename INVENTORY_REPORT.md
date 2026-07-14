# Inventory Report - Technical Documentation

**Version**: 1.49
**Date**: July 2026
**Author**: System Analysis
**Status**: Stable - Verified

**Changelog**:
1. **v1.49** — **Removed the `Itinerary` and `IATA` columns and narrowed `Passenger Name` to stop the right-side money columns overflowing off the A4 page.**

   **Why:** at 23 columns with 7 money columns and amounts in the millions (e.g. `2,518,219.00`), the wide numbers pushed `Total Cost` (and then `IATA`) off the right edge. Money can't be truncated, so width had to be reclaimed by dropping low-value columns.

   **What changed:**
   1. **`Itinerary` and `IATA` columns removed** entirely — from the row object, `data1Total`, `columnHeaders`, `columnWidths`, `excelColumnWidths`, `rightAlignKeys` (IATA), and the now-dead `itinerary` variable in the row loop. Down to **21 columns**.
   2. **`Passenger Name` narrowed** — `capWidths.passenger_name` `8em → 6em` (names wrap onto more lines inside the tighter cap; nothing truncated) and its `columnWidths` share cut from `5.3% → 4%`.
   3. **Freed width redistributed to the money columns** so million-value amounts fit without expanding past the page: `publish_fare → 8.4%`, `tax_amount → 7.5%`, `total_price → 9%`, `cost → 8.4%`. `columnWidths` still sums to exactly 100%.
   4. Font size and page size **unchanged** (still 14px on A4 landscape) — user chose to drop low-value columns and shrink a name rather than shrink the font or move to A3.
2. **v1.48** — **New `Total Price` column (the selling price), inserted immediately before `Total Cost` (now 23 columns).**

   **What it shows:** the customer-facing selling total from `invoice.total_price` (base price + fees + tax) — the same invoice whose number appears in the row's Invoice Number column. Like Total Cost, it is shown as **each passenger's share**: `invoice.total_price ÷ passenger count`. So a Visa billed at 82,725 for 3 passengers shows 27,575 per row and a section subtotal of 82,725. Verified: service 6933 → 27,575/row; Air service 6421 (88 pax, 10,652,400) → 121,050/row.

   **Wired through:** row object (`total_price` between `extra_charges` and `cost`), `columnHeaders` (`Total Price`), `numericTotalKeys` (grand total), `subtotalKeys` (per-section subtotal), the 2-decimal format loop, `rightAlignKeys`, `excelColumnWidths` (16), and `columnWidths` — which was re-normalised so all 23 columns still sum to exactly 100% (`total_price` = 6.5%, every other column shaved slightly). Excel columns derive from the row keys, so `Total Price` appears there automatically.

   **Notes:** where a service has several invoices the row uses the same `Invoices[0]` already used for the Invoice Number, so Price and Invoice Number always agree. 23 columns on A4 landscape is tight — if it reads cramped, the next lever is a smaller font or a wider page.
2. **v1.47** — **Money columns now show each passenger's *share* of the service cost, not the whole-service total repeated on every passenger row.**

   **Why:** the report renders one row per passenger, but cost is stored once per service. For a booking whose price is a whole-service total (e.g. Visa invoice KHIN00000002: 3 passengers, one price of 78,000, `cost.quantity = 1`), every passenger row showed the full 78,000, so the section subtotal was 3× too high.

   **What changed:** each money column (Publish Fare, Tax, Commission, WHT, Extra, Total Cost) is multiplied by `shareFactor = rawQty / totalPax` — see **Cost Calculation** for the full definition. `rawQty` is `cost.quantity` (or `totalPax` when quantity is null/0, i.e. treat as per-person); `totalPax` is the service's true passenger count, fetched via a new grouped `service_passenger` count keyed by `service_id`.

   **What changes vs. what doesn't:**
   - **Changes:** only whole-service-total bookings with more than one passenger — **261 non-Air** (Visa, Hotel, Car, Cruise, Car_Transfer, Misc) and **3 Air** across the DB. These were over-counting before. Verified: Visa service 6933 goes 78,000 → 26,000 per row (subtotal 78,000).
   - **Unchanged:** every single-passenger service, and every per-person booking (quantity = pax — the vast majority of Air, e.g. an 88-pax ticket stays 32,065/row; a null-quantity 3-pax Air stays 168,770/row). Earlier-verified Air numbers hold.
   - Rows round to 2 decimals, so an uneven split can leave a subtotal off by a cent.
2. **v1.46** — **Non-Air services now filter by their invoice Issue Date (`invoice_date`) instead of the ticket issue date.**

   **Why:** v1.45 let all service types through the root filter, but the **Ticket Issue Date** range filter still tested only `service.ticket_issue_date` — a column that is **NULL for 100% of non-Air services** (verified: Visa, Hotel, Hajj, Umrah, Tour, Insurance, Car, Cruise, Train, Misc all blank). A NULL date can never fall inside a range, so every non-Air row was dropped again the moment any date range was set, and the report reverted to Air-only in practice.

   **What changed:**
   1. **Date filter uses a fallback.** The range now compares against `COALESCE(Service.ticket_issue_date, (SELECT MAX(invoice_date) FROM invoices WHERE service_id = Service.id))`. Air rows (which have a ticket issue date) are filtered exactly as before; non-Air rows are filtered by their invoice's Issue Date. Done in SQL (a correlated subquery via `Sequelize.literal` + `Sequelize.where`) so the comparison runs in the DB timezone — a JS comparison would shift local-midnight invoice dates by a day. The root `where` was restructured from a bare `Op.or` to `{ [Op.and]: [ <the Op.or>, <date condition> ] }` so the two OR-groups AND together.
   2. **Ticket Date column.** `issue_date` now falls back to `invoice.invoice_date` when there's no ticket issue date, so non-Air rows show their invoice Issue Date instead of a blank cell.

   **Notes / limits:**
   - Air is byte-identical: verified old-filter vs new-filter Air counts match exactly (1633 = 1633 for company 1001 over 01 Jun–10 Jul 2026). `COALESCE` returns the real ticket date whenever it exists.
   - A non-Air service with **no invoice** has no fallback date, so it will not appear while a date range is active. This is the intended consequence of "filter by invoice Issue Date."
   - The ~26 Air rows that hold a ticket but no `ticket_issue_date` now filter by their invoice date instead of being silently excluded — a minor, arguably-better change.
   - Display uses the first invoice's date (`Invoices[0]`); the filter's correlated subquery uses `MAX(invoice_date)`. Identical in the normal one-invoice case.
2. **v1.45** — **Report is no longer Air-only. All service types now show, grouped into sections.**

   **What changed:**
   1. **Root filter relaxed.** The report was rooted on `service_passengers` where `ticket_number IS NOT NULL`. Only flights carry ticket numbers, so Visa / Hotel / Hajj / Umrah / Tour / Insurance / Car / Cruise / Train / Misc were silently dropped (e.g. 97% of Visa services, 94% of Hotel). The filter is now an `Op.or`: **Air still requires a ticket number** (its identity is unchanged, drafts stay hidden), but **every other type shows regardless**. Null-type rows are treated as non-Air (shown). Implemented as `{ [Op.or]: [ {ticket_number: {ne:null}}, {'$Service.service_type.type$': {ne:'Air'}}, {'$Service.service_type.type$': {is:null}} ] }`.
   2. **Sections.** After all row processing, `data1` is grouped by `product_type`. Each section gets a bold **heading row** carrying the section's product code(s) (`Air — Product Code: 12, 13`), the rows, then a bold **subtotal row** (Publish Fare / Tax / Commission / WHT / Extra / Total Cost summed for that section, labelled `Subtotal` in the XO Number column). One **Grand Total** still closes the report. Section order: Air first, then a fixed business order (Visa, Hotel, Insurance, Car, Car_Transfer, Cruise, Tour, Train, Hajj, Umrah, Miscellaneous), then any unrecognised type alphabetically.
   3. **PDF** needed no template change — `report1.ejs` already renders `groupHeading` and `_isSubtotal` rows. **Excel** was updated: columns are now derived from the first *real* data row (not `data1[0]`, which is now a heading), heading rows render as a merged bold blue band, and subtotal rows render bold on light grey.

   **Notes / limits:**
   - Air-only columns (Ticket Num, Air-Code, PNR, Itinerary, Departure/Arrival, IATA) are blank on non-Air rows. Money columns fill in because every type has a cost record.
   - The `status` column still shows `Ticketed`/`Refunded` for non-Air rows (computed from refunds, same as Air). Not changed — out of scope for this request.
   - Grain is still one row per `service_passenger`; a service with no passenger row does not appear.
2. **v1.44** — **Readability pass: row banding, repeating header, tabular figures, Inter font stack.** All scoped to `.compact-table`, which only this report uses.

   1. **Row banding** — `#F7F7F7` on even `<tbody>` rows. At 22 columns this is the single biggest legibility gain; it lets the eye track across a row. The rule beats the generic `tr:nth-child(even) { transparent }` on specificity, and explicitly excludes `.total-row` (the Grand Total lives inside `<tbody>` and would otherwise get banded).
   2. **Repeating header** — `thead { display: table-header-group; }`. Without it the header prints once and every later page arrives with 22 unlabelled columns. Also `page-break-inside: avoid` on body rows, so a wrapped multi-line row is never split across pages.
   3. **Tabular figures** — `font-variant-numeric: tabular-nums` + `font-feature-settings: "tnum"`. Arial's digits are already fixed-width; **Inter's are not**, so the amount columns would misalign vertically without this.
   4. **Font stack** — `'Inter', 'Arial', 'Helvetica', sans-serif`.

   ⚠️ **Inter is not installed** on the machine that runs wkhtmltopdf (verified: 175 font files, none Inter). Naming it in CSS is therefore a **no-op today** — the report still renders in Arial. To actually use Inter, either install it on every server that generates PDFs, or embed it as a base64 `@font-face` in the template. Not done here; neither should happen silently.

   **Sizing context:** at `zoom: 0.85` and 96 DPI, printed size is `px × 0.75 × 0.85`. The current 14px/15px is **8.9pt / 9.6pt**, which is squarely the norm for a dense financial register (7–8pt for operational tables, 9–10pt for financial statements). The font is not the readability problem; the column count is.
2. **v1.43** — `customer_name` added to `wrapKeys`, so **both free-text columns now wrap instead of truncating**. Caps unchanged (`customer_name: 7em`, `passenger_name: 8em`), so column widths and the page layout are untouched — wrapping costs row height only. Customer names contain spaces, so they break at word boundaries naturally. A row can now be tall because of either name, so more rows run to two or three lines.
2. **v1.42** — **`Passenger Name` wraps instead of truncating.** New additive template flag **`wrapKeys`** (`['passenger_name']`). Keys listed in it render their `capWidths` div as `white-space: normal; word-wrap: break-word` instead of `nowrap` + ellipsis. The `8em` cap is unchanged — it still fixes the column width, so `Total Cost` and `IATA` stay on the page; the name now flows onto extra lines inside that cap rather than being cut. Rows holding long names get taller.

   The **zero-width space after each `/`** is reinstated (removed in v1.34 when wrapping was turned off). Passenger names are `SURNAME/GIVENNAME` with no space, so `MANSOOR/MUHAMMAD` is a single 16-character token; without a break opportunity a narrow column chops it mid-word (`MANSOOR/M` + `UHAMMAD`). The name itself is unchanged — only where a line *may* wrap. The value is HTML-escaped before the entity is injected, since it is user-entered data rendered via `<%-`.

   `customer_name` still truncates with `…` (not requested to change).
2. **v1.41** — **`Total Cost` and `IATA` restored. `capWidths` must be an absolute value, never `100%`.**

   v1.38 changed the ellipsis div from a fixed `88px` to `width: 100%`, on the reasoning that it should fill whatever width the column was given. That is wrong, and it is the regression that pushed the last two columns off the page:

   1. `width: 100%` resolves against the column.
   2. The column sizes itself to the div's content.
   3. The two chase each other, so the name renders at full length — `PAK SUZUKI MOTOR CO. LTD.` (24 chars) and `KHAN/MUHAMMAD IBRAHIM SAEED MR` (30 chars) in columns nominally 5.9% and 5.7% wide.
   4. The table outgrows the page and `cost` / `iata_no` are clipped off the right edge.

   The div's width **is** the cap — it is the only thing constraining those two columns. Restored as absolute values in `em` so they track the font: `customer_name: '7em'`, `passenger_name: '8em'`. That reclaims ~170px, against the ~160px `Total Cost` + `IATA` need.

   **Never set `capWidths` to a percentage.** A percentage makes the cap self-referential and silently disables it.
2. **v1.40** — **Fixed: `Total Cost` and `IATA` were being clipped off the right edge of the page.**

   **Cause — a latent bug, not an A4 problem.** A CSS `width` sets the **content** box by default; padding is added *on top*. Each column carries 2px of padding per side, so across 22 columns the table rendered at **100% + 88px** and overflowed the page. Whatever sat at the right edge — `Total Cost`, then `IATA` — was clipped away. The column percentages summed to exactly 100% the whole time; they were simply never the full story.

   This bug existed on A3 and A2 too. There the same 88px was a smaller fraction of a much wider page, so the overflow amounted to less than one column and nothing visibly vanished. Narrowing to A4 made those 88px worth two whole columns.

   **Fix:** `box-sizing: border-box` on `.compact-table th, td`, so the declared percentage includes padding. 100% now means 100% at any page size. Scoped to `.compact-table`; no other report is affected.

   **Rule of thumb:** if a column disappears off the right of this report, do **not** start shaving percentages. Check that they still sum to 100% and that `box-sizing: border-box` is intact.
2. **v1.39** — **Page set to A4 landscape (user preference). Font necessarily dropped to 14px/15px.**

   A4 landscape is 297mm wide — about **70% of A3** and **half of A2**. The 22 columns always fill the page exactly (`table-layout: fixed`, percentages summing to 100%), so squeezing the page squeezes every column with it. The font must shrink in step or every column truncates: `td` 28px → **14px**, `th` 29px → **15px**.

   Cell padding also cut `2px 4px` → **`2px 2px`**. At this font size the fixed 8px of horizontal padding consumed ~14% of each column (vs ~9% on A3), so trimming it returns roughly 7% of the page to text.

   Set in **both** `createPdf(html, true, { pageSize: 'A4' })` and the template's `pageSize` flag — wkhtmltopdf ignores CSS `@page` when its own option is set.

   **Known tension:** this reverses the readability goal of v1.38. 22 columns on A4 cannot be both complete and comfortably legible. The only way to get a larger font at this page size is **fewer columns** — `Air-Code`, `P-Type`, `P-Code` and `IATA` together hold ~9.6% of the width and carry the least information.
2. **v1.38** — **Page widened A3 → A2 landscape; font raised to 28px/29px for readability.** *(Superseded by v1.39, which moved to A4.)*

   **Why the page had to change.** The table is `table-layout: fixed` at `width: 100%` with percentages summing to 100%, so **narrowing a column never creates space — it hands that space to another column**. Raising the font makes every column need more room simultaneously, and there is nowhere to take it from. At 22 columns, 21px was the ceiling on A3. Only two levers change overall density: font size and page size. Reclaiming the genuine slack in `Air-Code`, `P-Type`, `P-Code`, `WHT` and `IATA` frees just ~1.8% — nowhere near enough.

   A2 landscape is 594mm wide (+41% over A3), which funds `td` 21px → **28px** and `th` 22px → **29px** with room to spare. Set in **both** `createPdf(html, true, { pageSize: 'A2' })` and the template's `pageSize` flag — wkhtmltopdf ignores the CSS `@page` when its own option is set, so the two must always move together.

   **Widths rebalanced** (still exactly 100%): the short-value columns gave up width to the two long free-text ones — `airline`/`product_type`/`p_code` 2.6% → 2.2%, `wht` 2.8% → 2.6%, `iata_no` 2.8% → 2.4%; `customer_name` 5% → **5.9%**, `passenger_name` 4.8% → **5.7%**.

   **`capWidths` fixed.** It was a hard `88px` on `passenger_name`, sized for the old A3/21px layout. On a wider page that div would sit *inside* a much wider column and leave dead space. It is now `100%` for both `customer_name` and `passenger_name`, so the ellipsis div fills whatever the column is at any page or font size.

   **Printing note:** A2 needs A2 paper or "scale to fit". On screen it just renders larger.
2. **v1.37** — **New `Customer Name` column, inserted directly after `Ticket Date` (column 2 of 22).**

   Source is `service.Order.customer.customer_name`. The `order` include previously loaded only `user` and `branch`; `customer` is now added as a **LEFT JOIN** (`required: false`) so an order with a null `customer_id` can never silently drop rows. All 1,426 orders currently have a customer, so no row renders blank today.

   Column order is driven by key order in `passengerObj` **and** `data1Total` — both were updated, and Excel derives its columns from `data1[0]`, so PDF and Excel stay in step.

   **Widths re-normalized.** The PDF percentages must total exactly 100%, so adding a 22nd column required taking 5% from the existing 21. Relative proportions were preserved; the new set sums to exactly 100.0%. Excel gets `customer_name: 24`.

   **Customer names truncate with `…`** at 5% of the page: they average 20 chars, reach 54, and 43% exceed 20 chars. Same behaviour as Passenger Name.
2. **v1.36** - **Inventory PDF is full-page again.** Replaced shrink-fit/spacer sizing with a fixed full-width table and explicit percentage widths that sum to 100%. Passenger Name stays narrow at 5% and its text is capped by an inner `88px` ellipsis div, so the visible report reaches the right side of the page without making Passenger Name absorb the blank space.
1. **v1.35** - **Passenger Name column border now collapses to the 6em cap.** `passenger_name` is now included in `shrinkFitKeys`, and `report1.ejs` applies cap sizing to the `<col>`, `<th>`, normal `<td>`, and Grand Total `<td>`. `.compact-table` now uses `width: 1px` instead of `width: auto`, and Inventory passes `trailingSpacer: true` so any wkhtmltopdf leftover table width is absorbed after IATA instead of inside Passenger Name. The text is still truncated inside the fixed-width div, but the column itself no longer absorbs spare page width.
1. **v1.34** — **Passenger Name no longer wraps. Names are now truncated with `…`.** Per user request the column is single-line and narrowed `7em` → **`6em`**.

   **This is a deliberate loss of information.** On a fixed-width column text either wraps or truncates; there is no third option. With wrapping off, most names on a typical report are cut: `MR ALI ASGHAR` → `MR ALI A…`, `MANSOOR/MUHAMMAD ZAID BIN MR` → `MANSOOR/…`. The upside is single-line rows, so the report is far shorter.

   Template flag renamed `wrapWidths` → **`capWidths`** (the old name no longer described the behaviour). The inner `<div>` is now `white-space: nowrap; overflow: hidden; text-overflow: ellipsis`. The v1.33 zero-width-space slash-break logic and its HTML-escape helper were removed — both are inert once wrapping is off, and the value returns to EJS's own `<%=` escaping.

   `6em` is the single knob: raise it to reveal more of each name at the same row height.
2. **v1.33** — **Passenger names no longer break mid-word** (superseded by v1.34, which removed wrapping entirely). At `7em` the column chopped words: `MANSOOR/MUHAMMAD` rendered as `MANSOOR/M` + `UHAMMAD`, because passenger names are `SURNAME/GIVENNAME` with no space — a single 16-character token. Fixed at the time by injecting a zero-width space after each `/`.
2. **v1.32** — `passenger_name` cap narrowed `9em` → **`7em`** (user request). Names wrap onto more lines and those rows get taller; nothing is truncated. Single-value change in `wrapWidths`; no other column, and no font size, touched.
2. **v1.31** — **Passenger Name hard-capped with an inner fixed-width div. The v1.30 `width: 170px` was silently ignored.**

   **Why tightening the other columns made Passenger Name grow.** Every other column is `white-space: nowrap`, so its width is *locked* to its text — it can neither give nor take. `passenger_name` is the only column that can wrap, so it is the only *flexible* one. The table algorithm gives each locked column exactly what it needs, then dumps **all remaining page width into the single flexible column**. Every pixel saved on the other 20 columns therefore lands in Passenger Name. Tightening them and fattening it are the same action.

   **Why `width: 170px` did nothing.** A `width` on a table cell is a *suggestion*, not a limit. When the table algorithm has surplus width to distribute, it overrides the declared width. The column ignored 170px and took roughly 40% of the table.

   **Fix.** The value is now rendered inside a `<div>` with an explicit width (new additive template flag **`wrapWidths`**, replacing `wrapKeys`). A div's width *is* a hard limit, so the column's max-content is capped and it can no longer absorb surplus. Set in **`em`** (`9em`), not px, so the cap scales with the font instead of breaking on every font change.

   **Consequence — this is expected, not a bug.** With no column able to absorb surplus, the table no longer fills the page and a real empty margin appears on the right. That margin is the *measurement* of how much headroom is left for a larger font. Font held at 21px/22px this round: raising it while simultaneously capping the column would have risked overshooting into truncation.
2. **v1.30** — **Passenger Name capped; table no longer stretches; font raised to 21px/22px.** The `170px` width in this version never took effect — see v1.31.

   After v1.29 every column hugged its data except `passenger_name`, which looked enormous. That was structural, not a bug: it was the **sponge**. The table was `width: 100%`, so the leftover page width had to go somewhere, and with no width of its own `passenger_name` absorbed all of it.

   **Fix, in two parts:**
   1. `passenger_name` gets an explicit **170px** and wraps onto a second line. It is the only column whose content varies wildly (up to 75 chars), so left flexible it swallows the page.
   2. `.compact-table` gets **`width: auto`**. The table is now exactly as wide as its contents — leftover space collects as a right-hand page margin instead of inflating any column. This is what removes the need for a sponge at all.

   **Font raised** to `td` **21px** / `th` **22px** (from 19/20). Narrowing `passenger_name` freed roughly 100px, and that headroom is what the larger font spends. Estimated table width ≈ 96% of the A3 landscape page.

   **Ceiling warning:** the font is now close to its maximum. Under automatic layout there is no ellipsis — if the contents exceed the page the table overflows and the right-hand columns (IATA first, then Total Cost) are clipped off the sheet entirely. Lower the font if that happens.
2. **v1.29** — **Switched to shrink-to-fit columns, copying the Daily Sale Report's technique. All manual width values deleted.**

   **Why the previous approach could never work.** `.compact-table` used `table-layout: fixed`, which forces the declared column widths to fill the page exactly. Under that model the widths always sum to 100% of the sheet, so empty space could only be *moved* from one column to another, never removed. v1.24 through v1.28 were all variations on moving it around.

   **What the Daily Sale Report does differently** (`daily-sale-report.ejs`): its table uses the browser's **automatic** layout, and marks tight columns with `white-space: nowrap; width: 1%`. Under automatic layout `width: 1%` does not mean one percent — it means *"collapse this column to its content width."* A few flexible columns (Client Name, Supp Name, Pax) then absorb all the leftover width. That is why `Invoice No.` and `Inv. Date` hug their data there.

   **Applied here:**
   1. `table-layout: fixed` removed from `.compact-table` (verified: no other report uses that class).
   2. New additive template flag **`shrinkFitKeys`** — every listed column gets `width: 1%` and collapses to its widest value. 20 of the 21 columns are listed.
   3. New additive template flag **`wrapKeys`** — `passenger_name` only. It is deliberately *excluded* from `shrinkFitKeys`, so it carries no width, stays flexible, and **absorbs all leftover table width**. Its cell wraps instead of widening the column on long names (75 chars max in the data).
   4. The `columnWidths` percentage map is **deleted**. Widths are no longer declared anywhere for this report.

   **Consequences:**
   - Column headers wrap onto two lines wherever the header is wider than the data (`Ticket Date` → `Ticket` / `Date`). This is required — the wrap is what lets the column hug its values.
   - The Grand Total row's large sums now **set** the width of the amount columns instead of being truncated by them.
   - **New failure mode:** under automatic layout there is no ellipsis safety net. If the columns' combined natural width exceeds the page, the table overflows and the right edge is clipped outright. If that happens, **lower the font** (`.compact-table`) — do not reintroduce widths. Daily Sale runs 28 columns at 8px; this report runs 21 at 19px.
2. **v1.28** — **The three date columns tightened; freed space given to Passenger Name.**

   `Ticket Date`, `Departure Date` and `Arrival Date` all hold the same string (`01 Jun 2026`, `DD MMM YYYY`) and all carried the same 4.60% share. Two compounding causes left a visible gap under the dates:
   1. The share was over-generous. Shares were computed from Arial character-width tables, which **over-estimate digit-heavy strings**. A date is 6 digits, 2 spaces and a 3-letter month — nearly all narrow glyphs. Measured against a rendered PDF the value filled only ~76% of its cell.
   2. The **bold 20px header was wider than the 19px regular data**, and the column was wide enough for it to sit on one line, holding the column open.

   Each date column is now **3.95%** — the data width plus padding. The headers now wrap onto **two lines by design** (`Ticket` / `Date`); that wrap is what allows the column to hug the value. The freed **1.95%** went to `passenger_name` (10.86% → **12.81%**), the one column that genuinely needed it: 530 of 9,970 names exceed 30 chars and were clipping.

   Shares still sum to exactly 100.00%.
2. **v1.27** — **Font raised to 19px/20px. On this report, "narrow the columns" and "enlarge the font" are the same action.**

   **The key fact.** `.compact-table` sits inside a `<table>` styled `width: 100%`, on a fixed A3 landscape page. **The 21 columns therefore always sum to exactly the page width — no width value can make the table narrower.** Shrinking one column only donates that space to the others. Empty space inside a cell is *never* a width problem here; it is small text in a normal-sized column. The only way to close the gap is to grow the text until it fills the cell.

   Because v1.26 made every column's share proportional to its data, a single font increase now tightens all 21 columns evenly and simultaneously. PDF `td` 15px → **19px**, `th` 16px → **20px** (both the base rule and the `@media print` rule).

   **Why the earlier pixel math never matched the render.** `psback/services/pdf.js:67` sets `zoom: 0.85` for all landscape PDFs, and `dpi: 96` with `disableSmartShrinking: true`. The 0.85 zoom means the CSS viewport is `414mm ÷ 25.4 × 96 ÷ 0.85 ≈ 1841 CSS px`, not the ~1565px that a naive A3-at-96dpi calculation gives. Any future width/font reasoning on this report must account for that zoom factor.

   Excel is unchanged this round (header 16, data/Grand Total 15 from v1.26) — it has no page-width constraint, so it has no equivalent problem.
2. **v1.26** — **Column widths switched from px to percentages, and the report font increased.**

   **Root cause of the persistent blank space.** The table is `width: 100%` with `table-layout: fixed`. When the column widths are given in px and their sum is less than the page width, CSS redistributes the leftover proportionally back into every column. So *no px value could ever stop a column from stretching* — v1.24 and v1.25 both tried and both failed. Lowering a px value only handed that space to the other columns.

   **Fix.** `columnWidths` is now expressed in percentages that sum to exactly **100%**. With no remainder there is nothing to redistribute, so each column renders at exactly its share, independent of page size, margins, and wkhtmltopdf's DPI (which is what made the earlier px math unverifiable).

   **How each share was derived.** Proportional to the widest thing the column must hold — its longest real value (measured against live data), or its longest *unbreakable* header word, whichever is wider. `th` is `white-space: normal` so headers wrap at spaces, but a single long word cannot break. Two columns are therefore **header-bound, not data-bound**:
   - `Itinerary` — the word "Itinerary" (9 chars) is wider than the data `KHI/LHA` (always 7 chars; `city_codes.code` is 3 chars in all 9,513 rows).
   - `Commission` — a single 10-char word that cannot wrap.

   **Fonts raised** — PDF `td` 13px → **15px**, `th` 14px → **16px** (in both the base rule and the `@media print` rule; see v1.21 for why both are required). Excel column header 14 → **16**, data and Grand Total rows 13 → **15**, with `excelColumnWidths` scaled up so the larger glyphs don't clip.

   The three columns the user asked to tighten all shrank: Itinerary `3.96% → 3.34%`, Supplier No `6.84% → 6.41%`, XO Number `6.84% → 6.08%`.
2. **v1.25** — **Six columns resized to fit their real data.** `Ticket Date`, `Ticket Num`, `PNR`, `Itinerary`, `Supplier No`, `XO Number` were all truncating with `…` because v1.24 raised the data font to 13px while tightening the widths. Column lengths were measured against live data before choosing values (Ticket Date 55→82px, Ticket Num 60→88px, PNR 50→79px, Itinerary 40→55px, Supplier No 55→95px, XO Number 60→95px). Superseded by v1.26, which replaced all px values with percentages.
2. **v1.24** — Multiple iterations rolled into one entry:
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

The Inventory Report lists sold services per passenger, **grouped into a section per service type** (Air, Visa, Hotel, Hajj, Umrah, Tour, Insurance, Car, Cruise, Train, Miscellaneous). Each section has a heading with its product code(s), the rows, and a subtotal; one Grand Total closes the report. For each row it shows the product, invoice, customer, passenger, and the published/commission/tax/cost breakdown — plus airline/ticket details for Air.

**Air is still ticketed-only** (an Air row requires a `ticket_number`); every other service type appears regardless, because only flights carry ticket numbers (see Data Source). Prior to v1.45 the report showed Air only.

It answers: *"What have we sold, broken down by service type, and what does each line look like financially?"*

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

**Filter retention & inline preview**:
- Selected filters are retained after generating (module-level `cachedFilter`); returning to the page keeps prior selections. A full page refresh resets to `DEFAULT_FILTER`.
- PDF is rendered inline below the filters (iframe → `{VITE_API_URL}/document/view/{report_number}`) with a **Preview** button (opens the full PDF in a new tab). No navigation away from the page. Excel still downloads and clears the inline PDF.

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

Root query: `db.service_passenger.findAll(...)`. As of v1.45 the root filter is an `Op.or`, not a flat `ticket_number IS NOT NULL`: a row is kept if it **has a ticket number** OR its **service type is not Air** (or has no type). Net effect — Air is ticketed-only; every other type is shown regardless. The `$Service.service_type.type$` reference works because `service_type` is always eager-loaded (see below).

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
      3. `customer` (`required: false` — LEFT JOIN) — for `customer.customer_name` (Customer Name column). Deliberately not `required`, so an order with a null `customer_id` can never silently drop rows from the report.

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

Applied to `COALESCE(service.ticket_issue_date, MAX(invoice.invoice_date))` — i.e. the ticket issue date for Air, falling back to the invoice's Issue Date for services that have no ticket date (all non-Air types). See v1.46.

1. Operators `=`, `<`, `<=`, `>`, `>=`, `<>` → compared against `new Date(startDate)`.
2. Operator `between` → compared against `[new Date(startDate), new Date(endDate)]`.
3. `dateFilter === "blank"` (UI default) is skipped.
4. The comparison is built as `Sequelize.where(Sequelize.literal('COALESCE(...)'), { <op>: <value> })` and pushed into the root `Op.and` array, so it runs in SQL (DB timezone). A non-Air service with no invoice has no date on either side of the `COALESCE` and is excluded while a range is active.

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
| 1 | Ticket Date | `issue_date` | `service.ticket_issue_date`, falling back to `invoice.invoice_date` | Ticket issue date for Air; for non-Air services (no ticket date) shows the invoice's Issue Date instead of blank. Formatted `DD MMM YYYY`. See v1.46. |
| 2 | Customer Name | `customer_name` | `service.Order.customer.customer_name` | Customer the order was booked for. Added v1.37. Empty string if the order has no customer (none currently do). Truncates with `…` — names average 20 chars and reach 54. |
| 3 | Ticket Num | `ticket_no` | `service_passenger.ticket_number` | Ticket number of the passenger. |
| 4 | Air-Code | `airline` | `service.airline_code.airline_ticket_prefix` | Airline ticket prefix (e.g. `PK`, `EK`). |
| 5 | PNR | `pnr` | `service.pnr` | GDS PNR (Passenger Name Record) — typically a 6-character alphanumeric code. |
| 6 | Status | `status` | computed from `service.refunds` | `"Ticketed"` if no printed refund covers this ticket, otherwise `"Refunded"` (no refund document number is appended). See "Filter Logic → Ticket Status" for the detection rules. |
| 7 | Invoice Number | `invoice_no` | `service.Invoice.invoice_number` (falls back to `service.Invoices[0].invoice_number`) | Invoice number that this ticket is billed on. |
| 8 | P-Type | `product_type` | `service.service_type.type` | Product type description. |
| 9 | P-Code | `p_code` | `service.service_type.product_code` | Product code (e.g. 12, 13 for air products). |
| 10 | Passenger Name | `passenger_name` | `passenger.passenger_name` | Passenger's full name. |
| 11 | Departure Date | `dep_date` | `service_flights[0].departure_date` | Departure date of the first flight leg, `DD MMM YYYY`. |
| 12 | Arrival Date | `arr_date` | `service_flights[last].arrival_date` | Arrival date of the last flight leg, `DD MMM YYYY`. |
| 13 | Supplier No | `supp_no` | `service.supplier.supp_no` | Supplier code for the ticket issuer (e.g. BSP supplier). Empty string if service has no supplier. |
| 14 | XO Number | `xo_number` | `cost.Document.document_number` with explicit `where: { document_type: 'costing' }` on the include | XO (costing) document number ONLY. Empty if the cost has no costing document. The explicit `where` prevents invoice numbers from leaking when an invoice happens to share the cost's id. |
| 15 | Publish Fare | `publish_fare` | `cost.published_rate` | Published (gross) rate per unit. |
| 16 | Tax Amount | `tax_amount` | Σ `cost.cost_taxes[i].tax_amount` | Sum of regular cost taxes (excludes WHT). |
| 17 | Commission | `commision` | `cost.published_rate × cost.commission / 100` | Commission amount (from agent's cost). The key has a typo (`commision`) preserved for compatibility; the override map forces the header to display "Commission". |
| 18 | WHT | `wht` | `commissionAmount × cost.sst / 100` | Withholding tax — WHT percent (stored in `cost.sst`) applied to the commission amount. |
| 19 | Extra Charges | `extra_charges` | `cost.free_of_cost` | Extra charges on the cost (the `free_of_cost` field is repurposed for extra charges). Shown as its own column AND folded into Total Cost (as of v1.23). |
| 20 | Total Price | `total_price` | `invoice.total_price ÷ passengers` | Selling price charged to the customer (base price + fees + tax), taken from the same invoice whose number is shown on the row. Shown as each passenger's share; the section subtotal equals the invoice's full selling total. Added v1.48. |
| 21 | Total Cost | `cost` | See formula below | Unit cost after commission deduction, plus taxes and WHT. Per-passenger share (v1.47). |

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
| `customer_name` | `Customer Name` |
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
| `supp_no` | `Supplier No` |
| `xo_number` | `XO Number` |
| `publish_fare` | `Publish Fare` |
| `tax_amount` | `Tax Amount` |
| `commision` | `Commission` |
| `wht` | `WHT` |
| `extra_charges` | `Extra Charges` |
| `total_price` | `Total Price` |
| `cost` | `Total Cost` |

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
10. `cost = costPerUnit × shareFactor` *(per-passenger share — see below; v1.47)*

**Per-passenger share (`shareFactor`, v1.47)**: The report renders one row per passenger, but a service's cost is stored once. Each money column (Publish Fare, Tax, Commission, WHT, Extra, Total Cost) is multiplied by `shareFactor = rawQty / totalPax`, where:
- `totalPax` = true passenger count of the service (counted fresh, includes non-ticketed passengers so Air group tickets divide correctly).
- `rawQty` = `cost.quantity` when it is a positive number, otherwise `totalPax` (a null/0 quantity is treated as per-person → factor 1, leaving the row unchanged).

Net effect:
- **Per-person prices** (quantity = passenger count, e.g. most Air group tickets): `pax / pax = 1` → unchanged.
- **Whole-service totals** (quantity = 1, multiple passengers, e.g. most Visa/Hotel): `1 / pax` → the total is split evenly across passengers, so each row shows one passenger's share and the section subtotal equals the true booking total.
- **Single-passenger services**: `1 / 1 = 1` → unchanged.

Only bookings stored as a whole-service total with more than one passenger change (261 non-Air + 3 Air across the DB at time of writing). Rows round to 2 decimals, so a total that doesn't divide evenly can leave the subtotal off by a cent.

**Plain-language**: Start with the published rate, strip out the commission the agent earns, add back the taxes that are passed through, add withholding tax on the commission portion, and add Extra Charges — then, if the price was for the whole booking, divide it across the passengers so each row shows that passenger's share. Now matches Airline Sales / Daily Sale reports.

---

## Sections, Subtotals & Grand Total

As of v1.45 the report is grouped into one **section per service type**.

**Section heading** — a bold full-width row before each service type's rows, reading `<Type> — Product Code: <codes>` (e.g. `Air — Product Code: 12, 13`). Built as a `{ groupHeading: '…' }` row in `data1`.

**Per-section subtotal** — a bold row after each section's rows, summing `publish_fare`, `tax_amount`, `commision`, `wht`, `extra_charges`, `total_price`, `cost` for that section only, labelled `Subtotal` in the `xo_number` column. Built as an `{ …, _isSubtotal: true }` row (all 23 keys present, in order, so columns align).

**Section order:** Air first, then Visa, Hotel, Insurance, Car, Car_Transfer, Cruise, Tour, Train, Hajj, Umrah, Miscellaneous; any unrecognised type trails alphabetically.

**Grand Total** — a single row still closes the whole report (both PDF and Excel).

1. Label `"Grand Total"` in the `xo_number` column; `Subtotal` labels sit in the same column per section.
2. Numeric columns summed across **all** data rows (not per-section): `extra_charges`, `wht`, `publish_fare`, `tax_amount`, `commision`, `total_price`, `cost`.
3. Non-numeric / ID columns left blank.
4. In PDF: `data1Total` variable, bold, `.total-row` class.
5. In Excel: bold row, light-grey fill (`#EFEFEF`), after the last section. Section headings render as a merged bold blue band; subtotals bold on light grey (`#F2F2F2`).
6. The Grand Total is computed **before** grouping, from the flat row set, so it is unaffected by section boundaries.

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
   1. `pageSize: 'A4'` — A4 landscape, 297mm wide (user preference, v1.39). Note: actual page size is also explicitly forced via `createPdf(html, true, { pageSize: 'A4' })` because wkhtmltopdf ignores CSS `@page` when its own `pageSize` option is set. **The two must always be changed together.** Page width and font size are the only two levers on this report's density — see flag 5.
   2. `alignAllLeft: true` — every header and data cell renders `text-align: left` by default.
   3. `rightAlignKeys: ['publish_fare', 'tax_amount', 'commision', 'wht', 'extra_charges', 'cost', 'iata_no']` — per-key override that right-aligns the 6 amount columns plus IATA even though `alignAllLeft` is true. The template's `cellAlign` helper checks this list first.
   4. `prominentTitle: true` — adds a `.prominent` class to the `.page-subtitle` div, bumping the font size to 16px so the report title stands out.
   5. `compactTable: true` — adds a `.compact-table` class to the `<table>` element. Sets `th` font-size to **15px** and `td` font-size to **14px**, tightens padding to 2px/2px, and uses `width: 100%; table-layout: fixed`. The fixed table layout is deliberate: the 22 visible columns fill the page according to `columnWidths` instead of letting wkhtmltopdf assign surplus width to one flexible column. Both the base rule and the `@media print` rule must carry the same font sizes — wkhtmltopdf does not reliably apply `@media print`, so a mismatch silently leaks the screen rule into the PDF (see v1.21).

      **Tuning the density of this report:** the visible table is fixed to the full page width, so **narrowing a column never creates space — it hands that space to another column**. Only two things change the overall density: the font size, and the page size. If a value clips too aggressively, adjust that column's percentage and take the same amount from another column so the map still sums to 100%. If *everything* clips, the font is too large for the page.
   6. `columnHeaders` — per-key header override map for the 22 final display labels.
   7. `columnWidths` — explicit percentage widths for all 21 visible columns. The values sum to exactly 100%, so the table fills the page while each column keeps its assigned share. Removing `Itinerary` + `IATA` and narrowing `Passenger Name` (v1.49) freed width that went to the wide money columns so million-value amounts stop overflowing.

      Listed: `issue_date: 4.5%`, `customer_name: 4.9%`, `ticket_no: 4.5%`, `airline: 2.1%`, `pnr: 3.6%`, `status: 3.5%`, `invoice_no: 5.3%`, `product_type: 2.1%`, `p_code: 2.1%`, `passenger_name: 4%`, `dep_date: 4.5%`, `arr_date: 4.5%`, `supp_no: 5.5%`, `xo_number: 5.5%`, `publish_fare: 8.4%`, `tax_amount: 7.5%`, `commision: 3.9%`, `wht: 2.3%`, `extra_charges: 3.9%`, `total_price: 9%`, `cost: 8.4%`.

      **These must always sum to exactly 100%.** Adding or removing a column means re-normalizing every share — you cannot simply insert a new one.

      **They only add up to 100% because `.compact-table th, td` is `box-sizing: border-box`.** Without it a CSS `width` is the *content* width and cell padding is added on top, so the table silently renders wider than the page and the right-hand columns (`Total Cost`, then `IATA`) are clipped off. See v1.40.

   8. `capWidths` — `customer_name: '7em'`, `passenger_name: '6em'` (narrowed from 8em in v1.49 to reclaim width). The text is wrapped in `<div style="width: …; white-space: nowrap; overflow: hidden; text-overflow: ellipsis">`, because **wkhtmltopdf does not reliably ellipsize raw text sitting directly inside a `<td>`**.

      **The div's width is what actually caps these two columns** — they are the only free-text columns and, left alone, they size themselves to their content and push `Total Cost` and `IATA` off the page. It **must be an absolute value**; `em` is used so the cap tracks the font size. Never use a percentage: it resolves against a column that is itself sizing to the div's content, which makes the cap self-referential and silently disables it (see v1.41).

   9. `wrapKeys` — `['customer_name', 'passenger_name']`. Keys listed here render their `capWidths` div as `white-space: normal; word-wrap: break-word` instead of `nowrap` + ellipsis, so the value **wraps onto extra lines rather than truncating** (v1.42–v1.43). The cap still fixes the column width, so wrapping costs row height, never page width. A zero-width space is injected after each `/` in the value, because passenger names are `SURNAME/GIVENNAME` with no space and would otherwise break mid-word; customer names contain spaces and break naturally.

      **Names do not wrap and are truncated with `…` when they exceed the column** (v1.34, by user request). This is an accepted loss of information: on a fixed-width column, text either wraps or truncates.

      Excel is independent and still uses `excelColumnWidths` (chars): Ticket Date 16, Ticket Num 16, Air-Code 8, PNR 14, Status 8, Invoice Number 16, P-Type 8, P-Code 8, Passenger Name 24, Departure/Arrival Date 11, Supplier No 18, XO Number 18, Publish Fare 15, Tax Amount 15, Commission 15, WHT 12, Extra Charges 15, Total Price 16, Total Cost 16. (Itinerary and IATA removed in v1.49.)
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
7. **Fixed layout means column percentages must balance.** The Inventory PDF column map sums to 100%. If one column is widened, another must be narrowed by the same amount or the right edge can clip.
8. **Header words can still set a practical floor**: `th` is `white-space: normal`, so headers wrap at spaces, but a single long word cannot break cleanly. Very narrow columns may show cramped headers.
9. **`passenger_name` is 5% wide and its text is capped at `88px`.** Long passenger names are cut off with `…` — this is intentional (v1.34), not a defect.
10. **`.compact-table` is `width: 100%; table-layout: fixed`.** Restoring shrink-fit/auto layout can either make Passenger Name absorb spare width or make the visible table stop before the right side of the page.

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
