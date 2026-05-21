# Opening Invoice Import (Excel)

## Purpose

A new feature that lets the system import historical / opening invoices from another company's old system into Astra via an Excel file. The imported invoices act as **outstanding opening balances** so the new company can continue billing/settlement workflows inside Astra without losing what their customers already owe them.

This document covers **Invoice import only**. XO import will be covered in a separate document later.

---

## 1. Excel Template

### Sheet name: `INV`

### Columns

| Column | Type | Mandatory | Notes |
|---|---|---|---|
| `INVNO` | Text | Yes | Invoice number (digits only; see Rule 1) |
| `BRANCH` | Text | Yes | Branch code (e.g., `KH`, `TT`, `TP`) |
| `CUSTNO` | Text | Yes | Customer code, must already exist in system |
| `CUSTNAME` | Text | Yes | Customer name (case-insensitive match) |
| `INVDATE` | Date | Yes | Invoice date — format `DD-MM-YYYY` |
| `BILLCURCODE` | Text | Yes | Currency code (`PKR`, `USD`, etc.) |
| `EXRATE` | Number | Yes | Exchange rate (`1` for PKR; e.g., `282` for USD) |
| `BILLINVAMT` | Number | Yes | Total invoice amount (gross, in `BILLCURCODE`) |
| `PAX1` | Text | Yes | Passenger name(s) — see Rule 11 |
| `REMARK` | Text | No | Optional notes — saved as-is |

### Accepted file formats

1. `.xlsx` (primary)
2. `.xls` (legacy supported)
3. CSV is **not** supported.

---

## 2. Rules

### Rule 1 — Invoice Number Generation

1. The final invoice number in our system is built as: `[BRANCH] + OI + [8-digit number]`.
2. `OI` is the fixed type code for **Opening Invoice**.
3. `INVNO` may have an **optional letter prefix** which is stripped — only the number part is used.
   1. Allowed: leading letters then digits — e.g. `KHOI00501515` → `00501515`, `YI0076364` → `0076364`.
   2. Not allowed: letters **between or after** digits — e.g. `00TY736000` → validation error.
   3. Not allowed: no digits at all → validation error.
4. The number part is then normalised to 8 digits:
   1. **More than 8 digits** → drop the leftmost extras, keep only the **last 8 digits**.
      - Example: `2365769736` → `65769736` → final `TTOI65769736`.
   2. **Less than 8 digits** → pad with **leading zeros** to 8 digits.
      - Example: `675` → `00000675` → final `TTOI00000675`.
   3. **Exactly 8 digits** → use as-is.
      - Example: `67267826` → final `TTOI67267826`.
5. `INVNO` blank → validation error: *"Please enter invoice number for row X."*
6. `BRANCH` blank → validation error: *"Please enter branch for row X."*

### Rule 2 — Customer Matching

1. `CUSTNO` is looked up in our `customer` table for the active company.
2. If `CUSTNO` does **not exist** in our system → validation error: *"Customer `TAH690` not found in system — row X."*
3. If `CUSTNO` exists, `CUSTNAME` is compared:
   1. Comparison is **case-insensitive** (e.g., `tahir testing` = `Tahir Testing` = `TAHIR TESTING`).
   2. Leading / trailing **spaces are trimmed** before comparison.
   3. If the names still differ in spelling → validation error: *"Customer name mismatch for `TAH690` in row X."*
4. We **do not auto-create** customers from Excel. User must add the customer in the normal Customer module first.

### Rule 3 — Invoice Status

1. There is no status column in the Excel.
2. Every imported invoice is saved with `status = 'Printed'`.
3. This means imported invoices are **outstanding** and increase the customer's outstanding balance until a receipt/settlement is created against them.

### Rule 4 — GL / Journal Entry Impact

1. Imported invoices **do NOT create automatic journal entries**.
2. The GL side is handled separately via a **GL import file** (separate feature, not in scope here).
3. Import only writes to the `invoices` (and supporting) tables — no `journal_entry` rows are inserted.

