---
name: "spot-collections-refresh-suiteql"
description: "Refresh Spot AI's unbilled + AR collections forecast (SuiteQL-ONLY — no dependency on the flaky saved searches customsearch2369 / customsearch_spot_finance_invoices) AND wire it into the Weekly Cash Flow model. MANDATORY EVERY WEEKLY REFRESH — the unbilled forecast goes stale within ~5 weeks. Reconstructs unbilled from billing-schedule recurrence + terms (Due-on-receipt = same-day) and open AR from invoices, buckets to collection week (anchor 2026-05-01 = W1), then refreshes Collections-by-Customer rows 13 (AR Fcst) / 14 (Unbilled Fcst) via direct SUMIFS. Also produces the standalone 3-tab Weekly Collections workbook. Use whenever refreshing the weekly cash forecast, the collections forecast, or when the unbilled/AR forecast looks stale."
---

# Spot AI Collections Refresh — SuiteQL-only (unbilled + AR) → wire into the cash-flow model

**MANDATORY EVERY WEEKLY CASH-FORECAST REFRESH.** The unbilled/AR forecast goes stale fast (it typically runs out ~5 weeks past the as-of and misses new deals), so re-pull and re-wire it **every week**. No saved searches — reconstructs everything from SuiteQL, so it's immune to the `customsearch2369` / `customsearch_spot_finance_invoices` failures.

