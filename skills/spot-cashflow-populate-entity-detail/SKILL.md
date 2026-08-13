---
name: "spot-cashflow-populate-entity-detail"
description: "Guarantee that Spot AI's Weekly Cash Flow model has per-ENTITY forecast detail — every customer's AR/Unbilled in Collections by Customer and every vendor's AP in Expense by Vendor — populated for ALL forecast weeks, not just summary totals. Run on every weekly cash-forecast refresh, or whenever the customer/vendor detail blocks look empty, aggregated (e.g. a single '(AR forecast)'/'(pool)' row), or stale in forecast periods, or when someone asks to 'populate the customer/vendor forecast detail', 'show per-customer/per-vendor forecast', or 'make sure all vendor and customer detail is filled in'."
---

# Spot AI — Populate Customer & Vendor FORECAST Detail (entity-level, all forecast weeks)

**Goal:** every forecast week of the Weekly Cash Flow model shows the forecast **per entity**, not just a category total or a single aggregate placeholder row. Customer weeks show each customer's AR + Unbilled; vendor weeks show each vendor's AP. Summary rows tie to the detail block, and the detail block lists every entity that appears in the forecast data.

Run this **every weekly refresh** (after the collections/unbilled and AP overlays are pulled) and any time the detail looks empty, collapsed into one `(AR forecast)` / `(unbilled)` / `(pool)` row, or short of the horizon. It is the entity-detail QA gate for the model.

The live model is the Google Sheet **"Spot AI — Rolling 13 Weekly Cash Flow"** (file id `1A6VZK2iPXOcyyBTlVnZK36CuK3Wv3eZXdy1ZYvOi6D0`). Work on a downloaded `.xlsx` copy (openpyxl), recalc with LibreOffice, deliver a versioned file — do not push unless explicitly asked.

## The model is a single-source cascade
Everything rolls up from the **Transaction Detail** tab via `SUMIFS`:
- Col A = week-start string `YYYY-MM-DD` (Fri; anchor **W1 = 2026-05-01**), C = description, D = **entity name**, E = **cash-flow category**, F = cash impact. SUMIFS range is `$5:$1700`.
- A cell is "actual" vs "forecast" purely by which TD rows exist for that week — there is no separate formula path.

**Forecast categories that need per-entity rows:**
- Customers → `AR Collection (Fcst)` and `Unbilled (Fcst)` (one TD row per customer per collection week).
- Vendors → `AP Pmts (Fcst)` (one TD row per vendor per pay week).

The single biggest failure mode: booking a forecast week as ONE aggregate row (`D = "(AR forecast)"`, `"(unbilled)"`, `"(pool)"`) — the summary total is right but the entity detail block shows nothing for that week. This skill replaces those aggregates with per-entity rows.

## Tab layouts (verify against the live file each run — rows/cols drift as weeks are appended)
**Collections by Customer** — column = week# + 1 (`B`=W1 … `X`=W23 …), Total column immediately right of the last week.
- Summary rows: 7 Collections · 8 Interest · 9 Equity/ESPP · 10 Other Income · 11 New Business (tops-down) · 12 Renewals Won (tops-down) · **13 AR Collection (Fcst)** · **14 Unbilled (Fcst)** · 15 Total = `SUM(row7:row14)`.
- Detail block: customers in **rows 19 … N**, then a **Total row** (`Total Collections (detail…)`) = `SUM(B19:B{N})`; detail header row 18 carries the week labels.

**Expense by Vendor** — column = week# + 2 (`C`=W1 … `Y`=W23 …), Total column right of the last week.
- Summary rows: 6 Payroll & Benefits · 7 AP / Vendor Pmts · 8 Ramp Cards · 9 Debt Service · 10 Direct Opex & Other · 11 CapEx · 12 Vendor Opex Run-Rate (Fcst) · **13 Loan Pmts (Fcst)** · 14 Benefits & Insurance (Fcst) · 15 Ramp Payoff (Fcst) · 16 Payroll (Fcst) · 17 Payroll Taxes (Fcst).
- Detail block: vendors in **rows 21 … M**, then a **Total by vendor** row = `SUM(col21:col{M})`.

## Procedure
1. **Identify forecast weeks** = columns whose Weekly Cash Flow row-7 status is `Forecast` (skip Actual / partial-actual weeks — their detail is bank-pool actuals, handled elsewhere). Get each week's start string and its CC column (week+1) and EV column (week+2).

