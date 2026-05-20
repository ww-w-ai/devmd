---
devmd: schema
version: 0.1.0

database:
  type: ""                       # postgresql | sqlite | d1 | mongodb | ...
  orm: ""                        # prisma | drizzle | sqlalchemy | none

tables:
  - name: ""
    description: ""
    fields:
      - name: ""
        type: ""
        nullable: false
        default: null
        constraints: []          # pk | unique | fk:table.field | check:expr
    indexes:
      - fields: []
        type: ""                 # btree | unique | gin | ...
    relations:
      - type: ""                 # has_many | belongs_to | many_to_many
        target: ""
        through: ""              # join table if m2m

enums:
  - name: ""
    values: []

migration_rules:
  strategy: ""                   # sequential | timestamp
  naming: ""                     # e.g. "NNNN_description.sql"
  rollback_required: true
---

# SCHEMA.md

> Database tables, relations, indexes, enums, and migration rules.

## Entity-Relationship Overview

<!-- Describe main entities and their relationships. Reference @GLOSSARY.md#terms for naming. -->

## Tables

<!-- Expand frontmatter tables. One subsection per table. -->

### [table_name]

| Field | Type | Nullable | Default | Constraints |
|-------|------|----------|---------|-------------|
|       |      |          |         |             |

**Indexes:**
**Relations:**

## Enums

| Enum | Values | Used By |
|------|--------|---------|
|      |        |         |

## Migration Rules

<!-- Strategy, naming, rollback requirements. Reference @DEVOPS.md#ci-cd for migration pipeline. -->

## Cross-References

- Domain terms: @GLOSSARY.md
- API using this data: @API.md
- Error codes for DB errors: @ERRORS.md
