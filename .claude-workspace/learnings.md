# learnings.md — Ashwin's study material

Curated study material for the concepts behind this PR. Each topic: high-level overview + why it matters here + dig-deeper if criterion + links. Status flag tells Ashwin what he's covered.

**Read order:** topics 1-6 are eval theory (read before/during execute.py work). Topics 7-10 are this-PR-specific (read before/during coding). Order within each block doesn't matter much.

**Status values:** `not-started` / `read-overview` / `digging` / `done`.

---

## How to use this file

1. Read the **overview** for each topic — 2-5 minutes per topic.
2. Skip "dig deeper" unless you hit a concrete need (writing the relevant code, debugging, the maintainer asks something specific). Don't pre-emptively read papers.
3. When you do dig deeper, write what you learned into the topic — these notes become your reference.
4. Mark status when you change levels.

---

## 1. What an eval actually is

**Status:** not-started

**Overview (~150 words):**

An eval is a controlled loop: feed a fixed input to a system, capture its output, compare against an expected answer or assert properties about it, score the comparison, aggregate across many inputs. That's it — no magic.

Four moving parts:
- **Cases**: input + expected output (or expected properties)
- **System under test (SUT)**: the thing you're measuring — here, code-to-docs's discovery and generation
- **Judge**: turns (output, expected) into a score (pass/fail, 0-1, or a number)
- **Aggregator**: turns N judge results into a summary (pass rate, mean, F1)

The whole point: make quality changes measurable instead of vibes-based. Without an eval, every prompt-tweak PR is "looks better to me," which is unreviewable. With an eval, you say "F1 went 0.71 → 0.78, content-assertion pass rate went 84% → 91%, regression gate clean."

**Why this matters for PR 1:** the harness IS this loop. `execute.py` runs the SUT, `eval/scoring/score.py` runs the judges, `eval/judges/selection.py` is one specific judge. `eval.yaml` declares how they connect.

