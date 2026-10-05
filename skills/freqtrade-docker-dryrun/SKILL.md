---
name: freqtrade-docker-dryrun
description: "Deploy or rebuild a Freqtrade bot as a clean Docker Compose service running Binance spot dry-run with secrets injected from a .env file. Use when: (1) Setting up Freqtrade for the first time, (2) Rebuilding a bot after deleting or migrating an old install, (3) Fixing Freqtrade config validation errors about jwt_secret_key, missing config, or env vars not being substituted, (4) Moving a bot from a manual/systemd run into Docker."
---

# Freqtrade Docker Dry-Run Deployment

Deploy Freqtrade as a clean Docker Compose service in Binance spot **dry-run** mode, with
every secret injected from a `.env` file and never written into `config.json`. The outcome is
a running container, reachable local REST API, and a verified simulated balance.

## Quick quality checklist

- [x] Secrets exist only in `.env` (mode `600`), never in `config.json` or the shell history
- [x] `config.json` passes schema validation with `dry_run: true`
- [x] No `${VAR}` placeholders in `config.json` — Freqtrade does not interpolate them
- [x] Docker port bind is loopback-only (`127.0.0.1:<hostport>:8080`)
- [x] Live trading is never enabled without explicit human confirmation

## When to use

- Use case 1: The user wants Freqtrade running and has no installation yet.
- Use case 2: An old installation must be deleted and rebuilt cleanly.
- Use case 3: `Configuration error: '...jwt_secret_key...' is too short` or `... is too short` errors.
- Use case 4: `Error: Config file ... not found` inside the container even though it exists on the host.
- Use case 5: Moving an existing bot from a manual or systemd launch into Docker Compose.

## Required tools / APIs

- **Docker Engine** — runs the bot container.
- **Docker Compose v2** — defines and runs the service.
- **jq** — parses API responses without printing secrets.

```bash
# Ubuntu/Debian
sudo apt-get install -y docker.io docker-compose-v2 jq

# macOS
brew install docker docker-compose jq
```

## Skills

### critical_finding_env_vars_do_not_interpolate

This is the single most common failure and the reason most guides do not work. Freqtrade
**does not expand `${VAR}` strings in `config.json`**. The config file is validated against a
JSON schema *before* anything else, so a placeholder like `"${FREQTRADE_JWT_SECRET}"` is
treated as a literal 21-character string and rejected.

Injection happens only through the `FREQTRADE__SECTION__KEY` convention, merged into the loaded
config file:

```bash
# Docker image path of the loader
docker run --rm --entrypoint sh freqtradeorg/freqtrade:stable -c "cat /freqtrade/freqtrade/configuration/environment_vars.py"
```

The merge is `deep_merge_dicts(env_data, config)` — the **environment wins**. But schema
validation runs on the fully merged document, so `config.json` still has to be schema-valid on
its own: use inert placeholders in the config file and let the environment override them.

### verify_variable_names_do_not_assume_them

A variable whose name does not match a real config key is accepted **silently**. It becomes a
brand-new config key that the application never reads, and the placeholder in `config.json`
stays live. Nothing warns you. Always prove each name overrides what you expect:

```bash
docker run --rm \
  -e FREQTRADE__API_SERVER__JWT_SECRET_KEY=PROBE_VALUE_THAT_IS_AT_LEAST_32_CHARS \
  -e FREQTRADE__API_SERVER__USERNAME=PROBE_USER \
  -v ./config.json:/c.json:ro \
  freqtradeorg/freqtrade:stable show-config -c /c.json 2>/dev/null \
  | grep -E 'jwt_secret_key|username'
# expect: the PROBE values to appear. Anything else means the name is wrong.
```

`show-config` redacts exchange and Telegram secrets, so for those either confirm no phantom key
appears in the output, or rely on a successful `dry_run: true` startup using the real values.

The single easiest mistake to make:

| Wrong | Right | Symptom of the wrong one |
| --- | --- | --- |
| `FREQTRADE__API_SERVER__JWT_SECRET` | `FREQTRADE__API_SERVER__JWT_SECRET_KEY` | a phantom `jwt_secret` key appears in `show-config` while `jwt_secret_key` still holds the placeholder |

The wrong name is not an error. It leaves the bot authenticating its REST API with a
placeholder that is sitting in plaintext in the repository.

