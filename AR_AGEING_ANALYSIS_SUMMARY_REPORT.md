# AR Ageing Analysis Summary Report - Technical Documentation

**Version**: 1.9
**Date**: 2026-07-07
**Author**: System Analysis
**Status**: Stable — Manual JE match now company-scoped (v1.9); Total OverDue / Total Deposits / Total Credits columns added, informational only (v1.8); Customer column split into Customer Name + Cust No (v1.7); Branch "Is Not Blank" no longer restricts (v1.6); Manual JE settlement subtracted from outstanding (v1.5); PDF/Excel parity (v1.4), Detail/Summary parity (v1.3)

---

## Overview

The AR Ageing Analysis Summary Report provides a condensed view of accounts receivable aging per customer. Unlike the Detail Report (which shows individual invoices), this report shows one row per customer with aggregated aging bucket totals and net outstanding amounts.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/ARAgeingAnalysisSummaryReport.jsx`

**Filters**:
- Customer Number: isNotBlank, isBlank, isEqual, between (LiveComboBox via `/customer/getCustomers`)
- As Of Date: Single DateInput (default: today)
- Invoice Date: isNotBlank, isBlank, =, <, <=, >, >=, <>, between (DateInput)
- Branch: isNotBlank, isBlank, isEqual, between (Combobox, client-side)

> **Note (v1.6)**: For **Branch**, the default "Is Not Blank" applies **no restriction** on the backend — customers with no branch assigned are still included (parity with Detail report v1.10). "Is Blank", "Is Equal", and "Between" behave as before.

**Output**: PDF or Excel via dropdown button.

**Filter retention & inline preview**:
- Selected filters are retained after generating (module-level `cachedFilter`); returning to the page keeps prior selections. A full page refresh resets to `DEFAULT_FILTER` (with `asOfDate` recomputed to today on mount).
- PDF is rendered inline below the filters (iframe → `{VITE_API_URL}/document/view/{report_number}`) with a **Preview** button (opens the full PDF in a new tab). No navigation away from the page. Excel still downloads and clears the inline PDF.

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getARAgeingAnalysisSummaryReport` (line ~16445)

### PDF Template

**File**: `psback/views/pages/reports/ar_ageing_analysis_summary.ejs`

---

## Report Layout

### Columns

| Column | Description |
|--------|-------------|
| Customer Name | `customer_name` |
| Cust No | `customer_number` |
| Average Days | Average days overdue across invoices |
| Current | Invoices within credit period |
| 1-30 Days | 1-30 days overdue |
| 31-60 Days | 31-60 days overdue |
| 61-90 Days | 61-90 days overdue |
| 91-120 Days | 91-120 days overdue |
| 121+ Days | Over 120 days overdue |
| Total OverDue | Sum of the five overdue buckets (1-30 + 31-60 + 61-90 + 91-120 + 121+); Current excluded — informational only |
| Total Deposits | Customer deposit credit (`deposit_credit`) — informational only |
| Total Credits | Credit note credit (`credit_note_credit`) — informational only |
| Total Outstanding | Net outstanding (invoices - deposits - credit notes) |

### Grand Total Row

Sum of all columns across all customers.

---

## Calculation Logic

### Per-Customer Totals

```
Ageing Buckets: Sum of invoice outstanding amounts per bucket
Total Outstanding = Sum(Invoice Outstanding) - Sum(Deposits) - Sum(Credit Notes)
  // Allows negative values (customer credit balance)
```

### Data Fields from Controller

The controller passes each customer with:
- `customer_number`, `customer_name`
- `total_aging` - Average days overdue
- `ageing.current` - Current bucket total
- `ageing['1to30days']` - 1-30 days bucket
- `ageing['31to60days']` - 31-60 days bucket
- `ageing['61to90days']` - 61-90 days bucket
- `ageing['91to120days']` - 91-120 days bucket
- `ageing['over120days']` - 121+ days bucket
- `total_outstanding` - Net outstanding after deposits & credit notes

---

## Formatting Rules

### Negative Values (Accounting Format)

Negative values in all columns use accounting bracket format with red color:
- Example: `-60,000` displays as `(60,000)` in red
- Applied to: All ageing bucket cells, Total Outstanding cell, and Grand Total row
- PDF uses CSS class `.negative` (`color: red`); Excel uses the `FFFF0000` font color on the same cells.

