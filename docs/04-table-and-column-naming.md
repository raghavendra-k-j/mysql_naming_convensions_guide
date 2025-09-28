# Chapter 4 – Table & Column Naming

[← Previous: Database-Level Conventions](03-database-level-conventions.md) • [Back to index](index.md) • [Next chapter →](05-constraints-and-indexes.md)

Tables and columns are the backbone of every MySQL schema. This chapter explains how to name them so relationships stay clear,
queries stay predictable, and migrations stay easy even when the database grows to millions of rows.

## 4.1 Primary Keys → Why Surrogate Keys (id) Beat Natural Keys

**Rule.** Give every table a single-column surrogate primary key named `id`. Let MySQL assign it automatically with `AUTO_INCREMENT` or a UUID generator.

**Why this matters.**
- Surrogate keys never change when business rules change.
- Joining tables becomes predictable because every foreign key points to `id`.
- ORMs and migration tools expect the pattern and behave well when it is present.

**What goes wrong without the rule.**
- Natural keys like email addresses or phone numbers change, forcing cascading updates in child tables.
- Composite primary keys complicate joins and lead to duplicate records when one column is missed in a query.
- Debugging becomes slower because logs and monitoring tools cannot refer to a single identifier.

**How to apply it.**
1. Declare the primary key column as `id` with `BIGINT UNSIGNED` (or `CHAR(36)` for UUIDs).
2. Mark it as `PRIMARY KEY` and `NOT NULL`.
3. Use a surrogate key even if you also enforce a unique natural key (for example, email).
4. Keep the pattern consistent across all tables so teams build muscle memory.

**Practical example.**
```sql
CREATE TABLE customer_accounts (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    email VARCHAR(320) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY ux_customer_accounts_email (email)
);
```

> **Field Note:** Business identifiers come and go. A stable `id` column is the anchor that keeps history accurate.

## 4.2 Foreign Keys → Why `<table>_id` Pattern Avoids Confusion

**Rule.** Name every foreign key column after the referenced table plus `_id`, such as `customer_id` or `order_id`.

**Why this matters.**
- Readers understand the relationship without scanning the schema.
- ORMs can infer associations automatically.
- Query planners show clear join paths when column names match table names.

**What goes wrong without the rule.**
- Columns like `user` or `customerRef` force developers to open table definitions to confirm the link.
- Mismatched names (`client_id` referencing `customers.id`) cause foreign keys to be skipped during migrations.
- Accidental cross-service joins happen when teams reuse vague column names.

**How to apply it.**
1. Take the referenced table name in singular form (for example, `customers` → `customer`).
2. Append `_id` and declare the column with the same type as the primary key.
3. Add a foreign key constraint named `fk_<table>__<referenced_table>` in Chapter 5 style.
4. Index the column to keep joins fast.

**Practical example.**
```sql
CREATE TABLE customer_orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_id BIGINT UNSIGNED NOT NULL,
    total_amount_cents INT UNSIGNED NOT NULL,
    PRIMARY KEY (id),
    KEY ix_customer_orders_customer_id (customer_id),
    CONSTRAINT fk_customer_orders__customers
        FOREIGN KEY (customer_id) REFERENCES customers (id)
);
```

> **Field Note:** When every foreign key ends with `_id`, junior developers can write correct joins on day one.

## 4.3 Timestamps → Why Every Table Needs `created_at` and `updated_at`

**Rule.** Add `created_at` and `updated_at` columns to every table that stores mutable data. Store UTC timestamps in `DATETIME` or `TIMESTAMP`.

**Why this matters.**
- You can audit when records were inserted and last changed.
- Change data capture tools rely on timestamps to sync downstream systems.
- Support teams can answer "when did this happen?" without reading logs.

**What goes wrong without the rule.**
- Bug investigations stall because no one knows when a record appeared.
- Batch jobs may reprocess old data because there is no easy way to filter by recent changes.
- Deleting or archiving data becomes risky without a timeline.

**How to apply it.**
1. Declare both columns as `DATETIME(6)` or `TIMESTAMP(6)` with `NOT NULL`.
2. Default `created_at` to `CURRENT_TIMESTAMP(6)` and `updated_at` to `CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)`.
3. Ensure application code updates `updated_at` when bulk updates bypass the automatic trigger.
4. Keep all times in UTC to avoid daylight-saving surprises.

**Practical example.**
```sql
CREATE TABLE invoices (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_id BIGINT UNSIGNED NOT NULL,
    total_amount_cents INT UNSIGNED NOT NULL,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        ON UPDATE CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id)
);
```

> **Field Note:** When an incident happens, the first question is "when." These two columns provide the answer instantly.

## 4.4 Soft Deletes → Why `deleted_at` Beats Boolean Flags