Two outputs each run:
1. The **standalone** 3-tab workbook (Summary / Unbilled Invoices / AR Collection) — the reviewable collections model.
2. 2. The **wire-in** to the live Weekly Cash Flow model (`Collections by Customer` rows 13/14) — so forecast collections in the cash model are current.
  
   3. ## Key definitions
   4. - **Anchor:** W1 = **2026-05-01** (Fri), same as the cash-flow model.
      - - **Actual vs forecast split** = the **as-of date**. Weeks starting ≤ as-of are ACTUAL; later weeks FORECAST.
        - - **Unbilled forecast** = future billing-schedule milestone's **send date + payment-term days** = collection week. **Term id 4 = "Due on receipt" = 0 days → collects the same day it bills** (e.g. the 10/10 Due-on-receipt billings — S&D CarWash + schedules 129/130 at ~$107.7K each — land in W24, not W28).
          - - **AR forecast** = open invoice on its **due date**, forecast weeks only. Terms-only: do NOT project past-due AR.
            - - **Term id → net days** (verify with `SELECT id,name,daysuntilnetdue FROM term`): `{1:15, 2:30, 3:60, 4:0, 5:30, 6:30, 7:90, 8:5, 9:120, 10:180, 11:60, 12:45}`.
             
              - ## Step 1 — Pull 4 SuiteQL sources → `raw/` (add `RPAD('x',N,'x') AS _pad` to force the spill-to-file; parse `"data":[...]`, drop `_pad`)
              - `raw/actuals.json` `{customer,cname,d,amt}`:
              - ```sql
                SELECT t.entity AS customer, BUILTIN.DF(t.entity) AS cname, TO_CHAR(t.trandate,'YYYY-MM-DD') AS d, SUM(t.foreigntotal) AS amt
                FROM transaction t WHERE t.type='CustPymt' AND t.trandate BETWEEN TO_DATE('2026-05-01','YYYY-MM-DD') AND TO_DATE('{ASOF}','YYYY-MM-DD')
                GROUP BY t.entity, BUILTIN.DF(t.entity), TO_CHAR(t.trandate,'YYYY-MM-DD')
                ```
                `raw/recurrence.json` `{billingschedule,recurrenceid,recurrencedate,amount,paymentterms}` (the latest billings / new deals):
                ```sql
                SELECT billingschedule, recurrenceid, TO_CHAR(recurrencedate,'YYYY-MM-DD') AS recurrencedate, amount, paymentterms
                FROM billingschedulerecurrence WHERE recurrencedate IS NOT NULL ORDER BY billingschedule, recurrenceid
                ```
                `raw/bs2cust.json` `{bs,customer}`:
                ```sql
                SELECT DISTINCT tl.billingschedule AS bs, BUILTIN.DF(t.entity) AS customer
                FROM transactionline tl JOIN transaction t ON t.id=tl.transaction WHERE tl.billingschedule IS NOT NULL AND t.type='SalesOrd'
                ```
                `raw/open_ar.json` `{customer,duedate,open_amt}`:
                ```sql
                SELECT BUILTIN.DF(t.entity) AS customer, TO_CHAR(t.duedate,'YYYY-MM-DD') AS duedate, SUM(t.foreignamountunpaid) AS open_amt
                FROM transaction t WHERE t.type='CustInvc' AND t.foreignamountunpaid > 0 GROUP BY BUILTIN.DF(t.entity), TO_CHAR(t.duedate,'YYYY-MM-DD')
                ```

                ## Step 2 — Build the standalone workbook
                ```
                python3 collections_refresh_suiteql.py --raw ./raw --asof <YYYY-MM-DD> --anchor 2026-05-01 --weeks <cash-flow last week #> --out Weekly_Collections_Model_<asof>.xlsx
                ```
                (Script is in the `Weekly Cash Forecast` project folder; full source embedded at the bottom.)

                ## Step 3 — WIRE INTO THE WEEKLY CASH FLOW MODEL (do this every weekly refresh)
                Compute per-`(collection-week-start, customer)` **Unbilled (Fcst)** and **AR Collection (Fcst)** for the model's FORECAST weeks (weeks after the current partial week), then edit the model workbook:
                1. **Clear** the stale `Unbilled (Fcst)` / `AR Collection (Fcst)` Transaction-Detail rows for forecast weeks (tag ≥ first forecast week-start): set col F = 0.
                2. **Write fresh** per-customer TD rows: A/B = collection-week start string, C = "Collections forecast (refreshed <asof>)", D = customer, E = "Unbilled (Fcst)" or "AR Collection (Fcst)", F = amount. Keep the last data row ≤ 1700 (the SUMIFS range); if it would exceed, widen the ranges.
                3. **Set `Collections by Customer` rows 13 (AR Fcst) & 14 (Unbilled Fcst) for each forecast column to a DIRECT SUMIFS** over Transaction Detail (NOT the per-customer detail-block `={c}280`, which silently drops customers not already listed in the block):
                   - `{col}13 = SUMIFS('Transaction Detail'!$F$5:$F$1700, $A$5:$A$1700, "<week-start>", $E$5:$E$1700, "AR Collection (Fcst)")`
                      - `{col}14 = SUMIFS(... "<week-start>" ... "Unbilled (Fcst)")`
                         Column map (WCF/CC col = week# + 1): W16=Q, W17=R, W18=S, W19=T, W20=U, W21=V, W22=W, W23=X, W24=Y, W25=Z …
                         4. Recalc (LibreOffice, 0 new errors) and run the `spot-cashflow-output-check` gate.

                         ## ⚠️ Double-count watch (tops-down vs bottom-up)
                         The model also has **tops-down milestone rows** — `Collections by Customer` row 11 (New Business / Land+Expand) and row 12 (Renewals Won), landing at W18 & W23 via `'Tops down cash forecast'`. These are **summed with** the bottom-up rows 13/14 into total collections (row 15). After refreshing, **check W18/W23 for double-count** (on 2026-08-10 W18 total hit $2.12M = tops-down $1.24M + bottom-up AR $878K). Confirm with the user whether tops-down or bottom-up owns those weeks; if bottom-up supersedes, zero rows 11/12 for those weeks.

                         ## Validation
                         - Summary `Total` per week = Actual + Unbilled + AR; actual-weeks total ties `sum(actuals amt)`.
                         - Big Due-on-receipt billings land the week they bill (0-day term).
                         - Benchmark 2026-08-10 (as-of): forecast by collection week — W16 $177,831 (AR) · W17 $122,181 · **W18 $878,414** · W19 $139 · W20 $73,584 · W21 $17,485 · W22 $13,141 · W23 $5,393 · **W24 $693,330** (incl. $648K 10/10 Due-on-receipt unbilled) · W25 $6,426.

                         ## Full build script (recreate `collections_refresh_suiteql.py` in the project folder if missing)
                         ```python
                         #!/usr/bin/env python3
                         import argparse, json, os
                         from datetime import datetime, timedelta
                         TERM_DAYS={1:15,2:30,3:60,4:0,5:30,6:30,7:90,8:5,9:120,10:180,11:60,12:45,None:0}
                         TERM_NAME={1:'Net 15',2:'Net 30',3:'Net 60',4:'Due on rcpt',5:'Net 30',6:'Net 30',7:'Net 90',9:'Net 120',10:'Net 180',11:'Net 60',12:'Net 45',None:'—'}
                         def dparse(s): return datetime.strptime(s[:10],'%Y-%m-%d')
                         def load(raw,name): return json.load(open(os.path.join(raw,name)))
                         def main():
                             ap=argparse.ArgumentParser()
                                 ap.add_argument('--raw',required=True); ap.add_argument('--asof',default=datetime.today().strftime('%Y-%m-%d'))
                                     ap.add_argument('--anchor',default='2026-05-01'); ap.add_argument('--weeks',type=int,default=0); ap.add_argument('--out',default=None)
                                         a=ap.parse_args()
                                             import openpyxl
                                                 from openpyxl.styles import Font, PatternFill
                                                     anchor=dparse(a.anchor); asof=dparse(a.asof)
                                                         N=max(a.weeks, ((asof+timedelta(weeks=13)-anchor).days//7)+1, 13)
                                                             weeks=[anchor+timedelta(days=7*i) for i in range(N)]
                                                                 def wkidx(d):
                                                                         if d<anchor: return None
                                                                                 i=(d-anchor).days//7
                                                                                         return i if i<N else None
                                                                                             act=load(a.raw,'actuals.json'); rec=load(a.raw,'recurrence.json'); bsc=load(a.raw,'bs2cust.json'); ar=load(a.raw,'open_ar.json')
                                                                                                 bs2c={r['bs']:r['customer'] for r in bsc}
                                                                                                     A=[0.0]*N; U=[0.0]*N; R=[0.0]*N; ubd=[]; ard=[]
                                                                                                         for r in act:
                                                                                                                 i=wkidx(dparse(r['d']))
                                                                                                                         if i is not None and weeks[i]<=asof: A[i]+=r['amt']
                                                                                                                             for r in rec:
                                                                                                                                     d=dparse(r['recurrencedate'])
                                                                                                                                             if d<=asof: continue
                                                                                                                                                     cd=d+timedelta(days=TERM_DAYS.get(r.get('paymentterms'),0)); i=wkidx(cd)
                                                                                                                                                             if i is not None and weeks[i]>asof:
                                                                                                                                                                         U[i]+=r['amount']; ubd.append([bs2c.get(r['billingschedule'],f"BS{r['billingschedule']}"),r['recurrencedate'],TERM_NAME.get(r.get('paymentterms')),cd.strftime('%Y-%m-%d'),f"W{i+1}",round(r['amount'],2)])
                                                                                                                                                                             for r in ar:
                                                                                                                                                                                     dd=r.get('duedate')
                                                                                                                                                                                             if not dd: continue
                                                                                                                                                                                                     d=dparse(dd)
                                                                                                                                                                                                             if d<=asof: continue
                                                                                                                                                                                                                     i=wkidx(d)
                                                                                                                                                                                                                             if i is not None and weeks[i]>asof:
                                                                                                                                                                                                                                         R[i]+=r['open_amt']; ard.append([r['customer'],dd,f"W{i+1}",round(r['open_amt'],2)])
                                                                                                                                                                                                                                             wb=openpyxl.Workbook()
                                                                                                                                                                                                                                                 hf=Font(bold=True,color='FFFFFF'); hfl=PatternFill('solid',fgColor='2F5496'); grn=PatternFill('solid',fgColor='C6EFCE'); blu=PatternFill('solid',fgColor='DDEBF7')
                                                                                                                                                                                                                                                     ws=wb.active; ws.title="Summary"
                                                                                                                                                                                                                                                         ws.append([f"Spot AI — Weekly Cash Collections (refreshed {a.asof}: actuals + latest unbilled/AR, SuiteQL path)"]); ws["A1"].font=Font(bold=True,size=13)
                                                                                                                                                                                                                                                             ws.append([f"Anchor W1={a.anchor} (=Weekly Cash Flow W1). Actual=live CustPymt; Forecast=unbilled billing-schedules (send+terms) + open AR (due date)."]); ws.append([])
                                                                                                                                                                                                                                                                 ws.append(["Week","Start","A/F","Actual","Unbilled (Fcst)","AR Collection (Fcst)","Total"])
                                                                                                                                                                                                                                                                     for c in range(1,8): ws.cell(4,c).font=hf; ws.cell(4,c).fill=hfl
                                                                                                                                                                                                                                                                         for i in range(N):
                                                                                                                                                                                                                                                                                 af='ACTUAL' if weeks[i]<=asof else 'FORECAST'
                                                                                                                                                                                                                                                                                         ws.append([f"W{i+1}",weeks[i].strftime('%Y-%m-%d'),af,round(A[i],2),round(U[i],2),round(R[i],2),round(A[i]+U[i]+R[i],2)])
                                                                                                                                                                                                                                                                                                 ws.cell(ws.max_row,3).fill=grn if af=='ACTUAL' else blu
                                                                                                                                                                                                                                                                                                     ws.append(["TOTAL","","",round(sum(A),2),round(sum(U),2),round(sum(R),2),round(sum(A)+sum(U)+sum(R),2)]); ws.cell(ws.max_row,1).font=Font(bold=True)
                                                                                                                                                                                                                                                                                                         for col,w in {'A':6,'B':12,'C':10,'D':13,'E':15,'F':20,'G':13}.items(): ws.column_dimensions[col].width=w
                                                                                                                                                                                                                                                                                                             wu=wb.create_sheet("Unbilled Invoices (Fcst)"); wu.append(["Customer","Billing date","Terms","Collection date","Coll. week","Amount"])
                                                                                                                                                                                                                                                                                                                 for c in range(1,7): wu.cell(1,c).font=hf; wu.cell(1,c).fill=hfl
                                                                                                                                                                                                                                                                                                                     for row in sorted(ubd,key=lambda x:(x[3],-x[5])): wu.append(row)
                                                                                                                                                                                                                                                                                                                         for col,w in {'A':40,'B':13,'C':12,'D':14,'E':10,'F':13}.items(): wu.column_dimensions[col].width=w
                                                                                                                                                                                                                                                                                                                             wa=wb.create_sheet("AR Collection (Fcst)"); wa.append(["Customer","Due date","Coll. week","Open amount"])
                                                                                                                                                                                                                                                                                                                                 for c in range(1,5): wa.cell(1,c).font=hf; wa.cell(1,c).fill=hfl
                                                                                                                                                                                                                                                                                                                                     for row in sorted(ard,key=lambda x:(x[1],-x[3])): wa.append(row)
                                                                                                                                                                                                                                                                                                                                         for col,w in {'A':40,'B':12,'C':10,'D':13}.items(): wa.column_dimensions[col].width=w
                                                                                                                                                                                                                                                                                                                                             out=a.out or f"Weekly_Collections_Model_{a.asof}.xlsx"; wb.save(out)
                                                                                                                                                                                                                                                                                                                                                 print(f"saved {out}"); print(f"Actual thru {a.asof}: {sum(A):,.0f} | Forecast unbilled: {sum(U):,.0f} | Forecast AR: {sum(R):,.0f}")
                                                                                                                                                                                                                                                                                                                                                     for i in range(N):
                                                                                                                                                                                                                                                                                                                                                             if weeks[i]>asof and (U[i] or R[i]): print(f"  W{i+1} {weeks[i].strftime('%Y-%m-%d')}: unbilled {U[i]:,.0f} + AR {R[i]:,.0f} = {U[i]+R[i]:,.0f}")
                                                                                                                                                                                                                                                                                                                                                             if __name__=='__main__': main()
                                                                                                                                                                                                                                                                                                                                                             ```
                                                                                                                                                                                                                                                                                                                                                             