**Dig deeper if** you want intuition for why this looks the way it does (esp. why agent-eval-harness has its specific separation of execute vs score vs aggregate).
- [Anthropic's eval cookbook intro](https://docs.claude.com/en/docs/build-with-claude/develop-tests) — the simplest intro to LLM eval design
- [HumanEval paper](https://arxiv.org/abs/2107.03374) — the canonical example of code-generation eval; not the format we're using but the mental model is clear

---

## 2. Precision, recall, F1

**Status:** not-started

**Overview (~150 words):**

Standard information-retrieval metrics. Given a system that selects a set of items and a known correct set:

- **Precision** = how many of what we selected were right = `|selected ∩ correct| / |selected|`
- **Recall** = how many of the correct items we found = `|selected ∩ correct| / |correct|`
- **F1** = harmonic mean of precision and recall = `2·P·R / (P+R)`

High precision, low recall: we don't select much, but when we do we're usually right. High recall, low precision: we cast a wide net and catch most correct items but bring noise.

Edge case for our harness: empty expected set (`NO_UPDATE_NEEDED` case). If the SUT also selects nothing → both P and R = 1.0 by convention. If the SUT selects anything → both = 0.0.

**Why this matters for PR 1:** the selection judge IS precision/recall/F1. Need to feel these in your bones because every regression we catch shows up as a number here.

**Dig deeper if** the empty-set case starts producing weird aggregate scores. The convention you pick (1.0 vs undefined vs skip-from-aggregate) materially affects how regressions show up.
- [Wikipedia: Precision and recall](https://en.wikipedia.org/wiki/Precision_and_recall) — short and adequate
- [Skip if comfortable]

---

## 3. LLM-as-judge vs deterministic assertions vs reference-based eval

**Status:** not-started

**Overview (~150 words):**

Three strategies for judging LLM output, with different cost/signal tradeoffs:

- **Deterministic assertions** (what we're using in v1): Python rules over the output — does it mention X, parse as Markdown, stay under N lines. Cheap, fast, fully reproducible. Misses semantic quality.
- **Reference-based eval**: compare LLM output against a hand-written "gold" answer using a similarity metric (BLEU, ROUGE, exact match, embedding similarity). Better signal but expensive to author and brittle to acceptable rewrites.
- **LLM-as-judge**: ask another LLM to score the output, often pairwise (which of A/B is better) or rubric-based (rate 1-5 on dimensions). Captures semantic quality, but introduces judge variance, costs $$, and can have systematic biases (preference for longer outputs, sycophancy).

**Why this matters for PR 1:** csoceanu approved v1 = assertions only. The eval.yaml schema reserves an LLM-judge slot for v2. If first 10-15 cases show the harness missing failures the eyeball test catches, we switch to LLM-as-judge against gold refs. That's the v2 trigger condition Ashwin committed to in Round 1.

**Dig deeper if** v1 results don't correlate with eyeball quality (the trigger to plan v2).
- [Anthropic's evaluating outputs guide](https://docs.claude.com/en/docs/build-with-claude/grading-evaluations)
- [LLM-as-judge bias paper (G-Eval)](https://arxiv.org/abs/2303.16634) — known biases of LLM judges

---

## 4. SUT variance vs judge variance (the `samples:N` gotcha)

**Status:** not-started

**Overview (~150 words):**

Two distinct sources of randomness in an LLM eval:

- **SUT variance**: the system under test (code-to-docs's LLM calls) is stochastic. Same diff in, different files selected out across runs. Same docs + diff in, slightly different rewritten content out.
- **Judge variance**: if the judge is itself an LLM, *it* is stochastic. Same output + reference in, different score out.

These are independent. You have to measure both separately, otherwise you can't tell whether a regression is in the SUT or just judge noise.

`agent-eval-harness` has a `samples:N` knob in `eval.yaml`. It runs the *judge* N times on the same output to average out judge variance. **It does NOT re-run the SUT.** Our runner has to handle SUT-variance averaging itself: invoke `find_relevant_files_optimized` N times per case, then judge each output, then track pass rate across the N invocations.

**Why this matters for PR 1:** if you don't loop the SUT in `execute.py`, the harness reports flaky single-shot pass/fail and the maintainer will see results bouncing 7/10 → 5/10 → 8/10 with no actual code change. This is what made the maintainer raise non-determinism in Round 2.

**Dig deeper if** you're tuning N or noticing weird aggregate behavior. The math on how many samples you need to detect a regression with confidence is real (binomial test).
- [Wikipedia: binomial proportion confidence interval](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval) — for sizing N

---

## 5. Regression gates and baselines

**Status:** not-started

**Overview (~150 words):**

A regression gate is a CI-time check that compares current metrics against a committed baseline. If current is meaningfully worse than baseline, fail the build. Anatomy:

- **Baseline**: a committed JSON/YAML file containing reference metric values (F1=0.78, pass_rate=0.87, etc.).
- **Tolerance**: the amount of allowed drop before failing (commonly 5% relative, or 2 standard deviations of historical variance).
- **Gate logic**: compute current metrics, compare to baseline ± tolerance, exit 0 (pass) or non-zero (fail) accordingly.
- **Baseline bumps**: when an intentional improvement lands, the PR includes an updated baseline file. The bump is explicit and reviewable.

`agent-eval-harness` has this as `score.py regression --run-id X --baseline path/to/baseline.json`.

**Why this matters for PR 1:** this is what PR 2 (CI wiring) hinges on. PR 1 ships the harness; PR 2 generates the first baseline and wires the gate. The harness's `regression` mode is one of the three reasons we're copying agent-eval-harness instead of building our own.

**Dig deeper if** you're designing the tolerance numbers for PR 2 (not in PR 1 scope).

---

## 6. agent-eval-harness mental model

**Status:** not-started

**Overview (~150 words):**

Three layers, cleanly separated:

1. **Execution layer**: invokes the SUT, writes outputs to a runs directory. For Claude Code skill evals, this means spawning a CC subprocess. **For us, this is `execute.py` — we write our own because code-to-docs isn't a CC skill.**

2. **Scoring layer** (`score.py` + `agent_eval/` pkg): reads case outputs from the runs directory, applies judges declared in `eval.yaml`, supports `judges` / `pairwise` / `regression` subcommands. Execution-agnostic. **This is what we copy.**

3. **`eval.yaml` schema**: declarative config — judges (inline/module/LLM/builtin), thresholds (min_pass_rate, min_mean, min_win_rate), `samples:N` for judge variance.

The runs directory layout (Ashwin will verify when reading the source) is what `execute.py` writes and `score.py` reads — that's the contract between our code and the copied code.

**Why this matters for PR 1:** understanding this separation is what unlocked the whole approach. If you don't see this clearly, you'll either pull in more of the harness than you need or write things that conflict with the contract.

**Dig deeper if** anything about `eval.yaml` judge declaration confuses you, or when reading `score.py` to understand the contract.
- The agent-eval-harness `README` + `skills/eval-run/` directory (in `agent-eval-harness/agent-eval-harness/`)
- `agent-eval-harness-notes/` will get populated when we read the source together

---

## 7. code-to-docs discovery pipeline (recap)

**Status:** read-overview (already covered by walkthroughs 10 + 11)

**See:** `code-to-docs-notes/10-walkthrough-review-docs.md` (Steps 5 + the full-scan fallback) and `code-to-docs-notes/11-walkthrough-caching.md` (the two-tier cache + the two stages).

**One-paragraph recap for execute.py:** `find_relevant_files_optimized(diff)` calls into `doc_index.py` to (a) `fetch_indexes_from_main` from git, (b) maybe `build_all_indexes` if missing, (c) ask the LLM which folders are relevant via `find_relevant_areas_from_indexes`, (d) ask the LLM which files within those folders, (e) commit any new indexes via `commit_indexes_to_repo`. Our runner stubs (a) and (e) to no-ops, ships the indexes pre-built per case, so (b) is skipped. The LLM calls in (c) and (d) are the SUT signal we measure.

---

## 8. code-to-docs generation pipeline

**Status:** not-started — pending walkthrough 12

Walkthrough 12 (forthcoming) will cover `ask_ai_for_updated_content`, the format-specific prompts (md/adoc/rst), `truncate_diff`, and the `NO_UPDATE_NEEDED` sentinel. Read that walkthrough when it's written.

**One-paragraph preview for execute.py:** `ask_ai_for_updated_content(diff, file_path, current_content, ...)` returns either the literal string `NO_UPDATE_NEEDED` or the rewritten file content. Our content judges run on this output. The format-specific prompts at `src/generation.py:96-164` are what make the global format checks (parses as md/adoc/rst, balanced fences, no leading ```) meaningful — the prompts forbid fence-wrapping, so when the LLM does it anyway, that's a real regression signal.

---

## 9. Temp-repo + monkey-patch mechanics (the execute.py risk surface)

**Status:** not-started

**Overview:**

`execute.py` per case:
1. Create a temp directory (`tempfile.TemporaryDirectory` for auto-cleanup).
2. `git init` in it.
3. Copy the case's `docs/` tree into the temp dir.
4. Stage + commit the base state (this is "the docs as they exist before the diff").
5. Capture the base commit's SHA. This is what we set `PR_BASE` to in env.
6. Apply the case's `diff.patch` (git apply or python patch lib).
7. Stage + commit the diff-applied state. Now `HEAD` is the post-diff commit, `PR_BASE` is the base commit.
8. Copy the case's frozen `.doc-index/` into the temp dir.
9. `os.chdir(temp_dir)` — code-to-docs uses CWD-relative paths.
10. Monkey-patch `discovery.fetch_indexes_from_main` and `discovery.commit_indexes_to_repo` to no-ops.
11. For N=1..samples: invoke `find_relevant_files_optimized(diff)` and `ask_ai_for_updated_content(...)`. Capture output.
12. Restore CWD, unpatch (or use context manager). Tempdir auto-cleans.

**Why this matters for PR 1:** this is the new code most likely to break. The git init / diff / SHA / chdir sequence has many ways to go subtly wrong (wrong commit, wrong CWD, leaked state, incomplete cleanup).

**Dig deeper if** you're debugging execute.py.
- Python `subprocess.run` + `git apply` docs
- `unittest.mock.patch` context manager idiom

---

## 10. Python `from x import y` binding gotcha

**Status:** not-started

**Overview:**

When you write `from doc_index import fetch_indexes_from_main` in `discovery.py`, Python looks up `fetch_indexes_from_main` in the `doc_index` module's namespace at import time, then **rebinds the name into the `discovery` module's namespace**. After that, `discovery.fetch_indexes_from_main` and `doc_index.fetch_indexes_from_main` are two separate references that happen to point at the same function object.

If you later patch `doc_index.fetch_indexes_from_main = noop`, calls inside `discovery` that go through `discovery.fetch_indexes_from_main(...)` will still hit the original function — they resolve through the local binding, not through `doc_index`.

**Fix:** patch `discovery.fetch_indexes_from_main` directly, or use `unittest.mock.patch('discovery.fetch_indexes_from_main')` which Python resolves to the discovery namespace.

**Why this matters for PR 1:** if Ashwin patches the wrong namespace, the stubs silently no-op and the real git calls run anyway, failing in confusing ways.

**Dig deeper if** you ever see a "patched a function but it didn't take effect" debugging session.
- [Python docs: where to patch](https://docs.python.org/3/library/unittest.mock.html#where-to-patch) — the canonical explanation, short
