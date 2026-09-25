---
name: chatgpt-webui-token-recovery
description: "Recover and repair the ChatGPT WebUI MCP session token on Linux when the server answers 'failed_to_get_access_token_from_session_cookie_400', 401, or an empty model list. Use when: (1) chatgpt_webui_* tools return a session-cookie error, (2) the stored token file is truncated, contains ANSI escape junk, or is the wrong length, (3) chatgpt-token-extract reports 'No session token found' even though the user is logged in to chatgpt.com, (4) re-acquiring the account id / plan / model list, or (5) setting up the token file for the first time. Covers chunked NextAuth cookies, Chrome v10/v11 OS-crypt framing, keyring-derived AES keys, and validating a JWE before writing it."
---

# ChatGPT WebUI MCP session-token recovery (Linux)

The `chatgpt_webui_*` MCP tools authenticate with a NextAuth session cookie
stored in a file. Almost every failure is a **bad token file**, not a revoked
account — and the account is usually still logged in and fine.

## Symptom → cause table

| What you see | Real cause |
|---|---|
| `failed_to_get_access_token_from_session_cookie_400` | token file unreadable, truncated, or corrupt |
| token file has `\x1b[A` / `\x1b[B` bytes | arrow keys were typed into a `cat`/`read` that was writing the file |
| token file is only a few dozen chars | paste was truncated; real token is ~4 kB |
| `chatgpt-token-extract` says "No session token found" but user *is* logged in | NextAuth split the cookie into `NAME.0`, `NAME.1` chunks; a lookup for the bare name misses |
| extractor decrypts to binary garbage | wrong key (macOS `peanuts` constant used on Linux) |
| first 16 bytes garbage, rest is clean base64 | wrong IV — CBC only corrupts block 0, so this looks like partial success |

## Step 1 — triage the token file

Never print the token. Report shape only:

```bash
f=~/.config/chatgpt-webui-mcp/session-token.txt
stat -c '%s bytes, mode %a' "$f"
LC_ALL=C grep -qP '^[\x21-\x7e]+$' "$f" \
  && echo "clean" || echo "CONTROL CHARS PRESENT"
xxd "$f" | head -3
```

Healthy = a few thousand printable ASCII chars, mode `600`, no ESC bytes.

## Step 2 — go to the source, not the file

If the file is wrong, regenerate it from Chrome. A working Chrome login is
always better than a pasted token.

## Step 3 — the three Chrome-cookie traps

Read all three before writing any extraction code. Each one has silently
produced "works on my machine" breakage.

### Trap 1 — chunked cookies

Long cookie values are split with a numeric suffix. Query the base name **and**
the suffixed forms, then sort by numeric index (lexicographic sort breaks at
`.10`):

```sql
select host_key,name,encrypted_value from cookies
where name='__Secure-next-auth.session-token'
   or name like '__Secure-next-auth.session-token.%'
```

Decrypt each chunk and concatenate the **plaintext** in index order.

### Trap 2 — key derivation is platform-specific

`b'peanuts'` is the **macOS** constant. On Linux the key comes from the
keyring (`secret-tool lookup application chrome`). Using `peanuts` on Linux
yields ~37% printable bytes — obvious garbage, but easy to misread as
"encryption failed" rather than "wrong password":

```python
pw = subprocess.run(['secret-tool','lookup','application','chrome'],
                    capture_output=True).stdout.split(b'\n')[0].strip()
key = hashlib.pbkdf2_hmac('sha1', pw, b'saltysalt', 1, 16)
```

### Trap 3 — v10 vs v11 framing

`v11` (Chrome 130+ on Linux) is **not** prefix + ciphertext-with-16-space-IV:

```
v10:  'v10' | ciphertext                     IV = b' ' * 16
v11:  'v11' | header(16) | IV(16) | ciphertext
```

A wrong IV corrupts only the first plaintext block, so the output is
16 garbage bytes followed by perfectly valid base64url. That is the signature
of a misaligned frame, not a corrupt cookie.

Scan candidate offsets and pick the one that yields 100% base64url:

```python
for off in range(0, 64, 16):
    iv, ct = body[off:off+16], body[off+16:]
    pt = aes_cbc_decrypt(key, iv, ct)   # strip PKCS7
    if is_base64url(pt): break          # 'eyJhbGci' == base64('{"alg":')
```

## Step 4 — validate before writing

A NextAuth session token is a **JWE compact serialisation**: 5 dot-separated
base64url parts, `header.encrypted_key.iv.ciphertext.tag`. For `alg=dir`,
part 1 is legitimately **empty** — requiring all 5 parts to be non-empty
rejects a perfectly good token.

Refuse to write anything that is not 5 parts. This check is what converts a
silent future breakage into an immediate, obvious error.

## Step 5 — permissions and hygiene

- Token file: `chmod 600`, directory `700`.
- Work on a **copy** of `~/.config/google-chrome/Default/Cookies`; never open
  the live DB read-write (Chrome holds it locked, and WAL means a plain copy
  can miss recent writes).
- `shred -u` every temp copy of the cookie DB and every staged token file when
  done. Encrypted cookies plus a keyring read are enough to reconstruct a
  session.

## Step 6 — verify

```bash
# must print: token length in the thousands, no control chars, mode 600
```

Then confirm the MCP itself is healthy, and read the account id / plan /
expiry from the session response:

- session resolves (no `..._400`) → cookie is valid
- `models` returns a non-empty list → the whole chain works
- session response carries `account.id`, `account.planType`, `expires`

## Reporting rules

The token, the `accessToken`, and the `sessionToken` echoed by the session
tool are **account credentials**. Never paste them into output, a commit, an
issue, a PR, or a log. Report only: length, validity, account id, plan, expiry.

## Reusable extraction skeleton

```python
import hashlib, os, shutil, sqlite3, subprocess, tempfile
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

COOKIE = '__Secure-next-auth.session-token'
src = os.path.expanduser('~/.config/google-chrome/Default/Cookies')

tmp = tempfile.NamedTemporaryFile(suffix='.db', delete=False); tmp.close()
shutil.copy2(src, tmp.name)                      # never touch the live DB
db = sqlite3.connect(tmp.name)
rows = db.execute(
    "select host_key,name,encrypted_value from cookies "
    "where name=? or name like ?", (COOKIE, COOKIE + '.%')).fetchall()
db.close(); os.unlink(tmp.name)

rows.sort(key=lambda r: int(r[1].rsplit('.', 1)[-1]))   # numeric, not lexical

pw = subprocess.run(['secret-tool','lookup','application','chrome'],
                    capture_output=True).stdout.split(b'\n')[0].strip()
key = hashlib.pbkdf2_hmac('sha1', pw, b'saltysalt', 1, 16)

def decrypt(ev):
    body = ev[3:]
    iv, ct = (body[16:32], body[32:]) if ev[:3] == b'v11' else (b' ' * 16, body)
    d = Cipher(algorithms.AES(key), modes.CBC(iv)).decryptor()
    pt = d.update(ct) + d.finalize()
    pad = pt[-1]
    return pt[:-pad] if 1 <= pad <= 16 and pt[-pad:] == bytes([pad]) * pad else pt

tok = b''.join(decrypt(ev) for _, _, ev in rows).decode().strip()
segs = tok.split('.')
assert len(segs) == 5, 'not a JWE - refusing to write'   # then write 0600
```
