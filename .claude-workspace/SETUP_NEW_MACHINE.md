# Setup on a new machine

Self-contained recipe for picking up the code-to-docs eval-harness contribution on a fresh laptop. Assumes Windows + PowerShell. For macOS/Linux, swap `Set-Content -Encoding ASCII` for `echo ... >>` and PowerShell-specific syntax accordingly.

**Total time: ~10-15 minutes.**

---

## Prerequisites

- **Git** installed and configured with your name/email
- **Python 3.10+** (3.11 or 3.12 preferred)
- **Terminal** (PowerShell on Windows; bash/zsh elsewhere)
- **GitHub access** to your fork at `AshwinUgale/code-to-docs`
- **A Claude product** (Cowork, Claude Code, etc.) where you'll continue the work

---

## Step 1 — Clone the fork

Choose a working folder. Example uses `C:\Users\<you>\contrib\`. Replace `<you>` with your Windows username.

```powershell
mkdir C:\Users\<you>\contrib -Force
cd C:\Users\<you>\contrib
git clone https://github.com/AshwinUgale/code-to-docs.git code-to-docs-fork
cd code-to-docs-fork
```

Verify the remote:

```powershell
git remote -v
```

Should show `origin` pointing at `AshwinUgale/code-to-docs`. If you also want an `upstream` remote pointing at the canonical Red Hat repo (useful for syncing your fork's main later):

```powershell
git remote add upstream https://github.com/redhat-community-ai-tools/code-to-docs.git
git fetch upstream
```

Not strictly needed for the PR.

---

## Step 2 — Check out the PR working branch

```powershell
git fetch origin eval-harness-pr1
git checkout eval-harness-pr1
git status
```

Expected: `On branch eval-harness-pr1`, `Your branch is up to date with 'origin/eval-harness-pr1'`, `nothing to commit, working tree clean`.

---

## Step 3 — Pull `.claude-workspace/` from the `claude-context` branch

```powershell
git fetch origin claude-context
git checkout origin/claude-context -- .claude-workspace/
git restore --staged .claude-workspace/
```

The two-step (`checkout` then `restore --staged`) is mandatory. `checkout <branch> -- <path>` both writes the files AND stages them; we want them only on disk, not in the index for `eval-harness-pr1`.

Verify:

```powershell
Get-ChildItem .claude-workspace | Select-Object Name
git status
```

`Get-ChildItem` should list 6 markdown files + 2 subfolders (`agent-eval-harness-notes`, `code-to-docs-notes`).
`git status` should still say `nothing to commit, working tree clean`.

---

## Step 4 — Re-add the gitignore exclude

`.claude-workspace/` must never end up in a commit on `eval-harness-pr1`. The `.git/info/exclude` file is local-only and doesn't transfer in a clone, so you have to re-add it on each new machine.

```powershell
Set-Content -Path .git\info\exclude -Value ".claude-workspace/" -Encoding ASCII
```

Verify:

```powershell
Get-Content .git\info\exclude
git status
```

`Get-Content` should print `.claude-workspace/` and only that.
`git status` still clean.

---

## Step 5 — Clone `agent-eval-harness` as a read-only reference

This is the upstream repo whose `score.py + config.py + events.py` we're copying into the PR. We read from it but never modify it.

```powershell
cd C:\Users\<you>\contrib
git clone https://github.com/opendatahub-io/agent-eval-harness.git
```

Verify:

```powershell
Get-ChildItem C:\Users\<you>\contrib\agent-eval-harness\skills\eval-run\scripts\score.py
```

Should print a file listing for `score.py`.

---

## Step 6 — Bootstrap a Claude session

Open Cowork (or whichever Claude product you're using). Mount **both folders** as Cowork directories:

- `C:\Users\<you>\contrib\code-to-docs-fork`
- `C:\Users\<you>\contrib\agent-eval-harness`

Or as a single parent mount:

- `C:\Users\<you>\contrib`

Then paste this message as the first one in the new conversation:

```
I'm continuing work on the code-to-docs eval harness contribution. Read these files in order before doing anything:

