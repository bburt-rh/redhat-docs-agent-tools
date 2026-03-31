---
name: docs-review-technical
description: Technical accuracy review and code-aware validation with confidence scoring. Supports local branch review, PR/MR review with optional inline comment posting, interactive comment actioning, and code-aware technical validation against source code repos. MUST BE USED when the user asks to validate documentation against code, check technical accuracy, verify commands/APIs/configs in docs match source code, or run a technical review. Also use when the user provides a --code URL or mentions code-aware review.
argument-hint: "[--local | --pr <url> [--post-comments] | --action-comments [url]] [--code <url>] [--fix] [--threshold <0-100>]"
allowed-tools: Read, Write, Glob, Grep, Edit, Bash, Skill, Agent, WebSearch, WebFetch, AskUserQuestion
---

# Technical Accuracy and Code-Aware Review

Multi-agent technical accuracy review with confidence-based scoring and optional code-aware validation against source repositories.

For style guide compliance and modular docs review, use `docs-tools:docs-review-style`.

## Modes

| Arguments | Mode | Description |
|-----------|------|-------------|
| `--local` | Local review | Review doc changes in current branch vs base branch |
| `--pr <url>` | PR/MR review | Review doc changes in a GitHub PR or GitLab MR |
| `--pr <url> --post-comments` | PR/MR + post | Review and post inline comments to PR/MR |
| `--action-comments [url]` | Action comments | Fetch and interactively action unresolved PR/MR review comments (auto-detects PR if URL omitted) |
| *(no arguments)* | Error | Display usage |

## Global Options

| Option | Description |
|--------|-------------|
| `--threshold <0-100>` | Confidence threshold for reporting issues (default: 80) |
| `--code <url>` | Code repository URL for technical validation (repeatable). Enables Agent 2. |
| `--fix` | Auto-fix high-confidence issues (>=65%), then interactively walk through remaining |

If no arguments are provided, display usage and ask the user to specify a mode.

## Agent Assumptions

These apply to ALL agents and subagents:

- All tools are functional. Do not test tools or make exploratory calls.
- Only call a tool if required. Every tool call should have a clear purpose.
- The confidence threshold is 80 by default (adjustable with `--threshold`).

---

# Multi-Agent Review Pipeline

The `--local` and `--pr` modes share the same pipeline. The difference is how files are discovered and how results are delivered.

## Step 1: Pre-flight Checks

### For --pr mode

Launch a haiku agent to run pre-flight checks using `docs-tools:git-pr-reader`. Stop if any condition is true (still review Claude-generated PRs):

- **PR/MR is closed or draft**: Check the PR/MR state from the platform API.
- **No documentation files changed**: Run `python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py files "${PR_URL}" --json` and check if any changed files end with `.adoc` or `.md`.
- **Claude already commented**: Run `python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py comments "${PR_URL}" --include-resolved --json` and check if any comment `author` matches Claude's username.

### For --local mode

```bash
CURRENT_BRANCH=$(git branch --show-current)
# Detect base branch from remote default, fall back to local refs
BASE_BRANCH=$(git rev-parse --abbrev-ref origin/HEAD 2>/dev/null | sed 's|^origin/||')
if [ -z "$BASE_BRANCH" ]; then
    if git show-ref --verify --quiet refs/heads/main; then
        BASE_BRANCH="main"
    elif git show-ref --verify --quiet refs/heads/master; then
        BASE_BRANCH="master"
    else
        echo "ERROR: Cannot determine base branch"; exit 1
    fi
fi
if [ "$CURRENT_BRANCH" = "$BASE_BRANCH" ]; then
    echo "ERROR: Currently on $BASE_BRANCH. Switch to a feature branch first."; exit 1
fi
```

## Step 2: Discover Documentation Files

### For --local mode

```bash
git diff --name-only "$BASE_BRANCH"...HEAD > /tmp/docs-review-all-files.txt
git diff --name-only HEAD >> /tmp/docs-review-all-files.txt
git diff --name-only --cached >> /tmp/docs-review-all-files.txt
sort -u /tmp/docs-review-all-files.txt | grep -E '\.(adoc|md)$' > /tmp/docs-review-doc-files.txt || true
DOC_FILES=$(wc -l < /tmp/docs-review-doc-files.txt)
```

