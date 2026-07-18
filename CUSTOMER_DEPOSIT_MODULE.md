# Advanced Deposit from Customer (Customer Deposit) Module

Taking money from a customer in advance (a deposit/credit on account) before any
invoice exists. The deposit is held and later settled against the customer's invoices.

## 1. Where it lives
1. **Page**: `psfront/src/pages/RecieptPage/AdvancedDeposit.jsx` — route
   `/receipt/apply-deposit` (under **Receipt → Advanced Deposit**).
2. **Listing**: the **Deposit** tab on the Documents page (`DepositList.jsx`).
3. **Printed document**: `type=customer_deposit`
   (`psback/views/pages/customerDepositDocument.ejs`, rendered by `getDepositDocument`
   in `deposit.controller.js`); table `customer_deposits`.

## 2. Data model (same as supplier deposit)
1. `amount` = entered amount in the **entered currency**.
2. `exchange_rate` = entered-currency → **base-currency** rate (1 if entered = base).
3. `base_amount` = `current_amount` = `amount × exchange_rate` = the **base-currency**
   equivalent; `current_amount` decreases as the deposit is settled.
4. So **Received/Original = entered currency**, **Available (current_amount) = base
   currency**.

## 3. Currency selection (base-currency aware)
1. The company **base currency** is derived from the loaded currency list — the
   `to_currency` of a configured rate record (e.g. `EUR → USD` ⇒ base **USD**),
   defaulting to `PKR`.
2. The **Currency** dropdown offers only the **base currency** itself plus currencies
   **configured to convert to the base** (`to_currency === base` with `exchange_rate > 0`,
   e.g. EUR). PKR is no longer force-added; for a USD-base company the list is
   **USD + EUR**.
3. The **default** (and post-submit reset) currency is the **base currency**.
4. Selecting the base currency sets `exchange_rate = 1`; the "… Equivalent" helper under
   Amount shows the base-currency equivalent and only appears when the rate ≠ 1.

## 3a. Bank account selection (manual banks supported)
1. The bank dropdowns list bank accounts as `bank?.name || manual_bank_name (account_number)`;
   a **manual-bank** account has `bank_id = NULL` and its typed name in `manual_bank_name`.
2. **Validation** keys on the picked account (`selectedBankAccountId`), not `bank_id` —
   a manual account never gets a `bank_id`, so checking `bank_id` wrongly showed
   "Bank account is required" even with an account selected.
3. **Edit-mode preselect**: with a saved `bank_id` the dropdown preselects by
   `bank_id` + `account_number`; a manual-bank deposit (`bank_id` null) preselects by
   `account_number` + `is_manual_bank` instead.
4. Same rules as the supplier side — see `ADVANCE_PAYMENT_MODULE.md` §3a.
5. **Open item**: unlike the supplier side, `customer_deposits` has **no `bank_name`
   snapshot column** yet, so the printed customer-deposit document still shows the bank
   name via the live `bank_id` join — blank for manual-bank deposits.

## 4. Printed document (currency-aware)
1. **Received** shows the entered amount in the **entered currency**.
2. **Amount In Words** uses the entered amount + entered currency via
   `PKRtoWords(amount, currencyCode)` — currency-aware (dollars/cents, euros,
   rupees/paisas, …; unknown currencies fall back to the code).
3. For a foreign-currency deposit, the **Exchange Rate** and **`<base>` Equivalent**
   lines are labelled with the company **base currency** (was hard-coded "PKR"); the base
   currency is derived in the controller from `currencies.to_currency`.

## 5. Related
1. Settling a customer deposit against invoices happens in the Receipt module.
2. Base-currency labelling rules across documents/listings: see
   `INVOICE_DOCUMENT_CURRENCY.md`. Supplier counterpart: `ADVANCE_PAYMENT_MODULE.md`.
