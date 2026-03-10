---
description: Run the two-phase technical review workflow to validate documentation against code repositories and interactively apply fixes. Phase 1 auto-fixes high-confidence issues, Phase 2 walks through remaining fixes interactively.
argument-hint: --docs <source> [--docs <source>...] [--code URL] [--jira TICKET] [--pr URL] [--gdoc URL] [--fix] [--auto] [--phase1-only]
allowed-tools: Read, Write, Bash, Skill, AskUserQuestion
---

# Technical Review Workflow

Orchestrates the complete technical review workflow for validating documentation against code repositories:
- **Phase 1**: Validate docs against code repos, auto-fix high-confidence issues (>=65%)
- **Phase 2**: Interactively review and apply lower-confidence fixes (<65%)

This command combines two separate skills into a unified workflow:
1. `docs-technical-review-validate` - Phase 1 validation and auto-fix
2. `docs-technical-review-apply` - Phase 2 interactive agentic review

## Benefits

- **Simpler for casual users**: Single command for full workflow
- **Pause points for review**: User can review Phase 1 results before continuing
- **Maintains reusability**: Individual skills can still be called independently

## Required Argument

- **--docs \<source\>**: (required) Path to documentation files or directory. Can be specified multiple times to review multiple sources.

**IMPORTANT**: This command requires at least one `--docs` argument. If none is provided, stop and ask the user to provide one.

## Repository Discovery Options

Specify one or more methods to discover code repositories for validation:

- **--code URL**: Explicit code repository URL
- **--ref BRANCH**: Git ref for previous --code (default: main)
- **--jira TICKET-123**: Auto-discover repos from JIRA ticket
- **--pr URL**: Auto-discover from PR/MR URL (can specify multiple)
- **--gdoc URL**: Auto-discover from Google Doc

**Discovery priority** (first to last): `--code` > `--pr` > `--jira` > `--gdoc` > AsciiDoc `:code-repo-url:` attributes

## Options

| Option | Description |
|--------|-------------|
| --fix | Enable auto-fixing of high-confidence issues (>=65%) |
| --dry-run | Preview changes without applying |
| --auto | Run both phases without pausing (useful for CI/CD) |
| --phase1-only | Stop after Phase 1 validation and auto-fix |
| --confidence RANGE | Phase 2 only: confidence range to apply (e.g., "50-64") |
| --items ID,ID | Phase 2 only: specific item IDs to apply (e.g., "MR-1,MR-3") |

## Usage Examples

### Interactive Mode (Recommended)

```bash
/docs-tools:docs-technical-review --docs modules/ \
  --jira RHAISTRAT-123 --fix
```

### Multiple Documentation Sources

```bash
/docs-tools:docs-technical-review --docs modules/ --docs guides/admin/ \
  --jira RHAISTRAT-123 --fix
```

### Automatic Mode (No Pause)

```bash
/docs-tools:docs-technical-review --docs modules/ \
  --jira RHAISTRAT-123 --fix --auto
```

### Phase 1 Only

```bash
/docs-tools:docs-technical-review --docs modules/ \
  --jira RHAISTRAT-123 --fix --phase1-only
```

### Multiple Repository Sources

```bash
/docs-tools:docs-technical-review --docs modules/ \
  --jira RHAISTRAT-123 --pr https://github.com/org/repo/pull/456 \
  --code https://github.com/org/extra-repo --fix
```

### Dry Run

```bash
/docs-tools:docs-technical-review --docs modules/ \
  --code https://github.com/nodejs/node --dry-run
```

## Step-by-Step Instructions

### Step 1: Parse Arguments

Extract the following from the user's command arguments:

| Argument | Type | Collects into |
|----------|------|---------------|
| `--docs` | repeatable | `DOCS_SOURCES` array |
| `--code` | single | `CODE_URL` |
| `--ref` | single | `REF` (default: main) |
| `--jira` | single | `JIRA` |
| `--pr` | repeatable | `PR_URLS` array |
| `--gdoc` | single | `GDOC` |
| `--fix` | flag | `FIX` |
| `--dry-run` | flag | `DRY_RUN` |
| `--auto` | flag | `AUTO` |
| `--phase1-only` | flag | `PHASE1_ONLY` |
| `--confidence` | single | `CONFIDENCE_RANGE` (Phase 2 only) |
| `--items` | single | `ITEM_IDS` (Phase 2 only) |

If `DOCS_SOURCES` is empty, stop and ask the user to provide `--docs`.

### Step 2: Validate Prerequisites

Verify at least one repository discovery method is provided (`--code`, `--jira`, `--pr`, or `--gdoc`). If none, stop with an error listing the available options.