Canonical names for the fields that matter:

| Purpose | `.env` variable | Placeholder in `config.json` |
| --- | --- | --- |
| Exchange key | `FREQTRADE__EXCHANGE__API_KEY` | `"replace_me"` |
| Exchange secret | `FREQTRADE__EXCHANGE__SECRET` | `"replace_me"` |
| API username | `FREQTRADE__API_SERVER__USERNAME` | `"replace_me"` |
| API password | `FREQTRADE__API_SERVER__PASSWORD` | `"replace_me"` |
| API JWT secret | `FREQTRADE__API_SERVER__JWT_SECRET_KEY` | 32+ inert chars |
| Telegram bot token | `FREQTRADE__TELEGRAM__TOKEN` | `"000000:replace_me"` |
| Telegram chat id | `FREQTRADE__TELEGRAM__CHAT_ID` | `"0"` |

Three rules that follow from the merge order:

1. `api_server.jwt_secret_key` has a schema minimum of **32 characters**. The placeholder
   itself must clear that bar, even though the real value arrives from the environment.
2. Never put a section *key* in the environment that you have removed from `config.json`.
   An environment section replaces the config section for keys only it defines.
3. `docker compose` interpolates `${...}` in `compose.yaml` itself. If you use `env_file`,
   reference variables as `$VARNAME` only in `compose.yaml`, never as `${VARNAME}` pointing
   at a name you also expect Freqtrade to read.
4. A value shorter than the schema minimum fails validation even if the real value would be
   long enough. `jwt_secret_key` must be >= 32 characters in the merged document.

### basic_usage

```bash
# 1. Directory layout (config.json at repo root, user_data mounted separately)
mkdir -p ~/freqtrade-binance/user_data/strategies
chmod 700 ~/freqtrade-binance

# 2. .env, mode 600, one secret per line, no quotes, no 'export'
cat > ~/freqtrade-binance/.env <<'EOF'
FREQTRADE__EXCHANGE__API_KEY=replace_with_real_key
FREQTRADE__EXCHANGE__SECRET=replace_with_real_secret
FREQTRADE__API_SERVER__USERNAME=ftadmin
FREQTRADE__API_SERVER__PASSWORD=replace_with_strong_password
FREQTRADE__API_SERVER__JWT_SECRET=replace_with_64_random_hex_chars
FREQTRADE__TELEGRAM__TOKEN=000000:replace_me
FREQTRADE__TELEGRAM__CHAT_ID=0
EOF
chmod 600 ~/freqtrade-binance/.env

# 3. Compose file. env_file passes secrets into the container.
#    The explicit config.json bind is required: the image ships its own default
#    user_data/config.json which would otherwise be used.
cat > ~/freqtrade-binance/compose.yaml <<'EOF'
services:
  freqtrade:
    image: freqtradeorg/freqtrade:stable
    container_name: freqtrade
    restart: unless-stopped
    command:
      - trade
      - --config
      - /freqtrade/user_data/config.json
      - --strategy
      - ${FREQTRADE_STRATEGY:-SampleStrategy}
      - --strategy-path
      - /freqtrade/user_data/strategies
    env_file:
      - .env
    volumes:
      - ./user_data:/freqtrade/user_data
      - ./config.json:/freqtrade/user_data/config.json:ro
    ports:
      - "127.0.0.1:8181:8080"
EOF
```

Minimal schema-valid `config.json` with dry-run on and no secrets:

```bash
cat > ~/freqtrade-binance/config.json <<'EOF'
{
  "max_open_trades": 3,
  "stake_currency": "USDT",
  "stake_amount": 10,
  "tradable_balance_ratio": 0.99,
  "fiat_display_currency": "USD",
  "dry_run": true,
  "dry_run_wallet": 1000,
  "timeframe": "5m",
  "exchange": {
    "name": "binance",
    "key": "replace_me",
    "secret": "replace_me",
    "pair_whitelist": ["BTC/USDT", "ETH/USDT"],
    "pair_blacklist": []
  },
  "api_server": {
    "enabled": true,
    "listen_ip_address": "0.0.0.0",
    "listen_port": 8080,
    "username": "replace_me",
    "password": "replace_me",
    "jwt_secret_key": "replace_me_jwt_secret_min_32_chars_long"
  },
  "telegram": {
    "enabled": false,
    "token": "000000:replace_me",
    "chat_id": "0"
  },
  "initial_state": "running"
}
EOF
python3 -m json.tool ~/freqtrade-binance/config.json > /dev/null && echo "JSON ok"
```

