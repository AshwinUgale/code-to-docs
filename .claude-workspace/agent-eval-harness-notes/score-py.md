# score-py.md — the file we copy: modes, flow, exit codes

`skills/eval-run/scripts/score.py` — the scoring engine. Execution-agnostic: it only reads
files from the runs dir + dataset dir, never runs the system under test. This is why we can
reuse it with our own `execute.py`.

## CLI modes (L9-11)

```
score.py judges     --run-id <id> --config eval.yaml     # score all cases, print per-judge agg
score.py pairwise   --run-id <id> --baseline <id> ...    # LLM A/B vs a baseline run (unused v1)
score.py regression --run-id <id> --config eval.yaml     # gate: exit 1 if thresholds violated
```

We use **`judges`** (see the numbers) and **`regression`** (the CI gate). `pairwise` is v2.

## Top-level imports (the copy surface)

Only `agent_eval.config` (top, L31) and `agent_eval.events` (lazy, L200/L355). Plus stdlib +
`yaml`. The heavy stuff is all lazy behind branches we don't hit: `jinja2` (L336, LLM judge),
`mlflow.genai` (L943), `anthropic` (L1185-1189), `BuiltinJudgeRegistry` (L413). So a straight
copy of `score.py` + `config.py` + `events.py` runs our `check`/`module` judges without ever
importing the LLM/MLflow deps. (This is the import trace that cleared D-01's hinge condition.)

## Scoring flow (`score_cases`, L750-843)

1. `_get_case_dirs(run_id, runs_dir)` (L1402) → `runs_dir/<run_id>/cases/*` subdirs. Exits 1 if
   the cases dir is missing.
2. Parallel over cases (ThreadPoolExecutor, workers = min(#cases, cpu_count)).
3. Per case: `load_case_record` builds the `record` dict; each judge's `scorer(outputs=record)`
   runs; result stored as `{value, rationale, judge_type}`.
4. Aggregate per judge across cases: numeric → `mean` (fmean), bool → `pass_rate`.

## Regression detection (`detect_regressions`, L1363-1395) + exit code

Two independent checks per judge named in `thresholds`:

- **Absolute floor** (the real gate): `min_pass_rate` / `min_mean` / `min_win_rate`. If the
  current aggregate is below the floor → regression.
- **Baseline delta** (only if `--baseline` passed): regression if
  `current < baseline - 0.5` (L1391). **Note the 0.5 absolute tolerance** — on a 0..1 F1 or
  pass_rate scale a 0.5 drop is enormous, so the baseline-delta check is very loose. **Tight
  gating must come from the absolute `min_*` thresholds, not the baseline delta.** Relevant when
  we design the PR-2 baseline gate (D-09): set real `min_mean`/`min_pass_rate` floors; don't
  lean on the baseline comparison.

Any regression → `cmd_regression` prints them and the process **exits 1** (proven by the spike:
broken run → 3 regressions → exit 1; good run → exit 0). This non-zero exit is what makes a CI
gate real.

## What score.py writes

`summary.yaml` at `runs_dir/<run_id>/summary.yaml` (merged via `_merge_summary` L1410). We don't
author it; it's score.py's output. Our runner should not create it.
