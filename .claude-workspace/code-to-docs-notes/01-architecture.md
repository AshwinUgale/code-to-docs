# Architecture Map — code-to-docs

A GitHub Action that turns PR comments (`[review-docs]`, `[update-docs]`, `[review-feature] PROJ-123`) into AI-generated docs PRs against a separate (or same) docs repo, using any OpenAI-compatible LLM endpoint.

## Top-level surface

```
.github action trigger  (issue_comment on a PR)
        │
        ▼
action.yml  ──►  Docker image (Dockerfile, UBI10 + Python 3.12)
        │
        ▼
entrypoint.sh  ── validates env, sets up git, writes Google SA key to /tmp
        │
        ▼
python /app/suggest_docs.py    ◄── orchestrator
```

`entrypoint.sh` does git/credential plumbing then execs `suggest_docs.py`. Everything below is Python.

## Module responsibilities (src/)

| Module | Role |
|---|---|
| `suggest_docs.py` | Orchestrator. Parses the comment, picks mode, runs discovery → generation → comment/PR. ~400 lines. |
| `config.py` | Env-var access + OpenAI client factory + context-budget truncation (per-file & per-diff). |
| `github_ops.py` | `git diff`, `git merge-base`, docs-repo clone or subfolder setup, `gh pr` creation. |
| `discovery.py` | Two-stage file selection: semantic index → AI shortlist. Batches previews by char budget. |
| `generation.py` | Per-file content rewrite via LLM, format-aware (md/adoc/rst), parallel with ThreadPoolExecutor. |
| `comments.py` | Builds the PR review comment with checkbox list, parses previous review for `[update-docs]`. |
| `doc_index.py` | Builds `.doc-index/` (folder index .md + summaries manifest), commits to docs `main`. |
| `jira_integration.py` | mcp-atlassian + gws CLI to fetch Jira ticket, Confluence pages, Google Docs. |
| `security_utils.py` | Output sanitization (token redaction), `validate_file_path()`, safe subprocess wrapper. |
| `utils.py` | `retry_with_backoff()` decorator. |

## Request flow per command

### `[review-docs]`
1. `get_diff()` — `git diff $(git merge-base origin/main HEAD)..HEAD`, fallback to `PR_BASE...HEAD`.
2. Context-budget guard: if `len(diff) > 0.9 * MAX_CONTEXT_CHARS`, abort with a useful message.
3. `setup_docs_environment()` — either `cd $DOCS_SUBFOLDER` (same-repo) or clone `DOCS_REPO_URL`.
4. Discovery:
   - If `.doc-index/manifest.json` exists → `find_relevant_files_optimized()` (1 LLM call per area, then file shortlist).
   - Else → full scan: `get_file_content_or_summaries()` + `ask_ai_for_relevant_files()`.
5. Generation: `generate_updates_parallel()` — up to 5 workers, each calling `ask_ai_for_updated_content()`.
6. `post_review_comment()` — markdown checklist into a `/tmp/review_comment.md` file, then `gh pr comment --body-file`.

### `[update-docs]`
- Same as above, but `parse_previous_review()` scans the PR for the bot's prior comment, harvests `- [x]` checkboxes, and skips discovery entirely (uses the human-curated file list).
- Inline instructions parsed from the comment body: first line is global; `file.ext: instructions` lines are per-file overrides.
- After generation, `push_and_open_pr()` either commits onto a `doc-update-from-pr-<N>` branch (subfolder mode) or pushes to the docs repo with a `gh pr create`.

### `[review-feature] PROJ-123`
- Wraps `[review-docs]` and adds a "Spec vs Code Analysis" section.
- `fetch_jira_context_sync()` spawns mcp-atlassian over stdio, pulls the ticket, regex-extracts all Google Docs / Confluence URLs.
- Confluence pages fetched via the same MCP server; Google Docs via the `gws` Node CLI (`npm install -g @googleworkspace/cli`) using the service account key.
- `analyze_feature_coverage()` sends (diff + spec text + ticket) to the LLM with a fixed prompt asking for {covered, missing, unplanned} buckets.

## Caching layer (`.doc-index/`)

Persisted in the **docs repo's `main` branch**, shared across all PRs:

```
.doc-index/
├── manifest.json            # { "folders": {...}, "schema_version": 1 }
├── summaries-manifest.json  # { "<sha256>": "...summary..." }
└── <folder>/.index.md       # AI-generated semantic summary of that folder
```

