# Chapter 9 – Examples & Patterns in Action

[← Previous: Additional Conventions That Pay Off at Scale](08-additional-conventions.md) • [Back to index](index.md) • [Next chapter →](10-cheat-sheet.md)

This chapter turns the naming rules into a working example. We build a small commerce data model with three tables, wire up the constraints, add a reporting view, write a helper function and trigger, and finish with a migration script. Each section explains why the names follow the standard and what would go wrong if we cut corners.

## 9.1 Example Schema: `user`, `currency`, `order`

### Design goals
- Show how lowercase snake case keeps object names readable.
- Demonstrate consistent suffixes such as `_id`, `_at`, and `_amount`.
- Highlight how descriptive names make joins and reports self-explanatory.

### Table definitions
The three core tables mirror a real checkout flow. `user` stores account details, `currency` stores allowed ISO codes, and `order` records purchases.

```sql
CREATE TABLE `user` (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    public_id CHAR(26) NOT NULL UNIQUE COMMENT 'Short ID shared with customers',
    email VARCHAR(320) NOT NULL UNIQUE,
    full_name VARCHAR(200) NOT NULL,
    preferred_currency_code CHAR(3) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME NULL,
    CONSTRAINT fk_user_currency
        FOREIGN KEY (preferred_currency_code)
        REFERENCES currency (code)
        ON UPDATE CASCADE
);

CREATE TABLE currency (
    code CHAR(3) PRIMARY KEY COMMENT 'ISO 4217 currency code',
    name VARCHAR(64) NOT NULL,
    fraction SMALLINT UNSIGNED NOT NULL COMMENT 'Number of fractional digits'
);

CREATE TABLE `order` (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    public_id CHAR(26) NOT NULL UNIQUE,
    user_id BIGINT UNSIGNED NOT NULL,
    currency_code CHAR(3) NOT NULL,
    total_amount_cents BIGINT UNSIGNED NOT NULL,
    status VARCHAR(32) NOT NULL,
    placed_at DATETIME NOT NULL,
    fulfilled_at DATETIME NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_order_user
        FOREIGN KEY (user_id) REFERENCES `user` (id),
    CONSTRAINT fk_order_currency
        FOREIGN KEY (currency_code) REFERENCES currency (code)
);
```

### Why the names matter
- Readers can guess relationships: `user_id` points to `user.id`, `currency_code` points to `currency.code`.
- Timestamp columns share the `_at` suffix so a report can scan for date fields quickly.
- `total_amount_cents` makes the unit explicit; `total_amount` alone would force readers to check documentation.

> **Field Note:** Using consistent names across tables lets BI tools auto-discover joins instead of requiring manual mapping.

## 9.2 Example Constraints & Indexes

Constraints and indexes deserve descriptive names too. They explain intent when MySQL throws errors or when you inspect the schema with tooling.

```sql
ALTER TABLE `user`
    ADD CONSTRAINT ux_user_email UNIQUE (email),
    ADD INDEX ix_user_deleted_at (deleted_at);

ALTER TABLE `order`
    ADD CONSTRAINT ck_order_status
        CHECK (status IN ('pending', 'paid', 'refunded', 'cancelled')),
    ADD INDEX ix_order_user_id_status (user_id, status),
    ADD INDEX ix_order_placed_at (placed_at);
```

### Why the names matter
- `ux_user_email` tells reviewers that the unique key protects email addresses. Without the prefix the intent would be hidden.
- `ck_order_status` documents the allowed values in plain words. Debugging failed inserts becomes faster because MySQL includes the name in the error message.
- Composite indexes like `ix_order_user_id_status` expose the column order so engineers can plan queries that match it.

> **Field Note:** When a migration fails in production, a clear constraint name lets you find and fix the broken rule without guessing.

## 9.3 Example View, Function, Trigger

Naming rules help readers understand logic beyond tables. The following objects show how the prefixes from Chapter 6 apply in practice.

```sql
CREATE OR REPLACE VIEW v_user_recent_orders AS
SELECT
    u.id AS user_id,
    u.email,
    o.public_id AS order_public_id,
    o.total_amount_cents,
    o.placed_at,
    o.status
FROM `user` u
JOIN `order` o ON o.user_id = u.id
WHERE o.placed_at >= NOW() - INTERVAL 30 DAY;
```

```sql
DELIMITER $$
CREATE FUNCTION fn_order_total_in_currency(p_order_id BIGINT UNSIGNED, p_currency_code CHAR(3))
RETURNS DECIMAL(18, 2)
DETERMINISTIC
BEGIN
    DECLARE v_total_cents BIGINT;
    DECLARE v_fraction SMALLINT;

    SELECT total_amount_cents INTO v_total_cents
    FROM `order`
    WHERE id = p_order_id;

    SELECT fraction INTO v_fraction
    FROM currency
    WHERE code = p_currency_code;

    RETURN v_total_cents / POW(10, v_fraction);
END$$
DELIMITER ;
```

```sql
DELIMITER $$
CREATE TRIGGER trg_order_set_fulfilled_at
BEFORE UPDATE ON `order`
FOR EACH ROW
BEGIN
    IF NEW.status = 'paid' AND OLD.status <> 'paid' THEN
        SET NEW.fulfilled_at = NOW();
    END IF;
END$$
DELIMITER ;
```

### Why the names matter
- The `v_`, `fn_`, and `trg_` prefixes tell the reader what kind of object they are before reading the definition.
- `fn_order_total_in_currency` describes inputs and behaviour. A vague name like `fn_convert` would leave the purpose unclear.
- `trg_order_set_fulfilled_at` encodes both the target table and the action so maintainers can find it quickly.

> **Field Note:** Clear routine names reduce onboarding time because new engineers can map business flows without reading every line of SQL.

## 9.4 Example Migration File

Migrations are the final piece. A good filename and structure keep releases predictable.

### File name pattern
```
20240718-120000_add-order-totals.sql
```
- The timestamp sorts migrations chronologically.
- Hyphen-separated words describe the action in plain language.
- The verb-noun structure (`add-order-totals`) mirrors the change inside the file.

### File contents
```sql
-- 20240718-120000_add-order-totals.sql
-- Adds total_amount_cents and fulfilled_at columns to order table.

START TRANSACTION;

ALTER TABLE `order`
    ADD COLUMN total_amount_cents BIGINT UNSIGNED NOT NULL DEFAULT 0 AFTER currency_code,
    ADD COLUMN fulfilled_at DATETIME NULL AFTER placed_at,
    ADD INDEX ix_order_placed_at (placed_at);

UPDATE `order`
SET total_amount_cents = 0
WHERE total_amount_cents IS NULL;

COMMIT;
```

### Why the structure works
- A short comment at the top documents the intent for reviewers and incident responders.
- Wrapping changes in a transaction makes the migration safe to rerun if a step fails.
- Naming the index inside the migration keeps schema diffs clean when running `SHOW CREATE TABLE` later.

> **Field Note:** Teams that describe the change and index names directly in the migration file avoid the "mystery DDL" problem when reading production history.

[← Previous: Additional Conventions That Pay Off at Scale](08-additional-conventions.md) • [Back to index](index.md) • [Next chapter →](10-cheat-sheet.md)
