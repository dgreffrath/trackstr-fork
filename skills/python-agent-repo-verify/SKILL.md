---
name: python-agent-repo-verify
description: "Verifies a Python agent/CLI repo is truly installable and working: venv deps, test suite, and a live entrypoint run, fixing the common blockers it finds. Use when: (1) User asks to 'check/confirm X is installed and working', (2) Auditing a cloned Python repo before trusting it, (3) User explicitly asks."
---

# Python Agent Repo Verify

Proves a cloned Python repo (agent frameworks, CLIs, SDKs) actually works end to end: dependencies resolve, the test suite passes, and the main entrypoint runs. Fixes the predictable blockers instead of just reporting them.

## Quick quality checklist

- `name` matches folder name exactly (kebab-case)
- All examples are tested and runnable
- Uses free/public tools first (or explains paid fallback)
- No secrets, API keys, or personal data in examples

## When to use

- Use case 1: "Check and confirm <repo> is installed and working 100%"
- Use case 2: Before relying on a forked/patched repo in production
- Use case 3: Auditing a repo after a machine reinstall or version bump

## Required tools / APIs

- `uv` (recommended) — installs isolated py-version-correct deps without conda

```bash
# Ubuntu/Debian
curl -LsSf https://astral.sh/uv/install.sh | sh

# macOS
brew install uv
```

- For agent repos configured against a local LLM (e.g. Ollama at `localhost:11434`), confirm the model endpoint responds before running tests that mutate nothing.

## Skills

### basic_usage

Five-step verify loop, run from the repo root.

```bash
# 1. Deps: let uv build/maintain the project venv at the pinned Python
uv sync --frozen            # or: uv run --frozen python -c "import <pkg>"

# 2. Test deps are often NOT in the runtime lockfile — add pytest handlers
#    into the venv without touching the lock:
uv pip install pytest pytest-asyncio --python .venv/bin/python

# 3. Run the suite with the venv Python (never bare `pytest`, it may pick
#    the system Python and fail imports):
PYTHONPATH=. .venv/bin/python -m pytest tests/ -q

# 4. Import-level sanity of the real entrypoint (no network side effects):
.venv/bin/python -c "import main"   # or the repo's documented module

# 5. Live smoke run when an LLM/local service is configured:
printf 'Reply OK\n' | timeout 300 .venv/bin/python main.py 2>&1 | tail -20
```

### robust_usage

Triage the failures this skill exists for. Each is a known recurring pattern.

```bash
# Pattern A — bare `pytest` uses system Python 3.13/3.14 -> ModuleNotFoundError
# Fix: always invoke through the venv interpreter with PYTHONPATH=. (step 3)

# Pattern B — tests crash in fake/mock subclasses of vendored client
#   classes (e.g. a `FakeSession(ClientSession)` that assigns
#   `self.instructions = ...`) because a newer `mcp`/SDK major turned that
#   attribute into a read-only property.
# Diagnose: `uv pip list --python .venv/bin/python | grep -E "<sdk>"` vs the
#   pin in requirements.txt. If the pin is stale, prefer minimal test fixes:
#   in FakeSession, shadow at class level so the property is overridden:
#     instructions = None

# Pattern C — flaky sandbox/Docker tests that pass solo but fail in-suite
#   (resource contention). Rerun the failing node alone before "fixing".
PYTHONPATH=. .venv/bin/python -m pytest tests/sandbox/test_client.py::test_x -q

# Pattern D — stale assertions about base images (e.g. `assert "Python 3.10"
#   in result` against an image now shipping 3.12). Loosen the assertion:
sed -i 's/assert "Python 3.10" in result/assert "Python 3." in result/' tests/sandbox/test_*.py
```

**Verify the backing model/service before blaming the repo:**

```bash
curl -s --max-time 10 http://localhost:11434/v1/models | grep -o '"id":"[^"]*"'
curl -s --max-time 90 http://localhost:11434/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"<model>","messages":[{"role":"user","content":"Reply with the word OK."}],"max_tokens":10}' \
  | grep -o '"content":"[^"]*"'
```

**Node.js (when the repo is JS/TS instead):**

```javascript
const out = await sh(`npm test --silent 2>&1 | tail -20`);
// npm test exit code + tail decide pass/fail; never trust a silent pass.
```

## Output format

- `gates`: array of {step (deps/tests/import/live), status passed|fixed→passed|failed, evidence (one line: test totals, token usage, exit code)}
- `fixes`: for each code change: file, one-line diff summary
- `verdict`: working | working-with-fixes | broken (with the remaining blocker)
- Error shape: `{gate, error, actionable-fix}`

## Rate limits / Best practices

- Timebox live LLM runs (`timeout 300`); a looping small local model is evidence the *pipeline* works, not that the model is good — report both separately.
- Long test suites (~2 min) are normal for repos sandboxing code in Docker; do not shorten the timeout below ~10 min.
- Never chase flaky acquires with parallel reruns; run the failing node solo first.
- Do not print `api_key` values when inspecting `config.toml` — print `= [REDACTED present]` only.
