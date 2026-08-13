---
name: "spot-cashflow-forecast-entity-detail"
description: "Populate the CUSTOMER-level (Collections by Customer) and VENDOR-level (Expense by Vendor) detail blocks with per-entity AR/unbilled and AP forecast in the FORECAST weeks of Spot AI's Weekly Cash Flow model — not just the summary totals. Use every weekly refresh, or whenever the collections-by-customer or expense-by-vendor detail looks empty/stale in forecast periods, or the user asks to 'populate collections by customer / expense by vendor with the forecast', 'show per-customer/per-vendor forecast', or 'include entity-level detail in the forecasted periods'. Handles CUS####/VEN#### prefix stripping, name-matching to the existing detail lists, adding missing entities (expanding the list + moving the Total row + fixing references), per-entity SUMIFS, and keeping AR and Unbilled as separate visible summary lines while the detail block shows the per-customer combination."
---

# Spot AI Weekly Cash Flow — populate CUSTOMER + VENDOR forecast detail

Every weekly refresh, the forecast weeks must carry **per-entity detail**, not just summary totals:
- **Collections by Customer** — each forecast week shows each customer's **AR + Unbilled** forecast in the detail block (rows 19…N, `Total` row below), while the summary keeps **AR (row 13)** and **Unbilled (row 14)** as **separate, visible lines**.
- **Expense by Vendor** — each forecast week shows each vendor's **AP** forecast (detail block rows 21…M, `Total` row below), feeding the AP forecast line.

Do this after the actuals flip / append, before recalc+QA. Data-driven cascade: **Transaction Detail (per entity) → detail block → summary**.

## The trap that makes it look empty (ALWAYS handle)
NetSuite pulls return names **with an entity prefix** — `CUS#### <Customer>`, `VEN#### <Vendor>` (via `BUILTIN.DF(entity)`); some billing schedules have **no customer** (`BS###`). The model's detail blocks store names **without** the prefix. If you write Transaction-Detail rows with prefixed names, the per-entity `SUMIFS` (matching on the un-prefixed detail-block name) return **0** and the detail looks empty even though the summary total is right. So:
- **Strip** `^CUS\d+\s+` / `^VEN\d+\s+` from every name before writing TD rows and before matching.
- **Label** unmapped schedules `BS<id>` → `"Billing schedule <id> (unmapped)"` (real scheduled billings with a NetSuite data gap; include them).
- The Transaction-Detail **col D (entity)** you write MUST equal the detail-block **col A (name)** exactly, or the SUMIFS miss.

## ⚠️ Keep AR and Unbilled as SEPARATE summary lines (don't merge)
The detail cell **combines** AR + Unbilled per customer (see formula below). Do NOT set summary row 14 to the detail Total and zero row 13 — that blanks the AR line and mislabels Unbilled as AR+Unbilled. Instead keep BOTH summary lines as **direct SUMIFS by category** so each reads correctly:
- **row 13 "AR Collection (Fcst)" {col} = `SUMIFS('Transaction Detail'!F, A="<week-start>", E="AR Collection (Fcst)")`**
- **row 14 "Unbilled (Fcst)" {col} = `SUMIFS(... E="Unbilled (Fcst)")`**
The detail Total row is then a **tie check**: detail Total == row 13 + row 14 each forecast week (it shows the per-customer combination, but does not drive the summary). Typically AR dominates and Unbilled is $0 except where scheduled billings land (e.g. 2026-08-10: Unbilled only W18 $33 and **W24 $648,348** = the 10/10 Due-on-receipt batch).

