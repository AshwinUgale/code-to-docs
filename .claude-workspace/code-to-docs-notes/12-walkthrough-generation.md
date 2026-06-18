# Walkthrough 3 — The generation pipeline (`src/generation.py`)

Walkthroughs 1 and 2 covered the orchestrator and the caching layer. This one covers the third stage: **once we know which files to update, how do we actually update them?**

Read with `src/generation.py` open. This is a short file (~265 lines), but the prompt at lines 167-205 is doing more work than it looks like. That's where to focus.

Why this walkthrough matters for PR 1: the content judges in our eval (the `must_mention` / `must_not_mention` / `max_new_lines` / `must_not_add_sections` assertions, plus the global format checks) all target failure modes that *this code's prompts* try to prevent. You can't write good assertions without understanding what the prompt is trying to make the LLM do — or, more importantly, what the prompt is fighting against.

## The public surface

```python
generate_updates_parallel(diff, relevant_files, max_workers=5, user_instructions="", file_instructions=None)
    → list[(file_path, original_content, updated_content)]

ask_ai_for_updated_content(diff, file_path, current_content, user_instructions="", file_instructions=None)
    → str    # either "NO_UPDATE_NEEDED" or the rewritten file

load_full_content(file_path)
    → str    # current file contents, or "" on validation failure

overwrite_file(file_path, new_content)
    → bool   # True on success
```

That's it. `suggest_docs.py` calls `generate_updates_parallel`; everything else is internal.

## Step 1 — Parallel dispatch (lines 21-72)

```python
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    futures = {
        executor.submit(process_file, file_path): file_path
        for file_path in relevant_files
    }
    for future in as_completed(futures):
        result = future.result()
        if result:
            results.append(result)
```

One thread per file, capped at 5. Each thread:
1. Reads the current file content (with path validation).
2. Calls `ask_ai_for_updated_content`.
3. Drops the result if it's the literal string `"NO_UPDATE_NEEDED"`.
4. Otherwise returns the `(path, original, updated)` tuple.

Threads, not processes, because the work is I/O-bound (LLM HTTP calls). The `client` returned by `get_client()` is a thread-safe `openai.OpenAI` instance — same client object used across all threads.

**`as_completed` returns futures in finish order**, not submission order. That means the result list is not in `relevant_files` order. Downstream code (the review comment builder, the PR builder) treats it as a set. Don't assume ordering.

## Step 2 — Load the current file (lines 75-88)

```python
def load_full_content(file_path):
    if not validate_file_path(file_path):
        print(f"❌ Security: Invalid file path rejected: {file_path}")
        return ""
    return Path(file_path).read_text(encoding="utf-8")
```

`validate_file_path` (in `security_utils.py`) is the **read** trust boundary. Wraps `path.resolve()` to canonicalize, then checks it's inside the docs root. Path traversal (`../../etc/passwd`) is rejected here.

Returns `""` on failure (not `None`, not raise). The caller `process_file` checks `if not current: return None` — so a validation failure makes the whole file silently disappear from the result list. Not ideal for debugging, but it's also not catastrophic.

