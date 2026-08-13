---
name: "spot-unbilled-forecast"
description: "Dynamically rebuild Spot AI's UNBILLED collections forecast from ALL current revenue / billing schedules — projecting every open schedule's remaining installments by its cadence (monthly/quarterly/annual/etc.) and timing them by payment terms into weekly buckets. Use every weekly cash-forecast refresh, or whenever the unbilled forecast looks blank/stale/undercounted, or the user asks to 'refresh the unbilled forecast', 'pull in the latest revenue schedules', 'the unbilled needs updating', or 'build unbilled by week'. Pulls the customsearch2369 Forecasted-Billings backlog + the billingschedule master + a customer map, reconstructs future billings, and wires per-customer unbilled into the Weekly Cash Flow model's Unbilled (Fcst) line + detail block."
---

# Spot AI — dynamic UNBILLED forecast (from all revenue/billing schedules)

Rebuilds the **Unbilled (Fcst)** line of the Weekly Cash Flow model from the **full open billing-schedule backlog**, every weekly refresh. Run it alongside the AR/actuals refresh (`spot-collections-refresh-suiteql`) and the entity-detail step (`spot-cashflow-forecast-entity-detail`).

## Why this exists (the failure it fixes)
Unbilled = contracted revenue **not yet invoiced**. If you pull unbilled only from **explicitly dated** milestones (`billingschedulerecurrence` rows with a future date), you miss almost everything: most subscriptions bill on a **cadence** with no materialized future row, so the unbilled line comes out near-blank. The real backlog is large — on 2026-08-10 it was **$24.9M remaining unbilled across 693 open SOs / 932 active schedules**. This skill reconstructs every schedule's future billings from its cadence.

