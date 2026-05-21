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
13. Access gated by `IMPORT_ENABLED` env flag.

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
| Phase 1 (this) | CN import — upload, preview, confirm, history, delete, template. |
| Phase 2 (later) | Opening CNs shown in Documents → Credit Note tab and Receipt Settlement. |
| Phase 3 (later) | Settling an opening CN via Receipt Settlement (mirror of opening invoice / opening XO settlement). |
