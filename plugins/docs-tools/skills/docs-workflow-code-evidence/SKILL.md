---
name: docs-workflow-code-evidence
description: Retrieve code evidence from a source repository to ground documentation in actual implementation. Indexes the codebase using AST chunking and hybrid search, then retrieves relevant code snippets for each topic in the documentation plan. Uses two-pass retrieval — source-scoped for API accuracy, unfiltered for README/narrative context. Requires vibe2doc to be available locally.
argument-hint: <ticket> --base-path <path> --repo <code-repo-path> [--reindex] [--limit N]
allowed-tools: Read, Write, Glob, Grep, Bash
---

# Code Evidence Retrieval Step

Step skill for the docs-orchestrator pipeline. Follows the step skill contract: **parse args → run tool → write output**.

This skill bridges vibe2doc's code analysis capabilities into the docs-orchestrator workflow. It indexes a source code repository using AST chunking and hybrid search (BM25 + vector), then retrieves code snippets relevant to the documentation plan topics. The writer step can then use this evidence to ground its output in actual implementation details.

## Prerequisites

- **vibe2doc** must be cloned locally with its virtual environment set up
- Set the `VIBE2DOC_PATH` environment variable to the vibe2doc root directory, or pass `--vibe2doc-path` as an argument
- The vibe2doc virtual environment must have dependencies installed (`pymilvus`, `sentence-transformers`, `rank-bm25`, `tree-sitter`)

## Arguments

- `$1` — JIRA ticket ID (required)
- `--base-path <path>` — Base output path (e.g., `.claude/docs/proj-123`)
- `--repo <path>` — Path to the source code repository to index and search
- `--reindex` — Force re-indexing even if a cached index exists
- `--limit <N>` — Max results per topic (default: 5)

## Input

```
<base-path>/planning/plan.md
```

## Output

```
<base-path>/code-evidence/evidence.json
<base-path>/code-evidence/summary.md
```

## Execution

### 1. Parse arguments

Extract the ticket ID, `--base-path`, `--repo`, and optional flags from the args string.

Set the paths:

```bash
PLAN_FILE="${BASE_PATH}/planning/plan.md"
OUTPUT_DIR="${BASE_PATH}/code-evidence"
EVIDENCE_FILE="${OUTPUT_DIR}/evidence.json"
SUMMARY_FILE="${OUTPUT_DIR}/summary.md"
VIBE2DOC="${VIBE2DOC_PATH:-$HOME/Documents/PROJECTS/vibe2doc_withAgents}"
mkdir -p "$OUTPUT_DIR"
```

Validate:
- `--repo` is required. If missing, STOP and ask the user.
- Verify `$PLAN_FILE` exists. If not, STOP with error: "Planning step must complete before code-evidence."
- Verify `$VIBE2DOC/src/claude_context/skills/evidence_retrieval.py` exists. If not, STOP with error: "vibe2doc not found at $VIBE2DOC. Set VIBE2DOC_PATH or pass --vibe2doc-path."

### 2. Detect source directories

Before running queries, identify the repository's source code directories for the filtered pass. Look for common patterns:

- `src/`, `lib/`, `pkg/`, `cmd/`, `internal/`, `app/`
- Language-specific: `src/<project_name>/` (Python), `src/main/` (Java/Kotlin)

If a PR URL is available in the requirements step output, extract changed file paths and derive their parent directories as the filter scope.

Store the detected source paths for use in step 3.

### 3. Extract topics from the plan

Read `$PLAN_FILE` and extract the key topics to search for. Look for:

- Module names and file paths mentioned in the plan
- Section headings that describe features or components
- JTBD statements or user goals that reference specific functionality
- API or CLI references

Produce a list of 5-15 natural language search queries that cover the plan's scope. Each query should be specific enough to retrieve relevant code (e.g., "authentication middleware implementation" not "auth").

### 4. Run two-pass evidence retrieval for each topic

For each search query, run vibe2doc's evidence retrieval **twice** to capture both accurate source code and narrative context:

**Pass 1 — Source-scoped** (API accuracy):

```bash
cd "$VIBE2DOC" && source venv_langraph/bin/activate && \
PYTHONPATH=src python -m claude_context.skills.evidence_retrieval \
  --repo <REPO_PATH> \
  --query "<search_query>" \
  --limit <LIMIT> \
  --filter-paths <SOURCE_DIRS>
```

Where `<SOURCE_DIRS>` is the comma-separated list of source directories detected in step 2 (e.g., `src/speculators`).

This pass returns function signatures, class definitions, and implementation details scoped to actual source code.

**Pass 2 — Unfiltered** (narrative context):

```bash
cd "$VIBE2DOC" && source venv_langraph/bin/activate && \
PYTHONPATH=src python -m claude_context.skills.evidence_retrieval \
  --repo <REPO_PATH> \
  --query "<search_query>" \
  --limit <LIMIT>
```

This pass picks up READMEs, documentation, examples, and configuration files that provide the "why", installation steps, quickstart patterns, and architectural context.

**Note on indexing**: The index is built once on the first query and cached at `{repo}/.vibe2doc/index.db`. Both passes and all subsequent queries reuse the cached index. The second pass adds ~30-200ms per query, not a full re-index.

If `--reindex` is specified, add it to the **first** query only. Subsequent queries reuse the freshly built index.

Collect all results into a combined evidence structure:

```json
{
  "ticket": "<TICKET>",
  "repo_path": "<REPO_PATH>",
  "topics": [
    {
      "query": "authentication middleware implementation",
      "source_results": [ ... ],
      "context_results": [ ... ]
    }
  ],
  "index_info": { ... }
}
```

Write this to `$EVIDENCE_FILE`.

### 5. Generate evidence summary

Create a human-readable markdown summary at `$SUMMARY_FILE` with:

```markdown
# Code Evidence Summary

**Ticket:** <TICKET>
**Repository:** <REPO_PATH>
**Topics searched:** <N>
**Total code snippets found:** <N> (source: <N>, context: <N>)

## Topics

### 1. <query>

**Source code:**
- **<file_path>:<start_line>-<end_line>** — `<function_name>` (<chunk_type>)
  Score: <combined_score>
- ...

**Context (READMEs, docs, examples):**
- **<file_path>:<start_line>-<end_line>** — `<section_name>` (<chunk_type>)
  Score: <combined_score>
- ...

### 2. <query>
- ...
```

This summary is for human review. The JSON file is what downstream steps consume.

### 6. Verify output

After completion, verify that both `$EVIDENCE_FILE` and `$SUMMARY_FILE` exist.

## How downstream steps use the evidence

The **writing step** can reference the evidence to ground documentation in actual code:

> The code evidence at `<base-path>/code-evidence/evidence.json` contains two types of evidence per topic:
> - **`source_results`**: Accurate function signatures, parameter types, class structure from source code. Use these for API references, code examples, and technical accuracy.
> - **`context_results`**: README content, documentation, examples. Use these for narrative flow, installation instructions, quickstart guides, and architectural context.
>
> Prefer source_results for "what the code does" and context_results for "why and how to use it."

The **technical review step** can use it to verify claims:

> Cross-reference documentation claims against the code evidence at `<base-path>/code-evidence/evidence.json`. Use `source_results` to verify function signatures, parameters, and return types. Flag any claims that contradict the retrieved source code.

## Notes

- First run on a repo takes a few seconds to a few minutes depending on repo size (AST chunking + embeddings)
- Subsequent runs reuse the cached index at `{repo}/.vibe2doc/index.db`
- Use `--reindex` after significant code changes
- The index is deterministic — same code produces the same index
- Evidence retrieval uses hybrid search: BM25 for exact keyword matches + vector search for semantic similarity
- Default index exclusions skip `archive/`, `vendor/`, `node_modules/`, `docs/generated/`, `.vibe2doc/`, and other non-source directories
- The two-pass approach adds negligible overhead (~30-200ms per query) since both passes reuse the same cached index
