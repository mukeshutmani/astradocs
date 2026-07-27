# Security — Tenant Isolation (Cross-Company Data Leaks)

Tracks the audit of cross-company data leaks (broken tenant isolation / IDOR) in `psback`
and the fixes applied, section by section.

## Tenant model (how isolation should work)
1. Each logged-in user has `req.user.company_code` (their tenant) and `req.user.allowed_branch_ids`.
2. Every read/write of tenant data must be constrained by one of:
   - `company_code = req.user.company_code`, or
   - `branch_id` in `req.user.allowed_branch_ids` (helper `branchScope(req)` in `utils/tenantScope.js`), or
   - an ownership walk: service → order → user.company_code.
3. `authenticate` sets `req.user`. `permission("...")` only checks a feature flag — it does NOT enforce any company boundary.

## Two classes of leak found
- **Tier 1 — No authentication at all.** Routes/static mounts reachable with no login; anyone can read a document by guessing an id/number.
- **Tier 2 — Logged in, but no company check.** Handlers fetch by raw `id` (or list with no filter), so a user of Company A can read/modify Company B's data.

## Fix log

### 2026-07-07 — Tier 1, first pass (DONE)
1. `utils/mailer.js` — `POST /api/send-invoice-email` now requires `authenticate`, and `companyCode`
   is taken from `req.user.company_code` (the request-body value is ignored). Closes an
   unauthenticated any-company invoice-PDF exfiltration + open mail-relay hole. Route is not used
   by the frontend (the app emails via the authenticated `POST /document/:documentNumber/send-email`).
2. `routes/dashboard.routes.js` — removed the unauthenticated `GET /dashboard/debug-revenue` route.
   Its controller has no company filter, so gating alone was insufficient; the route is a leftover
   test endpoint and is not used by the frontend.

### 2026-07-08 — Login-required document routes, slice 1: Invoice document viewer (implemented, pending UI verification)
Made the invoice/costing document render routes **login-required** so the raw URL cannot be
opened in another (unauthenticated) browser and there is no shareable link.
- `utils/docLink.js` (new) — `signDocToken` / `verifyDocToken` (JWT, `JWT_SECRET`). Used
  **server-side only** now (see below).
- `middlewares/docTokenAuth.js` (new) — accepts a normal **Bearer header** (the browser path) OR a
  signed `dt` token (used only by the server's own wkhtmltopdf fetches); 401 otherwise. Company
  scope comes from the verified user/token, never from `?company_code=`.
- `routes/document.route.js` — `GET /:documentNumber` and `GET /pdf/:documentNumber` now require
  `docTokenAuth`. (Other `/document/*` routes — header/footer/view/:key — deliberately NOT yet
  touched; `view/:key` is shared by ~30 report pages and is a later slice.)
- `services/pdf.js` + `controllers/document.controller.js` (`exportPdf`) — the internal wkhtmltopdf
  fetches of `/document/:num` attach a `dt` token (company-scoped where available; a plain service
  token for `generateDocumentPDF`). These URLs live server-side only and never reach a browser.
- **No `dt` token is exposed to the browser and there is no endpoint to obtain one** — so users
  cannot build a shareable link. (An earlier draft added `GET /auth/doc-token` + a URL `dt=`; that
  was removed because a token in the URL is copyable/shareable, which the requirement forbids.)
- Frontend `ViewDocument.jsx` + `api/document.js` — the invoice/costing **preview** is fetched via
  authenticated axios and shown with `srcDoc`; **Print/Download** fetch the PDF via authenticated
  axios and open it as a local `blob:` URL (tab opened synchronously first to avoid popup blockers).
  The `stamp_image` is a Base64 data URI, so the srcDoc preview renders identically.

Verify: open an invoice document → Preview/Print/Download work; then open
`/document/<invoice document number>` in another browser with **no login** → 401 (does not open).

### 2026-07-24 — Cross-company WRITE: `updateStatus` print/void flipped other companies' documents (FIXED)
`PUT /document/updateStatus/:documentNumber` (`controllers/document.controller.js`, `updateStatus`)
looked up documents by `document_number` only — no company filter. Because document numbers repeat
per company, one company printing (or voiding) their document updated **every company's** invoices
or costs with that number.
- Proven damage (company 1001): voided invoices KHIN00000003 / KHIN00000011 were silently flipped
  back to `Printed` when other companies printed their same-numbered invoices
  (fingerprint: many companies' rows sharing one `updated_at` second — 2026-06-22 09:43:01,
  2026-07-20 10:00:17 / 10:08:59, 2026-07-21 09:38:43, 2026-07-23 11:28:14). Verified against the
  01-Jul and 13-Jul backups and the 14-Jul staging clone.
- Fix: `Document.findAll` now joins `db.user` with `where company_code = req.user.company_code`
  (same pattern as `batchPrintDocuments`), and the follow-up `Document.update` targets the
  authorized row ids instead of `document_number`. The derived `invoiceIds` / `costIds` are
  therefore company-scoped too.
- Data repair still pending after deploy: company 1001 invoices id 6947 (KHIN00000003) and
  id 6958 (KHIN00000011) must be set back to `Void` in production, then JE void-reversals run.

## Pending (need a decision before coding)

### Tier 1 — document render routes + static file folders
These are opened by the browser via `window.open` / direct links (see `psfront/.../ViewDocument.jsx`),
so they **cannot** carry a Bearer token — adding `authenticate` would break invoice/receipt printing
and downloading. Affected:
- `invoice.route.js`: `generateInvoice`, `exportPdf`, `generateCreditNote(ByRefundNo)`,
  `generateDebitNote(ByRefundNo)`, `exportCreditNotePdf`, `generateReceiptSettlement(PDF)`,
  `receipt/downloadReport`.
- `payment.route.js`: `slip/:costId`, `generatePdf/:costId`, `cost/getPaymentSlip/:costId`,
  `settlement/:paymentNumber`, `getPaymentSettlement(PDF)`.
- `document.route.js`: `:documentNumber(/header|/footer)`, `pdf/:documentNumber`, `view/:key`.
- `deposit.route.js`: `customer_deposit/receipt/:receiptNumber`.
- `supplier_deposit.route.js`: `supplier_deposit/receipt/:paymentNumber`.
- `index.js`: `express.static` mounts for `/payments`, `/invoices`, `/documents`.

Recommended approach: **signed short-lived document links** — an authenticated endpoint mints a
time-limited signed token for a specific document + company; the render routes verify the token
(server-side company scope) instead of trusting a `company_code` query param. Frontend appends the
token to the `pdfUrl`. No Bearer header needed, so `window.open` keeps working.

### Tier 2 — by-id IDOR across controllers
Detail/update/delete/PDF handlers that fetch by raw id with no company/branch scope. The list
endpoints already scope correctly; the fix is to apply the same `company_code` / `branchScope`
constraint to the by-id handlers. To be done module by module (financial modules first).
