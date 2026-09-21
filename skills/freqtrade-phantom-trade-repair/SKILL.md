---
name: freqtrade-phantom-trade-repair
description: "Surgically repair a corrupted freqtrade open trade whose recorded amount/orders do not match the exchange wallet, without resetting the whole trade history. Use when: (1) The bot loops on 'Insufficient funds to create stop_loss_limit', (2) The bot repeats 'Not enough amount to exit trade' or 'Trying to refind lost order', (3) Trade amount in the DB differs from the real wallet balance, or (4) Open trades carry dry-run order ids or cross-pair orders after a dry-run/live DB merge."
---

# freqtrade Phantom Trade Repair

A live freqtrade bot can crash-loop on an open trade whose database record no longer
matches reality: the recorded `trades.amount` exceeds the wallet balance, and/or its
`orders` rows contain dry-run (`dry_run_...`) or cross-pair orders (e.g. an ENA/USDT
trade holding ADA/USDT orders) left over from a dry-run/live DB merge. The bot then
repeats "Insufficient funds to create stop_loss_limit sell order", "Not enough amount
to exit trade", and endless "Trying to refind lost order" messages, and `forceexit`
crashes on the same phantom amount.

This skill repairs the specific row(s) so the position reflects the real wallet, then
lets the bot manage or exit it normally. It does **not** wipe trade history (unlike a
full operator reset).

## When to use

- The bot logs `Insufficient funds to create stop_loss_limit` or `Not enough amount to exit trade`.
- The bot repeats `Trying to refind lost order ... dry_run_...` against an open trade.
- `GET /api/v1/status` shows an open trade whose `amount` clearly exceeds the wallet holdings.
- Trades were imported/restored from a dry-run DB into a live DB and orders are mismatched.

## Required tools

- `systemctl --user` (user-level systemd service managing the bot)
- Python 3 (or Node.js 22+) with `sqlite3`
- Bot REST API credentials (`api_server.username` / `api_server.password` from config)

## Skills

### diagnose

1. Confirm the failure loop in the journal:

```bash
journalctl --user -u freqtrade.service --no-pager -n 200 | grep -iE "insufficient funds|not enough amount|refind lost order" | tail
```

2. Get the open trade and its recorded amount, plus the bot's own wallet view:

```bash
# open trades (count, then full status)
curl -s -u "$API_USER:$API_PASS" http://127.0.0.1:8080/api/v1/status
# wallet free balance for the held asset
curl -s -u "$API_USER:$API_PASS" http://127.0.0.1:8080/api/v1/balance
```

3. Inspect the trades and orders rows for the corrupted trade id:

```python
# tradesv3.live.sqlite (adjust path to your user_data)
import sqlite3
con = sqlite3.connect("user_data/tradesv3.live.sqlite")
con.row_factory = sqlite3.Row
for t in con.execute("SELECT id, pair, amount, open_rate, open_rate_requested, stake_amount, is_open FROM trades WHERE is_open=1"):
    print(dict(t))
# orders attached to the open trade (look for dry_run ids / wrong pair)
for o in con.execute("SELECT id, ft_trade_id, ft_pair, ft_order_side, order_id, status, filled FROM orders WHERE ft_trade_id IN (SELECT id FROM trades WHERE is_open=1) ORDER BY id"):
    print(dict(o))
```

**Acceptance:** an open trade's `amount` > free wallet holdings for that asset, and/or
`dry_run_*` or cross-pair order ids attached to it. The **real** position = the sums of the
legit closed live `buy` orders on that trade (use `cost/filled` for the true average price).

### repair

Always back up first, then stop the bot so it does not fight the edit:

```bash
systemctl --user stop freqtrade.service
cp user_data/tradesv3.live.sqlite "user_data/tradesv3.live.sqlite.bak-$(date +%Y%m%d_%H%M%S)"
```

Then fix the trade to match the wallet and delete the junk orders:

