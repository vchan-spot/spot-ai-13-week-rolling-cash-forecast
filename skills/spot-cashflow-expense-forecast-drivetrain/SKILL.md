---
name: "spot-cashflow-expense-forecast-drivetrain"
description: "Build Spot AI's forecasted cash OUTFLOW (AP / vendor line) in the Weekly Cash Flow model VENDOR BOTTOMS-UP and TIMED TO ACTUAL DUE DATES — never smoothed. Amounts come from the current vendor budget (VMS/Agents Cash Economics workbook Q3, or Drivetrain); timing comes from each vendor's historical invoice cadence + net terms. Use every weekly refresh, when updating the AP/opex forecast, when the Expense-by-Vendor detail looks smoothed/flat/stale, or on 'update the AP forecast', 'time vendor payments to due dates', 'don't smooth the vendors'."
---

# Spot AI — Vendor AP cash-outflow forecast (bottoms-up, due-date-timed, NEVER smoothed)

Build the vendor AP line of the Weekly Cash Flow model **per vendor**, with each vendor's spend placed in the **week it is actually due** — from its historical invoice cadence + net terms. **Nothing is ever smoothed**: no flat run-rate, no monthly amount spread evenly across the weeks of a month, and no vendor paid every month unless it actually invoices every month. Pair with `spot-cashflow-bottomsup-check`.

Work on a downloaded `.xlsx` (openpyxl), recalc with LibreOffice, deliver a versioned file. Each run **appends exactly one new forecast week** (expanding horizon — see the update procedure) and re-times the vendors into the current window.

## 1. AMOUNTS — the vendor budget (what)
Preferred source: the **FP&A budget workbook** ("VMS vs Agents Cash Economics", tab `Drivetrain Upload` / `Complete Vendor & Payroll L4Q S`). Take the current-quarter **non-payroll** vendor spend per vendor (departments COGS + R&D + S&M + G&A; exclude the `Payroll`/HC dept and the `None` total rows). As of the FY27-Q3 pack this = **$3,579,519/qtr** ($1,193,173/mo). Apply manual overrides (e.g. **Searce ×2** if the budget understates it — confirm each run; the workbook already had Searce at $200K/mo, so no ×2 there).
Fallback if no workbook: Drivetrain `Opex_Corporate_Model2` (Current Plan, Base_Case, `Vendor` dim, non-salary) — see git history. Drivetrain's own `(blank)` bucket is a source-level line, booked as "Unallocated opex (Drivetrain source bucket)".
Each vendor's **quarter total** = its budget for the quarter (monthly × 3 if the budget is monthly-flat).

## 2. TIMING — historical due dates + cadence (when) — NEVER smooth
Pull each vendor's bill history from NetSuite (`transaction` type `VendBill`, last ~12–15 months), grouped by `BUILTIN.DF(entity)`:
- `due_day` = ROUND(AVG(day-of-month of `t.duedate`))  ← the day it's actually due (invoice date + terms already baked in)
- `net_days` = AVG term days (fallback for vendors with no due date)
- `months_billed` = COUNT(DISTINCT YYYY-MM)  ← cadence signal
- `last_bill` = MAX(trandate)  ← phase
(Query returns hundreds of rows — force it to spill to a file with an `RPAD('x',1600,'x')` pad column, then parse.)

**Cadence → number of payments in the quarter** (this is the anti-smoothing rule):
- `months_billed ≥ 10` → **monthly** → 3 payments (one per month).
- `months_billed 5–9` → **~bi-monthly** → 2 lumps.
- `months_billed ≤ 4` (or no history) → **quarterly / sporadic** → **1 lump** (the full quarter amount in a single week).

**Placement:** amount per payment = vendor quarter total ÷ (number of payments). Put each payment in the **week containing (its month + `due_day`)** — Fri-anchored week (anchor 2026-05-01). Phase multi-vs-single-payment vendors off `last_bill + cadence` so a quarterly vendor lands in the month it's actually next due. Clamp to the forecast horizon. Result: a **lumpy** weekly AP profile (big weeks where many vendors are due, quiet weeks otherwise) — e.g. Datadog (due ~24th, monthly) → W17/W21/W26; Velasea (~5mo/yr) → 2 lumps; Anthropic (~4mo/yr) → one lump; SVB → one lump.

## 3. Wire into the model (data-driven cascade)
- Book **one Transaction-Detail row per vendor per payment**: A/B = week-start string, C = memo (`AP <cadence> due~<day>`), D = vendor (matched to the EV detail name), E = `AP Pmts (Fcst)`, F = −amount. (This is far fewer rows than smoothing — ~1 per vendor per quarter-lump.)
- **Expense by Vendor** (col = week#+2): row 12 (label "Vendor Opex / AP (Drivetrain/Budget Fcst)") per forecast col = `SUMIFS(...,"AP Pmts (Fcst)")`; per-vendor **detail** rows = `SUMIFS(...,$D=$A{r},"AP Pmts (Fcst)")`. Append every budget vendor not already listed; move/rebuild the Total row; keep the SUMIFS ranges wide enough for all data.
- **Weekly Cash Flow** AP row = `='Expense by Vendor'!{col}7 + {col}12` for every forecast week (EV col = WCF col + 1).
- **No hardcodes outside Transaction Detail** (payroll, benefits, debt, ramp, bank-rec all live as TD rows too — the only literal allowed is Weekly Cash Flow B9 Beginning Cash).

## 4. Verify
- Per-vendor detail Total == row 12 == WCF AP, every forecast week (bottoms-up ties).
- **Not smoothed:** the weekly AP is lumpy; the vast majority of vendors are NOT paid an identical amount every week/month. A vendor paid the same figure in every forecast week is a smoothing bug.
- Quarter AP total == the vendor budget. Recalc 0 new errors. Run `spot-cashflow-bottomsup-check`. Deliver versioned + Change Log row naming the appended week.

## Standing rules
Vendor AP is bottoms-up + due-date-timed **at all times** — re-pull the budget and the vendor timing every refresh, re-apply overrides (Searce ×2), and **append exactly one new forecast week per run** (expanding horizon, never drop history). Never revert to a smoothed run-rate or flat-monthly-repeated payments.
