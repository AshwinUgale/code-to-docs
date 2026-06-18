# Walkthrough 1 — `[review-docs]` end-to-end

This walks through what happens when a developer types `[review-docs]` on a PR. Read it with `src/suggest_docs.py` and the modules it touches open. Annotations point to real file:line refs.

Why start here: this is the spine of the project. `[update-docs]` and `[review-feature]` are variations that swap or wrap pieces of this flow — once you've got this in your head, the other two are 30 minutes to understand.

## The trigger surface

Nothing in this codebase listens to webhooks. The whole loop is:

```
GitHub user types comment "[review-docs]"
   ↓
GitHub Actions workflow fires on issue_comment.created
   ↓
docs-assistant job spawns an Ubuntu runner
   ↓
Job step "Documentation Assistant" pulls the action's Docker image
   ↓
Docker container runs entrypoint.sh which execs `python /app/suggest_docs.py`
   ↓
Container exits. PR gets a new bot comment with checkboxes. Done.
```

The workflow file the user copies into their repo (`.github/workflows/docs-assistant.yml` from the README) is responsible for two things: parsing `issue_comment` payload and checking out the **PR's** code (not main). The action itself doesn't know anything about webhook payloads — it reads everything from env vars set by the workflow's `with:` block, mapped through `action.yml`.

## Step 1 — Mode detection (suggest_docs.py:70–144)

`main()` reads three env vars set by the workflow:

```python
comment_body = os.environ.get("COMMENT_BODY", "")  # the literal text of the user's comment

feature_mode = "[review-feature]" in comment_body.lower()
review_mode  = "[review-docs]"    in comment_body.lower()
update_mode  = "[update-docs]"    in comment_body.lower()
```

Note `.lower()` — so `[Review-Docs]` works too. The three modes are not mutually exclusive in principle (someone could write `[review-docs] [update-docs]` in the same comment) but in practice they're handled in order: feature_mode is checked first and short-circuits if Jira credentials are missing; then review and update are evaluated. If only `[review-feature]` is present, review_mode is forced on (line 137):

```python
if not review_mode and not update_mode:
    if feature_mode:
        review_mode = True
    else:
        update_mode = True   # legacy fallback when bare command run
```

