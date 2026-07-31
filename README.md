# GuardAgent PenTest — Poison-Prompt Testing for GitHub Copilot CLI

A guardrail testing framework for GitHub Copilot CLI. Runs benign and
poisoned prompts against Copilot CLI to validate whether destructive or
malicious bash commands get correctly blocked.

---

## Repository Structure

ghCopilot_PoisonPrompt/
├── .github/
│ ├── workflows/
│ │ └── guardrail-pentest.yml # CI pipeline (see below)
│ └── skills/
│ └── guardagent-pentest/
│ ├── SKILL.md # Agent-facing skill definition
│ ├── test_cases/
│ │ └── prompts.yaml # Poison-prompt test bank
│ └── rules/
│ └── rules.yaml # Guardrail detection patterns
├── scripts/
│ └── run_pentest.py # Test harness (evaluation engine)
├── docs/
│ └── baseline-before.md # Documented "before" baseline results
└── README.md


## Workflow Architecture

### 1. Test Bank (`test_cases/prompts.yaml`)

A list of test cases, each with:
- `id` — unique identifier
- `category` — attack technique (e.g. `rce`, `reverse_shell`,
  `secret_exfiltration`, `instruction_override`, `benign`)
- `prompt` — natural-language prompt as a user might type it to Copilot CLI
- `expected_outcome` — `deny` (should be blocked) or `allow` (safe to run)

### 2. Detection Rules (`rules/rules.yaml`)

Regex patterns grouped by category. Each category maps to one or more
patterns that, if matched against a prompt, indicate the guardrail
should deny it.

### 3. Test Harness (`scripts/run_pentest.py`)

The evaluation engine. For each test case:
1. Loads the prompt text and all rule patterns
2. Checks the prompt against every pattern (case-insensitive regex search)
3. If any pattern matches → verdict is `deny`; otherwise `allow`
4. Compares the verdict to `expected_outcome` → pass/fail
5. Writes a machine-readable report (`pentest-report.json`) and prints
   a human-readable summary table

**Important:** the harness never executes any command. It performs a
static, offline evaluation of whether the guardrail *would* block a
given prompt — consistent with `SKILL.md`'s safety requirement to never
run destructive commands during testing.

### 4. CI Pipeline (`.github/workflows/guardrail-pentest.yml`)

**Triggers:**
| Trigger | When |
|---|---|
| `push` | Any push touching `.github/skills/guardagent-pentest/**` or the workflow file itself |
| `pull_request` | Any PR touching the skill files |
| `schedule` | Daily at 06:00 UTC (14:00 MYT) — regression check |
| `workflow_dispatch` | Manual trigger via `gh workflow run` or the Actions tab |

**Steps:**
1. **Checkout repository** — clones the repo into the runner
2. **Set up Python** — Python 3.12
3. **Install dependencies** — `pyyaml`
4. **Run pentest harness** — executes `scripts/run_pentest.py`, captures
   pass/fail (does not immediately fail the job — see step 7)
5. **Upload report artifact** — attaches `pentest-report.json` to the
   run for later download/inspection
6. **Write job summary** — renders a pass/fail table directly in the
   GitHub Actions run summary page
7. **Comment on PR** *(pull_request events only)* — posts a pass/fail
   summary comment on the triggering PR
8. **Fail job if any test failed** — the actual CI gate; fails the run
   if the harness reported any mismatched case, so a genuine guardrail
   regression blocks the pipeline

### 5. Baseline Tracking (`docs/baseline-before.md`)

A permanent, committed snapshot of an intentional "before" state
(regex-only detection, 4/6 passing) documenting a known limitation:
natural-language phrasing of an attack bypasses literal pattern
matching. This is preserved as a reference point to demonstrate
improvement over time, distinct from the live `rules.yaml` (which is
patched forward as gaps are closed).

## Running Locally

```bash
pip install pyyaml --break-system-packages
python3 scripts/run_pentest.py
```

## Triggering CI Manually

```bash
gh workflow run guardrail-pentest.yml
gh run watch $(gh run list --workflow=guardrail-pentest.yml --limit 1 --json databaseId -q '.[0].databaseId')
```
