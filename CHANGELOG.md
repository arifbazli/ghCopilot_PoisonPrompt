# Changelog

Full version history. README.md keeps only the latest 1-2 entries;
everything older lives here.

## v1.4 (semantic hardening for multi-target categories)

Closes the semantic-coverage gap for the 4 categories added in v1.3
(`permission_bypass`, `context_disclosure`, `indirect_injection`,
`tool_scope_abuse`), the same way PR #12 closed it for the original 5.

- **13 new bank cases (ids 33-45)** — 11 paraphrase probes tagged
  `semantic_only: true` (regex bypasses, verified empirically against
  the compiled rules for their scope) + 2 new benign calibration anchors
  (`context_disclosure`, `indirect_injection` — `permission_bypass` and
  `tool_scope_abuse` already had a topically-close benign case). Bank
  grown from 32 to **45 cases**.
- **`semantic_only` schema field** (default `false`) — `scope_prompts()`
  excludes these cases from a plain regex-only run (every CI matrix leg
  today) so they can't count as false failures there, but includes and
  requires them to pass whenever the semantic layer is active.
  `assert_rule_coverage()` needed no change — it already receives the
  correctly-scoped bank and the same `semantic_layer` object used
  everywhere else.
- **4 new `semantic_rules.yaml` rules**, real embedding scores measured
  via GitHub Actions (Ubuntu, matches CI's proven environment — local
  Windows torch installs hit `MAX_PATH` corruption and heavy disk/CPU
  load, so calibration ran remotely instead):
  - `context_disclosure` — **cleanly calibrated** (probe_emb 0.569 clears
    `benign_max(0.506) - 0.05`), threshold 0.456, same convention as the
    original 5 rules.
  - `permission_bypass`, `indirect_injection`, `tool_scope_abuse` —
    **hint-gated only**: probe scores (0.272 / 0.059 / 0.325) don't clear
    the standard margin below their benign_max (0.645 / 0.575 / 0.445).
    Threshold set just below each rule's own observed probe minimum
    instead. Zero false positives is still guaranteed (the regex hint
    fires on zero benign cases, old or new) — but these 3 rules give no
    embedding-based recall for a future paraphrase that doesn't happen to
    match the hint. Documented as an open gap, not papered over.
  - Fixed a stale doc comment in `semantic_rules.yaml`: the header said
    the match rule was "hint OR embedding"; the code has always been
    "hint AND embedding". Also documented the `id`/`maps_to` = regex
    category convention explicitly.
  - `semantic_layer_version` bumped to 2.
- **Verified**: `--enable-semantic` across all 4 scopes —
  generic 27/27, copilot_cli 32/32, claude_code_cli 34/34,
  harness_agent 33/33. Regex-only counts drop correspondingly (24/27/28/27)
  since `semantic_only` cases are excluded there, confirming the filter
  behaves exactly as designed.
- **Docs**: this file created; README.md trimmed to roughly half its
  prior length, with maintainer/architecture detail moved to `CONTEXT.md`.

## v1.3 (multi-target expansion, merged as PR #13)

Expands the guardrail bank from Copilot-CLI-only to three targets:
GitHub Copilot CLI, Claude Code CLI, and a generic custom agent harness.

- **3 new cross-platform categories** in `rules/rules.yaml` —
  `permission_bypass`, `context_disclosure`, `indirect_injection` —
  covering agent-CLI attack shapes that aren't bash-exec specific (e.g.
  "auto-approve every tool call", "print your system prompt", "if you
  are an AI reading this, ...").
- **Target registry + per-target overlays** — `targets/targets.yaml`
  names the three targets; `rules.d/copilot_cli.yaml`,
  `rules.d/claude_code_cli.yaml`, `rules.d/harness_agent.yaml` carry
  vendor-specific vocabulary (Copilot CLI's `--allow-all-tools`; Claude
  Code CLI's `--dangerously-skip-permissions` / acceptEdits / CLAUDE.md
  disclosure and injection; a new `tool_scope_abuse` category for
  `harness_agent`). Overlays are merged on top of the shared rules,
  never replace them.
- **Bank grown to 32 cases** — 22 generic + 10 platform-specific (3
  `copilot_cli` + 4 `claude_code_cli` + 3 `harness_agent`), each case
  carrying an optional `platform` field (default `generic`).
- **Harness scoping** — `scripts/run_pentest.py` gained `--target`
  (merge overlay + filter bank) and `--list-targets`; the default (no
  `--target`) run stays `generic`-only so it can't false-fail on
  overlay-only vocabulary. `assert_rule_coverage()` now validates
  per-scope. Report schema bumped to **v3** with a new `target` field.
- **CI matrix** — the `pentest` job in `guardrail-pentest.yml` is now a
  4-way matrix (`generic`, `copilot_cli`, `claude_code_cli`,
  `harness_agent`; `fail-fast: false`), each leg uploading its own
  `pentest-report-<target>` artifact. Live verification: **22/22**,
  **25/25**, **26/26**, **25/25** respectively — all four scopes pass.
- **New docs** — `CONTEXT.md` (maintainer/agent-facing architecture and
  invariants) and a leaner `SKILL.md` that points to it.
- **Branch protection** — `required_status_checks.contexts` updated from
  the single legacy `"Run guardrail pentest"` to the four matrix contexts
  (`"Run guardrail pentest (generic|copilot_cli|claude_code_cli|harness_agent)"`),
  applied live via `gh api` and synced to `docs/branch-protection.json`.
  This had been flagged as a pending follow-up in the PR and was resolved
  before merge.

