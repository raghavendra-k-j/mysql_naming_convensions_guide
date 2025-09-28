# Chapter 6 – Views, Routines & Triggers

[← Previous: Constraints & Indexes](05-constraints-and-indexes.md) • [Back to index](index.md) • [Next chapter →](07-migrations-and-versioning.md)

Derived objects—views, stored programs, and triggers—sit on top of your tables. A consistent naming style tells reviewers what each object does before they open the definition.

## 6.1 Views → `v_<purpose>`

**Learn the rule.** Prefix every view with `v_` and describe the dataset that comes back.

**Why it works.**

- Anyone scanning the schema can distinguish views from tables immediately.
- Analytics teams reuse shared logic instead of rebuilding the same SELECT statements.
- ORMs avoid writing to views accidentally because the prefix signals "read-only." 

**Follow the steps.**

1. Start with `v_`.
2. Add a noun phrase that explains the result (`project_capacity`, `member_summary`).
3. Keep the name action-free—views return data rather than perform work.

```sql
CREATE OR REPLACE VIEW v_project_recent_activity AS
SELECT
    p.id AS project_id,
    p.name,
    te.team_member_id,
    te.logged_minutes,
    te.created_at
FROM project p
JOIN time_entry te ON te.project_id = p.id
WHERE te.created_at >= NOW() - INTERVAL 14 DAY;
```

## 6.2 Stored procedures → `sp_<verb>_<object>`

**Learn the rule.** Prefix procedures with `sp_`, then use a verb plus the target object.

**Why it works.**

- Procedures perform actions, and the verb sets expectations.
- Reviewers understand the side effects before reading the code.
- Application teams can search for `sp_` to audit write paths.

**Follow the steps.**

1. Begin with `sp_`.
2. Use a present-tense verb that describes the action (`close`, `rebuild`, `archive`).
3. Add the object in singular form (`project`, `time_entry`).
4. Append context if needed (`sp_archive_project_monthly`).

```sql
DELIMITER //
CREATE PROCEDURE sp_archive_project (IN p_project_id BIGINT)
BEGIN
    UPDATE project
    SET is_active = 0,
        archived_at = NOW()
    WHERE id = p_project_id
      AND archived_at IS NULL;
END //
DELIMITER ;
```

## 6.3 Functions → `fn_<result>`

**Learn the rule.** Prefix stored functions with `fn_` and describe the value they return.

**Why it works.**

- Developers know immediately that the routine returns a scalar value.
- The name clarifies where to use the function (SELECT, WHERE, or computed column).
- Ambiguous helper names disappear.

**Follow the steps.**

1. Start with `fn_`.
2. Describe the computed result (`remaining_budget_minutes`).
3. Include context when necessary (`fn_format_project_code`).

```sql
DELIMITER //
CREATE FUNCTION fn_project_remaining_budget(p_project_id BIGINT)
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE v_budget INT;
    DECLARE v_logged INT;

    SELECT time_budget_minutes INTO v_budget
    FROM project
    WHERE id = p_project_id;

    SELECT COALESCE(SUM(logged_minutes), 0) INTO v_logged
    FROM time_entry
    WHERE project_id = p_project_id;

    RETURN GREATEST(v_budget - v_logged, 0);
END //
DELIMITER ;
```

## 6.4 Triggers → `trg_<table>__<timing>__<event>`

**Learn the rule.** Show the target table, the timing, and the event, separated by double underscores.

**Why it works.**

- Readers can tell exactly when the trigger fires.
- Incident responders can search for all triggers on a table quickly.
- Prevents confusion between similar triggers on the same table.

**Follow the steps.**

1. Prefix with `trg_`.
2. Add the table name in singular form.
3. Add the timing (`before`, `after`).
4. Add the event (`insert`, `update`, `delete`).

```sql
DELIMITER //
CREATE TRIGGER trg_time_entry__before__insert
BEFORE INSERT ON time_entry
FOR EACH ROW
BEGIN
    IF NEW.logged_minutes < 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'logged_minutes must be non-negative';
    END IF;
END //
DELIMITER ;
```

**Quick checklist**

- [ ] Does the prefix tell you the object type (`v_`, `sp_`, `fn_`, `trg_`)?
- [ ] Does the rest of the name explain the purpose, timing, or result?
- [ ] Could someone understand the object’s intent without opening the definition?

If the answer is yes, your derived objects match the convention.
