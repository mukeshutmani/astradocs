# AR Ageing Analysis Summary Report - Technical Documentation

**Version**: 1.2
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Verified

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

**Output**: PDF or Excel via dropdown button.

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
| Customer | `{customer_number}-{customer_name}` |
| Average Days | Average days overdue across invoices |
| Current | Invoices within credit period |
| 1-30 Days | 1-30 days overdue |
| 31-60 Days | 31-60 days overdue |
| 61-90 Days | 61-90 days overdue |
| 91-120 Days | 91-120 days overdue |
| 121+ Days | Over 120 days overdue |
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
- CSS class `.negative` applies `color: red`

This follows standard accounting conventions where brackets indicate credit balances or negative amounts.

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

---

## Version History

### Version 1.0 (February 2026)
- Initial implementation with customer aging summary

### Version 1.1 (March 2026)
- **Accounting bracket format for negatives** - Negative values now display in `(brackets)` with red color instead of `-minus` format, following standard accounting conventions
- Applied to all numeric cells (ageing buckets, total outstanding, grand totals)

### Version 1.2 (April 2026)
- **Currency conversion fix** - Invoice amounts now converted to PKR using invoice's own `exchange_rate` field first (matching customer account statement report logic), with fallback to `currencies` table. Previously only looked up from `currencies` table which returned no results.
- **N+1 query elimination** - Replaced per-customer queries (3 queries × N customers) with batch queries using `Op.in`. Reduces DB round-trips from ~1500 to 5 for 500 customers.

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
