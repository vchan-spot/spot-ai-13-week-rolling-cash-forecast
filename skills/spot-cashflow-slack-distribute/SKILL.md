---
name: "spot-cashflow-slack-distribute"
description: "Distribute Spot AI's Weekly Cash Flow summary to Slack (Step 6 of the weekly run). Formats the exact finance-team template (cash position + WoW, per-account balances, last week's collections, 13-week projected ending cash, Drivetrain plan-vs-actual QTD variances, flags, dashboard link) and posts to #finance, or DMs named recipients (CFO, Lisa Manney, AJ Chlarson) resolved via Slack user search. Use after the reconcile/flags step, or when asked to post the weekly cash summary to Slack, send the cash flow update, or DM the cash summary. In unattended runs, post a draft for review rather than sending, unless told otherwise."
---

# Spot AI — Distribute the Weekly Cash Flow summary via Slack (Step 6)

Posts the weekly cash summary to Slack in the finance-team template. **Step 6** of the weekly run (orchestrator `spot-weekly-cashflow-run`). Consumes the reconciled snapshot + flags from `spot-cashflow-reconcile-flags` (Step 4) and the published dashboard link from Step 5.

Connector: the Slack MCP. Post to **#finance** (default) or **DM** named recipients. Resolve recipient user IDs with **`slack_search_users`** (e.g., CFO, **Lisa Manney**, **AJ Chlarson**) before DMing.

## Message format (mrkdwn — use real values, never placeholders)
```
:moneybag: *Weekly Cash Flow — week of {date}*
*Cash position:* ${total} ({+/-}{wow_change} vs last Monday)
 • SVB Sweep ${x} · JPM ${y} · SVB Checking ${z}
 *Collections last week:* ${amount} ({n} payments)
 *13-week projected ending cash:* ${projection} (week of {end_date})
 *Drivetrain plan vs actual (QTD):*
  • Collections: {variance}  • Payroll: {variance}  • Opex: {variance}
  *Flags:* {anomalies or "none"}
  *Dashboard:* {netlify_url}
  ```
  - `{date}` = the Monday of the current week. Money formatted with thousands separators; WoW change carries its sign and can show $ and %.
  - Per-account line lists each bank account from the Step 1 balances (add/rename accounts if the pool changed).
  - `{variance}` values come from the QTD plan-vs-actual in Step 4 (actual − plan per category).
  - `{anomalies}` = the flag list from Step 4, or the literal word `none`.

  ## How to run
  1. Assemble the values from the Step 4 snapshot (cash + WoW, per-account balances, last week's collections + payment count, 13-week ending cash + its week, the three QTD variances, flags) and the Step 5 dashboard URL.
  2. Build the message text above with real values.
  3. **Send** to #finance with `slack_send_message` (or DM: resolve each recipient via `slack_search_users`, then message them). **In an unattended/scheduled run, post a draft (`slack_send_message_draft`) or hold for review instead of sending live**, unless the run was explicitly told to send.
  4. Return the permalink/confirmation for the run's final status line.

  ## Guardrails
  - Never fabricate numbers — every figure must trace to the Step 1/2/4 outputs. If a value is missing because a source failed, write `n/a (source unavailable)` rather than guessing.
  - Keep it to the template; this is a leadership-facing summary, so no extra commentary unless a flag needs a one-line explanation.
  
