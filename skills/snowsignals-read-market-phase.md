---
name: snowsignals-read-market-phase
description: >-
  Read the current crypto market phase (regime) for one or more currencies and timeframes from
  SnowSignals TrendVane, price the call before spending, and interpret the reading with the free
  phase-resolution model. Works over REST (apiKey) or the hosted MCP server (OAuth daas:read).
api: SnowSignals daas API (https://snowsignals.io/v1) / MCP https://snowsignals.io/mcp
operations:
- GET /api/phases            # listPhaseMeta (overlay id) / MCP list_phase_meta — free, no auth
- GET /api/phase/resolution-stats  # getPhaseResolutionStats / MCP phase_resolution_stats — free, no auth
- GET /user/balance          # getBalance / MCP get_balance — account-bound, unmetered
- GET /api/phase/boundary    # getPhaseBoundary / MCP get_phase(kind=boundary) — METERED
- GET /api/phase/updates     # getPhaseUpdates / MCP get_phase(kind=updates), compose_phases — METERED
generated: '2026-09-10'
method: generated
source: openapi/snowsignals-daas-openapi.json + live MCP tools/list + llms/snowsignals-llms.txt
---

# Read a market phase, priced before you spend

TrendVane reports market STATE, not trade signals: which of 12 phases (6 shapes x bull/bear) a
currency is in on a timeframe. The published spec has no operationIds — operations are named
METHOD + path (overlay ids in comments above).

## Steps

1. **Discover, free.** `GET /api/phases` (no auth) — the enabled currencies (BTC, ETH, GRAM, SOL,
   TRX), timeframes (15m, 1h, 2h, 4h, 1d, 1w), the 12-phase enum with a card per phase, AND the
   live pricing model. Over MCP this is `list_phase_meta`. Never hardcode currencies/tfs; read
   them here.
2. **Price the call before making it.** rows = |currencies| x |timeframes|;
   `debit_micro_usd = round(rows x base_rate x mult(rows))` with mult 1.25 (1 row), 1.15 (2-5),
   1.00 (6+). Batching is cheaper per row. Optionally confirm affordability with
   `GET /user/balance` (unmetered).
3. **Read the phase (metered).** `GET /api/phase/boundary?currency=BTC&tf=1h,4h` for the settled
   last-closed-bar truth, or `/api/phase/updates` for the intra-bar read. Comma-lists or `all`
   on both params. Over MCP: `get_phase` (one currency) or `compose_phases` (many). If the
   response carries a `stale` block, the reading may still reflect the previous minute — poll
   again after `stale.pollAfterMs`.
4. **Interpret, free.** `GET /api/phase/resolution-stats` — successor-phase transition
   probabilities, trend-continuation rates, and MFE/MAE per phase, each with its sample count
   (BTC research model, 2022-2026). Pair it with the live reading; it takes no parameters.

## Rules

- REST auth is one of two mutually exclusive per-account apiKey methods (`?apiKey=` query, or the
  nonce Authorization header). On the nonce method, only ONE request may be in flight per key and
  the nonce must strictly increase — sync your clock with the free `GET /api/time` first.
- Errors: `400` invalid currency/tf (re-read step 1), `402` out of credits (no partial serve —
  top up, see the balance skill), `423` refund settling (wait), `429` honor `Retry-After`.
- There is NO idempotency/replay protection on spend: a retried metered read is billed again.
  Retry only on 5xx/timeouts you can afford, and check `meta.debitMicroUsd` on every response.
- `label` is display-only; key logic on `phase`.
