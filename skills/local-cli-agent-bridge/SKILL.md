---
name: local-cli-agent-bridge
description: "Expose a locally installed CLI-based coding agent as a selectable primary agent inside OpenCode via a thin bash relay. Use when: (1) Wiring a local agent (ZeroClaw, aider, goose, a custom `myagent -m` binary) into OpenCode's agent selector, (2) An OpenCode custom agent fails with 'Unexpected server error' or a bogus model, (3) A relay script returns empty output, hangs, or uses the wrong config/credentials, (4) Debugging why OpenCode agents that shell out to another agent silently do nothing."
---

# Bridge a Local CLI Agent into OpenCode

Make a locally installed, non-OpenCode agent selectable inside OpenCode's agent picker, by defining a markdown agent whose only job is to shell out to that CLI and echo the reply.

Outcome: the agent appears in OpenCode's agent list, runs, and returns the local agent's real output.

## The two failure modes that cause nearly every broken bridge

**1. The `model:` field must be a real, fully-qualified model id.** OpenCode does not validate it at load time. A made-up id like `fast-cheap` loads fine, `opencode agent list` shows the agent, and then every invocation dies with an opaque error:

```
Error: { "name": "UnknownError", "data": { "message": "Unexpected server error. Check server logs for details." } }
```

Verify the id against the live catalog *before* writing the file:

```bash
opencode models | grep -Fx 'provider/model-id'   # -F: no regex, no substring false-positives
```

Use the OpenCode default (usually already in `opencode.json`) or a cheap confirmed id. Never invent a tier name like "fast" or "cheap" — those are not a namespace.

**2. The relay must export the env the target CLI needs to find its own config.** CLI agents that read a workspace/root from an env var will silently fall back to a *different* config directory when it is unset — usually `~/.<tool>/`, not the one the running daemon uses. The bridge then appears to "work" but uses stale credentials, dead keys, or a broken plugin set, and errors come from the wrong installation.

Find the authoritative path from the service definition, not from the CLI's default:

```bash
systemctl --user cat <service> | grep -iE 'WorkingDirectory|Environment'
# e.g. WorkingDirectory=/opt/myagent/var/myagent
```

Then hard-pin it in the relay and make the relay's own `PATH` explicit, so it behaves the same when OpenCode invokes it with a minimal environment.

## When to use

- The user wants another local agent to appear in OpenCode's agent selector.
- An OpenCode agent built around a bash relay returns nothing, hangs, or errors.
- You are debugging which config/credentials a bridged agent is actually reading.

## Required tools

- OpenCode CLI on `PATH` (provides `opencode models`, `opencode agent list`, `opencode run`).
- The target local agent CLI installed locally.

No external API required. The relay's own model is whatever OpenCode already has configured.

## Step 1: verify the target CLI and its config path

```bash
# Does the target agent answer at all, standalone?
myagent -m "Say OK" --model <tag>

# Which config will it read with no env set?
myagent doctor 2>&1 | head -5

# Which config does the *service* actually use?
systemctl --user cat myagent.service | grep -iE 'WorkingDirectory|Environment'
```

If the standalone call fails, stop. The bridge cannot be more correct than the tool underneath it.

## Step 2: confirm the OpenCode model id

```bash
opencode models | grep -Fx 'provider/model-id'
opencode agent list | grep -E '^[a-z0-9_-]+ \((primary|subagent|all)\)$'
```

The second command confirms which agents are already registered and their `mode`. `primary` puts it in the selector; `subagent` hides it from the picker.

## Step 3: write the relay

The relay is the whole integration. Make it self-configuring, bounded, and quiet on failure.

```bash
#!/usr/bin/env bash
# myagent-ask: send one prompt to the local agent, echo the reply.
# Used by the OpenCode agent that relays to it.
set -o pipefail

if [ $# -eq 0 ]; then
  echo "Error: No prompt provided" >&2
  exit 1
fi

# Pin the config dir the *service* uses. Without this the CLI silently falls
# back to ~/.<tool>/, a different config with different credentials.
export MYAGENT_WORKSPACE="${MYAGENT_WORKSPACE:-/opt/myagent/var/myagent}"
export PATH="/opt/myagent/bin:$PATH"

msg="$*"
model="${MYAGENT_MODEL:-<tag that the local runtime actually reports>}"

# A wedged or cold local model must surface as an error, not hang OpenCode.
output=$(timeout "${MYAGENT_TIMEOUT:-300}" myagent -m "$msg" --model "$model" 2>&1)
status=$?

if [ $status -eq 124 ]; then
  echo "MyAgent timed out after ${MYAGENT_TIMEOUT:-300}s (model $model)." >&2
  exit 124
fi

if [ $status -ne 0 ]; then
  echo "MyAgent error:" >&2
  echo "$output" >&2
  exit $status
fi

echo "$output"
```

