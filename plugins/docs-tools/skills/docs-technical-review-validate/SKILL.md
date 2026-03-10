---
name: docs-technical-review-validate
description: Validate documentation technical accuracy against code repositories. Detects removed commands, changed API signatures, stale code examples, renamed config keys, and moved file paths. Auto-fixes high-confidence issues (>=65%) and generates a report for manual review of the rest. Use this skill when asked to check if docs match the code, verify CLI examples still work, validate API references, find outdated commands or stale documentation, compare docs against a PR or JIRA ticket, or run a technical review. Also use when the user says things like "are these docs accurate" or "check the code examples".
argument-hint: --docs <source> [--docs <source>...] [--code URL] [--jira TICKET] [--pr URL] [--gdoc URL] [--fix] [--auto]
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Skill, WebFetch
---

# Technical Review Skill

Validate documentation for technical accuracy by comparing against source code repositories. Automatically fixes high-confidence issues and generates detailed reports for manual review.

## Purpose

This skill validates documentation against actual code to catch:
- **Removed components** - Commands, functions, or APIs documented but no longer in code
- **Syntax changes** - Command flags, function signatures, or API endpoints that changed
- **Conceptual inaccuracies** - Descriptions that don't match code behavior
- **Configuration errors** - Invalid keys, incorrect defaults, or wrong value types
- **Stale examples** - Code snippets that no longer compile or run

## Workflow Overview

This skill implements **Phase 1** of the technical review workflow. It validates docs, auto-fixes high-confidence issues (>=65%), and generates a report. Phase 2 (interactive apply of lower-confidence fixes) is handled by the `docs-technical-review-apply` skill.

Auto-fixable issues (>=65% confidence) include renamed command flags, updated function signatures, changed API endpoints, renamed config keys, and whitespace corrections.

## Input Modes

### Mode 1: Explicit Code Repositories

```bash
/docs-technical-review-validate --docs modules/ \
  --code https://github.com/org/repo1 \
  --code https://github.com/org/repo2 \
  --fix
```

Specify code repositories directly. Validates all documentation files against these repos.

### Mode 2: Auto-Discovery from JIRA

```bash
/docs-technical-review-validate --docs .claude_docs/drafts/rhaistrat-123/ \
  --jira RHAISTRAT-123 \
  --fix
```

**Discovery process**:
1. Fetch JIRA ticket using `jira-reader` skill
2. Extract linked PR/MR URLs from ticket
3. Parse PR/MR descriptions for repository references
4. Fall back to JIRA description for repo mentions
5. Extract repo URLs and default to `main` branch

### Mode 3: Auto-Discovery from PRs

```bash
/docs-technical-review-validate --docs modules/ \
  --pr https://github.com/org/repo/pull/456 \
  --pr https://gitlab.com/org/project/-/merge_requests/789 \
  --fix
```

### Mode 4: Auto-Discovery from Google Docs

```bash
/docs-technical-review-validate --docs modules/ \
  --gdoc https://docs.google.com/document/d/ABC123 \
  --fix
```

### Mode 5: Combined Discovery

```bash
/docs-technical-review-validate \
  --docs .claude_docs/drafts/rhaistrat-123/ \
  --jira RHAISTRAT-123 \
  --pr https://github.com/org/repo/pull/456 \
  --code https://github.com/org/other-repo \
  --fix
```

Combines all discovery methods, deduplicates repository URLs, validates against all sources.

### Mode 6: AsciiDoc Attribute Fallback

If no repos specified and auto-discovery finds nothing, searches for `:code-repo-url:` attributes in AsciiDoc files:

```asciidoc
:code-repo-url: https://github.com/org/repo
:code-repo-ref: v2.0.0
```

## Implementation Workflow

Follow this exact workflow when invoked:

### Step 1: Parse Arguments

