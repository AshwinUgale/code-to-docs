# judges.md — judge taxonomy, dispatch, sampling

Source: `score.py` `load_judges` (L376-439), `score_cases` (L750-843),
`_aggregate_samples` (L697-747), `_normalize_result` (L688-694). Config: `JudgeConfig` (L261-296).

## Four judge types (dispatched by which field is set)

`load_judges` decides the type by field presence, in this order (L398-431):

| eval.yaml field(s) | judge_type | how scored |
|---|---|---|
| `builtin: <name>` | `builtin` | resolved via `BuiltinJudgeRegistry` (we don't use) |
| `check: \|` (Python snippet) | `check` | snippet compiled, runs `(outputs, arguments)` → `(bool, str)` |
| `prompt:` / `prompt_file:` | `llm` | LLM call against the output (we reserve, don't use in v1) |
| `module:` + `function:` | `code` | imports `module`, calls `function(outputs=record, **arguments)` |

These are **mutually exclusive** (builtin conflicts validated L403-410). Returns a 5-tuple per
judge: `(name, scorer, condition, judge_type, samples)`.

We use two: **`check`** (inline assertions — must_mention etc.) and **`module`/`function`**
(our `eval/judges/selection.py` precision/recall/F1). Both are deterministic.

## What a judge receives and returns

- Receives: `scorer(outputs=record)` — the single `record` dict from `load_case_record`
  (see runs-layout.md). For module judges, signature is `judge_fn(outputs=None, **kwargs)`.
- Returns: a `(value, rationale)` tuple, OR a `Feedback`-like object with `.value`/`.rationale`,
  OR a bare value (`_normalize_result` L688-694 handles all three).
- `value` may be `bool` (pass/fail) or numeric (a score like F1). Type matters for aggregation
  and thresholds: bool → `pass_rate`, numeric → `mean`.

## `condition` (eval.yaml key `if:`) — per-case skip

`JudgeConfig.condition` (from yaml `if`, L525) is a Python expression eval'd against
`{annotations, outputs}` (L771-788). False → judge skipped for that case, **not counted** in
pass_rate/mean. Useful later (e.g. only run the content judge when `expect_no_update` is false),
but the v1 plan keeps judges unconditional.

## The `samples` gotcha (validates D-08)

`JudgeConfig.samples` (L294-296) reruns a judge N times and reduces (`_aggregate_samples`).
**Critical:** `load_judges` L433-438 forces `samples=1` for any judge that isn't `llm`, with a
warning. And `score_cases` L791-796 only applies N>1 when `judge_type == "llm"`.

So `samples` measures **judge** stochasticity (an LLM judge giving different scores on the same
output), NOT **system-under-test** stochasticity. Our judges are deterministic — `samples` does
nothing for us. Running the *pipeline* N times is **our runner's** job, not this knob. This is
exactly why D-08 says execute.py owns the N-loop. Do not try to use `samples` for SUT variance.

## Aggregation (`_aggregate_samples`, only fires for sampled LLM judges)

- bool values → strict majority (ties → fail), `stability.stable` if unanimous.
- numeric → `median_low` (returns an actually-observed value), records min/max/mean/spread.
- Records `stability.stable=False` + per-sample rationales when samples disagree.

For our deterministic judges this path is bypassed (n=1). We get our cross-run pass-rate from
**our own** N-loop in execute.py producing N runs, each scored once — aggregated at our layer,
not score.py's.