### Rule 5 — Order & Service Linkage

1. Imported invoices have **no `order_id`** — they are stored as standalone rows.
2. Service type is fixed as **`Miscellaneous Domestic`** for all imported invoices.
3. In reports / listings, these are tagged as **"Opening Invoice" / "Opening Balance"**.

### Rule 6 — Currency Handling

1. If `BILLCURCODE = PKR` → `EXRATE` must equal `1`.
   - `BILLINVAMT` is the PKR amount and is saved as-is.
   - If `EXRATE ≠ 1` when `BILLCURCODE = PKR` → validation error.
2. If `BILLCURCODE` is a foreign currency (USD / EUR / SAR / …):
   - `EXRATE` must be `> 0` and not blank.
   - The PKR-equivalent amount stored = `BILLINVAMT × EXRATE`.
   - Original `BILLINVAMT` + `BILLCURCODE` + `EXRATE` are also saved for reference.
3. Currently the client will send only PKR rows, but the logic must support foreign currency for future use.

### Rule 7 — `is_opening` Flag

1. A new boolean column `is_opening` is added to the `invoices` table.
2. Default = `0` (normal invoice).
3. Set to `1` only for imported (opening) invoices.
4. Reports and downstream modules use `is_opening = 1` as the primary identifier of opening invoices (preferred over string matching on the `OI` substring).

### Rule 8 — Date Format

1. `INVDATE` column must always be in **`DD-MM-YYYY`** format.
2. If the value is in any other format (e.g., `12-15-2026` MM-DD-YYYY) → validation error: *"Date format is not correct in row X — must be DD-MM-YYYY."*
3. Impossible dates (e.g., `31-02-2026`) are also rejected.

### Rule 9 — Duplicate Check

1. Duplicate uniqueness is decided by **branch code + the 8-digit number**, ignoring the type prefix (`IN` / `OI` / `OX` / etc.).
2. **Within the same Excel file** — if two rows produce the same final invoice number → validation error showing both row numbers and the invoice number highlighted in **red**: *"Duplicate invoice number `TTOI00000078` found in row 5 and row 12."*
3. **Against existing DB** — if any existing invoice in the DB for the same branch has the same 8-digit number (regardless of type code), the import row fails: *"Invoice number `TTOI00000078` already exists in system for branch TT — row X."*
4. Different branches with same 8-digit number → allowed (e.g., `TTOI00000078` and `KHIN00000078` co-exist).

### Rule 10 — Amount & Display

1. `BILLINVAMT` is the **total gross amount** of the invoice (tax already included).
2. No separate tax / FED / WHT columns are needed.
3. Imported invoices behave **differently from normal invoices** in the UI:
   1. They do NOT have a full printable invoice document.
   2. When the user **clicks or hovers** on an opening invoice number anywhere in the system, an **overlay / dialog** appears.
   3. The dialog shows: invoice number, customer name, invoice date, total amount + currency, PAX names, REMARK.

### Rule 11 — PAX1 & REMARK Handling

1. `PAX1` cell is stored in DB **exactly as written in Excel** (raw text, no parsing).
2. `REMARK` cell is stored in DB **exactly as written in Excel**, no validation, no transformation.
3. **Overlay display logic for PAX1**:
   1. If the value contains a **comma** → split into separate pax labels.
      - e.g., `ZIA ULLAH, MUKESH` → overlay shows `PAX1: ZIA ULLAH`, `PAX2: MUKESH`.
   2. If the value has the `NAME X N` pattern (no comma) → show as one pax with the count suffix.
      - e.g., `ZIA ULLAH X 3` → overlay shows `PAX1: ZIA ULLAH X 3` (meaning one named passenger, total 3 travelers).
   3. Plain text → display as one pax line.
4. `PAX1` is **mandatory**. Blank cell → validation error.
5. `REMARK` displays below pax names in the same overlay if present.