```bash
# 4. Validate BEFORE starting anything. list-strategies exercises config load,
#    schema validation, and exchange init without opening a trade loop.
cd ~/freqtrade-binance
docker compose run --rm freqtrade list-strategies
```

### robust_usage

Validate in escalating steps, so a failure names its own layer.

```bash
cd ~/freqtrade-binance

# Layer 1: config file is found, parsed, and schema-valid
docker compose run --rm freqtrade show-config \
  | jq '{dry_run, state, runmode: .runmode, strategy, exchange: .exchange.name}'

# Layer 2: strategy resolves
docker compose run --rm freqtrade list-strategies

# Layer 3: exchange connectivity and market data
docker compose run --rm freqtrade download-data --timerange 7d --pairs BTC/USDT

# Layer 4: the real thing, bounded so it cannot hang a terminal
timeout 90 docker compose run --rm freqtrade > /tmp/ft_run.log 2>&1
grep -E 'Runmode set to dry_run|Using Exchange|heartbeat|CRITICAL' /tmp/ft_run.log
```

Start persistently and confirm it survives a restart:

```bash
docker compose up -d
sleep 25
docker compose ps --format '{{.Status}}'
docker compose restart && sleep 30
docker compose logs --since 40s 2>&1 | grep -c heartbeat   # expect >= 1
```

Query the API without ever printing a secret. Read credentials from `.env` into shell
variables, authenticate with `curl -u`, and never `echo` them.

```bash
u=$(grep -E '^FREQTRADE__API_SERVER__USERNAME=' .env | cut -d= -f2)
p=$(grep -E '^FREQTRADE__API_SERVER__PASSWORD=' .env | cut -d= -f2)
unset u p

curl -fsS -u "$u:$p" http://127.0.0.1:8181/api/v1/ping
curl -fsS -u "$u:$p" http://127.0.0.1:8181/api/v1/balance \
  | jq '{total: .total, stake: .stake, dry: (.currencies[0].balance == 1000)}'
curl -fsS -u "$u:$p" http://127.0.0.1:8181/api/v1/show_config \
  | jq '{dry_run, state, strategy}'
```

`/api/v1/status` returning `[]` is correct and healthy: it lists open trades, and a fresh
dry-run bot has none. Do not treat an empty array as a failure.

**Node.js** alternative for scripted health checks:

```javascript
const { execFileSync } = require('node:child_process');

function creds(envPath) {
  const out = execFileSync('grep', ['-E', '^FREQTRADE__API_SERVER__(USERNAME|PASSWORD)=', envPath], { encoding: 'utf8' });
  const m = Object.fromEntries(out.trim().split('\n').map(l => l.split('=')));
  return [m.FREQTRADE__API_SERVER__USERNAME, m.FREQTRADE__API_SERVER__PASSWORD];
}

async function check(envPath = '/home/USER/freqtrade-binance/.env', port = 8181) {
  const auth = 'Basic ' + Buffer.from(creds(envPath).join(':')).toString('base64');
  const get = async ep => {
    const r = await fetch(`http://127.0.0.1:${port}/api/v1/${ep}`, { headers: { Authorization: auth } });
    if (!r.ok) throw new Error(`${ep} -> HTTP ${r.status}`);
    return r.json();
  };
  const [ping, balance, config] = [await get('ping'), await get('balance'), await get('show_config')];
  return { ping: ping.status, dryRun: config.dry_run, state: config.state, total: balance.total };
}

