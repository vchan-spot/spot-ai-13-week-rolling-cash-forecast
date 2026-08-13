---
name: "spot-cashflow-bottomsup-check"
description: "QA gate that verifies Spot AI's Weekly Cash Flow model is ENTIRELY bottoms-up — every forecast (and actual) line tied to a real customer or vendor, with NO plugs, catch-alls, smoothed/flat-repeated run-rates, or hand-keyed summary totals — and that any newly-appended forecast week is wired correctly (EV col = WCF col +1, real-newline headers, currency format). Run at the end of every weekly refresh and any time the model is built or edited; if it finds a consolidation/plug/smoothing/append bug, it names it and triggers the fixing skill. Also use when someone asks to 'check the model is bottoms-up', 'make sure there are no plugs', 'no consolidation', 'nothing is smoothed', or 'did anything get summarized'."
---

# Spot AI — Bottoms-up integrity check (no plugs / no consolidation / no smoothing)

**Standing rule (Victor): the Weekly Cash Flow model is bottoms-up ALWAYS.** Every collection is a real customer; every outflow is a real vendor/payee. No plugs, no "(misc)/(other)/(pool)" catch-alls, no smoothed run-rate, no hand-keyed summary total. Run this **at the end of every refresh** and after any build/edit. When it finds a violation, name it and **trigger the fixing skill** — do not leave it.

Work on the workbook you just built (or a fresh download). Recalc with LibreOffice first so values are current.

## What to check

### A. No placeholder/plug ENTITIES in Transaction Detail (forecast + actual)
Scan `Transaction Detail` col D (entity) for the forecast categories (`AR Collection (Fcst)`, `Unbilled (Fcst)`, `Paystand Recurring (Fcst)`, `AP Pmts (Fcst)`) and the actual `Collections` category. **FLAG** any entity that is not a real customer/vendor name:
- regex-ish matches: `(` at start, `pool`, `misc`, `other`, `plug`, `run-rate`, `reconcil`, `summary`, `unallocated`, `blank`, `TBD`, `various`, empty/blank D.
- **Allowed exception (one only):** Drivetrain's own source bucket booked as `Unallocated opex (Drivetrain source bucket)` — Drivetrain provides ~$135K/mo of opex with no vendor. That is a *source-level* line, not a model plug. Everything else is a violation.

### B. No plug ROWS in the detail blocks
Scan `Collections by Customer` (rows 19→total) and `Expense by Vendor` (rows 21→total) first column for the same tokens. A row named `(Unbilled - … misc …)`, `(Other opex vendors …)`, `(AR forecast)`, `Actual collections reconciled to summary …`, etc. is a violation. (The Drivetrain `Unallocated opex` row is the sole allowed aggregate.)

### C. Summary rows are FORMULAS, not hardcodes
Every forecast summary cell must be a `SUMIFS` over Transaction Detail (or a `SUM`/cross-sheet ref), never a typed number:
- Collections by Customer rows 13 (AR) & 14 (Unbilled) and the Paystand contribution → direct `SUMIFS`.
- Expense by Vendor row 12 (Vendor Opex / AP) → `SUMIFS(... "AP Pmts (Fcst)")`, NOT a hardcoded `-174404`-style value.
- FLAG any forecast summary cell whose value is a literal number. The ONLY allowed hardcode in the whole workbook is the Transaction Detail source rows and WCF `B9` Beginning Cash (seed).

### D. Detail ties to summary, every forecast week
For each forecast week column: sum the per-entity detail rows and confirm it equals the summary line it feeds (CC detail == AR + Unbilled + Paystand summary; EV per-vendor detail == row 12). A gap means an entity is missing from the detail block (dropped by a `=col{Total}` reference or an un-appended entity) → violation.

### E. Cascade wiring
- Collections summary rows must be direct `SUMIFS`, never `={col}{DetailTotalRow}` (that silently drops unlisted entities).
- WCF collections/AP reference the cascade tabs; no hand-keyed WCF forecast outflow numbers that bypass Expense by Vendor.

