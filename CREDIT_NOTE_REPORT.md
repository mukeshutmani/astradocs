# Credit Note Report

## Overview

The Credit Note Report lists customer credit notes (refund documents) grouped by customer, showing CN details, billing amount (PKR + original foreign currency where applicable), refund linkage, invoice reference, and segment info. It supports PDF and Excel export.

---

## Endpoint

- **Method**: `POST`
- **URL**: `/api/report/getCreditNoteReport`
- **Permission**: `Credit-Note-Report`
- **Controller**: `exports.getCreditNoteReport` in `psback/controllers/report.controller.js`
- **Template (PDF)**: `psback/views/pages/reports/credit-note.ejs`

---

## Request Body

| Field             | Type    | Description |
|-------------------|---------|-------------|
| `dateFilter`      | string  | `blank`, `=`, `<`, `<=`, `>`, `>=`, `<>`, `between` |
| `startDate`       | string  | Required when `dateFilter` is set (other than `blank`) |
| `endDate`         | string  | Required when `dateFilter = between` |
| `branchFilter`    | string  | `isNotBlank`, `isBlank`, `isEqual` |
| `branch_id`       | int     | Required when `branchFilter = isEqual` |
| `customerFilter`  | string  | `isNotBlank`, `isBlank`, `isEqual`, `between` |
| `customer_id`     | int     | Single customer |
| `customer_idStart`/`customer_idEnd` | int | Customer range |
| `statusFilter`    | string  | `isNotBlank`, `isBlank`, `isEqual` |
| `status`          | string  | One of `Printed`, `Settled`, `Partially Settled` |
| `type`            | string  | `pdf` (default) or `excel` |

---

## Default Behavior

- **Status**: Only credit notes with `doc_status IN ('Printed', 'Settled', 'Partially Settled')` are included. `Raised` and `Void` are excluded.
- **Sort**: Within each customer, credit notes are sorted by `doc_date` **descending** (latest first).
- **Display filters**: The `Date Range` filter is pushed to the top of the report header, followed by Customer, Branch, and Status filters.

---

## Columns

| Column            | Source |
|-------------------|--------|
| CN Date           | `credit_notes.doc_date` |
| CN No.            | `credit_notes.reference` |
| Issue By          | `users.username` resolved from `credit_notes.buy_staff_id` (fallback: `System`) |
| Status            | `credit_notes.doc_status` |
| Billing Currency  | ISO code from `currency_codes` resolved via `credit_notes.currency_id` |
| Billing Amount    | PKR-first display: `{pkr_amount} PKR ({original} {code})` for foreign currency; `{amount} PKR` for PKR |
| Refund No.        | `refunds.refund_no` |
| Pax Name          | `refunds.passenger_name` |
| Invoice No.       | `refunds.invoice_no` |
| Original Segment  | `refunds.refund` |
| Refund Segment    | `refunds.refund` |

### Billing Amount Logic

1. If `currency_id !== base_currency_id` (foreign currency):
   - `pkr_amount` = `billing_amount_base` (or `billing_amount × exchange_rate` if base missing).
   - `original_amount` = `billing_amount` (document currency).
   - Display: `{pkr_amount} PKR ({original_amount} {CODE})` e.g. `77,105.67 PKR (7,105.00 AED)`.
2. If `currency_id === base_currency_id` (PKR):
   - Display: `{billing_amount} PKR`.
3. **Totals / subtotals**: always sum `pkr_amount` only — totals are PKR-only.

---

## Data Flow

1. **Fetch**: `db.customer.findAll` with nested include `Orders → refunds → credit_note` (alias `'credit_note'`). Credit notes filtered by `creditNoteWhere` (status + date + doc_status).
2. **Enrichment**:
   - Pre-fetch currency codes (`currency_codes`) → map `id → currency_code`.
   - Pre-fetch users referenced by `buy_staff_id` → map `id → username`.
   - Attach computed fields to each credit note: `is_foreign_currency`, `original_amount`, `pkr_amount`, `currency_code`, `buy_staff_name`.
3. **Flatten + sort**: All refunds from a customer's orders are flattened into one list and sorted by `credit_note.doc_date DESC`, then wrapped in a single synthetic order so the template iteration continues to work.
4. **Render**:
   - PDF: EJS template (`credit-note.ejs`) → `createPdf` → MinIO upload → URL returned.
   - Excel: `createExcel` on flattened rows → MinIO upload → signed download URL.

---

## Template Layout (PDF)

- **A4 landscape**, 3 mm margins.
- Cells wrap long content (`word-break: break-word`); table uses `table-layout: fixed; width: 100%` in print so all 11 columns fit within page width.
- Customer header row spans all columns.
- Subtotal row per customer (sum of PKR amounts).
- Final Total row (grand total in PKR).

---

## Frontend

- **Component**: `psfront/src/pages/Report/CreditNoteReport.jsx`
- **API module**: `getCreditNoteReport` in `psfront/src/api/report.js`
- **Filters in UI**: Date, Branch, Customer, Status (dropdown options: `Printed`, `Settled`, `Partially Settled`).
- **Generate buttons**: PDF and Excel (via dropdown).
- Report history section was removed per UX request — only the filter form remains.

---

## Known Edge Cases

1. **Duplicate CN references across companies**: Two different companies/branches can both use prefix `TT` and reach `doc_no = 5`, producing two credit notes with the same `reference` string (e.g. `TTCN00000005`). The current query does not scope by `company_code`, so results may leak across companies. Fix pending: filter customers/CNs by `req.user.company_code`.
2. **Missing `buy_staff_id`**: falls back to `System`.
3. **Missing `billing_amount_base` on foreign currency CN**: falls back to `billing_amount × exchange_rate`.

---

## File Locations

```
Backend:
├── controllers/
│   └── report.controller.js       # getCreditNoteReport
├── routes/
│   └── report.route.js            # POST /api/report/getCreditNoteReport
└── views/pages/reports/
    └── credit-note.ejs            # PDF template

Frontend:
├── pages/Report/
│   └── CreditNoteReport.jsx       # Filter form + generate buttons
└── api/
    └── report.js                  # getCreditNoteReport()
```