## Sources (pull live each run; big results spill to a tool-results file → parse `"data"`)
1. **customsearch2369** "Forecasted Billings (Unbilled)" — `ns_runSavedSearch searchId=customsearch2369` (paginate range 0–1000). Fields per open SO: `Customer` (id), `Document Number`, `Payment Terms`, `Billing Schedule` (id), `Amount` (= **remaining unbilled $**), `Start Contract Date`, `End Contract Date`. Save as `raw/cs2369.json`. *(This search failed intermittently in the past; if it errors, retry — it recovers.)*
2. **Billing schedule master** (cadence) — save as `raw/bs_master.json`:
   ```sql
      SELECT id, frequency, repeatevery, numberremaining, initialamount, recurrenceterms, initialterms
         FROM billingschedule WHERE numberremaining > 0
            ```
               (add `RPAD('x',600,'x') AS _pad` to force the spill; drop `_pad` when saving. `frequency` ∈ MONTHLY/QUARTERLY/ANNUALLY/SEMIANNUALLY/DAILY/CUSTOM.)
               3. **Customer id→name** — save as `raw/customers.json`: `SELECT id, BUILTIN.DF(id) AS nm FROM customer` (spill via `_pad`).

               ## Reconstruction (what the script does)
               For each open SO/schedule: per-installment = `Amount / numberremaining`; project future billing dates anchored on **Start Contract Date** stepping by the cadence period (MONTHLY 30d · QUARTERLY 91d · ANNUALLY 365d · SEMIANNUALLY 182d · DAILY/CUSTOM 30d), first billing after the as-of date, up to `numberremaining` installments, not past End Contract Date. Collection date = billing date + **payment-term days** (`recurrenceterms`, else SO Payment Terms; **term id 4 = Due-on-receipt = 0**). Bucket the collection into its week (anchor W1 = 2026-05-01); keep only the model's forecast weeks. Customer name = `BUILTIN.DF` **stripped of the `CUS####` prefix** (unmapped schedules → `"Billing schedule <id> (unmapped)"`).

               ```
               python3 unbilled_forecast_recon.py --raw ./raw --asof <YYYY-MM-DD> \
                   --anchor 2026-05-01 --first-fcst-week <first forecast week #> --last-week <model last week #> \
                       --out unbilled_recon.json
                       ```
                       Script is in the `Weekly Cash Forecast` project folder (full source embedded at bottom). It prints the weekly unbilled totals and writes `unbilled_recon.json` keyed `"<week-start>|<customer>" -> amount`.

                       ## Wire into the Weekly Cash Flow model
                       1. **Clear** the old `Unbilled (Fcst)` Transaction-Detail rows for forecast weeks (`col A ≥ first forecast week-start`).
                       2. **Write fresh** per-`(week, stripped-customer)` `Unbilled (Fcst)` TD rows (A/B = week-start, C = note, D = customer, E = "Unbilled (Fcst)", F = amount).
                       3. **Watch the SUMIFS range:** the model's formulas sum `Transaction Detail $5:$1700`. This forecast adds many rows (2026-08-10 pushed TD to ~1900). If the last data row exceeds 1700, **widen every `$1700` → `$2000`** (workbook-wide string replace in formula cells) or rows past 1700 are silently dropped.
                       4. **Populate per-customer detail** and keep AR/Unbilled separate — hand off to `spot-cashflow-forecast-entity-detail`: add any new unbilled customers to the Collections-by-Customer detail block (move the Total row, fix refs), fill the per-customer detail cells, and keep summary **row 13 "AR Collection (Fcst)"** and **row 14 "Unbilled (Fcst)"** as separate visible SUMIFS lines (detail Total = row13 + row14 tie-check).
                       5. Recalc (LibreOffice, **0 new** errors — only pre-existing `Payroll Tax Trend!A21`); run `spot-cashflow-output-check`.

                       ## Validate
                       - Unbilled line is populated across (not blank in) the forecast weeks; total in-horizon ≈ the script's printed total.
                       - Detail Total == AR + Unbilled every forecast week.
                       - ⚠️ **Double-count watch:** the model also sums tops-down milestone rows (Collections by Customer 11 New Business / 12 Renewals, W18/W23) — confirm with the user whether tops-down or this bottom-up unbilled/AR owns those weeks before trusting the total.

                       ## Benchmark (2026-08-10, as-of; forecast weeks W16–W25)
                       In-horizon unbilled ≈ **$2.68M**: W16 $38,671 · W17 $22,558 · W18 $13,619 · W19 $305,609 · W20 $202,936 · W21 $79,261 · W22 $179,890 · W23 $446,551 · **W24 $1,269,886** · W25 $125,607. (841 projected installment-collections; $24.9M total remaining backlog.)

                       ## Full script (recreate `unbilled_forecast_recon.py` if missing)
                       ```python
                       #!/usr/bin/env python3
                       import argparse, json, os, re
                       from datetime import datetime, timedelta
                       from collections import defaultdict
                       TERM_DAYS={1:15,2:30,3:60,4:0,5:30,6:30,7:90,8:5,9:120,10:180,11:60,12:45,None:30,'':30}
                       PERIOD_DAYS={'MONTHLY':30,'QUARTERLY':91,'ANNUALLY':365,'SEMIANNUALLY':182,'DAILY':30,'CUSTOM':30}
                       def strip(n):
                           if not n: return n
                               m=re.match(r'^CUS\d+\s+(.*)$',n); return m.group(1) if m else n
                               def pdate(s):
                                   if not s: return None
                                       for f in ('%m/%d/%Y','%Y-%m-%d'):
                                               try: return datetime.strptime(s,f)
                                                       except: pass
                                                           return None
                                                           def main():
                                                               ap=argparse.ArgumentParser()
                                                                   ap.add_argument('--raw',required=True); ap.add_argument('--asof',default=datetime.today().strftime('%Y-%m-%d'))
                                                                       ap.add_argument('--anchor',default='2026-05-01'); ap.add_argument('--first-fcst-week',type=int,default=16)
                                                                           ap.add_argument('--last-week',type=int,default=25); ap.add_argument('--out',default='unbilled_recon.json')
                                                                               a=ap.parse_args()
                                                                                   cs=json.load(open(os.path.join(a.raw,'cs2369.json')))
                                                                                       mas={str(r['id']):r for r in json.load(open(os.path.join(a.raw,'bs_master.json')))}
                                                                                           id2name={str(r['id']):r.get('nm') for r in json.load(open(os.path.join(a.raw,'customers.json')))}
                                                                                               anchor=datetime.strptime(a.anchor,'%Y-%m-%d'); asof=datetime.strptime(a.asof,'%Y-%m-%d')
                                                                                                   fcweeks={}
                                                                                                       for i in range(a.first_fcst_week-1,a.last_week):
                                                                                                               ws=anchor+timedelta(days=7*i); fcweeks[ws.strftime('%Y-%m-%d')]=f"W{i+1}"
                                                                                                                   hz_end=anchor+timedelta(days=7*a.last_week-1)
                                                                                                                       def wkstart(d):
                                                                                                                               i=(d-anchor).days//7; return anchor+timedelta(days=7*i)
                                                                                                                                   unb=defaultdict(float); n=0
                                                                                                                                       for r in cs:
                                                                                                                                               amt=float(r.get('Amount') or 0)
                                                                                                                                                       if amt<=0: continue
                                                                                                                                                               bs=r.get('Billing Schedule'); m=mas.get(bs) if bs else None
                                                                                                                                                                       freq=(m or {}).get('frequency','MONTHLY'); nrem=int((m or {}).get('numberremaining') or 0)
                                                                                                                                                                               terms=(m or {}).get('recurrenceterms') or r.get('Payment Terms') or 2
                                                                                                                                                                                       try: terms=int(terms)
                                                                                                                                                                                               except: terms=2
                                                                                                                                                                                                       tdays=TERM_DAYS.get(terms,30); per=PERIOD_DAYS.get(freq,30)
                                                                                                                                                                                                               start=pdate(r.get('Start Contract Date')) or asof; end=pdate(r.get('End Contract Date')) or (asof+timedelta(days=365))
                                                                                                                                                                                                                       if nrem<=0: nrem=max(1,int((end-asof).days/30)); per=30
                                                                                                                                                                                                                               inst=amt/nrem
                                                                                                                                                                                                                                       cust=strip(id2name.get(str(r.get('Customer')),'')) or (bs and f"Billing schedule {bs} (unmapped)") or 'Unknown'
                                                                                                                                                                                                                                               j=int(((asof-start).days)//per)+1 if start<=asof else 0
                                                                                                                                                                                                                                                       cnt=0
                                                                                                                                                                                                                                                               while cnt<nrem:
                                                                                                                                                                                                                                                                           bd=start+timedelta(days=per*j)
                                                                                                                                                                                                                                                                                       if bd>end+timedelta(days=3): break
                                                                                                                                                                                                                                                                                                   if bd>asof:
                                                                                                                                                                                                                                                                                                                   cnt+=1; coll=bd+timedelta(days=tdays)
                                                                                                                                                                                                                                                                                                                                   if coll<=hz_end:
                                                                                                                                                                                                                                                                                                                                                       ws=wkstart(coll).strftime('%Y-%m-%d')
                                                                                                                                                                                                                                                                                                                                                                           if ws in fcweeks: unb[(ws,cust)]+=inst; n+=1
                                                                                                                                                                                                                                                                                                                                                                                       j+=1
                                                                                                                                                                                                                                                                                                                                                                                                   if bd>hz_end+timedelta(days=400): break
                                                                                                                                                                                                                                                                                                                                                                                                       wk=defaultdict(float)
                                                                                                                                                                                                                                                                                                                                                                                                           for (w,c),v in unb.items(): wk[w]+=v
                                                                                                                                                                                                                                                                                                                                                                                                               print(f"projected installment-collections in horizon: {n}")
                                                                                                                                                                                                                                                                                                                                                                                                                   for w in sorted(fcweeks): print(f"  {fcweeks[w]} {w}: ${wk.get(w,0):,.0f}")
                                                                                                                                                                                                                                                                                                                                                                                                                       print(f"TOTAL in-horizon unbilled: ${sum(wk.values()):,.0f}")
                                                                                                                                                                                                                                                                                                                                                                                                                           json.dump({f"{k[0]}|{k[1]}":round(v,2) for k,v in unb.items()},open(a.out,'w')); print(f"wrote {a.out}")
                                                                                                                                                                                                                                                                                                                                                                                                                           if __name__=='__main__': main()
                                                                                                                                                                                                                                                                                                                                                                                                                           ```
                                                                                                                                                                                                                                                                                                                                                                                                                           
