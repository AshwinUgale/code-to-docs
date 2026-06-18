# PROJECT_CONTEXT — code-to-docs eval harness contribution

Personal open-source contribution to `redhat-community-ai-tools/code-to-docs` — an AI GitHub Action that updates docs from code PRs (`[review-docs]` / `[update-docs]` / `[review-feature]`).

**This is unrelated to any other Ashwin project (ARIA/chatbot/Enidus work). Do not pull cross-project context.**

## What this is

Contributing an **evaluation harness** to `redhat-community-ai-tools/code-to-docs`.

**Problem being solved:** there's no way to measure whether a change to file selection or content generation improves or regresses quality. All validation is manual eyeballing. The harness makes quality changes (embeddings retrieval, prompt edits, the style-config feature in issue #15, etc.) measurable instead of vibes-based.

**Status: maintainer APPROVED the plan. Building PR 1 now.**

- Maintainer: **csoceanu (Carmel Soceanu)**, also cc **@HofniGartner**.
- Proposal went through ~4 rounds of comments on an Ashwin-authored eval-harness issue.
- Note: issue **#15** = "configurable doc style guidelines/templates" — a *different*, maintainer-authored feature request. Good follow-up candidate, NOT what we're building now.

## The plan (approved)

Two PRs:

**PR 1 — the harness (self-contained, no CI).** What we're building now. Everything runs locally: execute → score → regression.

**PR 2 — CI wiring (follow-up, after PR 1 lands + a baseline exists).** PR-triggered smoke (subset, N=3) + scheduled daily full run (N=10), gated by `score.py regression` vs a committed baseline.

Split deliberately: the baseline can only be generated once PR 1's runner has run against the real corpus.

## Locked technical decisions

See `DECISIONS.md` for the structured version with rejected alternatives. Summary:

| Decision | Choice |
|---|---|
| Scoring code | **Straight copy** of agent-eval-harness `score.py + config.py + events.py` into `eval/scoring/` (NOT pip dependency, NOT a trimmed/stripped copy, NOT a from-scratch rewrite) |
| Eval location | `eval/` at repo root |
| Index side-effects | **Option (c)**: pre-built frozen indexes committed per case fixture; stub `fetch_indexes_from_main` + `commit_indexes_to_repo` to no-ops |
| Discovery paths | Score the **two paths separately**: index path (`find_relevant_files_optimized`) and full-scan fallback (`ask_ai_for_relevant_files`). Force full-scan via `--no-index`. |
| Content judges (v1) | **Assertions only.** Per-case (`must_mention`, `must_not_mention`, `max_new_lines`, `must_not_add_sections`) in each case's `annotations.yaml` + global checks (parses as declared format, balanced fences, no leading ``` wrapper). **No LLM judge in v1.** |
| Selection judge | module judge: precision / recall / F1 vs expected file set |
| Corpus | ~10 hand-authored cases, **balanced** should-update vs `NO_UPDATE_NEEDED` |
| Non-determinism | Run each case **N times, track pass rate** (not single pass/fail). N=3 on PR, N=10 daily. The harness's `samples:N` is per-*judge*; running the *system under test* N times is our runner's job. |

**Owed to the maintainer in the PR:** document the (c) choice + the two rejected index options (a = mock indexing, b = run indexing + patch git push), and a **per-run LLM call-count estimate** so cost is concrete before the CI PR.

## PR 1 contents (all new files, no edits to existing code-to-docs code)

```
eval/
├── eval.yaml              # agent-eval-harness schema (validated by spike)
├── scoring/               # straight copy: score.py, config.py, events.py (imports rewired)
├── execute.py             # OUR runner: per case → temp git repo (git init + base commit +
│                          #   diff applied, PR_BASE=base SHA so get_diff works w/ no remote);
│                          #   frozen indexes copied in; git side-effect fns stubbed; invokes
│                          #   real find_relevant_files_optimized + ask_ai_for_updated_content;
│                          #   writes outputs in harness runs layout; supports samples=N
├── judges/selection.py    # module judge: precision/recall/F1, index vs full-scan separate
└── dataset/cases/
    ├── 001_add_cli_flag/            # input.yaml, diff.patch, docs/, annotations.yaml
    ├── 002_refactor_no_doc_impact/  # negative → expected NONE / NO_UPDATE_NEEDED
    └── ...                          # ~10, balanced
```

## Key facts about the code-to-docs codebase (verified by reading)

