# Claude session onboarding — READ FIRST

You are working with **Ashwin** (`AshwinUgale` on GitHub) on a contribution to `redhat-community-ai-tools/code-to-docs`. This file gives you the orientation you need before doing anything.

## What this folder is

`.claude-workspace/` is a **gitignored** scratchpad inside the fork. It's listed in `.git/info/exclude` (local-only, never committed). Nothing here ships in the PR. Use it freely for context, notes, design docs, anything Claude-internal.

## Where you are

- **Repo:** `C:\Users\ugale\contrib\code-to-docs-fork\code-to-docs\` — Ashwin's fork of `redhat-community-ai-tools/code-to-docs`
- **Reference repo:** `C:\Users\ugale\contrib\agent-eval-harness\agent-eval-harness\` — `opendatahub-io/agent-eval-harness` (READ-ONLY, never modify)
- **Working branch:** `eval-harness-pr1` (already created, pushed to `origin`, tracking set)
- **Fork's `main`:** caught up with `redhat-community-ai-tools/code-to-docs:main` (latest commit `06f62db` — fullsend init by csoceanu, that's upstream not local cruft)

## The task

Build **PR 1** of a two-PR contribution: a self-contained evaluation harness for code-to-docs at `eval/` in the repo root. Maintainer (`csoceanu`, Carmel Soceanu) has approved the plan in a 4-round comment exchange. PR 2 (CI wiring) is a follow-up after this lands and a baseline exists.

**Full plan and locked decisions: `DECISIONS.md`.**
**Original handoff doc: `PROJECT_CONTEXT.md`.**
**Verbatim maintainer comment thread: `MAINTAINER_THREAD.md`.**

## Hard rules

1. **You cannot run git commands.** The Cowork mount has FUSE filesystem restrictions — `.git/` writes either fail outright or leave orphan lock files. Ashwin runs all git commands from his Windows PowerShell terminal. Your job: file contents (Read, Write, Edit). His job: `git add`, `commit`, `push`, branch ops.

2. **Never commit to `main`.** All work happens on `eval-harness-pr1`. If Ashwin asks you for git commands, your output goes to him; you don't execute.

3. **`agent-eval-harness/` is read-only.** It's the source of truth for the scoring code we're copying. Reading it: yes. Modifying it: no.

4. **Don't introduce changes outside `eval/` in PR 1.** No tweaking the existing `code-to-docs` source, no Dockerfile changes, no test additions outside `eval/`, no README updates. PR 1 is strictly additive at `eval/`. Other improvements are parked (see `02-contributions.md` if curious).

5. **Don't write to `.git/`.** Even from the mount. The FUSE layer rejects unlinks → orphan locks block future config writes.

## What to do depending on what Ashwin asks

| Ashwin says | Read this first | Then |
|---|---|---|
| "let's continue" / "what's next" | `PROGRESS.md` | Pick up the in-progress task |
| "explain X" about code-to-docs | `code-to-docs-notes/0X-…` or `1X-walkthrough-…` | Answer with file:line refs |
| "explain X" about agent-eval-harness | `agent-eval-harness-notes/` (may be empty if we haven't read that file yet) | If empty, read the source, then write the note before answering |
| "explain X" about evals (concepts) | `learnings.md` | If the topic isn't there, add it before answering |
| "let's design Y" | `DECISIONS.md` | Add a new decision entry with options + rejected alternatives |
| "let's code Z" | `DECISIONS.md` for the design, then build | Update `PROGRESS.md` when done |
| "I don't understand the PR plan" | `PROJECT_CONTEXT.md` + `DECISIONS.md` | Walk through |
| "what did the maintainer say about X" | `MAINTAINER_THREAD.md` | Search/grep |

## Ashwin's stated constraints

- **He's learning evals as we go.** Don't assume background. Don't dump jargon without grounding it. `learnings.md` is for his study material — keep it filled.
- **He doesn't fully understand code-to-docs yet.** The walkthroughs in `code-to-docs-notes/` are his reference. When unclear, point at them.
- **He'll run code himself** (REPL exercises, pytest, the eval harness). You write the code, he validates.
- **Style preferences:** technical, direct, no padding. Prose over bullets unless structure is clearly justified. No emojis unless he uses them first.

## What's already in this folder

```
.claude-workspace/
├── CLAUDE_ONBOARDING.md          ← this file
├── PROJECT_CONTEXT.md            ← the handoff (normalized)
├── MAINTAINER_THREAD.md          ← verbatim 4-round comment exchange + approval
├── DECISIONS.md                  ← locked technical decisions + reasoning + rejected alternatives
├── PROGRESS.md                   ← running task log
├── learnings.md                  ← Ashwin's study material (10 topics)
├── code-to-docs-notes/           ← architecture, walkthroughs, contributions list
│   ├── 01-architecture.md
│   ├── 02-contributions.md       (informational; not what we're building)
│   ├── 03-local-setup.md
│   ├── 10-walkthrough-review-docs.md
│   └── 11-walkthrough-caching.md
└── agent-eval-harness-notes/     ← (empty for now; populated as we read the harness)
    └── README.md
```

## First-session checklist

If you're a fresh Claude in a new session:

1. Read this file (you just did).
2. Read `PROJECT_CONTEXT.md` to load the full plan.
3. Skim `DECISIONS.md` — these are settled, do NOT relitigate.
4. Glance at `PROGRESS.md` to see where we are.
5. Ask Ashwin what he wants to work on. Don't start independently.