2. **Get the per-entity forecast** the detail must show. Use the values already in Transaction Detail if per-entity rows exist; otherwise pull fresh (SuiteQL, as-of = today):
   - **Unbilled** by (customer, collection week): future `billingschedulerecurrence` (recurrencedate > as-of) → collection date = recurrencedate + term days (term id 4 "Due on receipt" = 0 days; map via `term.daysuntilnetdue`); map billing schedule → customer. (See `spot-collections-refresh-suiteql`.)
      - **AR** by (customer, due week): open `CustInvc` (`foreignamountunpaid > 0`) bucketed on due date, forecast weeks only, terms-only (don't project past-due).
         - **AP** by (vendor, pay week): open AP by due date + vendor-timing overlay (see `spot-ap-cashflow` / `spot-vendor-timing`).

         3. **Write per-entity Transaction Detail rows** (one row per entity per week), replacing any aggregate placeholder:
            - First **zero** the stale/aggregate forecast rows for the affected weeks: for TD rows with week ≥ first forecast week and category in {`AR Collection (Fcst)`,`Unbilled (Fcst)`,`AP Pmts (Fcst)`}, set F = 0 (keep the rows, just null them) — including any `(AR forecast)`/`(unbilled)`/`(pool)` aggregates.
               - Then append per-entity rows: A/B = week-start string, C = `"… forecast (refreshed <asof>)"`, **D = the exact entity name as it appears in the detail block**, E = the forecast category, F = amount. Keep the last data row ≤ 1700 (the SUMIFS ceiling); if it would exceed, widen every `$…$1700` range first.

               4. **Make summary rows read the data directly (NOT the detail-block total).** Set, for each forecast column:
                  - CC `{col}13` = `SUMIFS('Transaction Detail'!$F$5:$F$1700, $A$5:$A$1700, "<wk>", $E$5:$E$1700, "AR Collection (Fcst)")`
                     - CC `{col}14` = same with `"Unbilled (Fcst)"`
                        - EV AP forecast likewise via `"AP Pmts (Fcst)"`.
                           Never point row 14 at `={col}{DetailTotalRow}` — the detail-block total silently **drops any entity not already listed in the block**, which is how the forecast goes stale/understated.

                           5. **Populate the DETAIL BLOCK per entity, expanding the list when new entities appear.** For each detail row (existing entity), the forecast cell is a per-entity SUMIFS:
                              - CC customer detail `{col}{r}` = `SUMIFS(…,"<wk>", $D…, $A{r}, $E…,"AR Collection (Fcst)") + SUMIFS(…,"<wk>", $D…, $A{r}, $E…,"Unbilled (Fcst)")` (AR **and** Unbilled combined on the customer line; keep rows 13 & 14 as separate visible summary lines).
                                 - EV vendor detail `{col}{r}` = `SUMIFS(…,"<wk>", $D…, $A{r}, $E…,"AP Pmts (Fcst)")`.
                                    - **Entity name-matching:** strip `CUS####` / `VEN####` prefixes and match to the existing detail names; a NetSuite name like `"CUS099 S and D CarWash Management, LLC"` matches the detail row `"S and D CarWash Management, LLC"`. Book the TD row's D with the name the detail row uses so the per-entity SUMIFS matches.
                                       - **Add missing entities:** any entity in the forecast data not in the detail list → append a new row **below the last detail row** (above the Total), give it the same per-entity SUMIFS across every week column, then **move the Total row down**, rebuild it as `SUM(first_detail:new_last_detail)` for every column, and **fix any reference to the old Total row** (e.g. a lingering `={col}{oldTotal}`).

                                       6. **Handle the Total column and appended weeks.** When a new week was appended the entity Total **column** shifts one right; rebuild each detail row's Total as `SUM(B{r}:{lastweekcol}{r})` and clear the vacated old-Total column in the detail rows so it doesn't re-sum into a phantom.

                                       7. **Tops-down vs bottom-up overlap.** The model also has tops-down milestone rows (CC row 11 New Business, row 12 Renewals Won) that land at specific weeks. If bottom-up AR is now populated for those same weeks, the week double-counts. Flag it and confirm which basis owns the week before finalizing (don't silently stack both).

                                       ## Verify before delivering
                                       - **Detail ties to summary, every forecast week:** CC detail-Total row == CC row 15 minus the tops-down/other summary lines it doesn't include (at minimum, `sum(detail AR+Unbilled) == row13 + row14`); EV vendor-detail Total == EV AP forecast line. No forecast week left with a blank or single-aggregate detail column.
                                       - **No entity dropped:** every entity in the forecast TD rows appears as a detail row.
                                       - **Recalc** with LibreOffice headless (`--convert-to xlsx`, set `fullCalcOnLoad`) → **0 new formula errors** (the pre-existing `Payroll Tax Trend!A21 #VALUE!` is expected). If a W7-type `=Collections by Customer'!H7` recalcs to 0 under LibreOffice, pin that cell in the recalc copy only, keep the formula in the deliverable.
                                       - Deliver a versioned `.xlsx`; add a Change Log row naming the weeks/entities populated. Push only if explicitly asked.
                                       