### For --pr mode

Use `docs-tools:git-pr-reader` to get changed files:

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py files "${PR_URL}" --json | \
    python3 -c "import json,sys; files=[f['path'] for f in json.load(sys.stdin) if f['path'].endswith(('.adoc','.md'))]; print('\n'.join(files))" > /tmp/docs-review-doc-files.txt
```

### For both modes

If no documentation files found, report and exit.

## Step 3: Summarize Changes

Launch a sonnet agent to view changes and return a summary noting:
- Which files are new vs modified
- Whether files appear to be concepts, procedures, references, or assemblies
- Any structural patterns (modular docs, release notes)

For `--pr` mode: `python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py diff "${PR_URL}"`
For `--local` mode: `git diff "$BASE_BRANCH"...HEAD -- $(cat /tmp/docs-review-doc-files.txt)`

## Step 4: Agent 1 — Technical Accuracy and Consistency

- `subagent_type`: `docs-tools:technical-reviewer`
- `model`: `opus`

Follow the full technical review process: doc type detection, reviewer persona (developer/architect lens), 6 review dimensions, confidence scoring, and output format. Use `docs-tools:jira-reader`, `docs-tools:git-pr-reader`, and `docs-tools:article-extractor` to cross-check technical claims. Do not duplicate style or formatting checks.

Returns issues with: `file`, `line`, `description`, `reason`, `confidence` (0-100), `severity` (error/warning/suggestion).

For `--pr` mode, use `python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py extract` for deterministic line numbers.

**Important**: The agent file describes a JIRA-based drafts workflow for standalone use. In this context, ignore JIRA/drafts sections — review changed files from the diff and return issues in the format above.

## Step 5: Agent 2 — Code-Aware Technical Scan (conditional)

**Only runs when**: `--code <url>` is provided, or code repos can be auto-discovered from the PR URL, JIRA ticket context, or `:code-repo-url:` AsciiDoc attributes in the changed files.

**Dispatched as**: a general-purpose agent.

Workflow:

1. **Clone repos** to `/tmp/tech-review/<repo-name>/` using `git clone` (full history, not `--depth 1` — `git log` search needs history). Try specified ref, fall back to default branch.

2. **Extract references** from doc files:
   ```bash
   python3 scripts/code_scanner.py extract $(cat /tmp/docs-review-doc-files.txt) --output /tmp/tech-review-refs.json
   ```

3. **Search repos directly** — Read `/tmp/tech-review-refs.json` and search the cloned repos using native tools (Grep, Glob, Bash, Read). For each reference category:

   - **Commands**: Skip external system commands (sudo, grep, git, oc, kubectl, docker, podman, curl, ssh, systemctl, ansible, make, etc.). For project-specific commands, use Grep to find the binary in the repo. Read the source to check flag names, default values, and subcommands match the docs.
   - **Code blocks**: Use Grep to find the first meaningful line of each block in the repo. Read surrounding source to check values, function signatures, and import paths match.
   - **APIs** (functions/classes/endpoints): Use Grep to find definitions. Read the source to verify signatures, parameters, and return types match the docs.
   - **Configs**: Use Glob to find config files (*.yaml, *.yml, *.json, *.toml). Read them to compare key names, default values, and structure against what the docs show.
   - **File paths**: Use Glob to check if the documented path exists. If not, search by basename to detect renames.
   - **Git history**: When a reference is not found in code, use `git log --all --oneline --grep="<term>"` to check for renames, deprecations, or removals. This provides evidence for confidence scoring (e.g., "flag `--enable-feature` was renamed to `--feature-enable` in commit abc123").

   Focus on finding **concrete discrepancies**: wrong default values (e.g., "docs say pool_size=10, code says pool_size=5"), renamed flags, missing parameters, stale import paths, undocumented features. Do not report "not found in code" without evidence of a real problem.

4. Return issues in the standard format: `file`, `line`, `description`, `reason`, `confidence`, `severity`. Include the code evidence in `reason`.

## Step 6: Validate Agent 2 Findings

Review each issue reported by Agent 2. For each finding:

- Confirm the agent read the actual source file (not just grep output) and the discrepancy is real
- Confirm the issue is not a test fixture, example, or deprecated path that is intentionally different
- Assign confidence based on evidence quality: direct value mismatch = 90%+, renamed/moved = 70-85%, absent with no code evidence = skip
- Assign severity: `High` = users will hit errors. `Medium` = misleading but not blocking. `Low` = cosmetic.

Discard findings where the agent could not confirm the discrepancy by reading source code.

### Signal quality filter

**Flag issues where:**
- Documentation will actively mislead users (wrong commands, broken examples, incorrect terminology)
- Code examples contain wrong default values, renamed flags, or missing parameters
- API signatures, return types, or import paths don't match source code
- Configuration keys or values are stale or incorrect

**Do NOT flag:**
- "Not found in code" without concrete evidence of a problem
- Test fixtures, examples, or intentionally different deprecated paths
- External system commands (sudo, grep, git, etc.) that aren't project-specific
- Pre-existing issues in unchanged content
- Minor discrepancies that don't affect functionality

## Step 7: Validate All Issues

For each issue from Steps 4-6, launch parallel subagents to validate:
- Wrong command/flag -> verify the correct command exists in the code
- Stale API reference -> confirm the API was renamed or removed
- Broken code example -> verify the example doesn't compile/run as documented
- Incorrect config value -> confirm the actual default in source

Use opus subagents for structural/technical issues.

## Step 8: Filter Issues

Remove issues that:
- Were not validated in Step 7
- Score below the confidence threshold (default: 80)

## Step 9: Whole-Repo Anti-Pattern Scan (conditional)

**Only runs when Agent 2 ran.** Catches issues extraction+search may miss.

**Scan scope**: `.adoc` and `.md` files in the parent directories of the files listed in `/tmp/docs-review-doc-files.txt`.

**9a: Anti-pattern scan** — For each confirmed issue from Agent 2, use Grep to search the broader doc tree for additional occurrences of the same error pattern (e.g., same wrong flag name, same stale config key, same renamed path).

**9b: Blast radius scan** — For each issue from Step 6, search the doc tree for additional occurrences. Record every file and line.

## Step 10: Generate Report and Present Results

Write report to `/tmp/docs-review-technical-report.md` using the format below. Output summary to terminal:

```
## Technical Review