### Rule 12 — Import Batch & History

1. Every import action creates a single **Import Batch** record in DB capturing:
   - Batch ID, import date/time, file name, importing user, row count, batch type (INV / XO).
2. A new **Import** tab is added to the UI, with two sub-tabs: **INV** and **XO**.
3. History list columns: **Import Date**, **File Name**, **Imported By (username)**, **Delete** button.
4. Clicking a history row → opens the batch detail view showing all invoices created by that batch.
5. **Delete batch rules**:
   1. If any invoice in the batch already has a **receipt / settlement** against it → delete is **blocked** with message: *"Cannot delete — invoice `TTOI00000078` has a settlement. Please void the settlement first."*
   2. Once all settlements are voided, delete removes the batch and all its invoice rows in one click.

### Rule 13 — Import Flow (Preview-then-Import)

1. The flow is **two-step**:
   1. **Step 1 — Upload + Preview**: User uploads file → system parses + validates → shows preview screen.
   2. **Step 2 — Confirm**: User clicks **Confirm Import** → only then rows are written to DB.
2. **Preview screen layout**:
   1. **Summary header (cards)** at the top:
      - Total branches included (e.g., `TT, KH, TP`).
      - Total rows / total invoices in the file.
      - Total amount (sum of `BILLINVAMT`) with currency.
      - File name and upload time.
   2. **Row table** below:
      - **Green** rows = valid (will be imported).
      - **Red** rows = invalid, with reason shown in a tooltip / column (e.g., *"Customer not found"*, *"Date format wrong"*, *"Duplicate invoice number"*).
3. **All-or-nothing rule**:
   1. If **any** row has a validation error → entire file is **rejected**.
   2. Popup message: *"Please correct the errors first and re-upload. X rows have issues — see preview for details."*
   3. User must fix the Excel and upload a clean file.
4. **Cross-branch support**: One file may contain multiple branches mixed (`KH`, `TT`, `TP` rows). System reads `BRANCH` per row and imports into the correct branch automatically.

### Rule 14 — Editing / Voiding After Import

1. **Editing**: Imported invoice data fields (amount, customer, date, pax) are **read-only**. To correct a wrong field, the user must **delete the import batch** (if no settlements yet) and **re-import** a corrected Excel.
2. **Voiding**: Allowed. User can void an individual imported invoice the same way as a normal invoice. Status → `Void`, customer outstanding reduces by that amount.
3. **Settlements**: A receipt / payment / Manual JE can be created against an imported invoice exactly like a normal invoice. Status updates to `Settled` / `Partially Settled` accordingly.
4. After import, the invoice follows the **same workflow** as normal invoices for void / settlement / receipt / Manual JE.

### Rule 15 — Single Currency Per File

1. One file = **one currency only**.
2. If the file contains mixed currencies → validation error: *"Multiple currencies found in file — please split into one file per currency."*
3. Future enhancement: multi-currency in one file (not in scope now).

### Rule 16 — Net Outstanding Only

1. The client sends **only outstanding amount** in `BILLINVAMT`.
2. Fully paid invoices from the old system are **not included** in the file.
3. Partially paid invoices are sent with the **remaining (knockoff) amount** only.
4. Past receipts / payments are NOT imported — out of scope.

---

## 3. Validation Summary (popup messages)

