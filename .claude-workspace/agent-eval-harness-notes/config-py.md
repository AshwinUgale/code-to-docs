# config-py.md — eval.yaml → EvalConfig: required vs tolerated

`agent_eval/config.py`. Parses `eval.yaml` into dataclasses. Confirms **D-01**: the
CC-skill-specific fields are NOT required.

## Everything has a default (L299-356)

`EvalConfig` and every nested dataclass (`DatasetConfig`, `OutputConfig`, `RunnerConfig`,
`ModelsConfig`, `TracesConfig`, `JudgeConfig`, etc.) default every field. `from_yaml` (L369-587)
reads with `raw.get(..., default)` throughout. So a **minimal eval.yaml** is valid:

```yaml
name: code-to-docs-eval
skill: code-to-docs            # REQUIRED in practice — see discovery below
dataset:
  path: dataset/cases
outputs:
  - path: selected
  - path: content
traces: { events: false, stdout: false, stderr: false, metrics: true }
judges:
  - name: selection_f1
    module: judges.selection
    function: judge_selection
  - name: content_assertions
    check: |
      ...
thresholds:
  selection_f1: { min_mean: 0.9 }
  content_assertions: { min_pass_rate: 1.0 }
```

`runner`, `models`, `permissions`, `mlflow`, `inputs`, `hooks` — all omittable. They default to
empty and the branches that read them never fire for `check`/`module` judges. (The spike ran a
config with none of them.)

## The one practical requirement: `skill`

Two reasons `skill` matters:

1. **Discovery** (`discover_configs` L604-659): scans `eval/*/eval.yaml`, `eval/*.yaml`, root
   `eval.yaml`, and **skips any file without a `skill` field** (L625). If we ever run via the
   harness's discovery, no `skill` = invisible.
2. **Runs path** (`_get_runs_dir`): the runs dir is suffixed by `skill`, so it's the
   `code-to-docs` segment in `eval/runs/code-to-docs/<run_id>/`.

`skill` must be a single path segment — no `/`, `\`, control chars (`_is_valid_eval_name` L595).

## Path resolution

`dataset.path` resolves relative to the **eval.yaml's own directory** (`config_dir`,
`resolve_path` L357-367). So `dataset: { path: dataset/cases }` in `eval/eval.yaml` →
`eval/dataset/cases/`. Absolute paths allowed for dataset only (`allow_absolute=True` L451).
`outputs[].path` is rejected if `.` (would mean project root — L475, `reject_root=True`).

## Output config (L96-116)

Two kinds: `path:` (file artifacts dir — what we use) or `tool:` (capture tool calls from event
stream — not us). Each has a natural-language `schema:` string (read by LLM-driven steps in the
full harness; harmless/ignored for our deterministic scoring).

## Judge config quirks (L261-296, parse L506-538)

- yaml `if:` → `JudgeConfig.condition` (L525). Don't write `condition:` in yaml.
- `samples` parsed as int (L536), default 1. Only honored for `llm` judges (see judges.md).
- `arguments:` must be a mapping; passed as `**kwargs` to module/builtin judges, Jinja vars to
  LLM judges.
