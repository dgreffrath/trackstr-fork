---
name: opencode-mcp-server
description: "Install and wire an MCP server into opencode so the assistant can call external tools (Spotify, ChatGPT, Codex, homebrew, etc.). Use when the user says 'install/add/configure an MCP server', names a service they want opencode to integrate, or asks why an MCP tool is missing. Covers choosing the right package (npm/PyPI/a CLI's own mcp-server subcommand), verifying the binary, editing ~/.config/opencode/opencode.json, the {env:VAR} secret pattern, and the mandatory restart."
---

# Install an MCP server into opencode

Wire a third-party MCP server into opencode's global config so its tools appear
in the agent's toolset after restart.

## When to use

- User asks to "install / add / configure [service] MCP" on opencode.
- An MCP server should cover Spotify, ChatGPT, Codex bridge, a browser
  automation driver, or any external service.
- Deciding between "npx package", "globally installed CLI", and "the tool's own
  `mcp-server` subcommand".

## Key facts (learn the hard way once)

1. **opencode config is NOT hot-reloaded.** Editing `opencode.json` does nothing
   until the user quits and reopens opencode. Say so every time.
2. **Secret values must use `{env:VAR}` interpolation**, not literal secrets.
   opencode substitutes `{env:NAME}` from the process environment at launch.
   Existing entries (github, youtube, spotify) follow this pattern — copy it.
3. **Invalid JSON = hard failure.** opencode refuses to start on a malformed
   config. Always validate with `python3 -m json.tool` after editing.
4. **Three supply paths, verify whichever you pick:**
   - Registry package: `npm view <pkg> bin` (npx) or `pip index versions <pkg>` (uvx/pip).
   - Globally installed CLI with a real `bin` (`npm i -g`, `pipx install`).
   - A CLI that ships its own bridge: `codex mcp-server` (OpenAI Codex CLI),
     or any tool exposing an MCP stdio endpoint.
5. **Verify the command launches** (run it, watch for a stdio/version banner,
   or `--help`) before you write it into the config; a bad command string fails
   silently at startup.
6. **`type: "local"` + `command: [...]` with a 120000 timeout** is the standard
   shape for a local stdio server. Absolute paths avoid PATH surprises in the
   MCP launcher.

## Steps

1. **Pick the package.** Ask the user how they want the service reached
   (official cloud API vs. browser automation vs. local bridge). Check the
   registry first: `npm view <pkg> version bin` or `pip index versions <pkg>`.
   Reject builds that target the wrong OS (many ChatGPT MCPs are macOS-only
   `AppleScript` drivers and will not run on Linux).
2. **Install and verify the binary.**
   - npx: `npx -y <pkg>` (fetches on demand; good for short-lived servers).
   - global: `npm i -g <pkg>` or `pip install --user <pkg>`; confirm
     `command -v <bin>` resolves.
   - built-in bridge: run `<tool> mcp-server --help` to confirm flags.
3. **Edit `$HOME/.config/opencode/opencode.json`.** Insert a block under the
   top-level `"mcp"` object, keeping the existing `{env:...}` secret pattern:

```json
"servicename": {
  "type": "local",
  "command": ["<cmd>", "<args...>"],
  "enabled": true,
  "timeout": 120000,
  "environment": {
    "SERVICE_TOKEN": "{env:SERVICE_TOKEN}"
  }
}
```

4. **Validate** the edited file:

```bash
python3 -m json.tool ~/.config/opencode/opencode.json > /dev/null && echo OK
```

5. **Hand the user the setup checklist** — every env var they must `export`
   before restart (and where to get the value: developer dashboard, a session
   cookie from an authenticated browser, an API key page), plus the restart
   step: quit opencode and reopen.

## Output format

- The exact config diff (server name, command, env vars), the env vars the user
  must set and where to obtain each, and the restart reminder.

## Best practices

- Never put a real token/key in the config or in logs — always `{env:VAR}` and
  tell the user how to get the value.
- Prefer the tool's own auth when one exists (OAuth app for Spotify, `chatgpt
  login`, `codex login`) over scraping a session token; only fall back to a
  browser/session-cookie token when there is no API.
- Timeout 120000 compensates for slow first-spawn (npx fetch + native addons).

## Troubleshooting

**Config looks right but the server never appears**
- Restart opencode; config is read at startup only.

**opencode refuses to start after the edit**
- `python3 -m json.tool ~/.config/opencode/opencode.json` — fix the JSON.

**Tool exists but launches nowhere / errors at spawn**
- Run the exact `command` array yourself; the MCP launcher does not use your
  interactive PATH, so use absolute paths (`command -v <bin>`).

**Server starts, then dies with a Cloudflare / login error**
- The env token is missing or stale. Confirm the `{env:VAR}` name matches an
  exported var, and re-grab the token/cookie from the source.

## Agent prompt

```text
You have opencode-mcp-server capability. When the user asks to install or
configure an MCP server for opencode:

1. Pick a supply path (npm/pyPI package verified in its registry, a globally
   installed bin, or a CLI's own mcp-server subcommand) that matches the
   platform; reject OS-specific drivers that will not run here.
2. Install/verify the binary launches before touching the config.
3. Add the block under the "mcp" object in ~/.config/opencode/opencode.json
   using type local, an absolute command, enabled true, timeout 120000, and
   {env:VAR} for every secret. Validate the JSON with json.tool.
4. Report the diff, the exact env vars the user must export (with where to get
   each), and that a restart of opencode is required.
```