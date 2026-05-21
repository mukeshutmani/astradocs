# Invoice/XO Opening Report

## Purpose

Lists the **imported opening documents** — opening invoices (`invoices.is_opening = 1`) and
opening XOs (`costs.is_opening = 1`) — brought into Astra via the Opening Invoice Import and
Opening XO Import features. One report, two possible sections.

---

## 1. Filters

| Filter | Notes |
|---|---|
| **Document** | Dropdown: `Invoice`, `XO`, `All`. Decides which section(s) are shown. |
| **Date** | Operator (`=`, `<`, `<=`, `>`, `>=`, `<>`, `between`) + date(s). Invoice rows filter on `invoice_date`, XO rows on the XO date (`costs.created_at`). |
| **Supplier** | XO side only. |
| **Customer** | Invoice side only. |
| **Branch** | The 2-letter branch code (`KH`, `TT`, `TP`, …). Matches the prefix of the document number (`KHOX…`, `KHOI…`). |

### Filter-combination rules (popups)

Supplier belongs to the XO side; Customer belongs to the Invoice side.

1. **Supplier** selected + Document = `Invoice` or `All` → popup: *"Please select Document type XO."*
2. **Customer** selected + Document = `XO` or `All` → popup: *"Please select Invoice for Document type."*
3. Document = `All` + no Customer + no Supplier → report shows **both sections** (Opening XO + Opening Invoices).
4. Document = `XO` → XO section only. Document = `Invoice` → Invoice section only.

The popup is shown by the frontend before the request; the backend also returns a `400`
with the same message as a safety guard.

---

## 2. Report Columns

### Opening XO section
`XO No`, `Date`, `Supplier`, `Currency`, `Ex. Rate`, `Amount`, `Status`

### Opening Invoices section
`Invoice No`, `Date`, `Customer`, `Currency`, `Ex. Rate`, `Amount`, `PAX`, `Status`

`Amount` is the imported document amount in its own currency (`costs.total_costing` /
`invoices.total_price`). Each section shows a **Total** of the Amount column.

Excel and PDF use the **same layout and columns**.

---

## 3. Technical Implementation

### Backend
- **Controller** (new, separate file): `psback/controllers/reports/invoiceXoOpeningReport.report.controller.js` → `getInvoiceXoOpeningReport`.
- Registered in `psback/controllers/reports/index.js`.
- **Route**: `POST /report/invoiceXoOpeningReport` in `psback/routes/report.route.js`, gated by `authenticate` + `permission("Invoice-Xo-Opening-Report")`.
- **PDF template**: `psback/views/pages/reports/invoice-xo-opening-report.ejs`.
- Data is company-scoped through the `import_batches` table (`company_id`, `batch_type` `XO` / `INV`).

### Frontend
- **Page**: `psfront/src/pages/Report/InvoiceXoOpeningReport.jsx`.
- **API**: `getInvoiceXoOpeningReport` in `psfront/src/api/report.js`.
- **Route**: `reports/invoiceXoOpeningReport` in `App.jsx`.
- **Sidebar**: entry under the **History** group in `psfront/src/pages/Report/Report.jsx`, gated by the `Invoice-Xo-Opening-Report` permission.

### Request body
```
{
  documentType: "invoice" | "xo" | "all",
  dateFilter:   "blank" | "=" | "<" | "<=" | ">" | ">=" | "<>" | "between",
  startDate, endDate,
  supplier_id,        // XO filter
  customer_id,        // Invoice filter (resolved to customer_number server-side)
  branchCode,         // 2-letter prefix
  type: "pdf" | "excel"
}
```

---

## 4. Setup Note

`Invoice-Xo-Opening-Report` is a **new permission**. Until it is created and granted to a
role, the sidebar link is hidden and the API returns `403`. Add it in the permissions admin
and assign it to the roles that should access this report.
