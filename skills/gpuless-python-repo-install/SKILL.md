---
name: gpuless-python-repo-install
description: "Install Python research/AI repos (PyTorch-era, GPU- and conda-assumed READMEs) on a GPU-less Linux host using uv instead of conda/pip, then record every deviation with evidence. Use when: (1) a repo README says 'conda install / pip install / needs NVIDIA GPU' and your host has none, (2) setup.py and pyproject.toml disagree on pins, or (3) a user explicitly asks."
---

# GPU-less Python repo install (uv, no conda)

Reproduces a research repo's documented install flow on a machine with no
GPU, no conda, and no pip binary in venvs — using uv. Deliverables are two:
a verified install (imports resolve, data files counted, critical pins
intact) and a `SETUP_DEVIATIONS.md` where every read-the-books deviation
carries evidence. Expected outcome: the env runs everything CPU-bound the
README promises; anything GPU-bound is listed as a hard blocker, never
attempted blindly.

## Quick quality checklist

- `name` matches folder name exactly (kebab-case)
- Every command in this file was copy-paste tested against a real install
- Shell/Bash plus Python snippets only — Node.js scripts do not apply here
- No secrets, API keys, tokens, or personal host paths in examples
- Free/public tools only (uv is Apache-2.0, Temurin JDK is free)

## When to use

- Use case 1: README says `setup.sh` / conda env / `pip install -e .` / CUDA,
  and the host is GPU-less CPU-only Linux.
- Use case 2: repo has both `setup.py` (newer pins) and `pyproject.toml`
  (stale pins) and the resolver rejects them against each other.
- Use case 3: dataset download commands point at dead/private links
  (expired Google Drive IDs, 404s) or require a conda-provided Java.

## Required tools / APIs

- **uv** — venv + resolver instead of conda/pip. No external API needed.
- **Temurin 21 JDK** — only if the stack needs modern Java (see robust_usage);
  free. Untar under `~/.local/opt/`.
- **huggingface_hub** — for dataset mirrors, when original hosts are dead.

Install options:

```bash
# uv (Linux/macOS)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Temurin 21 JDK (only if jdk.incubator.vector is required)
# grab from adoptium.net, then untar:
mkdir -p ~/.local/opt && tar -xzf OpenJDK21U-jdk_x64_linux_*.tar.gz -C ~/.local/opt/
```

## Skills

### basic_usage

The linear happy path. Read the docs before touching anything.

```bash
# 1. Read first: README.md, setup.py, setup.sh (they rarely agree)
# 2. Create the venv with uv (no conda)
cd ~/projects/<repo>
~/.local/bin/uv venv

# 3. Install the repo's own pins (installs build deps fast, no pip binary needed)
~/.local/bin/uv pip install --python .venv/bin/python -e ".[extras-you-can-afford]"
# Omit GPU extras (e.g. flash-attn groups) on GPU-less hosts — gate them below.

# 4. Smoke-test: versions happened to land intact?
.venv/bin/python - <<'EOF'
import importlib.metadata as md
for pkg in ['torch', 'transformers', 'numpy']:
    print(pkg, md.version(pkg))
EOF

# 5. Confirm rw-data commands exist and what they download
.venv/bin/<repo>-download --help
```

**Python** (pin snapshot helper — verify nothing critical got
downgraded as a side-effect of extra deps):

```python
# scripts/check_pins.py — compare md.version(pkg) vs. a dict of required pins
import importlib.metadata as md

required = {'torch': '2.6.0', 'transformers': '4.51.1', 'numpy': '1.26.4'}
for pkg, want in required.items():
    got = md.version(pkg)
    assert got == want, f'{pkg}: want {want}, got {got}'
print('pins intact')
```

### robust_usage

The hard-won steps. Each one exists because the naive path failed.

**(a) setup.py is single source of truth.** Research repos often carry a
stale `pyproject.toml` (e.g. `transformers<4.48`) that contradicts
`setup.py` (e.g. `transformers<=4.51.1`) and uv refuses the mix. Keep one
pin source:

```bash
# mark deps dynamic in pyproject.toml; comment out their static lists
# before: [project] dependencies = [...]
# after:  dynamic = ["dependencies", "optional-dependencies"]
```

**(b) setuptools<81 stays, if pkg_resources is imported.** `verl`-era
projects import `pkg_resources` at runtime; setuptools 81+ removed it.
Pin explicitly:

```bash
~/.local/bin/uv pip install --python .venv/bin/python 'setuptools<81'
```

**(c) Snapshot the working solution as constraints.** Once the reference
install works, freeze it — exact versions, not bounds — so a later
reinstall is reproducible:

```bash
~/.local/bin/uv pip freeze --python .venv/bin/python > constraints.txt
```

**(d) Hardware gate: try cheaply, else document.** On a GPU-less host,
never start a build that requires CUDA (they detect and fail only after a
long build). Attempt them in a throwaway venv quickly or read if the
package gate depends on `NVCC`/`CUDA_HOME`; then record as hard blockers.
Same for GPU-only tests: run the CPU subset, record which tests fail
because there is no driver — with the failing test names.

**(e) Java via jdk.incubator.vector.** pyserini/Lucene 0.39+ hits that
feature at import; Java 17 dies with a vector-feature crash. Use Temurin
21 and export before bare `python -m` scripts:

```bash
export JAVA_HOME=~/.local/opt/jdk-21.0.13+9
export PATH="$JAVA_HOME/bin:$HOME/projects/<repo>/.venv/bin:$PATH"
# conda was the upstream way to make Java resolvable; with no conda,
# PATH export is the substitute the .sh wraps need.
```

