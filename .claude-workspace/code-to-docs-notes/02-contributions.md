# Contribution Plan — code-to-docs

## The landscape

The repo has **zero open issues, zero open PRs, and zero external contributors**. All 12 historical PRs are authored and self-merged by the maintainer `csoceanu` (Carmel Soceanu). There is no triaged "good first issue" backlog to pick from.

This means the standard OSS path (find an issue → claim it → PR) doesn't apply. The path that will actually work:

1. **Open an issue first** describing the bug or improvement before writing any code. Keep it specific (file/line refs, repro steps, proposed fix). This both confirms csoceanu wants the change and creates a public record of the work.
2. **Start small.** A first PR that's tiny, low-risk, and obviously useful (test coverage, a docstring fix, a regex hardening) is far more likely to merge than a 500-line refactor.
3. **Match their style.** All their PRs follow conventional-commit titles (`feat:`, `docs:`, `refactor:`). README+demo are both kept current — touch them when you change behavior.

## Candidates derived from code review

Ranked by `(value × likelihood-of-merge) / effort`. Cite real file:line refs so the issue/PR descriptions are concrete.

### A. Low-effort, high-likelihood-of-merge

**A1. Add tests for `jira_integration.py`** *(effort: 1–2 days)*
Currently only `parse_feature_command` is tested (`tests/test_comment_parsing.py:16`). `fetch_jira_context_sync`, `fetch_google_doc`, `analyze_feature_coverage`, `_find_all_links` are all untested despite being the most fragile module (async + MCP + Node CLI subprocess). Mock the MCP `stdio_client` and `subprocess.run` for `gws`. Test the URL-extraction regex against weird inputs (trailing punctuation, percent-encoding, query strings). Issue title: *"Add unit tests for jira_integration module"*.

**A2. Replace `/tmp/*.md` writes with stdin-piped `gh pr comment`** *(effort: 1–2 hrs)*
Four hardcoded `/tmp/` paths: `comments.py:431`, `suggest_docs.py:89`, `:118`, `:382`. Non-portable (no Windows runners) and races if two action runs collide on the same self-hosted runner. `gh pr comment N --body-file -` reads from stdin. Use `subprocess.run(..., input=body.encode(), ...)`. Issue title: *"Avoid hardcoded /tmp paths when posting comments"*.

**A3. Parameterize `MAX_FILES_PER_BATCH`** *(effort: 30 min)*
`discovery.py:163` hardcodes 10. The README already has a precedent for tunable context with `MAX_CONTEXT_CHARS`. Add env var `MAX_FILES_PER_BATCH` (default 10), document it in README. Issue title: *"Make discovery batch size configurable"*.

**A4. Tighten checkbox parsing regex** *(effort: 1–2 hrs + tests)*
`comments.py` checkbox parser expects exactly `- [x] [path](url): summary` or `- [x] **path**: summary`. Users sometimes edit GitHub comments and reflow whitespace, drop the colon, or remove the URL. Add tolerant variants and a test matrix. Issue title: *"Make `[update-docs]` review-comment parser tolerant of formatting drift"*.

### B. Moderate effort, clear win

**B1. Extract `generation.py` format prompts to template files** *(effort: half day)*
Lines 97–163 are 60+ lines of inline Markdown/AsciiDoc/RST instruction strings. Extract into `src/templates/format_*.txt` loaded with `pathlib.Path(__file__).parent / "templates"`. Makes it easier to add Sphinx/MyST variants. Issue title: *"Move format-specific prompt boilerplate into template files"*.

**B2. Break the `generation.py` ↔ `comments.py` circular import** *(effort: half day)*
`generation.py:214` does a lazy `from comments import _resolve_file_instructions` mid-function to avoid a circular import at module load. The function is short — moving it into `utils.py` or a new `instructions.py` removes the cycle and the workaround. Pure refactor, easy to review. Issue title: *"Resolve circular import between generation and comments"*.

**B3. End-to-end smoke test for `suggest_docs.py`** *(effort: 1–2 days)*
`tests/test_suggest_docs.py` exists but covers helpers only. Build a fixture repo (in-memory or `tmp_path`) with a fake docs subfolder and a fake `git diff`, mock the OpenAI client, run `main()` for each of the three commands, assert the right `gh` calls would have been made. This catches integration regressions the unit tests miss. Issue title: *"Add end-to-end smoke test for the orchestrator"*.

### C. Higher risk, propose before building

**C1. Stop putting GitHub PAT in `git config` as plaintext** *(effort: 1 day, security-sensitive)*
`security_utils.setup_git_credentials()` stores the token via `git config credential.helper 'store --file=...'`. Visible to anything that runs `git config --list` inside the container. Safer pattern: `git -c http.extraheader="AUTHORIZATION: bearer $TOKEN"`. Mention this is a security improvement, not a vulnerability — the container is single-tenant per action run. Issue title: *"Use HTTP Authorization header instead of credential.helper for git auth"*.

**C2. Reject symlinks in `validate_file_path()`** *(effort: half day, security-sensitive)*
Currently `path.resolve()` happily follows symlinks out of the docs base. Add a check that `resolved_path.is_relative_to(base_dir)` *and* `not path.is_symlink()`. Likely accepted if framed as defense-in-depth. Issue title: *"Reject symlink targets in docs-path validation"*.

**C3. Robust `gws` CLI output parsing** *(effort: 1 day)*
`jira_integration.py:134–150` has cascading fallbacks (`output_file`, `download.txt`, JSON metadata) because the CLI's output format isn't pinned. Lock the call to `gws drive export --format=text --output=<explicit>` and fail loudly on schema drift. Touches an external tool — propose the approach in an issue before implementing.

## Suggested first PR

**Start with A3 (`MAX_FILES_PER_BATCH`) or A2 (stdin-piped `gh` calls).** Both are sub-day, mechanically simple, obviously correct, touch one or two files, and have minimal review surface. They give you a merged PR on the board before you reach for anything substantive.

After that, A1 (jira_integration tests) is the highest-value follow-up because it's the maintainer's biggest blind spot — the most-fragile module is untested.

## Workflow checklist

1. Fork on GitHub (web UI).
2. Clone your fork to a normal working folder on Windows (not the Cowork mount — git can't write there).
3. Branch: `git checkout -b feat/configurable-batch-size`.
4. Open the corresponding issue on the upstream repo first; link it from your PR.
5. Match commit style (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`).
6. Run `pytest tests/` and any new tests you add — see `03-local-setup.md` for the env.
7. Push to your fork; open PR against `redhat-community-ai-tools/code-to-docs:main`.
8. Tag csoceanu in the PR description.

## Talking points for the issue / PR description

When opening any of the above, you have credibility built in by being able to reference exact file:line. Concrete grounded suggestions are far more likely to be reviewed than abstract ones. A good opening sentence template:

> While studying the codebase I noticed `<file>:<line>` does `<thing>`. This causes `<concrete problem>` because `<reason>`. Happy to send a PR — proposed approach: `<approach>`.

That's much harder to ignore than "I'd like to contribute, where do I start?"