## A. Customer detail (Collections by Customer)
**Data (`spot-collections-refresh-suiteql`):** AR = open `CustInvc` by due date; Unbilled = `billingschedulerecurrence` future billings by (send date + term days; **term id 4 = Due-on-receipt = 0 days**). Map schedule→customer via `transactionline.billingschedule → BUILTIN.DF(transaction.entity)`; strip prefix. Bucket to collection week (anchor W1 = 2026-05-01). Col map (CC col = week#+1): W16=Q, W17=R, W18=S, W19=T, W20=U, W21=V, W22=W, W23=X, W24=Y, W25=Z…

**Detail cell formula** (per customer × forecast week; combines AR + Unbilled):
```
=SUMIFS('Transaction Detail'!$F$5:$F$1700,'Transaction Detail'!$A$5:$A$1700,"<week-start>",
        'Transaction Detail'!$D$5:$D$1700,$A<row>,'Transaction Detail'!$E$5:$E$1700,"AR Collection (Fcst)")
        +SUMIFS('Transaction Detail'!$F$5:$F$1700,'Transaction Detail'!$A$5:$A$1700,"<week-start>",
                'Transaction Detail'!$D$5:$D$1700,$A<row>,'Transaction Detail'!$E$5:$E$1700,"Unbilled (Fcst)")
                ```
                **Steps:**
                1. Clear stale `Unbilled (Fcst)` / `AR Collection (Fcst)` TD rows for forecast weeks (`col A ≥ first forecast week-start`); write **fresh per-(week, stripped-customer, category)** TD rows (A/B = week-start, D = stripped name, E = category, F = amount). Keep last TD row ≤ 1700 (SUMIFS range) or widen it.
                2. Read existing detail names (col A, rows 19→`Total`−1). Compute the fresh customer set (stripped). **Add the missing ones** as new rows starting at the current Total row, and **move the Total row down** to just below the last new row.
                3. Fill **every** detail row × forecast column with the combined formula above.
                4. Rebuild the Total row: `{col}<total> = SUM({col}19:{col}<lastdetail>)` for every column; **repoint every existing reference to the OLD Total row** (grep the workbook — e.g. `H7`, `K14…P14` referenced `=…280`).
                5. **Summary lines stay separate:** row 13 = SUMIFS AR, row 14 = SUMIFS Unbilled (as above) for every forecast column. (Actual-week cols keep their existing behavior; both resolve to 0.)
                6. Verify: row 13 + row 14 == detail Total row == independently-computed fresh weekly (AR, Unbilled) totals, every forecast week.

                ## B. Vendor detail (Expense by Vendor)
                **Data (`spot-ap-cashflow` + `spot-vendor-timing`):** open AP bills bucketed by **due date** (past-due → first forecast week) + terms-based projection of recurring vendors. Strip `VEN####`. Write per-(week, stripped-vendor) `AP Pmts (Fcst)` TD rows.
                **Detail cell** (per vendor × forecast week): `SUMIFS(... "AP Pmts (Fcst)") + SUMIFS(... "Loan Pmts (Fcst)")` on `$D=$A<row>`.
                **Steps:** same pattern — clear stale `AP Pmts (Fcst)` rows; write fresh per-vendor rows (stripped); match to EV detail block (rows 21→`Total`−1); add missing vendors, move Total row, fix refs; fill every detail row × forecast col; rebuild Total `SUM(col21:col<lastdetail>)`; set EV **row 12 (AP forecast) for forecast cols = a direct `SUMIFS "AP Pmts (Fcst)"`** (replacing the smoothed run-rate hardcode) — or `= detail Total` if you want it detail-driven. EV col = week#+2 (W16=Q…). Keep payroll/benefits/loan/Ramp on their own summary rows. Exclude payroll/benefits/tax/insurance/Ramp-card vendors from the AP detail (handled by their own lines).

                ## Adding entities + moving the Total row (the fiddly part)
                Detail block is a fixed range `19…lastdetail` with `Total` directly below. openpyxl `insert_rows` does NOT fix formula refs — instead write new entities into the rows the old Total occupied and below, write the Total row at the new bottom, and **manually repoint every reference** to the old Total row number (grep the workbook). New rows only need the **forecast-column** formulas (actual-week cells resolve to 0). CC/EV tabs have ~1000 spare rows below the block.

                ## Validate
                - Recalc (LibreOffice, **0 new** formula errors — only pre-existing `Payroll Tax Trend!A21`).
                - row13 + row14 == detail Total == independently-computed weekly totals, every forecast week.
                - Spot-check big entities appear per-row (2026-08-10: S&D CarWash W24 $406,364; Crown Cork W18 $122,582; BS129 W24 $107,737).
                - Run **`spot-cashflow-output-check`** (history preserved).
                - ⚠️ **Double-count watch:** Collections-by-Customer also has tops-down milestone rows (11 New Business / 12 Renewals, at W18/W23) summed WITH this bottom-up detail — confirm which owns those weeks (2026-08-10 W18 total hit $2.12M = tops-down $1.24M + bottom-up $878K).

                ## Benchmark (2026-08-10)
                AR by week: W16 $177,831 · W17 $122,181 · W18 $878,381 · W20 $73,584 · W24 $44,982 · W25 $6,426. Unbilled: W18 $33 · **W24 $648,348**. 39 customers added (261 → 300); Total row moved 280 → 319.
                