- **Two discovery paths.** `find_relevant_files_optimized` (`src/discovery.py:309`) = index-based, the default. `ask_ai_for_relevant_files` = full-scan fallback (no git side effects, just LLM). In `suggest_docs.py`: `use_index` default True; falls back to full-scan when indexes missing or AI bails. `--no-index` forces full-scan.
- **Index path side effects** (in `src/doc_index.py`): `fetch_indexes_from_main` (git fetch), `build_all_indexes` (LLM calls per folder), `commit_indexes_to_repo` (git push to main). These break in a remote-less temp repo → why option (c) stubs the two git fns + freezes indexes.
- **Content generation:** `ask_ai_for_updated_content` (`src/generation.py:90`). Returns literal `NO_UPDATE_NEEDED` or the rewritten file. Format rules (md/adoc/rst) hardcoded at lines 96-164. Prompts forbid wrapping output in a code fence (the global no-fence assertion targets this).
- **Instruction injection seam:** `combined_instructions` block at `generation.py:207-225` (global + per-file). Per-comment instructions parsed in `src/comments.py::parse_update_instructions`. (Relevant to issue #15 if picked up later.)
- **Threading path for a new param:** `suggest_docs.py` → `generate_updates_parallel` → `process_file` → `ask_ai_for_updated_content`.
- Repo has pytest tests, **no CI workflow**, deps are unpinned in Dockerfile, no requirements.txt (deps: openai, mcp, mcp-atlassian). action.yml author is placeholder. (Other contribution ideas noted but parked.)

## The monkey-patch surface (critical for execute.py)

`src/discovery.py` does `from doc_index import (fetch_indexes_from_main, commit_indexes_to_repo, ...)` at module load. Python binding means `discovery.fetch_indexes_from_main` is now its own reference. **Patching `doc_index.fetch_indexes_from_main` will NOT redirect `discovery`'s call site.** Patch **`discovery.fetch_indexes_from_main`** and **`discovery.commit_indexes_to_repo`** directly (or use `unittest.mock.patch('discovery.fetch_indexes_from_main')`).

## agent-eval-harness — what we reuse

[`opendatahub-io/agent-eval-harness`](https://github.com/opendatahub-io/agent-eval-harness) — Claude Code skill eval framework (8 skills + Python lib + MLflow).

- **Execution layer** = runner-coupled (spawns CC subprocess). NOT usable for us (code-to-docs is a plain Python entry point). We write our own `execute.py`.
- **Scoring layer** (`skills/eval-run/scripts/score.py` + `agent_eval/` pkg) = execution-agnostic. Reads case outputs from a runs dir, applies judges from `eval.yaml`, has `judges` / `pairwise` / `regression` modes. **This is what we copy.**
- Judge types: inline `check` (Python `(bool,str)`), `module/function`, LLM `prompt`/`prompt_file` (incl pairwise), `builtin`. Thresholds: `min_pass_rate` / `min_mean` / `min_win_rate`.
- `samples:N` exists but **per-judge** (reruns a stochastic LLM *judge*), not per system-under-test.
- Core deps just `pyyaml` + `jinja2`; mlflow/anthropic/openai are optional extras.
- **`config.py` does NOT require the CC-skill fields** (`runner`/`models`/`permissions`) — proven by the spike running a minimal eval.yaml without them.

**Other two repos the maintainer pointed at — both rejected:**
- `eval-hub` (Go REST service, OpenShift, fleet-scale benchmarks) — wrong scale. Skip.
- `sdg_hub` (synthetic data generation flows) — defer to v2 for scaling the corpus past hand-authored.

## The spike (proof the scoring reuse works)

Originally at `C:\Users\Ashwin U\eval-research\spike\` on the original machine. Self-contained. Proved the **integration contract**: `score.py` runs our judges + regression gate against hand-authored outputs with **no CC runner**.

- 2 cases (1 positive, 1 negative), 3 judges (module F1 + inline content assertions + no-fence).
- `runs/code-to-docs/spike-run-1/` = good outputs → all judges pass, regression exit 0.
- `runs/code-to-docs/spike-run-2-broken/` = wrong file / hallucination / fence wrapper → all 3 judges fail, regression **exit 1** (proves the CI gate is real).
- Run via: install harness into a venv, set `AGENT_EVAL_RUNS_DIR=runs` + `PYTHONPATH=.`, then `score.py judges --run-id <id> --config eval.yaml` and `score.py regression --run-id <id> ...`.
- Caveat stated to maintainer: this proves the **machinery** is reusable, NOT that the eval is *good* — judge quality/coverage is what the real corpus is for.

## Build order (for PR 1)

1. Copy `score.py` + `config.py` + `events.py` into `eval/scoring/`, rewire imports, sanity-run.
2. Write `execute.py` — the real risk surface (temp-repo + frozen-index fixture + stubbing the two git fns + invoking the real pipeline + writing the runs layout).
3. Selection judge.
4. ~10 balanced cases + annotations.
5. Document the (c) index choice + rejected options + per-run LLM call-count estimate in the PR description.

PR against `redhat-community-ai-tools/code-to-docs` `main` from branch `eval-harness-pr1` on `AshwinUgale/code-to-docs`.