1. .claude-workspace/CLAUDE_ONBOARDING.md — orientation + hard rules
2. .claude-workspace/PROGRESS.md — where we are right now
3. .claude-workspace/DECISIONS.md — locked decisions (skim, don't relitigate)

The fork is mounted at C:\Users\<you>\contrib\code-to-docs-fork\code-to-docs.
The branch is eval-harness-pr1.
The agent-eval-harness reference clone is at C:\Users\<you>\contrib\agent-eval-harness.

After reading those three files, tell me what state we're in and propose the next action.
```

Replace `<you>` with your username. New Claude reads, summarizes, proposes. You're back in flow in ~60 seconds.

---

## Step 7 (optional) — Python venv for REPL or local testing

If you want to re-run the REPL exercise, run pytest, or test code locally:

```powershell
cd C:\Users\<you>\contrib\code-to-docs-fork\code-to-docs
py -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install openai pytest
```

Confirm:

```powershell
pip show openai
```

For LLM API access (REPL or future `execute.py` runs), set env vars in each PowerShell session:

```powershell
$env:PYTHONPATH = "C:\Users\<you>\contrib\code-to-docs-fork\code-to-docs\src"
$env:MAX_CONTEXT_CHARS = "32000"

# For OpenAI:
$env:MODEL_API_BASE = "https://api.openai.com/v1"
$env:MODEL_API_KEY  = "<your OpenAI key>"
$env:MODEL_NAME     = "gpt-4o-mini"

# OR for Gemini (if free tier is available on your account):
# $env:MODEL_API_BASE = "https://generativelanguage.googleapis.com/v1beta/openai/"
# $env:MODEL_API_KEY  = "<your Gemini key>"
# $env:MODEL_NAME     = "gemini-2.0-flash"
```

The venv folder `.venv` is in code-to-docs's `.gitignore` upstream → no PR-diff risk.

---

## How to push workspace updates back to `claude-context`

After Claude adds new files or decisions to `.claude-workspace/`, sync them to the orphan branch using a temporary worktree (avoids destructive branch switches):

```powershell
cd C:\Users\<you>\contrib\code-to-docs-fork\code-to-docs

git worktree add ..\claude-context-tmp claude-context
robocopy .claude-workspace ..\claude-context-tmp\.claude-workspace /MIR
cd ..\claude-context-tmp
git add -f .claude-workspace/
git commit -m "claude-context: <what changed today>"
git push
cd ..\code-to-docs
git worktree remove ..\claude-context-tmp
```

Rule of thumb: push after any session where you added decisions, walkthroughs, or completed a phase (REPL, agent-eval-harness reading, execute.py design, etc.). Don't push for every typo fix.

---

## Common pitfalls

- **Pasting prose into PowerShell.** When copying instructions, only paste lines between code-fence markers. PowerShell will try to execute prose lines as commands and error.
- **PowerShell `>>` writes UTF-16 with BOM.** Use `Set-Content -Encoding ASCII` (or `-Encoding UTF8NoBOM`) for any file git or unix-style tools will read, including `.git/info/exclude`.
- **Forgetting `git restore --staged` after the path-restricted checkout.** Leaves the workspace files staged on `eval-harness-pr1`. If you forget, run `git restore --staged .claude-workspace/` to fix.
- **Running `git clone` from inside an existing clone.** Creates a nested duplicate. Always `cd` to the parent folder before cloning.
- **Mount path mismatch.** Cowork mount path in the bootstrap message must match the actual host path. Update the message for each new machine.

---

## Quick sanity checklist before working

After Steps 1-6, confirm:

```powershell
cd C:\Users\<you>\contrib\code-to-docs-fork\code-to-docs

git status
# Should say: On branch eval-harness-pr1, working tree clean

git branch -a
# Should list: eval-harness-pr1 (current), claude-context, main, plus origin/* counterparts

Get-Content .git\info\exclude
# Should print: .claude-workspace/

Get-ChildItem .claude-workspace
# Should show 6 .md files + 2 folders

Get-ChildItem C:\Users\<you>\contrib\agent-eval-harness\skills\eval-run\scripts\score.py
# Should show score.py file info
```

All five clean → you're ready. Bootstrap Claude with the message in Step 6 and continue.
