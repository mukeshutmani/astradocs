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
| `dateFilter`      | string  | `blank`, `=`, `<`, `<=`, `>`, `>=`, `<>`, `between`. **Default on UI: `between`.** |
| `startDate`       | string  | Required when `dateFilter` is set (other than `blank`) |
| `endDate`         | string  | Required when `dateFilter = between` |
| `branchFilter`    | string  | `isNotBlank`, `isBlank`, `isEqual`, `isIn` |
| `branch_id`       | int     | Required when `branchFilter = isEqual` |
| `branch_ids`      | int[]   | Required when `branchFilter = isIn` — array of branch IDs (`Op.in`) |
| `customerFilter`  | string  | `isNotBlank`, `isBlank`, `isEqual`, `isIn`, `between` |
| `customer_id`     | int     | Single customer |
| `customer_ids`    | int[]   | Required when `customerFilter = isIn` — array of customer IDs (`Op.in`) |
| `customer_idStart`/`customer_idEnd` | int | Customer range |
| `statusFilter`    | string  | `isNotBlank`, `isBlank`, `isEqual`, `isIn` |
| `status`          | string  | One of `Printed`, `Settled`, `Partially Settled` |
| `statuses`        | string[]| Required when `statusFilter = isIn` — any subset of `Printed`, `Settled`, `Partially Settled`. Values outside the allowed set are silently dropped. |
| `type`            | string  | `pdf` (default) or `excel` |

---

## Default Behavior

- **Status**: Only credit notes with `doc_status IN ('Printed', 'Settled', 'Partially Settled')` are included. `Raised` and `Void` are excluded.
- **Sort**: Within each customer, credit notes are sorted by `doc_date` **descending** (latest first).
- **Display filters**: The `Date Range` filter is pushed to the top of the report header, followed by Customer, Branch, and Status filters.
- **UI default**: Date filter opens in **Between** mode so `startDate` and `endDate` inputs are visible on first load. Branch / Customer / Status default to **Is Equal**.
- **Missing filter values = no restriction**: If the user opens the form and clicks Generate without picking any date / branch / customer / status, the report runs unrestricted (subject only to company scope and the default `doc_status IN ('Printed','Settled','Partially Settled')` rule). Previously, Between with missing dates returned `400` — this was removed so "empty form = full report" works as expected.
- **Multi-value (`isIn`) filters**: Branch, Customer, and Status each support an `isIn` mode that maps to a Sequelize `Op.in` clause.
  - Branch `isIn` renders a native multi-select of all branches (Ctrl/Cmd+click).
  - Customer `isIn` renders the existing live search; each pick is appended as a removable chip.
  - Status `isIn` renders a multi-select of `Printed`, `Settled`, `Partially Settled` — any value outside this set is dropped server-side.
  - When `isIn` produces an empty list, the filter is treated as unset (no restriction applied) rather than returning zero rows.

---

## Columns

Column order (12): `CN Date | CN No. | Issue By | Status | Billing Currency | Exch Rate | Billing Amount | Refund No. | Pax Name | Invoice No. | Original Segment | Refund Segment`.

| Column            | Source |
|-------------------|--------|
| CN Date           | `credit_notes.doc_date` |
| CN No.            | `credit_notes.reference` |
| Issue By          | `users.username` resolved from `credit_notes.buy_staff_id` (fallback: `System`) |
| Status            | `credit_notes.doc_status` |
| Billing Currency  | For foreign CN: `{CODE} ({original_amount})` (e.g. `AUD (408.10)`). For PKR CN: `PKR`. |
| Exch Rate         | `credit_notes.exchange_rate` (e.g. `281.00` when USD→PKR). For PKR CN displays `-`. |
| Billing Amount    | Always PKR only — `{pkr_amount} PKR` (e.g. `77,105.67 PKR`). |
| Refund No.        | `refunds.refund_no` |
| Pax Name          | `refunds.passenger_name` |
| Invoice No.       | `refunds.invoice_no` |
| Original Segment  | `refunds.refund` |
| Refund Segment    | `refunds.refund` |

### Billing Amount Logic

1. If `currency_id !== base_currency_id` (foreign currency):
   - `pkr_amount` = `billing_amount_base` (or `billing_amount × exchange_rate` if base missing).
   - `original_amount` = `billing_amount` (document currency).
   - **Billing Currency cell**: `{CODE} ({original_amount})` (e.g. `AUD (408.10)`).
   - **Exch Rate cell**: `credit_notes.exchange_rate` formatted to 2 decimals.
   - **Billing Amount cell**: `{pkr_amount} PKR` (e.g. `77,105.67 PKR`).
2. If `currency_id === base_currency_id` (PKR):
   - **Billing Currency cell**: `PKR`.
   - **Exch Rate cell**: `-`.
   - **Billing Amount cell**: `{billing_amount} PKR`.
3. **Totals / subtotals**: always sum `pkr_amount` only — totals are PKR-only.

---

## Data Flow

1. **Fetch** (inverted — credit-note-first): `db.credit_note.findAll` with required includes `branch` (company scope via `branches.company_code = req.user.company_code`) and `refund → Order → customer`. All chain joins use integer foreign keys (`credit_notes.refund_id`, `refunds.orderId`, `orders.customer_id`, `credit_notes.branch_id`) — **not** the default `order.hasMany(refund, sourceKey: 'order_number')` varchar join, which would cross-join because order numbers are not unique across customers/companies. After fetch, CNs are grouped by `customer.id` in JS and shaped into the same `customers → Orders → refunds → credit_note` structure the EJS template expects.
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

1. **Duplicate CN references across companies** — _resolved_. The CN query is now company-scoped via a required nested include on `branch` where `company_code = req.user.company_code`, so cross-company CNs are dropped before any display logic runs. If `req.user.company_code` is missing, the endpoint returns `400` instead of silently returning unscoped data.
2. **Cross-customer duplication (order_number collision)** — _resolved_. Previously the query started from `customer → Orders → refunds → credit_note`, and the `order.hasMany(refund)` association joins refunds by `order_number` (varchar). Order numbers are not unique (e.g. `KHSO00000001` exists 7 times across different customers), so a single refund cross-joined to every matching customer — dragging in cross-company customers. The query was inverted to start from `credit_note` and traverse `refund → Order → customer` via integer `belongsTo` FKs only.
3. **Missing `buy_staff_id`**: falls back to `System`.
4. **Missing `billing_amount_base` on foreign currency CN**: falls back to `billing_amount × exchange_rate`.
5. **CN with `NULL branch_id`**: excluded by the company-scope join. Verified safe — all such CNs in current data have no `refund_id`, so they never reached the report anyway.

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
