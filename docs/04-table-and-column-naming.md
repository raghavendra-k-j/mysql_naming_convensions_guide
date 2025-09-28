# Chapter 4 – Table & Column Naming

[← Previous: Database-Level Conventions](03-database-level-conventions.md) • [Back to index](index.md) • [Next chapter →](05-constraints-and-indexes.md)

Tables and columns are where the naming rules show up the most. Each section below gives you a step-by-step recipe with a working example from our sample project-tracking schema.

## 4.1 Give every table a surrogate primary key called `id`

**Learn the rule.** Use a single-column surrogate key named `id`. Let MySQL populate it automatically (`AUTO_INCREMENT` or UUIDs).

**Why it works.**

- Surrogate keys stay stable even when business identifiers change.
- Joins become predictable because every foreign key targets `<table>.id`.
- ORMs and migration tools expect this pattern and behave well when they find it.

**Follow the steps.**

1. Add an `id` column to every table (`BIGINT UNSIGNED` for numeric keys or `CHAR(36)` for UUIDs).
2. Mark it `PRIMARY KEY` and `NOT NULL`.
3. Keep business identifiers (like `project_code`) as separate unique columns.
4. Use the same approach in every table to build muscle memory.

```sql
CREATE TABLE project (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(160) NOT NULL,
    project_code CHAR(12) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY ux_project_project_code (project_code)
);
```

## 4.2 Name foreign keys after the referenced table + `_id`

**Learn the rule.** For a foreign key that points to `team_member.id`, name the column `team_member_id`.

**Why it works.**

- Relationships are obvious without opening schema diagrams.
- Query builders can infer joins automatically.
- Reviewers catch mistakes quickly because the column names match table names.

**Follow the steps.**

1. Take the referenced table in singular form (`team_member`).
2. Append `_id` and copy the same data type as the primary key.
3. Add an index to keep joins fast.
4. Create the foreign key constraint using the naming style from Chapter 5.

```sql
CREATE TABLE project_member (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    project_id BIGINT UNSIGNED NOT NULL,
    team_member_id BIGINT UNSIGNED NOT NULL,
    role_code VARCHAR(32) NOT NULL,
    PRIMARY KEY (id),
    KEY ix_project_member_project_id (project_id),
    KEY ix_project_member_team_member_id (team_member_id)
);
```

## 4.3 Standardise audit timestamps

**Learn the rule.** Add `created_at` and `updated_at` to any table that changes over time. Store UTC timestamps.

**Why it works.**

- You can answer “when did this change?” in seconds.
- Replication and change-data-capture tools rely on timestamps.
- Support teams can investigate issues without digging through logs.

**Follow the steps.**

1. Declare both columns as `DATETIME(6)` or `TIMESTAMP(6)` with `NOT NULL`.
2. Default `created_at` to `CURRENT_TIMESTAMP(6)`.
3. Default `updated_at` to `CURRENT_TIMESTAMP(6)` and add `ON UPDATE CURRENT_TIMESTAMP(6)`.
4. Keep everything in UTC to avoid daylight-saving surprises.

```sql
ALTER TABLE project
    ADD COLUMN created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    ADD COLUMN updated_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        ON UPDATE CURRENT_TIMESTAMP(6);
```

## 4.4 Use `deleted_at` for soft deletes

**Learn the rule.** Instead of a boolean `is_deleted`, store the deletion moment in a nullable `deleted_at` column.

**Why it works.**

- You capture the exact time a record was archived.
- Restoring data is a matter of clearing the timestamp.
- Analytics can filter `deleted_at IS NULL` or compare against a date range.

**Follow the steps.**

1. Add `deleted_at DATETIME(6) NULL` to tables that need soft deletes.
2. Treat any non-null value as deleted.
3. Index the column if you filter active vs. deleted rows often.
4. Optionally create a view that hides deleted rows from application queries.

```sql
ALTER TABLE project_member
    ADD COLUMN deleted_at DATETIME(6) NULL;
```

## 4.5 Preface booleans with `is_` or `has_`

**Learn the rule.** Name boolean columns so they read like questions: `is_active`, `has_time_budget`.

**Why it works.**

- Queries and logs read like plain English.
- Reviewers can spot misuse immediately.
- Prevents confusion with similarly named foreign keys.

**Follow the steps.**

1. Start state booleans with `is_` and possession booleans with `has_`.
2. Use `TINYINT(1) NOT NULL` with an explicit default (`0` or `1`).
3. Avoid storing more than two states in a boolean column—switch to enums when needed.

```sql
ALTER TABLE project
    ADD COLUMN is_active TINYINT(1) NOT NULL DEFAULT 1,
    ADD COLUMN has_time_budget TINYINT(1) NOT NULL DEFAULT 0;
```

## 4.6 Make units explicit in column names

**Learn the rule.** Include the measurement unit in the column name, especially for money and quantities.

**Why it works.**

- Prevents accidental currency or unit mix-ups.
- Keeps exported data self-explanatory.
- Makes refactoring easier because the scale is obvious.

**Follow the steps.**

1. Append the unit or scale: `_minutes`, `_cents`, `_usd`, `_grams`.
2. Prefer integers for currency (`_cents`) to avoid rounding errors.
3. Document conversion rules in your service README.

```sql
CREATE TABLE time_entry (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    project_id BIGINT UNSIGNED NOT NULL,
    team_member_id BIGINT UNSIGNED NOT NULL,
    logged_minutes INT UNSIGNED NOT NULL,
    billable_amount_cents INT UNSIGNED NOT NULL,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        ON UPDATE CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id)
);
```

## 4.7 Use descriptive enums or lookup tables

**Learn the rule.** When a column can only hold a limited set of values, either use a lookup table or a descriptive enum name.

**Why it works.**

- The allowed values stay obvious.
- Business users can map codes to meanings without guesswork.
- Future changes (adding a new status) are straightforward.

**Follow the steps.**

1. Create a lookup table when values carry extra metadata (such as `project_status`).
2. Use a `CHECK` constraint for short, stable lists.
3. Name enum columns with `_code` or `_status` suffixes for clarity.

```sql
CREATE TABLE project_status (
    status_code VARCHAR(32) PRIMARY KEY,
    display_label VARCHAR(64) NOT NULL
);

ALTER TABLE project
    ADD COLUMN status_code VARCHAR(32) NOT NULL DEFAULT 'planning',
    ADD CONSTRAINT fk_project__project_status
        FOREIGN KEY (status_code) REFERENCES project_status (status_code);
```

**Checklist before ending a migration**

- [ ] Does each table use a surrogate `id` primary key?
- [ ] Do foreign keys follow the `<table>_id` pattern and match data types?
- [ ] Are `created_at` and `updated_at` present on mutable tables?
- [ ] Do soft deletes use `deleted_at`?
- [ ] Do booleans start with `is_` or `has_`?
- [ ] Are units spelled out in column names?
- [ ] Are limited-value columns backed by a lookup table or clear enum name?

Tick every item and your table design will align with the rest of this guide.
