---
name: freqtrade-self-heal
description: Deploy and operate a self-healing, self-learning, self-improving layer on top of a live freqtrade bot. Use when a trading bot should recover from downtime/mode-drift, remember recurring loss patterns (signatures of pair + enter_tag + exit_reason), and automatically mitigate mistakes when they reappear (runtime pair blacklisting, whipsaw and RED-mode safeguards, milestone trade limits with human assessment checkpoints).
---

# freqtrade Self-Heal / Self-Learn / Self-Improve Controller

Weaponized recall for a live freqtrade deployment: keep the bot alive and honest,
remember its mistakes, and act before they compound.

## When to Use

- The operator wants a live trial capped at N trades with an assessment checkpoint at M < N.
- The bot must not silently run in the wrong mode (e.g., dry-run when live was requested).
- Losing patterns (same pair + signal + exit reason) should be detected and mitigated automatically.
- The bot must self-restart when its REST API stops responding.

## Architecture

Three layers, all stdlib-only (sqlite + urllib), run every 5 minutes from cron or a
systemd timer — verify which is actually scheduled before relying on any of it
(`crontab -l | grep -iE 'heal|trader_heal'`; `systemctl --user list-timers 'freqtrade*'`).
In the field the heal cron was absent while a monitor timer ran a different script,
so the self-heal layer was silently offline.

0. **DISCOVER THE LIVE UNIT FIRST** (`systemctl --user list-units 'freqtrade*'`).
   The live bot may run as `freqtrade-ultimate.service` while a legacy
   `freqtrade.service` sits failed-but-enabled. Every layer below must name the
   LIVE unit — otherwise self-heal revives the wrong service or nothing at all.
   Audit all restart paths at once: `grep -rn 'restart freqtrade' -- *.sh *.py`
   (ensure_bot, watchdog, netwatch, trader_heal each hardcode a unit name).

1. **SELF-HEAL** — health probe the REST API; restart the live systemd user service if dead.
   Guard rails against mode drift: `runmode` must be `live` and `dry_run=False`; alert
   (or auto-fix + restart when `AUTO_FIX=1`) otherwise. Detect phantom open trades
   (amount == 0, never filled) that crash force-exit. Any monitor must watch the LIVE
   unit **and** exit non-zero on failure — a field monitor checked the legacy unit
   (CRITICAL every cycle while the real bot was healthy) and ended with `exit $EOF`
   (a heredoc leftover), so it always exited 0 anyway.

2. **SELF-LEARN** — load closed trades from the live DB, summarize every losing trade by
   signature `"<pair>|<enter_tag>|<exit_reason>"`, and persist win/loss counts, net P&L
   and first/last-seen in `lessons.json`. Whipsaw detector: >= N losses on one pair inside
   24h is captured as a `WHIPSAW` lesson. Scope lessons to a trial baseline
   (`user_data/.milestone/baseline`) so historical trades don't count against a fresh trial.

3. **SELF-IMPROVE** — when a learned loser signature recurs (>= `REPEAT_THRESHOLD` losses):
   blacklist the pair at runtime via `POST /api/v1/blacklist` (`{"blacklist": [...]}` —
   this version's only runtime config lever). Whipsaw pairs get a 24h blacklist, recurring
   patterns 7 days, and RED mode (>= N trades at a net P&L below threshold) blacklists the
   worst pairs until an operator resets. Expired mitigations are lifted via
   `POST /api/v1/reload_config` (the proven reset, also clears `DELETE`-resistant state).

## Runtime Facts Authenticated in the Field

- `POST /api/v1/blacklist` with body key `blacklist` (not `pairlist`) works on `StaticPairList`.
- `DELETE /api/v1/blacklist` did **not** remove entries in the tested build; `reload_config` does.
- This build has **no** `change_config` endpoint (PUT/POST/PATCH all 405) — blacklist + reload
  + forceexit are the available runtime levers.
- **`reload_config` does NOT re-read config file *values*** (field-verified): edit
  `stake_amount`, `POST /api/v1/reload_config` → 200 "Reloading config ...", but
  `/api/v1/show_config` still reports the old value. It is proven only for clearing
  blacklist/resolver state. Config value changes apply on `systemctl --user restart
  <live-unit>`; a restart is safe while exchange-side `stoploss_on_exchange` orders
  protect open positions (verified: the stop survived and the open trade resumed
  with live P&L).
- freqtrade has no built-in "stop after N total trades"; implement it with a milestone monitor
  (count closed trades since a stored baseline, alert at M, `systemctl --user stop` at N)
  and patch the auto-respawner cron to a guard that refuses to relaunch once halted.
- **2026-09-21 addition — the dry-flip desync trap (largest single loss class):**
  never flip a live bot to `dry_run` while open positions exist. `dry_run` loads a
  *different* sqlite, so live positions "disappear" from the bot while exchange-side
  stop/sell orders keep consuming the wallet → on return the bot emergency-exits a
  0-balance position forever (`Filter failure: NOTIONAL`, "Not enough X in wallet").
  Guards: (a) network watchdog refuses the live→dry flip when `/status` reports >0 open
  trades (exchange-side `stoploss_on_exchange` orders protect positions offline), and
  (b) a wallet-reconcile probe in HEAL compares each open trade's `amount` vs the real
  exchange `total` balance (ccxt from the bot venv) and alerts on `wallet < 90% of DB`
  instead of blind market-exit. Read-only; the operator reconciles.
- **Blacklist `BNB/<stake>` and stablecoin stables** on Binance spot: if you hold BNB for
  the fee-discount, a BNB/USDT trade can consume the fee asset and become unsellable.
- **Schema gotcha:** freqtrade's config schema rejects an empty string for
  `api_server.jwt_secret_key` (minLength 32) *at validation time*, before env
  substitution. Keep a non-secret >=32-char placeholder in the config and make the
  config string reference whichever env name the deployment actually defines —
  field config used `'${FREQTRADE__API_SERVER__JWT_SECRET}'` (plain `${VAR}`
  interpolation from `EnvironmentFile=.env`), not the `FREQTRADE__API_SERVER__JWT_SECRET_KEY`
  path-override name. Match the config's reference, don't assume the convention.
- Telegram env names vary the same way: the field `.env` defined
  `FREQTRADE__TELEGRAM__TOKEN`/`FREQTRADE__TELEGRAM__CHAT_ID` (freqtrade section
  overrides), not bare `TELEGRAM_TOKEN`/`TELEGRAM_CHAT_ID`. Parse key *names* out of
  `.env` first and use what exists; never print values.
- Pair blacklisting + reload_config also works with `VolumePairList` chains (the resolver
  logs "Pair X in your blacklist. Removing it from whitelist").
- **Strategy sweep before you trust "the best"**: the strategy the operator *believes* is best
  often isn't. A 90-day head-to-head sweep (same `config.json` + StaticPairList overlay +
  `--timerange <90d ago>-`, one `freqtrade backtesting` per strategy) showed the "live" pick
  (ResearchStrategyV5) at **-95% drawdown / 29% win rate**, while ResearchStrategyV8 was
  the best of all-time runs (-2.77 USDT / 357 trades / 68% win / -11% DD). Rank by
  `Tot Profit USDT` + `Drawdown%`, not by age or name. To make a deliberate dry-run stick,
  disable `freqtrade-netwatch.timer` (it auto-flips dry→live on the next good ping) and
  comment the heal cron (`AUTO_FIX=0` means heal only *alerts* WRONG MODE every 5 min);
  `FREQTRADE__DRY_RUN` + separate dry DB is governed by the `.net_mode` file at restart.