**(f) Dead data links: mirror, don't retry blindly.** If map/repo/Drive
IDs 404, do not rerun the downloader until they resolve. Find a mirror
(Hugging Face hub), download, extract, then clean the archive:

```bash
~/.local/bin/uv pip install huggingface_hub  # into the venv
huggingface-cli download <org>/<repo> <file>.tar.gz --repo-type dataset --local-dir /tmp/dl
tar -xzf /tmp/dl/<file>.tar.gz -C <repo-data-dir>/ && rm /tmp/dl/<file>.tar.gz
```

**(g) Verify data against the manifest, not against folklore.** After a
dataset script finishes, count the files and diff against the downloader's
own URL manifest:

```bash
find ~/.cache/<repo> -name '*.gamefile' | wc -l
grep -rn 'https://' $(which <repo>-download)   # URLs the tool defines
# expected output: count of [.gamefile] == manifest counts; no stale *.zip
```

**(h) Write the deviations doc.** Every step that diverted from
`setup.sh`/README — one section per subsystem with the evidence
(command + observed output), plus a hard-blocker list at the end:

```markdown
## <subsystem> (installed and verified)
... version snapshot + command evidence ...
## Hard blockers on this host (GPU-bound, do not attempt blindly)
- any training run needing an NVIDIA GPU / driver
```

## Output format

- `venv_path` — path of the .venv (string, e.g. `~/projects/<repo>/.venv`)
- `versions` — list of `name==version` after the pin check (all critical)
- `data_counts` — per dataset: expected vs. actual file/split counts
  (e.g. `train 2435/2435`)
- `deviations_doc` — path + section list of `SETUP_DEVIATIONS.md`
- `blockers` — array of strings, each naming the missing hardware and the
  exact workload it gates
- Error shape: never "done" without these six fields; a blocker omitted
  means the install is misreported.

## Rate limits / Best practices

- Do not re-download datasets that already exist in cache; `--help` the
  downloader first and use the flag that skips existing files (e.g. `-f`
  is overwrite — force only when the dir is known-corrupt).
- One subsystem per commit for the repo's own pin fixes; keep host-local
  venvs out of git (add `.venv/` to .gitignore).
- Rely on evidence in the doc — a claim without the observed line of
  output does not count as verified.
- Nothing above calls any paid API; mirror hunting uses free Hugging Face
  hosting.

## Agent prompt

```text
You have gpuless-python-repo-install capability. When a user wants to run
a research repo whose README assumes conda + NVIDIA GPU, on a CPU-only host:

1. Read README.md, setup.py, setup.sh and list every conda/GPU/pip-era
   assumption before running anything.
2. Create a uv venv; mark pyproject deps dynamic if they conflict with
   setup.py; pin setuptools<81 when pkg_resources is a runtime dep.
3. Install core deps only (omit GPU extras); verify pins with
   importlib.metadata; snapshot constraints.txt once green.
4. Any build-required CUDA dep or GPU-only test: try cheaply, then record
   as a hard blocker — with the failing test name or missing library —
   never start a long build that cannot end at `nvidia-smi` present.
5. Datasets: prefer the repo's own downloader; on dead links hunt a Hugging
   Face mirror; verify by file counts against the downloader's manifest.
6. Produce SETUP_DEVIATIONS.md: one section per subsystem (with evidence),
   plus a hard-blocker list. Do not report the task done without it.
```

## Troubleshooting

**Error scenario 1: resolver rejects pin mixes**
- Symptom: uv dies reporting e.g. `transformers<4.48` from pyproject vs
  `transformers<=4.51.1` from setup.py
- Solution: make `dynamic = ["dependencies", "optional-dependencies"]` in
  pyproject and comment out the static lists; retry.

**Error scenario 2: `ModuleNotFoundError: pkg_resources`**
- Symptom: importing the repo drops that at runtime, current setuptools
  is 80+81
- Solution: `uv pip install --python .venv/bin/python 'setuptools<81'`.

**Error scenario 3: venv has no pip binary**
- Symptom: `.venv/bin/pip list` cannot be found
- Solution: inspect via `.venv/bin/python -c "import importlib.metadata as md; md.version('pkg')"`; that is how uv venvs work.

**Error scenario 4: Java feature vector crash**
- Symptom: `UnsupportedClassVersionError` or a crash mentioning
  `jdk.incubator.vector` while indexing
- Solution: install a free Temurin 21 and `export JAVA_HOME` + PATH for
  the exact invocation. Also export the venv `bin` on PATH for bare `python -m` script use.

**Error scenario 5: "GPU-less" — CUDA crashes, libcuda.so.1 missing**
- Symptom: `torch.cuda.is_available()` is False; driver tests hang
- Solution: document as blocker (not a bug). Do not install CUDA-enabled
  backends that the CPU subset does not require.

**Error scenario 6: dataset 404 / Access Denied**
- Symptom: original dead Drive / repo URL
- Solution: mirror to Hugging Face, extract and verify file counts vs.
  the downloader manifest; delete the archive afterwards.

## See also

- [CONTRIBUTING.md](../../CONTRIBUTING.md) — contribution standards for this collection

---

## Notes

- Adapted template: works with Bash + Python (uv) snippets rather than a
  Node.js pair — Node does not apply to Python repo installs.
- The `jdk-21.0.13+9` path is a version example, not a required pin; any
  Temurin 21 patch release works.