| Trigger | Popup message |
|---|---|
| `INVNO` blank | *"Please enter invoice number for row X."* |
| `INVNO` has letters between digits | *"INVNO is invalid — letters are only allowed before the number, not between digits"* |
| `BRANCH` blank | *"Please enter branch for row X."* |
| `CUSTNO` not found | *"Customer `TAH690` not found in system — row X."* |
| `CUSTNAME` mismatch | *"Customer name mismatch for `TAH690` in row X."* |
| Wrong date format | *"Date format is not correct in row X — must be DD-MM-YYYY."* |
| Invalid date (e.g., 31-02) | *"Invalid date in row X."* |
| Duplicate in same file | *"Duplicate invoice number `TTOI00000078` found in row 5 and row 12."* (both rows red) |
| Duplicate vs DB | *"Invoice number `TTOI00000078` already exists in system for branch TT — row X."* |
| EXRATE ≠ 1 for PKR | *"EXRATE must be 1 when BILLCURCODE = PKR — row X."* |
| EXRATE blank/≤0 for foreign currency | *"EXRATE must be greater than 0 for foreign currency — row X."* |
| `BILLINVAMT` blank or ≤ 0 | *"Invalid amount in row X."* |
| `PAX1` blank | *"PAX1 is required — row X."* |
| Mixed currencies in same file | *"Multiple currencies found in file — please split into one file per currency."* |
| Any errors present | *"Please correct the errors first and re-upload. X rows have issues — see preview for details."* |

---

## 4. Permissions & Access

1. Access is gated by an **environment flag** (e.g., `IMPORT_ENABLED=true` in `.env`).
2. For now only the developer enables this flag during the import session, then turns it back off.
3. No UI-level permission control is built in the first phase. A normal user-permission flow can be added in a later phase.

---

## 5. Phase Plan

| Phase | Scope |
|---|---|
| **Phase 1 (now)** | Build the import feature itself — Excel upload, preview, validation, save to DB, import history tab with delete batch. |
| **Phase 2 (later)** | Update downstream modules to handle imported invoices specifically: `[Opening]` badge in listings, AR Aging behavior, Customer Outstanding screen, Receipt Settlement picker, Daily Settlement Report, invoice listing filters, etc. |
| **Phase 3 (later)** | XO import feature (separate Excel sheet / column set — TBD). |
| **Phase 4 (later)** | GL import feature (separate file). |

---

## 6. Technical Implementation (Phase 1 — built)

### Database
1. New table `import_batches` — one row per Excel upload.
2. New columns on `invoices`: `is_opening` (TINYINT, default 0) and `import_batch_id` (INT, FK to `import_batches`).
3. Migration SQL: `psback/migrations/add_opening_invoice_import.sql`.

### Backend (all new files — no existing controller/service modified)
1. `psback/models/import_batch.js` — Sequelize model (registered in `models/index.js`).
2. `psback/services/invoiceImport.service.js` — Excel parsing + row validation (pure logic).
3. `psback/controllers/invoiceImport.controller.js` — endpoints.
4. `psback/routes/invoiceImport.route.js` — routes (mounted at `/invoice-import` in `index.js`).
5. `psback/middlewares/importEnabledGate.js` — blocks all endpoints unless `IMPORT_ENABLED=true`.
6. Model-only additive change: `is_opening` + `import_batch_id` declared on `psback/models/invoice.js`.

### API endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/invoice-import/preview` | Upload Excel → parse + validate → return preview (no DB write) |
| POST | `/invoice-import/confirm` | Re-validate + write all rows (all-or-nothing) |
| GET | `/invoice-import/batches` | Import history list |
| GET | `/invoice-import/batches/:id` | Invoices created by one batch |
| DELETE | `/invoice-import/batches/:id` | Delete a batch (blocked if any invoice has a settlement) |
| GET | `/invoice-import/template` | Download the blank Excel template (`INV` sheet + headers + sample rows + Instructions sheet) |

### Template tab
1. The Invoice Import page has a **Template** tab with a **Download Template** button.
2. The template `.xlsx` is generated by the backend (`services/importTemplate.service.js`) from the same column list — it can never drift out of sync with the validation rules.
3. Give this file to a client so they know exactly which sheet name, columns, and formats to send back.

