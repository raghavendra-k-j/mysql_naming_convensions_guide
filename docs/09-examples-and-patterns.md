# Chapter 9 – Examples & Patterns in Action

[← Previous: Additional Conventions That Pay Off at Scale](08-additional-conventions.md) • [Back to index](index.md) • [Next chapter →](10-cheat-sheet.md)

This chapter turns the rules into a small, consistent scenario: a project-tracking application. You will see tables, constraints, routines, a view, and a migration file that all follow the conventions from earlier chapters.

## 9.1 Example schema: `project`, `team_member`, `time_entry`

### Design goals

- Demonstrate lower_snake_case across the entire schema.
- Show how surrogate keys and foreign keys use the `_id` pattern.
- Highlight explicit units (`_minutes`, `_cents`) and timestamps (`_at`).

### Table definitions

```sql
CREATE TABLE project (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    public_id CHAR(27) NOT NULL,
    name VARCHAR(160) NOT NULL,
    time_budget_minutes INT UNSIGNED NOT NULL DEFAULT 0,
    status_code VARCHAR(32) NOT NULL DEFAULT 'planning',
    is_active TINYINT(1) NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    archived_at DATETIME NULL,
    UNIQUE KEY ux_project__public_id (public_id)
);

CREATE TABLE team_member (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(200) NOT NULL,
    email VARCHAR(320) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY ux_team_member__email (email)
);

CREATE TABLE time_entry (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    project_id BIGINT UNSIGNED NOT NULL,
    team_member_id BIGINT UNSIGNED NOT NULL,
    logged_minutes INT UNSIGNED NOT NULL,
    billable_amount_cents INT UNSIGNED NOT NULL,
    submitted_at DATETIME NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_time_entry__project_id__project
        FOREIGN KEY (project_id) REFERENCES project (id),
    CONSTRAINT fk_time_entry__team_member_id__team_member
        FOREIGN KEY (team_member_id) REFERENCES team_member (id)
);
```

### Why the names matter

- Each table uses a surrogate `id` and optional public identifier when needed.
- Foreign keys mirror the referenced table names, so joins read naturally.
- Units and suffixes make queries self-explanatory (`logged_minutes`, `submitted_at`).

## 9.2 Example constraints and indexes

```sql
ALTER TABLE project
    ADD CONSTRAINT ck_project__time_budget_minutes_nonneg
        CHECK (time_budget_minutes >= 0);

ALTER TABLE time_entry
    ADD CONSTRAINT ck_time_entry__billable_amount_cents_nonneg
        CHECK (billable_amount_cents >= 0),
    ADD CONSTRAINT ux_time_entry__project_id_team_member_id_submitted_at
        UNIQUE (project_id, team_member_id, submitted_at),
    ADD INDEX ix_time_entry__project_id_submitted_at (project_id, submitted_at);
```

### Why the names matter

- Check constraints describe the business rule being enforced.
- The unique constraint spells out the natural key used for deduplication.
- The index name reveals which query pattern it accelerates.

## 9.3 Example view, function, and trigger

```sql
CREATE OR REPLACE VIEW v_project_recent_activity AS
SELECT
    p.id AS project_id,
    p.name,
    te.team_member_id,
    te.logged_minutes,
    te.billable_amount_cents,
    te.submitted_at
FROM project p
JOIN time_entry te ON te.project_id = p.id
WHERE te.submitted_at >= NOW() - INTERVAL 30 DAY;
```

```sql
DELIMITER //
CREATE FUNCTION fn_project_logged_minutes(p_project_id BIGINT)
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE v_total INT;
    SELECT COALESCE(SUM(logged_minutes), 0) INTO v_total
    FROM time_entry
    WHERE project_id = p_project_id;
    RETURN v_total;
END //
DELIMITER ;
```

```sql
DELIMITER //
CREATE TRIGGER trg_project__after__update
AFTER UPDATE ON project
FOR EACH ROW
BEGIN
    IF NEW.is_active = 0 AND OLD.is_active = 1 THEN
        INSERT INTO project_activity_log (project_id, action_code, occurred_at)
        VALUES (NEW.id, 'archived', NOW());
    END IF;
END //
DELIMITER ;
```

### Why the names matter

- Prefixes (`v_`, `fn_`, `trg_`) highlight the object type instantly.
- Function and trigger names read like sentences, so reviewers know their purpose at a glance.
- The trigger name records both timing and event, making it easy to spot in schema dumps.

## 9.4 Example migration files

```text
20240718120000_create-project-tables.sql
20240718120500_add-time-entry-constraints.sql
20240718121000_create-v_project_recent_activity.sql
```

```sql
-- 20240718120500_add-time-entry-constraints.sql
-- Adds deduplication and non-negative checks to time_entry.
-- Rollback: 20240718120500_add-time-entry-constraints_down.sql
ALTER TABLE time_entry
    ADD CONSTRAINT ck_time_entry__billable_amount_cents_nonneg
        CHECK (billable_amount_cents >= 0),
    ADD CONSTRAINT ux_time_entry__project_id_team_member_id_submitted_at
        UNIQUE (project_id, team_member_id, submitted_at);
```

### Why the names matter

- Timestamps maintain order across environments.
- Slugs summarise the intent without opening the file.
- Rollback comments explain how to undo the change.

By following the same patterns in your own projects, you make the schema self-documenting and easier to maintain.
