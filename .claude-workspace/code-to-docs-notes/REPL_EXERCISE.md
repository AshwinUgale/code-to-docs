# REPL exercise — see code-to-docs's discovery and generation live

**Goal:** Build muscle memory for what `find_relevant_files_optimized` / `ask_ai_for_relevant_files` / `ask_ai_for_updated_content` actually do. ~30 minutes. No code committed — this is exploratory.

Why this matters for PR 1: you'll be wrapping these functions in `execute.py`. Doing them by hand once gives you ground truth for the LLM's failure modes — which makes the assertions in your eval concrete, and the runner design obvious.

**What you'll see:**
- The discovery LLM picking files (correctly, sometimes incorrectly)
- The generation LLM returning `NO_UPDATE_NEEDED` (sometimes when it shouldn't)
- The generation LLM wrapping output in ``` (sometimes — the failure mode the global judge catches)
- Truncation log messages when the diff is bigger than budget

**Important:** Do this from your Windows terminal in your fork. The Cowork mount can't run Python interactively due to FUSE permissions. All commands below are PowerShell.

---

## Part 1 — Setup (~5 min)

### 1.1 Create a venv inside the repo

```powershell
cd C:\Users\ugale\contrib\code-to-docs-fork\code-to-docs

py -m venv .venv
.\.venv\Scripts\Activate.ps1

# Confirm
python --version    # 3.10+
where.exe python    # should point to .venv\Scripts\python.exe
```

The venv folder `.venv` is already in the upstream `.gitignore` (verify with `Get-Content .gitignore`). If not, add it to `.git/info/exclude` the same way we did `.claude-workspace/`.

### 1.2 Install only what's needed

For discovery + generation (without the index path's git/MCP side effects), all you need is `openai`. Skip `mcp` and `mcp-atlassian` — they're for `[review-feature]`.

```powershell
pip install openai
```

### 1.3 Get a Gemini API key (free tier is fine)

1. Go to https://aistudio.google.com/apikey
2. Create an API key, copy it.
3. Set env vars in the same PowerShell session:

```powershell
$env:MODEL_API_BASE = "https://generativelanguage.googleapis.com/v1beta/openai/"
$env:MODEL_API_KEY  = "<paste your key here>"
$env:MODEL_NAME     = "gemini-2.0-flash"
$env:MAX_CONTEXT_CHARS = "32000"
```

These are read at function call time by `config.get_client()` and `config.get_model_name()`. The `MAX_CONTEXT_CHARS=32000` is intentionally tight — you'll see truncation kick in on bigger diffs, which is informative.

### 1.4 Confirm the import path

The code-to-docs source lives at `src/`. Add it to `PYTHONPATH` for this session, OR run Python from inside `src/`. The cleaner option:

```powershell
$env:PYTHONPATH = "C:\Users\ugale\contrib\code-to-docs-fork\code-to-docs\src"
```

---

## Part 2 — Build a tiny fake docs tree (~5 min)

Create a scratch directory **outside** the repo so you don't accidentally commit it:

```powershell
mkdir C:\Users\ugale\contrib\repl-scratch
cd C:\Users\ugale\contrib\repl-scratch

# Make a docs tree
mkdir docs
mkdir docs\cli
mkdir docs\config
```

Now create four small Markdown files. Open Notepad or VS Code and create:

**`docs\cli\reference.md`:**
```markdown
# CLI Reference

The `mytool` command-line interface.

## Synopsis

    mytool [options] <command>

## Options

- `--help` — Show help
- `--version` — Show version
- `--quiet` — Suppress output
```

**`docs\config\settings.md`:**
```markdown
# Configuration

Settings live in `~/.mytool/config.yaml`.

## Available keys

- `cache_dir` — where mytool stores its cache (default: `~/.cache/mytool`)
- `timeout` — request timeout in seconds (default: 30)
```

**`docs\getting-started.md`:**
```markdown
# Getting Started

Install with `pip install mytool`. Run `mytool --version` to confirm.

For configuration, see [the configuration docs](config/settings.md).
For the CLI reference, see [the CLI docs](cli/reference.md).
```

**`docs\index.md`:**
```markdown
# Documentation

- [Getting Started](getting-started.md)
- [CLI Reference](cli/reference.md)
- [Configuration](config/settings.md)
```

### Sanity check

```powershell
cd C:\Users\ugale\contrib\repl-scratch
Get-ChildItem -Recurse docs
# Expected: 4 .md files in the structure above
```

---

## Part 3 — Discovery exercise (~10 min)

We're going to call the **full-scan** path (`ask_ai_for_relevant_files`) directly — it has no git side effects. The index path (`find_relevant_files_optimized`) requires the git/index machinery; we'll cover that when we design `execute.py`.

### 3.1 Hand-craft a diff that should affect CLI docs

Create `scratch\diff_cli.txt` (this is just text — you'll feed it as a string, not as a real git diff applied to a repo):

```
diff --git a/src/cli.py b/src/cli.py
index abc1234..def5678 100644
--- a/src/cli.py
+++ b/src/cli.py
@@ -42,6 +42,7 @@ def build_parser():
     p.add_argument("--help", action="store_true")
     p.add_argument("--version", action="store_true")
     p.add_argument("--quiet", action="store_true")
+    p.add_argument("--verbose", action="store_true", help="Print debug output")
     return p
```

This adds a `--verbose` flag. The docs we'd expect discovery to pick: `cli/reference.md`. Maybe also `getting-started.md` (mentions running commands).

### 3.2 Run the discovery REPL

From `C:\Users\ugale\contrib\repl-scratch`:

```powershell
# Make sure docs/ is the CWD — the function rglob's "."
cd C:\Users\ugale\contrib\repl-scratch\docs

python
```

Inside the Python REPL:

```python
# Discovery — full-scan path
from discovery import ask_ai_for_relevant_files, get_file_content_or_summaries

# Walk the docs tree, build previews
previews = get_file_content_or_summaries()
print(f"Found {len(previews)} files")
for p in previews:
    print(f"  - {p[0]}")

# The diff (paste as a triple-quoted string)
diff = """diff --git a/src/cli.py b/src/cli.py
index abc1234..def5678 100644
--- a/src/cli.py
+++ b/src/cli.py
@@ -42,6 +42,7 @@ def build_parser():
     p.add_argument("--help", action="store_true")
     p.add_argument("--version", action="store_true")
     p.add_argument("--quiet", action="store_true")
+    p.add_argument("--verbose", action="store_true", help="Print debug output")
     return p
"""

# Ask the LLM which files are relevant
selected = ask_ai_for_relevant_files(diff, previews)
print("Selected:", selected)
```

**What to observe:**
- Which file(s) did discovery pick? Expected: `cli/reference.md` definitely; possibly also `getting-started.md`.
- If it picked nothing (`NONE`), the prompt's conservatism won out — note this. Re-run a few times to see variance.
- If it picked `config/settings.md` or `index.md`, that's an over-selection — the prompt explicitly warns against this. Note it.
- Look at the print output during the call. You'll see "Batch 1/1: Found N relevant files" or "Batch 1/1: No relevant files found."

### 3.3 Try the negative case

Replace the diff with one that shouldn't affect any docs:

```python
diff_negative = """diff --git a/src/internal_helpers.py b/src/internal_helpers.py
index abc1234..def5678 100644
--- a/src/internal_helpers.py
+++ b/src/internal_helpers.py
@@ -10,7 +10,7 @@ def _normalize_path(p):
     if p is None:
         return None
-    return p.strip().lower()
+    return p.strip().casefold()
"""

selected = ask_ai_for_relevant_files(diff_negative, previews)
print("Selected:", selected)
```

This is an internal refactor — `strip().lower()` → `strip().casefold()`. No user-facing change. **Expected: `selected` is empty.** If discovery picked anything, that's exactly the over-selection failure mode our negative cases will measure.

Run this 3-5 times. Watch the variance. This is the non-determinism csoceanu flagged in Round 2.

---

## Part 4 — Generation exercise (~10 min)

Same Python session (or restart and re-set env vars + PYTHONPATH).

### 4.1 Call generation on the file discovery picked

```python
from generation import ask_ai_for_updated_content
from pathlib import Path

# Read the current file
current = Path("cli/reference.md").read_text(encoding="utf-8")

# Same diff as before
diff = """diff --git a/src/cli.py b/src/cli.py
index abc1234..def5678 100644
--- a/src/cli.py
+++ b/src/cli.py
@@ -42,6 +42,7 @@ def build_parser():
     p.add_argument("--help", action="store_true")
     p.add_argument("--version", action="store_true")
     p.add_argument("--quiet", action="store_true")
+    p.add_argument("--verbose", action="store_true", help="Print debug output")
     return p
"""

updated = ask_ai_for_updated_content(diff, "cli/reference.md", current)
print("=" * 60)
print(repr(updated[:200]))  # repr() shows escape characters and BOMs
print("=" * 60)
print(updated)
```

**What to observe:**
- Does `updated` start with ` ``` `? That's the fence-wrapping failure mode. The global judge "first non-blank line is not ```" catches this. Note how often it happens.
- Does the output mention `--verbose`? (Should — it's the literal change in the diff. This is what `must_mention: ["--verbose"]` would check.)
- Does the output mention `--debug`, `--trace`, or anything else not in the diff? That's hallucination. `must_not_mention: ["--debug"]` would catch this.
- Did it add a new section ("## Verbose mode" or similar)? That's what `must_not_add_sections: true` catches.
- How many extra lines vs the original? `max_new_lines: 20` would bound this.

### 4.2 Force NO_UPDATE_NEEDED

Run generation on a file that has no relationship to the diff:

```python
config_current = Path("config/settings.md").read_text(encoding="utf-8")
updated = ask_ai_for_updated_content(diff, "config/settings.md", config_current)
print("Result:", repr(updated.strip()))
```

**Expected:** `'NO_UPDATE_NEEDED'`. If the LLM rewrote the config file, that's a regression of the conservatism mechanism — exactly what our negative content cases will catch.

### 4.3 Watch for fence wrapping

This is the most common LLM failure with this prompt. Run generation 5-10 times on the same input and count how often `updated.startswith("```")` is true. You'll see it happen at least once or twice.

```python
fence_failures = 0
for i in range(5):
    out = ask_ai_for_updated_content(diff, "cli/reference.md", current)
    if out.strip().startswith("```"):
        fence_failures += 1
        print(f"Run {i+1}: FENCE WRAPPED ({len(out)} chars)")
    else:
        print(f"Run {i+1}: clean")
print(f"Total fence failures: {fence_failures}/5")
```

This is **why** the global no-fence judge exists. It's not theoretical.

---

## Part 5 — What to take away

Before you close the REPL, take 5 minutes to write down (in this file or wherever):

1. **How many of 5 runs did discovery pick the right files for the positive diff?** (Tells you SUT variance for selection.)
2. **How many of 5 runs did the negative diff get `[]` from discovery?** (Tells you how often over-selection happens.)
3. **How many of 5 runs did generation return `NO_UPDATE_NEEDED` for the unrelated config file?** (Tells you how often conservatism succeeds on the unrelated case.)
4. **How many of 5 runs did generation fence-wrap the output?** (Tells you how common that specific failure is.)
5. **Did you observe any unexpected behavior?** (Hallucination, missed updates, format mixing, etc.)

These numbers are the back-of-envelope baselines for what the eval is going to measure rigorously. They also tell you whether N=3 (for fast PR feedback) and N=10 (for daily) are sensible — if a failure mode happens 1 in 5 runs, N=3 will miss it 50% of the time.

---

## Cleanup

```powershell
deactivate                              # exit venv
cd C:\Users\ugale\contrib\repl-scratch
# Leave the docs tree around — useful as reference / to re-run later.
# Delete the venv if you want: Remove-Item -Recurse -Force C:\Users\ugale\contrib\code-to-docs-fork\code-to-docs\.venv
```

---

## What this primes you for

- **Designing `execute.py`** — you'll have hands-on intuition for what the runner is wrapping. The function signatures, the parameter passing, the env var setup — none of it will be abstract anymore.
- **Sizing the judge assertions** — you'll have rough variance numbers for the failure modes, which inform N.
- **Reading the agent-eval-harness `score.py`** — you'll understand what the judges are scoring because you've seen the SUT outputs they score.

Next: agent-eval-harness reading + `execute.py` design in `DECISIONS.md`.
