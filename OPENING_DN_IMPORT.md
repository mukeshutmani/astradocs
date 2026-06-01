# Opening Debit Note Import (Excel)

## Purpose

Imports historical / opening **debit notes** (for **suppliers**) from another
company's old system into Astra via an Excel file. Mirrors the **Opening Credit
Note Import** — same flow, same rules — but for the supplier (debit) side.

---

## 1. Excel Template

### Sheet name: `DN`

| Column | Type | Mandatory | Notes |
|---|---|---|---|
| `DRNO` | Text | Yes | Debit note number (digits; optional letter prefix is stripped) |
| `BRANCH` | Text | Yes | Branch code (e.g., `KH`, `TT`, `TP`) |
| `SUPPNO` | Text | Yes | Supplier number — must already exist in the system |
| `SUPPNAME` | Text | Yes | Supplier name (case-insensitive match) |
| `DNDATE` | Date | Yes | Debit note's **doc_date** — format `DD-MM-YYYY` |
| `BILLCURCODE` | Text | Yes | Currency code (`PKR`, `USD`, …) |
| `EXRATE` | Number | Yes | Exchange rate (`1` for PKR) |
| `BILLDNAMT` | Number | Yes | Debit note amount (gross, in `BILLCURCODE`) |
| `PAX1` | Text | No | Optional passenger name — saved on the debit note |
| `REMARK` | Text | No | Optional notes — saved as-is |

Accepted formats: `.xlsx` (priority), `.xls`. CSV not supported.

---

## 2. Rules

1. **DN number** = `[BRANCH] + OD + [8-digit number]`. `OD` is the fixed Opening Debit Note type code.
   - Letter prefix on `DRNO` is stripped; letters between digits → error.
   - More than 8 digits → keep last 8; less than 8 → pad left with zeros.
   - e.g. `73256` → `00073256` → `KHOD00073256`.
2. **Supplier matching** — `SUPPNO` must exist in the `suppliers` table for the company;
   `SUPPNAME` compared case-insensitively (trimmed). Mismatch → validation error.
3. **Status** — every imported DN saved as `doc_status = 'Printed'`.
4. **No GL/JE** — DN import does not post journal entries.
5. **Currency** — `PKR` → `EXRATE` must be 1. Foreign currency → `EXRATE > 0`; PKR amount =
   `BILLDNAMT × EXRATE` and is stored in `*_base` columns.
6. **`is_opening` flag** on `debit_notes` identifies imported DNs.
7. **`pax_name`** new column on `debit_notes` — stores the optional `PAX1` from the file.
8. **Date format** `DD-MM-YYYY`; invalid format/date → validation error. `DNDATE` is the
   debit note's own document date.
9. **Duplicate check** — branch + 8-digit number, across existing `debit_notes.reference`
   (which holds e.g. `KHDN00000020` or `KHOD00073256` for opening imports).
10. **Import batch** — every upload creates an `import_batches` row (`batch_type = 'DN'`),
    shown in the Import History tab with delete (blocked if any DN has a settlement/payment).
11. **Preview-then-import** — upload → green/red preview → all-or-nothing confirm.
12. **Single currency per file**; cross-branch / cross-supplier allowed in one file.
13. Gated by the `IMPORT_ENABLED` env flag — **write actions only**. When `false`:
    upload (`/preview`, `/confirm`) and batch **delete** return `403`, and the frontend
    blocks the Choose-file button (popup *"Import Disabled"*) and disables Preview/Confirm
    + the trash icon. **Read-only Import History and the Template download stay available**
    so users can still review past imports. The frontend reads the flag via `GET /import-status`
    (auth-only, ungated) through the `useImportEnabled()` hook.

---

## 3. Technical Implementation

### Database (already migrated)
- New columns on `debit_notes`: `is_opening` (TINYINT default 0), `import_batch_id` (INT,
  FK to `import_batches`), `pax_name` (VARCHAR(255)).
- `import_batches.batch_type` enum extended to include `'DN'`.

### Backend (new files)
- `psback/services/dnImport.service.js` — Excel parsing + validation.
- `psback/controllers/dnImport.controller.js` — preview / confirm / history / delete / template.
- `psback/routes/dnImport.route.js` — mounted at `/dn-import` in `index.js`.
- `psback/services/openingDn.service.js` — Phase 2 helper for the Documents → Debit Note tab.
- Reuses `middlewares/importEnabledGate.js`.
- Additive model change: `is_opening`, `import_batch_id`, `pax_name` on `models/debit_note.js`.
- New association in `models/index.js`: `import_batch ↔ debit_note`.
- `controllers/debit_note.controller.js` `listDebitNotes` appends opening DNs (Documents tab only;
  skipped when `status='Printed'`, the payment-fetch path).

### API endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/dn-import/preview` | Parse + validate (no DB write) |
| POST | `/dn-import/confirm` | Re-validate + write all rows (all-or-nothing) |
| GET | `/dn-import/batches` | Import history |
| GET | `/dn-import/batches/:id` | Debit notes created by one batch |
| DELETE | `/dn-import/batches/:id` | Delete a batch (blocked if any DN settled/paid) |
| GET | `/dn-import/template` | Download the blank Excel template |

### Frontend (new files)
- `psfront/src/api/dnImport.js`
- `psfront/src/pages/DnImport/DnImportPage.jsx` (New Import / Import History / Template tabs)
- `psfront/src/pages/DnImport/DnImportUpload.jsx`, `DnImportHistory.jsx`
- `psfront/src/components/OpeningDebitNoteDialog.jsx` (Documents tab click-through popup)
- Route `iur/dn-import` in `App.jsx`; sidebar entry **"Debit Note Import"** under the Import section.
- `pages/Document/DebitNoteList.jsx` — opening DN rows are clickable (→ dialog), labelled **"opening dn"**, Order No. blank.

### Storage of one imported debit note
One `debit_notes` row with:
- `supplier_id` resolved from `SUPPNO`, `supplier_name` = `SUPPNAME`
- `doc_no` = the 8-digit number (as int)
- `doc_date` = `DNDATE`
- `doc_type` = `'Debit Note'`, `doc_status` = `'Printed'`
- `reference` = `[BRANCH]OD[8digits]` (the full DN number)
- `currency_id` from `BILLCURCODE` (stored as string — `debit_notes.currency_id` is varchar)
- `base_currency_id` = 110 (PKR)
- `exchange_rate` = `EXRATE`
- `amount`, `billing_amount`, `base_amount`, `refund_amount` = `BILLDNAMT`
- `*_base` variants = `BILLDNAMT × EXRATE` (PKR)
- `pax_name` = `PAX1`, `remarks` = `REMARK`
- `is_opening` = 1, `import_batch_id` set, `je_generated` = 0

---

## 4. Phase Plan

| Phase | Scope |
|---|---|
| Phase 1 (done) | DN import — upload, preview, confirm, history, delete, template. |
| Phase 2 (done) | Opening DNs shown in **Documents → Debit Note tab** (clickable → detail popup, "opening dn" label). |
| Phase 3 (later) | Settling an opening DN via Supplier Payment (mirror of opening invoice / opening XO settlement). |