```bash
# Arguments:
# --docs SOURCE       - Documentation to validate (required, repeatable)
# --code URL          - Code repository URL (repeatable)
# --ref BRANCH        - Git ref for previous --code (default: main)
# --jira TICKET-123   - JIRA ticket for auto-discovery
# --pr URL            - PR/MR URL for auto-discovery (repeatable)
# --gdoc URL          - Google Doc URL for auto-discovery
# --fix               - Enable auto-fixing (>=65% confidence)
# --auto              - Run both phases without pausing
# --dry-run           - Show what would be fixed without applying changes

REPOS=()
DOCS_SOURCES=()

while [[ $# -gt 0 ]]; do
  case "$1" in
    --docs) DOCS_SOURCES+=("$2"); shift 2 ;;
    --code) REPOS+=("$2"); shift 2 ;;
    --ref) REF="$2"; shift 2 ;;
    --jira) JIRA_TICKET="$2"; shift 2 ;;
    --pr) PR_URLS+=("$2"); shift 2 ;;
    --gdoc) GDOC_URL="$2"; shift 2 ;;
    --fix) FIX=true; shift ;;
    --auto) AUTO=true; shift ;;
    --dry-run) DRY_RUN=true; shift ;;
    *) shift ;;
  esac
done
```

### Step 1b: Resolve Docs Sources

Each `--docs` source is auto-detected and resolved:

| Source Type | Detection | Resolution |
|-------------|-----------|------------|
| Local file | Path exists as file | Use directly |
| Local directory | Path exists as directory | Glob for `*.adoc` and `*.md` files |
| Glob pattern | Contains `*` or `?` | Expand pattern to matching files |
| PR/MR URL | Matches GitHub/GitLab PR URL pattern | Fetch changed doc files via `git_review_api.py` |
| Google Doc URL | Matches `docs.google.com` | Read via `docs-read-gdoc` skill, save to temp file |
| Remote repo URL | Matches `https://github.com` or `https://gitlab.com` (non-PR) | Clone and glob for doc files |

Collect all resolved files into `DOCS_FILES` array for validation.

**Discovery priority** (first to last):
1. Explicit `--code` (highest priority - user knows what they want)
2. PR URLs (most specific - actual code being documented)
3. JIRA ticket (may have multiple PRs, use most recent)
4. Google Docs (may have outdated links)
5. AsciiDoc attributes (may be stale)

### Step 2: Validate Input

```bash
if [[ ${#DOCS_SOURCES[@]} -eq 0 ]]; then
  echo "ERROR: At least one --docs source is required"
  exit 1
fi

if [[ ${#REPOS[@]} -eq 0 ]]; then
  echo "ERROR: No code repositories specified or discovered"
  echo "Use --code, --jira, --pr, --gdoc, or add :code-repo-url: attribute to docs"
  exit 1
fi
```

### Step 3: Clone Code Repositories

```bash
CLONE_DIR="/tmp/tech-review"
mkdir -p "$CLONE_DIR"

for repo_url in "${REPOS[@]}"; do
  repo_name=$(basename "$repo_url" .git)
  clone_path="$CLONE_DIR/$repo_name"

  [[ -d "$clone_path" ]] && continue

  echo "Cloning $repo_url..."
  if ! git clone --depth 1 --branch "${REF:-main}" "$repo_url" "$clone_path" 2>/dev/null; then
    git clone --depth 1 "$repo_url" "$clone_path" 2>/dev/null || {
      echo "WARNING: Failed to clone $repo_url - skipping"; continue
    }
  fi
done
```

### Step 4: Extract Technical References

Run the `extract_tech_references.rb` script from this skill's `scripts/` directory. It parses AsciiDoc and Markdown files to identify commands, code blocks, APIs/functions, configuration keys, and file paths, outputting structured JSON.

```bash
SKILL_SCRIPTS="$(dirname "$0")/../skills/docs-technical-review-validate/scripts"
REFS_JSON="/tmp/tech-review-refs.json"

ruby "$SKILL_SCRIPTS/extract_tech_references.rb" \
  "${DOCS_FILES[@]}" \
  --output "$REFS_JSON"
```

If the script path cannot be resolved, search for `extract_tech_references.rb` using Glob to locate it.

### Step 5: Search and Validate References Against Code

Run the `search_tech_references.rb` script from this skill's `scripts/` directory. Pass all cloned repo paths as positional arguments after the refs JSON:

```bash
ruby "$SKILL_SCRIPTS/search_tech_references.rb" \
  "$REFS_JSON" \
  "$CLONE_DIR/repo1" "$CLONE_DIR/repo2" \
  --output /tmp/tech-review-search.json
```

The script searches each repository for matches against every extracted reference.

**Search results by category**:

