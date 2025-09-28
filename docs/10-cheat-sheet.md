# Chapter 10 – Cheat Sheet (Quick Reference)

[← Previous: Examples & Patterns in Action](09-examples-and-patterns.md) • [Back to index](index.md) • [Next chapter →](11-conclusion.md)

Print this page or keep it open during reviews. Each line matches the detailed chapters but fits into a glanceable table.

## 10.1 Database names

| Do this | Why | Example |
| --- | --- | --- |
| `<product>_<purpose>_<env>` in lower_snake_case | Ownership and environment are obvious | `project_service_prod` |
| Keep environment suffixes short (`dev`, `stg`, `prod`) | Scripts deploy to the right place | `analytics_stg` |
| Avoid camelCase or hyphens | Works consistently across OS and tooling | `identity_archive` |

## 10.2 Tables and columns

| Rule | Reason | Example |
| --- | --- | --- |
| Tables use singular nouns | Joins read naturally | `CREATE TABLE project (...);` |
| Primary key column is `id` | Every foreign key points to the same name | `id BIGINT UNSIGNED PRIMARY KEY` |
| Foreign keys end with `_id` | Relationships stay readable | `team_member_id BIGINT UNSIGNED` |
| Timestamps end with `_at` | Signals a point in time | `submitted_at DATETIME NOT NULL` |
| Soft deletes use `deleted_at` | `NULL` = active, value = archived | `deleted_at DATETIME NULL` |
| Amounts include units | Prevents ambiguity | `billable_amount_cents` |

## 10.3 Constraints and indexes

| Object | Prefix | Example name | Quick reminder |
| --- | --- | --- | --- |
| Primary key | `pk_` | `pk_time_entry` | Anchor for joins |
| Foreign key | `fk_` | `fk_time_entry__project_id__project` | Include table, column, reference |
| Unique constraint | `ux_` | `ux_project_member__project_id_team_member_id` | Shows protected business rule |
| Non-unique index | `ix_` | `ix_time_entry__project_id_submitted_at` | Mirrors query order |
| Check constraint | `ck_` | `ck_project__time_budget_minutes_nonneg` | Reads like a rule |
| Full-text index | `ft_` | `ft_project_note__title_body` | Easy to find search indexes |

## 10.4 Derived objects

| Object type | Prefix pattern | Example |
| --- | --- | --- |
| View | `v_<purpose>` | `v_project_recent_activity` |
| Stored procedure | `sp_<verb>_<object>` | `sp_archive_project` |
| Function | `fn_<result>` | `fn_project_logged_minutes` |
| Trigger | `trg_<table>__<timing>__<event>` | `trg_time_entry__before__insert` |

## 10.5 Migrations

| Guideline | Reminder |
| --- | --- |
| Filename starts with UTC timestamp | `20240718120000_add-project-status.sql` |
| One change per file | Easier reviews and rollbacks |
| Document rollback in comments or paired `_down` file | Saves time during incidents |

## 10.6 Quick checklist before merging

- [ ] Names use lower_snake_case and full words.
- [ ] Reserved words are avoided or extended with qualifiers.
- [ ] Booleans start with `is_` or `has_`.
- [ ] Amount columns include units.
- [ ] Timestamps use UTC and `_at` suffixes.
- [ ] Constraints, indexes, and derived objects follow their prefixes.
- [ ] Migration filenames start with a timestamp and describe one action.

If every box is checked, your change aligns with the convention.