### Step 3: Run Phase 1 (Validation & Auto-Fix)

Build Phase 1 arguments by forwarding `--docs`, `--code`, `--ref`, `--jira`, `--pr`, `--gdoc`, `--fix`, and `--dry-run` to the skill invocation.

Invoke Phase 1 using the Skill tool:
- skill: `docs-technical-review-validate`
- args: all forwarded arguments

After Phase 1 completes, verify the report file exists at `.claude_docs/technical-review-report.md`. If missing, stop with an error.

### Step 4: Display Phase 1 Summary

Read the report file and extract summary statistics:
- Total items validated
- Auto-fixed count
- Manual review needed count

Display these to the user.

### Step 5: Decide Whether to Continue to Phase 2

**Phase 1 only mode** (`--phase1-only`): Stop and show how to run Phase 2 later with `/docs-technical-review-apply`.

**No manual review items**: Stop with success message.

**Auto mode** (`--auto`): Proceed directly to Phase 2.

**Interactive mode** (default): Use AskUserQuestion with these options:
1. **Apply all items interactively** (Recommended) - approve, modify, or skip each fix
2. **Apply specific confidence range** - follow up to ask which range
3. **Apply specific items by ID** - follow up to ask which IDs
4. **I'll do it manually later** - stop and show how to resume

### Step 6: Run Phase 2 (Interactive Apply)

If user chose to continue, build Phase 2 arguments from the report file path plus any `--confidence` or `--items` values.

Invoke Phase 2 using the Skill tool:
- skill: `docs-technical-review-apply`
- args: report file path plus filtering options

### Step 7: Display Final Summary

Display completion message with next steps:
1. Review changes made to documentation files
2. Run Vale to check for style issues
3. Test updated code examples and commands
4. Commit changes to version control

## Workflow Steps Summary

### Phase 1: Validation & Auto-Fix
1. **Discover code repositories** from JIRA, PRs, Google Docs, or explicit URLs
2. **Clone repositories** to `/tmp/tech-review/`
3. **Extract technical references** (commands, code blocks, APIs, configs, file paths)
4. **Validate against code** using fuzzy matching and git history
5. **Auto-fix high-confidence issues** (>=65%)
6. **Perform whole-repo scan** for related issues across all .adoc files
7. **Generate detailed report** at `.claude_docs/technical-review-report.md`

### Pause Point
- **Interactive mode**: User reviews Phase 1 report and decides whether to continue
- **Auto mode**: Proceeds directly to Phase 2
- **Phase 1 only**: Stops here

### Phase 2: Interactive Agentic Review
1. **Read report** from Phase 1
2. **Filter items** by confidence range or specific IDs (if specified)
3. **Present each issue** with evidence, confidence score, and suggested fix
4. **Get user approval** for each fix
5. **Apply approved changes** to documentation files

See the `docs-technical-review-validate` skill for validation categories and confidence scoring details.

## Output Files

1. **`.claude_docs/technical-review-report.md`** - Human-readable report with summary, auto-fixed diffs, manual review items, and whole-repo scan results
2. **`.claude_docs/technical-review-report.json`** - Structured issue data for Phase 2 consumption
3. **`/tmp/tech-review/<repo-name>/`** - Cloned code repositories, retained for inspection

## Error Handling

- **Phase 1 fails**: Stop workflow, display error, exit
- **No repos discovered**: Phase 1 will error, workflow stops
- **Report file missing**: Error and exit
- **Phase 2 fails**: Display error (Phase 1 changes are already applied)
- **User cancels Phase 2**: Exit gracefully, remind how to resume

## Prerequisites

- `ruby` - Ruby interpreter (for reference extraction and search scripts)
- `python3` - Python 3 (for JIRA and Git review API scripts)
- `jq` - JSON processor (for parsing reports)
- `git` - Git CLI (for cloning repositories)
- Code repositories must be accessible (public or with valid credentials)
- For JIRA discovery: `JIRA_AUTH_TOKEN` in `~/.env`
- For PR discovery: `GITHUB_TOKEN` or `GITLAB_TOKEN` in `~/.env`

## Notes

- Individual skills can still be called independently for advanced use cases
- Phase 2 can be run later using `/docs-technical-review-apply`
- Auto mode (`--auto`) skips all pause points
- The workflow preserves all Phase 1 auto-fixes even if Phase 2 is cancelled

## Related Commands

| Command | Purpose |
|---------|---------|
| `/docs-technical-review-validate` | Run Phase 1 only (validation and auto-fix) |
| `/docs-technical-review-apply` | Run Phase 2 only (interactive review) |
