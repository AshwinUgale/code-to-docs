# Maintainer Comment Thread — verbatim

The 4-round exchange that produced the approved plan. Speakers labelled. Preserve verbatim — when a future Claude or Ashwin needs to verify "what did csoceanu actually say," this is the source. Don't paraphrase from this; cite it.

---

## Round 1 — @AshwinUgale opens the proposal

### The problem

I had a few ideas for improving precision/recall on file selection. None of them are worth proposing yet, because there isn't a way to tell whether a change actually helps. The existing tests cover plumbing — parsing, path validation, truncation, git ops — but nothing measures what gets selected or what the LLM produces. So today every quality change is evaluated by eyeballing a few PRs.

I'd like to try landing an eval harness first so any future quality PR can be backed by numbers.

### Method I'm thinking of

Two evaluators, one per pipeline stage.

**File selection**

A "case" is a folder containing `diff.patch`, a minimal `docs/` tree, and an `expected.json` listing the files the pipeline should have selected. The runner builds a throwaway git repo in `tmp_path`, materializes the docs tree, applies the diff against a base commit, points env at a real model, and calls `find_relevant_files_optimized(diff)`. The returned set is compared against the expected set to compute precision (of files selected, how many were right), recall (of files we should have caught, how many we did), and F1, per case and aggregate.

The two code paths inside `find_relevant_files_optimized` — the index-based fast path and the full-scan fallback — get scored separately by running each case twice, once with `.doc-index/` populated and once with it removed. Otherwise a regression on one path can be hidden by improvement on the other.

**Content generation**

The case folder adds `assertions.json` with concrete claims about each file's output:

```json
{
  "docs/cli/reference.md": {
    "must_mention": ["--verbose"],
    "must_not_mention": ["--debug"],
    "must_not_add_sections": true,
    "max_new_lines": 20
  }
}
```