**Rule.** Represent soft deletes with a nullable `deleted_at` column instead of a boolean flag like `is_deleted`.

**Why this matters.**
- You capture the exact moment a record became inactive.
- Restoring data is easier because the original record stays in place.
- Analytics can filter on a date range rather than juggling boolean logic.

**What goes wrong without the rule.**
- Boolean flags drift out of sync when bulk scripts forget to update them.
- There is no audit trail showing when the delete occurred or who triggered it.
- Race conditions appear when two processes toggle the flag simultaneously.

**How to apply it.**
1. Add `deleted_at DATETIME(6) NULL` to tables that require soft deletes.
2. Treat any non-null value as deleted.
3. Apply an index on `(deleted_at)` if you routinely filter active vs. deleted rows.
4. Create views that filter `deleted_at IS NULL` for application reads.

**Practical example.**
```sql
CREATE TABLE user_profiles (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    display_name VARCHAR(120) NOT NULL,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        ON UPDATE CURRENT_TIMESTAMP(6),
    deleted_at DATETIME(6) NULL,
    PRIMARY KEY (id)
);
```

> **Field Note:** A timestamp soft delete turns every "oops" moment into a reversible event.

## 4.5 Booleans → Why `is_` and `has_` Prefixes Improve Readability

**Rule.** Name boolean columns with `is_` or `has_` prefixes, such as `is_active` or `has_consented`.

**Why this matters.**
- The value reads like a sentence: `is_active = 1` means "this record is active." 
- Code reviews go faster because the intention is clear in both SQL and application code.
- Avoids confusion with nouns that may be mistaken for foreign keys or enums.

**What goes wrong without the rule.**
- Columns named `active` or `status` require extra documentation.
- Teams mix numeric flags (`0/1`) and textual flags (`Y/N`), making queries brittle.
- Developers may store more than two states in a boolean column, causing silent bugs.

**How to apply it.**
1. Start boolean names with `is_` for states and `has_` for possession.
2. Use `TINYINT(1) NOT NULL` and default to `0` or `1` based on the safe default.
3. Reserve enums for multi-state values instead of overloading booleans.
4. Document any boolean defaults in migration notes.

**Practical example.**
```sql
ALTER TABLE customer_accounts
    ADD COLUMN is_active TINYINT(1) NOT NULL DEFAULT 1,
    ADD COLUMN has_marketing_opt_in TINYINT(1) NOT NULL DEFAULT 0;
```

> **Field Note:** Read the column name aloud. If it forms a true/false question, you named it well.

## 4.6 Amounts & Units → Why Explicit Units Prevent Ambiguity

**Rule.** Include the measurement unit or scale in the column name, especially for money and physical quantities.

**Why this matters.**
- Prevents accidental mixing of currencies or measurement systems.
- Makes queries self-explanatory without comments.
- Supports analytics by clarifying which conversions are required.

**What goes wrong without the rule.**
- Columns named `amount` lead to bugs when one service expects dollars and another sends cents.
- Exported data loses context, causing downstream teams to apply the wrong math.
- Unit changes become painful because no one knows which tables still use the old scale.

**How to apply it.**
1. Append the unit to the column name, such as `_cents`, `_usd`, `_grams`, or `_meters`.
2. Store integers for money to avoid floating point rounding issues.
3. Document any conversion logic in the service README.
4. Use consistent units across the database; convert at the application boundary if needed.

**Practical example.**
```sql
CREATE TABLE subscription_invoices (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_id BIGINT UNSIGNED NOT NULL,
    subtotal_amount_cents INT UNSIGNED NOT NULL,
    tax_amount_cents INT UNSIGNED NOT NULL,
    total_amount_cents INT UNSIGNED NOT NULL,
    PRIMARY KEY (id)
);
```

> **Field Note:** When column names say "cents" or "grams," spreadsheets stay accurate even after export.

## 4.7 JSON Columns → Why `_json` Helps Contain Flexibility

**Rule.** Suffix JSON columns with `_json` and restrict their use to clearly defined scenarios.

**Why this matters.**
- Signals to readers that the column stores semi-structured data.
- Keeps search code honest—developers know to use JSON functions instead of plain SQL operators.
- Makes it easy to audit which tables allow flexible payloads.

**What goes wrong without the rule.**
- Ambiguous names invite people to stuff arbitrary blobs into critical tables.
- JSON usage spreads unchecked, making migrations difficult because the shape is undocumented.
- Indexes break silently when the column type switches between JSON and text without renaming.

**How to apply it.**
1. Use names like `metadata_json` or `payload_json`.
2. Document the allowed structure in the API or schema docs.
3. Add JSON schema validation in the application or via MySQL `CHECK` constraints.
4. Avoid mixing JSON with key relational data—keep the core attributes as columns.

