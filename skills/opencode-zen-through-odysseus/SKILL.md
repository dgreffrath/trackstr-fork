---
name: opencode-zen-through-odysseus
description: "Wire opencode to OpenCode Zen models (big-pickle et al.) through a local OpenAI-compatible proxy such as Odysseus (127.0.0.1:7000/v1). Use when: (1) opencode should talk to a local bridge instead of the zen cloud directly, (2) GET localhost:7000/v1/models returns 503 'OPENCODE_ZEN_API_KEY is not set', (3) the bridge answers 403 FreeTierError 'can only be used from within OpenCode', or (4) the proxy returns 401 'Invalid API key' in opencode."
---

# opencode→Zen through a local bridge (e.g. Odysseus)

Odysseus ships an OpenAI-compatible `/v1` surface (`GET /v1/models`,
`POST /v1/chat/completions`) that **proxies server-side** to OpenCode Zen
(`https://opencode.ai/zen/v1`). opencode then points its provider at
`http://127.0.0.1:7000/v1` with **no client-side key** — the zen API key lives
only in the bridge's `.env`. This keeps the key out of client config and gives
one gateway for model traffic.

## When to use

- opencode CLI should route its zen models (e.g. `big-pickle`) through a local daemon.
- `curl http://127.0.0.1:7000/v1/models` → `503` with body
  `"OPENCODE_ZEN_API_KEY is not set. Add it to ~/odysseus/.env (...)"`.
- The bridge returns `403` `FreeTierError: "OpenCode's free tier can only be used from within OpenCode"`.
- The bridge or upstream returns `401` `AuthError: Invalid API key.`

## Key facts (learn the hard way once)

1. **Free-tier app keys cannot go through the proxy.** The api key stored by
   opencode itself (`~/.local/share/opencode/auth.json`, provider `opencode`,
   `type: api`) is a free-tier credential. The zen gateway explicitly rejects
   it outside the CLI with `FreeTierError` — even when opencode's own client
   fingerprint is faked. The bridge needs a **paid** Zen API key created at
   `https://opencode.ai/zen` → Settings → API keys.
2. **The effective zen endpoint is `https://opencode.ai/zen/v1`.** Confirm from
   your own opencode traffic in `~/.local/share/opencode/log/opencode.log`
   (look for `https://opencode.ai/zen/v1/chat/completions`). Public guesses like
   `models.opencode.ai` return Cloudflare `1010`, never a real auth answer.
3. **Key placement:** zen accepts the key as `Authorization: Bearer <key>`
   (what the bridge sends upstream) and also in the request body as `api_key`.
   Sending `Bearer` with an invalid key returns `401 Invalid API key`, not
   `403` — use the distinction to tell wrong-key from free-tier.
4. **The bridge is a dumb forwarder.** It only relays `content-type` and the
   `Authorization` it builds from its own env. It will not add opencode client
   markers, so never assume it can smuggle free-tier keys through.

## Required tools

- `systemctl --user` (or the service manager hosting the bridge daemon)
- `curl`, Python 3
- Access to the machine running the bridge (`~/odysseus/.env` here)

## Skills

### diagnose

1. Check the bridge is up and look at what it says:

```bash
curl -s http://127.0.0.1:7000/v1/models
```

2. Inspect the proxy route to learn its contract (base URL, auth header it
   sends upstream, hop-by-hop filtering):

```bash
rg -n "OPENCODE_ZEN_API_KEY|ZEN_BASE|Authorization" routes/zen_proxy_routes.py
```

3. If the proxy has no key, probe the upstream directly with the client's
   stored key to classify the failure. Never print the key:

```python
import json, os, urllib.request, urllib.error
key = json.load(open(os.path.expanduser("~/.local/share/opencode/auth.json")))["opencode"]["key"].strip()
url = "https://opencode.ai/zen/v1/chat/completions"
body = json.dumps({"model":"big-pickle","messages":[{"role":"user","content":"Reply with exactly OK"}],
                   "max_completion_tokens":8, "stream": False}).encode()
for label, data, hdrs in [
    ("api_key in body ", {**json.loads(body.decode()), "api_key": key}, {}),
    ("Bearer header    ", None, {"Authorization": f"Bearer {key}"}),
]:
    req = urllib.request.Request(url, data=json.dumps(data).encode() if data else body,
                                 headers={**hdrs, "Content-Type": "application/json",
                                          "User-Agent": "opencode/"})
    try:
        with urllib.request.urlopen(req, timeout=40) as r:
            print(label, r.status, r.read(150))
    except urllib.error.HTTPError as e:
        print(label, e.code, e.read(150))
```

