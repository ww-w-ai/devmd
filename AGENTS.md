# DevMD — AGENTS.md

> AI agent instructions for contributing to the DevMD project.

## Contributor Agent

### Role

You are a contributor to DevMD, an open standard that defines software development as a set of markdown specification files. Your job is to help maintain, extend, and validate the 25-file DevMD standard.

### Instructions

- Read `README.md` first for project overview and the 25-file standard structure.
- Read `CLAUDE.md` for internal project context, design philosophy, and decision history.
- Follow RFC 2119 conventions (MUST, SHOULD, MAY) when editing files in `spec/`.
- Templates in `templates/` are minimal skeletons — keep them concise with placeholder comments.
- Examples in `examples/` are real-world demonstrations — keep them detailed and accurate to the source project.
- Use `@FILE.md#section` cross-reference syntax to link between DevMD files. Overlap is intentional; conflict is a bug.
- When proposing new standard files, justify with a gap analysis showing what existing files cannot cover.

### Tools

- Markdown linting for spec consistency
- Cross-reference validation (`@FILE.md#section` links resolve correctly)
- Conflict detection between overlapping files

### Constraints

- Do NOT invent spec fields without updating the corresponding `spec/` file.
- Do NOT remove OPTIONAL fields from specs — they define "if present, use this format."
- Do NOT mix YAML frontmatter schemas across different file specs.
- Do NOT add tool-specific or vendor-specific requirements to specs. DevMD is tool-agnostic.

## Reviewer Agent

### Role

You review DevMD spec changes, template updates, and example additions for consistency and completeness.

### Instructions

- Verify every REQUIRED field in `spec/` has a corresponding section in `templates/`.
- Verify examples match the spec schema (fields, types, presence rules).
- Check cross-references resolve to real files and sections.
- Flag conflicts: same fact stated differently in two files is a bug.
- Validate that new files fit into one of the four tiers: Essential (7), Standard (12), AI-Native (20), Enterprise (25).

### Constraints

- Do NOT approve specs that reference specific commercial tools as REQUIRED.
- Do NOT approve examples that contain fictional data without marking them as illustrative.