| Category | Result Fields | What It Finds |
|----------|---------------|---------------|
| **Commands** | `found`, `found_path`, `flags_missing`, `similar_commands`, `git_log_mentions` | Whether command binary/script exists, which flags are missing, git history for renames |
| **Code Blocks** | `match_type` (exact/partial/none), `match_ratio`, `matched_file`, `actual_code` | Exact or fuzzy content matches in source files of the matching language |
| **APIs/Functions** | `definition_found`, `actual_signature`, `type` (function/class/endpoint) | Function/class/endpoint definitions and whether signatures match |
| **Configuration** | `key_found`, `found_in_file`, `git_log_mentions` | Whether config keys exist in schema/example files, git history for renames |
| **File Paths** | `exists`, `moved_to`, `similar_files` | Whether referenced paths exist, fuzzy matches if file was moved |

**Interpreting results and assigning confidence**:

The search results are raw evidence — use your judgment to interpret them and assign confidence scores. The key principle: confidence reflects how certain you are about both the problem and the fix, not just the problem.

- **Exact matches** with only syntax/formatting differences → high confidence (>=85%), because the fix is obvious
- **Git log evidence** of renames or deprecation → medium-high (70-90%), because the history confirms intentional change
- **Partial matches** or similar-but-different results → medium (50-64%), because the right fix is ambiguous
- **No matches at all** → low (<50%), because it could be removal OR wrong repo OR the reference is fine but lives elsewhere
- **Context matters**: a missing flag in a command that otherwise exists is higher confidence than a completely missing command, because you know the command is right and only the flag changed

Cross-reference multiple signals (search results + git history + related files) before finalizing confidence.

### Step 6: Perform Whole-Repo Scanning

For each flagged issue (removed command, changed API, etc.), search all `.adoc` and `.md` files for additional occurrences of the same pattern. Record every file and line where the pattern appears so the report captures the full blast radius.

### Step 7: Calculate Confidence Scores

| Scenario | Confidence | Action |
|----------|------------|--------|
| Whitespace/formatting only | 100% | Auto-fix |
| Exact match with syntax change | 95% | Auto-fix |
| Flag renamed (git log evidence) | 90% | Auto-fix |
| Function signature changed (exact function) | 85% | Auto-fix |
| Import updated (package renamed) | 80% | Auto-fix |
| Config key renamed (migration doc) | 75% | Auto-fix |
| Path changed (file moved) | 65% | Auto-fix |
| Command replaced (similar found) | 60% | Manual review |
| Endpoint path changed | 55% | Manual review |
| Config structure changed | 50% | Manual review |
| Component removed (no replacement) | 45% | Manual review |
| Semantic changes | 40% | Manual review |
| No match found | 35% | Manual review |

**Threshold**: >=65% = Auto-fix, <65% = Manual review

### Step 8: Apply Auto-Fixes (if --fix enabled)

For each issue with confidence >=65%, apply the fix using the Edit tool. Track each fix applied and its before/after text for the report.

### Step 9: Generate Markdown Report

Generate comprehensive markdown report at `.claude_docs/technical-review-report.md` with these sections:

1. **Header** - docs sources, timestamp, repo count
2. **Summary table** - per-category counts (validated, issues, auto-fixed, manual review)
3. **Code Repositories** - URL, ref, source, clone path for each repo
4. **Issues Auto-Fixed** - each with ID (`AF-N`), location, issue, evidence, and diff
5. **Issues Requiring Manual Review** - each with ID (`MR-N`), location, severity, issue, evidence, suggested diff, reasoning, and agentic action command
6. **Whole-Repo Scan Results** - grouped by issue type, listing all files/lines affected
7. **Next Steps** - review auto-fixes, run Phase 2, address whole-repo findings, run Vale, test examples
8. **Agentic Follow-Up Command** - example `/docs-technical-review-apply` invocations

### Step 9b: Generate JSON Sidecar Report

Write `.claude_docs/technical-review-report.json` alongside the markdown report. This structured output enables programmatic consumption by Phase 2 and CI/CD pipelines.

**Format**: Array of issue objects:

```json
[
  {
    "id": "AF-1",
    "file": "modules/proc-install.adoc",
    "line": 42,
    "category": "commands",
    "confidence": 90,
    "old_text": "--enable-feature",
    "new_text": "--feature-enable",
    "evidence": "Flag renamed in commit abc123",
    "description": "Command flag renamed in v2.0"
  },
  {
    "id": "MR-1",
    "file": "modules/ref-api.adoc",
    "line": 18,
    "category": "apis",
    "confidence": 55,
    "old_text": "getUserProfile(id)",
    "new_text": "getProfile(userId)",
    "evidence": "Function signature changed, similar function found",
    "description": "API function signature may have changed"
  }
]
```

