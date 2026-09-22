# Skill: freqtrade-binance-auth-diagnostics

Diagnose "auth failed binance api key" / Binance AUTH FAILED alerts on a live
freqtrade bot before touching the credentials. Most of these alerts are **not**
a revoked key — they are a flaky network path, clock wobble, or an IP whitelist
that was never cleared. Rotating keys when the key is fine causes more work than
it fixes and can lock you out mid-rotation.

## When to Use

- A health monitor emits `Binance API-key AUTH FAILED (HTTP 4xx)` or
  `reconcile probe unavailable`.
- The bot logs `ccxt.base.errors.NetworkError: Connection closed by remote
  server, closing code 1008` on its websocket OHLCV streams.
- Telegram is spammed with `AUTH FAILED` while the service is otherwise up.

## Triage Steps (in this order — they are cheap and read-only)

1. **Prove the key works right now** with a real signed call, and print the
   exact Binance error body on failure (urllib raises `HTTPError`; read `e.read()`):
   sign `/api/v3/account` with HMAC-SHA256 using `X-MBX-APIKEY`, and include the
   server-time skew in the output. If it returns 200 with balance fields, the
   key + secret are valid **at this moment**.
2. **Check the clock**: `chronyc tracking`. `-1021` (HTTP 400) appears when
   `|local - serverTime| > recvWindow`. A 238 ms skew rules skew out; a stepped
   clock (NTP step after suspend / mobile failover) is a real cause.
3. **Check IP whitelist**: compare the key's Binance unlock IP list with the
   current egress IP (`curl -s ifconfig.me`). A 401 `-2015` with
   "Invalid API-key, IP, or permissions" means the box's IP changed or the key
   IP whitelist was never updated after rotation.
4. **Correlate the event, don't isolate it**: grep the logs at the exact
   failure minute. A single 400/401 surrounded by WS `1008` drops and DNS
   `Temporary failure in name resolution` lines means *transport*, not
   *credentials*. On the same minute the exchange REST worked find for the
   trade loop → the monitor, not the bot, is wrong.

## Classification

- `404/403/401` repeated on every retry with a stable IP and clock → real key
  problem (revoked, IP-restricted, wrong secret, or the value in `.env` was
  scrubbed to empty by a rotation script). Rotate deliberately, keep a rollback
  copy of the previous `.env`, and never leave the peer key out of the whitelist.
- `400` sporadic on a flaky link with the key proving valid on re-test → false
  positive (time wobble or mangled request through a middlebox). Fix the probe,
  not the key.
- DNS / socket / `1008` errors (no HTTP response body) → transport. Network
  watchdog owns this; do not rotate credentials.

## Hardened Auth Probe (stdlib-only)

Sign with Binance server time, widen `recvWindow` to 10000, and retry 3x before
reporting failure. A dead key still fails every retry with -2015, so a true
failure is never masked:

```python
import sys, time, json, urllib.request, urllib.parse, hashlib, hmac
key, secret = sys.argv[1], sys.argv[2]
def signed(path, timestamp):
    q = urllib.parse.urlencode({"timestamp": timestamp, "recvWindow": 10000})
    sig = hmac.new(secret.encode(), q.encode(), hashlib.sha256).hexdigest()
    req = urllib.request.Request(path + "?" + q + "&signature=" + sig,
                                 headers={"X-MBX-APIKEY": key})
    return urllib.request.urlopen(req, timeout=10)
base = int(time.time()*1000)
try:
    server = json.loads(urllib.request.urlopen(
        "https://api.binance.com/api/v3/time", timeout=10).read())
    base = int(server["serverTime"])
except Exception:
    pass
for attempt in range(3):
    try:
        signed("https://api.binance.com/api/v3/account", base + attempt*2000).read()
        print("OK"); break
    except Exception as e:
        last = e; time.sleep(2)
else:
    print("FAIL: " + str(last))
```

Same principle for a ccxt reconcile/balance probe: catch
`ccxt.NetworkError`/`ccxt.AuthenticationError` and log to the heal log instead
of paging Telegram — keep Telegram notifications for real wallet desync or a
hard exchange rejection.

## Rules

- Never rotate a key because a probe failed once on an unstable link.
- Never print full secrets to the terminal; fingerprint with first 6 / last 4.
- Keep the rollback copy and the empty-seed check before concluding a rotation
  "landed" (a rotation left blank `BINANCE_API_KEY=` in the seed file means the
  new key was never active).