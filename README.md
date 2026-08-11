# GuardAgent PenTest — Poison-Prompt Testing for GitHub Copilot CLI, Claude Code CLI, and Custom Agent Harnesses

A guardrail testing framework for GitHub Copilot CLI, Claude Code CLI,
and generic custom agent harnesses, built as a learning resource for
**AI penetration testing (AI red-teaming)**: deliberately attacking an
AI system to find where its safety controls break, before a real
attacker does. AI coding agents read files, run commands, and act on
natural-language instructions — a malicious or careless prompt (a
**poison prompt**, via **prompt injection**) can trick one into running
a destructive command, leaking a secret, or opening a backdoor. The
defensive layer that catches this is a **guardrail**.

Runs a bank of **45 benign and poisoned prompts** against detection-rule
guardrail logic across the 3 targets below, and does it automatically in
CI, so every change to the detection rules is checked against the full
attack bank before it ships.

For the maintainer/agent-facing architecture, invariants, and full
detection-rule reference, see `CONTEXT.md`. For version history, see
`CHANGELOG.md`.

If you're new to this space, start with:
1. `docs/architecture.md` — how the pieces fit together
2. `.github/skills/guardagent-pentest/test_cases/prompts.yaml` — the live 45-case attack bank
3. `docs/baseline-before.md` — a real example of a guardrail gap being found and fixed

---

## Targets

Three targets are supported. Each names a real agentic-CLI (or harness)
surface with its own approval/injection vocabulary; the 8 shared
`rules/rules.yaml` categories (`destructive_commands`,
`instruction_override`, `remote_code_execution`, `reverse_shell`,
`secret_exfiltration`, `permission_bypass`, `context_disclosure`,
`indirect_injection`) apply to every target unconditionally.

| Target | Display name | Real risk surface | Overlay |
|---|---|---|---|
| `copilot_cli` | GitHub Copilot CLI | `--allow-all-tools` / "stop asking for approval" style permission bypass | `rules.d/copilot_cli.yaml` |
| `claude_code_cli` | Claude Code CLI | `--dangerously-skip-permissions` / bypassPermissions mode / acceptEdits permission bypass; CLAUDE.md-based indirect injection (auto-read every session); disclosure of CLAUDE.md / tool schemas | `rules.d/claude_code_cli.yaml` |
| `harness_agent` | Generic custom agent harness | Tool-scope abuse (using a tool outside its intended directory/domain); indirect injection via tool output treated as a trusted instruction | `rules.d/harness_agent.yaml` |

The registry lives in `targets/targets.yaml`; run
`python3 scripts/run_pentest.py --list-targets` to print it.

**How `--target` works:**

```bash
python3 scripts/run_pentest.py                          # generic scope only (default)
python3 scripts/run_pentest.py --target copilot_cli      # + copilot_cli overlay + cases
python3 scripts/run_pentest.py --target claude_code_cli  # + claude_code_cli overlay + cases
python3 scripts/run_pentest.py --target harness_agent    # + harness_agent overlay + cases
```

Passing `--target <name>` merges that target's `rules.d/<name>.yaml`
overlay on top of the shared `rules/rules.yaml` (categories are unioned;
an existing category's patterns are extended, never replaced) and scopes
the prompt bank to cases tagged `platform: generic` or `platform: <name>`.

**Omitting `--target` runs the generic-only scope**: the shared rules
with no overlay, and only `platform: generic` cases — `semantic_only`
paraphrase-probe cases are excluded too unless the semantic layer is
active (see `CONTEXT.md`). This is deliberate: platform-specific and
semantic-only cases can only pass with their overlay/layer active, so
including them by default would guarantee false failures. All 4 CI
matrix legs (`generic` + the 3 targets) are required status checks on
`main`; see `CONTEXT.md` for the branch-protection settings.

---

## Recent changes

### v1.4 — semantic hardening for multi-target categories

Closes the semantic-coverage gap for `permission_bypass`,
`context_disclosure`, `indirect_injection`, `tool_scope_abuse` (same as
PR #12 did for the original 5 categories). 13 new bank cases (11
`semantic_only` paraphrase probes + 2 benign calibration anchors), 4 new
`semantic_rules.yaml` rules — 1 calibrates cleanly, 3 are hint-gated only
(documented in Limitations below). Bank now **45 cases**. Full detail
and calibration numbers: `CHANGELOG.md`.

### v1.3 — multi-target expansion

Expanded from Copilot-CLI-only to 3 targets (GitHub Copilot CLI, Claude
Code CLI, generic custom agent harness): 3 new cross-platform
categories, per-target `rules.d/` overlays, `--target` scoping, and the
4-way CI matrix. Full detail: `CHANGELOG.md`.

Older versions (v1.1, v1.2) and the baseline before/after progression
table: see `CHANGELOG.md`.

---

## Quickstart

```bash
# one-time setup
python3 -m venv .venv
. .venv/bin/activate
pip install -r scripts/requirements.txt

# run the harness (generic scope — same as CI's default leg)
python3 scripts/run_pentest.py

# or scope to a target (merges its rules.d/ overlay + cases)
python3 scripts/run_pentest.py --target copilot_cli
python3 scripts/run_pentest.py --target claude_code_cli
python3 scripts/run_pentest.py --target harness_agent

# list the target registry, or just the bank for the current scope
python3 scripts/run_pentest.py --list-targets
python3 scripts/run_pentest.py --list-only

# optional semantic/paraphrase layer (~250 MB install; see CONTEXT.md)
pip install -r scripts/requirements-semantic.txt
python3 scripts/run_pentest.py --enable-semantic
```

Local runs are byte-identical to CI: same Python 3.12, same venv install
path, same `SALUS_PENTEST_STATIC=1` env (set it yourself to simulate the
CI step exactly). Trigger CI manually with
`gh workflow run guardrail-pentest.yml`.

Adding a test case or detection rule, the CI pipeline's steps, and the
applied branch-protection settings are all documented in `CONTEXT.md`.

## Limitations (and what this repo does NOT do)

- **Semantic coverage is partial, by design and by measurement.** The
  semantic layer (`--enable-semantic`) catches paraphrases the regex
  layer misses, but only 6 of the 9 attack categories
  (`instruction_override`, `destructive_commands`, `secret_exfiltration`,
  `remote_code_execution`, `reverse_shell`, `context_disclosure`)
  calibrate cleanly — probe scores clear a real margin above
  topically-similar benign text, giving embedding-based recall against a
  brand-new paraphrase. The other 3 (`permission_bypass`,
  `indirect_injection`, `tool_scope_abuse`) are **hint-gated only**: zero
  false positives is still guaranteed (the regex hint fires on zero
  benign cases), but there's no embedding-based recall for a paraphrase
  that doesn't happen to match the hint. Full calibration table and
  per-rule reasoning: `rules/semantic_rules.yaml`'s header comment.
- **No LLM in the loop.** Pure offline static evaluation — never calls
  Copilot CLI, Claude Code CLI, or any other live agent process, never
  executes a prompt, no API keys required.
- **No mutation of external state.** The only filesystem write is
  `pentest-report.json` (atomic); the semantic layer also writes to the
  gitignored `.cache/semantic-refs/` embedding cache.
- **English-only.** Patterns and references are tuned for English;
  multilingual paraphrases aren't covered.
- **Semantic layer is fail-open.** A model load failure (network error,
  disk space) prints a warning and falls back to regex-only verdicts —
  a deliberate choice for this learning repo; a production fork should
  override with fail-closed.