**Practical example.**
```sql
ALTER TABLE webhooks
    ADD COLUMN payload_json JSON NOT NULL,
    ADD COLUMN headers_json JSON NOT NULL;
```

> **Field Note:** A `_json` suffix is a gentle reminder that flexibility is the exception, not the rule.

## 4.8 Enums vs Lookup Tables → Why Lookup Tables Scale Better

**Rule.** Prefer lookup tables with foreign keys over MySQL `ENUM` types for evolving categories.

**Why this matters.**
- Lookup tables allow you to add, rename, or deactivate values without schema changes.
- Foreign keys enforce data integrity across services.
- Reporting queries can join for friendly labels.

**What goes wrong without the rule.**
- Updating an `ENUM` requires migrations across every environment and risks value order bugs.
- Application code often duplicates enum values, leading to mismatched states.
- You cannot attach metadata (like descriptions or sort order) to raw enum values.

**How to apply it.**
1. Create a lookup table with `id`, `code`, and `label` columns.
2. Reference it from the main table with a foreign key (`status_id`).
3. Seed the lookup table in migrations so environments stay consistent.
4. Document how to add or retire entries through change management.

**Practical example.**
```sql
CREATE TABLE order_statuses (
    id TINYINT UNSIGNED NOT NULL AUTO_INCREMENT,
    code VARCHAR(32) NOT NULL,
    label VARCHAR(64) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY ux_order_statuses_code (code)
);

CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    status_id TINYINT UNSIGNED NOT NULL,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id),
    CONSTRAINT fk_orders__order_statuses
        FOREIGN KEY (status_id) REFERENCES order_statuses (id)
);
```

> **Field Note:** Lookup tables turn a one-word status into a mini knowledge base that can grow with the business.

## 4.9 History, Logs, and Backups → Why Explicit Suffixes Save Time

**Rule.** Use descriptive suffixes like `_history`, `_log`, and `_backup` for tables that store derived or archival data.

**Why this matters.**
- Distinguishes primary tables from supporting tables at a glance.
- Helps performance tuning by signaling which tables can be excluded from hot queries.
- Keeps retention policies clear when different rules apply to each suffix.

**What goes wrong without the rule.**
- Teams confuse operational tables with audit tables and accidentally query the wrong one.
- Storage costs climb because duplicates hide among similarly named tables.
- Incident responders waste time locating the correct backup or log table.

**How to apply it.**
1. Keep the base noun the same as the source table, then add the suffix (for example, `payments` → `payments_history`).
2. Store archival tables in the same schema unless compliance requires separation.
3. Document retention and pruning policies near the table definition.
4. Expose views that join primary and history tables when needed for analytics.

**Practical example.**
```sql
CREATE TABLE payments (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_id BIGINT UNSIGNED NOT NULL,
    amount_cents INT UNSIGNED NOT NULL,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id)
);

CREATE TABLE payments_history (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    payment_id BIGINT UNSIGNED NOT NULL,
    change_type VARCHAR(32) NOT NULL,
    changed_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id)
);
```

> **Field Note:** Clear suffixes keep operations calm when auditors or incident teams request a data trail.

## 4.10 Partitions & Shards → Why Native Partitioning Beats Table Explosion

**Rule.** When scaling, prefer MySQL's native partitioning or sharding features instead of manually creating many similarly named tables.

**Why this matters.**
- Partitioning keeps the schema simple: one logical table, many storage segments.
- Query planners can prune partitions automatically, improving performance.
- Operations teams maintain fewer objects, reducing human error.

**What goes wrong without the rule.**
- Creating tables like `events_2023`, `events_2024`, `events_2025` fragments data and confuses tooling.
- Application code must dynamically pick table names, leading to hard-to-test branches.
- Dropping or archiving old data becomes a risky manual process.

**How to apply it.**
1. Design a single table with partitioning by range, list, or hash based on access patterns.
2. Name the table after the business concept (`events`), not the partition scheme.
3. Automate partition maintenance (adding new ones, dropping old ones) with scheduled jobs.
4. If sharding across databases is required, encode the shard in the database name, not the table name (see Chapter 3).

**Practical example.**
```sql
CREATE TABLE events (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    occurred_at TIMESTAMP(6) NOT NULL,
    payload_json JSON NOT NULL,
    PRIMARY KEY (id, occurred_at)
)
PARTITION BY RANGE (YEAR(occurred_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

> **Field Note:** One well-partitioned table is easier to secure and monitor than a fleet of nearly identical tables.

---

[← Previous: Database-Level Conventions](03-database-level-conventions.md) • [Back to index](index.md) • [Next chapter →](05-constraints-and-indexes.md) _(coming soon)_
