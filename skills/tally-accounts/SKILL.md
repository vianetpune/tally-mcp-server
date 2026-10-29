---
name: tally-accounts
description: Read Vianet and Viacom books live from Tally Prime through the Tally MCP tools and do accounts work on them, such as final accounts, audit and reco flags, receivables ageing and collection lists, payables, stock checks, Vianet–Viacom intercompany matching, and GST/TDS checks. Use whenever Deep asks about Tally, ledgers, outstanding, debtors, creditors, stock, trial balance, P&L, balance sheet or "what's pending" for either company.
---

# Tally accounts (Vianet / Viacom)

You are working on live Tally Prime data through the Tally MCP server. The tools are read-only: nothing here can post, alter or delete a voucher. Every figure you give must come from a tool result in this conversation, never from memory or estimation.

## The tools

Tool names may carry a prefix depending on how the server is connected (for example `...Tally_Prime__trial-balance`). Match on the part after the last `__`.

| Tool | Inputs | Returns |
|---|---|---|
| `list-master` | `collection`, `targetCompany`? | Names: company, ledger, group, stockitem, godown, vouchertype, costcentre, gstin… |
| `chart-of-accounts` | `targetCompany`? | tableID: group_name, group_parent, bs_pl, dr_cr, affects_gross_profit |
| `trial-balance` | `fromDate`, `toDate` | tableID: ledger_name, group_name, opening_balance, net_debit, net_credit, closing_balance |
| `profit-loss` | `fromDate`, `toDate` | tableID: ledger_name, group_name, amount |
| `balance-sheet` | `toDate` | tableID: ledger_name, group_name, closing_balance |
| `ledger-balance` | `ledgerName`, `toDate` | One number |
| `ledger-account` | `ledgerName`, `fromDate`, `toDate` | tableID: date, voucher_type, voucher_number, party_ledger, amount, narration |
| `bills-outstanding` | `nature` (receivable/payable), `toDate` | tableID: bill_date, reference_number, outstanding_amount, party_name, overdue_days |
| `stock-summary` | `fromDate`, `toDate` | tableID: name, parent, opening/inward/outward/closing quantity and value |
| `stock-item-balance` | `itemName`, `toDate` | One number |
| `stock-item-account` | `itemName`, `fromDate`, `toDate` | tableID: item movements with quantity, amount, narration |
| `query-database` | `sql` | Runs DuckDB SQL over the cached tables |

Dates are always `YYYY-MM-DD`. All tools except `query-database` accept an optional `targetCompany`.

### Rules that prevent wrong numbers

1. **Signs: debit is negative, credit is positive** in every amount and balance column (`closing_balance`, `amount`, `outstanding_amount`). So a debtor who owes money shows negative; an asset is negative; income is positive. Flip the sign when presenting, and label every figure Dr or Cr. Stock quantity is the exception: inward positive, outward negative.
2. **Never total or sort by reading rows.** Tabular tools return a `tableID`; do every sum, count, filter, top-N and ageing bucket with `query-database`. Show the SQL result, not mental arithmetic.
3. **Cached tables are dropped after 15 minutes.** If a query says the table doesn't exist, re-run the source tool and use the new tableID. Don't reuse a tableID from an earlier turn without checking.
4. **An empty tableID means Tally returned no rows** (wrong company, wrong dates, or a name typo), not zero balances. Say so and check the inputs.
5. **Validate names before using them.** Run `list-master` with `company` before setting `targetCompany`, and with `ledger` or `stockitem` before passing a ledger or item name. Use the exact spelling Tally returns. Use `ILIKE '%term%'` via `query-database` when the person gives a partial name.
6. **Indian financial year** runs 1 April to 31 March. "This year" means the current FY; "last year" means the previous FY. State the period you used.
7. **Tally must be open with the company loaded.** If a tool errors with a connection failure, tell the person that Tally isn't reachable (Tally closed, XML port 9000 off, PC asleep, or wrong TALLY_HOST) rather than retrying in a loop.

## Getting oriented (start of any session)

1. `list-master` with `company` to see which companies are open. Vianet and Viacom are separate companies; ask which one if the request doesn't say, unless the task covers both (intercompany, group view).
2. Confirm the period. Default to the current FY to date if none is given, and say so.

## Workflows

