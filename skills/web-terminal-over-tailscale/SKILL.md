---
name: web-terminal-over-tailscale
description: "Expose an interactive shell (Codex CLI, any TUI tool) to a phone browser by running ttyd as a systemd user service bound to the Tailscale interface only. Use when: (1) user wants to run a server CLI tool from their iPhone/iPad, (2) 'run codex on my phone' / 'just like odysseus' / remote shell requests, (3) a web terminal is needed without exposing a shell to the LAN or internet, (4) deciding how a phone should reach a headless Linux server's tools."
---

# Web terminal on a phone via Tailscale + ttyd

Bind `ttyd` to the `tailscale0` interface as a systemd **user** service. Only
Tailnet devices can reach it; the Tailnet already authenticates and encrypts,
so no password is needed and the shell never touches the LAN or internet.

## When to use

- A CLI tool on the server must be usable from a phone (Codex CLI, htop, git…).
- The machine already runs Tailscale and the phone is in the same Tailnet
  (`tailscale status` shows both nodes).
- Exposure rules: never bind a shell to `0.0.0.0` / the LAN IP without auth.

## Key facts (learn the hard way once)

1. **ttyd ≥1.7 is readonly by default.** Without `-W` the terminal renders but
   ignores keystrokes — an interactive shell silently looks broken.
2. **`-i` takes an interface name** (`tailscale0`), not an IP. Binding the
   interface binds only that interface's addresses, so LAN/loopback are closed
   for free. Interface-specific binds survive IP churn; a hardcoded IP does not
   (a re-registered Tailnet node gets a new IP and the service goes dead).
3. **User services die without linger** after logout/reboot. Run
   `loginctl enable-linger $USER` once so `enabled` services start at boot.
4. **`-w <dir>` sets the child's cwd**; run a login shell (`/bin/bash -l`) so
   `.profile`/PATH load and tools like `codex` resolve. Without `-l` the shell
   misses PATH entries and the tool looks "not installed".
5. **Tailscale is the auth.** No `-c user:pass` needed inside the Tailnet.
   Basic auth only becomes relevant if you deliberately widen the bind.

## Required tools

- Tailscale on server and phone, same Tailnet
- `ttyd` — `brew install ttyd` (user-level, no sudo) or `sudo apt install ttyd`
- systemd user units (`systemctl --user`)

## Steps

1. Confirm both nodes and the interface:

```bash
tailscale status            # phone must appear
ip -br addr | grep tailscale   # expect tailscale0
```

2. Install ttyd and write `~/.config/systemd/user/ttyd-web.service`:

```ini
[Unit]
Description=ttyd web terminal over Tailscale
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ttyd -i tailscale0 -p 7681 -W -w %h /bin/bash -l
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
```

(Use `$(command -v ttyd)` for the correct absolute path — brew installs to
`/home/linuxbrew/.linuxbrew/bin/ttyd`, apt to `/usr/bin/ttyd`.)

3. Enable, start, persist:

```bash
systemctl --user daemon-reload
systemctl --user enable --now ttyd-web.service
loginctl enable-linger $USER
```

4. **Verify scope from the server** — this is the security gate:

```bash
TS_IP=$(tailscale ip -4)          # or hostname -I | grep 100.
curl -s -o /dev/null -w '%{http_code}\n' http://$TS_IP:7681/   # expect 200
LAN_IP=$(hostname -I | awk '{print $1}')
curl -s --max-time 3 http://$LAN_IP:7681/ || echo "blocked (good)"  # expect fail
```

Acceptance: `200` on the Tailscale IP, connection refused/timeout on the LAN IP.

5. On the phone: Tailscale connected → open `http://<server-tailscale-ip>:7681`
   in Safari → type the tool name (e.g. `codex`) in the shell.

## Output format

- Report the exact URL for the phone, the bind interface, and both scope-check
  results (Tailscale 200 / LAN blocked).

## Best practices

- Interface bind (`-i tailscale0`) over `0.0.0.0` + firewall — fewer ways to
  misconfigure, and the security check above proves it.
- Keep the service a **user** unit (no root shell over HTTP) with `Restart=on-failure`.
- The phone URL uses the Tailscale IP; if the node re-registers, refresh the IP
  in instructions (the interface bind keeps the service itself correct).

## Troubleshooting

**Terminal loads but keystrokes do nothing**
- ttyd 1.7 readonly default. Add `-W` to ExecStart and restart.

**`codex: command not found` (or similar) inside the web shell**
- Missing login shell. Use `/bin/bash -l` in ExecStart so `.profile` runs.

**LAN IP also serves the page (or `0.0.0.0` in use)**
- `-i` omitted or wrong interface name. Confirm with `ss -ltnp | grep ttyd`
  — bound address must be the Tailscale IP only.

**Service dies after reboot / on logout**
- `loginctl show-user $USER -p Linger` → `no`. Run `loginctl enable-linger $USER`.

**URL stopped working after a network change**
- The phone fell off the Tailnet (`tailscale status`) — reconnect the app;
  the service itself needs no change when bound by interface.

## Agent prompt

```text
You have web-terminal-over-tailscale capability. When a user wants a server CLI
tool (Codex CLI etc.) usable from their iPhone like their other web services:

1. Confirm both machines are on the same Tailnet and note the tailscale0 interface.
2. Install ttyd (brew preferred), write a systemd user unit binding
   -i tailscale0 -p 7681 -W with a login shell (-w for cwd), enable linger,
   enable --now the unit.
3. Security-gate: curl the Tailscale IP (expect 200) AND the LAN IP (expect
   refuse). Report the phone URL only after both pass.
```

## See also

- [../opencode-zen-through-odysseus/SKILL.md](../opencode-zen-through-odysseus/SKILL.md) — the Codex-to-Zen bridge this terminal drives on the phone.
