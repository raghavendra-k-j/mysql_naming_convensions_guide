# Chapter 8 – Additional Conventions That Pay Off at Scale

[← Previous: Migrations & Versioning](07-migrations-and-versioning.md) • [Back to index](index.md) • [Next chapter →](09-examples-and-patterns.md)

These conventions do not fit into a single object type, yet they protect every MySQL project once it starts to grow. Think of them as the safety net around the core naming rules. Each section explains the recommendation, the pain you avoid by following it, and a simple way to put the idea into practice.

## 8.1 Character Sets & Collations → Why utf8mb4 Is the Only Safe Default

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

## 8.2 Time Handling → Why Always Store UTC with `_at`

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

## 8.3 DATETIME vs TIMESTAMP → Differences, Trade-offs, and Best Practices

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

## 8.4 Explicit Types & Lengths → Why `VARCHAR(320)` Beats `TEXT`

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

## 8.5 Nullability → Why `NOT NULL` Should Be Default

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

## 8.6 IDs → Why Dual IDs (`id` + `public_id`) Improve Internal/External Safety

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

## 8.7 Naming Collisions → Why Extending Reserved Names Prevents Bugs

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

## 8.8 API Alignment → Why DB Schema Should Mirror API Fields

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

## 8.9 Archival Tables (`_archive`) → Why Separation Improves Performance

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

## 8.10 Temporary Tables (`tmp_...`) → Why Explicit Naming Avoids Accidents

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

## 8.11 Consistency Over Cleverness → Why Exceptions Must Be Documented

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

## 8.12 Defaults: Database vs Application — When to Use Which

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

[← Previous: Migrations & Versioning](07-migrations-and-versioning.md) • [Back to index](index.md) • [Next chapter →](09-examples-and-patterns.md)
