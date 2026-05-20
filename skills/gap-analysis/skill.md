---
name: devmd-gap-analysis
description: Compare DevMD files against actual source code. Measures coverage, accuracy, and consistency with deterministic counting and evidence-backed findings.
triggers:
  - devmd gap
  - devmd check
  - devmd verify
  - 갭 분석
  - 충돌 검사
user-invocable: true
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Agent
  - Write
  - AskUserQuestion
---

# DevMD Gap Analysis

Compares DevMD specification files against actual source code.

## Arguments

```
/devmd-gap-analysis                          # Analyze all DevMD files vs current directory
/devmd-gap-analysis SCHEMA.md                # Single file analysis
/devmd-gap-analysis SCHEMA.md --source ../myapp  # Explicit source path
/devmd-gap-analysis --consistency-only       # Cross-file consistency check only (no source needed)
```

## When to Use (and When Not To)

**Use:**
- DevMD files were written manually or edited after scan — verify they still match code
- Code changed since last scan — find what DevMD files are now stale
- Reviewing someone else's DevMD files against the actual project

**Don't use:**
- Immediately after `/devmd-scan` — that's self-grading. Scan has its own self-verification step.

## Phase 0: Setup

### 0a. Language detection + recon

Same as `/devmd-scan` Phase 0. Reuse if run in same session.

### 0b. Inventory DevMD files

```bash
# Find all DevMD files
for f in PRODUCT GLOSSARY BRAND ARCHITECTURE SCHEMA API ERRORS LOGGING \
         DESIGN UI SCREENS FLOWS SEO TESTING SECURITY INFRA CONFIG DEVOPS \
         OPERATIONS CHANGELOG CLAUDE AGENTS HARNESS RUNTIME LIFECYCLE; do
  [ -f "${DEVMD_PATH}/${f}.md" ] && echo "PRESENT: ${f}.md ($(wc -l < "${DEVMD_PATH}/${f}.md") lines)"
done
```

Only analyze files that exist. Do not flag missing files as gaps — that's a tier decision, not a gap.

## Phase 1: Deterministic Analysis (automated, reproducible)

For each DevMD file present, run the matching Count command from `skills/common/patterns.md` against the source code.

### Per-file counting

For each file, produce:

```
FILE: SCHEMA.md
  code_count: 23        # tables/models found in source (deterministic grep)
  doc_count: 18         # tables/models documented in SCHEMA.md (grep "^  - " or table rows)
  coverage: 78%         # doc_count / code_count
  missing_items:        # items in code but not in doc
    - audit_log         (src/models/audit-log.ts:5)
    - session           (src/models/session.ts:3)
    - migration_lock    (prisma/migrations/lock.ts:1)
    ...
```

### Counting rules per file

