# DECISIONS — locked, do not relitigate

Each entry records: **what was decided**, **why**, and **what was rejected**. Future Claude sessions: read before proposing alternatives. If you think a decision should be reopened, state explicitly that you're reopening D-NN and give a new reason — don't silently work around it.

Sources cited inline: `MAINTAINER_THREAD.md#round-N` and code refs.

---

## D-01 — Scoring code provenance

**Decision:** Straight copy of `score.py + config.py + events.py` from `agent-eval-harness` into `eval/scoring/`. Imports rewired local. No pip dependency, no rewrite, no trimming.

**Why:**
- Maintainer (csoceanu, Round 5) explicitly preferred copy over pip dep: "the harness's config schema is designed for Claude Code skill eval, most of it doesn't apply here."
- Import-graph trace (Ashwin Round 6) showed `score.py`'s core path only pulls `config.py` + `events.py`, both stdlib + `pyyaml`. No internal cross-imports for the path we use. LLM/MLflow/builtin machinery is lazy-imported behind branches we don't hit.
- Spike proved `config.py` doesn't require the CC-skill schema fields (`runner`/`models`/`permissions`) — minimal `eval.yaml` ran without them.
- Provenance preserved → can re-sync from upstream if they fix bugs.

**Rejected:**
- **Pip dependency.** Schema couples us to upstream releases; most of the CC-skill schema is irrelevant; csoceanu didn't want it. ✗
- **Trimmed/stripped copy.** Loses re-syncability; we'd have to maintain a diff. The dormant LLM-judge branches are exactly what v2 wants. ✗
- **From-scratch rewrite.** csoceanu's conditional fallback: only if untangling proved too expensive. Trace showed it's clean → no rewrite needed. ✗

**Hinge condition:** csoceanu approved the copy conditional on "if it's too entangled, a simpler self-contained implementation might be less work." The import trace cleared that condition. **The PR description must surface the trace** so the reviewer sees why we proceeded.

**Harness-source detail:** see `agent-eval-harness-notes/score-py.md` (import surface), `config-py.md` (schema tolerance), `judges.md`, `runs-layout.md`.

---

## D-02 — Index path side-effects in the test environment

**Decision:** Option **(c)** — pre-built frozen `.doc-index/` committed per case fixture; stub `fetch_indexes_from_main` and `commit_indexes_to_repo` to no-ops at the call sites inside `discovery.py`.

**Why:**
- A file-selection eval is about index *consumption*, not *building*. Index-building variance + cost shouldn't be baked into every eval run.
- The maintainer enumerated (a)/(b)/(c) (Round 5) and asked us to choose deliberately. (c) is the cheapest and tests the most-common production path (cached indexes already on main).
- `fetch_indexes_from_main` does `git fetch origin` — fails with no remote in a temp repo. `commit_indexes_to_repo` does `git push` to main — same problem. Stubbing them out is the cleanest fix.

**Rejected:**
- **(a) Mock indexing entirely.** Doesn't test the real `find_relevant_files_optimized` consumption path. Loses signal. ✗
- **(b) Run indexing + patch git push.** Real signal but expensive — every case pays the per-folder LLM index-build cost. Csoceanu flagged cost: "10 cases × N=10 with LLM calls at each stage adds up." Repeating index-build N=10 times per case compounds this. ✗

**Owed in PR description:** name the three options and explain why (c) was chosen. csoceanu will look for this.

**Superseded sub-point:** an earlier version of this entry had a `get_diff()` / `PR_BASE` bullet. That is **retired by D-18** — we don't call `get_diff()` at all, so there's no `PR_BASE` to set. Stubbing the two index git functions remains correct; the frozen index's manifest hashes must match the docs bytes as materialized, or `update_indexes_if_needed` triggers an LLM rebuild (see D-19).

---

## D-03 — Monkey-patch surface

**Decision:** Patch `discovery.fetch_indexes_from_main` and `discovery.commit_indexes_to_repo` (the `discovery` module namespace), NOT `doc_index.fetch_indexes_from_main` etc.

**Why:**
- `src/discovery.py:26-35` does `from doc_index import (fetch_indexes_from_main, commit_indexes_to_repo, ...)`. Python's `from x import y` rebinds the names into `discovery`'s namespace.
- Patching `doc_index.fetch_indexes_from_main` won't affect `discovery.fetch_indexes_from_main` — they're now separate references to the original function object.
- The call sites inside `discovery.find_relevant_files_optimized` resolve through `discovery`'s namespace (verified: `discovery.py:326` and `:434`).

**How:** `unittest.mock.patch('discovery.fetch_indexes_from_main')` and `('discovery.commit_indexes_to_repo')`, or direct attribute assignment before invoking the pipeline.

**Rejected:** Patching the `doc_index` module directly. Would silently no-op (the patched function never gets called). ✗

**Note:** Easy to get wrong, hellish to debug. Add a comment in `execute.py` near the patches explaining the binding semantics so the next maintainer doesn't "fix" it.

---

## D-04 — Discovery paths scored separately

