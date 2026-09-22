# Skill: freqtrade-nightly-backtest

Wire a nightly regression backtest into a live freqtrade deployment so it tests the
strategy that is actually trading, on the deployment's own data, and keeps a rolling
window. Use when: a scheduled backtest service/timer keeps failing, backtests run
against the wrong user-data or an empty data store, dynamic pairlists break
backtesting, or the operator wants a daily "does production still hold up" check.

## When to Use

- A daily backtest systemd unit/timer exits with `status=2` or errors like
  "Pairlist Handlers X do not support backtesting", "Impossible to load Strategy Y",
  or "Starting balance 0 is smaller than stake_amount".
- The backtest uses a different freqtrade install / user-data dir than the live bot,
  so it can't even see the bot's candles.
- You want a nightly run that re-downloads data and backtests a rolling recent window
  instead of a fixed (and eventually data-less) historical timerange.

## Field-Proven Facts

- Dynamic pairlists (`VolumePairList`, `AgeFilter`, `RangeStabilityFilter`,
  `DelistFilter`) are **not supported in backtesting** — freqtrade aborts with
  "do not support backtesting". Fix: an overlay config that sets
  `{"pairlists": [{"method": "StaticPairList"}]}` and pass BOTH configs to
  `download-data` AND `backtesting` so the same static whitelist feeds both steps.
  `StaticPairList` uses the whitelist in the main config, so keep one source of truth.
- Always run the backtest with the **live bot's own venv and `--userdir`**. The pipx/
  default install usually resolves a different `~/user_data` with no candles; data and
  results should live where the bot keeps them.
- The strategy's `informative_timeframe` / `informative_pairs` data must be downloaded
  too (e.g. 3m core + 15m MTF + BTC 1h regime), or those indicators silently fall back
  / merge empty.
- A rolling window beats a fixed timerange: download `--days N+` then backtest
  `--timerange <N days ago>-` (open end = "to now"). Compute the start date in the
  wrapper script so the window tracks today.
- `systemctl --user start <oneshot.service>` blocks until the unit finishes (systemd
  waits for the job). For a long download+backtest, either accept blocking in a big
  Bash timeout or use `--no-block`.
- Result artifacts land in `userdata/backtest_results/` as
  `backtest-result-YYYY-MM-DD_HH-MM-SS.zip` (+ `.meta.json`). Presence of a fresh one
  dated "now" is the success signal.

## Workflow

### 1. Diagnose (what is actually failing)

```bash
systemctl --user status freqtrade-backtest.service --no-pager   # exit-code?
journalctl --user -u freqtrade-backtest.service -n 60 --no-pager
```

Categorise the failure: strategy not found (bad `--strategy-path` / strategy file
moved/renamed), wrong user-data (no candles), unsupported dynamic pairlists, or a
timerange with no downloaded data.

### 2. Build the wrapper script (rolling window, deploy's venv + userdata)

Generic shape (replace `<BOT_DIR>` / `<USERDATA>` / strategy names):

```bash
#!/usr/bin/env bash
set -u
VENV=<BOT_DIR>/.venv
USERDATA=<BOT_DIR>/user_data
CFG="$USERDATA/config.json"
OVERLAY="$USERDATA/config.backtest.live.json"     # {"pairlists":[{"method":"StaticPairList"}]}
STRATEGY=YourLiveStrategy
DL_DAYS=100
BT_DAYS=90
START=$(date -u -d "$BT_DAYS days ago" +%Y%m%d)
CFG_ARGS=(--config "$CFG" --config "$OVERLAY")
"$VENV/bin/freqtrade" download-data "${CFG_ARGS[@]}" --userdir "$USERDATA" \
    --timeframes 3m 15m 1h --days "$DL_DAYS" \
    || echo "download-data failed; continuing with cached data"
exec "$VENV/bin/freqtrade" backtesting "${CFG_ARGS[@]}" --userdir "$USERDATA" \
    --strategy "$STRATEGY" --timerange "${START}-"
```

### 3. Point the systemd unit at the script

```ini
[Unit]
Description=Daily Freqtrade back-test (<Strategy>, rolling <N>d)

[Service]
Type=oneshot
ExecStart=/path/to/daily_backtest.sh

[TIMER]
# in freqtrade-backtest.timer
OnCalendar=*-*-* 02:00
Persistent=true
```

`chmod +x` the script, then:
```bash
systemctl --user daemon-reload
systemctl --user reset-failed freqtrade-backtest.service
systemctl --user start freqtrade-backtest.service   # or --no-block
systemctl --user status freqtrade-backtest.service --no-pager   # expect exit-code 0
```

### 4. Verify honestly

- Exit status 0 and `Finished` in the journal.
- A fresh `user_data/backtest_results/backtest-result-*.zip` from this run.
- Read the STRATEGY SUMMARY row — a green pipeline can still print a red strategy
  (that's the point of a regression check). Report the numbers as-is.
- Confirm the timer is armed: `systemctl --user list-timers | grep backtest`.

## Rules

- Never point the scheduled job at a fixed historical timerange unless data for it is
  actually downloaded every run; a rolling `start-` window self-maintains.
- Don't pass the live config to backtesting without the StaticPairList overlay unless
  the live config already uses only static pairlists.
- Keep secrets out: the backtest config/script files need no API keys (downloads run
  from the deployment's own credentials).
- Do not modify the live bot's `config.json` or the running service to make backtests
  pass — use overlay configs and separate wrapper scripts.