**Fields**:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Issue ID (`AF-N` for auto-fixed, `MR-N` for manual review) |
| `file` | string | Path to the documentation file |
| `line` | number | Line number in the file |
| `category` | string | One of: `commands`, `code_blocks`, `apis`, `configs`, `file_paths`, `conceptual` |
| `confidence` | number | Confidence score (0-100) |
| `old_text` | string | Original text in the documentation (enables content-based matching in Phase 2) |
| `new_text` | string | Suggested or applied replacement text |
| `evidence` | string | What was found in the code repository |
| `description` | string | Human-readable description of the issue |

The `old_text` field is critical for Phase 2: it enables content-based matching to locate issues even if line numbers have shifted due to earlier edits.

### Step 10: Output Summary

```bash
echo "Technical review complete!"
echo "  Total issues: $TOTAL_ISSUES"
echo "  Auto-fixed: $AUTO_FIXED (>=65% confidence)"
echo "  Manual review: $MANUAL_REVIEW (<65% confidence)"
echo "  Report: .claude_docs/technical-review-report.md"
echo "  JSON:   .claude_docs/technical-review-report.json"
```

## Validation Categories Summary

| Category | What to Check | Auto-Fix Examples | Manual Review Examples |
|----------|---------------|-------------------|------------------------|
| **Commands** | Existence, flags, syntax | Flag renamed, syntax updated | Command removed |
| **Code Blocks** | Match source, syntax correct | Import updated, formatting | Semantic changes |
| **APIs/Functions** | Exist, signatures match | Signature updated | Component removed |
| **Configuration** | Keys exist, types valid | Key renamed | Structure changed |
| **File Paths** | Files exist | Path updated | File deleted |
| **Conceptual** | Descriptions match behavior | Terminology updated | Architecture changes |

## Error Handling

- **No repos discovered**: Exit with error, show discovery methods
- **Clone failures**: Warn and skip repo, continue with others
- **No references found**: Exit gracefully, report summary
- **Invalid AsciiDoc**: Report errors but continue processing
- **Git command failures**: Catch and report, do not crash

## Integration with Other Skills

This skill can be called by:
- `docs-technical-review` command (orchestrates Phase 1 + Phase 2)
- `docs-workflow` command (as optional Stage 5)
- CI/CD pipelines for automated validation

## Output Files

| File | Description |
|------|-------------|
| `.claude_docs/technical-review-report.md` | Comprehensive validation report (Markdown) |
| `.claude_docs/technical-review-report.json` | Structured issue data for Phase 2 and CI/CD (JSON) |
| `/tmp/tech-review-refs.json` | Extracted technical references |
| `/tmp/tech-review-search.json` | Search results from code repositories |
| `/tmp/tech-review/<repo-name>/` | Cloned code repositories |

## Example Invocations

```bash
# Explicit code repo
/docs-technical-review-validate --docs modules/ --code https://github.com/org/repo --fix

# JIRA auto-discovery
/docs-technical-review-validate --docs .claude_docs/drafts/rhaistrat-123/ --jira RHAISTRAT-123 --fix

# Multiple docs sources and combined discovery
/docs-technical-review-validate \
  --docs modules/proc-install.adoc --docs modules/ref-api.adoc \
  --jira RHAISTRAT-123 --code https://github.com/org/extra-repo --fix

# PR as docs source (validate changed files from PR against code repo)
/docs-technical-review-validate \
  --docs https://github.com/org/docs-repo/pull/100 \
  --code https://github.com/org/code-repo --fix

# Google Doc discovery with dry-run
/docs-technical-review-validate --docs modules/ \
  --gdoc https://docs.google.com/document/d/ABC123 --dry-run
```

## Best Practices

1. **Always run Phase 1 first** (`--fix`) to handle high-confidence issues
2. **Review the report** before running Phase 2 (agentic apply)
3. **Test critical examples** manually after auto-fixes
4. **Run Vale** after technical review to catch style issues
5. **Commit auto-fixes separately** from manual fixes
6. **Re-run after major code changes** to catch drift

## Troubleshooting

| Issue | Solution |
|-------|----------|
| No repos discovered | Add `--code` or `:code-repo-url:` attribute |
| Clone failures | Check network, auth tokens, repo URL |
| Too many false positives | Increase confidence threshold in code |
| Missing valid issues | Decrease confidence threshold |
| Slow performance | Reduce scope (target specific files with `--docs`) |