**Decision:** Run each case through both discovery paths independently:
1. **Index path** — `find_relevant_files_optimized(diff)` with frozen `.doc-index/` present
2. **Full-scan path** — `get_file_content_or_summaries()` then `ask_ai_for_relevant_files(diff, file_previews)` (called directly; the "--no-index equivalent")

Score precision/recall/F1 for each path separately in the selection judge output. Aggregate per-path, not blended.

**Why:**
- They're alternative production strategies, not one-true-path. Production picks one per request based on whether indexes exist.
- A regression on one path can hide behind improvement on the other if scored together.
- Full-scan path has no git side effects → simpler to test (no fixture needed beyond `docs/`).

**Rejected:** Single blended score. Loses signal on which path regressed. ✗

---

## D-05 — Content evaluation: assertions only, no LLM judge in v1

**Decision:** v1 = deterministic assertions. Per-case in each case's `annotations.yaml`:
- `must_mention: [str, ...]`
- `must_not_mention: [str, ...]`
- `max_new_lines: int`
- `must_not_add_sections: bool`

Plus global `check` judges on every case:
- Output parses as the format declared by extension (md/adoc/rst)
- Code fences balance
- First non-blank line is not a code fence (generation prompt forbids fence-wrapping; LLM still does it sometimes)

The LLM-judge slot exists in the `eval.yaml` schema but stays empty in v1.

**Why:**
- Cheap, deterministic, no gold-reference authoring burden.
- Assertions catch the real failure modes: hallucination (`must_not_mention`), scope creep (`max_new_lines`, `must_not_add_sections`), format drift (global checks), missed updates (`must_mention`).
- LLM-judge requires gold references → not in PR 1 scope.
- Schema slot reservation = zero rework when v2 adds gold references.

**Rejected:**
- **LLM judge in v1.** No gold references yet, expensive, adds judge-variance on top of SUT-variance. Save for v2. ✗
- **Drop the LLM-judge schema slot.** Would require schema migration when we add gold refs. ✗

**Caveat to maintainer (already stated, Round 1):** assertions are an indirect proxy. If first 10-15 cases show the harness missing obvious failures, switch to LLM-as-judge. That's a v2 problem. csoceanu didn't push back on this in Round 2 → tacit accept.

---

## D-06 — Selection judge implementation

**Decision:** Module judge at `eval/judges/selection.py`. Computes precision / recall / F1 against each case's expected file set.

- **Precision** = |selected ∩ expected| / |selected|
- **Recall** = |selected ∩ expected| / |expected|
- **F1** = harmonic mean

Special-case handling:
- Empty expected (`NO_UPDATE_NEEDED` case): selection must also be empty for a pass. Define precision/recall = 1.0 in that case; non-empty selection → both 0.0.
- Path normalization: compare relative paths, lowercase, forward slashes (see D-13).

**Why:** Standard IR metrics. Matches what was proposed in Round 1 and approved. Module judge fits agent-eval-harness's existing judge-type infrastructure cleanly (see `agent-eval-harness-notes/judges.md`).

**Rejected:**
- **Set-membership exact-match only.** Misses partial credit (selected 2/3 right). ✗
- **Top-K ranking metrics.** code-to-docs returns an unranked set, not a ranked list. ✗

---

## D-07 — Corpus shape

**Decision:** ~10 hand-authored cases for PR 1, balanced ~50/50 between "should-update" and "`NO_UPDATE_NEEDED`."

Case folder structure:
```
eval/dataset/cases/NNN_short_name/
├── input.yaml             # case metadata (description, expected paths, trap shape if negative)
├── diff.patch             # git-format unified diff, fed directly as the `diff` string (D-18)
├── docs/                  # the docs tree as it should appear before the diff is applied
│   ├── file1.md
│   └── ...
├── .doc-index/            # pre-built frozen index for the index path (D-02 / D-19)
└── annotations.yaml       # must_mention / must_not_mention / max_new_lines / must_not_add_sections per output file; also expected_files for selection
```

**Why:**
- Balance: maintainer flagged that over-selection / unnecessary updates are the more user-visible failure mode (Round 2). Conservatism is a key feature of the system. Negative cases must be first-class.
- 10 cases: enough to surface a regression, cheap enough to run nightly at N=10 (= 100 SUT invocations).
- Hand-authored: maintainer confirmed no docs repo has been an active code-to-docs target → no PRs to mine. Synthetic from open-source repos is fine for PR 1; case schema is structured so real mined cases can drop in later.

**Rejected:**
- **Mining real merged docs-PRs.** csoceanu Round 2: no target repo exists yet. ✗
- **Heavily-positive corpus.** Would miss the over-selection failure mode. ✗

---

## D-08 — Non-determinism handling

**Decision:** Our runner invokes the system under test N times per case, tracks per-judge pass rate. N=3 on PR-triggered runs, N=10 on the scheduled daily run.