```python
import sqlite3
con = sqlite3.connect("user_data/tradesv3.live.sqlite")
cur = con.cursor()

trade_id = 87  # from diagnosis
wallet_free = 32.19676   # real free balance of the asset from /api/v1/balance
avg_price = 0.1697       # real average fill = sum(cost)/sum(filled) of legit live buy orders

# remove orders that do not belong (dry-run ids or a different ft_pair than the trade)
cur.execute("""DELETE FROM orders
               WHERE ft_trade_id=? AND (order_id LIKE 'dry_run%' OR ft_pair != (SELECT pair FROM trades WHERE id=?))""",
            (trade_id, trade_id))

# align the open trade with the wallet
cur.execute("""UPDATE trades SET
                 amount=?, open_rate=?, open_rate_requested=?,
                 open_trade_value=?, stake_amount=?, max_stake_amount=?
               WHERE id=?""",
            (wallet_free, avg_price, avg_price,
             round(wallet_free * avg_price, 8),
             round(wallet_free * avg_price * 1.001, 8),  # + ~0.1% taker fee
             round(wallet_free * avg_price * 1.001, 8),
             trade_id))
con.commit()

# sanity checks
print("open trades:", con.execute("SELECT COUNT(*) FROM trades WHERE is_open=1").fetchone()[0])
print("open orders:", con.execute("SELECT COUNT(*) FROM orders WHERE ft_is_open=1").fetchone()[0])
```

Restart and watch it converge:

```bash
systemctl --user start freqtrade.service
sleep 25
journalctl --user -u freqtrade.service --no-pager -n 30 | grep -iE "insufficient|refind|stop[- ]loss|exit|sell" | tail
curl -s -u "$API_USER:$API_PASS" http://127.0.0.1:8080/api/v1/count
```

**Acceptance:** no more "insufficient funds"/"refind" loops; the corrected trade either
re-enters normal management (fresh stop-loss placed on the real amount) or exits cleanly
(the recorded `exit_reason` at market, profit computed from the true position). Wallet and
`bot_owned` now agree.

### verify

```bash
# trade closed cleanly?
curl -s -u "$API_USER:$API_PASS" http://127.0.0.1:8080/api/v1/status   # open trades as expected
curl -s -u "$API_USER:$API_PASS" http://127.0.0.1:8080/api/v1/balance  # held asset ≈ 0 after exit
```

## Output format

- Pre-repair diagnostic: trade id, pair, recorded vs real amount, junk order ids removed.
- Post-repair state: open-trade count, exit reason/profit if it exited, wallet vs bot_owned match.

## Best practices

- Stop the bot before any DB write; restart only after commit succeeds.
- Always keep a timestamped `.sqlite` backup; restore it if the bot misbehaves.
- Prefer a surgical row fix over a full operator reset when history matters.
- If multiple trades are corrupt, fix them one at a time and verify after each restart.
- The real average entry price is `sum(cost)/sum(filled)` over **live** buy orders only — dry-run rows must be excluded, not averaged in.

## Troubleshooting

**Error: `sqlite3.OperationalError: no such column: trade_id`**
- Symptom: the orders query fails.
- Solution: the column is `ft_trade_id`, not `trade_id` (check with `PRAGMA table_info(orders)`).

**Error: bot still loops "Insufficient funds" after the fix**
- Symptom: amount mis-set or extra junk orders remain.
- Solution: set `amount` to the current free wallet balance (≤ wallet), not the requested amount, and delete **all** `dry_run%`/cross-pair orders on the trade.

**Error: trade exits at a bad price right after restart**
- Symptom: the corrected `open_rate` is below market, so the stored stop-loss fires immediately.
- Solution: expected. The phantom never really existed; exiting at market frees the wallet and is required to reach a consistent state. Understate any panic.

## Agent prompt

```text
You have freqtrade-phantom-trade-repair capability. When the bot loops on "Insufficient funds
to create stop_loss_limit", "Not enough amount to exit trade", or "Trying to refind lost order":

1. Diagnose via journalctl, /api/v1/status, /api/v1/balance, and the sqlite DB (orders use ft_trade_id).
2. Compute the real position from live buy orders: sum(filled) and avg = sum(cost)/sum(filled).
3. Stop the service, back up the DB, delete dry_run%/cross-pair orders, UPDATE the trade row to wallet reality.
4. Restart, verify no loops, and report the trade's close if it exits.
Never edit the DB without stopping the bot and keeping a backup.
```

## See also

- [../freqtrade-self-heal/SKILL.md](../freqtrade-self-heal/SKILL.md) — broader self-heal/self-learn/self-improve layer (rewind, whipsaw, RED safeguards)