Workflow:
1. On the first run, `build_all_indexes(force=True)` walks the docs tree, summarizes each folder's files, writes `.index.md`.
2. Index files are committed back to `main` with a token-protected fetch/push (`commit_indexes_to_repo`).
3. Subsequent PRs read the indexes → AI is asked which *folders* are relevant → only those folders' files are scanned.
4. File-level summaries are cached by SHA256 of file content in `summaries-manifest.json` and reused indefinitely.

Reported speedup: ~20 min → ~4 min on large doc trees.

## External dependencies

| Tool | Why | How invoked |
|---|---|---|
| `git` | diffs, clone, merge-base, commit, push | subprocess wrapped in `run_command_safe` |
| `gh` CLI | PR comments, PR creation, fetching PR data | subprocess; needs `GH_TOKEN` env |
| `openai` Python SDK | Any OpenAI-compatible API (vLLM, Gemini OpenAI shim, Ollama, OpenAI) | `OpenAI(base_url=..., api_key=...)` |
| `mcp` + `mcp-atlassian` | Jira/Confluence reads | stdio MCP server spawned per run, async via `asyncio.run` |
| `gws` CLI (Node) | Google Docs export | subprocess; needs `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` |

No DB. No persistent service. All state lives in env vars, `/tmp/*`, and committed `.doc-index/`.

## Notable design choices

- **Single Docker image**, runs entirely inside the GitHub Actions runner per comment. No webhooks/lambdas.
- **OpenAI-compatible-only.** PR #6 removed direct Gemini SDK usage. Vendor-portable.
- **Budget-aware truncation.** `config.truncate_diff()` splits on `diff --git ` boundaries and includes whole-file chunks greedily, so the LLM always sees complete file diffs (or none).
- **Comment-driven UX.** No web UI. The user reviews via GitHub's native checkbox markdown — the bot re-parses that on the next `[update-docs]`.
- **Same-repo and split-repo modes.** `DOCS_SUBFOLDER=""` → clone separate docs repo. `DOCS_SUBFOLDER=docs` → operate on the same repo, branch off `main`.
- **Parallelism**: ThreadPoolExecutor for index building and per-file generation (max 5 workers default). Generation is I/O-bound (LLM calls).

## Fragility / sharp edges noted in review

- **Circular import** between `generation.py` and `comments.py`; `_resolve_file_instructions` is imported lazily inside a function.
- **Hardcoded `/tmp/review_comment.md`** in `comments.py:431` — not portable, races if multiple actions run on the same runner.
- **Checkbox parsing regex** assumes strict `- [x] [path](url): summary` or `- [x] **path**: summary` — minor formatting drift breaks it.
- **`gws` output detection** has multiple fallback file locations, fragile to CLI behavior changes.
- **`git config` puts the token in cleartext** during `setup_git_credentials()` — visible to anything reading the config inside the container.
- **`MAX_FILES_PER_BATCH = 10`** in discovery.py is hardcoded.
- **No tests for `jira_integration.py`** (async + external MCP + gws — needs mocking).
- **`doc_index.commit_indexes_to_repo`** has complex branch-switching with tempfile.mkdtemp; on exception path could leave orphaned temp dirs.

## Test coverage

`tests/` has 11 files: discovery, generation, comments, doc_index, github_ops, security_utils, utils, truncation, URL/path handling. `conftest.py` provides a `clean_env` fixture that strips sensitive env vars. No tests for `jira_integration.py`, no end-to-end test of `suggest_docs.py` orchestration.

## File map quick-reference

```
src/
  suggest_docs.py        # entrypoint — argparse, mode detection, orchestration
  config.py              # env, OpenAI client, context budget, truncation
  github_ops.py          # git/gh CLI wrapper, diff, env setup, push & open PR
  discovery.py           # find_relevant_files_optimized, ask_ai_for_relevant_files
  generation.py          # ask_ai_for_updated_content, parallel updates, format prompts
  comments.py            # review comment building, checkbox parsing, posting
  doc_index.py           # folder indexes, summaries manifest, push to main
  jira_integration.py    # MCP + gws CLI, feature coverage analysis
  security_utils.py      # sanitize_output, validate_file_path, run_command_safe
  utils.py               # retry_with_backoff
tests/                   # pytest suite (11 files)
demo/                    # static HTML demo (index.html, review-feature.html)
action.yml               # GitHub Action manifest (inputs/outputs/runs)
Dockerfile               # UBI10 Python 3.12 + git/gh/gws/uv/openai/mcp/mcp-atlassian
entrypoint.sh            # validates env, exports vars, exec suggest_docs.py
```