**Why:**
- LLM stochasticity in `find_relevant_files_optimized` and `ask_ai_for_updated_content` makes single-shot pass/fail flaky. csoceanu raised this in Round 2.
- `samples:N` in agent-eval-harness exists but is **per-judge** (re-runs a stochastic LLM *judge* against the same output, not the SUT) — confirmed in `agent-eval-harness-notes/judges.md`. Our runner has to handle SUT-N-times itself (see D-21).
- Different N for PR vs daily: fast feedback on PRs, accurate signal on daily.

**Rejected:**
- **Use `samples:N` for SUT variance.** Wrong semantic — measures judge variance, not SUT variance. ✗
- **Single-shot per case.** Flaky. ✗
- **Higher N on every PR.** Too slow + expensive for fast feedback. ✗

**Owed in code:** comment in `execute.py` near the N-loop explaining the per-judge vs per-SUT distinction so the next contributor doesn't try to "simplify."

**Open to changing N** if csoceanu wants different numbers — Ashwin offered this in Round 3 and got no preference back.

---

## D-09 — PR scope: PR 1 is the harness only, PR 2 is CI wiring

**Decision:** PR 1 contains the harness (execute/score/judge/cases) and a documented manual-run recipe. **No CI workflow.** PR 2 (follow-up) adds the GitHub Actions workflow, PR-triggered smoke (N=3), scheduled daily (N=10), gated by `score.py regression` vs a committed baseline.

**Why:**
- The CI gate needs a baseline. The baseline can only be generated once the harness has run against the real corpus. So the harness must land first.
- PR 2 needs maintainer secrets (model API keys for the scheduled run) and is structurally a separate review.

**Note for PR 2:** `score.py`'s baseline-delta check uses a loose 0.5 absolute tolerance (`score-py.md`); the real gate must be the absolute `min_mean`/`min_pass_rate` thresholds, not the baseline delta.

**Rejected:**
- **Single PR.** Would require committing a baseline generated locally before merge → maintainer can't reproduce. ✗

---

## D-10 — Eval location

**Decision:** `eval/` at the repo root (sibling to `src/`, `tests/`, etc.)

**Why:** csoceanu explicitly chose this in Round 5: "For location — `eval/` at the repo root makes sense."

**Rejected:**
- **`tests/eval/`.** Conflates with pytest test discovery (`tests/` is collected by pytest by default). Eval is a separate concept and should not be auto-collected. ✗

---

## D-11 — RETIRED (superseded by D-18)

**Was:** "`get_diff` works in a remote-less temp repo" — set `PR_BASE=<base SHA>` after a `git init` + base commit + diff-applied commit so `get_diff`'s local merge-base succeeds.

**Retired because:** D-18 verified that `get_diff()` is called only from `suggest_docs.main()` (`suggest_docs.py:147`), never by the mid-level functions the runner invokes. The runner feeds the `diff` string directly, so `get_diff()` is never called, so there is no `PR_BASE` to set and no git repo to build. See D-18.

---

## D-12 — Workspace layout for Claude session continuity

**Decision:** `.claude-workspace/` folder at repo root, listed in `.git/info/exclude` (local-only, never committed). Contains: `CLAUDE_ONBOARDING.md`, `PROJECT_CONTEXT.md`, `MAINTAINER_THREAD.md`, `DECISIONS.md`, `PROGRESS.md`, `learnings.md`, `code-to-docs-notes/`, `agent-eval-harness-notes/`.

**Why:** Ashwin works across multiple Claude sessions. Without persisted context, every new session re-litigates settled decisions. The onboarding file gives a new Claude the entire context in one read.

**Rejected:**
- **`.gitignore` entry.** Would be committed → noise in PR diff. ✗
- **Outside the repo.** Loses path locality; harder to reference relative paths. ✗

---

## D-13 — Path normalization in selection judge

**Decision:** The selection judge normalizes both LLM-returned paths and expected-set paths before set comparison. Normalization:
1. Replace backslashes with forward slashes
2. Lowercase the path
3. Strip a leading `src/` segment (and any single common docs-tree prefix segment, e.g. matching `$DOCS_SUBFOLDER`)
4. Strip any leading `./`
5. Strip surrounding whitespace

Apply to BOTH `selected` and `expected` lists before computing the intersection.

**Why:** Direct empirical signal from REPL. In 5 positive-case runs on 4o-mini, **2/5 runs returned `src/cli/reference.md`** instead of the actual `cli/reference.md`. The LLM parrots a fictional subfolder prefix back from the previews. Without normalization, those 2 runs falsely score precision=0, recall=0 despite picking the correct file. With normalization, the metric reflects the actual selection quality (5/5 correct).

Production code at `src/discovery.py:280-292` does similar prefix-stripping when `$DOCS_SUBFOLDER` is set. The judge needs the same defense even when the env var isn't set, because the LLM can hallucinate a prefix that wasn't in the env.

**Rejected:**
- **No normalization (strict string match).** Loses 40% of selection signal to a known LLM behavior. ✗
- **Fuzzy similarity score (Levenshtein, etc.).** Hides real path-error failure modes (LLM picking a sibling-named file). ✗

**Implementation note:** Helper at `eval/judges/selection.py::normalize_path(path: str, docs_subfolder: str = "") -> str`. Unit-test against fixtures: `("src/cli/reference.md", "src") == "cli/reference.md"`, `("./Docs/x.md", "docs") == "x.md"`, etc.

