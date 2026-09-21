# Skill: freqtrade-bot-audit

Deep-audit a live freqtrade deployment and produce an evidence-based, source-linked
improvement plan. Use when the operator wants to know why a bot is (not) making
money and what to change next — without blind config guesswork.

## When to Use

- "Why is my freqtrade bot losing money?" / "make my bot trade more efficiently".
- Before changing pairs, stake, stoploss, ROI, or strategy on a live bot.
- Before adopting any community strategy / GitHub idea.

## Workflow (always in this order)

### 1. Inspect the deployment (read, don't change anything)

Inventory the bot directory:

- `config.json` (the active one under `user_data/`) — copy sanitized values, never
  secrets. Note: `stake_amount`, `max_open_trades`, `pairlists`, `pair_whitelist`,
  `strategy`, `timeframe`, `minimal_roi`, `stoploss`, `trailing_*`, DCA settings,
  `order_types` (esp. `stoploss_on_exchange`), `cancel_open_orders_on_exit`,
  `unfilledtimeout`.
- The active strategy in `user_data/strategies/` — read entry/exit conditions,
  `custom_stoploss`, `adjust_trade_position`, protections, `informative_pairs`.
- Ops/guard scripts next to the bot dir (heal controllers, respawners, milestone
  monitors, network watchdogs, backup). Note every place a credential appears.
- Git log/status for what changed and when.

Then measure reality from the trade DB (`user_data/tradesv3*.sqlite`):

```sql
-- shape of the whole trial
SELECT count(*),
       sum(case when is_open=0 then 1 else 0 end),
       round(sum(realized_profit),2), max(close_date) FROM trades;
-- where money is lost (exit_reason drill-down + worst pairs + enter tags)
SELECT exit_reason, count(*), round(sum(realized_profit),2)
  FROM trades WHERE is_open=0 GROUP BY exit_reason ORDER BY 3;
SELECT pair, count(*), round(sum(realized_profit),2)
  FROM trades WHERE is_open=0 GROUP BY pair ORDER BY 3 ASC LIMIT 8;
SELECT enter_tag, count(*), round(sum(realized_profit),2)
  FROM trades GROUP BY enter_tag ORDER BY 3 DESC LIMIT 12;
```

Always read the service journal around loss clusters — root causes (NOTIONAL filter
failures, "Not enough X in wallet", emergency-exit loops, reconciliation desyncs)
live in logs, not in the strategy:

```bash
journalctl --user -u freqtrade.service --since "<mm-dd HH:MM>" | grep -iE "emergency|stoploss|NOTIONAL|wallet|amount|ERROR"
```

### 2. Classify the losses before touching the strategy

Rank root causes, not symptoms. The usual killers, in order of frequency:

1. **Reconciliation / restart desync** — restarting while `stoploss_on_exchange`
   orders already executed, or a network watchdog flipping live↔dry_run with open
   positions (dry uses a different DB → the bot "forgets" positions). Shows up as
   `emergency_exit`, `force_exit_dryrun_switch`, "Not enough <coin> in wallet".
2. **Stake below exchange minimum notional** — flat small stakes + stoploss reserve
   falling under Binance MIN_NOTIONAL (5 USDT) → `Filter failure: NOTIONAL` and
   unstoppable emergency-exit loops. Check
   `max(trade.stake_amount*|stoploss|, withdrawals) > exchange min notional`.
3. **Coin quality** — dust coins and 24h-pump meme pairs in a static whitelist with
   **no pairlist filters** (the biggest single differentiator).
4. **Fee drag / DCA concentration** — ROI buckets below round-trip fees, DCA that
   doubles concentration on a tiny wallet.
5. Strategy signal quality (last resort, only after 1–4 are addressed).

### 3. Research improvements from real sources (not vibes)

- Pull current freqtrade release notes (GitHub `freqtrade/freqtrade/releases`) and
  check the local install: `freqtrade --version`, `freqtrade --help | grep -i edge`,
  and the installed pairlist inventory
  (`freqtrade/plugins/pairlist/*.py` — verify `PrecisionFilter`, `PriceFilter`,
  `VolumePairList`, `OffsetFilter`, `RangeStabilityFilter`, `FullTradesFilter`,
  `PerformanceFilter`, `DelistFilter`, `PairInformationFilter`).
- Verify every config snippet against `docs.freqtrade.io` or the local plugin
  source. Parameter names must be exact; mark anything unverifiable `UNVERIFIED`.
- Scrape GitHub for strategies/techniques that attack the *classified* root causes
  (whipsaw filters for hot coins, exchange-side stop persistence on spot,
  reconciliation-after-restart, dynamic stake for small accounts).
- Check pairlist + order-type + protection semantics against the bot's actual
  exchange mode (spot vs futures) — many filters/order types differ.

### 4. Prioritize by repair-cost, not profit-optimism

Order deliverables:

1. Ops fixes that stop the bleeding (network-watchdog flip policy, wallet reconcile,
   exchange-side stops, `unfilledtimeout`, `cancel_open_orders_on_exit`).
2. Config-level wins (pairlist filters, blacklist `BNB/<stake>`, dynamic stake).
3. Strategy-level changes (ATR-stop grace period, momentum-stall exits, regime
   filters, protections sized to the wallet).
4. Research/validation discipline (gate on backtest **mean-profit p-value** and
   Expectancy, judge by out-of-sample walk-forward windows — never headline PnL).

Ship one change → one backtest → confirm. On a tiny account the PnL error bar is
±8–10 percentage points per ~100 trades; most parameter "improvements" are noise.

## Rules

- Never paste keys/tokens/chat-ids into any shared artifact; note plainly where
  credentials live in the deployment and if any are plaintext.
- Read before editing; never change the live bot without an explicit operator go.
- Report findings as: inspection facts → root-cause ranking → source-linked plan.