# Customer → Terms → Finance Section

## Overview

The Finance section appears on the Customer edit page (`/customers/edit-customer/:id`) inside the **Terms** tab. It displays a high-level financial snapshot of the customer with five read-only fields:

| Field | Source |
|---|---|
| Total Invoices | Sum of all `Printed` / `Partially Settled` / `Settled` invoices belonging to this customer (PKR, FX-converted via `invoice.exchange_rate`). |
| Total Receipts & Deposits | G/L-account receipt payments + customer deposit amounts (PKR). |
| Total Refund | Sum of `refund_amount` (or `amount`) on non-Void credit notes for this customer. |
| **Total JEs** | Sum of live Manual JE adjustments (debit + credit per row) tagged via `analysis_code1` to any of this customer's invoice numbers (PKR). |
| Current Outstanding | `Total Invoices − Total Refund − Total Receipts & Deposits − Total JEs` |

## Files

| Layer | File | Purpose |
|---|---|---|
| Frontend page | `psfront/src/pages/CustomerPage/CreateCustomer.jsx` | Reads `financialSummary` from API, passes individual values into `FinanceFields`. |
| Frontend component | `psfront/src/components/CustomerForm/FinanceFields.jsx` | Renders the five read-only inputs. |
| Backend controller | `psback/controllers/customer.controller.js → getCustomer` | Computes `financialSummary` and returns it alongside `customer`. |
| Backend service | `psback/services/manualJeAdjustment.js → sumManualJeAdjustmentForMany` | Batched helper that sums Manual JE adjustments across many refs in one DB roundtrip. |

## Backend: how each value is computed

`getCustomer` runs the following in sequence (inside `customer.controller.js`):

1. **Total Invoices** — direct SQL on `invoices` joined to `services → orders` filtered to this `customer_id`, statuses `IN ('Printed', 'Settled', 'Partially Settled')`. Each invoice's PKR amount is computed via the shared `calculateInvoiceAmount` helper (`services/customer_balance_calculator`).
2. **Total Refund** — `credit_notes.refund_amount` (or `amount` fallback), filtered to non-Void notes whose `customer_id` matches OR whose `to` field matches the customer name.
3. **Total Receipts & Deposits** — `combinedReceipts = totalSettlementsAmount + totalDepositAmount`, where:
   - `totalSettlementsAmount` = sum of `receipt_settlement_payment.base_amount`/`.amount` over receipts whose payment has a G/L settle account AND is NOT a deposit-backed row.
   - `totalDepositAmount` = sum of `customer_deposit.amount × exchange_rate` for non-Void deposits.
4. **Total JEs** — `sumManualJeAdjustmentForMany(invoiceNumbers)`. The helper applies `liveManualJeWhereClause` (`batch_type = 'Manual JE'`, batch `status ≠ 'Void'`, row description NOT LIKE `'VOID REVERSAL -%'`) and sums `debit + credit` per matched row. Returns 0 when the customer has no invoices.
5. **Current Outstanding** — `totalInvoicesAmount − customer_refund_amount − combinedReceipts − totalManualJEs`.

The response shape:

```json
{
  "customer": { ... },
  "financialSummary": {
    "totalInvoicesAmount": ...,
    "combinedReceipts": ...,
    "customer_refund_amount": ...,
    "totalManualJEs": ...,
    "outstandingBalance": ...,
    "criteria": { ... }
  }
}
```

## Frontend: state flow

```
getSingleCustomer(id)
  → res.data.financialSummary
    → setFinancialSummary
    → setCustomerRefundAmount
    → setCurrentOutstandingBalance
    → setTotalManualJEs
      → <FinanceFields
          FinancialSummary={financialSummary}
          CusRefAmount={customerRefundAmount}
          TotalOutstanding={currentOutstandingBalance}
          TotalManualJEs={totalManualJEs} />
        → values.total_manual_jes
        → readonly input "Total JEs"
```

`FinanceFields` mirrors all incoming props into local `values` state and writes them back into `formValues.terms` so the values persist if the user submits the form (they remain read-only inputs).

## Filter parity with other JE integrations

`sumManualJeAdjustmentForMany` reuses the same `liveManualJeWhereClause` used by:

- `sumManualJeAdjustment` — single-ref version (used in receipt-settlement listing, supplier-payment listing, Order Detail tab, status recalculation).
- `recalculateInvoiceStatusByNumber` / `recalculateCostStatusByDocNumber` — flip `invoice.status` / `cost.status` after a JE save/edit/void.

So a JE that flips an invoice to `Partially Settled` in the AR module contributes the same PKR amount to this section. A voided JE (and its `VOID REVERSAL -` reversal batch) both drop out and the customer's outstanding returns to its pre-JE state automatically.

## Edge cases

1. **Customer with no invoices** — `sumManualJeAdjustmentForMany` short-circuits to `0`.
2. **Cross-company invoice-number collision** — if another company has an invoice with the same number, its JE rows would contribute here as well. Pre-existing concern across the whole Manual JE integration (filter is by `analysis_code1` string); not unique to this section.
3. **Multi-leg JE within this customer's invoices** — only one leg is normally tagged to a customer invoice (the other leg is the offset GL account or an XO). So summing `debit + credit` per row returns the JE's effect once, not doubled.

## Related docs

- `docs/JOURNAL_ENTRIES_PLAN.md` (item 12) — the underlying Manual JE adjustment service.
- `docs/RECEIPT_SETTLEMENT.md` — receipt settlement listing JE integration.
- `docs/SUPPLIER_PAYMENT_SETTLEMENT.md` — supplier payment listing JE integration.
- `docs/ORDER_DETAIL_RECEIPT_PAYMENT_TAB.md` — Order Detail tab JE integration.
