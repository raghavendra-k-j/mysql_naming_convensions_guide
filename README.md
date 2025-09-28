# MySQL Naming Conventions Guide

> A detailed reference for teams that need consistent, scalable, and debuggable MySQL schemas.

## How to Use This Guide

Read the chapters in order when establishing a new standard. Each section explains:

* **The rule** – the convention we recommend.
* **Why it matters** – the problems the rule prevents.
* **Examples** – good and bad patterns from real systems.
* **Field notes** – short reminders or call-outs after a topic to reinforce the reasoning.

---

## Table of Contents

- [Introduction](#introduction)
  - [1.1 Why Naming Conventions Matter in Scalable MySQL Projects](#11-why-naming-conventions-matter-in-scalable-mysql-projects)
  - [1.2 Common Pain Points in Poorly Named Schemas](#12-common-pain-points-in-poorly-named-schemas)
  - [1.3 Goals of This Standard (Consistency, Clarity, Scalability, Lintability)](#13-goals-of-this-standard-consistency-clarity-scalability-lintability)
- [Core Naming Rules](#core-naming-rules)
  - [2.1 Case & Style → Why lower_snake_case Wins](#21-case--style--why-lower_snake_case-wins)
  - [2.2 Language → Why Full Words Beat Abbreviations](#22-language--why-full-words-beat-abbreviations)
  - [2.3 Acronyms & Abbreviations → How to Use Them When Necessary (Consistency & Meaning)](#23-acronyms--abbreviations--how-to-use-them-when-necessary-consistency--meaning)
  - [2.4 Singular vs Plural Table Names → Why Singular Is Less Ambiguous](#24-singular-vs-plural-table-names--why-singular-is-less-ambiguous)
  - [2.5 Avoiding Noise Prefixes (tbl_, col_) → Why They Harm More Than Help](#25-avoiding-noise-prefixes-tbl_-col_--why-they-harm-more-than-help)
  - [2.6 Reserved Words → Why They Create Long-Term Bugs](#26-reserved-words--why-they-create-long-term-bugs)
- [Database-Level Conventions](#database-level-conventions)
  - [3.1 Naming Databases by Product and Context](#31-naming-databases-by-product-and-context)
  - [3.2 Handling Multiple Environments (dev, stg, prod) Safely](#32-handling-multiple-environments-dev-stg-prod-safely)
  - [3.3 Why Database Names Should Be Predictable](#33-why-database-names-should-be-predictable)
- [Table & Column Naming](#table--column-naming)
  - [4.1 Primary Keys → Why Surrogate Keys (id) Beat Natural Keys](#41-primary-keys--why-surrogate-keys-id-beat-natural-keys)
  - [4.2 Foreign Keys → Why table_id Pattern Avoids Confusion](#42-foreign-keys--why-table_id-pattern-avoids-confusion)
  - [4.3 Timestamps → Why Every Table Needs created_at and updated_at](#43-timestamps--why-every-table-needs-created_at-and-updated_at)
  - [4.4 Soft Deletes → Why deleted_at Beats Boolean Flags](#44-soft-deletes--why-deleted_at-beats-boolean-flags)
  - [4.5 Booleans → Why is_ and has_ Prefixes Improve Readability](#45-booleans--why-is_-and-has_-prefixes-improve-readability)
  - [4.6 Amounts & Units → Why Explicit Units Prevent Ambiguity](#46-amounts--units--why-explicit-units-prevent-ambiguity)
  - [4.7 JSON Columns → Why _json Helps Contain Flexibility](#47-json-columns--why-_json-helps-contain-flexibility)
  - [4.8 Enums vs Lookup Tables → Why Lookup Tables Scale Better](#48-enums-vs-lookup-tables--why-lookup-tables-scale-better)
  - [4.9 History, Logs, and Backups → Why Explicit Suffixes Save Time](#49-history-logs-and-backups--why-explicit-suffixes-save-time)
  - [4.10 Partitions & Shards → Why Native Partitioning Beats Table Explosion](#410-partitions--shards--why-native-partitioning-beats-table-explosion)
- [Constraints & Indexes](#constraints--indexes)
  - [5.1 Why Naming Constraints Matters for Debugging](#51-why-naming-constraints-matters-for-debugging)
  - [5.2 Primary Keys (pk_...)](#52-primary-keys-pk_)
  - [5.3 Foreign Keys (fk_...)](#53-foreign-keys-fk_)
  - [5.4 Unique Constraints (ux_...)](#54-unique-constraints-ux_)
  - [5.5 Non-Unique Indexes (ix_...)](#55-non-unique-indexes-ix_)
  - [5.6 Check Constraints (ck_...)](#56-check-constraints-ck_)
  - [5.7 Fulltext Indexes (ft_...)](#57-fulltext-indexes-ft_)
- [Views, Routines & Triggers](#views-routines--triggers)
  - [6.1 Views (v_...) → Why Explicit Purpose Names Help Reporting](#61-views-v_--why-explicit-purpose-names-help-reporting)
  - [6.2 Stored Procedures (sp_...) → Why Action-Noun Naming Is Clearer](#62-stored-procedures-sp_--why-action-noun-naming-is-clearer)
  - [6.3 Functions (fn_...) → Why Function Names Should Be Self-Explanatory](#63-functions-fn_--why-function-names-should-be-self-explanatory)
  - [6.4 Triggers (trg_...) → Why Timing and Event in Names Prevent Mistakes](#64-triggers-trg_--why-timing-and-event-in-names-prevent-mistakes)
- [Migrations & Versioning](#migrations--versioning)
  - [7.1 File Naming → Why Timestamped Names Are Essential for Ordering](#71-file-naming--why-timestamped-names-are-essential-for-ordering)
  - [7.2 One Action/Object per File → Why Small Atomic Changes Matter](#72-one-actionobject-per-file--why-small-atomic-changes-matter)
  - [7.3 Rollback & CICD Safety](#73-rollback--cicd-safety)
- [Additional Conventions That Pay Off at Scale](#additional-conventions-that-pay-off-at-scale)
  - [8.1 Character Sets & Collations → Why utf8mb4 Is the Only Safe Default](#81-character-sets--collations--why-utf8mb4-is-the-only-safe-default)
  - [8.2 Time Handling → Why Always Store UTC with _at](#82-time-handling--why-always-store-utc-with-_at)
  - [8.3 DATETIME vs TIMESTAMP → Differences Trade-offs and Best Practices](#83-datetime-vs-timestamp--differences-trade-offs-and-best-practices)
  - [8.4 Explicit Types & Lengths → Why VARCHAR320 Beats TEXT](#84-explicit-types--lengths--why-varchar320-beats-text)
  - [8.5 Nullability → Why NOT NULL Should Be Default](#85-nullability--why-not-null-should-be-default)
  - [8.6 IDs → Why Dual IDs (id + public_id) Improve InternalExternal Safety](#86-ids--why-dual-ids-id--public_id-improve-internalexternal-safety)
  - [8.7 Naming Collisions → Why Extending Reserved Names Prevents Bugs](#87-naming-collisions--why-extending-reserved-names-prevents-bugs)
  - [8.8 API Alignment → Why DB Schema Should Mirror API Fields](#88-api-alignment--why-db-schema-should-mirror-api-fields)
  - [8.9 Archival Tables (_archive) → Why Separation Improves Performance](#89-archival-tables-_archive--why-separation-improves-performance)
  - [8.10 Temporary Tables (tmp_...) → Why Explicit Naming Avoids Accidents](#810-temporary-tables-tmp_--why-explicit-naming-avoids-accidents)
  - [8.11 Consistency Over Cleverness → Why Exceptions Must Be Documented](#811-consistency-over-cleverness--why-exceptions-must-be-documented)
  - [8.12 Defaults: Database vs Application — When to Use Which](#812-defaults-database-vs-application--when-to-use-which)
- [Examples & Patterns in Action](#examples--patterns-in-action)
  - [9.1 Example Schema: user currency order](#91-example-schema-user-currency-order)
  - [9.2 Example Constraints & Indexes](#92-example-constraints--indexes)
  - [9.3 Example View Function Trigger](#93-example-view-function-trigger)
  - [9.4 Example Migration File](#94-example-migration-file)
- [Cheat Sheet (Quick Reference)](#cheat-sheet-quick-reference)
- [Conclusion](#conclusion)
  - [11.1 Why These Conventions Work in Real-World Scale](#111-why-these-conventions-work-in-real-world-scale)
  - [11.2 How to Enforce Them (Linters CICD Code Review)](#112-how-to-enforce-them-linters-cicd-code-review)
  - [11.3 The Golden Rule → Consistency Beats Cleverness](#113-the-golden-rule--consistency-beats-cleverness)

---

## Introduction

### 1.1 Why Naming Conventions Matter in Scalable MySQL Projects

**Rule**: Define and follow a shared naming language before the first schema change lands in production.

**Why it matters**: MySQL does not enforce naming consistency. Without a guide, every engineer invents their own style. Over years this becomes a barrier:

* Tooling cannot guess intent, so automated migrations and code generators break.
* New hires waste time deciphering tables instead of shipping value.
* Bugs hide in plain sight because similarly named columns do not mean the same thing.

**Example**:

```sql
-- Without a convention
SELECT * FROM Customers c
JOIN CUSTOMERDETAILS cd ON c.CustID = cd.CustomerID;

-- With a convention
SELECT *
FROM customer c
JOIN customer_profile cp ON c.id = cp.customer_id;
```

The second query is shorter, more predictable, and easier for reviewers to scan.

**Field note**: Consistency is not a cosmetic concern. It is a guardrail that keeps velocity high as teams grow.

### 1.2 Common Pain Points in Poorly Named Schemas

* Duplicate concepts: `order`, `orders`, `tbl_order` all point to the same entity but force mental translation.
* Hidden intent: `flag1` does not explain why the column exists.
* Unstable integrations: APIs and data pipelines fail when column names change unexpectedly.
* Migration churn: Renaming columns to fix clarity triggers downtime, reindexing, and API updates.

**Field note**: Every rename is an outage in disguise—prevent it with clear names on day one.

### 1.3 Goals of This Standard (Consistency, Clarity, Scalability, Lintability)

* **Consistency**: Every schema reads like it was created by one engineer.
* **Clarity**: Column names describe the data without footnotes.
* **Scalability**: Names stay stable when tables grow into billions of rows.
* **Lintability**: Automated checks can enforce these rules because they are explicit and measurable.

**Field note**: If a rule cannot be linted, document it carefully and explain the intent so reviewers can enforce it.

---

## Core Naming Rules

### 2.1 Case & Style → Why lower_snake_case Wins

**Rule**: Use `lower_snake_case` for all database objects.

**Why**:

* MySQL treats names case-insensitively by default. Mixed case hides bugs on macOS (case-insensitive file system) and shows them only on Linux.
* Snake case improves readability in wide tables and long join chains.
* Lowercase reduces keystrokes and avoids quoting requirements.

**Examples**:

```sql
-- ✅ Good
SELECT created_at FROM customer_order;

-- ❌ Bad (requires quoting, breaks tooling)
SELECT `CreatedAt` FROM `CustomerOrder`;
```

**Field note**: Copy/pasting quoted identifiers into ORMs is a common cause of production-only failures.

### 2.2 Language → Why Full Words Beat Abbreviations

**Rule**: Prefer complete words.

**Why**:

* Abbreviations are culture-specific. `acct` might mean account or actual.
* Full words are easier for translators, analysts, and APIs.

**Examples**:

```sql
-- ✅ Good
SELECT account_status FROM billing_account;

-- ❌ Bad
SELECT acct_stat FROM bill_acct;
```

**Field note**: Use abbreviations only when space is limited (see section 2.3).

### 2.3 Acronyms & Abbreviations → How to Use Them When Necessary (Consistency & Meaning)

**Rule**: When abbreviations are unavoidable, document them and keep the same expansion everywhere.

**Why**:

* Consistency reduces guesswork for global teams.
* Static analysis tools can map known acronyms to business terms.

**Examples**:

| Context | Allowed Acronym | Must Expand To |
|---------|-----------------|-----------------|
| Payments | `vat` | value_added_tax |
| Identity | `oauth` | open_authorization |

**Field note**: Keep a shared glossary in source control so reviewers can challenge new acronyms.

### 2.4 Singular vs Plural Table Names → Why Singular Is Less Ambiguous

**Rule**: Name tables in singular form (e.g., `customer`, not `customers`).

**Why**:

* Singular names read naturally in joins (`order.item_id`).
* Pluralization rules differ by language (`people`, `statuses`), creating uneven patterns.

**Examples**:

```sql
-- ✅ Good
SELECT * FROM order_item;

-- ❌ Bad
SELECT * FROM orders_items;
```

**Field note**: Use plural names only for true collections like views that aggregate multiple entities.

### 2.5 Avoiding Noise Prefixes (tbl_, col_) → Why They Harm More Than Help

**Rule**: Do not prefix tables or columns with their type (`tbl_`, `col_`).

**Why**:

* MySQL already knows what is a table or column. Redundant prefixes slow down auto-complete and clutter code reviews.
* Prefixes encourage copy/paste errors (`tbl_tbl_order`).

**Example**:

```sql
-- ✅ Good
SELECT user_id FROM invoice_item;

-- ❌ Bad
SELECT tbl_user_id FROM tbl_invoice_item;
```

**Field note**: Linters can flag prefixes faster than humans—hook them into CI.

### 2.6 Reserved Words → Why They Create Long-Term Bugs

**Rule**: Avoid MySQL reserved words (`order`, `group`, `select`, etc.).

**Why**:

* Reserved names require backticks, which are easy to forget.
* Some drivers silently quote identifiers, masking production bugs.

**Example**:

```sql
-- ✅ Good
SELECT order_number FROM purchase_order;

-- ❌ Bad
SELECT `order` FROM `order`;
```

**Field note**: When in doubt, check `INFORMATION_SCHEMA.KEYWORDS` or MySQL docs.

---

## Database-Level Conventions

### 3.1 Naming Databases by Product and Context

**Rule**: Use `<company>_<domain>`.

**Why**:

* Large organizations run multiple products. Names like `acme_billing` show ownership.

**Example**:

```sql
CREATE DATABASE acme_billing;
```

**Field note**: Include business unit when companies merge to avoid collisions (`contoso_payments`).

### 3.2 Handling Multiple Environments (dev, stg, prod) Safely

**Rule**: Suffix environment explicitly.

**Why**:

* Prevents accidental cross-environment connections.
* Allows automated tooling to apply migrations in the right order.

**Example**:

```sql
CREATE DATABASE acme_billing_dev;
CREATE DATABASE acme_billing_stg;
CREATE DATABASE acme_billing_prod;
```

**Field note**: Block production credentials from connecting to `_dev` databases to catch script mistakes.

### 3.3 Why Database Names Should Be Predictable

Predictable names enable:

* Connection pooling configuration.
* Dashboards that auto-discover environments.
* Least-privilege grants (`GRANT SELECT ON acme_billing_prod.* TO 'bi_app'`).

**Field note**: Agree on a canonical spelling (`stg` vs `stage`) and stick to it.

---

## Table & Column Naming

### 4.1 Primary Keys → Why Surrogate Keys (id) Beat Natural Keys

**Rule**: Give every table a surrogate `id` using `BIGINT UNSIGNED AUTO_INCREMENT` or UUID.

**Why**:

* Natural keys change (email, username).
* Surrogate keys keep joins stable and compact.

**Example**:

```sql
CREATE TABLE customer (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(320) NOT NULL UNIQUE
);
```

**Field note**: Use UUID only when writes happen on multiple regions simultaneously.

### 4.2 Foreign Keys → Why table_id Pattern Avoids Confusion

**Rule**: Name foreign keys after the referenced table plus `_id`.

**Why**:

* Grep-friendly: `rg "customer_id"` finds every relationship.
* ORM-friendly: frameworks auto-detect associations.

**Example**:

```sql
CREATE TABLE customer_order (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  FOREIGN KEY fk_customer_order__customer_id__customer (customer_id)
    REFERENCES customer (id)
);
```

**Field note**: Always index foreign keys for performance.

### 4.3 Timestamps → Why Every Table Needs created_at and updated_at

**Rule**: Include `created_at` and `updated_at` (UTC) on mutable tables.

**Why**:

* Debugging: answer "when did this change?"
* Auditing: required for compliance.

**Example**:

```sql
created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
  ON UPDATE CURRENT_TIMESTAMP(6)
```

**Field note**: Use fractional seconds (`DATETIME(6)`) so multiple updates in a second are ordered correctly.

### 4.4 Soft Deletes → Why deleted_at Beats Boolean Flags

**Rule**: Use nullable `deleted_at` instead of `is_deleted`.

**Why**:

* Tells you when the delete happened.
* Keeps historical data without separate tables.

**Example**:

```sql
deleted_at DATETIME(6) NULL
```

**Field note**: Add a partial index on `deleted_at IS NULL` for performance.

### 4.5 Booleans → Why is_ and has_ Prefixes Improve Readability

**Rule**: Prefix boolean columns with `is_` or `has_`.

**Why**:

* Queries read naturally (`WHERE is_active = 1`).
* Avoids confusion with nouns (`active` could be a state or a command).

**Example**:

```sql
is_active TINYINT(1) NOT NULL DEFAULT 0
```

**Field note**: Store booleans as `TINYINT(1)` to avoid MySQL interpreting strings as truthy values.

### 4.6 Amounts & Units → Why Explicit Units Prevent Ambiguity

**Rule**: Append the unit to numeric columns.

**Why**:

* Prevents currency mix-ups (cents vs dollars).
* Engineers can do math without checking documentation.

**Example**:

```sql
price_cents INT NOT NULL,
weight_kg DECIMAL(10,2) NOT NULL
```

**Field note**: For percentages, store basis points (`rate_bps`) to avoid floating errors.

### 4.7 JSON Columns → Why _json Helps Contain Flexibility

**Rule**: Suffix JSON columns with `_json` and document their schema.

**Why**:

* Signals flexible structure to reviewers.
* Encourages contracts for producers/consumers.

**Example**:

```sql
preferences_json JSON NOT NULL
```

**Field note**: Add a comment with a JSON Schema link when possible.

### 4.8 Enums vs Lookup Tables → Why Lookup Tables Scale Better

**Rule**: Prefer lookup tables to ENUM unless values are tiny and immutable.

**Why**:

* ENUM changes require table rebuilds.
* Lookup tables allow metadata (`display_name`, `sort_order`).

**Example**:

```sql
CREATE TABLE order_status (
  id TINYINT UNSIGNED PRIMARY KEY,
  code VARCHAR(32) NOT NULL UNIQUE,
  description VARCHAR(255) NOT NULL
);
```

**Field note**: If you must use ENUM, mirror the values in application constants.

### 4.9 History, Logs, and Backups → Why Explicit Suffixes Save Time

**Rule**: Use `_hist` for immutable history, `_log` for append-only logs, `_bk` for temporary backups.

**Why**:

* Clarifies retention policies.
* Allows scheduled jobs to find the right tables.

**Example**:

```sql
CREATE TABLE payment_hist (...);
CREATE TABLE webhook_log (...);
CREATE TABLE customer_bk (...);
```

**Field note**: Keep history tables in the same schema to simplify joins.

### 4.10 Partitions & Shards → Why Native Partitioning Beats Table Explosion

**Rule**: Use MySQL partitioning. If physical splits are required, suffix with time (`event_2025q3`).

**Why**:

* Partitioning keeps one logical table for analytics.
* Manual per-period tables multiply migration work.

**Example**:

```sql
ALTER TABLE event
PARTITION BY RANGE (YEAR(created_at)) (
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026)
);
```

**Field note**: When physical tables are unavoidable, document the naming schedule in runbooks.

---

## Constraints & Indexes

### 5.1 Why Naming Constraints Matters for Debugging

**Rule**: Name every constraint. Use prefixes that describe the type.

**Why**:

* Error messages include constraint names. A clear name tells you exactly what failed.
* Cloud consoles surface constraint names in dashboards.

**Field note**: Standard prefixes reduce Slack debugging time.

### 5.2 Primary Keys (pk_...)

**Rule**: `pk_<table>`.

**Example**:

```sql
CONSTRAINT pk_customer PRIMARY KEY (id)
```

**Field note**: Auto-generated names vary per environment; custom names keep diffs stable.

### 5.3 Foreign Keys (fk_...)

**Rule**: `fk_<table>__<column>__<ref_table>`.

**Example**:

```sql
CONSTRAINT fk_customer_order__customer_id__customer
  FOREIGN KEY (customer_id) REFERENCES customer (id)
```

**Field note**: Double underscores separate the parts visually.

### 5.4 Unique Constraints (ux_...)

**Rule**: `ux_<table>__<column>`.

**Example**:

```sql
CONSTRAINT ux_customer__email UNIQUE (email)
```

**Field note**: For compound keys list all columns (`ux_invoice_item__invoice_id_product_id`).

### 5.5 Non-Unique Indexes (ix_...)

**Rule**: `ix_<table>__<leading_column>[_<additional_columns>]`.

**Example**:

```sql
CREATE INDEX ix_customer_order__customer_id_created_at
  ON customer_order (customer_id, created_at);
```

**Field note**: Include order direction when relevant (`ix_event__occurred_at_desc`).

### 5.6 Check Constraints (ck_...)

**Rule**: `ck_<table>__<rule>`.

**Example**:

```sql
CONSTRAINT ck_invoice__amount_cents_nonneg
  CHECK (amount_cents >= 0)
```

**Field note**: Use MySQL 8+ `CHECK` constraints; earlier versions ignored them silently.

### 5.7 Fulltext Indexes (ft_...)

**Rule**: `ft_<table>__<column>`.

**Example**:

```sql
CREATE FULLTEXT INDEX ft_article__content
  ON article (content);
```

**Field note**: Fulltext indexes have storage costs—name them clearly so teams understand why they exist.

---

## Views, Routines & Triggers

### 6.1 Views (v_...) → Why Explicit Purpose Names Help Reporting

**Rule**: Prefix reporting views with `v_` and describe their purpose.

**Why**:

* Signals that the object is a view, not a table.
* Prevents BI tools from writing to views accidentally.

**Example**:

```sql
CREATE VIEW v_customer_revenue AS
SELECT customer_id, SUM(total_cents) AS revenue_cents
FROM customer_order
GROUP BY customer_id;
```

**Field note**: Document whether a view is updatable.

### 6.2 Stored Procedures (sp_...) → Why Action-Noun Naming Is Clearer

**Rule**: `sp_<verb>_<noun>`.

**Why**:

* Makes intent obvious: `sp_archive_invoice` vs `ProcessInvoices`.

**Example**:

```sql
CREATE PROCEDURE sp_archive_invoice (IN invoice_id BIGINT UNSIGNED)
BEGIN
  UPDATE invoice SET deleted_at = NOW(6) WHERE id = invoice_id;
END;
```

**Field note**: Use verbs like `create`, `archive`, `rebuild` to describe work clearly.

### 6.3 Functions (fn_...) → Why Function Names Should Be Self-Explanatory

**Rule**: `fn_<noun>_<qualifier>`.

**Example**:

```sql
CREATE FUNCTION fn_currency_to_usd(amount_cents INT, rate DECIMAL(10,6))
RETURNS INT
DETERMINISTIC
RETURN ROUND(amount_cents * rate);
```

**Field note**: Mark deterministic functions as `DETERMINISTIC` so MySQL can cache results.

### 6.4 Triggers (trg_...) → Why Timing and Event in Names Prevent Mistakes

**Rule**: `trg_<table>__<timing>_<event>`.

**Example**:

```sql
CREATE TRIGGER trg_customer_order__before_insert
BEFORE INSERT ON customer_order
FOR EACH ROW
SET NEW.created_at = COALESCE(NEW.created_at, NOW(6));
```

**Field note**: Include timing and event so maintainers instantly know when the trigger runs.

---

## Migrations & Versioning

### 7.1 File Naming → Why Timestamped Names Are Essential for Ordering

**Rule**: Use `YYYYMMDDHHMMSS__description.sql`.

**Why**:

* Chronological names prevent merge conflicts when two teams add migrations simultaneously.

**Example**:

```
20250315093045__create_customer.sql
```

**Field note**: Use UTC timestamps to avoid reordering when developers work in different time zones.

### 7.2 One Action/Object per File → Why Small Atomic Changes Matter

**Rule**: Each migration should do one thing.

**Why**:

* Easy rollbacks.
* Clean code reviews.

**Example**:

```
20250316094510__add_customer_is_active.sql
```

**Field note**: Combine `ALTER TABLE` statements that modify the same table in one file to reduce locks.

### 7.3 Rollback & CICD Safety

* Pair forward migrations with rollback statements where possible.
* Use CI to run migrations on an ephemeral database before merge.
* Document manual steps (like backfilling data).

**Field note**: Treat rollbacks as first-class citizens—they are the safety net during incidents.

---

## Additional Conventions That Pay Off at Scale

### 8.1 Character Sets & Collations → Why utf8mb4 Is the Only Safe Default

**Rule**: Use `utf8mb4` and a deterministic collation (`utf8mb4_unicode_ci`).

**Why**:

* Supports full Unicode, including emoji.
* Prevents data loss when integrating with mobile apps.

**Example**:

```sql
ALTER DATABASE acme_billing_prod CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Field note**: Avoid `utf8` (3-byte) because it truncates characters silently.

### 8.2 Time Handling → Why Always Store UTC with _at

**Rule**: Store timestamps in UTC and suffix with `_at`.

**Why**:

* Simplifies comparisons across regions.
* Avoids daylight saving bugs.

**Example**:

```sql
expires_at DATETIME(6) NOT NULL
```

**Field note**: Convert to local time in the application layer.

### 8.3 DATETIME vs TIMESTAMP → Differences Trade-offs and Best Practices

* `DATETIME` stores values exactly as given; good for past/future events.
* `TIMESTAMP` auto-converts to/from UTC but has range limits (1970-2038).
* Prefer `DATETIME(6)` for business events, `TIMESTAMP` for system metadata like row versioning.

**Field note**: When using `TIMESTAMP`, ensure default time zone is UTC to avoid surprises.

### 8.4 Explicit Types & Lengths → Why VARCHAR320 Beats TEXT

**Rule**: Pick types and lengths intentionally.

**Why**:

* `TEXT` columns disable some indexes and increase storage.
* `VARCHAR(320)` handles the longest possible email address.

**Example**:

```sql
email VARCHAR(320) NOT NULL
```

**Field note**: Document reasoning in migration comments so future maintainers keep the same constraints.

### 8.5 Nullability → Why NOT NULL Should Be Default

**Rule**: Assume columns are `NOT NULL` unless business logic requires absence.

**Why**:

* NULL complicates queries (`COALESCE`, `IS NULL`).
* Enforces that data is present when needed.

**Example**:

```sql
is_active TINYINT(1) NOT NULL DEFAULT 0
```

**Field note**: When nulls are required, name the column to highlight it (`cancelled_at`).

### 8.6 IDs → Why Dual IDs (id + public_id) Improve InternalExternal Safety

**Rule**: Pair an internal surrogate ID with an external `public_id` when exposing data.

**Why**:

* Prevents guessing sequential IDs in APIs.
* Allows rotation without touching foreign keys.

**Example**:

```sql
public_id CHAR(27) NOT NULL UNIQUE COMMENT 'KSUID for public APIs'
```

**Field note**: Generate public IDs in the application so database backups do not leak them.

### 8.7 Naming Collisions → Why Extending Reserved Names Prevents Bugs

**Rule**: Detect collisions early and extend names instead of quoting.

**Common collisions**:

* Reserved words (`order`, `group`).
* Shared vocabulary across teams (`account` in finance vs identity).
* Existing routines (`calculate_tax` defined in multiple schemas).

**Bad handling**:

```sql
CREATE TABLE `order` (
  `group` VARCHAR(32)
);
```

* Needs quoting everywhere.
* Breaks when copied into tools that strip backticks.

**Good handling**:

```sql
CREATE TABLE purchase_order (
  order_number VARCHAR(32) NOT NULL,
  group_account VARCHAR(64) NOT NULL
);
```

* Adds context, avoids reserved words, and clarifies meaning.

**Field note**: Maintain a registry of core nouns. When a new project wants to reuse a name, negotiate scope or add qualifiers.

### 8.8 API Alignment → Why DB Schema Should Mirror API Fields

**Rule**: Keep API field names aligned with column names when they describe the same data.

**Why**:

* Reduces mapping layers.
* Simplifies debugging API payloads.

**Example**:

```json
{
  "order_id": "po_123",
  "amount_cents": 1500
}
```

**Field note**: If business names change, change them in both API and DB to avoid drift.

### 8.9 Archival Tables (_archive) → Why Separation Improves Performance

**Rule**: Move cold data to tables suffixed `_archive`.

**Why**:

* Keeps operational tables small.
* Allows cheaper storage tiers.

**Example**:

```sql
CREATE TABLE invoice_archive LIKE invoice;
```

**Field note**: Document retention and restoration steps next to the table definition.

### 8.10 Temporary Tables (tmp_...) → Why Explicit Naming Avoids Accidents

**Rule**: Prefix temporary tables with `tmp_`.

**Why**:

* Signals that the table is ephemeral.
* Prevents confusion when scanning schemas.

**Example**:

```sql
CREATE TEMPORARY TABLE tmp_monthly_totals (...);
```

**Field note**: Drop temporary tables explicitly at the end of sessions in long-running jobs.

### 8.11 Consistency Over Cleverness → Why Exceptions Must Be Documented

**Rule**: Break conventions only with explicit documentation and approval.

**Why**:

* Clever names age poorly as teams change.
* Documented exceptions help future migrations stay consistent.

**Field note**: Add a comment in the migration explaining the exception and link to design docs.

### 8.12 Defaults: Database vs Application — When to Use Which

**Database-managed defaults**:

* `created_at`, `updated_at` benefit from database clocks.
* Audit fields stay accurate even when APIs fail to send data.

**Application-managed defaults**:

* Business rules like `status` or `is_active` change often; keep logic in code.
* Avoid magic numbers (`0`, `''`) as database defaults—they hide bugs.

**Hybrid model**:

* Use database defaults for infrastructure concerns.
* Let the application decide business behavior but validate via constraints.

**Field note**: Review defaults whenever you create a new API version to keep behavior predictable.

---

## Examples & Patterns in Action

### 9.1 Example Schema: user currency order

```sql
CREATE TABLE currency (
  id TINYINT UNSIGNED NOT NULL AUTO_INCREMENT,
  code CHAR(3) NOT NULL UNIQUE,
  name VARCHAR(64) NOT NULL,
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
    ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id)
);

CREATE TABLE user (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  public_id CHAR(27) NOT NULL UNIQUE,
  email VARCHAR(320) NOT NULL,
  is_active TINYINT(1) NOT NULL DEFAULT 1,
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
    ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  CONSTRAINT ux_user__email UNIQUE (email)
);

CREATE TABLE customer_order (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  public_id CHAR(27) NOT NULL UNIQUE,
  user_id BIGINT UNSIGNED NOT NULL,
  currency_id TINYINT UNSIGNED NOT NULL,
  amount_cents INT NOT NULL,
  status_id TINYINT UNSIGNED NOT NULL,
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
    ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  CONSTRAINT fk_customer_order__user_id__user FOREIGN KEY (user_id)
    REFERENCES user (id),
  CONSTRAINT fk_customer_order__currency_id__currency FOREIGN KEY (currency_id)
    REFERENCES currency (id)
);
```

**Field note**: The sample schema demonstrates consistent naming across tables, columns, and constraints.

### 9.2 Example Constraints & Indexes

```sql
ALTER TABLE customer_order
  ADD CONSTRAINT ux_customer_order__public_id UNIQUE (public_id),
  ADD INDEX ix_customer_order__user_id_created_at (user_id, created_at);
```

**Field note**: Compound indexes follow the naming pattern and mirror query access paths.

### 9.3 Example View Function Trigger

```sql
CREATE VIEW v_order_summary AS
SELECT co.public_id AS order_id,
       u.email AS customer_email,
       co.amount_cents,
       co.created_at
FROM customer_order co
JOIN user u ON co.user_id = u.id;

CREATE FUNCTION fn_order_total_in_usd(order_id BIGINT UNSIGNED)
RETURNS DECIMAL(12,2)
DETERMINISTIC
BEGIN
  DECLARE amount DECIMAL(12,2);
  SELECT amount_cents / 100 INTO amount
  FROM customer_order
  WHERE id = order_id;
  RETURN amount;
END;

CREATE TRIGGER trg_customer_order__before_update
BEFORE UPDATE ON customer_order
FOR EACH ROW
SET NEW.updated_at = NOW(6);
```

**Field note**: The view, function, and trigger names make their roles obvious during incident response.

### 9.4 Example Migration File

```
20250401090000__create_customer_order.sql
```

```sql
-- Up
CREATE TABLE customer_order (...);

-- Down
DROP TABLE customer_order;
```

**Field note**: Keep rollback instructions next to the forward migration to reduce panic during rollbacks.

---

## Cheat Sheet (Quick Reference)

| Area | Pattern | Example |
|------|---------|---------|
| Database | `<company>_<domain>[_<env>]` | `acme_billing_prod` |
| Table | `lower_snake_case` singular | `customer_order` |
| Column | `lower_snake_case` with units or intent | `amount_cents`, `is_active` |
| Primary key | `pk_<table>` | `pk_customer` |
| Foreign key | `fk_<table>__<column>__<ref>` | `fk_invoice__user_id__user` |
| Unique | `ux_<table>__<cols>` | `ux_user__email` |
| Index | `ix_<table>__<cols>` | `ix_order__created_at` |
| View | `v_<purpose>` | `v_customer_revenue` |
| Procedure | `sp_<verb>_<noun>` | `sp_archive_invoice` |
| Function | `fn_<noun>_<qualifier>` | `fn_order_total_in_usd` |
| Trigger | `trg_<table>__<timing>_<event>` | `trg_user__after_insert` |
| History | `<table>_hist` | `payment_hist` |
| Archive | `<table>_archive` | `invoice_archive` |
| Temp table | `tmp_<purpose>` | `tmp_monthly_totals` |
| Migration file | `YYYYMMDDHHMMSS__description.sql` | `20250315093045__create_customer.sql` |

---

## Conclusion

### 11.1 Why These Conventions Work in Real-World Scale

* They have been battle-tested in companies that manage petabytes of data.
* They reduce onboarding time for new team members.
* They make automated tooling (linters, migration frameworks) reliable.

### 11.2 How to Enforce Them (Linters CICD Code Review)

* Add schema linters to CI (e.g., SQLFluff with custom rules).
* Use pull request templates that include a naming checklist.
* Run migrations against staging replicas before merging.

### 11.3 The Golden Rule → Consistency Beats Cleverness

* When in doubt, pick the name that reads the same way to every teammate.
* Document any deliberate exceptions and explain the reasoning.
* Review this guide quarterly to keep it aligned with business needs.

**Field note**: Culture, not tooling, keeps standards alive. Teach the reasoning in onboarding sessions so teams stay aligned.