check().then(r => console.log(JSON.stringify(r, null, 2)));
```

## Output format

Report these exact fields after a deployment:

- `container`: service name and `Up <n> seconds` status
- `dry_run`: must be `true`
- `state`: `running`
- `strategy`: resolved strategy class name
- `exchange`: `binance`
- `total`: simulated wallet balance (should equal `dry_run_wallet`)
- `open_trades`: `0` on a fresh bot
- `restart_policy`: `unless-stopped`
- Error shape: `CRITICAL` lines from the log, plus the first `Failed validating '<rule>'`
  line, quoted verbatim

## Rate limits / Best practices

- Never print `.env` values. Freqtrade redacts secrets in `show-config` but *not* in its own
  runtime errors — a rejected Telegram token is printed in plaintext three times in the log.
- If any secret appears in a log or terminal, treat it as burned: rotate it.
- Keep the API bound to `127.0.0.1` unless remote access is explicitly requested, and then put
  a reverse proxy with TLS in front of it.
- Check for a host port collision before starting: another Compose stack may already hold
  8080. Pick a free loopback port rather than moving to `0.0.0.0`.
- Validate with `list-strategies` before `up -d`; it is the cheapest layer that catches
  schema and exchange errors.
- Use `timeout` on any foreground `docker compose run` of `trade` so a hung exchange cannot
  hold the terminal.
- Prefer `restart: unless-stopped` over a host systemd unit; Docker already supervises it.

## Agent prompt

```text
You have Freqtrade Docker dry-run deployment capability. When a user asks to set up,
rebuild, or containerize a Freqtrade bot:

1. Check whether ~/freqtrade-binance already exists and is running (`docker compose ps`).
   Never delete an existing install before preserving .env, strategies, and git history.
2. Create the directory layout, .env (mode 600, FREQTRADE__* names only), compose.yaml,
   and config.json. Put inert placeholders in config.json — NEVER ${VAR} references,
   which fail schema validation.
3. Validate in order: JSON parse -> list-strategies -> bounded trade run. Stop and report
   at the first failing layer rather than starting the service.
4. Start with `docker compose up -d`, verify via the REST API on the loopback port, and
   confirm the restart policy works with one `docker compose restart`.
5. Report the Output format fields above.

Never enable live trading. Never print secret values. If a secret appears in output,
say plainly that it must be rotated.
```

## Troubleshooting

**`Configuration error: '${FREQTRADE_JWT_SECRET}' is too short`**
- Symptom: validation fails on `jwt_secret_key` `minLength`.
- Cause: `${VAR}` was left in `config.json`; Freqtrade does not interpolate it.
- Solution: put a 32+ character inert placeholder in `config.json`, and supply the real
  value via `FREQTRADE__API_SERVER__JWT_SECRET`.

**`Error: Config file /freqtrade/user_data/config.json not found`**
- Symptom: the container cannot see a config that exists on the host.
- Cause: only `./user_data` is mounted and the file lives at the repo root, or lives outside
  the mount.
- Solution: bind it explicitly —
  `- ./config.json:/freqtrade/user_data/config.json:ro`.

**`failed to set up container networking ... port is already allocated`**
- Symptom: `docker compose up -d` fails, container never starts.
- Cause: another container already publishes that host port.
- Solution: find the holder with `docker ps --format '{{.Names}}\t{{.Ports}}' | grep <port>`
  and map the bot to a different loopback port.

**Telegram logs `Unauthorized! Giving up on that message.` then `token ... was rejected`**
- Symptom: repeated `TelegramError`, `Telegram init failed`, bot otherwise healthy.
- Cause: the bot token is revoked, or belongs to a different bot than the chat id.
- Solution: set `"telegram": {"enabled": false}` to run headless, and rotate the token with
  BotFather. Rotate it regardless — the value is now in plaintext logs.

**Validation complains about a top-level key like `protections`**
- Symptom: `Additional properties are not allowed ('protections' was unexpected)`.
- Cause: `protections` moved out of `config.json` and into the strategy class.
- Solution: define `protections` as a property on the strategy, e.g.
  `protections = [{"enable_protections": False}]`.

**Container starts, API returns `{}` for `/api/v1/status`**
- Not a failure. A fresh dry-run bot has no trades. Use `/api/v1/balance` and
  `/api/v1/show_config` to confirm health.

## See also

- [../freqtrade-self-heal/SKILL.md](../freqtrade-self-heal/SKILL.md) — self-healing and
  self-learning layer for a live freqtrade bot (may live in a user-level skills directory)
- [../telegram-bot/SKILL.md](../telegram-bot/SKILL.md) — Telegram bot development and
  token handling

---

## Notes

- Freqtrade's `configuration/environment_vars.py` is the ground truth for variable naming;
  read it from the image rather than trusting docs, since it changes between versions.
- Commands in this skill were executed against `freqtradeorg/freqtrade:stable` reporting
  version 2026.8. Re-verify `environment_vars.py` after a major version bump.