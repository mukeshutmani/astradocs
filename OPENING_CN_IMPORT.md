# Opening Credit Note Import (Excel)

## Purpose

Imports historical / opening **credit notes** (for **customers**) from another
company's old system into Astra via an Excel file. Mirrors the **Opening Invoice
Import** and **Opening XO Import** features — same flow, same rules.

---

## 1. Excel Template

### Sheet name: `CN`

| Column | Type | Mandatory | Notes |
|---|---|---|---|
| `CRNO` | Text | Yes | Credit note number (digits; optional letter prefix is stripped) |
| `BRANCH` | Text | Yes | Branch code (e.g., `KH`, `TT`, `TP`) |
| `CUSTNO` | Text | Yes | Customer number — must already exist in the system |
| `CUSTNAME` | Text | Yes | Customer name (case-insensitive match) |
| `CNDATE` | Date | Yes | Credit note's **doc_date** — format `DD-MM-YYYY` |
| `BILLCURCODE` | Text | Yes | Currency code (`PKR`, `USD`, …) |
| `EXRATE` | Number | Yes | Exchange rate (`1` for PKR) |
| `BILLCNAMT` | Number | Yes | Credit note amount (gross, in `BILLCURCODE`) |
| `PAX1` | Text | No | Optional passenger name — saved on the credit note |
| `REMARK` | Text | No | Optional notes — saved as-is |

Accepted formats: `.xlsx` (priority), `.xls`. CSV not supported.

---

## 2. Rules

1. **CN number** = `[BRANCH] + OC + [8-digit number]`. `OC` is the fixed Opening Credit Note type code.
   - Letter prefix on `CRNO` is stripped; letters between digits → error.
   - More than 8 digits → keep last 8; less than 8 → pad left with zeros.
   - e.g. `73256` → `00073256` → `KHOC00073256`.
2. **Customer matching** — `CUSTNO` must exist in the `customers` table for the company;
   `CUSTNAME` compared case-insensitively (trimmed). Mismatch → validation error.
3. **Status** — every imported CN saved as `doc_status = 'Printed'`.
4. **No GL/JE** — CN import does not post journal entries.
5. **Currency** — `PKR` → `EXRATE` must be 1. Foreign currency → `EXRATE > 0`; PKR amount =
   `BILLCNAMT × EXRATE` and is stored in `*_base` columns.
6. **`is_opening` flag** on `credit_notes` identifies imported CNs.
7. **`pax_name`** new column on `credit_notes` — stores the optional `PAX1` from the file.
8. **Date format** `DD-MM-YYYY`; invalid format/date → validation error. `CNDATE` is the
   credit note's own document date.
9. **Duplicate check** — branch + 8-digit number, across existing `credit_notes.reference`
   (which holds e.g. `KHCN00000020` or `KHOC00073256` for opening imports).
10. **Import batch** — every upload creates an `import_batches` row (`batch_type = 'CN'`),
    shown in the Import History tab with delete (blocked if any CN has a settlement/payment).
11. **Preview-then-import** — upload → green/red preview → all-or-nothing confirm.
12. **Single currency per file**; cross-branch / cross-customer allowed in one file.
13. Gated by the `IMPORT_ENABLED` env flag — **write actions only**. When `false`:
    upload (`/preview`, `/confirm`) and batch **delete** return `403`, and the frontend
    blocks the Choose-file button (popup *"Import Disabled"*) and disables Preview/Confirm
    + the trash icon. **Read-only Import History and the Template download stay available**
    so users can still review past imports. The frontend reads the flag via `GET /import-status`
    (auth-only, ungated) through the `useImportEnabled()` hook.

---

## 3. Technical Implementation

### Database (already migrated)
- New columns on `credit_notes`: `is_opening` (TINYINT default 0), `import_batch_id` (INT,
  FK to `import_batches`), `pax_name` (VARCHAR(255)).
- `import_batches.batch_type` enum extended to include `'CN'`.

### Backend (new files)
- `psback/services/cnImport.service.js` — Excel parsing + validation.
- `psback/controllers/cnImport.controller.js` — preview / confirm / history / delete / template.
- `psback/routes/cnImport.route.js` — mounted at `/cn-import` in `index.js`.
- Reuses `middlewares/importEnabledGate.js`.
- Additive model change: `is_opening`, `import_batch_id`, `pax_name` on `models/credit_note.js`.
- New association in `models/index.js`: `import_batch ↔ credit_note`.

