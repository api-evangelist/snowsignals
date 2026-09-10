---
name: snowsignals-manage-prepaid-balance
description: >-
  Keep a SnowSignals account fundable and observable: check the prepaid stablecoin balance and
  holds, review the paid-call usage log, nudge deposit detection after sending funds, and handle
  the billing statuses (402 out of credits, 423 refund settling) an agent will meet mid-flow.
api: SnowSignals daas API (https://snowsignals.io/v1) / MCP https://snowsignals.io/mcp
operations:
- GET /user/balance       # getBalance (overlay id) / MCP get_balance — unmetered, account-bound
- GET /user/usage         # getUsage / MCP get_usage — unmetered, id-cursor paginated via `before`
- POST /user/deposit-now  # depositPoll / MCP deposit_poll — unmetered, moves no money
- GET /api/notifications  # listNotifications / MCP get_notifications — unmetered, cursor paginated
generated: '2026-09-10'
method: generated
source: openapi/snowsignals-daas-openapi.json + live MCP tools/list + llms/snowsignals-llms.txt
---

# Manage the prepaid balance

All metered reads debit a prepaid balance funded by stablecoin deposits (e.g. USDT/USDC; the
deposit addresses on the account are listed by the balance call). These account operations are
free; they require the account credential (REST apiKey or MCP OAuth `daas:read`).

## Steps

1. **Check funds before a paid flow.** `GET /user/balance` — spendable/refundable balance, holds,
   bonus credit + expiry, pause state, current pricing, and deposit addresses. Compare spendable
   micro-USD against the priced cost of the intended read (pricing model comes free from
   `GET /api/phases`).
2. **Top up when short.** Adding a deposit ADDRESS is dashboard-only (sign in on the web — not
   available over the API). After sending a deposit to an existing address, call
   `POST /user/deposit-now` so a lagging hot wallet notices it sooner; it moves no money, and an
   empty result means the account has no deposit address yet.
3. **Audit spend.** `GET /user/usage` — paid calls newest first (tool, currencies, timeframes,
   rows, debit). Page back with `before` using the cursor from the previous page. Reconcile
   against the `meta.debitMicroUsd` you observed per response.
4. **Watch account messages.** `GET /api/notifications` — support and billing notices, 10 newest
   per category, per-category cursors. Read-only; never marks anything read. `category=security`
   is never served over the API, and a category disabled for API access fails loud with 400
   rather than returning an empty page.

## Rules

- `402` on a metered read = insufficient balance, nothing was served or billed — fund the account
  (step 2), then retry.
- `423` = a refund is settling and spend is paused; wait and retry rather than re-funding.
- Deposits are the ONLY money-in path; there is no card/checkout API surface. Never invent or
  reuse deposit addresses — always read them from `GET /user/balance`.
- Balance reads are unmetered: poll balance, never poll metered phase reads, to watch funds.