Pick the one that matches the request. Each ends with a short findings summary; offer an Excel workbook (via the xlsx skill) only when the output is a table the person will keep or send to the CA.

### 1. Final accounts

1. `trial-balance` for the FY. With `query-database`, check that the total of `closing_balance` is zero. If it isn't, stop and report the difference before anything else.
2. `profit-loss` and `balance-sheet` for the same period.
3. Present a P&L and Balance Sheet grouped by `group_name`, in Indian format (₹ with lakh/crore separators), with Dr/Cr shown.
4. Then run the audit flags (workflow 2) on the same tables and append them.

### 2. Audit and reco flags

Run these on the trial balance and drill into ledgers with `ledger-account` where needed. Report each flag with ledger, amount, and why it matters. These are flags for the CA to review, not conclusions.

- **Suspense or unclassified balances**: any ledger whose name or group contains suspense, misc, unidentified, temporary, or clearing, with a non-zero closing balance.
- **Wrong-side balances**: cash or bank with a credit balance (negative cash is impossible; a credit bank balance is fine only for OD/CC accounts), debtors with credit balances (advances or unadjusted receipts), creditors with debit balances (advances or excess payments).
- **Group misclassification**: expense-sounding ledgers under asset or liability groups, and the reverse. Use `chart-of-accounts` to check `bs_pl`.
- **Period-end activity**: in `ledger-account` for material ledgers, entries in the last 3 days of the period that are large relative to the ledger's normal size, journal vouchers posted to sales or purchase ledgers, and entries with an empty narration.
- **Round-sum entries**: journals in exact lakhs with no narration.
- **Cash transactions to check with the CA**: cash payments to one party above ₹10,000 in a day, and cash receipts of ₹2,00,000 or more from one person. Pull the cash ledger with `ledger-account` and aggregate by date and party.
- **Depreciation**: check that a depreciation ledger exists and has entries for the year if fixed assets exist.
- **GST and TDS**: compare output GST minus input GST against GST payment entries, and TDS payable against TDS deposited. A balance carried forward is normal; a growing or unexplained one is a flag.
- **Intercompany**: run workflow 5.

### 3. Receivables ageing and collection list

1. `bills-outstanding` with `nature: receivable` as on today (or the date asked).
2. With `query-database`, bucket by `overdue_days` into 0–30, 31–60, 61–90, 91–180 and 180+, and total by party with the sign flipped so amounts owed show as positive.
3. Present: total receivable, bucket totals, and the top parties by 60+ day overdue amount.
4. If asked for reminders, draft short WhatsApp-style messages per party with the bill references and amounts. Draft only; never send anything.
5. Bills-wise outstanding needs bill-wise details enabled on the party ledgers. If a debtor has a ledger balance but no bills, say that bill-wise tracking is missing for that party.

### 4. Payables

Same as workflow 3 with `nature: payable`. Highlight bills due in the next 7 days and anything 90+ days old, since old payables may be disputed or already paid against the wrong reference.

### 5. Vianet–Viacom intercompany match

1. In each company, find the ledger for the other company with `list-master` (`ledger`) and a name match.
2. `ledger-balance` on each as on the same date.
3. The two should be equal and opposite (Vianet's receivable equals Viacom's payable). Report both figures and the difference.
4. If they differ, pull `ledger-account` on both sides for the period and match entries by amount and date with `query-database` to list the unmatched items on each side.

### 6. Stock checks

1. `stock-summary` for the period.
2. Flag negative closing quantities (sales billed without stock inward, or a godown transfer missing), items with closing stock but no outward movement in the period (slow-moving), and items with a closing value but zero quantity, or the reverse.
3. Compare total closing stock value with the stock figure on the balance sheet for the same date and report any gap.

### 7. Ledger or party deep-dive

For "show me X's account" or "why is X so high": `ledger-account` for the period, then with `query-database` give the opening balance, totals by voucher type, the largest entries, and anything with no narration. Keep the full statement out of the reply unless asked; give it as a sheet if they want it.

## How to present results

- Lead with the answer (the total, the difference, the count of flags), then the supporting table.
- Use ₹ with Indian grouping (₹12,34,567) and say whether each figure is Dr or Cr.
- State the company, the period or as-on date, and that the figures are live from Tally as of now.
- When a check can't be done from the available tools (for example, a bank statement reconciliation needs the bank statement), say what's needed instead of guessing.
