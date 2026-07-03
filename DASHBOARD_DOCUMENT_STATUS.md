# Dashboard — Document Status Overview

Pie chart on the executive dashboard showing the status distribution of the company's documents, with customer and document-type filters.

## Where

- **Frontend**: `psfront/src/components/dashboard/DocumentStatusChart.jsx`
- **API**: `GET /api/dashboard/document-status?customer_id=<no|all>&document_type=<type|all>`
- **Backend**: `getDocumentStatus` in `psback/controllers/dashboard.controller.js`

## Counting rules

| Document type | Source of count |
|---|---|
| Invoice | `documents` table (`document_type = 'invoice'`), **distinct `document_number`**, status from joined `invoices` row |
| XO (Costing) | `documents` table (`document_type = 'costing'`), **distinct `document_number`**, status from joined `costs` row |
| Deposit | `customer_deposits` rows (one row = one document) |
| Receipt | `receipt_settlements` rows |
| Payment | `payment_settlements` rows |
| Credit Note | `credit_notes` row count (no status column — shown as "Active") |

Company scoping: invoice/XO go through `service → order → order.user_id ∈ company users`; deposit/receipt/payment scope by their own `user_id`; credit notes via customer's `user_id`.

### Why distinct document numbers (changed 2026-07-02)

1. `invoices` / `costs` hold **one row per service line**, so a multi-service invoice has several rows. Counting rows inflated every slice (e.g. e-Safar: 406 rows vs 326 real printed invoices).
2. When a service is created, a price row is pre-created with default status `Raised` and **no document number**. These are not raised documents yet, but the old row-count included them (e-Safar: dashboard said 31 Raised invoices, Documents listing showed 1).
3. Counting distinct `documents.document_number` matches exactly what the Documents listing (`getDocuments` in `psback/controllers/document.controller.js`) shows.

## Known limitations

1. Opening (imported) XOs have no `order` row, so they are excluded by the company-scoping join. The Documents listing appends them separately; the dashboard has never included them.
2. If one document number has rows in more than one status (legacy duplicate-number data), it is counted once per status.
3. When "All Documents" is selected, counts are aggregated per status across document types.