### API endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/cn-import/preview` | Parse + validate (no DB write) |
| POST | `/cn-import/confirm` | Re-validate + write all rows (all-or-nothing) |
| GET | `/cn-import/batches` | Import history |
| GET | `/cn-import/batches/:id` | Credit notes created by one batch |
| DELETE | `/cn-import/batches/:id` | Delete a batch (blocked if any CN settled/paid) |
| GET | `/cn-import/template` | Download the blank Excel template |

### Frontend (new files)
- `psfront/src/api/cnImport.js`
- `psfront/src/pages/CnImport/CnImportPage.jsx` (New Import / Import History / Template tabs)
- `psfront/src/pages/CnImport/CnImportUpload.jsx`, `CnImportHistory.jsx`
- Route `iur/cn-import` in `App.jsx`; sidebar entry **"Credit Note Import"** under the Import section.
- Reuses `pages/InvoiceImport/ImportAlertDialog.jsx` for centered alerts.

### Phase 2 — Documents → Credit Note tab (built)
- `psback/services/openingCn.service.js` → `getOpeningCreditNotesAsDocuments(req)` shapes opening
  CNs (`is_opening = 1`) like the rows the Credit Note tab expects; company-scoped via the import batch.
- `credit_note.controller.js` `getCreditNotes` appends these rows (existing normal-CN query untouched —
  separate block: `[...openingCreditNotes, ...filteredCreditNotes]`).
- In the list, an opening CN number is **clickable** → opens a centered detail popup
  (`psfront/src/components/OpeningCreditNoteDialog.jsx`); small **"opening cn"** text under the number;
  Order No. and View Refund blank for opening CNs; the batch-void checkbox is shown faded/disabled
  (opening CNs are not batch-void-selectable).

### Credit Note Report (built)
- `report.controller.js` `getCreditNoteReport` fetches opening CNs separately (company-scoped via the
  CN import batches) and merges them per customer. Branch filter matches the CN-number prefix against
  the branch's **`document_prefix`** (`{document_prefix}OC…`, e.g. `TT` → `TTOC…`). See `docs/CREDIT_NOTE_REPORT.md`.

### Credit Note Payment / settlement (built — enables Phase 3)
- The **Credit Note Payment** screen (`/payment-requisition`) lists a customer's credit notes via
  `customer.controller.js` `getCustomersWithCreditNote` (`GET /customer/getCreditNotes`). That query walks
  Customer → Orders → refunds → credit_note, so opening CNs (no refund/order) were missing.
- Fix: after loading the customers, the controller fetches their opening CNs (company-scoped via the CN
  import batches) and attaches each as a **synthetic order/refund** on the customer, so the existing
  frontend list (`PaymentRequisition.jsx` `processCreditNoteData`) renders them with no frontend change.
  "Order No." is blank for opening CNs.
- **No settlement-write change needed.** `payment.controller.js` `settleCreditNote` keys only on
  `credit_note_id` + `used_amount` and the chosen branch's `document_prefix` for the payment number — it
  does not require a refund/order/branch on the CN. Settling an opening CN increments its `used_amount`;
  its available balance (`amount − used_amount`) shrinks on the next load and it drops off the list once
  fully settled.

### Storage of one imported credit note
One `credit_notes` row with:
- `customer_id` resolved from `CUSTNO`
- `doc_no` = the 8-digit number (as int)
- `doc_date` = `CNDATE`
- `doc_type` = `'Credit Note'`, `doc_status` = `'Printed'`
- `reference` = `[BRANCH]OC[8digits]` (the full CN number)
- `to` = `CUSTNAME`
- `currency_id` from `BILLCURCODE`, `base_currency_id` = 110 (PKR)
- `exchange_rate` = `EXRATE`
- `amount`, `billing_amount`, `base_amount`, `refund_amount` = `BILLCNAMT`
- `*_base` variants = `BILLCNAMT × EXRATE` (PKR)
- `pax_name` = `PAX1`, `remarks` = `REMARK`
- `is_opening` = 1, `import_batch_id` set, `je_generated` = 0

---

## 4. Phase Plan

| Phase | Scope |
|---|---|
| Phase 1 (done) | CN import — upload, preview, confirm, history, delete, template. |
| Phase 2 (done) | Opening CNs shown in **Documents → Credit Note tab** (clickable → detail popup, "opening cn" label, faded/disabled batch-void checkbox). Also shown in the **Credit Note Report** (branch filter by `document_prefix` prefix). |
| Phase 3 (done) | **Credit Note Payment** (`/payment-requisition`) now lists and settles opening CNs. Backend injects opening CNs into `getCustomersWithCreditNote` as synthetic order/refunds; the existing `settleCreditNote` write handles them unchanged (keys on `credit_note_id` + `used_amount`; payment-doc branch chosen on screen). |