`must_mention` catches missing updates, `must_not_mention` catches a specific class of hallucination (naming things the diff didn't introduce), `must_not_add_sections` enforces the "be conservative" instruction by comparing heading counts, `max_new_lines` bounds scope creep.

Global assertions run on every case regardless of contract: the output parses cleanly under the format declared by extension, code fences balance, and the first non-blank line isn't ``` (the prompts in `ask_ai_for_updated_content` forbid wrapping the file in a fence, but the LLM still does it sometimes).

The runner calls `ask_ai_for_updated_content` per case, runs per-case assertions for that file, then global assertions. Each is a pass/fail. Aggregate is the fraction of assertions passing across the corpus, broken out by assertion type so regressions on one specific kind are visible.

**Frozen baseline + CI**

After the first run, the aggregated numbers (F1 per path, per-assertion-type pass rates) get written to `baseline.json` and committed. Every subsequent CI run computes new numbers, diffs against baseline, posts the delta on the PR. Drops beyond a configurable tolerance fail the job. Baseline bumps are explicit commits when an intentional improvement lands.

### Honest caveat

Whether this is the right method, I genuinely don't know yet. Assertions are cheap and deterministic but they're an indirect proxy for "is this update actually good," and the only way to find out how much of the real failure surface they catch is to write some cases and look at what gets through. I'd rather try it on a small corpus, see whether the scores correlate with good output by eye, and then decide whether to keep this approach or switch to something else. If the first ~10-15 cases show the harness missing obvious failures, I'll write up what didn't work and move to an LLM-as-judge approach against gold updates.

Does this direction make sense before I start? And if so — is there a docs repo that's been an active code-to-docs target? Mining merged docs-PRs from it would give much higher-signal cases than synthetic ones.

---

## Round 2 — @csoceanu responds

Hey @AshwinUgale, thanks for the detailed proposal! The gap you identified is real, and the direction makes sense.

Before building everything from scratch, it might be worth taking a look at a few repos we contribute to, they may have relevant patterns or components you can reuse:

- **[agent-eval-harness](https://github.com/opendatahub-io/agent-eval-harness)** - generic eval framework with deterministic check judges, LLM-as-judge, threshold-based regression detection, and test cases as directories.
- **[eval-hub](https://github.com/eval-hub/eval-hub)** - evaluation orchestration service with pluggable providers (BYOF) for model benchmarking.
- **[sdg_hub](https://github.com/Red-Hat-AI-Innovation-Team/sdg_hub)** - composable synthetic data generation framework with LLM-as-judge evaluation flows.

A couple of things worth considering:

- The proposed assertions only cover cases where the file should be updated. It would be worth including negative cases too - diffs unrelated to a doc file where the correct result is `NO_UPDATE_NEEDED`. The pipeline's conservatism is a key feature (the prompts in `ask_ai_for_updated_content` push hard toward not updating), and regressions there would mean unnecessary or hallucinated updates.
- LLM non-determinism could make results flaky - the same test case might pass one run and fail the next. For file selection especially, the model might pick the right files 7 out of 10 times. Worth thinking about whether cases should run multiple times with a pass threshold, or how to handle that in the baseline comparison.

On your question about a docs repo - there's no repo that's been an active code-to-docs target yet, so there are no merged docs-PRs to mine cases from. But any code repo with documentation should work, whether the docs are in the same repo or a separate one. Code-to-docs handles both scenarios.

cc @HofniGartner

---

## Round 3 — @AshwinUgale responds

Thanks @csoceanu — really useful pointers, especially agent-eval-harness it sounds very close to what I was proposing, so my first move is to read it carefully and figure out what can I borrow from these repos.

On the two feedback points:

**Negative cases.** You're right, and I undersold this. The `NO_UPDATE_NEEDED` path is a first-class case class, not an afterthought — for both stages. For selection: diffs that touch code with no doc impact (refactors, dep bumps, internal renames), expected set is empty. For content: a diff that's in scope of a doc file but doesn't change anything that file actually documents, expected output is the literal string `NO_UPDATE_NEEDED`. The starter corpus should be roughly balanced between "should update" and "should not update" cases, since over-eagerness is the more user-visible failure.

**Non-determinism.** Plan is to run each case multiple times and track pass rate instead of single pass/fail — e.g. N=3 on every PR for fast feedback, full N=10 on a scheduled daily run for accurate numbers. Happy to go another way if you have a preference.

**Corpus.** Got it, no mining target. I'll seed with synthetic cases drawn from generic open-source code repos. Case format will be structured so real mined cases can drop in later if a target repo emerges.

I'll go through agent-eval-harness (and skim the other two) and come back with an updated approach.

---

## Round 4 — @AshwinUgale returns with the refined plan

Went through the repos. Here's the plan I'd like a green light on before raising the first PR.

**How I'd use agent-eval-harness**

Since code-to-docs runs as a plain Python entry point rather than a CC skill, the harness's runner layer isn't a fit for us — but the scoring side (`score.py` + the `agent_eval` package) looks usable as-is: it reads case outputs from a runs directory, applies judges from `eval.yaml`, and already covers everything I was about to hand-build — inline `check` judges, `module/function` judges, LLM judges (incl. pairwise), per-judge thresholds, and the `regression` mode that exits non-zero on threshold/baseline violations. Core deps are just `pyyaml` + `jinja2`.

I verified this locally with 2 hand-authored cases and outputs before proposing it — `score.py` ran our judges and the regression gate unmodified, passing good outputs (exit 0) and catching deliberately broken ones (exit 1).

So the plan: **use the harness's eval.yaml schema + scoring/regression as a dependency, and write only the execution layer ourselves.**

On the non-determinism point you raised: the `samples: N` knob applies per judge — it re-runs a stochastic LLM judge on the same output. There's no equivalent for running the *system under test* N times per case, which is where code-to-docs' randomness lives. Our runner will handle that (run each case N times, track pass rates), and if it proves useful I'm happy to offer it upstream.

## The PR I want to raise

Three pieces, all new files — no changes to existing code-to-docs code:

- **A small runner** that, per test case, builds a temp git repo (docs tree + diff applied), invokes the real pipeline functions (`find_relevant_files_optimized`, `ask_ai_for_updated_content`), and writes outputs in the harness's runs layout. Supports N runs per case for the stochasticity tracking above.
- **A selection judge** — precision/recall/F1 against each case's expected file set, with the index path and the full-scan fallback scored separately.
- **~10 test cases**, balanced between should-update and `NO_UPDATE_NEEDED`. Per-case content assertions (`must_mention`, `must_not_mention`, `max_new_lines`, `must_not_add_sections`) live in each case's `annotations.yaml`; global format checks (parses as declared format, balanced fences, no leading ``` wrapper) are shared `check` judges. The LLM-judge slot already exists in the harness schema — stays empty in v1, zero schema work when gold references are added later.

This first PR is self-contained — everything runs locally (execute → score → regression), nothing touches CI yet.

In a follow-up PR I'll add the CI workflow: a quick subset on every PR for fast feedback, the full corpus on a daily schedule, both failing the build if scores drop below the baseline. Keeping it separate because the baseline can only be generated after this PR's runner has run against the real corpus.

## Two questions

1. For pulling in the harness's scoring code: would you rather code-to-docs take it as a test-only dependency (pip install from their git repo), or copy the scoring files into this repo so there's no external dependency?
2. Where should the eval live — `eval/` at the repo root (the harness convention) or under `tests/eval/`?

If this looks right I'll start on the PR.

---

## Round 5 — @csoceanu approves with conditions

@AshwinUgale Thanks for going through the repos and the detailed plan.

I like the split of writing a custom runner and reusing existing scoring. On your dependency question, I think it would be better to copy the relevant scoring and judge files into the code-to-docs repo rather than taking it as a pip dependency. The harness's config schema is designed for Claude Code skill eval, most of it doesn't apply here. That said, the scoring code has internal dependencies between files (`score.py` imports from `config.py`, `events.py`, `judges/`, etc.) — so there will be some untangling to strip out what you don't need. Worth scoping that before committing to the copy approach, and if it's too entangled, a simpler self-contained implementation might be less work.

Good to see the balanced corpus with `NO_UPDATE_NEEDED` cases — that addresses the concern we raised.

For location — `eval/` at the repo root makes sense.

**Side effects in the pipeline functions.** `find_relevant_files_optimized` does `git fetch origin`, `build_all_indexes` (multiple LLM calls to summarize each doc folder), and `commit_indexes_to_repo` (git push to main). In a throwaway temp repo with no remote, the git operations will fail. But the harder problem is indexing — as I see it we have three options: (a) mock indexing entirely, but then you're not testing the real discovery path, (b) run indexing and patch out the git push — works but expensive per case, (c) pre-build indexes once and copy them into each temp repo — cheapest, but you're testing discovery with frozen indexes, not the full pipeline. Each option tests a different thing, and the choice should be deliberate. The full-scan fallback path (`ask_ai_for_relevant_files`) is cleaner — no git side effects, just LLM calls — so that one should be straightforward to test.

Whichever way you go, worth estimating cost per eval run before committing to CI — 10 cases × N=10 with LLM calls at each stage adds up.

You can go ahead with the PR.

cc @HofniGartner

---

## Auto-triage summary (bot)

> `<!-- fullsend:triage-agent -->`
>
> **Category:** Feature — evaluation harness for pipeline quality measurement
> **Severity:** Medium
>
> ### Agreed Plan (from discussion)
>
> The proposal has been iterated through multiple rounds of feedback and received maintainer approval:
> - **Runner** in `eval/` at repo root
> - **Scoring** — copy relevant scoring/judge code from agent-eval-harness rather than pip dependency (with fallback to self-contained implementation if untangling is too costly)
> - **Selection judge** — precision/recall/F1 against expected file sets, index path and full-scan fallback scored separately
> - **~10 test cases** — balanced between should-update and `NO_UPDATE_NEEDED` cases
> - **N runs per case** for non-determinism (N=3 on PRs, N=10 on daily schedule)
>
> ### Open Technical Considerations
>
> The maintainer flagged that `find_relevant_files_optimized` has side effects (`git fetch origin`, `build_all_indexes`, `commit_indexes_to_repo`) that need deliberate handling in the test environment. Three options were outlined — the choice should be documented in the PR.
>
> LLM cost per eval run should be estimated before the follow-up CI integration PR.
>
> This issue is ready for implementation. The first PR should be self-contained (no CI integration), with CI workflow following in a separate PR once baselines are established.
>
> **Labels:** enhancement

---

## Round 6 — @AshwinUgale closes the loop

Thanks @csoceanu

**Copy vs dependency.** Traced score.py's imports — the core we need (check + module judges, thresholds, regression) only pulls in `config.py` and `events.py`, both stdlib+pyyaml with no internal cross-imports. The LLM/MLflow/builtin machinery is all lazily imported behind branches we don't hit. So a straight copy into `eval/scoring/` works cleanly, imports rewired local. Confirmed `config.py` doesn't require the CC-skill schema fields — the spike's minimal eval.yaml ran without them.

**Indexing side-effects.** Going with option (c) — pre-built frozen indexes per case, committed into the fixture, with `fetch_indexes_from_main` and `commit_indexes_to_repo` stubbed to no-ops. A file-selection eval is about index consumption, so I'd rather not bake index-build variance and cost into every run. Temp repo gets `git init` + base commit + diff, `PR_BASE` at the base SHA, so `get_diff` works with no remote. Full-scan path needs none of this and gets tested separately.

I'll include a per-run LLM call-count estimate so cost is concrete before the CI PR. Starting now.

---

## What's pending from this thread

| Owed in the PR description | Source |
|---|---|
| Document option (c) choice + (a) and (b) rejected with reasoning | csoceanu Round 5 enumerated the three options |
| Per-run LLM call-count estimate | csoceanu Round 5 + Ashwin Round 6 commitment |
| Note that scoring code is a straight copy with provenance | csoceanu Round 5 + Ashwin Round 6 |
| Mention the spike (2 cases, regression-gate proven) | Ashwin Round 4 |
| @HofniGartner — silent in thread, cc'd twice by csoceanu; ping in PR | — |
| The `samples:N` per-judge gotcha — call out in code comment | Ashwin Round 4 said it; no response from csoceanu (tacit accept) |
