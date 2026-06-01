# Opening XO Import (Excel)

## Purpose

Imports historical / opening **XO documents** (supplier cost documents) from another
company's old system into Astra via an Excel file. Mirrors the **Opening Invoice Import**
feature (`docs/OPENING_INVOICE_IMPORT.md`) — same flow, same rules — but for the supplier
(cost) side instead of the customer (invoice) side.

---

## 1. Excel Template

### Sheet name: `XO`

| Column | Type | Mandatory | Notes |
|---|---|---|---|
| `XONO` | Text | Yes | XO number (digits; optional letter prefix is stripped) |
| `BRANCH` | Text | Yes | Branch code (e.g., `KH`, `TT`, `TP`) |
| `SUPPNO` | Text | Yes | Supplier number — must already exist in the system |
| `SUPPNAME` | Text | Yes | Supplier name (case-insensitive match) |
| `XO DATE` | Date | Yes | XO date — format `DD-MM-YYYY` |
| `BILLCURCODE` | Text | Yes | Currency code (`PKR`, `USD`, …) |
| `EXRATE` | Number | Yes | Exchange rate (`1` for PKR) |
| `XO AMOUNT` | Number | Yes | Total XO amount (gross, in `BILLCURCODE`) |
| `REMARKS` | Text | No | Optional notes — saved as-is |

Accepted formats: `.xlsx` (priority), `.xls`. CSV not supported.

---

## 2. Rules (same as Invoice Import)

1. **XO number** = `[BRANCH] + OX + [8-digit number]`. `OX` is the fixed Opening XO type code.
   - Letter prefix on `XONO` is stripped; letters between digits → error.
   - More than 8 digits → keep last 8; less than 8 → pad left with zeros.
   - e.g. `TTOX00501515` → `00501515` → `TTOX00501515`.
2. **Supplier matching** — `SUPPNO` must exist in the `suppliers` table for the company;
   `SUPPNAME` compared case-insensitively (trimmed). Mismatch → validation error.
3. **Status** — every imported XO saved as `Printed`.
4. **No GL/JE** — XO import does not post journal entries (GL handled separately).
5. **No order** — each XO links to a Miscellaneous Domestic **service** (with `supplier_id`,
   `order_id` = NULL). Stored as a row in the `costs` table.
6. **Currency** — `PKR` → `EXRATE` must be 1. Foreign currency → `EXRATE > 0`; PKR amount =
   `XO AMOUNT × EXRATE`.
7. **`is_opening` flag** on `costs` identifies imported XOs.
8. **Date format** `DD-MM-YYYY`; invalid format/date → validation error.
9. **Duplicate check** — branch + 8-digit number, across existing `documents` (costing) and
   `costs.xo_number`. Duplicate inside the file or vs the DB → validation error.
10. **Import batch** — every upload creates an `import_batches` row (`batch_type = 'XO'`),
    shown in the Import History tab with delete (blocked if any XO has a payment/settlement).
11. **Preview-then-import** — upload → green/red preview → all-or-nothing confirm.
12. **Single currency per file**; cross-branch allowed in one file.
13. Access gated by `IMPORT_ENABLED` env flag.

---

## 3. Technical Implementation

### Database
- New columns on `costs`: `is_opening` (TINYINT default 0), `import_batch_id` (INT, FK to
  `import_batches`), `xo_number` (VARCHAR(50) — `costs` has no native number field).
- `import_batches` table is shared with Invoice Import (`batch_type` = `'INV'` | `'XO'`).

### Backend (new files)
- `psback/services/xoImport.service.js` — Excel parsing + validation.
- `psback/controllers/xoImport.controller.js` — preview / confirm / history / delete.
- `psback/routes/xoImport.route.js` — mounted at `/xo-import` in `index.js`.
- Reuses `middlewares/importEnabledGate.js`.
- Additive model change: `is_opening`, `import_batch_id`, `xo_number` on `models/cost.js`.

### API endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/xo-import/preview` | Parse + validate (no DB write) |
| POST | `/xo-import/confirm` | Re-validate + write all rows (all-or-nothing) |
| GET | `/xo-import/batches` | Import history |
| GET | `/xo-import/batches/:id` | XO costs created by one batch |
| DELETE | `/xo-import/batches/:id` | Delete a batch (blocked if any XO settled) |
| GET | `/xo-import/template` | Download the blank Excel template (`XO` sheet + headers + sample rows + Instructions sheet) |

### Template tab
- The XO Import page has a **Template** tab with a **Download Template** button. The `.xlsx`
  is backend-generated (`services/importTemplate.service.js`) from the same column list, so
  it always matches the validation rules. Give this file to a client as the required format.

### Frontend (new files)
- `psfront/src/api/xoImport.js`
- `psfront/src/pages/XoImport/XoImportPage.jsx` (New Import / Import History tabs)
- `psfront/src/pages/XoImport/XoImportUpload.jsx`, `XoImportHistory.jsx`
- Route `iur/xo-import` in `App.jsx`; sidebar entry **"XO Import"** under the Import section.
- Reuses `components/InvoiceImport/ImportAlertDialog.jsx` for centered alerts.

### Storage of an imported XO
1. One `services` row — Miscellaneous Domestic, `supplier_id` set, `order_id` = NULL.
2. One `costs` row — `status` = Printed, `is_opening` = 1, `import_batch_id` set,
   `xo_number` = generated number, `currency` = code, `exchange_rate` = EXRATE,
   `total_costing`/`amount`/`published_rate`/`net_rate` = XO AMOUNT.

---

## 4. Phase Plan

| Phase | Scope |
|---|---|
| Phase 1 (done) | XO import — upload, preview, confirm, history, delete. |
| Phase 2-A (done) | Opening XOs shown in **Documents → XO tab**. |
| Phase 2-B (done) | Opening XOs shown in the **Supplier Payment** list (clickable → detail popup). |
| Phase 2-C (done) | Settling an opening XO via Supplier Payment. `settlePayment` (`payment.controller.js`) derives the branch from the XO number prefix (e.g. `TTOX…` → `TT`) when the cost has no order/branch — mirror of the `settleReceipt` fix for opening invoices. |

### Phase 2-B — Supplier Payment list (built)
1. `openingXo.service.js` → `getOpeningCostsForSupplier(companyCode, supplierId)` returns the
   supplier's opening XO costs shaped like `getPaymentCostsBySupplierId`'s `processedServices`.
2. `service.controller.js` → `getPaymentCostsBySupplierId` appends these (existing query, with
   its `document required:true` INNER JOIN, untouched — separate block).
3. `Payments.jsx` — opening XO rows show with the XO number **clickable** → `OpeningXoDialog`,
   small **"opening xo"** text under the number, Order No. blank.

### Phase 2-A — Documents → XO tab (built)
1. `psback/services/openingXo.service.js` → `getOpeningCostsAsDocuments(req)` shapes opening
   costs (`is_opening=1`) like costing-Document rows; company-scoped via the import batch.
2. `document.controller.js` `getDocuments` costing branch appends these rows (existing
   normal-XO query untouched — separate block).
3. In the list, an opening XO number is **clickable** → opens a centered detail popup
   (`psfront/src/components/OpeningXoDialog.jsx`); small **"opening xo"** text under the
   number; Order No. column blank for opening XOs.
