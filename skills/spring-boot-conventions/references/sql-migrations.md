# SQL Style and Migrations

## Contents

- [Scope](#scope)
- [CREATE TABLE layout](#create-table-layout)
- [Index and constraint names](#index-and-constraint-names)
- [Liquibase files and changeset IDs](#liquibase-files-and-changeset-ids)
- [Applied history and subsequent changes](#applied-history-and-subsequent-changes)

## Scope

Check the actual DB dialect, schema, naming, and adopted formatter. These layout rules are defaults. Liquibase rules apply only to projects using Liquibase; do not replace another migration tool.

Never reformat already-applied migrations for style. Types, lengths, defaults, and constraints come from business contracts and the actual DB, not the example below.

## CREATE TABLE layout

- Put `(` on the line after `CREATE TABLE <name>`; close with `);`.
- Use one column per line, indented four spaces.
- Within each table, use spaces to align the starts of **column name, type, and attributes** against the longest name/type. Do not use tabs for alignment.
- Put `NOT NULL`, `DEFAULT`, and column constraints in the attribute area. Do not pad a column without attributes before its comma.
- Keep single-column constraints with their columns; group multicolumn constraints below the column list. Wrap complex conditions by logical unit.

Illustrative PostgreSQL DDL:

```sql
CREATE TABLE catalog_item
(
    id                uuid         CONSTRAINT pk_catalog_item PRIMARY KEY,
    version           bigint       NOT NULL DEFAULT 0,
    name              text         NOT NULL,
    sku               varchar(64)  CONSTRAINT uq_catalog_item_sku UNIQUE,
    quantity          bigint       NOT NULL,
    reserved_quantity bigint       NOT NULL DEFAULT 0,
    created_at        timestamptz  NOT NULL,
    updated_at        timestamptz  NOT NULL,
    CONSTRAINT ck_catalog_item_quantity CHECK (
        quantity >= 0
        AND reserved_quantity >= 0
        AND reserved_quantity <= quantity
    )
);

CREATE INDEX idx_catalog_item_created_at ON catalog_item (created_at);
```

Indent a wrapped column constraint one additional level. Alignment must not change column order, parentheses, operators, or expression semantics. This is SQL column alignment, not Java argument padding. Do not apply an unadopted SQL formatter to override it.

## Index and constraint names

Without an existing rule, use explicit `snake_case` names with these prefixes:

| Object | Prefix | Example |
|---|---|---|
| Ordinary index | `idx_` | `idx_catalog_item_created_at` |
| Primary key | `pk_` | `pk_catalog_item` |
| Foreign key | `fk_` | `fk_order_item_order_id` |
| Check constraint | `ck_` | `ck_catalog_item_quantity` |
| Unique constraint/index | `uq_` | `uq_catalog_item_sku` |

Use `uq_` for unique indexes too. Include enough column/predicate meaning to distinguish composite or partial indexes, subject to DB identifier limits and existing abbreviations. Do not rename existing DB objects or add unnecessary indexes just to apply a naming convention.

## Liquibase files and changeset IDs

Keep the project's filename pattern, sequence width, date format, and separators.

| Pattern | Examples |
|---|---|
| Sequential | `000001-create_catalog_item.sql`, `000002-add_item_description.sql` |
| Date plus sequence | `20261030_001-create_catalog_item.sql`, `20261030_002-add_item_description.sql` |

These examples are alternatives, not mandatory names. Choose one consistent pattern for a new project; do not convert existing history to another pattern.

Formatted SQL begins with its header and gives each changeset an `author:id`:

```sql
--liquibase formatted sql

--changeset author:000001_001-create_catalog_item
```

Replace `author` with the project's actual author notation. Follow its changeset-ID scheme, description format, and mapping to filename/internal sequence. A date-based filename does not force a date-based ID. A new, unapplied file may contain several changesets; distinguish file sequence from in-file sequence.

Preserve resource locations and the root changelog format. For a project using explicit YAML includes:

```yaml
databaseChangeLog:
  - include:
      file: changes/000001-create_catalog_item.sql
      relativeToChangelogFile: true
  - include:
      file: changes/000002-add_item_description.sql
      relativeToChangelogFile: true
```

A changeset ID is not an execution-order instruction. With explicit includes, order dependencies using include order and changeset placement within files. For other include/discovery mechanisms, inspect their actual ordering rules.

## Applied history and subsequent changes

- Do not change an applied file's name, path, contents, changeset IDs, or authors. This includes whitespace-only cleanup.
- Do not append new changesets to an applied file, even for the same table. Create a subsequent file under the existing naming scheme.
- With explicit includes, preserve earlier order and add the new file in the appropriate subsequent position.
- Consider both clean-database installation and incremental upgrade. Earlier files must not depend on objects introduced later.

This immutability policy is a **skill convention**, not a claim that Liquibase cannot execute a newly appended changeset. Changeset identity and execution semantics are framework concerns; see [configuration and Liquibase sources](source-notes.md#configuration-and-liquibase). Respect project-approved migration exceptions rather than silently rewriting history.