**Source**: <branch vs base | PR/MR URL>
**Files reviewed**: X documentation files
**Issues found**: Y (Z above confidence threshold)

### Issues

1. **file.adoc:23** [confidence: 95] — Flag `--enable-feature` renamed to `--feature-enable` in v2.3 (code-scan)
2. **file.adoc:67** [confidence: 88] — Default `pool_size` is 5, not 10 (technical-review)

### Skipped (below threshold)

- **file.adoc:91** [confidence: 55] — Config key `max_retries` not found in source

Full report saved to: /tmp/docs-review-technical-report.md
```

### For --local mode: Offer to Apply Changes

After the summary, offer to apply fixes for errors. Describe suggestions but let the user decide.

### For --pr mode without --post-comments

Stop here.

### For --pr mode with --post-comments

If NO issues found, post a summary comment via `git-pr-reader`:

```bash
cat <<'SUMMARY' > /tmp/docs-review-summary.json
[{"file": "", "line": 0, "message": "## Technical review\n\nNo issues found. Checked for technical accuracy and code-aware validation.", "severity": "suggestion"}]
SUMMARY
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py post "${PR_URL}" /tmp/docs-review-summary.json
```

If issues found, continue to Step 11.

## Step 11: Post Inline Comments (--post-comments only)

Get deterministic line numbers:
```bash
LINE=$(python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py extract "${PR_URL}" "path/to/file.adoc" "pattern from the issue")
```

Build comments JSON and post:
```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py post "${PR_URL}" /tmp/docs-review-comments.json
```

For each comment: brief description with evidence from source code, include corrected values for small fixes, describe larger fixes without inline code. **Only ONE comment per unique issue.**

## Step 11a: Fix Mode (--fix only)

**Phase A — Auto-fix**: For each issue with confidence >=65%, apply the fix using the Edit tool.

**Phase B — Interactive walkthrough**: For each issue with confidence <65%, present to user:

```
Issue 1 of 5: Command flag renamed | Confidence: 60% | Severity: High
File: modules/proc-install.adoc

