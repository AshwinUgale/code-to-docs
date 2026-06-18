# runs-layout.md — the directory contract `execute.py` must produce

This is the **output spec for our runner**. `score.py` reads cases from this layout;
`execute.py` writes it. Get this exactly right or scoring silently finds nothing.

Source: `score.py` `_get_runs_dir` (L34-41), `_get_case_dirs` (L1402-1407),
`load_case_record` (L56-248).

## The two roots — keep them separate

There are **two** directory trees, joined by `case_id`:

1. **Dataset** (`config.dataset.path`, e.g. `eval/dataset/cases/`) — the source of truth,
   version-controlled. Holds the inputs and the answer key per case: `diff.patch`, `docs/`,
   `input.yaml`, `annotations.yaml`, frozen `.doc-index/`.
2. **Runs** (`$AGENT_EVAL_RUNS_DIR`, default `eval/runs/`) — per-execution OUTPUT, throwaway.
   Holds what the pipeline produced for each case on a given run.

`score.py` reads outputs from the **runs** tree but reads `annotations.yaml` from the
**dataset** tree (`load_case_record` L73-81: `dataset_root / case_id / "annotations.yaml"`).
The `case_id` (directory name) must match in both trees. **This is the key design fact:**
our runner writes only outputs to runs/; the answer key stays in dataset/.

## Runs tree layout

```
<runs_dir>/<run_id>/                       runs_dir = $AGENT_EVAL_RUNS_DIR or eval/runs,
│                                          then / <skill>  (skill = eval.yaml `skill:` field)
├── cases/
│   └── <case_id>/                         must equal the dataset case dir name
│       ├── <output.path>/                 one dir per eval.yaml outputs[].path entry
│       │   └── <files>                    e.g. selected/selected_files.json, content/<doc>
│       ├── events.json                    optional (traces.events)
│       ├── stdout.log                      optional (traces.stdout)
│       └── stderr.log                      optional (traces.stderr)
├── run_result.json                        optional, RUN-level metrics (not per-case)
└── summary.yaml                           written BY score.py, not us
```

Note `runs_dir` is suffixed by the **skill name**: with `skill: code-to-docs` and the default,
cases live at `eval/runs/code-to-docs/<run_id>/cases/<case_id>/`. (Confirmed by the spike:
`AGENT_EVAL_RUNS_DIR=runs` + `skill: code-to-docs` → `runs/code-to-docs/<run_id>/...`.)

## How outputs map into the judge's `record` dict

`load_case_record` builds the dict each judge receives as `outputs=`:

- For each `config.outputs[].path` dir under the case: every file is read into
  `record["files"][<relpath>]` (L106-123, `rglob`).
- Convenience key: `record["<dirname>_content"]` = text of the **first** file in that output dir
  (L126-140). So output `path: content` → `record["content_content"]`. Our spike judge used
  `outputs.get("content_content")` and `outputs["files"]["selected/selected_files.json"]`.
- `record["annotations"]` = parsed `annotations.yaml` from the dataset dir (L80).
- `record["exit_code"|"cost_usd"|"duration_s"|"token_usage"|"num_turns"]` from run-level
  `run_result.json` (L164-177) — only if `traces.metrics: true`.
- `record["events"]`, `record["conversation"]`, `record["stdout"]`, `record["stderr"]` — only
  if the matching `traces.*` flag is on. We don't need these (our outputs are files, not an
  agent transcript), so set `traces: {events: false, stdout: false, stderr: false}`.

## Implication for execute.py

`execute.py` must, per case per sample-run: create
`eval/runs/code-to-docs/<run_id>/cases/<case_id>/<output-dir>/<file>`, write the pipeline's
selection JSON + generated content there, and (optionally) a run-level `run_result.json`.
It does **not** touch annotations — those live in dataset/ and score.py finds them by case_id.
