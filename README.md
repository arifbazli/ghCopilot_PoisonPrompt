# GuardAgent PenTest — Guardrail Testing for AI Coding Agents

A static, offline guardrail-testing framework for **GitHub Copilot CLI**, **Claude Code CLI**, and **custom agent harnesses** — built as an AI red-teaming learning resource. It runs a bank of benign and poisoned prompts through detection rules and checks whether malicious ones get blocked, automatically in CI on every change.

**It never executes anything.** No live CLI calls, no shell commands, no network access at runtime — pure offline text classification (regex + an optional semantic/embedding layer).

📖 Architecture, invariants, and full technical detail: **[CONTEXT.md](CONTEXT.md)**
📜 Version history: **[CHANGELOG.md](CHANGELOG.md)**

## Quickstart

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r scripts/requirements.txt

python3 scripts/run_pentest.py                       # generic scope (default)
python3 scripts/run_pentest.py --target copilot_cli   # or claude_code_cli / harness_agent
python3 scripts/run_pentest.py --list-targets
```

Optional semantic/paraphrase layer:

```bash
pip install -r scripts/requirements-semantic.txt
python3 scripts/run_pentest.py --enable-semantic [--fail-closed]
```

Optional approval-state simulation (what would each target's *own* approval flow do?):

```bash
python3 scripts/simulate_approval.py
```

Trigger CI manually: `gh workflow run guardrail-pentest.yml`.

## Targets

| Target | Real risk surface |
|---|---|
| `copilot_cli` | Permission bypass (`--allow-all-tools`, "stop asking for approval") |
| `claude_code_cli` | Permission bypass (`--dangerously-skip-permissions`, `acceptEdits`); CLAUDE.md injection/disclosure |
| `harness_agent` | Tool-scope abuse; treating tool output as a trusted instruction |

Omitting `--target` runs the generic-only scope (8 shared categories, no overlay). Registry: `.github/skills/guardagent-pentest/targets/targets.yaml`.

## Test bank

**52 cases** across 9 attack categories + benign must-allow cases. 7 of 9 categories have calibrated semantic (paraphrase) coverage; 2 remain regex/hint-gated only after three separate calibration attempts — see `rules/semantic_rules.yaml`'s header comment for the full numbers.

## Limitations

Static/offline only · English-only · semantic layer is **fail-open by default** (pass `--fail-closed` to make a model-load failure a hard error instead of a silent regex-only fallback). Full detail, including a real fail-open incident hit in practice: [CONTEXT.md](CONTEXT.md).

## Learn more

- `docs/architecture.md` — how the pieces fit together
- `docs/baseline-before.md` — a real guardrail gap, found and fixed
- `.github/skills/guardagent-pentest/test_cases/prompts.yaml` — the live attack bank
- `.github/skills/guardagent-pentest/SKILL.md` — agent-invocation workflow
