# AP Ageing Analysis Detail Report

**Status**: Stable — base-currency fix 2026-06-17
**Scope**: Supplier (accounts-payable) ageing, line-by-line per supplier.

## 1. What it is
1. Shows, per supplier, every **outstanding** payable document bucketed by how overdue it
   is: Current (within credit period), 1–30, 31–60, 61–90, 91–120, 121+ days, and a Total
   Outstanding column.
2. Three kinds of rows per supplier: **Costs (XO)**, **Debit Notes** (reduce payable),
   and **Advance Payments / supplier deposits** (reduce payable). Then a **Supplier
   Total** and a **GRANDTOTAL**.

## 2. Where it lives
1. **Controller**: `getAPAgeingAnalysisDetailReport` in
   `psback/controllers/report.controller.js`.
2. **Template (PDF)**: `psback/views/pages/reports/ap_ageing_analysis_detail.ejs`.
3. **Route**: `POST /api/report/getAPAgeingAnalysisDetailReport` (permission
   `Ap-Ageing-Analysis-Detail-Report`). Output `type`: `pdf` (default) or `excel`.
4. **Filters**: supplier (equal/between/blank), date, branch, `asOfDate`.

## 3. How amounts are calculated
1. **Cost amount** is rebuilt from the cost fields (same formula as the XO document):
   `published_rate − commission + free_of_cost + cost_taxes + WHT(SST)`, × quantity.
2. That local-currency amount is converted to the **company base currency** using the
   company's saved exchange rate, then **outstanding = cost amount − settled**.
   Fully-settled costs are skipped.
3. **Debit notes** and **advance payments** subtract from the payable (shown negative).
   Advance uses the deposit's `current_amount` (already in base currency). Debit notes are
   matched company-safely: normal DNs by `supplier_name` + company branches, **opening DNs**
   by `supplier_id` (opening DNs have `branch_id = NULL`, so the branch filter alone dropped
   them — fixed 2026-06-22).
4. **Due date** comes from the supplier's credit terms (`calculateDueDate`); the bucket is
   chosen by `daysOverdue = asOfDate − dueDate`.

## 4. Base currency (fixed 2026-06-17)
1. The company **base currency** is the `to_currency` configured for that company in the
   `currencies` table (e.g. company `1010` has `EUR → USD` ⇒ base **USD**); default `PKR`.
2. **Bug before the fix**: the report converted every amount to **PKR** (hard-coded) and
   the exchange-rate lookup had **no company filter** — so a foreign cost was multiplied
   by *another* company's `from→PKR` rate. A EUR cost of 451.04 showed as ~**146,617**
   (451.04 × some other company's EUR→PKR ≈ 325) instead of **518.70** USD, while the
   USD advance was left un-converted — mixing currencies in one total.
3. **Fix**: the rate query is now `to_currency = <baseCurrency> AND company_code =
   <this company>`, so EUR→USD uses **1.15** and the cost reads **USD 518.70**. The
   `baseCurrency` is also shown on the report ("All amounts in USD") and defaults to
   `PKR` for Pakistani companies (unchanged for them).
4. **Note**: amounts already in the base currency (advances/deposits' `current_amount`,
   debit notes' `amount`) are taken as-is — they are not re-converted.

## 5. Header / output
1. Header: company name/address, Report ID (`APDET<timestamp>`), Print date/time, Print
   By, the filters, and now **(All amounts in `<base currency>`)**.
2. Excel export mirrors the same data; the currency note is currently shown on the PDF.

## 6. Related / still pending
1. The **AP Ageing Summary** and the **AR Ageing** reports use the same hard-coded-PKR /
   un-company-scoped rate pattern (`to_currency = 'PKR'` with no `company_code`) and have
   **not** been migrated yet — apply the same fix when needed.
2. Base-currency rules across the app: see `INVOICE_DOCUMENT_CURRENCY.md`.