### Frontend (all new files)
1. `psfront/src/api/invoiceImport.js` — API client.
2. `psfront/src/pages/InvoiceImport/InvoiceImportPage.jsx` — parent page (New Import / Import History tabs).
3. `psfront/src/pages/InvoiceImport/InvoiceImportUpload.jsx` — upload + preview.
4. `psfront/src/pages/InvoiceImport/InvoiceImportHistory.jsx` — history list + batch detail + delete.
5. Route `iur/invoice-import` added in `App.jsx`; sidebar entry "Invoice Import" added under the Import section.

### Storage of an imported invoice
1. One `services` row (Miscellaneous Domestic type, `order_id` = NULL, `description` = PAX text).
2. One `invoices` row linked to that service: `status` = Printed, `is_opening` = 1, `import_batch_id` set, `currency` = currency_code id, `exchange_rate` = EXRATE, `total_amount`/`total_price` = BILLINVAMT.

### Access
1. Gated by `IMPORT_ENABLED` in `psback/.env`. Set to `true` to use the feature, `false` to disable.

### Phase 2 — Documents → Invoice tab (built)
1. Opening invoices now appear in the **Documents → Invoice** list alongside normal invoices.
2. New service `psback/services/openingInvoice.service.js` — `getOpeningInvoicesAsDocuments()` shapes opening invoices like Document rows; company-scoped via the import batch.
3. `document.controller.js` `getDocuments` invoice branch appends these rows (existing normal-invoice query untouched).
4. In the list, an opening invoice number is **clickable** → opens a centered detail popup (`psfront/src/components/OpeningInvoiceDialog.jsx`) showing customer, date, amount, status, PAX, remark, branch, batch.
5. Small text **"opening invoice"** shown under the invoice number. Order No. column is blank for opening invoices.

### Phase 2 — Receipt Settlement (built)
1. Opening invoices now appear in the **Receipt Settlement** screen's invoice list for the selected customer.
2. `openingInvoice.service.js` → `getOpeningInvoicesForSettlement(companyCode, customerNumber)` fetches the customer's opening invoices (company-scoped).
3. `customer.controller.js` → `getOrdersByCustomerId` adds them into the grouped invoice list; outstanding = total − receipt settlements − Manual JE; only invoices with a balance show.
4. `invoice.controller.js` → `settleReceipt` now handles invoices with no order: the branch is derived from the invoice number prefix (e.g. `KH` from `KHOI...`) for company validation and receipt numbering. One targeted change; the rest of the function is unchanged.
5. In the settlement invoice table, an opening invoice number is **clickable** → opens the same `OpeningInvoiceDialog` popup, with small **"opening invoice"** text under the number (consistent with the Documents → Invoice tab). The `opening` detail payload is supplied by `getOrdersByCustomerId`.
6. Note: the printed receipt-settlement document for an opening-invoice settlement uses the default template (branch-specific template lookup is skipped because there is no order/branch link — handled by existing optional-chaining guards).

### Phase 2 — Customer Account Statement report (built)
1. A new **"Opening Invoices"** section appears in the report, right after **General / Other Bookings** (both PDF and Excel).
2. Columns: Date, Invoice, Pax, Branch, Remarks, Net — with a section total.
3. New summary line **"Add Opening Invoices"** below "Total Sales"; Net Balance now includes it.
4. Opening invoices are **never** counted in "Opening Balance B/F" or "Total Sales" — only in their own section + summary line (no double-count).
5. The section **respects the report date filter** — only opening invoices whose `invoice_date` falls in the selected range are shown.
6. `openingInvoice.service.js` → `getOpeningInvoicesForCustomers(companyCode, customerNumbers)` fetches them company-scoped; `getCustomerAccountStatementReport` builds the section + total per customer. The existing order-walking logic was not modified.

---

## 7. Open Items / Future Decisions

1. **XO import** — Excel column set still to be confirmed with client.
2. **Phase 2 downstream behaviour** — exact UI changes for each report / listing.
3. **GL import** — out of scope, separate feature.
4. **Auto-create customer** option — currently blocked; may be revisited if the client requests bulk customer setup.
5. **Maker-checker workflow** — not in Phase 1; importing user directly commits.
