---
name: spot-ap-cashflow
description: |
  Refresh Spot AI's "Expense by Vendor" + AP workbook from NetSuite and Ramp. Rebuilds three tabs:
    (1) Expense by Vendor - a 13-week cash-out view: 6 ACTUAL weeks (transaction-level vendor payments from
      NetSuite bank disbursements, Ramp travel reimbursements aggregated into a single "Travel (Ramp)" line) +
        7 FORECAST weeks (open AP by due date, past-due in the first forecast week);
          (2) Open Bills (AP) - NetSuite open VendBills plus any Ramp bill-pay bills not yet in the NS AP balance;
            (3) AP Payment Forecast - a 13-week forecast that buckets every open bill into the week of its due date
              (past-due in Week 1). Produces a versioned .xlsx that recalculates with zero formula errors.
              trigger: |
                Use whenever the user wants to refresh / rebuild / roll forward the Expense by Vendor workbook, the AP
                  aging / open bills list, the AP payment forecast, or "this week's vendor spend + AP". Also triggers on:
                    "update the expense by vendor file", "forecast AP by due date", "pull Ramp bills into AP",
                      "13-week AP cash forecast".
                      ---

                      # Spot AI - AP Cash-Flow Workbook (Expense by Vendor + AP + 13-week Forecast)

                      ## What it produces
                      A single `.xlsx` with three tabs, refreshed live from NetSuite (acct 6931845) and Ramp:

                      1. **Expense by Vendor** - 13 weekly buckets (Mon-Sun) of *cash out* by vendor: W1-W6 ACTUAL (completed weeks, vendor payments), W7-W13 FORECAST (open AP by due date, past-due in W7, final week absorbs anything due on/after its Monday). A banner row marks the ACTUAL vs FORECAST regions.
                      2. **Open Bills (AP)** - every open NetSuite vendor bill + Ramp bill-pay bills missing from NS AP.
                      3. **AP Payment Forecast** - 13 weekly buckets from the current week, each bill in its due-date week.

                      Output name: `AP_Workbook_Refresh_<YYYY-MM-DD>.xlsx` (never overwrite a prior dated file).

                      ## Key definitions (do not change without checking against a known-good prior file)
                      - **Expense by Vendor = cash disbursements.** Sum of NetSuite `transactionaccountingline` rows that post to a **Bank** account (`account.accttype='Bank'`, `posting='T'`) from outflow transaction types **`VendPymt`, `Check`, `Journal`** only. Exclude `Transfer` (internal) and all money-in types (`Deposit`, `CustPymt`, ...). Group by `BUILTIN.DF(entity)`, net per vendor per week, keep rows whose weekly net is **negative**. `entity IS NULL` -> label **"(No entity / JE)"** (net of bank journals).
                      - **Vendor display name** = NetSuite entity name with the leading `VEN####`/`CUS####` token stripped. Spot books Ramp employee **reimbursements** to vendor records literally named `"<Employee> Ramp"`, so those flow through naturally as `<Employee> Ramp` rows.
                      - **Travel aggregation (required):** For every `"... Ramp"` reimbursement vendor, split its weekly cash-out by the vendor's **travel fraction** = (reimbursement-bill expense lines posted to Travel & Entertainment accounts) / (total expense lines), over a trailing window. Travel accounts are ids **275, 506, 508, 509, 513** (`63000 / 63100 / 63200 / 63300 / 63900`). The travel portion of every Ramp vendor is summed into **one** `"Travel (Ramp)"` line per week; the remainder stays under `<Employee> Ramp`. Non-Ramp vendors are never split.
                      - **Open AP** = NetSuite VendBills with `status='A'` (Open); unpaid balance = `foreignamountunpaid`.
                      - **Ramp-only bills** = unpaid Ramp bills whose invoice number is **not** present as a NS VendBill `tranid` (1:1 vendor+amount fallback for blank/duplicate invoice numbers). Added to AP and forecast, flagged `Ramp (not in NS AP)`.
                      - **Forecast bucketing:** week index = floor((due_date - current_week_Monday)/7); clamp <0 -> Week 1 (past due), >12 -> Week 13.

                      ## Procedure

                      ### 0. Dates
                      `asof` = today unless told otherwise. `current_week_monday = asof - asof.weekday()` (= first FORECAST week). ACTUAL window = the 6 *completed* weeks before the current week: `EXP_START = current_week_monday - 6*7`, `EXP_END = current_week_monday - 1`. FORECAST window = `current_week_monday` + the next 6 weeks (7 weeks). The forecast region is built from the AP bills (Q3 + Ramp), not a separate pull.

                      ### 1. Pull sources (always live - never reuse a prior raw/)
                      Save each result to `raw/` as JSON, then run `refresh.py`.

                      | File | Source | Tool |
                      |------|--------|------|
                      | raw/expense_grid.json | weekly bank cash-out by vendor (Q1), as [wk, vendor_raw, amt] | NetSuite ns_runCustomSuiteQL |
                      | raw/ramp_travel_frac.json | per-Ramp-vendor travel vs total expense (Q2), as [vendor_raw, travel, total] | NetSuite ns_runCustomSuiteQL |
                      | raw/ns_ap.json | open VendBills (Q3), as [tranid, vendor_raw, "M/D/YYYY", unpaid] | NetSuite ns_runCustomSuiteQL |
                      | raw/ramp_bills.json | unpaid Ramp bills, as [invoice, vendor, amount] | Ramp ramp_list_bills (include_paid=false, paginate <=50) |
                      | raw/ramp_due.json | Ramp open-bill due dates (Q4), as {invoice: "YYYY-MM-DD"} | Ramp ramp_execute_analyst_query |

                      refresh.py dedupes ramp_bills vs ns_ap to derive the Ramp-only set and joins ramp_due for due dates.

                      #### Q1 - Expense by Vendor (sub {EXP_START}=current_week_monday-42, {EXP_END}=current_week_monday-1; wk index 0..5)
                          SELECT FLOOR((TRUNC(t.trandate)-TRUNC(TO_DATE('{EXP_START}','YYYY-MM-DD')))/7) AS wk,
                                     BUILTIN.DF(t.entity) AS vendor, SUM(tal.amount) AS amt
                                         FROM transaction t
                                             JOIN transactionaccountingline tal ON tal.transaction=t.id
                                                 JOIN account a ON a.id=tal.account
                                                     WHERE a.accttype='Bank' AND tal.posting='T' AND t.type IN ('VendPymt','Check','Journal')
                                                           AND t.trandate BETWEEN TO_DATE('{EXP_START}','YYYY-MM-DD') AND TO_DATE('{EXP_END}','YYYY-MM-DD')
                                                               GROUP BY FLOOR((TRUNC(t.trandate)-TRUNC(TO_DATE('{EXP_START}','YYYY-MM-DD')))/7), BUILTIN.DF(t.entity)
                                                                   HAVING SUM(tal.amount) < 0 ORDER BY wk, amt;

                                                                   #### Q2 - Ramp vendor travel fraction (trailing ~10 wks; {EXP_START_M2} ~ EXP_START - 5 weeks)
                                                                       SELECT BUILTIN.DF(t.entity) AS vendor,
                                                                                  SUM(CASE WHEN a.id IN (275,506,508,509,513) THEN tal.amount ELSE 0 END) AS travel,
                                                                                             SUM(tal.amount) AS total
                                                                                                 FROM transaction t
                                                                                                     JOIN transactionaccountingline tal ON tal.transaction=t.id
                                                                                                         JOIN account a ON a.id=tal.account
                                                                                                             WHERE t.type='VendBill' AND BUILTIN.DF(t.entity) LIKE '%Ramp%'
                                                                                                                   AND a.accttype IN ('Expense','OthExpense','COGS')
                                                                                                                         AND t.trandate BETWEEN TO_DATE('{EXP_START_M2}','YYYY-MM-DD') AND TO_DATE('{EXP_END}','YYYY-MM-DD')
                                                                                                                             GROUP BY BUILTIN.DF(t.entity);
                                                                                                                             
                                                                                                                             #### Q3 - Open AP
                                                                                                                                 SELECT t.tranid, BUILTIN.DF(t.entity) AS vendor, t.duedate, t.foreignamountunpaid
                                                                                                                                     FROM transaction t WHERE t.type='VendBill' AND t.status='A' ORDER BY t.duedate;
                                                                                                                                     
                                                                                                                                     #### Q4 - Ramp open-bill due dates (Ramp analyst SQL)
                                                                                                                                         SELECT invoice_number, bill_due_date FROM analyst.ap_bill_facts
                                                                                                                                             WHERE is_deleted=false AND is_fully_paid=false AND open_amount>0;
                                                                                                                                             
                                                                                                                                             ### 2. Build
                                                                                                                                                 python refresh.py --asof <YYYY-MM-DD>   # defaults to today; reads raw/, writes versioned .xlsx
                                                                                                                                                     python /path/to/skills/xlsx/scripts/recalc.py AP_Workbook_Refresh_<date>.xlsx 60
                                                                                                                                                     
                                                                                                                                                     Recalc MUST report total_errors: 0. Confirm the printed reconciliation:
                                                                                                                                                     Forecast grand total == NS AP total + Ramp-only total.
                                                                                                                                                     
                                                                                                                                                     ## Gotchas
                                                                                                                                                     - ramp_list_bills caps limit at 50 - paginate with next_page_cursor.
                                                                                                                                                     - ramp_list_bills does NOT return due dates; get them from Q4 (analyst.ap_bill_facts).
                                                                                                                                                     - Blank/duplicate Ramp invoice numbers: dedupe 1:1 against NS tranid, then vendor+amount fallback, so a duplicate the ERP only booked once is still flagged Ramp-only.
                                                                                                                                                     - The "(No entity / JE)" bucket is journals only; it differs slightly from a naive all-types bank net because deposits/customer receipts are intentionally excluded.
                                                                                                                                                     
