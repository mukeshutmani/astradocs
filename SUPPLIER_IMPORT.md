# Supplier Bulk Import (CSV)

Endpoint handled by `addSuppliers` in `psback/controllers/supplier.controller.js`.
Lets an Admin-Agt user upload a CSV to create many suppliers at once for their own company.

## Validation rules
- Supplier number (`supp_no`): 1–12 characters (letters, numbers, `-`, `_`, `@`, `&`).
- Supplier name (`supp_name`): required.
- Supplier type id: required, must match an existing type.
- Status: `active`, `inactive`, `blacklist`, or `suspend`.
- Boolean fields accept `true`/`false`, `1`/`0`, or empty (= false).

## Duplicate detection (per company)
Both duplicate checks are scoped to the importing user's company (`req.user.company_code`):

1. **By number** — a row is a duplicate only if its `supp_no` already exists **for the same company**.
   Supplier numbers are unique **per company**, not globally — the same `supp_no`
   (e.g. `SUPP00000001`) legitimately exists under several companies at once.
2. **By name** — a row is a duplicate only if its `supp_name` (case-insensitive) already
   exists for the same company.

Rows already present (by number or name) are skipped; the rest are inserted via `bulkCreate`.
If every row is a duplicate, the API returns `400` with
`{ message: "No new suppliers to import.", duplicatesByNumber, duplicatesByName, reasons }`.

### History
- Fixed: the by-number check previously queried `supp_no` with **no company filter**, so it
  matched supplier numbers belonging to other companies and falsely reported all rows as
  duplicates (e.g. importing into a company with zero suppliers still failed). It now filters
  by `$user.company_code$`, mirroring the by-name check.
