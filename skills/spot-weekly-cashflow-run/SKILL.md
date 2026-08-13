---
name: "spot-weekly-cashflow-run"
description: "End-to-end orchestrator for Spot AI's Weekly Cash Flow refresh & distribution. Runs the full Monday 7:00 AM PT workflow in order: pull NetSuite actuals, pull Drivetrain plan + actuals, pull Ramp card spend, rebuild the bottoms-up model (collections/AP/payroll) and extend the horizon one week, reconcile + flag anomalies, publish the model, distribute the summary to Slack, and confirm. Use to run the weekly cash flow, do the Monday cash refresh, refresh and send the cash forecast, or on the weekly schedule. Sequences the sub-skills, continues past any single failure and notes it, and never fabricates numbers."
---

# Spot AI — Weekly Cash Flow refresh & distribution (end-to-end run)

Master orchestrator for the weekly cash-flow refresh (runs **Monday 7:00 AM PT**). Executes the steps below **in order**, calling each sub-skill. **If any step fails, continue with the others and note the failure in the final summary. Never fabricate numbers — if a source is unreachable, say so.** In an **unattended/scheduled run, deliver files for review and do NOT push the live Google Sheet or send Slack live** (post a draft) unless explicitly told to publish.

Live model: Google Sheet `1A6VZK2iPXOcyyBTlVnZK36CuK3Wv3eZXdy1ZYvOi6D0`. NetSuite account `7735559`. Drivetrain server `d0352cc3`. Anchor W1 = 2026-05-01, Fri–Thu weeks. Deliverables go to the `Weekly Cash Forecast/` project folder.

## Standing model constraints (enforced every run)
Bottoms-up **always** (every collection a real customer, every outflow a real vendor); **no smoothing**, **no plugs/consolidation/summary lines**; **no hardcodes** except Transaction Detail rows + WCF `B9` beginning-cash seed; vendor AP timed to actual historical due dates + cadence + net terms; HC/vendor amounts tie to the workbook Q3 budget; **expand the horizon by exactly one week per run** (never drop history); pin cash to bank statements. Searce ×2 when using Drivetrain.

## Steps
**Step 1 — NetSuite actuals** → `spot-netsuite-actuals-pull`. Bank balances (pool 662/665/688/669/687 + UF 122), collections last 4 weeks (SS1722 basis), cash-out by category last 4 weeks. Note total cash, WoW change, largest single movement.

**Step 2 — Drivetrain plan + actuals** → `spot-cashflow-expense-forecast-drivetrain` (vendor bottoms-up non-salary opex, Searce ×2) + `spot-payroll-hc-forecast` (payroll/HC, semi-monthly). Confirm the active forecast version + actuals version; apply the **VigilanteX 1-quarter lag** (shift lag-list bookings ARR forward 13 weeks). Refresh collections/unbilled via `spot-collections-refresh-suiteql` (+ `spot-unbilled-forecast` if unbilled looks stale).

**Step 3 — Ramp card spend** → `spot-ramp-scheduled-payments` / `spot-ap-cashflow`. Try the Ramp bulk export; if it returns 403 (scope not granted), skip and note "Ramp bulk export blocked — using NS Ramp payments" (NS-side Ramp vendor payments, vendor 101180, as fallback).

**Step 4 — Build the model, extend one week, reconcile & flag** →
- Rebuild actuals with `spot-weekly-cashflow` (flip newly-closed weeks to actual; refresh customer/vendor detail for those weeks).
- **Extend the horizon exactly one week** with `spot-cashflow-extend-one-week` (self-verifying: EV col = WCF col +1, real-newline headers, detail-ties-to-summary, 0 new errors).
- Populate per-entity forecast detail with `spot-cashflow-populate-entity-detail` / `spot-cashflow-forecast-entity-detail`.
- Reconcile + flags with `spot-cashflow-reconcile-flags` (total cash vs last Monday, 13-week ending cash, plan-vs-actual variance, anomaly + reconciliation-break flags).
- **QA gates:** `spot-cashflow-bottomsup-check` (no plugs/smoothing; append integrity) and `spot-cashflow-output-check` (history didn't change). Fix and re-run until clean.

**Step 5 — Publish** → update `LIVE_SNAPSHOT` in `spot-cf-v6.js` with the new bank/collections/cashOut values and prepare the Drivetrain payload for `ingestDrivetrain()`. If the app is deployed with the Netlify Function (live refresh), confirm the refresh endpoint returns **200**. **Attended runs only** may replace-in-place the live Google Sheet; unattended runs deliver files for review.

**Step 6 — Distribute** → `spot-cashflow-slack-distribute`. Post the summary to **#finance** (or DM CFO / Lisa Manney / AJ Chlarson via `slack_search_users`) in the standard template. Unattended → draft, don't send live.

**Step 7 — Confirm** → reply in the Cowork thread with a one-line status:
`Sent {date} — cash ${total}, {n} flags, Drivetrain {ok/degraded}, Ramp {ok/blocked}.`

## Archival (after a clean build)
Run the archive + week-over-week forecast-accuracy comparison (`archive_final_cash_model.py` + `compare_cash_forecast_accuracy.py` in the project folder) so each run's forecast is graded against later actuals.

## Failure handling
Each step is independent enough to skip on error. Capture every failure (which source, which query) and surface them in the Step 7 status and the Slack summary's Flags/Failures. Do not estimate around a missing source.