Current:   $ my-tool --enable-feature
Suggested: $ my-tool --feature-enable

Evidence: Flag renamed in commit abc123
```

Ask user via AskUserQuestion: **Apply** | **Modify** | **Skip** | **Delete section**

---

# Mode: --action-comments

Fetch unresolved review comments from GitHub PRs or GitLab MRs and interactively action them.

## Step 1: Resolve PR/MR URL

If URL provided, use directly. If omitted, auto-detect:
```bash
PR_URL=$(python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py detect 2>/dev/null)
```

## Step 2: Fetch Unresolved Comments

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py comments "${PR_URL}" --json
```

The script automatically filters bot comments, resolved threads, and returns top-level comments only with: `id`, `path`, `line`, `body`, `author`, `resolved`.

## Step 3: Process Each Comment Interactively

For each unresolved comment, present:

```markdown
## Comment from @{author} on `{file_path}:{line}`

> {comment_body}

### Current Content
{relevant lines from the file}

### Suggested Change
{analysis and proposed change}
```

Ask user via AskUserQuestion: **Apply** | **Edit** | **Skip** | **View context**

When approved: Read target file, apply Edit, confirm, move to next.

## Step 4: Summary

```markdown
## Summary

| Metric | Count |
|--------|-------|
| Total comments | X |
| Comments addressed | Y |
| Comments skipped | Z |
| Bot comments filtered | N |
```

Categorize comments: **Required** (technical errors — must fix), **Suggestion** (improvements — user discretion), **Question** (needs discussion), **Outdated** (already addressed — skip).

---

# Report Format

```markdown
# Technical Review Report

**Source**: [Branch: <branch> vs <base> | PR/MR URL]
**Date**: YYYY-MM-DD

## Summary

| Metric | Count |
|--------|-------|
| Files reviewed | X |
| Errors (must fix) | Y |
| Warnings (should fix) | Z |
| Suggestions (optional) | N |

## Files Reviewed

### 1. path/to/file.adoc

**Type**: CONCEPT | PROCEDURE | REFERENCE | ASSEMBLY

#### Technical Accuracy

| Line | Severity | Issue | Evidence |
|------|----------|-------|----------|

#### Code Validation (if Agent 2 ran)

| Line | Severity | Issue | Evidence |
|------|----------|-------|----------|

Show specific value mismatches (e.g., "Docs: pool_size=10, Code: pool_size=5"), undocumented features, and import path errors. Do not list every `found: false` result — only report items where there is concrete evidence of a discrepancy or where a documented feature is provably absent from the code.

---

## Required Changes

1. **file.adoc:23** — Description (evidence)

## Suggestions

1. **file.adoc:91** — Description

---

*Generated with [Claude Code](https://claude.com/claude-code)*
```

**Sections**: Errors = must fix. Warnings = should fix. Suggestions = optional.

**Do NOT include**: positive findings, executive summaries, compliance percentages, references sections.

## Feedback Guidelines

- **In scope**: Content changed in the branch or PR/MR. **Out of scope**: Unchanged content, enhancement requests.
- **Required** (blocks merging): Incorrect commands, wrong API references, broken code examples, stale config values.
- **Optional** (does not block): Minor accuracy improvements, additional context. Mark with **[SUGGESTION]**.
- Include source code evidence for each issue. For recurring issues: "[GLOBAL] This issue occurs elsewhere."

---

# Notes

- Always use `python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py` for all Git platform interactions (see `docs-tools:git-pr-reader` for full API reference)
- Always use `git_pr_reader.py extract` for deterministic line numbers — never estimate or guess
- Use Bash with heredoc/cat for writing /tmp files (not the Write tool)
- Include source code evidence in each issue's `reason` field
- Comments are posted under YOUR username using tokens from `~/.env`
- The `code_scanner.py` script is co-located in `scripts/` for Agent 2's reference extraction
- Vale linting is NOT part of the technical review — use `docs-tools:docs-review-style` for that
