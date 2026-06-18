# PROGRESS — running task log

Update this file whenever a task starts, ships, or blocks. Format: status emoji + short subject + 1-line note. Keep it scannable.

## Legend

- `[x]` done
- `[~]` in progress
- `[ ]` pending
- `[!]` blocked
- `[-]` deferred / out of scope for PR 1

---

## PR 1 — eval harness (current)

### Setup phase

- `[x]` Fork `redhat-community-ai-tools/code-to-docs` to `AshwinUgale/code-to-docs`
- `[x]` Clone fork to `C:\Users\ugale\contrib\code-to-docs-fork\code-to-docs\`
- `[x]` Clone `opendatahub-io/agent-eval-harness` (read-only reference) to `C:\Users\ugale\contrib\agent-eval-harness\agent-eval-harness\`
- `[x]` Create `eval-harness-pr1` branch, pushed to `origin`, tracking set
- `[x]` Add `.claude-workspace/` to `.git/info/exclude` (gitignored locally)
- `[x]` Scaffold `.claude-workspace/` (CLAUDE_ONBOARDING, PROJECT_CONTEXT, MAINTAINER_THREAD, DECISIONS, PROGRESS, learnings skeleton, code-to-docs-notes/)
- `[x]` Seed `learnings.md` with 10 topic write-ups
- `[x]` Write walkthrough 12 (generation pipeline — `ask_ai_for_updated_content`, format prompts, truncation) into `code-to-docs-notes/`
- `[x]` Write REPL exercise recipe (`code-to-docs-notes/REPL_EXERCISE.md`)

### Study + design phase

- `[x]` Ashwin: REPL exercise — invoke discovery and generation manually against a tiny fake docs tree (run on `gpt-4o-mini` after Gemini free-tier blocked; results captured in D-13..D-17 and the empirical-evidence section of `DECISIONS.md`)
- `[x]` Add empirical decisions to `DECISIONS.md` based on REPL findings (D-13 path normalization, D-14 BOM handling, D-15 filter-drop logging, D-16 name-overlap trap design, D-17 per-stage fence rates)
- `[x]` Read agent-eval-harness: `score.py`, `config.py`, `events.py`, sample `eval.yaml`, runs/ layout. Write summaries into `agent-eval-harness-notes/` (2026-06-18: 5 notes — runs-layout, score-py, judges, config-py, events-py. New findings: dataset-vs-runs split is load-bearing for execute.py; baseline-delta gate has a loose 0.5 tolerance → PR-2 must gate on absolute `min_*`.)
- `[x]` Design `execute.py` on paper in `DECISIONS.md` (2026-06-18: D-18 feed-diff-directly/no-git-repo [retires D-11]; D-19 workspace lifecycle + invocation flow; D-20 content-gen on gold/annotated files; D-21 N-loop = N separate run dirs + our aggregator over copied scoring; D-22 API-error→flake bookkeeping. Also de-duped the corrupted DECISIONS.md and cross-linked D-02/D-06/D-08/D-09 to the harness notes.)

### Build phase

- `[x]` Copy `score.py + config.py + events.py` into `eval/scoring/`, rewire imports (2026-06-18: Option A — vendored `agent_eval/` as a subpackage beside `score.py`; all 5 files byte-identical [diff-verified], NO rewiring needed. Source: agent-eval-harness v1.14.0 @ `6fbba9f`. `eval/scoring/PROVENANCE.md` written.)
- `[x]` Import smoke-check — `import OK` confirmed (vendored agent_eval resolves; pyyaml installed)
- `[x]` Sanity-run scoring on a hand-authored mini-fixture — **PASSED in-tree** (2026-06-18). smoke-good → 0 regressions, exit 0; smoke-broken → 3 regressions (index 0.0, content 0.0, fence 0.5) with selection_f1_scan clean at 1.0, exit 1. Proves: vendored copy runs, module + inline-check judges both work, per-path separation (D-04), threshold gating + exit codes. Caught + fixed a missing `dataset:` block in eval.yaml (annotations weren't loading). **Cleanup owed before commit:** delete `eval/_diag.py`, `eval/dataset/cases/_smoke_*`, `eval/runs/`; add `eval/runs/` to `.git/info/exclude` (run outputs are regenerated, never committed).
- `[x]` Write `eval/execute.py` (the risk surface) — **VALIDATED end-to-end on real pipeline** (2026-06-18, gpt-4o-mini, case 001). No crashes, no flakes, outputs written + scored. D-18/19/20/03/22 implemented. Plus `eval/prep_index.py` (frozen-index prep, D-19) + case `001_add_cli_flag`. Deferred in code: D-15 filter-drops + D-17 fence_rate_discovery (need discovery-stdout capture); fence_rate_generation already covered by the no_fence judge.
- **FINDING (recorded in DECISIONS):** case 001 index path under-selects 0/3 vs full-scan 3/3 — systematic, reproducible, first real eval product. Generality TBD (1 case). Next: broaden corpus to test it.
- `[x]` Write `eval/judges/selection.py` (precision/recall/F1; normalize_path per D-13; `which=index|scan` for D-04)
- `[x]` Write `eval/eval.yaml` (real schema: 2 selection module-judges + content_assertions + no_fence_wrapper inline checks; thresholds). Note: `max_new_lines`/`must_not_add_sections` deferred to a content module-judge once execute.py emits the original-doc baseline.
- `[x]` Author case 001 (positive: add_cli_flag) — done + run
- `[x]` Author case 002 (negative trap: internal_rename_trap, D-16 private-vs-public)
- `[x]` Author case 003 (positive multi-candidate: add_config_option — probes the 001 index finding)
- `[x]` Author cases 004-010 (2026-06-18). Balanced 5/5 corpus. pos {001 cli-flag, 003 config-option, 004 public-fn, 006 multi-target-rename, 008 new-error}; neg {002 private/public trap, 005 vocab trap, 007 clean refactor, 009 v1/v2 version trap, 010 borderline perf-opt}. Designed to map WHEN discovery misses (literal vs conceptual links, single vs multi-target) + cover D-16 trap shapes. Awaiting full-corpus run.
- `[~]` Pre-build `.doc-index/` fixtures — 001 done; 002/003 via `prep_index.py --all` (Ashwin)
- `[x]` Write `eval/aggregate.py` — N-sample layer (D-08/D-21). Reuses vendored score.py judge dispatch; reports per-(case,judge) pass-rates across samples + flake exclusion (D-22). Replaces the throwaway per-sample score.py reads.
- `[x]` **HARDEN (before finishing corpus)** (2026-06-18):
  - Vacuous-pass guard: `judges/meta.py::judge_annotations_present` + `eval.yaml` judge `annotations_present` (min_pass_rate 1.0). A case whose answer key didn't load now goes RED instead of silently passing. **Must exist before authoring 004-010** so a typo'd annotations.yaml can't silently corrupt the corpus.
  - D-15 filter-drops: `execute.py` now tees discovery stdout, parses "Skipping non-documentation files: [...]", writes `meta/meta.json` {flakes, filter_drops}; `aggregate.py` reads it from disk (meta is intentionally not an eval.yaml output) and reports drop count. Also fixed: flake-exclusion never actually fired before (meta wasn't loaded into the record).
  - D-17 fence_rate_discovery: **deferred with reason** — not capturable black-box (discovery strips the fence inside the LLM response; never reaches stdout). Needs a src hook / tiny upstream PR. fence_rate_generation already covered by `no_fence_wrapper`.

### Verify phase

- `[x]` Spike replication — smoke good/broken in-tree confirmed exit 0 / exit 1
- `[~]` Per-run LLM call-count estimate — rough: ~4-5 calls/case/sample (index ~2 + full-scan ~1 + content-gen 1-2); 10 cases × N=3 ≈ 135 calls + prep ~18 one-time. Finalize in PR description.
- `[x]` Full corpus dry run — corpus-v3 (artifact-free) IS the validated end-to-end run + the v1 baseline.

### PR phase

- `[ ]` PR description draft (per `DECISIONS.md` and `MAINTAINER_THREAD.md` "owed" list)
- `[ ]` Push final commits to `eval-harness-pr1` on `AshwinUgale/code-to-docs`
- `[ ]` Open PR against `redhat-community-ai-tools/code-to-docs:main`
- `[ ]` Ping `@csoceanu` and `@HofniGartner` in PR description

---

## Blockers

(none currently)

---

## Notes since last update

- 2026-06-17 morning: Branch created, workspace scaffolded. `learnings.md` seeded, walkthrough 12 written.
- 2026-06-17 afternoon: REPL exercise done on `gpt-4o-mini` (Gemini 2.0 Flash free tier blocked — `limit: 0` on Ashwin's account, irrelevant to PR). Five new decisions added (D-13..D-17). Two new open items in the "Decisions explicitly NOT yet made" section — both surfaced by REPL observation. Next: read agent-eval-harness together.
- 2026-06-18: **Harness built and validated end-to-end.** Read agent-eval-harness (5 notes). Designed execute.py on paper (D-18..D-22, retired D-11, de-duped DECISIONS). Built: vendored scorer (`eval/scoring/`, byte-identical), `judges/selection.py`, `eval.yaml`, `execute.py`, `prep_index.py`, `aggregate.py`. Scoring proven in-tree (smoke good→exit0 / broken→exit1). Ran real pipeline (gpt-4o-mini) on 3 authored cases × 3 samples.

- 2026-06-18 **CURRENT CHECKPOINT (resume here):**
  - **Corpus complete + clean baseline.** Authored all 10 cases (balanced 5/5). First 10-case run (corpus-v2) had a FIXTURE CONFOUND — doc-folder names matching code-file basenames (e.g. `docs/api/` + `src/api.py`) made the model mangle paths (`api.py/reference.md`). Caught via `_diag.py`. Fixed by renaming the code file in 5 diffs (001/003/004/006/010); no re-prep needed. **Clean baseline = corpus-v3** (DECISIONS findings): selection_index 40% / scan 43% / content 60% / annotations_present 100% / no_fence 100%. **THE finding: discovery is lexically-driven** — renaming cli.py→main.py alone dropped 001 scan 3/3→1/3; without name bridges all 5 positives collapse to ~0 recall. Plus: conservatism holds on negatives except vocab trap 005; content-gen breaks on every name-overlap trap (002/005/009). All gpt-4o-mini.
  - **Hardened:** vacuous-pass guard (`annotations_present` judge), D-15 filter-drops (drops 8→0 after the collision fix confirmed the mechanism). D-17 discovery-fence deferred (not black-box capturable).
  - **Wrote `eval/README.md`** (maintainer-facing run recipe + case-authoring collision rule) and `eval/.gitignore` (runs/, pycache).
  - **PUSHED:** harness → `origin/eval-harness-pr1`; workspace context → `origin/claude-context` (recreated as a workspace-ONLY orphan branch). The old claude-context's "lost during recovery / re-add D-13..D-17" history was the source of the DECISIONS.md duplication we cleaned today.
  - **REMAINING for PR 1 (small):** (1) delete throwaway if not already — `eval/_diag.py`, `eval/dataset/cases/_smoke_pos`, `_smoke_neg`, `eval/runs/`; (2) finalize the per-run cost estimate; (3) draft the PR description (owed items: D-01 import trace, D-02 option-c rationale, cost, N=3/10 rationale, brief findings); (4) open PR vs `redhat-community-ai-tools/code-to-docs:main`, ping `@csoceanu` + `@HofniGartner`. **The build is DONE; this is polish + paperwork.**

---

## Deferred (NOT in PR 1)

- `[-]` CI workflow (GitHub Actions YAML, PR trigger, scheduled daily, regression gate) — **PR 2**
- `[-]` LLM-as-judge against gold references — **v2** (post PR 1 + PR 2)
- `[-]` Mining real merged docs-PRs for higher-signal cases — **v2**, no target repo exists yet
- `[-]` Style-config feature (issue #15) — separate contribution, unrelated to harness
- `[-]` Other contribution ideas (action.yml author, requirements.txt, Dockerfile dep pinning, etc.) — parked, see `code-to-docs-notes/02-contributions.md`
