# Supplier Payment (Payment Settlement) Module

Paying a supplier against their payable documents (costs/XOs, credit notes, debit
notes) — money can come from a **Form of Payment** (cash/cheque/bank/card), from the
supplier's **advance payments (supplier deposits)**, or both. The result is a printed
**payment settlement** (XO… number).

## 1. Where it lives
1. **Page**: `psfront/src/pages/PaymentsPage/Payments.jsx` — route `/payments`
   (**Payments → Supplier Payments**).
2. **Submit**: `POST /api/payment/settle` → `payment.controller.js` `settlePayment`.
3. **Printed document**: `/payment/settlement/:paymentNumber` (HTML preview / PDF);
   void via `POST /payment/settlement/:paymentNumber/void` (single) and
   `/payment/settlement/batchVoid`.
4. **Tables**: parent `payment_settlements` (payment number, supplier, issuing user,
   `status` Printed/Void) + child rows per source:
   - `payment_settlement_costs` — which costs/XOs were settled and for how much
   - `payment_settlement_payments` — the Form of Payment rows (pay type, amount,
     `bank_id`, `account_number`, cheque/card details)
   - `payment_settlement_deposits` — advance payments (supplier deposits) applied
   - `payment_settlement_overpayments`, `payment_settlement_debit_notes`,
     `payment_settlement_expenses` — other settlement sources

## 2. Page flow
1. Select supplier → their payable documents load (costs/XOs with outstanding
   amounts, credit/debit notes, supplier deposits).
2. Tick the documents to pay and enter the payable amounts.
3. Add one or more **Form of Payment** rows and/or apply available supplier deposits.
4. Submit → settlement is created with an XO payment number and the printed document
   opens.

## 3. Form of Payment (shared component)
1. The Form of Payment block is the shared component
   `psfront/src/components/Receipt/FormOfPayment.jsx` — also used by **Receipt**
   (invoice settlement), **Expense Payment**, **Internal Transfer** (both sides) and
   **Payment Requisition**. A fix here lands in all six flows.
2. The bank dropdowns (pay types 2, 6, 11, 12) list **bank accounts** as
   `bank name (account_number)`, keyed by the unique `bank_account_id`.
3. On select the component stores `bank_account_id`, the account's `bank_id`, its
   `account_number` and `chart_of_account_id`.
4. **Validation** keys on `bank_account_id` (the picked account), so manual-bank
   accounts pass validation correctly.
5. **Manual-bank accounts** (`bank_id = NULL`, typed name in `manual_bank_name`): the
   option label falls back to `bank?.name || manual_bank_name || "-"`. Previously it
   read `account.bank.name` unconditionally, which threw on the null bank and blanked
   the whole page in all six flows as soon as a bank pay type was selected (same root
   cause as the Advance Payment crash — see `ADVANCE_PAYMENT_MODULE.md` §3a).

## 4. Interaction with bank-account protection
1. A bank account cannot be **deleted or edited** while non-void settlements carry its
   `account_number` in `payment_settlement_payments` (company-scoped via the issuing
   user) — see `ADVANCE_PAYMENT_MODULE.md` §3a.8.

## 5. Open items
1. For a settlement paid from a **manual-bank** account, `bank_id` is NULL in
   `payment_settlement_payments`; how the printed settlement renders the bank name for
   that case has **not been audited** — no `bank_name` snapshot column exists here
   (unlike `supplier_deposits`).

## 6. Related
1. Advances used here are created in the Advance Payment module —
   `ADVANCE_PAYMENT_MODULE.md`.
2. Customer-side counterpart (receipts/deposits): `CUSTOMER_DEPOSIT_MODULE.md`.
