---
name: "spot-cashflow-reconcile-flags"
description: "Reconcile Spot AI's Weekly Cash Flow snapshot and flag anomalies — Step 4 of the weekly run. Ties NetSuite actuals to Drivetrain actuals, computes total cash vs last Monday, the 13-week ending-cash projection, and plan-vs-actual variance (payroll, benefits, opex, collections), then raises the standing anomaly + reconciliation-break flags. Use during the weekly cash refresh after actuals + forecast are built, or when asked to 'reconcile the cash snapshot', 'flag cash anomalies', 'check plan vs actual variance', or 'run the cash flags'. Never fabricates — reports unreachable sources as gaps."
---

# Spot AI — Reconcile & flag anomalies (Weekly Cash Flow, Step 4)

Builds the reconciled weekly cash snapshot and raises the standing flags. **Step 4** of the weekly run (orchestrator `spot-weekly-cashflow-run`). Runs after Step 1 actuals (`spot-netsuite-actuals-pull`), Step 2 Drivetrain plan/actuals, and Step 3 Ramp. **Never fabricate** — if a source is unreachable, note it and continue.

## Inputs
- NetSuite actuals (bank balances, collections, cash-out by category) from `spot-netsuite-actuals-pull`.
- Drivetrain plan + actuals (new bookings/renewal ARR, payroll/benefits/opex/headcount plan, next 3 months) — server `d0352cc3`; apply the **VigilanteX 1-quarter lag** (shift lag-list bookings ARR forward 13 weeks before treating as collectible).
- Ramp card spend (or NS-side Ramp vendor payments, vendor 101180, if the Ramp bulk export is blocked).
- The built Weekly Cash Flow workbook (current forecast + Drivetrain-fed collections/opex baselines).

## What to compute
1. **Total cash now vs last Monday** — $ and %.
2. **13-week ending-cash projection** — from the current forecast + Drivetrain-fed collections and opex baselines.
3. **Plan-vs-actual variance (QTD)** for **payroll, benefits, opex, collections** — actual (NetSuite) minus Drivetrain plan, per category.

## Flags to raise
**Anomalies:**
- Any **bank balance moved > 20% week-over-week**.
- **Payroll** outside the **$1.0M–$1.4M / month** band.
- **Collections < $150K** for the week.
- Any **past-due AP > $100K**.

**Reconciliation breaks:**
- **NetSuite actuals vs Drivetrain actuals diverge > 10%** for the same category/period.

## Output
A compact snapshot block: total cash + WoW ($/%); the 13-week ending-cash figure (and the week it lands); the four QTD plan-vs-actual variances; and a **Flags** list (each with the metric, the threshold breached, and the value) or "none". Also surface the **largest single movement** from Step 1. Hand this to `spot-cashflow-slack-distribute` (Step 6) and include it in the run's final summary. List any source that failed to load under "Failures" so the flags are read with the right caveats.

## Notes
- Weeks are Fri–Thu, anchor W1 = 2026-05-01, so QTD windows align with the model.
- Prefer the QA skills for structural integrity: `spot-cashflow-output-check` (history didn't change) and `spot-cashflow-bottomsup-check` (no plugs/smoothing) — this skill is about the *numbers* (variance + anomaly), not the model's structure.