---

## D-14 — BOM handling in case fixtures

**Decision:** Case fixture files are authored without BOMs. The runner strips a leading BOM (U+FEFF) from `current_content` before passing to `ask_ai_for_updated_content`, and from the comparison reference when applying judges.

**Why:** PowerShell's `Set-Content -Encoding UTF8` writes UTF-8 **with BOM** by default. REPL showed `current_content` started with a BOM (U+FEFF). The LLM stripped the BOM in its rewritten output. If any future judge compares exact content byte-for-byte, the BOM mismatch produces false failures.

`must_mention` substring matches aren't affected by this (the meaningful content isn't at the BOM position). But: a future reference-based judge, diff-based judge, or hash-equality judge would all break.

**Rejected:**
- **Leave BOMs in fixtures.** Future-tax on adding any byte-sensitive judge. ✗
- **Strip BOMs in the judge only, not the runner.** Inconsistent state — the runner would send a BOM'd file to the LLM while the judge expects no-BOM. Better to strip once at runner ingress. ✗

**Implementation note:** In `execute.py`, after `read_text(encoding="utf-8")`, do `if content.startswith("﻿"): content = content[1:]`. When authoring fixtures via PowerShell, use `Set-Content -Encoding UTF8NoBOM` or write via Python. **Also critical for the frozen index (D-19):** the index manifest hashes the docs bytes; a stray BOM changes the hash and triggers an LLM rebuild.

---

## D-15 — Log filter drops in run metadata

**Decision:** The runner captures and records every line the post-LLM filter drops, as part of the per-case run metadata in the runs/ output directory. Format: `filter_drops: [str, ...]` field in the per-case JSON.

**Why:** Direct REPL evidence. In one of 5 positive-case discovery runs, the LLM tried to "select" `src/cli.py` (a code file) as a doc file. `discovery.py:231` silently filtered it out via the `.adoc/.md/.rst` suffix check. The user-facing `selected` list looked clean (`['cli/reference.md']`) while the LLM was actively hallucinating.

Without logging the drop, the eval cannot regression-detect "the model started hallucinating code files more often." With logging, an aggregated `filter_drop_rate` metric becomes a real signal.

This is silent corruption — the LLM is producing wrong output, the codebase is masking it. The eval should surface what the prod code hides.

**Rejected:**
- **Track only the filtered output, ignore drops.** Lossy. The eval becomes blind to a real failure mode. ✗
- **Fail the case if any drop happens.** Over-strict; drops are usually harmless in production. Track and report, don't gate. ✗

**Implementation note:** Patch into the runner — capture stdout from the discovery call, parse lines matching the existing `Batch N: Skipping non-documentation files: [...]` log format, attach to the run record (this is the stdout-capture seam also used for fence-rate detection, D-17). Hacky but works without modifying `src/discovery.py`. Cleaner alternative: a tiny PR upstream adding a `return_drops=True` kwarg — out of PR 1 scope; do it locally for now.

---

## D-16 — Negative cases are name-overlap traps by construction

**Decision:** At least 2-3 of the 5 negative cases in the starter corpus are engineered as **name-overlap traps** — diffs that share vocabulary with a doc file but should NOT trigger an update. The remaining negative cases can be cleaner refactors.

Three trap shapes (use a mix):
- **Same vocabulary, different domain.** REPL example: CLI flag `--verbose` + config doc that lists `verbose` config keys. LLM bridges them and fabricates a config entry.
- **Same API surface, different layer.** Diff changes internal `_normalize()` helper; doc explains user-facing `normalize_path()` config.
- **Same component, different version.** Diff changes v2 module; doc is the v1 archive.

**Why:** The REPL exercise produced one concrete instance of this failure mode: the LLM rewrote `config/settings.md` to add a fabricated `verbose` config key after seeing the `--verbose` CLI flag in the diff. The trigger was name overlap; the conservatism gate didn't catch it; the output was plausible enough to fool a casual reviewer.

If our negative cases are all "clean" refactors (rename internal variable, dependency bump, etc.), the LLM has no reason to break conservatism, and we'd falsely conclude the system is robust. **Real production conservatism failures happen on name overlap.** The eval has to recreate the conditions to measure them.

**Rejected:**
- **All-clean negative cases.** Doesn't probe the real failure surface. False high pass rate. ✗
- **Only name-overlap negatives.** Loses signal on simpler failure modes. ✗

**Implementation note:** When authoring the 5 negative cases (002, 004, 006, 008, 010 in the corpus numbering), make 002, 006, 008 traps; 004 and 010 can be cleaner refactors. Document each case's trap shape in its `input.yaml` description so a reviewer understands why it's there. Trap files are content-gen targets with `expect_no_update: true` (D-20).

---

## D-17 — Fence-wrap rates tracked per stage

**Decision:** The eval reports `fence_rate_discovery` and `fence_rate_generation` as separate metrics in the runs/ output and baseline. Do NOT blend them into a single `fence_rate`.

