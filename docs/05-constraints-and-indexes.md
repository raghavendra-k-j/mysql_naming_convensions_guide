# Chapter 5 – Constraints & Indexes

[← Previous: Table & Column Naming](04-table-and-column-naming.md) • [Back to index](index.md) • [Next chapter →](06-views-routines-triggers.md)

Constraint and index names show up in error messages, migration logs, and monitoring dashboards. When the names follow a clear pattern, you can diagnose issues in seconds. This chapter follows the W3Schools flow: learn the rule, see why it matters, and copy an example.

## 5.1 Always name constraints and indexes yourself

**Learn the rule.** Never rely on auto-generated names like `project_ibfk_1`. Define every name explicitly.

**Why it works.**

- Error messages instantly tell you what failed.
- Schema diffs stay clean across environments.
- On-call responders can fix issues without opening the database diagram.

**Follow the steps.**

1. Pick a prefix per object type (`pk_`, `fk_`, `ux_`, `ix_`, `ck_`, `ft_`).
2. Include the table name and important columns.
3. Separate sections with double underscores `__` for readability.
4. Add the name when you create or alter the constraint.

```sql
ALTER TABLE project_member
    ADD CONSTRAINT fk_project_member__project_id__project
        FOREIGN KEY (project_id) REFERENCES project (id);
```

## 5.2 Primary keys → `pk_<table>`

**Learn the rule.** Name each primary key `pk_<table>`.

**Why it works.**

- Error logs and monitoring charts reference the key clearly.
- Tools such as Debezium and ORMs detect your intended primary key without guesswork.
- You avoid random renames when schemas are dumped and reloaded.

**Follow the steps.**

1. Declare the constraint inside the `CREATE TABLE` statement.
2. Keep the name stable even if you swap the underlying column type.

```sql
CREATE TABLE time_entry (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    CONSTRAINT pk_time_entry PRIMARY KEY (id)
);
```

## 5.3 Foreign keys → `fk_<table>__<column>__<ref_table>`

**Learn the rule.** Combine the referencing table, the foreign key column(s), and the referenced table.

**Why it works.**

- The error message spells out the entire relationship.
- Reviewers catch mismatched types or target tables quickly.
- Automation can check that every `_id` column owns a constraint.

**Follow the steps.**

1. Use the singular table names.
2. Join multiple columns with `_` inside the middle section (`assignment_id__sequence_number`).
3. Reference the target table’s primary key.

```sql
ALTER TABLE time_entry
    ADD CONSTRAINT fk_time_entry__project_id__project
        FOREIGN KEY (project_id) REFERENCES project (id);
```

## 5.4 Unique constraints → `ux_<table>__<columns>`

**Learn the rule.** Describe which columns must stay unique.

**Why it works.**

- Business rules stay visible to everyone.
- Dropping or recreating the constraint in migrations is safe because the name is predictable.
- Analytics teams can answer "what keeps this unique?" instantly.

**Follow the steps.**

1. Use the singular table name.
2. List the columns in order, separated by underscores.
3. Reuse the same name for a supporting index if you create one manually.

```sql
ALTER TABLE project_member
    ADD CONSTRAINT ux_project_member__project_id_team_member_id
        UNIQUE (project_id, team_member_id);
```

## 5.5 Non-unique indexes → `ix_<table>__<columns>`

**Learn the rule.** Explain the query pattern the index supports.

**Why it works.**

- Prevents duplicate indexes with the same columns.
- Helps performance reviews decide whether the index is still needed.
- Signals sort order or coverage when appended (`_created_at_desc`, `_covering`).

**Follow the steps.**

1. Use the singular table name.
2. List columns in the order used by your query.
3. Append suffixes such as `_desc` if the index enforces descending order.

```sql
CREATE INDEX ix_time_entry__project_id_logged_minutes
    ON time_entry (project_id, logged_minutes);
```

## 5.6 Check constraints → `ck_<table>__<rule>`

**Learn the rule.** Summarise the rule inside the name.

**Why it works.**

- Error messages are self-explanatory.
- Reviewers can confirm the expression matches the intent.
- Prevents duplicate constraints that enforce the same rule.

**Follow the steps.**

1. Write a short phrase describing the rule (`minutes_nonneg`).
2. Keep the SQL expression in sync with the name.
3. Update the existing constraint instead of adding another one when requirements change.

```sql
ALTER TABLE time_entry
    ADD CONSTRAINT ck_time_entry__logged_minutes_nonneg
        CHECK (logged_minutes >= 0);
```

## 5.7 Full-text indexes → `ft_<table>__<columns>`

**Learn the rule.** Label text-search structures so nobody confuses them with normal indexes.

**Why it works.**

- Search teams and DBAs can find all full-text surfaces quickly.
- Avoids accidental drops during migrations.
- Makes rebuild scripts straightforward.

**Follow the steps.**

1. Use the singular table name.
2. List the searchable columns joined by underscores.
3. Add a locale or configuration suffix when relevant (`__en_gb`).

```sql
CREATE FULLTEXT INDEX ft_project_note__title_body
    ON project_note (title, body);
```

**Quick recap checklist**

- [ ] Did you choose the prefix based on the object type?
- [ ] Does the name include the table and columns or rule?
- [ ] Are sections separated with double underscores for readability?
- [ ] Will the name still make sense in an error message at 3 a.m.?

If you can answer "yes" to each question, your constraints and indexes are ready for production.
