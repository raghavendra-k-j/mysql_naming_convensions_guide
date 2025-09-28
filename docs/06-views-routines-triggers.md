# Chapter 6 – Views, Routines & Triggers

[← Previous: Constraints & Indexes](05-constraints-and-indexes.md) • [Back to index](index.md) • [Next chapter →](07-migrations-and-versioning.md)

Views, stored routines, and triggers sit on top of tables and columns. They shape how data is read and written, so their names must make intent obvious. When names are predictable, engineers can reuse logic safely, review changes faster, and avoid accidental behaviour hidden inside the database. This chapter shows how to name each object type and why the patterns save teams from painful debugging sessions.

## 6.1 Views (`v_<purpose>`)

**Rule.** Name every view with the prefix `v_` followed by a short description of the result, such as `v_customer_balance`.

**Why this matters.**
- The prefix immediately tells readers that the object is a view, not a table.
- Describing the purpose in plain words helps analysts and application code pick the right view without reading the definition.
- Monitoring teams can distinguish real tables from reporting layers when investigating slow queries.

**What goes wrong without the rule.**
- A view named like a table (`customer_balance`) fools ORM tools into writing to it, causing errors.
- Teams duplicate logic because they do not realise a helpful view already exists.
- Production incidents grow worse when a hotfix drops or rewrites a view that looked like a temporary table.

**How to apply it.**
1. Start with `v_` to flag the object type.
2. Add a clear noun phrase that matches the data returned (for example, `invoice_summary`, `active_subscription`).
3. Keep verbs out of the name. Views return datasets; actions belong to procedures.
4. Document the business logic in the migration or source code so readers know when to use the view.

**Practical example.**
```sql
CREATE OR REPLACE VIEW v_active_subscription AS
SELECT
    s.id AS subscription_id,
    s.customer_id,
    p.plan_code,
    s.current_period_end
FROM subscription s
JOIN plan p ON p.id = s.plan_id
WHERE s.status = 'active';
```

> **Field Note:** Names that read like reports keep analysts from rebuilding the same SQL in dashboards.

## 6.2 Stored Procedures (`sp_<verb>_<object>`)

**Rule.** Name stored procedures with the prefix `sp_`, followed by a verb and the object they act on, like `sp_archive_orders`.

**Why this matters.**
- Procedures execute actions. Verb-first names set the expectation that running the routine will change data or trigger side effects.
- Reviews become faster because the name advertises the intent before anyone reads the body.
- Application code can search for `sp_` to audit which routines it depends on.

**What goes wrong without the rule.**
- A procedure named `orders` looks like a table and may be called accidentally during manual work.
- Verb-free names hide the action, leading to misuse (for example, a nightly job calling a routine that issues refunds instead of generating a report).
- Teams create duplicate procedures with slight differences because nobody realises an existing routine handles the same task.

**How to apply it.**
1. Prefix with `sp_`.
2. Use a present-tense verb that states the action (`calculate`, `archive`, `rotate`).
3. Add the primary object name in singular form (`invoice`, `customer_payment`).
4. If the routine focuses on a scope such as a timeframe, suffix it (`sp_rotate_api_keys_daily`).

**Practical example.**
```sql
DELIMITER //
CREATE PROCEDURE sp_archive_orders (IN cutoff_date DATE)
BEGIN
    UPDATE `order`
    SET archived_at = NOW()
    WHERE status = 'delivered'
      AND delivered_at < cutoff_date
      AND archived_at IS NULL;
END //
DELIMITER ;
```

> **Field Note:** Clear procedure names stop midnight deploys from calling the wrong routine.

## 6.3 Functions (`fn_<result>`)

**Rule.** Name stored functions with the prefix `fn_` followed by the value they return, such as `fn_next_invoice_number`.

**Why this matters.**
- Functions are used inside queries; a result-focused name explains what value appears in result sets.
- Prefixing prevents confusion with scalar columns or helper tables that use similar words.
- Code reviewers can quickly confirm that a function returns deterministic values before allowing it in production queries.

**What goes wrong without the rule.**
- A function called `increment` hides its purpose. Does it increment invoice numbers, customer counts, or retry attempts?
- Without the prefix, developers may call the function in application code thinking it is a stored procedure.
- Ambiguous names hide expensive logic, leading to slow queries when functions are misused in WHERE clauses.

**How to apply it.**
1. Prefix with `fn_` to signal a return value.
2. Use nouns that describe the computed value (`due_date`, `sanitised_email`).
3. If the function depends on context (such as tenant or timezone), include it (`fn_format_datetime_utc`).
4. Keep names short but specific; avoid generic endings like `_value`.

**Practical example.**
```sql
DELIMITER //
CREATE FUNCTION fn_next_invoice_number(p_account_id BIGINT)
RETURNS VARCHAR(32)
DETERMINISTIC
BEGIN
    DECLARE v_next_number VARCHAR(32);
    SELECT LPAD(COALESCE(MAX(sequence), 0) + 1, 8, '0')
      INTO v_next_number
    FROM invoice
    WHERE account_id = p_account_id;
    RETURN CONCAT('INV-', v_next_number);
END //
DELIMITER ;
```

> **Field Note:** When the name explains the return value, you know instantly whether it belongs in SELECT or WHERE clauses.

## 6.4 Triggers (`trg_<table>__<event>__<timing>`)

**Rule.** Name triggers with the prefix `trg_`, followed by the table, the fired event, and the timing, separated by double underscores. Example: `trg_invoice__after__insert`.

**Why this matters.**
- Triggers run automatically. A descriptive name helps teams trace side effects during incidents.
- Listing table, event, and timing keeps deployments safe because reviewers see exactly when the trigger fires.
- Tooling that exports schemas can filter `trg_` objects to confirm they align with application expectations.

**What goes wrong without the rule.**
- A trigger named `update_status` hides where it lives and when it runs, causing circular updates or recursion loops.
- Engineers forget that a trigger exists and duplicate logic in application code, causing inconsistent behaviour.
- Debugging replication issues takes longer because the trigger’s scope is unclear.

**How to apply it.**
1. Begin with `trg_`.
2. Add the table name in singular form.
3. Insert the event (`insert`, `update`, `delete`) between double underscores.
4. Add the timing (`before`, `after`) as the final section.
5. Document the reason for the trigger in the migration so reviewers know why the database performs the action.

**Practical example.**
```sql
DELIMITER //
CREATE TRIGGER trg_invoice__after__insert
AFTER INSERT ON invoice
FOR EACH ROW
BEGIN
    INSERT INTO invoice_log (invoice_id, action, occurred_at)
    VALUES (NEW.id, 'created', NOW());
END //
DELIMITER ;
```

> **Field Note:** A well-named trigger announces its behaviour before you open the code.

[← Previous: Constraints & Indexes](05-constraints-and-indexes.md) • [Back to index](index.md) • [Next chapter →](07-migrations-and-versioning.md)
