---
name: spot-ramp-scheduled-payments
description: |
  Pull every Ramp bill that has a scheduled (queued) payment date — i.e. approved bills that Ramp
    is set to actually disburse, identified by a non-null payment_date. Use this whenever the user
      wants to see what's "scheduled to be paid via Ramp", "queued payments", "what's about to go out
        on Ramp", "scheduled disbursements", "upcoming Ramp payments", "bills with a payment date", or
          asks to review/total payments before they release — even if they don't say the word "scheduled".
            Returns a dated table grouped by payment date with a total, and flags anything unusual before
              release. Triggers on phrases like "pull all bills scheduled to be paid via Ramp", "what's
                scheduled to pay on Ramp", "queued Ramp payments", "upcoming disbursements", "Ramp payments going
                  out", "bills with a scheduled payment date", "what's about to release on Ramp", and "show me this
                    week's scheduled AP payments". Also use proactively when reviewing AP right before a payment run.
                    ---

                    # Spot AI — Ramp Scheduled Payments

                    ## What it does
                    Pulls every Ramp bill that is **queued to actually disburse** — defined as an approved bill with a
                    **non-null `payment_date`**. Ramp does not reliably stamp the `PAYMENT_SCHEDULED` status summary, so
                    the reliable signal that a payment is scheduled is the presence of a concrete `payment_date`.

                    Output: a table grouped by payment date, with a grand total and a short pre-release flag list.

                    ## How to run it

                    ### 1. Load the Ramp bill tools
                    If not already loaded, call `tool_search(query="ramp bills payments scheduled")` to load
                    `Ramp:ramp_search_bills` (and siblings).

                    ### 2. Pull the candidate bills
                    Ramp has no single status that cleanly means "scheduled," so query the open/payable statuses and
                    filter client-side on `payment_date`. Call `ramp_search_bills` with:

                    - `status_summaries = ["PAYMENT_SCHEDULED", "PAYMENT_READY", "PAYMENT_PROCESSING", "AWAITING_RELEASE", "PAYMENT_NOT_INSTRUCTED"]`
                    - `limit = 100`
                    - `include_paid = false` (we want what's still going out, not what already cleared)

                    Paginate via `next_page_cursor` if `total_found` exceeds the returned count.

                    > Also run a direct `status_summaries=["PAYMENT_SCHEDULED"]` pass — if Ramp ever does populate it,
                    > those bills should be included even if a payment_date is somehow absent.

                    ### 3. Filter to scheduled
                    Keep only bills where **`payment_date` is not null**. Drop everything else (those are approved but
                    not yet instructed to pay, so they are *not* scheduled). Also drop anything with
                    `payment_status == "PAID"` or a non-null `paid_date` (already disbursed).

                    ### 4. Present
                    Sort by `payment_date` ascending, then by amount descending within each date. Show a table:

                    | Vendor | Invoice | Amount | Payment Date | Memo |

                    Group or subtotal by payment date, and give a **grand total** of scheduled cash out.

                    ### 5. Pre-release flags (always include)
                    Scan the scheduled set and call out, briefly, anything worth a second look before it releases:
                    - **Stale due dates** — original `due_date` more than ~6 months before today (often re-discovered
                      legacy invoices; confirm they're still owed).
                      - **Large items** — anything notably large relative to the rest of the run.
                      - **Past-due** (`is_past_due = true`) bills that are scheduled — fine, but worth noting.

                      Keep flags to a sentence or two; don't editorialize beyond what the data shows.

                      ## Definitions
                      - **Scheduled payment** = approved Ramp bill with a non-null `payment_date` (the date Ramp will
                        attempt the disbursement). `payment_date_range_end` may equal `payment_date` for a single-day window.
                        - **Not scheduled** = approved/open bill with `payment_date = null`. These are awaiting payment
                          instruction and should be excluded (offer them as a separate "approved, awaiting scheduling" list
                            only if the user asks).

                            ## Optional follow-ups the user may want
                            - Break the total out by **funding source / payment method** — use `ramp_get_bill_details` per bill.
                            - Split into a separate **"approved but not yet scheduled"** list (the excluded `payment_date = null` set).
                            - Cross-check against the AP forecast (`spot-ap-cashflow`) for the same window.

                            ## Notes
                            - Amounts are USD unless `currency` says otherwise — surface any non-USD bill explicitly.
                            - Do not instruct, release, schedule, or cancel any payment. This skill is read-only; payment actions
                              are performed by the user in Ramp.
                              
