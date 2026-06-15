# GL Import — Opening Trial Balance (Excel)

## Purpose

Imports a company's **opening trial balance** for **one branch** from an Excel file and posts
it as a single **Opening JE** (a new third JE type alongside Manual JE and System JE). This is
what surfaces the opening account balances in the GL / reports. Mirrors the flow of the other
imports (Template / Upload → Preview → Confirm / History).

---

## 1. Excel Template

The uploaded sheet may carry title/meta rows above the header (the client's
"Opening Trail Balance" export puts the header on row 5) — the parser **locates the header
row dynamically** (the row containing `Key Account` and `Account Type`).

| Column | Notes |
|---|---|
| `Date` | Opening date, `DD-MM-YYYY`. Must be in the **same month/year** as `JE Period`. |
| `JE Period` | `MMYYYY` (e.g. `062026`). Accepts `62026` too (normalized to `062026`). |
| `Document Prefix` | Branch code (`KH`, `TT`, …). **One branch per file.** |
| `Key Account` | Account code — **must already exist** in the company chart of accounts. |
| `Account Name` | For reference. |
| `Opening Debit` | PKR, `0` if none. A row can't have both debit and credit. |
| `Opening Credit` | PKR, `0` if none. |
| `Rollup Detail` | `R` = rollup/header, `D` = detail/posting. **Only Detail rows are posted.** |
| `Account Type` | `A`/`L`/`R`/`E`/`S`/`C` (informational). |

---

## 2. Validation rules (all enforced at Preview & Confirm)

1. **One branch per file** — if `Document Prefix` differs across rows → blocked
   (*"Only one branch can be imported at a time"*). The branch must belong to the company.
2. **One JE Period per file** — all rows share one period.
3. **Account must exist** — every `Key Account` must exist in the logged-in company's chart of
   accounts (matched by code). Missing → error per row (*"… does not exist in your company's
   chart of accounts"*).
3a. **Account Type must match** — the Excel's Account Type letter (A/L/R/E/S/C) must match the
    matched account's real `account_type` (e.g. `A` ↔ `A/Assets`). Mismatch → row error; blank →
    *"Account Type is required"*.
4. **Debit = Credit** — total Opening Debit must equal total Opening Credit across **Detail rows**
   (the trial balance). Rollups are sums and are not added to this total.
   - This rule is **optional via a checkbox** in the upload area: *"Allow unbalanced import
     (Debit ≠ Credit)"* (default **off**). When **off**, an unbalanced file is **blocked** (file
     error) as before. When **on**, an unbalanced file is **allowed** and only raises a
     non-blocking **warning** (*"GL import is unbalanced — Total Opening Debit (X) does not equal
     Total Opening Credit (Y)."*); the Opening JE is posted as-is (its debit/credit will not match).
   - The flag is sent as a multipart field `allowUnbalanced=true` on `/gl-import/preview` and
     `/gl-import/confirm`; the validator moves the imbalance from `fileErrors` to `warnings`.
   - All other checks (account exists, one branch, one period, rollup = Σ details, Trade
     Debtors/Creditors reconciliation, date↔period) remain **mandatory** regardless of the checkbox.
5. **Rollup = Σ details** — each Rollup account's net opening must equal the sum of the net
   openings of its Detail descendants (walked via the chart-of-accounts parent chain,
   `account_number`). Multi-level supported.
6. **Control reconciliation** (matched by key OR name):
   - **Trade Debtors** (`151110` / "Trade Debtors") opening (debit) = **total Opening Invoices**
     for the branch (PKR = `Σ total_price × exchange_rate`).
   - **Trade Creditors** (`231130` / "Trade Creditors") opening (credit) = **total Opening XOs**
     for the branch (PKR = `Σ total_costing × exchange_rate`).
   - If a control account has a balance but the matching opening sub-ledger hasn't been imported
     → error (*"… import the Opening Invoices/XOs first"*).
   - (Opening Credit/Debit Notes are **not** netted in — invoices & XOs only.)
7. **Date ↔ JE Period** must be the same month and year.

---

## 3. What gets created on Confirm

1. **One `journal_batch`** with `batch_type = 'Opening JE'`, `status = 'Posted'`,
   `batch_no = <branchPrefix>OJ<8 digits>` (e.g. `TTOJ00000001`, its own sequence per branch),
   `journal_entry_period` = the file's period, `dated` = the file's Date, `branch_id` = the branch.
2. **One `journal_entry` per Detail account** with a non-zero opening: `chart_of_account_id` =
   matched account, `debit`/`credit` from the file, `gl_entity_id` = null (the per-customer/
   supplier opening already lives in the opening invoices/XOs — this is the GL/trial-balance side),
   `analysis_code1` = the OJ number. Rollups are **not** posted.
3. **One `import_batches` row** (`batch_type = 'GL'`, `batch_no = IMP-GL-####`) with
   `journal_batch_id` → the Opening JE, for History + delete.

Clicking the Opening JE in History opens an overlay listing the imported account lines
(Key Account, Account Name, R/D, Type, Debit, Credit + totals).

---

## 4. Technical Implementation

### Database (already migrated)
- `journal_batches.batch_type` enum extended to include `'Opening JE'`.
- `import_batches.batch_type` enum extended to include `'GL'`.
- `import_batches.journal_batch_id` (INT NULL, FK → `journal_batches.id`).

### Backend (new files)
- `psback/services/glImport.service.js` — Excel parse (dynamic header) + all validations.
- `psback/controllers/glImport.controller.js` — context build, preview, confirm (Opening JE
  posting), history, batch detail (overlay lines), delete, template. `<prefix>OJ<8>` generator.
- `psback/routes/glImport.route.js` — mounted at `/gl-import` in `index.js`.
- `models/import_batch.js` — `batch_type` enum widened + `journal_batch_id`; `models/index.js`
  associates `import_batch ↔ journal_batch`.
- Reuses `middlewares/importEnabledGate.js` (writes gated; history read-only stays available).

### API endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/gl-import/preview` | Parse + validate (no DB write) |
| POST | `/gl-import/confirm` | Re-validate + create Opening JE + import batch |
| GET | `/gl-import/batches` | Import history |
| GET | `/gl-import/batches/:id` | Opening JE lines (overlay) |
| DELETE | `/gl-import/batches/:id` | Delete batch + its Opening JE (blocked if JE voided) |
| GET | `/gl-import/template` | Blank Excel template |

### Frontend (new files)
- `psfront/src/api/glImport.js`
- `psfront/src/pages/GlImport/GlImportPage.jsx` (New Import / Import History / Template)
- `psfront/src/pages/GlImport/GlImportUpload.jsx` (preview with per-row validity + reconciliation cards)
- `psfront/src/pages/GlImport/GlImportHistory.jsx` (history + Opening JE overlay + delete)
- Route `iur/gl-import` in `App.jsx`; sidebar entry **"GL Import"** under the Import section.

---

## 5. Phase Plan

| Phase | Scope |
|---|---|
| Phase 1 (done) | GL import — template, upload, preview (full validation), confirm (post Opening JE), history (with Opening JE overlay), delete. |
| Phase 2 (next) | Surface the **Opening JE** in the GL **Journal Entries** section: it lists with the other JEs, and clicking the `TTOJ…` number opens the imported-accounts overlay. Ensure GL/Trial-Balance reports include `Opening JE` batches. |
