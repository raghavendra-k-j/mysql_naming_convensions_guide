# MySQL Naming Conventions Guide

This single-page guide explains how to name every part of a MySQL schema in a way that scales for global teams. Each chapter builds on the previous one, uses simple English, and shows why the rule matters, what breaks when it is ignored, and how to apply it with clear SQL snippets. Treat the guide like a W3Schools tutorial: read one chapter at a time, pause to compare with your schema, and return whenever you need a refresher.

## Table of Contents

- [Chapter 1 – Introduction](#chapter-1)
- [Chapter 2 – Core Naming Rules](#chapter-2)
- [Chapter 3 – Database-Level Conventions](#chapter-3)
- [Chapter 4 – Table & Column Naming](#chapter-4)
- [Chapter 5 – Constraints & Indexes](#chapter-5)
- [Chapter 6 – Views, Routines & Triggers](#chapter-6)
- [Chapter 7 – Migrations & Versioning](#chapter-7)
- [Chapter 8 – Additional Conventions That Pay Off at Scale](#chapter-8)
- [Chapter 9 – Examples & Patterns in Action](#chapter-9)
- [Chapter 10 – Cheat Sheet (Quick Reference)](#chapter-10)
- [Chapter 11 – Conclusion](#chapter-11)

---

<a id="chapter-1"></a>
## Chapter 1 – Introduction


---

This opening chapter explains why naming rules matter for MySQL, the common problems that appear when teams ignore them, and the goals that guide the rest of the standard. The tone is practical and the language is simple so everyone on the team can follow along.

### 1.1 Why Naming Conventions Matter in Scalable MySQL Projects

#### The challenge
MySQL lets you name databases, tables, and columns however you want. That freedom sounds helpful, but it creates chaos when different people make different choices. Over time the schema becomes unpredictable, which slows down product work.

#### The recommended practice
Agree on a single naming playbook before you create or change any schema objects. Document it, share it, and review code against it. When every table follows the same pattern, teammates can read each other's work without guessing intent.

#### What goes wrong without it
- Automation scripts fail because they cannot guess which column contains the data they need.
- Reviewers spend time asking for explanations instead of checking business logic.
- Integrations break silently when two tables use similar names for different ideas.

#### Example
```sql
-- Without a shared convention
SELECT *
FROM Customers c
JOIN CUSTOMERDETAILS cd ON c.CustID = cd.CustomerID;

-- With an agreed convention
SELECT c.id, c.full_name, cp.locale
FROM customer c
JOIN customer_profile cp ON c.id = cp.customer_id;
```
The second query is shorter, easier to scan, and obvious about the relationships.

> **Field Note:** A naming guide is not about control. It keeps the team fast by removing the daily friction of decoding someone else's style.

### 1.2 Common Pain Points in Poorly Named Schemas

#### The challenge
When names drift, engineers, analysts, and data pipelines are forced to translate meaning manually. This creates bugs and increases cognitive load.

#### Warning signs to watch for
- **Duplicate concepts:** `order`, `orders`, and `tbl_order` all refer to the same entity.
- **Mystery columns:** `flag1` or `data_1` hide the real purpose of the field.
- **Changing names:** API consumers break when a column changes from `UserID` to `user_id` without notice.
- **Unclear relationships:** Foreign keys named `ref1` or `fk_user` do not show which tables they connect.

#### Example
```sql
-- Hard to understand
SELECT flag1
FROM tbl_order
WHERE status_f = 'A';

-- Clear and predictable
SELECT is_expedited
FROM `order`
WHERE status = 'approved';
```
The clear version uses real words. Anyone can explain what it does without asking the original author.

> **Field Note:** Confusing names slow down onboarding. New teammates need weeks to learn the hidden language before they can contribute.

### 1.3 Goals of This Standard (Consistency, Clarity, Scalability, Lintability)

#### The four pillars
1. **Consistency:** Similar things look the same everywhere in the schema. Reviewers can spot mistakes immediately.
2. **Clarity:** Names describe the actual business meaning. No mental translation is required.
3. **Scalability:** The rules keep working when you add more services, more data, and more people.
4. **Lintability:** Automated checks can verify the rules, which prevents regressions during fast releases.

#### How the pillars work together
- Consistency unlocks clarity because everyone recognises common patterns.
- Clarity supports scalability because downstream tools can rely on predictable names.
- Lintability protects both consistency and clarity by catching bad names before they land in production.

#### Practical example
```text
Good: payment_transaction
- Consistent: uses lower_snake_case
- Clear: joins two business words
- Scalable: leaves room for related tables like payment_receipt
- Lintable: a rule can check the pattern automatically

Bad: payTrxTbl
- Inconsistent casing and abbreviations
- Unclear purpose
- Hard to validate with tooling
```

> **Field Note:** The pillars act like a checklist during reviews. If a proposed name fails any one of them, the team should pause and fix it before merging.

---

<a id="chapter-2"></a>
## Chapter 2 – Core Naming Rules


This chapter explains the everyday rules that keep a MySQL schema readable. Each rule is written in simple English, but the ideas go deep so your team can apply them with confidence.

### 2.1 Case & Style → Why lower_snake_case Wins

**Rule.** Write every database name, table name, column name, index name, and routine name in `lower_snake_case`.

**Why this matters.**
- MySQL is case-sensitive on some operating systems. Using a single style removes "works on my machine" bugs.
- Snake case keeps multiword names readable without guessing where one word ends.
- Lowercase avoids mixed-case typos, especially when people copy names into shell scripts or config files.

**What goes wrong without the rule.**
- `CustomerOrders` may be stored as `customerorders` on Linux but not on macOS, causing migrations to fail in production.
- `UserID` and `userId` look similar yet MySQL treats them as different identifiers in case-sensitive setups, leading to duplicates.
- Query builders and ORMs often default to lowercase table names. Mixing cases forces awkward overrides in every service.

**How to apply it.**
- Convert every word to lowercase and join with underscores: `customer_order_items`.
- Avoid hyphens (`-`) because they require quoting and can be mistaken for subtraction.
- Use consistent pluralization rules (see Section 2.4 for table names).

**Quick example.**
```sql
-- Good
SELECT order_total FROM customer_orders WHERE customer_id = 42;

-- Risky: camelCase and hyphenated names require quoting and may break tools
SELECT `orderTotal` FROM `Customer-Orders` WHERE `CustomerID` = 42;
```

### 2.2 Language → Why Full Words Beat Abbreviations

**Rule.** Use full, descriptive words in names instead of shorthand.

**Why this matters.**
- Future teammates or auditors may not know your internal acronyms.
- Full words make search across the codebase easier (`shipping_address` matches plain English).
- You write names once but read them hundreds of times. Saving one or two characters is not worth the confusion.

**What goes wrong without the rule.**
- `cust_addr` leaves new engineers guessing whether the column holds billing or shipping data.
- Abbreviations drift over time: one team uses `amt`, another uses `amount`, and joins become harder to read.
- External partners cannot map unclear column names to their API fields, causing integration delays.

**How to apply it.**
- Spell out domain concepts: `expiration_date`, `tracking_number`, `discount_percentage`.
- If the term is truly long, create a glossary entry in the documentation so everyone uses the same word.
- Only shorten words that are universally understood (e.g., `id`, `url`, `ip`).

**Quick example.**
```sql
-- Good: the meaning is clear to anyone reading the schema
ALTER TABLE invoice_payments ADD COLUMN payment_reference TEXT NOT NULL;

-- Risky: "ref" and "amt" require prior tribal knowledge
ALTER TABLE inv_pay ADD COLUMN pay_ref TEXT NOT NULL;
```

### 2.3 Acronyms & Abbreviations → How to Use Them When Necessary

**Rule.** When you must use an acronym, keep it consistent, uppercase the letters inside the snake case, and document its meaning once.

**Why this matters.**
- Some acronyms (such as `VAT`, `SKU`, `API`) are industry standards, and using them keeps names short yet clear.
- Consistent formatting (`vat_rate`, `api_client_id`) prevents the same acronym from appearing as `Vat` in one place and `VAT` in another.
- Documented acronyms reduce onboarding time and ensure code search returns all related objects.

**What goes wrong without the rule.**
- `vat`, `Vat`, and `value_added_tax` may all exist in one schema, forcing developers to guess which column to join.
- Undocumented short forms (for example, `prc` for price) break analytics queries and BI dashboards that expect human-readable names.
- Mixed casing (like `Sku` vs `SKU`) triggers false negatives in grep searches and leads to duplicate columns during migrations.

**How to apply it.**
- Treat the acronym as a single word inside the snake case: `vat_number`, `api_key`.
- Record every approved acronym in a `docs/glossary.md` (create one if it does not exist yet).
- If two acronyms collide (`id` vs `invoice_id`), prefer the longer but clearer name.

**Quick example.**
```sql
-- Good: consistent style with documented acronyms
CREATE TABLE inventory_items (
    id BIGINT UNSIGNED PRIMARY KEY,
    sku VARCHAR(64) NOT NULL,
    vat_rate DECIMAL(5,2) NOT NULL
);

-- Risky: inconsistent casing and unexplained shorthand
CREATE TABLE inventory (
    ID BIGINT PRIMARY KEY,
    SkuCode VARCHAR(64) NOT NULL,
    v_rate DECIMAL(5,2) NOT NULL
);
```

### 2.4 Singular vs Plural Table Names → Why Singular Is Less Ambiguous

**Rule.** Name tables with singular nouns (`customer`, `order_item`) while keeping columns plural only when they store collections (such as JSON arrays).

**Why this matters.**
- A table represents the concept of one entity. Singular names match how we talk about rows: "this order" rather than "this orders".
- Singular names make joins read like sentences: `order JOIN customer ON order.customer_id = customer.id`.
- ORMs and code generators often assume singular table names when deriving class names.

**What goes wrong without the rule.**
- Mixed pluralization (`users` table joined to `order_item`) produces awkward SQL and confuses naming in model classes.
- English irregular plurals (`person` vs `people`) create mistakes in migrations and API responses.
- BI tools auto-generate field names by singularizing. If the table is already plural, the tool guesses wrong (`statuss`).

**How to apply it.**
- Use the domain noun in singular form: `invoice`, `invoice_payment`, `shipment_event`.
- For join tables, combine the two singular nouns alphabetically: `order_product` instead of `orders_products`.
- Document any intentional exception (such as `analytics` for a pre-aggregated table) so future maintainers know it is special.

**Quick example.**
```sql
-- Good: joins read naturally
SELECT customer.name, order_total.amount
FROM customer
JOIN order_total ON order_total.customer_id = customer.id;

-- Risky: mixed pluralization makes the query harder to read
SELECT customers.name, orders_totals.amount
FROM customers
JOIN orders_totals ON orders_totals.customer_id = customers.id;
```

### 2.5 Avoiding Noise Prefixes (tbl_, col_) → Why They Harm More Than Help

**Rule.** Do not prefix table names with `tbl_` or column names with `col_`. Use the object name itself to show what it is.

**Why this matters.**
- Schema objects already tell you their type: MySQL knows `customer` is a table, so `tbl_customer` repeats information.
- Noise prefixes make names longer and harder to scan in diagrams or queries.
- Removing prefixes later is painful because every application query must be updated.

**What goes wrong without the rule.**
- Developers forget the prefix and create both `tbl_order` and `order`, causing duplicated data structures.
- BI tools that alphabetically list tables will group all `tbl_...` together, hiding related tables from quick view.
- Prefix-heavy names hide meaningful words when truncated in dashboards or logs.

**How to apply it.**
- Name the table for the entity (`payment_method`) and the column for the attribute (`expiration_month`).
- Use suffixes only when they add meaning (for example, `_history`, `_archive`, `_log`).
- Review new migrations during code review to block regressions.

**Quick example.**
```sql
-- Good: no noise prefixes, meaning is obvious
CREATE TABLE payment_method (
    id BIGINT UNSIGNED PRIMARY KEY,
    customer_id BIGINT UNSIGNED NOT NULL,
    card_last_four CHAR(4) NOT NULL
);

-- Risky: prefixes make the schema harder to maintain
CREATE TABLE tbl_payment_method (
    col_id BIGINT UNSIGNED PRIMARY KEY,
    col_customer_id BIGINT UNSIGNED NOT NULL,
    col_card_last_four CHAR(4) NOT NULL
);
```

### 2.6 Reserved Words → Why They Create Long-Term Bugs

**Rule.** Never use MySQL reserved words or SQL keywords as identifiers unless they are suffixed or prefixed with another word.

**Why this matters.**
- Reserved words like `order`, `select`, and `group` require backticks every time you query them.
- Backticks hide bugs in application code because linters and editors cannot easily detect mis-typed keywords.
- Upgrading MySQL can add new keywords, turning a previously "safe" name into a breaking change.

**What goes wrong without the rule.**
- An `order` table forces every query to be written as ``SELECT * FROM `order`;``. Forgetting the backticks causes syntax errors in production.
- ORMs may silently rename or escape reserved words differently, creating inconsistent migrations across services.
- Reserved names collide with other systems (for example, a REST API expecting `/orders` while the table is `` `order` ``).

**How to apply it.**
- Check MySQL's official reserved word list before naming tables or columns. When in doubt, append a clarifying word: `customer_order`, `status_code`.
- Add linting to migrations (using tools like `sqlfluff` or custom scripts) to block reserved words during code review.
- Keep a shared glossary of approved names so new tables reuse proven, safe words.

**Quick example.**
```sql
-- Good: avoids the reserved word by adding context
CREATE TABLE customer_order (
    id BIGINT UNSIGNED PRIMARY KEY,
    customer_id BIGINT UNSIGNED NOT NULL,
    status_code VARCHAR(32) NOT NULL
);

-- Risky: requires quoting forever and may break tooling
CREATE TABLE `order` (
    `id` BIGINT UNSIGNED PRIMARY KEY,
    `customer_id` BIGINT UNSIGNED NOT NULL,
    `status` VARCHAR(32) NOT NULL
);
```

> **Field Note:** Revisit these rules whenever you design a new schema. Consistency at this level prevents painful refactors later.

---

<a id="chapter-3"></a>
## Chapter 3 – Database-Level Conventions


This chapter focuses on the database names themselves. Before you create tables, you should decide how to label each database so every tool, environment, and teammate understands its purpose at a glance.

### 3.1 Naming Databases by Product and Context

**Rule.** Combine the product or bounded context with the primary function when naming a database. Example pattern: `<product>_<area>` such as `billing_service` or `analytics_reporting`.

**Why this matters.**
- Readers know instantly which team owns the data and what problem the schema solves.
- Infrastructure tools can apply permissions automatically based on predictable prefixes.
- When a company has many services, descriptive database names prevent collisions.

**What goes wrong without the rule.**
- Generic names like `db1` or `main` leave on-call engineers guessing which workload they are touching.
- Security teams cannot scope access correctly because nothing in the name signals the business area.
- Backups and restores take longer because operators must inspect the data before knowing which database to recover.

**How to apply it.**
1. List the product, service, or bounded context (for example, `billing`, `support`, `identity`).
2. Add a short word that explains the database role: `service`, `reporting`, `warehouse`, `archive`.
3. Join the words with underscores and keep them lowercase.
4. Document the pattern in the engineering handbook so every new database follows it.

**Practical example.**
```sql
-- Clear, product-focused names
CREATE DATABASE billing_service;
CREATE DATABASE billing_reporting;

-- Risky, non-descriptive names
CREATE DATABASE db_main;
CREATE DATABASE billing2;
```
The clear pair tells operators which database powers transactions and which feeds BI dashboards.

> **Field Note:** Treat the database name like a customer-facing URL. If someone outside your team can guess its purpose, you chose a good name.

### 3.2 Handling Multiple Environments (dev, stg, prod) Safely

**Rule.** Extend the base name with an environment suffix such as `_dev`, `_stg`, or `_prod`. Keep the suffix short, consistent, and documented.

**Why this matters.**
- Engineers instantly know whether they are touching production data.
- Automated pipelines can deploy to the right environment without manual switches.
- Support teams can reproduce bugs by pointing tools at the matching environment name.

**What goes wrong without the rule.**
- Developers connect to the wrong database because `billing` and `billing_backup` look similar in a connection list.
- Scripts run destructive queries on production because the environment was hidden in a config file instead of the database name.
- Cloud providers sometimes list databases alphabetically. Without consistent suffixes, related environments drift apart and mistakes happen.

**How to apply it.**
1. Start with the base product name from Section 3.1 (for example, `billing_service`).
2. Append a short environment suffix: `_dev`, `_test`, `_stg`, `_prod`.
3. Use the same suffix order in every project so lists stay grouped.
4. Add monitoring alerts that look for queries to `_prod` from local developer machines.

**Practical example.**
```sql
-- Environment-aware names
CREATE DATABASE billing_service_dev;
CREATE DATABASE billing_service_stg;
CREATE DATABASE billing_service_prod;

-- Risky mix of styles
CREATE DATABASE billingDev;
CREATE DATABASE billing_stage;
CREATE DATABASE prod_billing;
```
The consistent suffix keeps the three environments grouped and leaves no doubt about their purpose.

> **Field Note:** When someone copies a connection string into a chat, the suffix acts as a final warning before anyone runs a query.

### 3.3 Why Database Names Should Be Predictable

**Rule.** Document a naming map so every integration, script, and team knows the exact database name before connecting. Predictability is more important than creativity.

**Why this matters.**
- Automation relies on string matching. If names follow a pattern, CI/CD jobs, backup scripts, and monitoring tools work without special cases.
- Predictable names reduce onboarding time because new hires can guess where to find data.
- Shared understanding prevents accidental schema duplication when teams spin up new services.

**What goes wrong without the rule.**
- Two teams might create `billing_service` and `billing_services`, each thinking the other name was taken.
- Auditors waste time matching database exports to business areas because nothing links the names.
- Renaming a database to fix confusion breaks connection pools, secrets management, and firewall rules.

**How to apply it.**
1. Record every database name and owner in a shared registry or `docs/database-inventory.md` file.
2. Create a checklist for new services that requires reserving a database name before writing code.
3. Add automated tests in deployment pipelines to verify the target database exists and matches the expected pattern.
4. Review the registry quarterly to retire unused names and free up mental space.

**Practical example.**
```text
Inventory excerpt
- billing_service_prod → Owned by Billing Platform Team
- billing_service_stg → Owned by Billing Platform Team
- billing_reporting_prod → Owned by Data Insights Team
```
This short list gives leadership and auditors instant answers about who is responsible for each data store.

> **Field Note:** Predictable names save the day during an incident. When the pager goes off, responders do not have time to decode clever naming jokes.

---

<a id="chapter-4"></a>
## Chapter 4 – Table & Column Naming


Tables and columns are the backbone of every MySQL schema. This chapter explains how to name them so relationships stay clear,
queries stay predictable, and migrations stay easy even when the database grows to millions of rows.

### 4.1 Primary Keys → Why Surrogate Keys (id) Beat Natural Keys

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

### 4.2 Foreign Keys → Why `<table>_id` Pattern Avoids Confusion

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

### 4.3 Timestamps → Why Every Table Needs `created_at` and `updated_at`

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

### 4.4 Soft Deletes → Why `deleted_at` Beats Boolean Flags

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

### 4.5 Booleans → Why `is_` and `has_` Prefixes Improve Readability

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

### 4.6 Amounts & Units → Why Explicit Units Prevent Ambiguity

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

### 4.7 JSON Columns → Why `_json` Helps Contain Flexibility

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

### 4.8 Enums vs Lookup Tables → Why Lookup Tables Scale Better

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

### 4.9 History, Logs, and Backups → Why Explicit Suffixes Save Time

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

### 4.10 Partitions & Shards → Why Native Partitioning Beats Table Explosion

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

<a id="chapter-5"></a>
## Chapter 5 – Constraints & Indexes


Good table and column names keep data readable, but the story is not complete until constraints and indexes are named just as clearly. When constraint names explain what they guard, error messages become self-explanatory, migrations stay predictable, and on-call engineers can fix problems without guesswork. This chapter walks through each constraint and index pattern so teams can spot issues in seconds instead of hours.

### 5.1 Why Naming Constraints Matters for Debugging

**Rule.** Give every constraint and index a deliberate name that follows a shared pattern. Never let MySQL auto-generate names.

**Why this matters.**
- Error messages include the failing constraint name. If the name is meaningful, the fix is obvious.
- Schema diff tools compare names to decide whether objects changed. Predictable names stop churn in migrations.
- Support engineers can read logs without opening the database diagram.

**What goes wrong without the rule.**
- Auto-generated names like `orders_ibfk_1` hide the relationship, forcing people to dig through metadata tables.
- Renaming a column causes MySQL to rebuild unnamed indexes with new random names, generating noisy diffs.
- Incident responders cannot tell which application rule failed, so they disable the constraint instead of fixing the data.

**How to apply it.**
1. Decide on a prefix for each constraint type (described in the next sections).
2. Include the table name and the important columns, separated with double underscores `__` for readability.
3. Create or alter constraints explicitly with the chosen name instead of relying on defaults.
4. Document any exceptions inside the migration file so the next engineer understands the reason.

**Practical example.**
```sql
ALTER TABLE invoices
    ADD CONSTRAINT fk_invoices__customer_id__customers
        FOREIGN KEY (customer_id) REFERENCES customers (id);
```

> **Field Note:** Clear names turn a 3 a.m. error message into a one-line fix. Vague names turn it into a war room.

### 5.2 Primary Keys (`pk_<table>`)

**Rule.** Name every primary key `pk_<table>` where `<table>` is the table name in singular form.

**Why this matters.**
- Primary keys anchor foreign keys, so the name appears in many error messages.
- Monitoring dashboards can alert on `pk_orders` violations and everyone knows what it means.
- ORMs often try to create their own keys if they cannot see an existing primary key; explicit naming avoids duplication.

**What goes wrong without the rule.**
- Default names like `PRIMARY` do not travel well in schema dumps or cross-environment comparisons.
- Multiple teams alter the same table and accidentally drop and recreate the key because they cannot refer to it by name.
- Cross-database tooling (for example, Debezium) may fail to detect the intended primary key when metadata is inconsistent.

**How to apply it.**
1. When creating the table, declare the primary key with `CONSTRAINT pk_<table>`.
2. Keep the name stable even if the primary key columns change (for example, switching from `INT` to `BIGINT`).
3. Use the singular table name to match the naming pattern used for foreign keys.

**Practical example.**
```sql
CREATE TABLE payment ( 
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    CONSTRAINT pk_payment PRIMARY KEY (id)
);
```

> **Field Note:** A named primary key gives tools and humans a fixed handle, even if the table evolves.

### 5.3 Foreign Keys (`fk_<table>__<column>__<ref_table>`)

**Rule.** Name foreign keys `fk_<table>__<column>__<ref_table>` so the relationship is obvious.

**Why this matters.**
- When a foreign key fails, the error points directly to the referencing table, the column, and the referenced table.
- Code reviewers can spot mismatched relationships at a glance.
- Automated schema checkers can verify that every `_id` column has the matching constraint.

**What goes wrong without the rule.**
- A generic name like `fk_user1` gives no clue which column is wrong.
- Teams end up with duplicate foreign keys because they cannot see that one already exists.
- Deleting records fails during incidents, but on-call engineers waste time guessing which child table is blocking it.

**How to apply it.**
1. Start with the table that holds the foreign key (singular form).
2. Add the foreign key column name, even for multi-column keys (`order_id__line_number`).
3. Finish with the referenced table (singular form).
4. Keep underscores for words but use double underscores between sections: table, column(s), referenced table.

**Practical example.**
```sql
ALTER TABLE order_payment
    ADD CONSTRAINT fk_order_payment__order_id__order
        FOREIGN KEY (order_id) REFERENCES `order` (id);
```

> **Field Note:** The name is a sentence: “foreign key on order_payment.order_id referencing order.” No guessing required.

### 5.4 Unique Constraints (`ux_<table>__<columns>`)

**Rule.** Name unique constraints `ux_<table>__<columns>` to show exactly which fields must stay unique.

**Why this matters.**
- Duplicate data is a common source of bugs; the name identifies the business rule you are protecting.
- When analytics asks “what keeps invoice numbers unique?” you can point to `ux_invoice__number` without opening SQL.
- Migration scripts can drop and recreate constraints safely because the name does not change when column order does.

**What goes wrong without the rule.**
- Auto-generated names differ across environments, so schema comparison tools report false differences.
- Dropping a unique constraint becomes risky because engineers cannot be certain they are targeting the right one.
- Business rules drift when no one remembers why the constraint exists.

**How to apply it.**
1. List the table name in singular form.
2. List the columns in column order, separated by underscores; for readability, join multi-word columns with single underscores (`start_date_end_date`).
3. Use the same name for both the constraint and any supporting index if you explicitly create one.

**Practical example.**
```sql
ALTER TABLE invoice
    ADD CONSTRAINT ux_invoice__number
        UNIQUE (number);
```

> **Field Note:** When someone proposes a change, the constraint name reminds the team of the promise the schema already makes.

### 5.5 Non-Unique Indexes (`ix_<table>__<columns>`)

**Rule.** Name non-unique indexes `ix_<table>__<columns>` and match the order of the indexed columns.

**Why this matters.**
- Explains why the index exists. `ix_payment__status_created_at` signals the query pattern it supports.
- Avoids duplicate indexes with the same columns in a different order.
- Helps performance reviews—DBAs can ask “do we still need this exact index?”

**What goes wrong without the rule.**
- Unnamed indexes are assigned numbers (`invoice_ibfk_2`) that convey nothing about their purpose.
- Developers unknowingly create duplicates, slowing down writes and consuming storage.
- During optimisations, teams drop helpful indexes because they cannot tell what they support.

**How to apply it.**
1. Use the singular table name.
2. List the indexed columns in the order the query uses them; include sorting directions if relevant (`created_at_desc`).
3. Document covering indexes by appending `_covering` when extra columns are included for `SELECT` performance.

**Practical example.**
```sql
CREATE INDEX ix_payment__status_created_at
    ON payment (status, created_at);
```

> **Field Note:** An index name should answer “Which query are we speeding up?” in five seconds.

### 5.6 Check Constraints (`ck_<table>__<rule>`)

**Rule.** For MySQL 8.0 and later, name check constraints `ck_<table>__<rule>` to describe the business rule.

**Why this matters.**
- Check constraints enforce simple rules in the database, keeping bad data out before it causes downstream errors.
- A descriptive name tells reviewers what the rule is without reading the SQL expression.
- Automated testing can verify that critical guards (for example, non-negative balances) exist in every environment.

**What goes wrong without the rule.**
- If the constraint fails with a generic name, developers disable it because they think it is optional.
- When a rule changes, teams create a second overlapping constraint instead of updating the original one.
- Legacy tools ignore unnamed checks, leading to inconsistent behaviour across environments.

**How to apply it.**
1. Summarise the rule in a short phrase, such as `amount_cents_nonneg`.
2. Use verbs sparingly; focus on the condition enforced.
3. Keep the expression in sync with the name so the error message stays trustworthy.

**Practical example.**
```sql
ALTER TABLE invoice
    ADD CONSTRAINT ck_invoice__amount_cents_nonneg
        CHECK (amount_cents >= 0);
```

> **Field Note:** The best constraint names read like a headline: “Invoices cannot have negative amounts.”

### 5.7 Full-Text Indexes (`ft_<table>__<columns>`)

**Rule.** Name full-text indexes `ft_<table>__<columns>` so search-related structures are easy to spot.

**Why this matters.**
- Full-text indexes behave differently from B-tree indexes; naming them clearly prevents accidental misuse.
- Search teams can find all text-search surfaces quickly when tuning relevance.
- Migration scripts can recreate full-text indexes in the correct order during rebuilds.

**What goes wrong without the rule.**
- Developers mistake full-text indexes for regular ones and drop them, breaking search in production.
- Duplicate full-text indexes are created with different stopword configurations, leading to inconsistent results.
- Disaster recovery scripts miss unnamed full-text indexes, causing search features to degrade after restores.

**How to apply it.**
1. Use the singular table name.
2. List the searchable columns, joined with underscores.
3. If using multiple languages or configurations, append the locale (`__en_gb`).

**Practical example.**
```sql
CREATE FULLTEXT INDEX ft_article__title_body
    ON article (title, body);
```

> **Field Note:** When search breaks, the team should find every related index by grepping for `ft_`—clear names make that possible.

### Wrapping Up

Well-named constraints and indexes act like documentation that lives inside the database. They shorten incident response, prevent duplicate structures, and make migrations safer. Adopt these patterns alongside the table and column rules from Chapter 4, and you will have a schema that explains itself to anyone who reads it.

<a id="chapter-6"></a>
## Chapter 6 – Views, Routines & Triggers


Views, stored routines, and triggers sit on top of tables and columns. They shape how data is read and written, so their names must make intent obvious. When names are predictable, engineers can reuse logic safely, review changes faster, and avoid accidental behaviour hidden inside the database. This chapter shows how to name each object type and why the patterns save teams from painful debugging sessions.

### 6.1 Views (`v_<purpose>`)

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

### 6.2 Stored Procedures (`sp_<verb>_<object>`)

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

### 6.3 Functions (`fn_<result>`)

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

### 6.4 Triggers (`trg_<table>__<event>__<timing>`)

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

<a id="chapter-7"></a>
## Chapter 7 – Migrations & Versioning


Naming database objects is only half the story. Teams also need a disciplined way to name the migration files that create, change, and remove those objects. Clear migration names help everyone understand deployment order, audit changes during incidents, and roll back safely when something goes wrong. This chapter explains how to name migration files so large teams can ship changes with confidence.

### 7.1 File Naming → Why Timestamped Names Are Essential for Ordering

**Rule.** Prefix every migration filename with a sortable timestamp (UTC, `YYYYMMDDHHMMSS`) followed by a short, descriptive slug, such as `20240415103000_add_invoice_indexes.sql`.

**Why this matters.**
- Timestamps give an absolute order so that distributed teams apply migrations in the same sequence.
- CI/CD systems can pick up new files by comparing timestamps without reading their contents.
- Auditors and incident responders can line up schema changes with application releases.

**What goes wrong without the rule.**
- Natural-language names sort differently on different operating systems, causing migrations to run in a dangerous order.
- Engineers forget to add numbers manually (`01`, `02`) and end up renumbering files, making pull requests hard to review.
- Re-running migrations on a new environment fails because there is no reliable order to follow.

**How to apply it.**
1. Generate the timestamp in UTC to avoid daylight savings issues.
2. Use a single underscore between the timestamp and the slug.
3. Keep the slug short but specific (`create_payment_table`, `drop_legacy_trigger`).
4. Store the script in a directory dedicated to migrations so tooling can locate it.

**Practical example.**
```
20240415103000_add_invoice_indexes.sql
20240418120000_create_v_invoice_summary.sql
20240422151500_sp_archive_orders.sql
```

> **Field Note:** When a deployment fails, the timestamp instantly tells you which migration to re-run and in what order.

### 7.2 One Action/Object per File → Why Small, Atomic Changes Matter

**Rule.** Keep each migration focused on a single object or closely related set of changes. If you need to alter multiple tables, create multiple files.

**Why this matters.**
- Atomic migrations are easier to review and test.
- Rollbacks become less risky because you only undo the change that failed.
- Merge conflicts shrink because two engineers are less likely to touch the same file.

**What goes wrong without the rule.**
- A giant script that creates tables, triggers, and indexes at once becomes unreviewable.
- Partial deploys leave the schema in a half-changed state that is hard to recover from.
- When a single statement fails, the rest of the script does not run, leaving the database inconsistent.

**How to apply it.**
1. Decide what the migration is responsible for (for example, create a table, add a column, drop a trigger).
2. Write a descriptive slug that matches the action (`add_customer_timezone_column`).
3. If you need to perform follow-up steps (such as backfilling data), place them in a separate script with its own timestamp.
4. Document cross-file dependencies in comments so reviewers understand the sequence.

**Practical example.**
```
20240501100000_create_customer_timezone_column.sql
20240501101000_backfill_customer_timezone.sql
20240501102000_add_customer_timezone_index.sql
```

> **Field Note:** When you keep migrations small, you can roll back one slice without touching other parts of the release.

### 7.3 Rollback & CI/CD Safety

**Rule.** For every migration, plan the rollback path and record it in the file header. Use consistent naming so automated tools can pair forward and backward steps.

**Why this matters.**
- Deployments occasionally fail. A documented rollback lets teams recover without delay.
- CI/CD pipelines can enforce that every forward script has a matching rollback script when names follow a pattern.
- Future engineers know the intent behind the change when they read the file years later.

**What goes wrong without the rule.**
- A hotfix removes a column, but nobody remembers how to rebuild it when the fix fails.
- Rollback scripts are named inconsistently, so the pipeline cannot find them and aborts the deploy.
- New environments drift because initial setup migrations include destructive statements without safety notes.

**How to apply it.**
1. Add a header comment describing the change, the reason, and the rollback steps.
2. If the team uses paired files, follow a shared pattern (for example, `20240502090000_add_index.sql` and `20240502090000_add_index_down.sql`).
3. Test the rollback locally or in staging before merging the migration.
4. Include guards such as `IF EXISTS` and `IF NOT EXISTS` to keep reruns safe.

**Practical example.**
```sql
-- 20240502090000_add_index.sql
-- Adds ix_invoice__customer_id_created_at for dashboard performance.
-- Rollback: drop the index with 20240502090000_add_index_down.sql
ALTER TABLE invoice
    ADD INDEX ix_invoice__customer_id_created_at (customer_id, created_at);
```

```sql
-- 20240502090000_add_index_down.sql
-- Rollback for 20240502090000_add_index.sql
ALTER TABLE invoice
    DROP INDEX ix_invoice__customer_id_created_at;
```

> **Field Note:** Naming rollback files with the same timestamp keeps automation from guessing.

<a id="chapter-8"></a>
## Chapter 8 – Additional Conventions That Pay Off at Scale


These conventions do not fit into a single object type, yet they protect every MySQL project once it starts to grow. Think of them as the safety net around the core naming rules. Each section explains the recommendation, the pain you avoid by following it, and a simple way to put the idea into practice.

### 8.1 Character Sets & Collations → Why utf8mb4 Is the Only Safe Default

**Rule.** Set every database, table, and column that stores text to use the `utf8mb4` character set with a modern collation such as `utf8mb4_unicode_ci` (case-insensitive) or `utf8mb4_0900_ai_ci` (MySQL 8+).

**Why this matters.**
- `utf8mb4` stores the full Unicode range, including emoji and multilingual names.
- A consistent collation keeps string comparisons predictable across the schema.
- Tooling (ORMs, ETL jobs, BI tools) assumes Unicode support and fails when it is missing.

**What goes wrong without the rule.**
- The legacy `utf8` charset silently truncates four-byte characters such as emoji or some Asian scripts.
- Mixed collations (`utf8_general_ci`, `latin1_swedish_ci`) make joins and sorting inconsistent, leading to hidden data bugs.
- Different collations block index usage because MySQL must perform conversions during comparisons.

**How to apply it.**
1. Set the database default: `CREATE DATABASE app_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`
2. Repeat the setting on each table in case the database default changes later.
3. Declare character sets on text columns explicitly to protect against accidental drift in migrations.

**Practical example.**
```sql
CREATE TABLE customer (
    id BIGINT UNSIGNED PRIMARY KEY,
    full_name VARCHAR(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
    email VARCHAR(320) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL
) CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

> **Field Note:** Aligning character sets from the start prevents a painful rewrite when you expand to international markets.

### 8.2 Time Handling → Why Always Store UTC with `_at`

**Rule.** Store all timestamps in UTC, name the columns with an `_at` suffix, and convert to local time only in the application or reporting layer.

**Why this matters.**
- UTC avoids daylight saving issues and regional time zone confusion.
- The `_at` suffix signals that the column records a point in time, not just a date.
- Logging, analytics, and cross-service integrations can compare timestamps without extra conversion rules.

**What goes wrong without the rule.**
- Local time storage causes gaps or duplicate times when clocks move forward or backward.
- Engineers misread columns like `created` or `event_time` and apply incorrect logic.
- External systems reject data when they receive ambiguous or timezone-free timestamps.

**How to apply it.**
1. Declare timestamp columns as `DATETIME` or `TIMESTAMP` and add `_at` (`created_at`, `processed_at`).
2. Normalize incoming times to UTC in the application before writing to MySQL.
3. Document how clients should convert back to local time for display.

**Practical example.**
```sql
ALTER TABLE payment
    ADD COLUMN processed_at DATETIME NOT NULL COMMENT 'UTC timestamp when the payment gateway confirmed processing';
```

> **Field Note:** When an incident spans services in different time zones, consistent `_at` columns allow fast event correlation.

### 8.3 DATETIME vs TIMESTAMP → Differences, Trade-offs, and Best Practices

**Rule.** Use `TIMESTAMP` when you need automatic `CURRENT_TIMESTAMP` behaviour and the stored range fits (1970–2038 UTC). Use `DATETIME` for anything else.

**Why this matters.**
- `TIMESTAMP` values are stored in UTC and auto-convert with time zones when MySQL knows the session offset.
- `DATETIME` stores the exact value you write, preserving historical dates and future schedules.
- Choosing deliberately prevents silent wraparound or conversion bugs.

**What goes wrong without the rule.**
- Using `TIMESTAMP` for far-future dates (like subscription renewals) causes overflow after 2038.
- Using `DATETIME` for audit columns and relying on `DEFAULT CURRENT_TIMESTAMP` fails because the default does not update automatically.
- Mixing both types within the same concept confuses ORMs and data warehouse jobs.

**How to apply it.**
1. Pick `TIMESTAMP` for operational audit columns (`created_at`, `updated_at`) when they stay within the supported range.
2. Pick `DATETIME` for business events (contract start dates, scheduled reminders) that may go beyond 2038.
3. State the choice in table comments so future migrations keep the same type.

**Practical example.**
```sql
CREATE TABLE subscription (
    id BIGINT UNSIGNED PRIMARY KEY,
    customer_id BIGINT UNSIGNED NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    renews_at DATETIME NOT NULL COMMENT 'UTC time when the subscription renews after 2038-safe range'
);
```

> **Field Note:** Teams that mix `DATETIME` and `TIMESTAMP` at random often hit bugs only after the system has been live for years.

### 8.4 Explicit Types & Lengths → Why `VARCHAR(320)` Beats `TEXT`

**Rule.** Define precise data types and lengths that match the business meaning instead of falling back to `TEXT` or overly wide defaults.

**Why this matters.**
- Clear limits stop bad data at the database edge instead of leaking into the application.
- Precise types keep indexes smaller and faster.
- Reviewers can understand intent quickly when the length communicates business rules (for example, 320 characters for email addresses).

**What goes wrong without the rule.**
- `TEXT` columns cannot use some indexes without extra steps, slowing down lookups.
- Unlimited lengths hide data quality problems until the application crashes or external partners reject payloads.
- ORMs map generic types differently, creating inconsistent schemas across services.

**How to apply it.**
1. Research the real-world maximum (email addresses: 320 characters; ISO country codes: 2 letters).
2. Document the reason for any unusual length in a column comment.
3. Add check constraints when the value has a predictable format (for example, length or pattern checks).

**Practical example.**
```sql
ALTER TABLE customer
    MODIFY email VARCHAR(320) NOT NULL COMMENT 'Maximum length defined by RFC 5321';
```

> **Field Note:** When lengths match business rules, developers spend less time validating data in every service.

### 8.5 Nullability → Why `NOT NULL` Should Be Default

**Rule.** Declare columns as `NOT NULL` unless you have a clear, documented reason to allow missing values.

**Why this matters.**
- `NOT NULL` removes a full class of bugs where code forgets to handle `NULL`.
- Queries run faster because MySQL can store fixed-length rows and skip nullable checks.
- Reviewers can tell that every row is expected to have a value, improving data trust.

**What goes wrong without the rule.**
- Analytics dashboards show gaps because `NULL` values propagate through calculations.
- Application code becomes littered with `IFNULL()` or conditional logic to handle missing values.
- Indexes on nullable columns can produce unexpected ordering and filtering behaviour.

**How to apply it.**
1. Start with `NOT NULL` on new columns and provide a default if necessary.
2. When `NULL` is meaningful, explain it in a comment (for example, `deleted_at DATETIME NULL -- NULL means record is active`).
3. Backfill existing rows before flipping a column from nullable to `NOT NULL` to avoid migration failures.

**Practical example.**
```sql
ALTER TABLE invoice
    ADD COLUMN due_at DATETIME NOT NULL COMMENT 'Every invoice must have a payment deadline';
```

> **Field Note:** Treating `NOT NULL` as the norm forces teams to think through nullable business cases deliberately.

### 8.6 IDs → Why Dual IDs (`id` + `public_id`) Improve Internal/External Safety

**Rule.** Use an internal numeric `id` as the primary key and expose a separate `public_id` (often a UUID or prefixed token) for external clients.

**Why this matters.**
- Internal IDs remain short, fast, and efficient for joins and indexing.
- Public IDs can follow business-friendly patterns without revealing internal counts.
- Security reviews approve systems faster when sequential IDs are not exposed to customers.

**What goes wrong without the rule.**
- Guessable IDs allow customers to fetch other users’ records by incrementing numbers.
- Migrations that change primary key type break API contracts when the same field is shared everywhere.
- Data warehouse exports leak sensitive information because internal and public IDs are the same.

**How to apply it.**
1. Keep `id BIGINT UNSIGNED AUTO_INCREMENT` as the clustered primary key.
2. Add `public_id CHAR(27)` or `CHAR(36)` for UUIDs and enforce uniqueness with `UNIQUE KEY ux_<table>__public_id`.
3. Generate the public ID in the application or with a trigger, and document the format.

**Practical example.**
```sql
CREATE TABLE api_token (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    public_id CHAR(27) NOT NULL COMMENT 'External token id like tok_abc123',
    user_id BIGINT UNSIGNED NOT NULL,
    created_at DATETIME NOT NULL,
    UNIQUE KEY ux_api_token__public_id (public_id)
);
```

> **Field Note:** Dual identifiers let you rotate public tokens without touching the relational core of the schema.

### 8.7 Naming Collisions → Why Extending Reserved Names Prevents Bugs

**Rule.** Avoid using bare reserved words or overloaded business terms. Extend the name with a qualifier that clarifies the meaning.

**Why this matters.**
- Reserved words (`order`, `group`, `user`) require quoting, which clutters queries and breaks ORM generators.
- Clear qualifiers (`order_number`, `group_account`) reduce confusion between similar concepts.
- Shared schemas across services stay compatible when the names are explicit.

**Common collision scenarios.**
- **Reserved words:** `order`, `select`, `index`, `key`.
- **Overloaded terms:** `status`, `type`, `name` when multiple meanings exist.
- **Cross-service reuse:** Two services calling different objects `account` even though one tracks customers and the other tracks billing entities.

**What goes wrong without the rule.**
- Developers wrap names in backticks and eventually forget, causing production SQL errors.
- Automated tools (report builders, ORM migrations) fail because they do not escape identifiers consistently.
- Teams misinterpret columns, leading to bugs during data integrations.

**How to apply it.**
1. Review new tables and columns against the MySQL reserved word list.
2. Add a meaningful suffix or prefix (`order_record`, `group_membership`, `status_code`).
3. Document naming decisions in pull requests so other teams reuse the same pattern.

**Practical example.**
```sql
-- Unsafe
CREATE TABLE `order` (
    `id` BIGINT UNSIGNED PRIMARY KEY
);

-- Safe
CREATE TABLE sales_order (
    id BIGINT UNSIGNED PRIMARY KEY
);
```

> **Field Note:** A few extra letters now save hours of debugging when SQL clients treat reserved words differently.

### 8.8 API Alignment → Why DB Schema Should Mirror API Fields

**Rule.** When an API exposes data from the database, align the column names with the API fields wherever practical.

**Why this matters.**
- Engineers can trace a request from the API to the database without translation tables.
- Documentation stays consistent because the same name appears in backend and frontend guides.
- Analytics teams reduce mapping errors when exporting data for dashboards.

**What goes wrong without the rule.**
- Application layers create fragile translation maps (`db_user_name` → `displayName`) that drift over time.
- Debugging becomes slower because logs and queries use different vocabulary.
- Integrations break when API changes do not cascade to the database schema.

**How to apply it.**
1. Confirm that the API name is clear and follows the core naming rules.
2. Use that same name for the database column, adjusting only for MySQL style (snake case, `_id`, `_at`).
3. Record exceptions (for example, legacy fields) in the API documentation so everyone knows the mapping.

**Practical example.**
```sql
-- API field: billing_address_line1
ALTER TABLE invoice
    ADD COLUMN billing_address_line1 VARCHAR(120) NOT NULL;
```

> **Field Note:** Aligning names across layers removes translation bugs when incident response teams search logs and tables simultaneously.

### 8.9 Archival Tables (`_archive`) → Why Separation Improves Performance

**Rule.** Move cold or historical data into tables with an `_archive` suffix once it is no longer needed for day-to-day queries.

**Why this matters.**
- Hot tables stay small, so primary queries remain fast and indexes stay efficient.
- Archival tables keep history for audits without slowing down regular operations.
- The suffix signals to everyone that the table has different retention and backup rules.

**What goes wrong without the rule.**
- Production tables grow endlessly, causing queries and backups to slow down.
- Engineers accidentally modify historical rows because they cannot distinguish them from live data.
- Storage costs spike when no one knows what can be purged safely.

**How to apply it.**
1. Create an archival table with the same structure as the live table plus metadata columns (`archived_at`).
2. Use scheduled jobs to move old rows and delete them from the live table.
3. Update documentation to explain the retention policy and how to query the archive.

**Practical example.**
```sql
CREATE TABLE invoice_archive LIKE invoice;
ALTER TABLE invoice_archive
    ADD COLUMN archived_at DATETIME NOT NULL;
```

> **Field Note:** When performance issues appear, having `_archive` tables lets you trim hot data without losing compliance history.

### 8.10 Temporary Tables (`tmp_...`) → Why Explicit Naming Avoids Accidents

**Rule.** Prefix ad-hoc or session-level temporary tables with `tmp_` and drop them as soon as you finish using them.

**Why this matters.**
- The prefix warns everyone that the table is disposable and not part of the core schema.
- Automated cleanup scripts can find and remove leftover temp tables safely.
- Monitoring dashboards can filter out these tables when checking schema drift.

**What goes wrong without the rule.**
- Forgotten temporary tables stick around, confusing schema diff tools.
- Another engineer mistakes a temporary table for production data and writes queries against it.
- Naming collisions occur when two processes create helper tables with the same name.

**How to apply it.**
1. Choose names like `tmp_customer_backfill_202405` when running maintenance scripts.
2. Add comments inside the script reminding the operator to drop the table.
3. Use MySQL temporary tables (`CREATE TEMPORARY TABLE tmp_name ...`) when possible so they auto-drop on session close.

**Practical example.**
```sql
CREATE TEMPORARY TABLE tmp_customer_backfill_202405 AS
SELECT id, email FROM customer WHERE marketing_opt_in = 1;
-- Process rows...
DROP TEMPORARY TABLE IF EXISTS tmp_customer_backfill_202405;
```

> **Field Note:** Consistent prefixes allow DBAs to search for `tmp_%` and remove stale tables after incident drills.

### 8.11 Consistency Over Cleverness → Why Exceptions Must Be Documented

**Rule.** Stick to the established naming patterns unless a documented business requirement forces an exception.

**Why this matters.**
- Consistency reduces cognitive load for every engineer reviewing queries or migrations.
- Exceptions become a shared decision, not a personal preference.
- Documentation helps newcomers understand why a pattern differs in one area.

**What goes wrong without the rule.**
- Developers introduce creative names that break automation scripts and style guides.
- Reviewers waste time arguing about style instead of focusing on data correctness.
- Future migrations copy the exception accidentally, spreading inconsistent names.

**How to apply it.**
1. When a deviation is unavoidable, explain the reason in the migration or table comment.
2. Add the exception to internal docs or the repository wiki with the agreed-upon context.
3. Revisit exceptions regularly to see if they can be removed after the temporary need passes.

**Practical example.**
```sql
-- Exception: Legacy partner requires camelCase column name
ALTER TABLE partner_ledger
    ADD COLUMN partnerLedgerId VARCHAR(40) NOT NULL COMMENT 'Exception agreed with Partner API v1';
```

> **Field Note:** Writing down exceptions signals that the rest of the schema should keep following the rules.

### 8.12 Defaults: Database vs Application — When to Use Which

**Rule.** Let the database manage mechanical defaults (timestamps, auto-increment IDs) and keep business defaults (status codes, feature flags) in the application layer.

**Why this matters.**
- Database defaults ensure consistent system-generated values such as `created_at` without depending on the application.
- Business logic in the application stays transparent and version-controlled.
- Splitting responsibilities avoids magic values that nobody remembers.

**What goes wrong without the rule.**
- Setting business defaults in the database hides behaviour from code reviewers and leads to surprises in new services.
- Relying on the application for `created_at` timestamps causes missing values when background jobs forget to set them.
- Overusing defaults like `'0000-00-00'` or empty strings makes it hard to tell real data from placeholders.

**How to apply it.**
1. Use database defaults for audit columns (`DEFAULT CURRENT_TIMESTAMP`) and surrogate keys (`AUTO_INCREMENT`).
2. Set business statuses in the application after validating the rules.
3. Avoid placeholder defaults that do not represent real data; prefer `NOT NULL` with meaningful values.

**Practical example.**
```sql
CREATE TABLE feature_flag (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    key_name VARCHAR(120) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) NOT NULL COMMENT 'Set by application after validation'
);
```

> **Field Note:** Clear responsibility for defaults keeps your schema portable across services and environments.

<a id="chapter-9"></a>
## Chapter 9 – Examples & Patterns in Action


This chapter turns the naming rules into a working example. We build a small commerce data model with three tables, wire up the constraints, add a reporting view, write a helper function and trigger, and finish with a migration script. Each section explains why the names follow the standard and what would go wrong if we cut corners.

### 9.1 Example Schema: `user`, `currency`, `order`

#### Design goals
- Show how lowercase snake case keeps object names readable.
- Demonstrate consistent suffixes such as `_id`, `_at`, and `_amount`.
- Highlight how descriptive names make joins and reports self-explanatory.

#### Table definitions
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

#### Why the names matter
- Readers can guess relationships: `user_id` points to `user.id`, `currency_code` points to `currency.code`.
- Timestamp columns share the `_at` suffix so a report can scan for date fields quickly.
- `total_amount_cents` makes the unit explicit; `total_amount` alone would force readers to check documentation.

> **Field Note:** Using consistent names across tables lets BI tools auto-discover joins instead of requiring manual mapping.

### 9.2 Example Constraints & Indexes

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

#### Why the names matter
- `ux_user_email` tells reviewers that the unique key protects email addresses. Without the prefix the intent would be hidden.
- `ck_order_status` documents the allowed values in plain words. Debugging failed inserts becomes faster because MySQL includes the name in the error message.
- Composite indexes like `ix_order_user_id_status` expose the column order so engineers can plan queries that match it.

> **Field Note:** When a migration fails in production, a clear constraint name lets you find and fix the broken rule without guessing.

### 9.3 Example View, Function, Trigger

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

#### Why the names matter
- The `v_`, `fn_`, and `trg_` prefixes tell the reader what kind of object they are before reading the definition.
- `fn_order_total_in_currency` describes inputs and behaviour. A vague name like `fn_convert` would leave the purpose unclear.
- `trg_order_set_fulfilled_at` encodes both the target table and the action so maintainers can find it quickly.

> **Field Note:** Clear routine names reduce onboarding time because new engineers can map business flows without reading every line of SQL.

### 9.4 Example Migration File

Migrations are the final piece. A good filename and structure keep releases predictable.

#### File name pattern
```
20240718-120000_add-order-totals.sql
```
- The timestamp sorts migrations chronologically.
- Hyphen-separated words describe the action in plain language.
- The verb-noun structure (`add-order-totals`) mirrors the change inside the file.

#### File contents
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

#### Why the structure works
- A short comment at the top documents the intent for reviewers and incident responders.
- Wrapping changes in a transaction makes the migration safe to rerun if a step fails.
- Naming the index inside the migration keeps schema diffs clean when running `SHOW CREATE TABLE` later.

> **Field Note:** Teams that describe the change and index names directly in the migration file avoid the "mystery DDL" problem when reading production history.

<a id="chapter-10"></a>
## Chapter 10 – Cheat Sheet (Quick Reference)


This quick reference condenses the full guide into a single page. Use it during code review, pair programming, or migration planning when you need fast reminders without rereading every chapter. Each section links the object type to the recommended pattern, explains the "why" in one sentence, and shows a safe example.

### 10.1 Database Names

| Do this | Why | Example |
| --- | --- | --- |
| Use lowercase snake case with product and context | Readers can guess ownership and purpose from the name alone | `billing_service_prod`
| Keep environment suffixes short and predictable | Automation scripts can target the right database safely | `analytics_stg`
| Avoid special characters or camelCase | Tools and shells behave consistently across platforms | `identity_dev`

> **Remember:** Database names change rarely. Choose clarity now to avoid risky renames later.

### 10.2 Table Names

| Rule | Why it helps | Example |
| --- | --- | --- |
| Use singular nouns (`order`, not `orders`) | Singular names avoid clashes with ORMs and make foreign keys obvious | `CREATE TABLE order (...);` |
| Include descriptive nouns | People can read intent without guessing business meaning | `shipping_manifest`
| Add suffixes only when they signal a pattern | Keeps names short while highlighting behaviour | `user_archive`, `tmp_daily_metrics`

### 10.3 Column Name Examples

| Column type | Pattern | Reason |
| --- | --- | --- |
| Primary key | `id` | Every table shares the same join target |
| Foreign key | `<referenced_table>_id` | Reveals the relationship at a glance |
| Timestamp | `<action>_at` | Communicates that the value is a point in time |
| Soft delete | `deleted_at` (`NULL` when active) | Queries can filter active rows simply |
| Amounts | `<thing>_amount_<unit>` or `<thing>_<unit>` | Prevents confusion when multiple currencies or units exist |
| JSON | `<topic>_json` | Signals a flexible payload and keeps structured columns clean |

> **Tip:** Add column comments in migrations when the purpose is not obvious from the name alone.

### 10.4 Constraint & Index Patterns

| Type | Prefix | Example name | Notes |
| --- | --- | --- | --- |
| Primary key | `pk_` | `pk_order` | Usually auto-generated, but naming helps during schema reviews |
| Foreign key | `fk_` | `fk_order_user` | Include both tables so you can spot relationships in error logs |
| Unique constraint | `ux_` | `ux_user_email` | Highlight the business rule being protected |
| Non-unique index | `ix_` | `ix_order_user_id_status` | Order columns from most to least selective |
| Check constraint | `ck_` | `ck_order_status` | Describe the allowed values in words |
| Full-text index | `ft_` | `ft_article_body` | Avoid mixing with non-text columns |

### 10.5 Boolean & Enum Conventions

| Situation | Preferred pattern | Why |
| --- | --- | --- |
| Boolean columns | Prefix with `is_`, `has_`, or `can_` | Makes true/false intent explicit | 
| Status codes | Use lookup tables or enums with meaning in the name | Prevents anonymous values like `status = 3` |
| Feature flags | `is_feature_enabled` or `can_use_feature` | Clear to business and engineering readers |
| Nullable booleans | Avoid when possible; use `NULL` only for "unknown" | Keeps logic simple and reduces three-state bugs |

### 10.6 Migration Filename Example

```
20240718-120000_add-user-preferred-currency.sql
```

- Timestamp (YYYYMMDD-HHMMSS) sorts files automatically.
- Verb-noun description communicates the intent.
- Hyphen-separated words stay readable on every operating system.

> **Field Note:** Agree on the filename pattern in your team charter so new engineers can contribute without asking for reminders.

<a id="chapter-11"></a>
## Chapter 11 – Conclusion


The final chapter explains why the naming standard works in the real world, how to enforce it without slowing teams down, and the guiding principle that keeps every exception honest.

### 11.1 Why These Conventions Work in Real-World Scale

- **They reduce decision fatigue.** Engineers do not spend time inventing names on the spot because the pattern is already agreed. That speed matters when dozens of teams ship features at once.
- **They keep automation reliable.** ETL pipelines, schema diff tools, and monitoring jobs expect predictable object names. When the schema stays consistent, automation scripts can be simple and resilient.
- **They improve onboarding.** New hires can read the database like a well-written book. Each object tells a story through its name, so domain knowledge spreads faster.
- **They make incidents calmer.** During an outage the right name points you to the right table or constraint immediately. Clear naming reduces the time spent searching and increases the time spent fixing.

> **Field Note:** Mature companies document naming rules because firefighting without a map is expensive and stressful.

### 11.2 How to Enforce Them (Linters, CI/CD, Code Review)

1. **Add linting to migration pipelines.** Tools such as `sqlfluff` or custom scripts can reject files that break naming patterns before they reach version control.
2. **Automate schema review.** Use CI jobs that run `SHOW CREATE TABLE` on new migrations and diff the output to confirm constraint and index names follow the prefixes.
3. **Write review checklists.** Pull request templates should include reminders such as "Are table names singular?" and "Do new timestamps end with `_at`?".
4. **Teach through examples.** Keep a shared folder of good migrations and bad anti-patterns so reviewers can point to concrete cases.
5. **Document exceptions.** When a business rule requires a deviation, record the reason and the owner. That transparency prevents accidental spread of one-off patterns.

> **Tip:** Enforcing conventions works best when automation catches mistakes early and reviewers explain the "why" rather than only the "what".

### 11.3 The Golden Rule → Consistency Beats Cleverness

- **Consistency keeps the schema teachable.** Clever names age poorly; consistent names stay readable for years.
- **Consistency is inclusive.** Simple English and predictable patterns help cross-functional teammates participate in design discussions.
- **Consistency scales.** When the schema follows one style, new services, analytics jobs, and data warehouses can integrate without friction.
- **Consistency allows safe exceptions.** When the default is clear, an exception stands out and receives the scrutiny it deserves.

> **Final Reminder:** You can always explain a consistent name. You spend hours explaining a clever one.

---

*Last updated: 2025-09-28 07:31 UTC.*