### F. No SMOOTHED / flat-repeated vendor payments (Victor: "nothing should ever be smoothed")
Vendor AP must be timed to each vendor's real cadence (historical invoice due-day + net terms), so a vendor's per-week outflow is **lumpy**, not a constant repeated every week. FLAG smoothing:
- For each vendor detail row in `Expense by Vendor`, read its forecast-week values. **A vendor whose non-zero forecast weeks are all (near-)equal across 3+ consecutive weeks is smoothed** → violation. Real cadence-timed vendors pay in 1–3 discrete weeks per quarter (monthly=3 lumps, bi-monthly=2, quarterly/sporadic=1), zeros elsewhere.
- Also FLAG the **row-12 total (Vendor Opex/AP)**: if the weekly AP total is a flat constant (e.g. every week ≈ −$237K) it means the underlying vendors were spread evenly (old run-rate), not due-date timed. The correct total is lumpy week-to-week (reference shape W24–W29 ≈ −190K / −333K / −208K / −3K / −137K / −269K).
- Heuristic: `stdev(nonzero weekly AP total)/mean < 0.15` over the forecast horizon ⇒ likely smoothed ⇒ inspect vendor rows.
On failure → run **`spot-cashflow-expense-forecast-drivetrain`** (re-time every vendor by historical due-day + cadence + net terms; NEVER divide a monthly/quarterly amount evenly across weeks).

### G. Append integrity (outer weeks added this run)
Each refresh appends exactly ONE forecast week (dry-runs may add 2), never dropping history. Verify the newly-added column(s):
- **EV column = WCF column + 1** (and CC column = WCF column). WCF forecast-outflow formulas (rows 19–24) must reference the EV column that is **one right of** the WCF column. An off-by-one silently pulls the PRIOR week's payroll/AP into the new week — symptom: a payday appears one week early, or AP is shifted by a week. Confirm the payday lands in the correct Fri–Thu bucket (e.g. an 11/15 semi-monthly payday belongs to the week that contains 11/15).
- **Header line breaks are real newlines** (`"W29\n11-13\nFORECAST"`), not the literal two-character string backslash-n. FLAG any header cell containing a backslash-n literal and rewrite with a real newline (`chr(10)`), matching the original W1-W24 headers.
- New CC/EV columns carry the currency number format of the prior column (`\$#,##0;"($"#,##0\);\-`), and the outline/grouping is intact and collapsed (`summaryRight=True`).
On failure → fix the WCF EV-column references and header strings directly, then recalc.

## On failure — trigger the fix (don't just report)
- **Unbilled/AR/collections plug or missing per-customer detail** → run **`spot-collections-refresh-suiteql`** (rebuild unbilled+AR bottoms-up) and **`spot-cashflow-populate-entity-detail`** (populate/expand the per-customer detail, direct SUMIFS). For an untraceable "(misc schedules)" plug, resolve each billing schedule to its customer (billing-schedule name, e.g. "S And D Annual" → S&D; drop detached template schedules with no customer/transaction link).
- **AP/vendor consolidation, run-rate hardcode, "(other vendors)" catch-all, or SMOOTHED vendor payments** → run **`spot-cashflow-expense-forecast-drivetrain`** (re-book every Drivetrain vendor per-vendor, due-date + cadence timed).
- **Hand-keyed summary total** → replace with `SUMIFS` and book the underlying per-entity rows.
- **Append off-by-one / literal-newline header / lost format** → fix in place (see F/G) and recalc.
After fixing, recalc (0 new errors) and re-run this check until it passes clean.

## Report format
List each violation as `tab / row / label / $amount / category`, the trigger taken, and a final line: `BOTTOMS-UP: PASS` or `FAIL (n violations, m fixed)`. A known-and-accepted item (the Drivetrain `Unallocated opex` bucket; a documented actuals reconciling line pending full per-customer actual booking) is reported as **ACCEPTED**, not a silent pass.

## Quick implementation (openpyxl)
Recalc a copy, then in Python: load `Transaction Detail`, group `SUMIF`-style by (category, entity) for forecast weeks; flag entities matching the token list; load CC/EV detail first-columns and flag the same; read the forecast summary cells and flag any non-formula numeric; compute per-week detail-vs-summary deltas; for each EV vendor row compute the coefficient of variation of its non-zero forecast weeks (flag near-constant); for the newest appended columns verify EV=WCF+1 references, real-newline headers, and copied currency format. Emit the report and, on any violation, invoke the relevant fixing skill above.