**Why:** REPL evidence: 5/5 discovery runs fence-wrapped on 4o-mini; 0/5 content generation runs did. The two stages have different prompt structures (discovery returns a list, generation returns file content) and the LLM's fence-wrapping instinct triggers very differently.

A blended `fence_rate` of 50% (5 of 10) hides what's actually a 100% rate at one stage and 0% at the other. A regression where content generation suddenly fence-wraps 80% of the time would barely move the blended metric. Per-stage tracking exposes the regression.

Also relevant: the discovery filter at `src/discovery.py:231` salvages fence-wrapped discovery output by dropping the triple-backtick lines (they don't match `.md/.adoc/.rst`). So 100% fence-wrap in discovery doesn't degrade the final selection. But content-gen fence-wrap goes straight to the docs file with no salvage, so even a single fence-wrap in content gen is a real production failure.

**Rejected:**
- **Single blended `fence_rate`.** Hides the asymmetric behavior. ✗
- **Only track content-gen fence-wrap.** Loses signal on discovery model regressions (a future model might start fence-wrapping discovery in a way the filter doesn't catch). ✗

---

## D-18 — Feed the diff directly; no git repo in the test workspace (retires D-11)

**Decision:** `execute.py` reads `diff.patch` as a string and passes it directly to `find_relevant_files_optimized(diff)`, `ask_ai_for_relevant_files(diff, previews)`, and `ask_ai_for_updated_content(diff, ...)`. It does **not** call `get_diff()`, does **not** `git init`, makes no commits, sets no `PR_BASE`. The temp workspace is a plain folder (docs tree + frozen `.doc-index/`), not a git repo.

**Why (verified against fork source 2026-06-18):**
- `get_diff()` is called in exactly one place: `suggest_docs.py:147`, inside `main()`. The mid-level functions the runner invokes all take `diff` as a plain string parameter — none call `get_diff()`.
- `discovery.py` touches git in exactly two places: `fetch_indexes_from_main()` (L326) and `commit_indexes_to_repo()` (L434) — both stubbed by D-03.
- Therefore, with the two stubs + a frozen index present, the pipeline performs **zero** git operations. No repo needed.

**Requirement:** `diff.patch` must be authored as a real git-format unified diff (the same shape `get_diff()` emits: `diff --git ...`, `@@` hunks). Feeding it directly is then byte-equivalent to what production would compute.

**Retires:** D-11 (PR_BASE/merge-base). Refines D-02 (drops its old get_diff bullet).

**Rejected:**
- **Replicate `main()` including `get_diff()`** — needs a 2-commit git history (code-before → code-after) per case purely to reconstruct a diff we already have on disk. Pointless complexity. ✗
- **Call `suggest_docs.main()` directly** — it posts PR comments, opens docs PRs, and has many other side effects. Not a test entry point. ✗

---

## D-19 — execute.py workspace lifecycle + invocation flow

**Decision:** Per case, per sample-run, the runner:

1. **Make a temp workspace** (tempfile.mkdtemp). Copy the case's `docs/` tree and frozen `.doc-index/` into it. No `git init`.
2. **Isolate the environment:** set `MODEL_API_BASE` / `MODEL_API_KEY` / `MODEL_NAME` from the runner's own env; **unset** `DOCS_SUBFOLDER` (so `get_docs_root()` resolves to cwd), `PR_BASE`, `PR_NUMBER`, `GH_TOKEN` (so production code can't pick up stray values).
3. **`chdir` into the workspace** — required: `find_relevant_files_optimized`, `get_file_content_or_summaries`, and `get_docs_root()` all operate on cwd (rglob the current dir).
4. **Apply the git stubs** (D-03): patch `discovery.fetch_indexes_from_main` and `discovery.commit_indexes_to_repo` to no-ops.
5. **Run the pipeline:**
   - Index path: `idx = find_relevant_files_optimized(diff)`. If it returns `None` (the "fall back to full scan" signal), record the signal and treat the index-path selection as `[]` for its own score.
   - Full-scan path: `previews = get_file_content_or_summaries()`; `scan = ask_ai_for_relevant_files(diff, previews)`.
   - Content gen: for each annotated file (D-20), `ask_ai_for_updated_content(diff, doc, strip_bom(current))`.
6. **Write outputs** in the runs/ layout (D-21 / `runs-layout.md`), restore cwd, delete the temp workspace.

**Frozen-index hash gotcha:** `find_relevant_files_optimized` → `update_indexes_if_needed` → `folder_needs_reindex` compares stored manifest hashes vs current doc-file hashes. If they differ, it rebuilds the index with LLM calls — defeating the freeze. So the frozen `.doc-index/` **must** be generated from the exact docs bytes the runner materializes (no BOM drift — D-14). Best practice: generate each case's frozen index once via a fixture-prep step that runs `build_all_indexes()` on the materialized docs, then commit the result.

**Rejected:** Running the pipeline in-place in the repo (pollutes the working tree, risks real git ops). ✗ — temp workspace per run is the isolation boundary.

---

## D-20 — Content generation runs on the annotated (gold) files, not the actual selection

**Decision:** Content generation targets = the set of files that have an entry in the case's `annotations.yaml`. The runner calls `ask_ai_for_updated_content(diff, f, current)` for each such file, **independent of what discovery actually selected**.

**Why:**
- Isolates content-quality signal from selection-quality signal. If content-gen ran on the actual (possibly wrong) selection, a selection miss would leave the content judge with no file to check — conflating two distinct failures.
- Matches the stage-separation spirit of D-04.
- Negative content cases (name-overlap traps, D-16) are annotated targets with `expect_no_update: true`; the runner feeds the trap file to `ask_ai_for_updated_content` and the assertion checks for `NO_UPDATE_NEEDED`.

**Cost:** not strictly end-to-end (selection→generation chained). Accepted for v1. An end-to-end "generation on actual selection" metric is a possible v2 addition.

**Rejected:**
- **Generation on the actual discovery selection.** Cascades selection errors into the content score; can't isolate generation quality. ✗

---

## D-21 — N-loop disk layout: N separate run dirs, aggregate in our layer

**Decision:** `execute.py --samples N` produces **N separate run directories**, each laid out in the full runs/ structure (`runs/code-to-docs/<run_id>/cases/<case_id>/...`) with intact, dataset-matching `case_id`s — one per sample. A thin aggregator (ours) reuses the **copied** scoring internals (`load_judges`, `score_cases`, `detect_regressions`) to score each of the N runs, then combines per-case and overall pass-rates across the N. The gate is our aggregator applying the thresholds (any regression → exit 1).

**Why:**
- `score.py` reads each case's answer key from `dataset/<case_id>/annotations.yaml`, keyed by the **case folder name** (`runs-layout.md`). Laying N samples out as pseudo-cases (`003_refactor__s01`, ...) would break annotation resolution — those names don't exist under `dataset/`. So samples must be separate runs, each preserving the real `case_id`.
- `score.py`'s model is "one run = one deterministic snapshot." The N-aggregation across samples belongs in our layer — exactly what D-08 says.
- We reuse the copied scoring functions (not the CLI) so the aggregator drives them directly and owns the cross-sample math.

**Rejected:**
- **Pseudo-cases under one run** (`<case>__s01`). Breaks `dataset/<case_id>/annotations.yaml` resolution. ✗
- **`samples:N` in eval.yaml.** Per-judge semantics, wrong target (D-08). ✗
- **Reimplement judging in our aggregator.** Wastes the copied scoring code; risks drift from the maintainer-trusted logic. ✗

---

## D-22 — API error vs genuine-empty: distinct flake bookkeeping

**Decision:** The runner wraps each pipeline call and catches the OpenAI/API client exception types specifically. On an API error (rate limit, 5xx, timeout), it marks that **sample** a *flake* — recorded in a separate `flake_count` in the run metadata — and excludes it from pass/fail. Per-judge pass-rate is computed over **non-flake** samples only.

**Why:**
- REPL evidence: a Gemini 429 was swallowed by the retry logic and surfaced as an empty `[]` — byte-identical to a correct "conservative, selected nothing" result. Counting that as a pass (or fail) corrupts the metric in exactly the place it matters (negative/conservatism cases).
- A separate flake count both keeps the quality signal clean and surfaces rate-limit / infra problems as their own number.

**Resolves** the "transient API error mid-run" open question.

**Rejected:**
- **Retry until success.** Hides systemic rate-limiting; unbounded cost/time. ✗
- **Count empty-after-error as a conservative pass.** The exact false signal we're trying to avoid. ✗
- **Fail the case on any API error.** Turns transient infra blips into red CI. ✗

**Implementation note:** the wrap point is each of the three pipeline calls in D-19 step 5. Distinguish API exceptions (from the `openai` client) from genuine empty returns. A genuine empty list/`NO_UPDATE_NEEDED` with no exception is a real result and is scored normally.

---

## Empirical evidence from REPL (note, not a decision)

Observed on `gpt-4o-mini`, N=5 per case:

| Stage | Case | Recall/Conservatism | Fence-wrap | Hallucination |
|---|---|---|---|---|
| Discovery | positive (CLI flag) | 5/5 | 5/5 | 1/5 (filtered) |
| Discovery | negative (refactor) | 5/5 conservatism | 0/5 | 0/5 |
| Content | positive (CLI flag) | 5/5 | 0/5 | 0/5 |
| Content | negative (config + name-overlap trap) | 4/5 conservatism | 0/5 | 1/5 fabricated `verbose` config key |

**Implication for N choice:** the worst-case empirical failure rate is the 20% conservatism break on content generation negative cases. At N=3, the probability of catching at least one failure when the true rate is 20% is `1 - 0.8^3 = 0.488` (~49%). At N=10 it is `1 - 0.8^10 = 0.893` (~89%). The N=3 PR cadence + N=10 daily cadence (D-08) is empirically justified by this data — fast feedback on PR (acknowledged-noisy), strong signal nightly.

This goes in the PR description's "per-run LLM call-count estimate" section as the rationale for N.

---

## Decisions explicitly NOT yet made (open)

- **`max_new_lines` value per case.** Set per-case in `annotations.yaml`, no global default in v1. Pick concrete values when authoring each case.
- **Per-run LLM call-count estimate methodology.** Count: index-path LLM calls + full-scan-path LLM calls + content-gen calls (one per annotated file), per case, × N. Produces the concrete number for the PR description.
- **Cost estimate format in PR description.** Markdown table or prose — decide at PR-write time.
- **Deterministic mode (`temperature=0`) for reproducibility experiments.** Out of scope for PR 1; tag as future work.

(Resolved since last revision: API-error-vs-flake → D-22.)

---

## Findings from the in-tree scoring sanity-run (2026-06-18)

Surfaced while validating the vendored scorer against hand-authored fixtures. Both are
**latent traps that masquerade as passes** — captured here so the fixes land before they bite.

1. **Vacuous-pass on missing annotations.** If `eval.yaml` lacks a `dataset:` block (or a case's
   `annotations.yaml` fails to load), `record["annotations"]` defaults to `{}`. Consequences:
   the selection judge sees an empty `expected_files` → treats every positive case as a "select
   NONE" negative → false F1=0; and `content_assertions` iterates an empty rule set → returns
   `True` having checked nothing → **false 100% pass**. The `dataset:` block is now in `eval.yaml`
   with a "do not remove" comment. **Guard — DONE (2026-06-18):** `judges/meta.py::
   judge_annotations_present` + the `annotations_present` judge in `eval.yaml` (min_pass_rate 1.0).
   A case whose answer key didn't load (no `expected_files` key) now scores RED instead of silently
   passing. Added before authoring 004-010 so a fixture typo can't silently corrupt the corpus.

2. **Windows `open()` encoding (cp1252).** The vendored `score.py` reads `annotations.yaml` via
   `open()` with no explicit encoding → on Windows, UTF-8 bytes are misread as cp1252 (an em-dash
   in a `description` came back as `â€"`). Cosmetic for `description`, but a non-ASCII
   `must_mention`/`must_not_mention` string would corrupt and **silently break matching**. We do
   not patch `score.py` (byte-identical provenance, D-01). **Mitigation:** author all fixtures /
   annotations ASCII-only, or run with `PYTHONUTF8=1`. Put this in the corpus-authoring guidance
   and the PR's run recipe.

---

## First real-pipeline finding — index path under-selects vs full-scan (2026-06-18)

**The eval's first product.** Case `001_add_cli_flag` (diff adds `--verbose`; `cli/reference.md`
is the CLI doc), gpt-4o-mini, run end-to-end through `execute.py`:

| judge | result |
|---|---|
| `selection_f1_index` | **0.00 across 3/3 samples** (real-001, real-001b-s01/02/03) |
| `selection_f1_scan` | **1.00 across 3/3** |
| `content_assertions` | 100% (rewrite added `--verbose`, no `--quiet`) |
| `no_fence_wrapper` | 100% |

**Mechanism (from the execute.py log):** the index path's *first* stage works — it finds the
`cli` area and narrows to the single candidate `cli/reference.md`. But the *final* selection call,
`ask_ai_for_relevant_files(diff, [cli/reference.md])`, returns NONE. The full-scan path calls the
**same function** on all 3 files and returns `cli/reference.md`. Same model, same diff, same
`cli/reference.md` preview (20 lines → full content in both paths, no summary). The only
difference is candidate-set size: **1 candidate → NONE, 3 candidates → correct pick.** Likely the
ULTRA-CONSERVATIVE selection prompt (`_FILE_SELECTION_PROMPT_TEMPLATE`, "prefer returning NONE")
is more likely to bail when it has a single borderline candidate and no comparative context.

**Why this matters:**
- It is **reproducible and systematic** (0/3), not LLM noise.
- It **vindicates D-04** (score the two paths separately) on case 1 — a blended score reads 0.5
  and hides that the index path specifically fails.
