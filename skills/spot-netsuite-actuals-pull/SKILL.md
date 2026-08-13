---
name: "spot-netsuite-actuals-pull"
description: "Pull Spot AI's live cash ACTUALS from NetSuite via SuiteQL (account 7735559) for the Weekly Cash Flow run — Step 1. Returns (a) current bank balances for the operating bank pool, (b) customer collections for the last 4 weeks, and (c) cash outflows by GL category for the last 4 weeks, plus total cash, week-over-week change, and the largest single movement. Use at the start of every weekly cash refresh, or whenever asked to 'pull the NetSuite cash actuals', 'get bank balances', 'pull collections/outflows from NetSuite', or 'refresh the actuals'. Never fabricates — if NetSuite is unreachable it says so."
---

# Spot AI — NetSuite cash ACTUALS pull (Weekly Cash Flow, Step 1)

Pulls the live cash actuals that seed the Weekly Cash Flow model from NetSuite via SuiteQL. This is **Step 1** of the weekly refresh (the orchestrator is `spot-weekly-cashflow-run`). **Never fabricate numbers** — if a query fails or a source is unreachable, report the failure and continue; do not estimate.

Connector: the NetSuite MCP (SuiteQL tool `ns_runCustomSuiteQL`), NetSuite **account/instance 7735559**. All amounts are USD.

## The operating bank pool
Spot AI's cash lives in a fixed pool of bank GL accounts. The canonical pool (NetSuite internal account ids) is **662, 665, 688, 669, 687** plus Undeposited Funds **122**. Bank balances and cash-flow categorization are scoped to this pool. (Confirm the pool each run with query (a); if a new bank account appears with a non-zero balance, add it and note it.)

## a) Bank balances (given)
```sql
SELECT id, acctnumber, accountsearchdisplayname AS account_name, balance
FROM Account
WHERE accttype='Bank' AND isinactive='F' AND balance != 0
ORDER BY balance DESC NULLS LAST
```
Capture each account's balance and the **total cash**. Typical named accounts: SVB Sweep, JPM, SVB Checking.

## b) Collections — last 4 weeks
Collections basis = **Saved Search SS1722** logic = customer cash in = `CustPymt` (customer payment) + `CustRfnd` (refund, negative) **including Undeposited Funds**, on the bank pool, grouped by week. Reconstruct in SuiteQL against `transaction` + `transactionline` filtered to the bank pool accounts and `type IN ('CustPymt','CustRfnd')`, `trandate >= (today - 28 days)`, summing `amount` by ISO week (Fri–Thu weeks, anchor W1 = 2026-05-01). Return per-week collection totals and the payment count. This is the same basis the model's `Collections` category ties to.

## c) Cash outflows by category — last 4 weeks
Sum posted disbursements on the bank pool for the last 28 days, grouped by GL category (map GL account → the model's cash-flow categories per the workbook's `Methodology & GL Map` tab: AP / Vendor Pmts, Payroll & Benefits, Ramp Cards, Debt Service, Direct Opex & Other, CapEx). Return per-category weekly totals. Vendor detail comes from the vendor on each disbursement (for the Expense-by-Vendor rollup).

> The exact collections/cashOut SQL is the "collections query" and "cashOut query" embodied in the model's Netlify refresh function. If you have that function's SQL, use it verbatim; otherwise the reconstructions above reproduce the same basis. Always tie the pulled totals back to the bank-pool balance change for the period.

## Output
Report: (1) each bank account + **total cash**; (2) **week-over-week change** vs last Monday ($ and %); (3) the **largest single movement** (account/day); (4) the 4-week collections series (with payment count) and the 4-week cash-outflow-by-category series. Hand these to `spot-weekly-cashflow` (build) and `spot-cashflow-reconcile-flags` (Step 4). If any query errored, list it under a "Failures" note so downstream steps know the gap.

## Notes / gotchas
- SuiteQL does not support `MEDIAN`/`EXTRACT` — use `TO_CHAR(trandate,'DD')` etc.
- Large result sets may spill to a tool-results `.txt`; parse via bash/Python.
- Undeposited Funds (122) must be included in collections or the week under-reports.
- Keep everything on the bank pool — do not include non-cash GL movement.