Note: this runs from the docs root (after `setup_docs_environment()` chdir'd there). `file_path` is relative. The validation gate is what keeps an LLM-generated path like `../../../../home/user/.ssh/id_rsa` from getting written to.

## Step 3 — The format-specific prompt block (lines 96-164)

Three big inline string constants for `.md` / `.adoc` / `.rst`, plus a fallback. They look verbose but they're surgical — every line is fighting a specific LLM failure mode. The Markdown block (lines 96-112) is the prototype:

```
CRITICAL FORMATTING REQUIREMENTS FOR MARKDOWN FILES:
**MOST IMPORTANT**: The output must be RAW MARKDOWN content that can be written DIRECTLY to a .md file.
- NEVER wrap the output in code fences like ```markdown or ```
- The FIRST character of your response should be the FIRST character of the file
- The LAST character of your response should be the LAST character of the file content
- NO "```markdown" at the beginning
- NO "```" at the end
- Return ONLY the raw file content, nothing else
...
```

That repetition isn't a typo. The LLM **wants** to wrap the output in a code fence because it's the "polite" thing to do — every chat-tuned LLM has been trained to format file content as a code block. The prompt is shouting at it to stop. And it still does it sometimes. Which is why one of our **global judge assertions** is "first non-blank line is not ```". The prompt fights the failure mode; the judge catches when the prompt loses.

The other failure modes the format blocks target:
- **Format mixing.** "Do NOT mix AsciiDoc syntax with Markdown" — LLMs trained on a mix of doc formats will splice them. `===== Header =====` showing up in a `.md` file. The format check "output parses as the declared format" catches this.
- **Table corruption.** Markdown blocks call out `|---|---|---|` separators explicitly — LLMs love to add backslashes or extra characters that look right but break the table. Hard to catch via assertion, but `max_new_lines` bounds the damage.
- **Cross-reference loss.** RST block (line 153) calls out `:ref:`, `:doc:`, `:class:` etc. — RST's cross-refs are easy for the LLM to "simplify" into broken links. Assertions don't easily catch this. v2 LLM-as-judge will.

The format dispatch is by extension only — no content sniffing. So a `.rst` file with no RST directives gets the same prompt as a complex Sphinx project. Workable, but means a freshly-created file with `# Header` syntax in a `.rst` would get told "use ===== underlines" and the LLM might rewrite the existing Markdown-style content. Hand-authored case for the corpus.

## Step 4 — The DECISION LOGIC prompt block (lines 167-205) — the heart of the file

This is the structural template the format block is embedded in. Lightly trimmed:

```
You are updating documentation based on a code diff. Be EXTREMELY conservative.

{format_instructions}
- Ensure consistent indentation and spacing

Git diff:
{DIFF_PLACEHOLDER}

Current documentation file `{file_path}`:
--------------------
{current_content}
--------------------

DECISION LOGIC:
1. Does this file document the EXACT thing being changed in the diff?
   - If NO → return `NO_UPDATE_NEEDED`
   - If YES → continue

2. Does the diff add something NEW that should be documented?
   - If NO → return `NO_UPDATE_NEEDED`
   - If YES → continue

3. Is that new thing already documented in this file?
   - If YES → return `NO_UPDATE_NEEDED`
   - If NO → add ONLY that specific change

WHAT YOU CAN ADD:
- Only content that directly reflects what was added/changed in the diff

WHAT YOU MUST NOT ADD:
- New sections or paragraphs not justified by the diff
- "Helpful" additions you think users might want
- Restructured or rewritten content

Return ONLY:
- `NO_UPDATE_NEEDED` (strongly preferred if changes aren't essential), OR
- The complete updated file with ONLY the minimal necessary changes
```

Three things to internalize:

**1. The decision tree is explicitly three gates.** Each one is an opportunity for the LLM to short-circuit to `NO_UPDATE_NEEDED`. Conservative-by-default. This is the same fight-against-over-selection that the discovery prompts have (`NONE`), now at the per-file level. If discovery picked a candidate but generation says no, the file is dropped.

**2. `NO_UPDATE_NEEDED` is a literal sentinel.** Not a description, not a structured response. The caller does `if updated.strip() == "NO_UPDATE_NEEDED"`. Trailing whitespace is OK; anything else (even a single character difference) is treated as a rewritten file. **Our content judges depend on this contract.** A `must_not_add_sections: true` assertion is meaningful only when the LLM returns either `NO_UPDATE_NEEDED` or a real rewritten file — not "Sure, here's the updated content: ...".

**3. "WHAT YOU MUST NOT ADD" maps directly to our assertions.**
- `must_not_add_sections` → catches "New sections or paragraphs not justified by the diff"
- `max_new_lines` → bounds "Helpful additions you think users might want"
- `must_not_mention` → catches the LLM adding things from outside the diff (a common hallucination)
- The global format checks → catch the format violations from Step 3

The eval is, mechanically, **measuring how well the LLM obeys this prompt's NOT-rules.** That's the lens.

## Step 5 — Reviewer instruction injection (lines 207-225)

```python
combined_instructions = []
if user_instructions:
    combined_instructions.append(f"Global: {user_instructions}")
if file_instructions:
    # Local import to avoid circular dependencies
    from comments import _resolve_file_instructions
    per_file = _resolve_file_instructions(file_path, file_instructions)
    if per_file:
        combined_instructions.append(f"For this file specifically: {per_file}")

if combined_instructions:
    prompt_template += f"""

ADDITIONAL INSTRUCTIONS FROM THE REVIEWER:
The human reviewer has provided the following guidance. Follow these instructions carefully:
{chr(10).join(combined_instructions)}
"""
```

This is the seam where the `[update-docs] some instruction` user input lands. Two sources:
- **Global**: first line after `[update-docs]` in the user's comment. Parsed in `comments.parse_update_instructions`.
- **Per-file**: lines like `config-ref.rst: only update the CLI usage example`. Same parser.

`_resolve_file_instructions` lives in `comments.py` but is needed here. The lazy import inside the `if` block is the workaround for the circular import (covered in walkthrough 1). Hard to clean up cleanly because `comments.py` already imports `get_client` from `config` and uses `generation.py`'s output for diff rendering. Contribution B2 in the contributions doc moves this helper to `utils.py` to break the cycle — but that's parked.

The injected block appears **after** the DECISION LOGIC. Reviewer instructions can guide what to add, but they can't override the conservatism gate — if the LLM decides the file doesn't document the changed thing, even an instruction "add a section about FOO" gets `NO_UPDATE_NEEDED`'d. Or that's what the prompt says, anyway. Whether the LLM actually obeys is an empirical question, and one a v2 eval with a "did the LLM follow the instruction" judge could measure.

**For our eval**: PR 1 cases don't exercise the instruction injection path. Leave `user_instructions=""`, `file_instructions=None`. If we want to test this in v2, add cases with non-empty instructions and assertions on whether they were respected.

## Step 6 — Diff truncation (lines 227-229)

```python
diff_budget = get_max_context_chars() - len(prompt_template)
truncated_diff = truncate_diff(diff, diff_budget, label=f"update diff for {file_path}")
prompt = prompt_template.replace("{DIFF_PLACEHOLDER}", truncated_diff)
```

The budget math: total context window (default 400,000 chars), minus everything else in the prompt template (format block + decision logic + current file content + instructions) = how much room is left for the diff.

`truncate_diff` (in `config.py:79-126`, covered in walkthrough 1) is the greedy whole-file-diff algorithm. Splits on `diff --git ` boundaries, includes complete file-diffs greedily, falls back to character truncation only if a single file-diff exceeds budget. Annotates the result with a `[... truncated: showing N/M complete files ...]` marker.

The implication: **on a large diff, the LLM sees a different subset of files for each docs file we generate**. The truncation isn't case-aware — it takes whatever fits, in order. If the eval gets weird single-case results because the diff was truncated to exclude the relevant hunk, that's the cause. The runner should log how much truncation happened.

`get_max_context_chars()` honors `MAX_CONTEXT_CHARS` env var. For the eval, set this conservatively to avoid surprises. We probably want it ~32K for fast feedback at first.

## Step 7 — The LLM call (lines 231-242)

```python
client = get_client()
model_name = get_model_name()
try:
    response = client.chat.completions.create(
        model=model_name,
        messages=[{"role": "user", "content": prompt}],
    )
    return (response.choices[0].message.content or "").strip()
except Exception as e:
    check_context_error(e)
    raise
```

Single-shot. No retries, no streaming, no tool use, no system prompt. The whole rule book lives in the user message.

`check_context_error` (config.py:129) is the "decode an OpenAI BadRequestError into something actionable" helper. If the error message contains "context length" / "maximum context" / "number of tokens" / "token limit", it prints a hint about setting `MAX_CONTEXT_CHARS` lower. The exception is then **re-raised** — the function only returns when the call succeeded. Important: this means a content gen failure crashes the thread, the parallel orchestrator catches it (`except Exception` at line 69 of generation.py), prints the error, and the file is dropped from the results. Same effect as a `NO_UPDATE_NEEDED` from the caller's perspective — but for a very different reason.

**For our eval**: API errors should be flagged in the runner output as flakes, not silent NO_UPDATE_NEEDED. Otherwise a transient 503 looks identical to "the LLM said no update needed," which destroys the signal.

## Step 8 — Write the file (lines 244-264)

```python
def overwrite_file(file_path, new_content):
    if not validate_file_path(file_path):
        print(f"❌ Security: Invalid file path rejected: {file_path}")
        return False
    if not validate_docs_file_extension(file_path):
        print(f"❌ Security: Only .adoc, .md, and .rst files allowed: {file_path}")
        return False
    Path(file_path).write_text(new_content, encoding="utf-8")
    return True
```

Two trust gates before writing:
1. **Path validation** (same as read): inside docs root, no traversal.
2. **Extension allowlist**: `.adoc`, `.md`, `.rst` only. Nothing else gets overwritten. So even if the LLM hallucinates a path like `secret-config.yaml`, the gate stops it.

This is the **write** trust boundary. Anything that escapes both is the security review surface. `validate_file_path` is in walkthrough 1's "two trust boundaries" mental model.

`overwrite_file` is called from `suggest_docs.py` after the review-comment dance, not from inside the parallel worker. The worker just produces the `(path, original, updated)` tuple. Whether to write depends on the mode (`review-docs` doesn't write, `update-docs` does).

**For our eval**: we never call `overwrite_file`. The runner captures the `updated` content in memory and writes it to the runs/ directory, not back to the case's `docs/` tree. The case's `docs/` tree is treated as immutable input.

## Mental model

If you remember nothing else:

1. **The prompt is the spec.** The DECISION LOGIC tree (decline three times before adding anything) and the WHAT-YOU-MUST-NOT-ADD list are the system's design contract. Our assertions are mechanical checks that the spec is being followed.

2. **`NO_UPDATE_NEEDED` is a load-bearing sentinel.** Literal string, trimmed, exact match. The caller branch on this is the difference between "the file disappears from the result list" and "the file gets a proposed rewrite."

3. **Format-specific prompts fight a chronic LLM failure mode** (fence-wrapping, format mixing). Verbosity is empirical, not stylistic. Global format judge assertions are the last line of defense.

4. **Two trust boundaries.** Read = `validate_file_path`. Write = `validate_file_path` + `validate_docs_file_extension`. Same pattern as walkthrough 1. The eval doesn't trigger write — only read.

5. **The truncated diff is per-file, not per-run.** Different files might see different subsets of the diff. If you see weird per-case results, suspect truncation.

6. **The reviewer instruction seam exists** but we don't exercise it in PR 1. v2 territory.

## What this means for execute.py

Pulled together, the runner has to:
- Set `MAX_CONTEXT_CHARS` to something predictable (probably 32K or 64K — small but enough)
- Capture the literal return string from `ask_ai_for_updated_content`. Don't .strip() further (the function already does). Don't parse. Don't normalize.
- Write the raw string to the runs directory. Both `NO_UPDATE_NEEDED` and the rewritten content are valid outputs.
- For each case + N invocations: log timing, diff truncation %, and any exception. Flakes are not failures, but they're not successes either.
- Never call `overwrite_file`. The case's `docs/` tree is fixture, not target.
- Pass `user_instructions=""` and `file_instructions=None`. No reviewer instruction in PR 1.

## What this means for the judges

- `must_mention`: substring search on the rewritten content. If the case expected an update and the LLM returned `NO_UPDATE_NEEDED`, all `must_mention` items fail (one fail per item, not a global fail — gives per-item signal).
- `must_not_mention`: substring search on the rewritten content. If `NO_UPDATE_NEEDED`, trivially passes (nothing to mention).
- `max_new_lines`: count `(len(updated.splitlines()) - len(original.splitlines()))`. If `NO_UPDATE_NEEDED`, delta is 0, passes.
- `must_not_add_sections`: count `#`-prefixed lines (or RST/AsciiDoc heading equivalents) in original vs updated, fail if updated > original. If `NO_UPDATE_NEEDED`, delta is 0, passes.
- Global: parses as declared format / balanced fences / no leading ``` — all skipped if `NO_UPDATE_NEEDED`. Only run on actual rewrites.

These all interact correctly with the sentinel, but the symmetry isn't obvious unless you've internalized that `NO_UPDATE_NEEDED` means "nothing happened" and is a valid passing answer for negative cases.

## Next

After you've read this, the next concrete piece is either:
- The REPL exercise — invoke `ask_ai_for_updated_content` by hand with a tiny example, see the prompt's failure modes happen live
- The agent-eval-harness read — `score.py` + `eval.yaml` + runs/ layout, to understand the contract `execute.py` has to honor

I'd do the REPL first — 30 minutes — because it gives you ground truth for "what does the LLM actually do" which makes reading the harness more concrete.