- It is almost certainly **new** — the REPL exercised full-scan, so never saw it.
- It is **not a harness artifact**: Stage 1 found the right area, the candidate content loaded,
  the real pipeline's own log printed "No relevant files found."

**Status:** ONE case. Could be case-001-specific or a general index-path weakness. **Next:**
broaden the corpus (esp. more positives) and see if the pattern holds. If it generalizes, it's a
reportable finding for the maintainer (the index optimization trades recall) and direct motivation
for the retrieval-quality work the eval was built to measure. Do NOT report upstream on n=1.

### Update — 3-case run (corpus-v1, N=3 each), refined characterization

| case | index | scan | content | fence |
|---|---|---|---|---|
| 001 add_cli_flag (pos) | **0/3** (selected=[]) | 3/3 (cli/reference.md) | 3/3 | 3/3 |
| 002 internal_rename_trap (neg) | 3/3 (selected nothing - correct) | 3/3 (correct) | **0/3** | 3/3 |
| 003 add_config_option (pos, multi-candidate) | **0/3** (selected=[]) | **0/3** (selected=[]) | 3/3 | 3/3 |

**Single-candidate hypothesis REFUTED.** 003 narrows to a 3-candidate `config/` folder and the
index path STILL returns empty — and so does full-scan. So 001's index miss is not about
candidate count.