Key points, each of which fixes an observed failure:

- **`export ..._WORKSPACE` with a `${VAR:-default}`** — a caller can override, but the default is the real service path, not the CLI's fallback.
- **`export PATH`** — OpenCode may invoke the relay with a minimal env; a Homebrew/Linuxbrew or custom-prefix install is not on the default `PATH`.
- **`timeout` + `status -eq 124`** — a local model on CPU can wedge; without this the OpenCode turn never returns.
- **stderr folded into the capture** (`2>&1`) so the error text is not silently discarded.
- **Non-zero propagates, it does not print-and-continue.** A bridge that swallows failure returns empty text and the user sees a blank reply.

## Step 4: write the agent definition

`~/.config/opencode/agent/myagent.md`:

```markdown
---
description: Talks to the local MyAgent agent with all its plugins and skills
mode: primary
model: <confirmed-provider/model-id>
tools:
  bash: true
  read: false
  edit: false
  write: false
  patch: false
  webfetch: false
  websearch: false
  task: false
---

You are a thin relay. For every user message, run `~/.config/opencode/bin/myagent-ask "${message}"` and return MyAgent's reply verbatim. Do not answer yourself, do not add commentary, and do not retry on MyAgent's own error text. If the command fails, show the error and stop.

Quote the message for the shell exactly as received. If it contains single quotes, escape them as `'\''` rather than switching to double quotes, so the message cannot expand local variables or command substitutions.
```

- `mode: primary` is what puts it in the agent selector.
- Restricting `tools` to `bash` keeps a relay from also editing your files. Omit `tools` and the agent inherits everything.
- The quoting instruction is not decorative: without it, a user message containing `` ` `` or `$()` is executed as a shell substitution.

## Step 5: verify in a clean environment

Test the relay with a stripped env, because that is how OpenCode actually calls it. This is the check that catches the wrong-config bug.

```bash
chmod +x ~/.config/opencode/bin/myagent-ask
bash -n ~/.config/opencode/bin/myagent-ask        # syntax

cd /tmp && env -i HOME="$HOME" PATH=/usr/bin:/bin bash -c \
  'time ~/.config/opencode/bin/myagent-ask "Say OK and nothing else."'
```

Then verify through OpenCode itself:

```bash
opencode run --agent myagent "reply with the single word PONG"
```

Expect the relay's command line plus the target agent's answer in the output. A bare `Error: UnknownError` means the model id is still wrong.

Also confirm registration and that nothing else regressed:

```bash
opencode agent list | grep -A2 '^myagent'
```

## Troubleshooting

**Symptom:** `Error: {"name":"UnknownError", ... "Unexpected server error"}`
- Cause: the `model:` id in the agent markdown does not exist.
- Fix: `opencode models | grep -Fx '<provider>/<id>'` and replace it. Do not "fix" it by guessing a tier name.

**Symptom:** the agent returns empty, or errors reference a config/credential the user never configured.
- Cause: the target CLI is reading its fallback config dir, not the service's.
- Fix: export the workspace env var in the relay; confirm with the target's own `doctor`.

**Symptom:** `ModuleNotFoundError` for a plugin/MCP dependency in the relay output.
- Cause: the config being read is not the one you think. Re-verify which config file the CLI prints.
- Fix: fix the workspace export. Do not install packages into the wrong Python until you have confirmed which config is live.

**Symptom:** first call takes minutes, later calls take ~1s.
- Cause: local model cold load. Expected on CPU inference, not a bug.
- Fix: keep the model resident (a small second model used for classification/heartbeats will do it), or warn the user that first calls are slow. Do not "optimize" by silently swapping to a weaker model.

**Symptom:** reply takes minutes even when warm, only in the bridged path.
- Cause: the bridge is starting the target agent's heavy plugin/MCP set on each call, or two model calls are serializing.
- Fix: check the target agent's log for per-call startup, and give the bridge its own lightweight provider for any preflight/classification step.

**Symptom:** the agent is not in the picker at all.
- Cause: `mode: subagent` (or omitted) instead of `primary`.
- Fix: set `mode: primary`, restart the OpenCode session, and re-check `opencode agent list`.

## Best practices

- Keep the relay stateless. No temp files, no session carry-over between turns.
- Make every path overridable via env var, but default to the *service's* path.
- Prefer a confirmed-cheap model id for the relay itself; the relay is orchestration, not reasoning.
- Never let the relay answer on the agent's behalf. Verbatim relay, or an error.
- Re-run Step 5 after any upgrade of either CLI.

## See also

- [../browser-automation-agent/SKILL.md](../browser-automation-agent/SKILL.md) — driving a browser CLI from an agent, same relay shape.
- [../using-telegram-bot/SKILL.md](../using-telegram-bot/SKILL.md) — running a local agent as a chat bot.