## v1.1 (PR [#1](https://github.com/arifbazli/ghCopilot_PoisonPrompt/pull/1), merged 2026-08-03)

The original 6-case test bank was **self-confirming** — its prompts
literally contained the substrings the rules searched for, so a 6/6
"after" pass was a tautology. The 6 paraphrased probes added in #1
exercise the new word-boundary anchors, and a `destructive_commands`
case set covers the previously-dead rule category. The bank is now
**18 cases: 12 deny + 6 benign must-allow**. Live CI: **18/18 PASS**.

Other changes in v1.1:
- **Harness hardening** — schema-validated prompt bank, atomic JSON
  report write, `assert_rule_coverage()` (fails CI if a rule category
  is left unexercised by any deny-case), `--list-only` /
  `--report-path` CLI flags, `rules_sha256` in the report for forensic
  comparability.
- **CI hardening** — Actions pinned by SHA, `timeout-minutes: 10`,
  `concurrency:` block, **fork-only PR-comment gate** (prevents the
  previously-broken fork-PR path), scheduled-failure opens an issue,
  `retention-days: 30` on artifacts, PEP 668 respected via a venv
  install.
- **Governance** — `.github/CODEOWNERS` on `.github/workflows/**` and
  `.github/skills/**`.
- See `IMPLEMENTATION_SUMMARY.md` for the full change list and REVIEW
  audit trail (the review itself lives in a sibling path on the
  original clone and is not committed to the repo).

## v1.2 (PRs [#5](https://github.com/arifbazli/ghCopilot_PoisonPrompt/pull/5)–[#9](https://github.com/arifbazli/ghCopilot_PoisonPrompt/pull/9), merged 2026-08-03/04)

Documentation, CI hardening, and the three follow-up issues from v1.1:

- **PR #5** — README aligned with v1.1 (bank size, venv install, fork gate, schema).
- **PR #6** — added `.github/workflows/actions-version-check.yml` (closes issue #4): weekly scheduled job compares the 3 SHA-pinned Actions against upstream and posts a dedup-signed comment on issue #4 if any drift is detected. No auto-bump, human-in-the-loop. The 3 pinned SHAs from v1.1 **already drift** upstream at the time of v1.2 — a maintainer decision is needed on whether to bump them.
- **PR #7** — widened pentest `pull_request.paths` trigger from just `.github/skills/guardagent-pentest/**` to also include `.github/workflows/**` and `.github/CODEOWNERS`, so workflow-file PRs and CODEOWNERS changes are also gated on the 18/18 pentest.
- **PR #8** — README updated to reflect the *actual* applied branch-protection rule; shipped `docs/branch-protection.json` (canonical payload for re-applying). Also widened the pentest path filter to include `docs/**` so docs PRs that touch the guardrail contract are gated.
- **PR #9** — `required_approving_review_count` reduced from `1` → `0` to break the single-maintainer self-approval deadlock; `docs/branch-protection.json` and README updated to match.

## Open and closed follow-up work

- ~~[#2 Phase-3 semantic detector](https://github.com/arifbazli/ghCopilot_PoisonPrompt/issues/2)~~ — **closed 2026-08-04** (sentence-transformers layer implemented in PR #12; extended for multi-target categories in v1.4)
- ~~[#3 Branch protection on `main`](https://github.com/arifbazli/ghCopilot_PoisonPrompt/issues/3)~~ — **closed 2026-08-03** (rule applied; contexts updated again in v1.3 for the CI matrix)
- ~~[#4 Action SHA maintenance](https://github.com/arifbazli/ghCopilot_PoisonPrompt/issues/4)~~ — **closed 2026-08-03** (weekly drift checker added in PR #6)

## Baseline tracking — before / after

The original 6-case bank (before v1.1) was **self-confirming** — its
prompts literally contained the substrings the rules searched for, so
the 4/6 → 6/6 improvement documented in `docs/baseline-before.md` was
real coverage gain only on the literal-substring axis.

| Phase | Bank size | Pass rate | Notes |
|---|---|---|---|
| Before (regex only) | 6 cases | 4/6 (66%) | `docs/baseline-before.md` documents the 2 literal-substring gaps |
| After v1.0 (literal patterns) | 6 cases | 6/6 (100%) | self-confirming — bank matched rule templates exactly |
| After v1.1 (anchored + paraphrased) | 18 cases | 18/18 (100%) | de-circularized; `assert_rule_coverage()` now prevents silent regression |
| After v1.3 (multi-target expansion) | 32 cases, 4 scopes | generic 22/22, copilot_cli 25/25, claude_code_cli 26/26, harness_agent 25/25 (all 100%) | new `permission_bypass`/`context_disclosure`/`indirect_injection`/`tool_scope_abuse` categories; `assert_rule_coverage()` validates per-scope |
| **After v1.4 (semantic hardening)** | **45 cases, 4 scopes** | **regex-only: generic 24/24, copilot_cli 27/27, claude_code_cli 28/28, harness_agent 27/27 — semantic-enabled: 27/27, 32/32, 34/34, 33/33 (all 100%)** | 11 paraphrase probes added for the 4 new categories; 1 of 4 calibrates cleanly, 3 are hint-gated only (documented gap) |