**Refined finding — discovery has a recall bias from the conservative prompt.** Across cases the
failure is always `selected=[]` (a pure recall miss), never a wrong-sibling pick. `cli/reference.md`
(literal "flag -> CLI doc" link) gets caught by full-scan; `config/reference.md` (conceptual
"parser-code change -> config-keys doc" link) is missed by BOTH paths. Root cause: the
`_FILE_SELECTION_PROMPT_TEMPLATE`'s ultra-conservative "documents the EXACT code being changed" /
"prefer NONE" framing rejects docs that should update when the doc is user-facing rather than a 1:1
mirror of the changed code file. **This is a precision/recall TRADEOFF, not a bug** — conservatism
is intentional (over-selection is the worse UX, per maintainer). The eval's value is QUANTIFYING
the recall cost, and showing the index path pays it harder than full-scan. Directly motivates the
retrieval-quality work (better prompt / embedding retrieval) the eval exists to measure.

**Second finding — content-gen breaks conservatism on name-overlap (002).** Selection correctly
resisted the trap on both paths (3/3 selected nothing), but content-gen, forced to run on the
trap's gold file (D-20), produced an update instead of `NO_UPDATE_NEEDED` (0/3). In production this
is gated by selection (which chose nothing), so no bad update ships — but content-gen's STANDALONE
`NO_UPDATE_NEEDED` logic is not robust to private-vs-public name overlap. Real defense-in-depth gap.

**Still n=3.** Need the fuller corpus (D-07, ~10 balanced) before any of this is reportable
upstream. But three cases already show coherent, reproducible patterns — the harness works.

### Update — 10-case run (corpus-v2) + a CORPUS-AUTHORING LESSON

Ran the full 10-case corpus. Raw overall: annotations_present 100%, selection_f1_index 40%,
selection_f1_scan 50%, content_assertions 60%, no_fence 100%. **But the peek (`_diag.py`) caught
a fixture confound before any of it could be written up:**

**Lesson — doc-folder names must not collide with code-file basenames.** Case 004 (`docs/api/`
+ diff on `src/api.py`) produced `selected=['api.py/reference.md']` — the model mashed the code
file `api.py` into the doc path. Several cases had this collision (001 `cli/`+`cli.py`, 003/006/010
`config/`+`config.py`, 004 `api/`+`api.py`). It confounds the recall measurement — you can't call a
miss "model conservatism" when a folder/file name collision is muddying the path. **Fixed by
renaming the code file in those 5 diffs** (`main.py`/`loader.py`/`cache.py`); docs + frozen indexes
untouched, so no re-prep. **Rule for authoring future cases: the doc folder name must differ from
every changed code-file basename.** D-15's filter-drops (8) also flagged the model selecting code
files that the suffix filter silently drops — a real recoverable signal that some "empty selections"
are actually code-file picks.

**What survives the confound (clean evidence):** 008 (no collision, conceptual link, both paths
miss) = real recall weakness; 006 caught `config/reference.md` but missed `guide/migration.md` =
real multi-target gap; 005 selection took the vocab-trap bait; content-gen broke on every
name-overlap trap (002, 005, 009). These don't depend on the collision.

**Pending:** clean re-run (corpus-v3) after the diff fixes → the real baseline. Do NOT cite
corpus-v2's raw numbers; they're confounded.

### Clean baseline — corpus-v3 (artifact-free), gpt-4o-mini, N=3

Overall: annotations_present 100%, selection_f1_index 40%, selection_f1_scan 43%,
content_assertions 60%, no_fence 100%. **filter-drops dropped 8 → 0** (the collision fix removed
the code-file path-mangling — confirms the collisions caused it).

**THE finding — discovery is lexically driven.** Renaming case 001's code file `cli.py` → `main.py`
(nothing else) dropped its scan recall 3/3 → 1/3. With lexical name bridges removed across the
corpus, **all 5 positives collapse to ~0 recall** (001 barely fires; 003/004/006/008 miss both
paths). The 40%/43% overall is carried ENTIRELY by correct negatives — the system "scores" by
doing nothing. So: the model finds the doc to update mainly when the changed code-file name
overlaps the doc path; real refactors often lack that overlap → recall collapses. This is the
clean, central result, and the direct motivation for the retrieval-quality work (better discovery
prompt / embeddings) the eval exists to enable.

Secondary (stable across v2/v3): conservatism holds on negatives (002/007/009/010 correct) except
the vocab trap 005 (selection took the bait); content-gen breaks on every name-overlap trap
(002/005/009) but updates correctly when handed the right file (positives 3/3 content) and resists
clean negatives (007/010).

**Caveat for the writeup:** these are gpt-4o-mini numbers. The harness is the deliverable; the
baseline is this model's behavior and a stronger model would likely score higher. Present as
"preliminary observations," not a verdict on code-to-docs in the abstract. **Build is DONE** —
remaining for PR 1 is cleanup + cost estimate + PR description.