This follows standard accounting conventions where brackets indicate credit balances or negative amounts.

---

## PDF / Excel Parity

Both outputs render the same 13 columns in the same order with the same labels and same formatting rules:

| # | Column | Source / formula |
|---|--------|------------------|
| 1 | Customer Name | `customer_name` |
| 2 | Cust No | `customer_number` |
| 3 | Average Days | `total_aging` (avg days overdue across that customer's overdue invoices) |
| 4 | Current | `ageing.current` |
| 5 | 1-30 Days | `ageing['1to30days']` |
| 6 | 31-60 Days | `ageing['31to60days']` |
| 7 | 61-90 Days | `ageing['61to90days']` |
| 8 | 91-120 Days | `ageing['91to120days']` |
| 9 | 121+ Days | `ageing['over120days']` |
| 10 | Total OverDue | Sum of columns 5–9 (informational; not part of any other total) |
| 11 | Total Deposits | `deposit_credit` (informational) |
| 12 | Total Credits | `credit_note_credit` (informational) |
| 13 | Total Outstanding | `total_outstanding` (net of deposits + credit notes; can be negative) |

**Reading check**: Current + Total OverDue − Total Deposits − Total Credits = Total Outstanding.

**Total row** (both outputs): the word `Total` spans the first three columns (Customer Name + Cust No + Average Days), then bucket grand totals in columns 4–9, Total OverDue / Total Deposits / Total Credits grand sums in columns 10–12, and the grand `Total Outstanding` sum in column 13. All amounts use the same negative-bracket / red-font rule.

**Excel-specific**: header strip (company name, address, title, report ID, printed by, print date/time) merges `A:M` (13 columns wide).

---

## Relationship to Detail Report

The Summary Report totals must match the Detail Report:

| Summary Column | Detail Report Equivalent |
|----------------|--------------------------|
| Current | Sum of all customers' Current bucket |
| 1-30 Days | Sum of all customers' 1-30 Days bucket |
| Total Outstanding | NET GRAND TOTAL from Detail Report |

### Consistency Requirements

- Both reports use the same `calculateDueDate()` function
- Both reports use the same aging bucket ranges
- Both reports allow negative net outstanding (credit balances)
- Both reports use the same company_code scoping via user association
- Both reports apply per-row PKR conversion for multi-currency invoices (Summary parity added in v1.3, matching Detail v1.7)
- Both reports use `service.ticket_issue_date` as the effective invoice date for Air services and `invoice.invoice_date` otherwise (Summary parity added in v1.3)

---

## Version History

### Version 1.9 (2026-07-07)

- **Manual JE match now company-scoped** — the inline Manual JE query (v1.5) matched JE rows by invoice **number** only. Invoice numbers repeat across companies, so a same-numbered invoice in another company with a Manual JE would wrongly reduce this company's outstanding. The `journal_batch` include now nests a required `createdBy` user include filtered by `req.user.company_code` (part of the system-wide Manual JE company-scoping fix — see `JOURNAL_ENTRIES_IMPROVEMENTS.md`, 2026-07-07).

### Version 1.8 (2026-07-06)

- **Three informational columns added** between "121+ Days" and "Total Outstanding" (report is now 13 columns, was 10) — mirrors the Detail report's v1.11 Total OverDue column and additionally exposes the credit figures the controller already computed:
  - **Total OverDue** = sum of the five overdue buckets (1-30 + 31-60 + 61-90 + 91-120 + 121+); Current excluded. Blank math impact — not added into any other total.
  - **Total Deposits** = `deposit_credit` (was already subtracted inside `total_outstanding`, now also shown).
  - **Total Credits** = `credit_note_credit` (same).
  - Grand Total row sums all three across customers; `Total Outstanding` math unchanged.
  - PDF (`ar_ageing_analysis_summary.ejs`): three new `th`/`td` pairs + three grand-total cells (standard negative-bracket rule).
  - Excel (`getARAgeingAnalysisSummaryReport`): `columns`/`columnLabels`/`amountColumns` extended; data rows write cells 4..13; `grandTotals` extended; header-strip merges `A:J → A:M`.
  - No controller calculation changes — `deposit_credit`/`credit_note_credit` were already on the customer data object.

### Version 1.7 (2026-07-06)

- **Customer column split into Customer Name + Cust No** — the single `Customer` column (`{customer_number}-{customer_name}`) was split into two columns: **Customer Name** (`customer_name`) first, then **Cust No** (`customer_number`). Report is now 10 columns (was 9). Applied to both outputs:
  - PDF (`ar_ageing_analysis_summary.ejs`): two `th`/`td` cells replace the combined one; Total-row label colspan `2 → 3`.
  - Excel (`getARAgeingAnalysisSummaryReport`): `columns`/`columnLabels` updated; data cells shifted right by one (Average Days now col 3, buckets 4–9, Total Outstanding col 10); Total label merge `1..2 → 1..3`; header-strip merges `A:I → A:J`.
  - Display-only — no calculation, filter, or ordering change.

### Version 1.6 (2026-06-11)

- **Branch "Is Not Blank" no longer restricts** — The default filter value used to add `branch_id != null` to the customer query, silently excluding customers with no branch (parity fix with Detail report v1.10, discovered via invoice HOIN00000005 / customer CUS004 / company 1004). Now "Is Not Blank" applies no condition in `getARAgeingAnalysisSummaryReport`; "Is Blank" / "Is Equal" / "Between" are unchanged. The Summary report has no Sales ID filter, so Branch was the only change.

### Version 1.5 (2026-05-15)

Brings the Summary controller into parity with the Detail report v1.9 for Manual JE settlements.

- **Manual JE settlement included in outstanding** — Previously the Summary only subtracted `receipt_settlement_invoice` amounts. If an invoice was settled (fully or partially) via a Manual JE batch referencing the invoice number in `analysis_code1`, that adjustment was ignored — the aging buckets and `Total Outstanding` showed the pre-JE figure.

    **Fix** (`psback/controllers/report.controller.js` inside `getARAgeingAnalysisSummaryReport`):
    1. After `allInvoicesSummary` is fetched, an inline query reads `journal_entries` (joined to `journal_batches`) filtered to live Manual JE rows for those invoice numbers: `batch_type = 'Manual JE'`, `journal_batch.status != 'Void'`, and `description NOT LIKE 'VOID REVERSAL -%'`.
    2. Results are grouped into `Map<invoice_number, manualJePKR>` (debit + credit per row, summed across rows).
    3. In the per-invoice group loop, `invoiceOutstanding = group.totalAmountPKR - group.totalSettledPKR - manualJePKR`.

    **Void behavior (automatic)**:
    - When a Manual JE batch is voided, its `status` flips to `Void` → all rows excluded.
    - The reversal batch posted by `voidBatch` carries `description` prefix `VOID REVERSAL -` on every row → also excluded.
    - Net effect: a voided JE contributes zero, so the invoice's outstanding pops back to its pre-JE value on the next report run.

    **Performance**: one batched query for all invoice numbers in the current report; no N+1 per customer. Runs inside the same Sequelize transaction as the rest of the report fetches.

    **Scope notes**:
    - Logic was written inline in the controller (not in `manualJeAdjustment.js`) to keep the change contained to the report.
    - Mirrors the where-clause shape used by `manualJeAdjustment.js`'s `liveManualJeWhereClause`. Detail report (v1.9) carries the same inline implementation; future changes there should be cross-applied here.

### Version 1.0 (February 2026)
- Initial implementation with customer aging summary

### Version 1.1 (March 2026)
- **Accounting bracket format for negatives** - Negative values now display in `(brackets)` with red color instead of `-minus` format, following standard accounting conventions
- Applied to all numeric cells (ageing buckets, total outstanding, grand totals)

### Version 1.2 (April 2026)
- **Currency conversion fix** - Invoice amounts now converted to PKR using invoice's own `exchange_rate` field first (matching customer account statement report logic), with fallback to `currencies` table. Previously only looked up from `currencies` table which returned no results.
- **N+1 query elimination** - Replaced per-customer queries (3 queries × N customers) with batch queries using `Op.in`. Reduces DB round-trips from ~1500 to 5 for 500 customers.

### Version 1.4 (2026-05-11)

Brought the Excel export into structural parity with the PDF.

- **Excel freeze pane fix** — Previously `ySplit: 7` froze only the title block (rows 1-7) so the column header row at row 8 would scroll out of view when paging through data. Now `ySplit: 8` with `topLeftCell: 'A9'` and `activeCell: 'A9'` so the column header row stays pinned along with the title block. The `topLeftCell` property is required — without it, Excel re-renders the frozen rows at the top of the scrollable pane too, producing a duplicate-header visual effect.

- **Excel column set rewritten** — was 12 columns (Customer, Sales ID, Credit Limit, Total Invoices, Total Deposits, Total Outstanding, Current, 1-30, 31-60, 61-90, 91-120, 120+ Days). Now 9 columns matching the PDF: Customer, Average Days, Current, 1-30, 31-60, 61-90, 91-120, 121+ Days, Total Outstanding.
- **Dropped from Excel** — Sales ID, Credit Limit, Total Invoices, Total Deposits (these were already commented out / hidden in the PDF).
- **Added to Excel** — Average Days column (sourced from `total_aging`).
- **Label fixed** — Excel now says "121+ Days" instead of "120+ Days", matching the PDF.
- **Total row** — `Total` label now spans columns 1–2 (mirrors PDF's `colspan="2"`), bucket totals in 3–8, grand Total Outstanding sum in column 9. Previously Excel put "Total" into the Total Outstanding column and omitted the grand sum.
- **Negative formatting** — Excel now uses accounting brackets `(amount)` with red font for negatives in both data rows and the total row, matching the PDF's `.negative` styling.
- **Header strip merge** — adjusted from `A:J` to `A:I` (was wider than the data table; now matches).

### Version 1.3 (2026-05-10)

Brings the Summary controller into full parity with the Detail controller (v1.7) so that the Summary's per-customer `Total Outstanding` sum equals the Detail's `NET GRAND TOTAL`, and the bucket distribution matches column-by-column.

- **Multi-currency invoice aggregation (parity with Detail v1.7)** — Replaced the old `totalAmount` accumulator (raw sum then multiply by first row's `exchange_rate`) with a per-row PKR conversion. For a `KHIN00000093`-style invoice with three SAR rows and one PKR row, the PKR row is no longer multiplied by the SAR rate. Each row is now converted to PKR using *its own* `currency` + `exchange_rate` at accumulation time, then summed.
  - Helper added: `resolveRowPkrRate(row)` returns `{ rate }` per row using stored `exchange_rate` first, falling back to `cachedConvertToPKRAR`, then to rate `1`.
  - Group accumulator now tracks `totalAmountPKR` and `totalSettledPKR` (both PKR-converted at accumulation).
  - Settlements deduped by `settlement.id` and converted using the parent invoice row's rate (settlements have no currency/rate of their own).
  - Voided settlements (`receipt_settlement.status === 'Void'`) still excluded.
- **Air → ticket_issue_date for due date (parity with Detail)** — Air services now use `service.ticket_issue_date` as the effective invoice date for due-date calculation; non-Air services keep using `invoice.invoice_date`. This ensures Air invoices land in the same ageing bucket on both reports.
  - Helper added: `resolveEffectiveInvoiceDate(row)`.
  - Required adding `Service` (with `service_type`) include to the Summary's `invoice.findAll` query.
- **Verified equivalences with Detail report**:
  - Deposit credit: same `current_amount` + `cachedConvertToPKRAR` path.
  - Credit note credit: same `amount - used_amount` formula.
  - Negative net outstanding allowed (credit balance) — both reports.
  - Customer inclusion criterion: same (has invoices, deposits, or credit notes).

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/ARAgeingAnalysisSummaryReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` | Controller logic (~line 16445) |
| `psback/views/pages/reports/ar_ageing_analysis_summary.ejs` | PDF template |
| `psfront/src/api/report.js` | API client (`getARAgeingAnalysisSummaryReport`) |

### Report Metadata

```javascript
{
  report_number: "ARSUM" + timestamp,
  report_type: "ar-ageing-analysis-summary-report",
  file_type: "xlsx" | "pdf"
}
```

---

**Document End**
