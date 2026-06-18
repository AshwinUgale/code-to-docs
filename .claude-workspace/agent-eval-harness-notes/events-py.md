# events-py.md — `agent_eval/events.py` (informational, minimal relevance)

Source: `agent_eval/events.py`. Imports: only `json` + `pathlib` (no internal cross-imports).

## What it does

Parses a Claude Code agent's structured event stream (JSONL of tool calls, reasoning text,
results) and exposes `extract_conversation_text(events)` — used by `score.py` (L200, L355) to
build `record["conversation"]` so a judge can inspect the agent's transcript.

## Why it barely matters to us

This is built for evaluating **agent transcripts** — when the thing under test is a Claude Code
skill whose behavior you judge by reading its tool calls and reasoning. Our system under test is
two Python functions producing **files** (a selection JSON + generated doc content). We judge the
files, not a transcript.

So:
- We set `traces: { events: false, stdout: false, stderr: false }` in eval.yaml → the
  event/conversation/log branches in `load_case_record` (L179-224) never populate, and
  `record["conversation"]` stays `""`.
- `events.py` is still imported (lazily, L200/L355) because score.py references it, so we **keep
  it in the copy** — but we never exercise its output. It's a dependency of the file we copy, not
  a feature we use.

## Net

Copy it (score.py needs the import to resolve), then ignore it. No design decisions hang on it.
