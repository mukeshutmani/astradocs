# Order Detail — Receipt / Payment Tab

## Overview

The Receipt / Payment tab on the Order Detail page (`/order/:id`) lists every financial event that touches the order's invoices and cost (XO) documents. It is a read-only audit view — users navigate to the underlying document via the linked Document No.

**Frontend**: `psfront/src/pages/OrderDetail.jsx` (Receipt/Payment `TabsContent`, ~line 3658)

## Columns

| Column | Purpose |
|---|---|
| Type | Receipt / Payment / Customer Deposit / Advance Payment / Supplier Overpayment / **Manual JE** |
| Document No. | Clickable link to the source document. For Manual JE it links to `/gl/JournalEntries/batch/<batch_no>`. For everything else it links to `/documents/<docNumber>?type=<linkType>&status=<status>&branch_id=<id>`. |
| Status | Source document's status (`Printed`, `Partially Settled`, `Posted`, `Void`, etc.) |
| Date | `created_at` of the source document, or `updated_at` if the source is `Void`. Manual JE rows use the row's `transaction_date`, falling back to `journal_batches.dated`. |
| Amount | Amount in PKR (or document currency where the source is already PKR-converted). |

## Where each row type comes from

| Type | Source | Notes |
|---|---|---|
| Receipt | `receipt_settlement_invoice → receipt_settlement` rows on each invoice | Grouped by `receipt_number` across services |
| Payment | `payment_settlement_cost → payment_settlement` rows on each cost | Grouped by `payment_number` across services |
| Customer Deposit | `receipt_settlement.receipt_settlement_deposits → customer_deposit` (via the receipt that consumed it) | Deduped by `receipt_number + deposit.id` |
| Advance Payment | `payment_settlement.payment_settlement_deposits → supplier_deposit` | Only when `amount > 0` |
| Supplier Overpayment | `payment_settlement.payment_settlement_overpayments → source_settlement (payment)` | Only when `amount > 0` |
| **Manual JE** | `journal_entries.analysis_code1 ∈ (this order's invoice_numbers + XO document_numbers)`, grouped by batch | See section below |

Void settlements/deposits/overpayments are silently filtered out before grouping.

## Manual JE rows — details

### Backend
- Endpoint: `GET /journalEntry/by-order/:orderId` → `journal_entry.controller.js → manualJeByOrder`
- Steps:
  1. Collect all `invoice.invoice_number`s belonging to this order's services (status ≠ `Void`).
  2. Collect all `documents.document_number`s of type `costing` whose cost belongs to this order's services (cost status ≠ `Void`).
  3. Query `journal_entries` where `analysis_code1 IN (those numbers)` AND row description NOT LIKE `'VOID REVERSAL -%'`, joined to `journal_batches` filtered to `batch_type = 'Manual JE'` AND `status ≠ 'Void'`. This is the same `liveManualJeWhereClause` filter used by `services/manualJeAdjustment.js`, so a voided JE plus its reversal both drop out and net the order back to its prior state.
  4. Group rows by `journal_batch_id`. Per batch:
     - `debitSum = Σ debit` over matching rows
     - `creditSum = Σ credit` over matching rows
     - `amount = max(debitSum, creditSum)` — works whether the JE has only one leg tagged to this order (single side sums; the other is 0) or both legs tagged (sums match → no double-counting of balanced pairs).
- Response: `[{ batch_id, batch_no, status, dated, transaction_date, amount }]`

### Frontend
- API client: `psfront/src/api/journal_entry.js → getManualJeByOrder(orderId)`
- State: `manualJeEntries` in `OrderDetail.jsx`, fetched once when the order loads (alongside `fetchOrderDetailsLite()`).
- Render: each JE batch is mapped to a `{ type: 'Manual JE', manualJe }` item and concatenated to `finalSettlements` so it appears after the receipts/payments/deposits.
- Link target: `/gl/JournalEntries/batch/<batch_no>` (the existing JE batch detail page).

## Filter parity with other JE integrations

The `liveManualJeWhereClause` used here is identical to the one in `services/manualJeAdjustment.js`. That gives the order tab the same view of "active JE adjustments" that:

- `sumManualJeAdjustment(invoice_number)` returns for the Settlement-page listing (`customer.controller.js → getOrdersByCustomerId`)
- `sumManualJeAdjustment(xo_document_number)` returns for the Supplier-Payment-page listing (`service.controller.js → getPaymentCostsBySupplierId`)
- `recalculateInvoiceStatusByNumber` / `recalculateCostStatusByDocNumber` consume when flipping `invoice.status` / `cost.status` after a JE save/edit/void.

So a JE that flips an invoice to `Partially Settled` in the AR module will also appear in this order tab, with the same amount.

## Edge cases

1. **Cross-order JE batches**: a single Manual JE batch can touch invoices/costs of multiple orders. This view shows only the **portion** that matches THIS order's refs — the per-batch `amount` is summed across matching rows only.
2. **Multi-leg JE within one order**: if a JE batch has two rows both tagged to docs of this order (e.g. DR KHIN…, CR KHXO…), `max(debitSum, creditSum)` returns the single side's value, not the doubled total.
3. **Voided JE + reversal**: both the original batch and the auto-created `VOID REVERSAL -` reversal batch are filtered out — the order tab simply stops showing the JE, mirroring how it's removed from the JE adjustment math.
4. **Cross-company invoice/XO number collision**: `analysis_code1` is matched as a string, so if another company happens to have an invoice/XO with the same number their JE would also surface here. Pre-existing concern across the whole Manual JE integration; not specific to this tab.

## Related docs

- `docs/JOURNAL_ENTRIES_PLAN.md` (item 12) — the underlying Manual JE adjustment service.
- `docs/RECEIPT_SETTLEMENT.md` — receipt settlement listing integration.
- `docs/SUPPLIER_PAYMENT_SETTLEMENT.md` — supplier payment listing integration.