The "legacy fallback" — running with no command at all defaults to update_mode — is a holdover from before the interactive review workflow existed (PR #4). Worth keeping in mind: if `COMMENT_BODY` is empty (e.g. someone wires the action to something other than a comment trigger), it'll silently try to push docs to the docs repo.

## Step 2 — Context budget gate (suggest_docs.py:53–57, 152–162)

```python
budget = get_max_context_chars()  # default 400_000 chars, ~100K tokens
# ...
diff_ratio = len(diff) / budget
if diff_ratio > 0.9:
    print(f"Error: Diff is too large ...")
    return
```

This 90% gate is **before** any LLM call. The reasoning: the pipeline needs room for prompt templates (~2K), file previews, and current doc content — if the diff alone takes >90% of the budget, there's no room left for the actual job. Bail out early with a useful error rather than failing inside an LLM call with a confusing "context length exceeded" message.

`MAX_CONTEXT_CHARS` is in **characters**, not tokens. The default 400K ≈ 100K tokens at the conventional 4 chars/token. The README's advice ("set to 32000 for an 8K-token model") works because that math holds across most tokenizers.

When the diff is below the gate but still big, `config.truncate_diff()` is called later (per-file in generation, per-batch in discovery) to clip it to whatever budget is left after the prompt template. `truncate_diff` is smart: it splits on `diff --git ` boundaries and includes **whole file-diffs greedily** so the model never sees a half-truncated file. Falls back to character truncation only if the first file-diff alone overflows.

## Step 3 — Get the diff (github_ops.py:21–70)

```python
merge_base_result = run_command_safe(["git", "merge-base", pr_base, "HEAD"], check=False)
if merge_base_result.returncode == 0:
    merge_base = merge_base_result.stdout.strip()
    result = run_command_safe(["git", "diff", f"{merge_base}...HEAD"], check=False)
else:
    # fallback
    result = run_command_safe(["git", "diff", f"{pr_base}...HEAD"], check=False)
```

The trick: `merge_base...HEAD` (three dots) vs `pr_base...HEAD`. The three-dot syntax means "changes on HEAD's side since divergence", which is exactly what a PR diff should be — but it depends on `pr_base` actually being a reachable ancestor. Hence the explicit `git merge-base` call first.

Why the fallback exists: GitHub Actions sometimes does shallow clones. `git merge-base` returns nothing if base and head don't share enough history. The workflow file the README ships uses `fetch-depth: 0` to avoid this, but if a user copies a different workflow, the action still works — it just gets a possibly wider diff.

`run_command_safe` (in `security_utils.py`) is a thin wrapper around `subprocess.run` that lists allowed binaries (`git`, `gh`, `gws`, etc.) and refuses to run anything else — defense against accidentally building shell-injection holes when user-controlled strings end up in command args.

## Step 4 — Set up the docs environment (github_ops.py:122–172)

Two scenarios:

```python
docs_subfolder = os.environ.get("DOCS_SUBFOLDER")
if docs_subfolder:
    # Same-repo mode: docs live in a subfolder of the code repo
    os.chdir(docs_subfolder)
else:
    # Separate-repo mode: clone the docs repo
    run_command_safe(["git", "clone", docs_repo_url, "docs_repo"], check=True)
    os.chdir("docs_repo")
    # Either reuse an existing doc-update-from-pr-<N> branch or create one
```

Tricky bit: after this call, **the process's CWD has changed**. Everything downstream that reads doc files (`Path(".").rglob("*.md")` in discovery, `Path(file_path).read_text` in generation) is implicitly scoped to wherever we just chdir'd. There's no global "docs root" abstraction. If you're debugging path issues, check whether the function in question was called before or after `setup_docs_environment()`.

The branch naming is per-PR: `doc-update-from-pr-{PR_NUMBER}` (see `config.get_branch_name`). This means re-running `[update-docs]` on the same PR updates the same branch in the docs repo, instead of creating a new one each time. The branch is reused via `git ls-remote --heads origin <branch>` check.

## Step 5 — File discovery (suggest_docs.py:233–256)

```python
if use_index:
    relevant_files = find_relevant_files_optimized(diff)
    if relevant_files is None:    # AI explicitly requested a full scan
        use_index = False

if not use_index:
    file_previews = get_file_content_or_summaries()
    relevant_files = ask_ai_for_relevant_files(diff, file_previews)
```

The interesting design choice is that `find_relevant_files_optimized` returning `None` (vs empty list) is a meaningful signal: "the indexes don't have enough info; please do a full scan." `find_relevant_areas_from_indexes` in `doc_index.py` returns None when the model's answer is `FULL_SCAN` — a sentinel string baked into the prompt. So the optimization path is **self-disabling**: if the indexes are missing or misleading, the model is allowed to say "scan everything." Walkthrough 2 will dig into the indexing — for now just know it's a two-stage call: pick folders, then pick files within those folders.

The full-scan path (`get_file_content_or_summaries` + `ask_ai_for_relevant_files`) is what runs the first time you use the action on a repo with no `.doc-index/` yet. It walks the whole docs tree, builds a preview per file (full content if <300 lines, AI-generated summary if longer), batches them by character budget, and sends parallel LLM calls asking which are relevant.

### The discovery prompt (discovery.py:135–160)

```
You are an ULTRA-CONSERVATIVE documentation assistant. Select ONLY files that
DIRECTLY document the EXACT code being changed.
...
6. Prefer returning NONE over selecting uncertain files
...
Return ONLY file paths (one per line) that DIRECTLY match the code changes.
If no files need updates, return "NONE".
```

The prompt is shouty for a reason: the failure mode the maintainer is fighting against is over-selection. LLMs love to "be helpful" and propose updates to half the docs tree because they share keywords with the diff. The shouting is empirical — see PR #9 which tuned this whole pipeline.

Key sentinels in this prompt:
- `NONE` → empty list returned
- A list of filenames, one per line → those files

The parser does light filtering after the call: `f.endswith('.adoc') or f.endswith('.md') or f.endswith('.rst')` — if the model hallucinates a non-doc file, it's dropped silently.

### The DOCS_SUBFOLDER prefix-stripping hack (discovery.py:280–292)

```python
# Strip DOCS_SUBFOLDER prefix if AI included it (common issue)
docs_subfolder = os.environ.get("DOCS_SUBFOLDER", "")
if docs_subfolder:
    cleaned_files = []
    for f in all_relevant_files:
        if f.startswith(docs_subfolder + "/"):
            cleaned_files.append(f[len(docs_subfolder) + 1:])
        ...
```

The model often parrots back paths with the subfolder prefix (since previews include the path). All downstream code expects paths **relative to the docs root** (we already `os.chdir`'d there), so this strip is required. This is a small but real source of bugs — if `DOCS_SUBFOLDER` has trailing slashes or weird casing, this won't fire and you'll get "file not found" errors during generation.

## Step 6 — Generate updates in parallel (suggest_docs.py:270–283, generation.py:21–72)

```python
files_with_content = generate_updates_parallel(
    diff, relevant_files, max_workers=args.max_workers,    # default 5
    user_instructions=user_instructions, file_instructions=file_instructions
)
```

Inside `generate_updates_parallel`, each file is processed in a ThreadPoolExecutor worker:

```python
def process_file(file_path):
    current = load_full_content(file_path)         # read the doc file
    if not current: return None
    updated = ask_ai_for_updated_content(
        diff, file_path, current,
        user_instructions=user_instructions,
        file_instructions=file_instructions
    )
    if updated.strip() == "NO_UPDATE_NEEDED":
        return None
    return (file_path, current, updated)
```

Two important details:

**`NO_UPDATE_NEEDED` is a sentinel string** baked into the generation prompt (generation.py:181–204). The prompt has explicit decision logic:
```
1. Does this file document the EXACT thing being changed in the diff?
   - If NO → return `NO_UPDATE_NEEDED`
   - If YES → continue
2. Does the diff add something NEW that should be documented?
   ...
```
So the LLM is doing a second pass of relevance filtering — discovery picked candidates, generation can still drop them. This is intentional belt-and-suspenders: discovery only saw previews, generation sees the full file.

**Format-aware prompts** (generation.py:96–164) — three big inline strings tell the model "write raw .md / .adoc / .rst, never wrap in ``` fences." The fence-wrapping is a chronic LLM failure mode (the model wants to be "helpful" by formatting the response as a code block). These format strings are the brittlest part of the file and a clear extraction candidate (see `notes/02-contributions.md` B1).

### The circular-import dance (generation.py:213–215)

```python
if file_instructions:
    # Local import to avoid circular dependencies
    from comments import _resolve_file_instructions
    per_file = _resolve_file_instructions(file_path, file_instructions)
```

`comments.py` already imports `get_client` from `config.py`, and `generation.py` already imports from `config.py` — but `_resolve_file_instructions` is a path-matching helper that lives in `comments.py` because it's about how user-typed file paths get resolved. Top-level import would create a cycle. Resolved by lazy import inside the function. The helper itself is 20 lines and could trivially move to `utils.py` — Contribution B2.

## Step 7 — Post the review comment (suggest_docs.py:311–313, comments.py:324–456)

This is where the user-facing magic happens. The comment is structured markdown that **encodes the entire state** the system needs to remember when `[update-docs]` runs later. There's no database. The PR comment IS the state.

```python
post_review_comment(files_with_content, pr_number, commit_info,
                    include_full_content=False, feature_section=feature_section)
```

`post_review_comment` calls `generate_summary_explanation`, which for each candidate file makes a small LLM call asking "in 1-2 sentences, what's this file about and what change do you suggest?" Files where the model returns `SKIP` (no real change vs original) are filtered out — that's why discovery → generation → comment can drop files at every stage.

Each surviving file becomes a line like:

```markdown
- [x] [config-ref.rst](https://github.com/.../blob/main/config-ref.rst): I suggest adding...
```

The `- [x]` is a **GitHub markdown checkbox**. When the user views the comment, they can uncheck files. When they later comment `[update-docs]`, the bot reads the comment back and parses which checkboxes are still ticked — see Step 8.

The full comment also contains `Latest commit: \`abc1234\`` so the next run can warn if the review is stale.

The temp-file write to `/tmp/review_comment.md` (line 431) is portability-fragile (won't work on Windows runners) and another listed contribution candidate (A2).

## What the workflow looks like from the user's perspective

After the comment is posted, the dev sees something like:

```
## 📚 Documentation Review

Latest commit: `abc1234`

Found 3 file(s) that may need updates:

### 📋 Select files to update

Uncheck any files you do not want updated:

- [x] [config-ref.rst](...): I suggest adding the new --quota flag to the CLI section.
- [x] [admin/setup.md](...): The suggested update is to mention the new env var FOO_BAR.
- [x] [release-notes.adoc](...): I recommend adding a 1.5.0 entry with the breaking change.

💡 Next Steps:
- Uncheck any files above that you don't want updated
- When ready, comment `[update-docs]` to create a PR with only the checked files
- ...
```

The dev unchecks `release-notes.adoc`, then comments `[update-docs] keep changes minimal`. That re-triggers the whole action.

## Step 8 — What changes when [update-docs] runs

(Briefly, since this is the variation, not the spine.)

The orchestrator branches at suggest_docs.py:209–227:

```python
if update_mode and not review_mode:
    user_instructions, file_instructions = parse_update_instructions(comment_body)
    previous_review = parse_previous_review(pr_number)
    if previous_review["review_found"]:
        # use only the still-checked files
```

`parse_previous_review` (comments.py:225–321) does `gh pr view --json comments`, walks comments **from newest to oldest**, finds the one with the `## 📚 Documentation Review` header, and runs a regex (comments.py:288–296) over checkbox lines:

```python
checkbox_pattern = re.compile(
    r'^- \[([ xX])\] '
    r'(?:'
    r'\[([^\]]+)\]\([^)]+\)'   # [path](url) form
    r'|'
    r'\*\*([^*]+)\*\*'          # **path** form
    r')'
    r':'
, re.MULTILINE)
```

Matches like `- [x] [config-ref.rst](url): summary` and `- [x] **config-ref.rst**: summary`. Files in accepted_files skip discovery entirely — we go straight to `generate_updates_parallel` with that exact list.

This regex is the system's parser for "human review state" and is also a known fragility (Contribution A4 — tolerate formatting drift like extra spaces or missing colons).

After generation, `push_and_open_pr` commits the modified files in the docs branch, pushes with `--force-with-lease`, checks if a PR already exists with `gh pr list --head <branch>`, and either creates a new one or just updates the existing branch. The user gets a confirmation comment back on the original PR with a unified diff for each updated file.

## The mental model

If you remember nothing else from this doc:

1. **`COMMENT_BODY` is the command line.** Mode detection is just substring matching.
2. **The PR comment is the state store.** There's no database, no cache that lives between action runs (except `.doc-index/`). The checkboxes in the bot's review comment ARE the state.
3. **Three filtering stages, decreasingly cheap.** Discovery picks candidates (cheap, parallel batches). Per-file generation can still drop with `NO_UPDATE_NEEDED` (medium, full file content). Summary generation can still drop with `SKIP` (cheapest, just diff vs new content). Each stage's job is to be conservative.
4. **Two trust boundaries.** `security_utils.run_command_safe` for subprocess. `validate_file_path` + `validate_docs_file_extension` for file writes. Anything that escapes those is a security review item.
5. **State changes that survive past one run live in two places only:** the docs repo's `main` branch (`.doc-index/` cache) and the docs repo's `doc-update-from-pr-<N>` branch (pending PR contents).

## Next walkthroughs (when you want them)

- **11 — the caching layer** (`doc_index.py`): folder-level semantic indexes, SHA256-keyed summaries, the commit-to-main loop with conflict handling
- **12 — the generation prompts in detail**: prompt engineering choices, format-specific tricks, the `truncate_diff` greedy-whole-file algorithm
- **13 — Jira / MCP / Google Docs** (`jira_integration.py`): MCP stdio client lifecycle, async-in-sync wrapping, the `gws` CLI subprocess

Let me know which to tackle next.
