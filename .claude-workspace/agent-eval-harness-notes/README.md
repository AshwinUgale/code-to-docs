# agent-eval-harness-notes — file-by-file notes

Notes built as we read `opendatahub-io/agent-eval-harness` (local reference clone at
`contrib/agent-eval-harness/`).

**Status: core scoring path read and documented (2026-06-18 session).** Enough to write
`execute.py` against. The LLM-judge / MLflow / builtin-registry surfaces are deliberately
un-read — they're dormant for our v1 (deterministic `check` + `module` judges only).

## Files

- `runs-layout.md` — **the contract `execute.py` must produce.** Dataset tree vs runs tree, how
  output dirs map into the judge `record` dict, where `annotations.yaml` is read from. Start here.
- `score-py.md` — the copied engine: CLI modes (`judges` / `regression`), scoring flow, the
  import surface that makes the straight copy work, regression detection + exit codes (incl. the
  0.5 baseline-delta tolerance finding).
- `judges.md` — the four judge types + dispatch, what a judge receives/returns, the `condition`
  skip, and the `samples` gotcha (per-judge, not per-SUT — validates D-08).
- `config-py.md` — eval.yaml → EvalConfig; minimal valid config; why `skill` is the one field
  that matters; path resolution. Validates D-01 (CC-skill fields not required). Folds in the
  planned `eval-yaml-schema.md`.
- `events-py.md` — informational; transcript parsing we don't exercise. Copy it (score.py imports
  it), ignore it.

## Cross-refs into DECISIONS.md

- `runs-layout.md` → spec behind D-11 (temp repo) and the execute.py design (D-18, TBD).
- `judges.md` samples section → **validates D-08** (SUT-N-loop is our runner's job).
- `score-py.md` import surface → **cleared D-01's hinge** (copy is clean, no rewrite).
- `score-py.md` 0.5 baseline tolerance → input to D-09 (PR-2 gate must use absolute `min_*`).

**When reading more source:** add/extend a `*.md` here, 200-500 words, technical, with line refs.