## Credentials / Notification

Read Telegram `token`/`chat_id` from env first — key names vary by deployment
(`FREQTRADE__TELEGRAM__TOKEN`/`FREQTRADE__TELEGRAM__CHAT_ID` in the field, sometimes
bare `TELEGRAM_TOKEN`/`TELEGRAM_CHAT_ID`; list `grep -oE '^[A-Z0-9_]+=' .env` to see
what exists), or fall back to the bot's own config `telegram` section (the env-file
token can be stale and return HTTP 401). Never hardcode the token in scripts (the
hardcoded token in a 2-line monitor was the leak pattern found in the field);
centralize in the bot's `.env` (mode 600, git-ignored) and resolve via env or
`.env` parse.

## Clean Halt (stop that actually stays stopped)

A plain `systemctl --user stop <unit>` **loses the restart war**: the
self-heal cron/timer, the health watchdog timer, netwatch, and the respawner all
bring it back within minutes (observed: watchdog relaunched it within ~90s) —
*if they name that unit*. Stopping "for real" means finding and cutting every
restart path first:

```bash
# 0) inventory everything that can revive a bot:
systemctl --user list-units 'freqtrade*' --all
systemctl --user list-timers 'freqtrade*'
crontab -l | grep -iE 'freqtrade|heal|ensure'
grep -rn 'systemctl.*freqtrade\|restart freqtrade' -- *.sh *.py 2>/dev/null

# 1) disable the timers that are ACTIVE (list varies by deployment; the field
#    had watchdog/netwatch already disabled while monitor+fundswatch ran):
systemctl --user disable --now freqtrade-monitor.timer freqtrade-fundswatch.timer
systemctl --user disable --now freqtrade-watchdog.timer freqtrade-netwatch.timer 2>/dev/null || true

# 2) cut cron paths (heal() restarts on dead /api):
crontab -l | sed 's|^\(\*/5 .*/trader_heal.py.*\)|#\1|' | crontab -

# 3) ensure_bot.sh will not relaunch:
echo end_20 >> user_data/.milestone/markers   # path relative to the bot dir

# 4) stop the LIVE unit (the one list-units showed as running):
systemctl --user stop freqtrade-ultimate.service

# 5) bracket-trick: plain pkill -f matches its OWN cmdline and kills the shell
pkill -f 'freqtrade [t]rade --config'
```

If step 0 finds a restart path still naming a *different* unit, retarget or disable
it too — otherwise you lose the restart war, or you "win" by stopping a unit nothing
was watching while the real one keeps trading. Verify no process remains
(`pgrep -fc 'freqtrade [t]rade --config'`) and that it is still down after one full
timer/cron cycle before telling the operator. To resume, invert every step (remove
the marker, uncomment cron, re-enable timers), then start the LIVE unit.

## Operator Reset (resume after RED/halt)

```bash
rm -f user_data/.milestone/baseline user_data/.milestone/ingested_close
systemctl --user start <live-unit>   # the unit list-units showed as running, e.g. freqtrade-ultimate
```

This clears the trial quota and any pending lessons; pair blacklists added by the
controller are in-memory and vanish on restart.