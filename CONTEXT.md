# CONTEXT

Maintainer/agent-facing orientation for this repo. For the public-facing
overview, CI details, and branch-protection status, see `README.md`. For
the agent-invocation workflow, see
`.github/skills/guardagent-pentest/SKILL.md`.

## What this repo is — and isn't

This repo is a **static, offline guardrail evaluator** for agentic-CLI
poison-prompt detection. It loads a bank of benign and poisoned prompts,
runs each one through a set of regex (and optionally, semantic-embedding)
detection rules, and reports whether the verdict matched the expected
outcome. It is **not** a live red-teaming tool: it never executes a
prompt, never spawns a shell, and never calls a real Copilot CLI, Claude
Code CLI, or any other live agent process. The only thing it ever writes
is `pentest-report.json` (or a `--report-path` override), and it writes
that atomically.

## Architecture

```
prompt bank (test_cases/prompts.yaml, tagged `platform: generic|<target>`)
        │
        ▼
shared rules (rules/rules.yaml — always active, every target)
        │
        ▼  (only when --target is given)
per-target overlay (rules.d/<target>.yaml — MERGED on top, never replaces)
        │
        ▼
harness (scripts/run_pentest.py) — scopes the bank to the active target,
compiles the merged rules, evaluates every in-scope prompt, asserts every
active rule category has ≥1 exercising deny-case
        │
        ▼
report (pentest-report.json — schema-versioned, target-tagged, sha256'd)
```

`targets/targets.yaml` is the registry: it names each target
(`copilot_cli`, `claude_code_cli`, `harness_agent`), gives a one-line
description of that platform's real risk surface, and points at its
`rules_overlay` file under `rules.d/`.

## The `platform` field convention

Every case in `prompts.yaml` has an optional `platform` field.

- Omitted, or `platform: generic` — the case exercises a cross-platform
  detection category (one defined in the shared `rules/rules.yaml`) and
  runs in every scope, including the default (no `--target`) run.
- `platform: <target>` (e.g. `platform: claude_code_cli`) — the case
  uses vocabulary specific to that target's real approval/injection
  surface, and is only included when `--target <target>` is passed
  (which also merges that target's overlay, without which the case
  cannot pass).

See README.md's "Adding a New Test Case" / "Adding a New Detection Rule"
for the step-by-step of tagging a new case and choosing generic vs. an
overlay.

## Hard invariants — must never break

- **No live CLI calls.** The harness never shells out to Copilot CLI,
  Claude Code CLI, or any other agent process.
- **No command execution.** Evaluation is regex/embedding matching
  against prompt text only.
- **Atomic report writes.** `atomic_write_json()` (temp file +
  `os.replace()`) — a partial write must never become a stale artifact.
- **`assert_rule_coverage()` must pass** for every scope — every active
  rule category (shared + merged overlay) must have at least one
  exercising deny-case in that scope's bank, or the harness fails fast.
- **The default run (no `--target`) stays `generic`-only and green.**
  Platform-specific cases are excluded from the default scope precisely
  because they can only pass with their overlay merged in; including
  them by default would guarantee false failures.

## File map

The contract-relevant surfaces — the ones this file's invariants are
about:

```
.github/skills/guardagent-pentest/
  SKILL.md                    — agent-facing invocation workflow
  targets/targets.yaml        — target registry (display_name, description, rules_overlay)
  rules/rules.yaml            — shared regex rules, active for every target
  rules/semantic_rules.yaml   — optional paraphrase/embedding layer (--enable-semantic)
  rules.d/<target>.yaml       — per-target overlay, merged on top of rules/rules.yaml
  test_cases/prompts.yaml     — the prompt bank (platform: generic|<target> per case)
scripts/run_pentest.py        — the harness: scope, merge, evaluate, report
.github/workflows/guardrail-pentest.yml — CI: 4-way matrix + optional semantic job
```

For everything else (docs/, requirements files, IMPLEMENTATION_SUMMARY.md)
see README.md's "Repository Structure" tree — no need to maintain the
same listing in two places.

## Where to look next

- `README.md` — public-facing overview, the "Targets" section, CI
  pipeline details, and the applied branch-protection rule.
- `.github/skills/guardagent-pentest/SKILL.md` — what an invoking agent
  should actually do, step by step.