| File | What to count in code | What to count in doc |
|------|----------------------|---------------------|
| SCHEMA.md | Tables/models (ORM decorators, Prisma models, CREATE TABLE) | Table names in frontmatter `tables:` or body table rows |
| API.md | Route handlers (method decorators, router calls) | Endpoint entries in frontmatter or body sections |
| ARCHITECTURE.md | Top-level packages/modules (directories with entry points) | Packages listed in frontmatter `packages:` or body |
| ERRORS.md | Error classes/codes (extends Error, error constants) | Error codes listed in body |
| CONFIG.md | Env vars (process.env.*, os.environ) | Env vars listed in frontmatter or body |
| TESTING.md | Test files (*.spec.*, *.test.*, test_*) | Test suites/categories mentioned in body |
| UI.md | Page/screen components (pages/*.tsx, routes) | Pages listed in frontmatter `pages:` or body |
| INFRA.md | Infrastructure config files (Dockerfile, terraform, k8s) | Infrastructure components listed in body |
| SECURITY.md | Auth-related files (auth/*, guards/*, middleware/auth*) | Auth mechanisms listed in body |

### Files NOT deterministically countable

These files describe qualitative concepts. Skip counting, handle in Phase 2.

- PRODUCT.md, GLOSSARY.md, BRAND.md, FLOWS.md, SCREENS.md, DESIGN.md
- LOGGING.md, SEO.md, HARNESS.md, RUNTIME.md, LIFECYCLE.md
- DEVOPS.md, OPERATIONS.md, CHANGELOG.md, CLAUDE.md, AGENTS.md

## Phase 2: Evidence-Based Findings (LLM judgment, tagged)

For each DevMD file (including non-countable ones), an Agent reviews:

### Agent prompt

```
You are auditing {FILE_NAME} against source code for accuracy and completeness.

## DevMD file content
{READ the DevMD file}

## Source files to check
{List of relevant source files from Phase 0 recon + patterns.md Glob results}

## Task

Find issues in these categories:

### A. Accuracy errors — claims in the doc that contradict source code
For each: quote the doc claim, quote the source code, explain the contradiction.

### B. Staleness — things in the doc that no longer exist in code
For each: quote the doc reference, show that Grep finds nothing.

### C. Coverage gaps (non-countable files only) — important source patterns not captured
For each: cite source file:line, explain what's missing and why it matters.

## Output format (strict)

Return ONLY a JSON array. No markdown, no explanation.

[
  {
    "type": "accuracy|staleness|coverage",
    "severity": "high|medium|low",
    "file": "{FILE_NAME}",
    "doc_claim": "exact quote from DevMD file or null",
    "source_file": "path/to/file.ts",
    "source_line": 42,
    "source_snippet": "1-3 lines of code",
    "description": "what's wrong, in one sentence"
  }
]

If no issues found, return [].
Do NOT invent issues. If you're unsure, don't include it.
```

### Confidence filter

After Agent returns, discard any finding where:
- `source_file` does not exist (verify with Glob)
- `source_line` does not contain `source_snippet` (verify with Read)
- `doc_claim` is not found in the DevMD file (verify with Grep)

This prevents hallucinated findings.

## Phase 3: Cross-File Consistency

No source code needed. Compares DevMD files against each other.

### 3a. Cross-reference validation

```bash
# Extract all @FILE.md#section references
grep -rn "@[A-Z].*\.md" ${DEVMD_PATH}/*.md
```

For each reference:
1. Does the target file exist?
2. Does the target section (after `#`) exist as a heading in that file?

### 3b. Overlap contradiction detection

Check these specific pairs for semantic contradictions:

| Pair | What to compare |
|------|----------------|
| GLOSSARY.md vs SCHEMA.md | Term definitions vs table/field names and enum values |
| API.md vs SCHEMA.md | Endpoint field names vs table field names |
| UI.md vs FLOWS.md | Page names referenced in flows vs pages defined |
| ARCHITECTURE.md vs INFRA.md | Layer names, service names |
| ERRORS.md vs API.md | Error codes referenced in API vs defined in ERRORS |
| CONFIG.md vs INFRA.md | Env vars listed vs deployment config |

For each pair, an Agent reads both files and reports contradictions in the same JSON format as Phase 2 (type: "consistency").

## Phase 4: Scoring

### Deterministic score (from Phase 1)

```
deterministic_coverage = sum(doc_count) / sum(code_count) × 100
```

Only includes files that have countable items. Files with code_count = 0 are excluded.

### Finding score (from Phase 2 + 3)

```
high_findings = count of severity=high
medium_findings = count of severity=medium
low_findings = count of severity=low

finding_penalty = (high × 5) + (medium × 2) + (low × 1)
finding_score = max(0, 100 - finding_penalty)
```

### Consistency score (from Phase 3)

```
total_refs = count of @FILE.md references
broken_refs = count of references that don't resolve
contradictions = count of consistency findings

consistency_score = ((total_refs - broken_refs) / total_refs × 50) + (max(0, 50 - contradictions × 5))
```

### Overall

```
overall = deterministic_coverage × 0.4 + finding_score × 0.35 + consistency_score × 0.25
```

## Output

Write report to `{devmd_path}/gap-analysis.md`:

```markdown
# DevMD Gap Analysis Report

**Source**: {source_path}
**Date**: {ISO date}
**Overall Score**: {N}/100

## Scoring Breakdown

| Dimension | Score | Method | Details |
|-----------|-------|--------|---------|
| Coverage (deterministic) | {N}/100 | Grep counting | {doc_count}/{code_count} items across {N} files |
| Findings (evidence-based) | {N}/100 | LLM + verified | {high}H / {medium}M / {low}L findings |
| Consistency | {N}/100 | Cross-ref + overlap | {broken_refs} broken refs, {contradictions} contradictions |

## Deterministic Coverage

| File | In Code | In Doc | Coverage | Missing |
|------|---------|--------|----------|---------|
| SCHEMA.md | 23 | 18 | 78% | audit_log, session, migration_lock, ... |
| API.md | 45 | 42 | 93% | POST /webhooks, DELETE /tokens |
| ... | | | | |

## Verified Findings

### High Severity

#### ARCHITECTURE.md — incorrect package count
- **Doc claim**: "monorepo with 8 packages"
- **Source**: `packages/` contains 12 directories with package.json
- **Evidence**: `find packages -maxdepth 1 -name "package.json" | wc -l` → 12

### Medium Severity
...

### Low Severity
...

## Consistency Issues

### Broken Cross-References
- FLOWS.md:34 references `@UI.md#checkout-page` — section "checkout-page" not found in UI.md

### Contradictions
- GLOSSARY.md defines "Recipient" with 5 roles; SCHEMA.md:78 defines enum with 4 values
  - GLOSSARY.md:12: "Recipient can be: sender, signer, cc, viewer, approver"
  - SCHEMA.md:78: "role: enum(sender|signer|cc|viewer)"

## Recommendations (ordered by impact)

1. {Fix with highest coverage/accuracy improvement}
2. ...
```

## Single-File Mode

When invoked as `/devmd-gap-analysis SCHEMA.md`:
1. Run Phase 0 (recon)
2. Run Phase 1 counting for SCHEMA.md only
3. Run Phase 2 Agent for SCHEMA.md only
4. Skip Phase 3 (cross-file consistency needs multiple files)
5. Score using Phase 1 + 2 only (no consistency dimension)
6. Output abbreviated report

## Consistency-Only Mode

When invoked as `/devmd-gap-analysis --consistency-only`:
1. Skip Phase 0, 1, 2
2. Run Phase 3 only
3. Score consistency dimension only
4. Useful for checking DevMD files against each other without source code
