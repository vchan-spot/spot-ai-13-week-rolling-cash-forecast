---
name: spot-vendor-timing
description: |
  Time a budgeted vendor amount so cash-out lands in the RIGHT forecast week, driven by the vendor's
    INVOICE trend: pull the vendor's NetSuite bill (invoice) dates, determine its invoicing cadence
      (monthly / quarterly / annual), project the next invoice dates forward, and assume each invoice is
        PAID 7 days after it is issued — bucketing that pay date into its Fri-Thu week. Cadence comes from an
          FP&A-owned vendor_cadence table (authoritative), falling back to invoice-date inference. Use whenever
            the user asks to "fix vendor payment timing", "get the right timing for the opex/vendor budget",
              "time the vendor forecast off invoices", or "when will vendor X actually get paid".
              ---

              # Spot AI — Vendor Budget Timing Adjustment

              Times a vendor budget by its **invoice trend**, not by guessing from past payment gaps. For each vendor:
              learn its invoicing cadence, project future invoice dates, and place cash-out **7 days after each
              invoice** (Spot's typical pay lag). This is the timing layer only — feed it invoice dates + amounts and
              it returns the week each dollar sits in.

              ## Inputs
              - `history`: past **invoice / bill dates** per vendor — NetSuite `VendBill.trandate` (NOT payment dates).
              - `budget`: amount per vendor (per-invoice by default; annual total if `--annual`).
              - `weeks`: forecast week-start dates (Fri-anchored), in order. Week 1 = soonest week.
              - `cadence` (optional): overrides the bundled `references/vendor_cadence.json`.
              - `open_ap` (optional): real open bills `{vendor,duedate,unpaid}` — placed at their own due date
                (past-due → Week 1); the invoice-trend projection covers everything after.

                ## Cadence (invoicing frequency — NOT payment-date gaps)
                Source priority:
                1. **Explicit map** `references/vendor_cadence.json` (substring match on vendor name) — always wins.
                2. **Fallback: infer from the INVOICE-date trend** (median gap ≥250d→annual, ≥70d→quarterly, else
                   monthly). Never infer from bank-payment dates — payments get batched/delayed and mislabel cadence.
                   3. **Unknown** → default monthly and **flag** it (`source` column) so it gets added to the map.

                   ## Timing rule (the core adjustment)
                   1. Project invoice dates from the trend: anchor = the vendor's **last invoice date**, then step forward
                      by cadence (monthly = +1 month same day-of-month, quarterly = +3 months, annual = +12 months).
                      2. **Pay date = invoice date + 7 days** (`--lag`, default 7).
                      3. Place the per-invoice amount in the Fri–Thu week that contains the pay date. Drop projections whose
                         pay date is past the last forecast week.
                         4. Real `open_ap` bills are booked at their actual due date first; the projection fills the rest.
                         Per-invoice amount: default `budget` = each invoice; with `--annual`, per-invoice = annual ÷ (12 / 4 / 1).

                         ## Use
                             python scripts/place_by_cadence.py --weeks weeks.json --budget budget.json \
                                     --history invoice_dates.json [--cadence vendor_cadence.json] [--open-ap open_ap.json] \
                                             [--annual] [--lag 7] --out placed.csv
                                             Output `placed.csv`: vendor × week matrix + `cadence` + `source` columns, ready to paste into the
                                             Expense-by-Vendor detail block. Vendors marked `DEFAULT-monthly(FLAG)` need a cadence added.

                                             ## Gotchas
                                             - Pull invoice dates from **VendBill.trandate** (direct-pay bills). Card-paid (Ramp) vendors don't belong.
                                             - Exclude vendors already on dedicated model lines (payroll, benefits, 401k, loan, tax) — double-count.
                                               See references/timing_rules.md for the exclusion substrings.
                                               - Keep `vendor_cadence.json` and `vendor_cadence.md` in sync as you learn a vendor's true billing terms.
                                               