Acceptance: a `FreeTierError` means a free-tier key and the paid-key path is
required. A `401 Invalid API key` with the paid key means the key is wrong or
rotated. HTTP 200 means the key works upstream and only the `.env` wiring is
left.

### configure

1. Get a paid Zen API key from `https://opencode.ai/zen` → Settings → API keys.
2. Append it to the bridge env without echoing it back:

```bash
cd ~/odysseus
cp .env ".env.bak-$(date +%Y%m%d_%H%M%S)"
python3 - "$KEY" <<'EOF'
import sys
key = sys.argv[1].strip()
lines = open(".env").read().splitlines()
lines = [l for l in lines if not l.startswith("OPENCODE_ZEN_API_KEY=")]
lines.append(f"OPENCODE_ZEN_API_KEY={key}")
open(".env", "w").write("\n".join(lines) + "\n")
EOF
```

`OPENCODE_ZEN_BASE_URL` defaults to `https://opencode.ai/zen/v1` in the proxy
code, which is correct; only set it explicitly if you target a mirror.

3. Restart the bridge so it re-reads `.env`:

```bash
systemctl --user restart odysseus-ui.service
```

### verify

```bash
# models list reaches you through the bridge
curl -s http://127.0.0.1:7000/v1/models | python3 -c "import json,sys; print([m['id'] for m in json.load(sys.stdin)['data']][:10])"

# one tiny completion through the bridge
curl -s http://127.0.0.1:7000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"big-pickle","messages":[{"role":"user","content":"Reply with exactly OK"}],"max_completion_tokens":8,"stream":false}'
```

Then point opencode at the bridge (it sends no key — loopback trust):

```json
{
  "provider": {
    "odysseus-zen": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Odysseus -> OpenCode Zen",
      "options": { "baseURL": "http://127.0.0.1:7000/v1" },
      "models": { "big-pickle": { "name": "big-pickle (via Odysseus)" } }
    }
  },
  "model": "odysseus-zen/big-pickle"
}
```

Config is read at startup — quit and restart opencode, then confirm the header
line shows the bridged model (e.g. `big-pickle (via Odysseus)`).

## Output format

- Diagnostic: bridge status, key classification (`missing / free-tier / paid-ok`),
  the effective upstream URL evidence.
- Post-wiring: `/v1/models` reachable through the bridge, one chat completion
  succeeded, opencode restart confirmed.

## Best practices

- Never copy a key into chat, shells with history, or configs; keep it in the
  bridge's `.env` and back up that file before editing.
- Bind the bridge to loopback (`127.0.0.1`) — it is a key holder; don't expose
  it to the LAN.
- Distinguish failure modes by HTTP status: `503` = env missing, `403` +
  `FreeTierError` = free-tier key, `401` = bad/rotated key, `1010`/`403` from
  guessed hosts = Cloudflare, not auth.

## Troubleshooting

**`FreeTierError` even though opencode works normally**
- Symptom: the CLI is fine but the proxy refuses.
- Solution: free-tier is hard-gated to the opencode app; obtain a paid Zen key
  and put it in the bridge `.env`. There is no client-fingerprint workaround.

**`OPENCODE_ZEN_API_KEY is not set` right after adding it**
- Symptom: bridge still 503s.
- Solution: the service holds an old env until restarted; restart the daemon
  and confirm the var is loaded
  (`systemctl --user show odysseus-ui.service | grep EnvironmentFile`).

**`1010`/Cloudflare when testing the upstream by hand**
- Symptom: direct curl to zen-side hosts gets blocked.
- Solution: that is bot filtering on your client, not auth. Test through the
  daemon or with a normal browser session instead.

## Agent prompt

```text
You have opencode-zen-through-odysseus capability. When opencode should reach Zen
models through a local bridge (Odysseus at 127.0.0.1:7000/v1) and /v1/models 503s:

1. Classify the key state: missing env key (503), free-tier (403 FreeTierError),
   or bad paid key (401). Free-tier keys from opencode's own auth.json NEVER work
   through the proxy — require a paid key from opencode.ai/zen Settings.
2. Add OPENCODE_ZEN_API_KEY to the bridge .env, back it up first, restart the daemon.
3. Verify /v1/models and one tiny chat/completions through the bridge, then have the
   user restart opencode. Never echo the key or write it into opencode config.
```

## See also

- [../freqtrade-self-heal/SKILL.md](../freqtrade-self-heal/SKILL.md) — unrelated daemon ops skill; kept for directory consistency.