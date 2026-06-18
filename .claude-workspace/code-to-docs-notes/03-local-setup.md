# Local Dev Setup

The repo ships only a Dockerfile — there is no `requirements.txt`, no `pyproject.toml`. You can run the **test suite** without Docker easily. Running the **action live** requires either Docker or a real GitHub Actions environment.

## Quick path: run the test suite without Docker

Verified working as of 2026-05-16: **215 tests pass in ~2 seconds** with no env vars set.

### Linux / macOS / WSL

```bash
# Clone (use your fork once you start contributing)
git clone https://github.com/redhat-community-ai-tools/code-to-docs.git
cd code-to-docs

# Optional but recommended: a venv
python3 -m venv .venv
source .venv/bin/activate

# Deps. There's no requirements file — the Dockerfile only installs these:
pip install -U openai mcp mcp-atlassian pytest

# Run the suite
python -m pytest tests/ -v
```

Expected: 215 passed.

### Windows PowerShell (native Python)

```powershell
git clone https://github.com/redhat-community-ai-tools/code-to-docs.git
cd code-to-docs

py -m venv .venv
.\.venv\Scripts\Activate.ps1

pip install -U openai mcp mcp-atlassian pytest

py -m pytest tests/ -v
```

`tests/conftest.py` adds `src/` to `sys.path` and the `clean_env` autouse fixture strips sensitive env vars per test, so nothing on your host environment leaks into the run.

### Running a single test

```bash
pytest tests/test_discovery.py -v
pytest tests/test_discovery.py::test_find_relevant_files_optimized -v
pytest -k "checkbox" -v   # any test name matching "checkbox"
```

## Linting / formatting

There is no ruff/black/flake8 config in the repo and no `pre-commit` hook. The maintainer's style (from skimming PRs): plain PEP 8, 4-space indent, double quotes preferred, type hints are sparse. Don't introduce formatters in your first PR unless explicitly asked.

## Running the action against a real PR

This is the harder path. Two routes.

### Route A — Run the Docker image directly

Needs Docker locally and a model endpoint.

```bash
docker build -t code-to-docs:dev .

docker run --rm \
  -e MODEL_API_BASE="https://generativelanguage.googleapis.com/v1beta/openai/" \
  -e MODEL_API_KEY="$GEMINI_API_KEY" \
  -e MODEL_NAME="gemini-2.0-flash" \
  -e DOCS_REPO_URL="https://github.com/<you>/<your-test-docs-repo>" \
  -e GH_TOKEN="$GITHUB_PAT" \
  -e PR_NUMBER="1" \
  -e PR_BASE="origin/main" \
  -e PR_HEAD_SHA="HEAD" \
  -e COMMENT_BODY="[review-docs]" \
  -e DRY_RUN="true" \
  -v "$PWD:/github/workspace" \
  -w /github/workspace \
  code-to-docs:dev
```

`DRY_RUN=true` skips the actual PR creation but exercises everything up to that point.

### Route B — Use it on a real GitHub repo via the Action

This is what the README documents. Steps:

1. Create a tiny test repo with a docs subfolder.
2. Add a `MODEL_API_KEY` + `MODEL_NAME` + `DOCS_REPO_URL` + `GH_PAT` secret in **Settings → Secrets → Actions**.
3. Drop the workflow file from the README into `.github/workflows/docs-assistant.yml`.
4. Open a trivial PR, comment `[review-docs]`, watch the Action tab.

Cheapest model endpoint for experiments: **Gemini's OpenAI-compatible shim** (free tier exists). Set:

```
MODEL_API_BASE = https://generativelanguage.googleapis.com/v1beta/openai/
MODEL_NAME     = gemini-2.0-flash
MODEL_API_KEY  = <gemini key from aistudio.google.com>
```

`gemini-2.0-flash` has a large context and free quota that's plenty for a small docs repo. The action will hit it the same way it would hit vLLM-on-OpenShift or OpenAI proper.

## What's mocked vs. what's not in the test suite

The test suite mocks:
- `openai.OpenAI` client → fake responses for discovery/generation
- `subprocess.run` for git/gh CLI calls
- File I/O via `tmp_path`

The suite does **not** cover:
- `jira_integration.py` beyond `parse_feature_command` — the MCP stdio client and `gws` CLI are not mocked anywhere
- End-to-end `suggest_docs.main()` orchestration with all branches (no test exercises the full review→update→PR loop)

Both are flagged as contribution candidates in `02-contributions.md` (A1 and B3).

## Working with the Cowork-mounted folder

`C:\Users\ugale\code-to-docs\` (the folder we mounted at the start of this session) is **read-write but git operations don't work inside it** — the FUSE layer rejects some `.git/` filesystem ops.

So:
- Use the mount for **reading** the code and these notes from any tool.
- For **git work** (commits, push, branches), re-clone fresh from your fork into a normal Windows folder, e.g. `C:\Users\ugale\dev\code-to-docs\`, and operate there.
- These notes can travel — copy `notes/` to wherever you do real work.

## Gotchas observed

- The Dockerfile is **the source of truth for runtime deps.** If `pip install` fails locally, check `Dockerfile` for the canonical version pins (currently unpinned — `pip install -U openai mcp mcp-atlassian`).
- `mcp-atlassian` pulls in heavy transitives (FastAPI, etc.) — install time ~30 s, install size ~150 MB.
- `gws` (Google Workspace CLI) is Node-based and only needed for `[review-feature]` with Google Docs. Skip it unless you're working on `jira_integration.py`.
- Python 3.10+ works (image uses 3.12, but 3.10/3.11 are fine — no PEP 695 / 3.12-only features observed).
