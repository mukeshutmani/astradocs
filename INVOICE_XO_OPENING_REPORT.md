# Opening Documents (report)

## Purpose

Lists the **imported opening documents** — opening invoices (`invoices.is_opening = 1`),
opening XOs (`costs.is_opening = 1`), opening credit notes (`credit_notes.is_opening = 1`)
and opening debit notes (`debit_notes.is_opening = 1`). The **Document** filter chooses
exactly **one** type, and the report shows that single section.

---

## 1. Filters

| Filter | Notes |
|---|---|
| **Document** | Dropdown: `Invoice`, `XO`, `Credit`, `Debit` (no "All"). Decides which single section is shown. |
| **Date** | Operator (`=`, `<`, `<=`, `>`, `>=`, `<>`, `between`) + date(s). Invoice → `invoice_date`; XO → `costs.created_at`; Credit/Debit → `doc_date`. |
| **Supplier** | Shown only for `XO` / `Debit`. |
| **Customer** | Shown only for `Invoice` / `Credit`. |
| **Branch** | The 2-letter branch code (`KH`, `TT`, `TP`, …). Matches the prefix of the document number (`KHOX…`, `KHOI…`, `KHOC…`, `KHOD…`). |

### Filter-combination rules (popups)

Supplier belongs to the **XO / Debit** side; Customer belongs to the **Invoice / Credit** side.

1. **Supplier** selected + Document not `XO`/`Debit` → popup: *"Please select XO or Debit Note for the Supplier filter."*
2. **Customer** selected + Document not `Invoice`/`Credit` → popup: *"Please select Invoice or Credit Note for the Customer filter."*

The frontend also auto-clears the supplier/customer value when the document type changes so
the wrong party filter can't be sent. The backend returns a `400` with the same message as a
safety guard.

---

## 2. Report Columns

All four sections use the **same 8-column layout** (only the party-column header differs):

| Section | Columns |
|---|---|
| **Opening XO** | `Document No`, `Date`, **`Supplier`**, `Currency`, `Ex. Rate`, `Amount`, `PAX`, `Status` |
| **Opening Invoices** | `Document No`, `Date`, **`Customer`**, `Currency`, `Ex. Rate`, `Amount`, `PAX`, `Status` |
| **Opening Credit Notes** | `Document No`, `Date`, **`Customer`**, `Currency`, `Ex. Rate`, `Amount`, `PAX`, `Status` |
| **Opening Debit Notes** | `Document No`, `Date`, **`Supplier`**, `Currency`, `Ex. Rate`, `Amount`, `Status` (no PAX) |

`Amount` is the imported document amount in its own currency (`costs.total_costing`,
`invoices.total_price`, `credit_notes.billing_amount`, `debit_notes.billing_amount`).
`PAX` is the imported passenger name (blank for XO). Each section shows a **Total** of the
Amount column.

Excel and PDF use the **same layout and columns**.

---

## 3. Technical Implementation

### Backend
- **Controller** (new, separate file): `psback/controllers/reports/invoiceXoOpeningReport.report.controller.js` → `getInvoiceXoOpeningReport`.
- Registered in `psback/controllers/reports/index.js`.
- **Route**: `POST /report/invoiceXoOpeningReport` in `psback/routes/report.route.js`, gated by `authenticate` + `permission("Invoice-Xo-Opening-Report")`.
- **PDF template**: `psback/views/pages/reports/invoice-xo-opening-report.ejs`.
- Data is company-scoped through the `import_batches` table (`company_id`, `batch_type` `XO` / `INV` / `CN` / `DN`).
- `report_type` = `opening-documents-report`. Route, controller file name and the `Invoice-Xo-Opening-Report` permission are unchanged (only the display title is "Opening Documents").

### Frontend
- **Page**: `psfront/src/pages/Report/InvoiceXoOpeningReport.jsx`.
- **API**: `getInvoiceXoOpeningReport` in `psfront/src/api/report.js`.
- **Route**: `reports/invoiceXoOpeningReport` in `App.jsx`.
- **Sidebar**: entry under the **History** group in `psfront/src/pages/Report/Report.jsx`, gated by the `Invoice-Xo-Opening-Report` permission.

### Request body
```
{
  documentType: "invoice" | "xo" | "credit" | "debit",
  dateFilter:   "blank" | "=" | "<" | "<=" | ">" | ">=" | "<>" | "between",
  startDate, endDate,
  supplier_id,        // XO / Debit filter
  customer_id,        // Invoice / Credit filter
  branchCode,         // 2-letter prefix
  type: "pdf" | "excel"
}
```

---

## 4. Setup Note

`Invoice-Xo-Opening-Report` is a **new permission**. Until it is created and granted to a
role, the sidebar link is hidden and the API returns `403`. Add it in the permissions admin
and assign it to the roles that should access this report.
