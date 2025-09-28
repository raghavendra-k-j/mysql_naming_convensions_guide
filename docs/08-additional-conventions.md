# Chapter 8 – Additional Conventions That Pay Off at Scale

[← Previous: Migrations & Versioning](07-migrations-and-versioning.md) • [Back to index](index.md) • [Next chapter →](09-examples-and-patterns.md)

These tips wrap around the core rules and become invaluable as your MySQL installation grows. Each mini-lesson follows the same pattern: learn the idea, understand why it matters, and apply it with a quick example.

## 8.1 Standardise character sets and collations

**Learn the rule.** Use `utf8mb4` everywhere with a modern collation such as `utf8mb4_unicode_ci` (case-insensitive) or `utf8mb4_0900_ai_ci` (MySQL 8+).

**Why it works.**

- Supports every Unicode character, including emoji and multi-language names.
- Keeps sorting and comparisons consistent across tables.
- Plays nicely with ORMs and analytics tools that expect Unicode.

**Follow the steps.**

1. Set the database default when you create it.
2. Repeat the setting on each table to guard against future default changes.
3. Declare the charset and collation on text columns explicitly in migrations.

```sql
CREATE TABLE team_member (
    id BIGINT UNSIGNED PRIMARY KEY,
    full_name VARCHAR(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
    email VARCHAR(320) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL
) CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

## 8.2 Store time in UTC and suffix with `_at`

**Learn the rule.** Keep all timestamps in UTC and end the column name with `_at`.

**Why it works.**

- Avoids daylight-saving glitches.
- Makes event ordering easy across services.
- `_at` instantly signals “point in time.”

**Follow the steps.**

1. Normalize incoming times to UTC before persisting.
2. Name the column `created_at`, `processed_at`, etc.
3. Document how to convert for display in the application layer.

```sql
ALTER TABLE time_entry
    ADD COLUMN submitted_at DATETIME NOT NULL COMMENT 'UTC timestamp when the entry was submitted by the member';
```

## 8.3 Choose between `TIMESTAMP` and `DATETIME` on purpose

**Learn the rule.** Use `TIMESTAMP` for audit columns that stay within 1970–2038 UTC, and `DATETIME` for business events outside that range.

**Why it works.**

- Prevents overflow bugs in far-future schedules.
- Keeps automatic `CURRENT_TIMESTAMP` behaviour where you need it.
- Documented choices stop accidental type swaps later.

**Follow the steps.**

1. Use `TIMESTAMP` for `created_at` and `updated_at` fields.
2. Use `DATETIME` for scheduling (`kickoff_at`, `renewal_at`).
3. Mention the reason in a column comment.

```sql
CREATE TABLE project_schedule (
    id BIGINT UNSIGNED PRIMARY KEY,
    project_id BIGINT UNSIGNED NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    kickoff_at DATETIME NOT NULL COMMENT 'UTC meeting start time, can extend beyond 2038'
);
```

## 8.4 Use precise types and lengths

**Learn the rule.** Match column types to the real-world values instead of falling back to generic `TEXT` or oversized lengths.

**Why it works.**

- Keeps indexes compact and fast.
- Stops bad data at the database boundary.
- Communicates business rules right inside the schema.

**Follow the steps.**

1. Research real limits (email: 320 chars, ISO country: 2 chars).
2. Note unusual lengths in column comments.
3. Add `CHECK` constraints for patterns when useful.

```sql
ALTER TABLE project
    ADD COLUMN sponsor_email VARCHAR(320) NOT NULL COMMENT 'Length per RFC 5321';
```

## 8.5 Default to `NOT NULL`

**Learn the rule.** Declare columns `NOT NULL` unless a nullable state carries business meaning.

**Why it works.**

- Reduces branching in application code.
- Improves storage layout and query planning.
- Makes dashboards and reports cleaner.

**Follow the steps.**

1. Start with `NOT NULL` on new columns.
2. Provide a safe default or backfill before enforcing it.
3. If `NULL` is meaningful, explain it in a comment.

```sql
ALTER TABLE project
    ADD COLUMN review_deadline_at DATETIME NOT NULL COMMENT 'Every project has a review milestone';
```

## 8.6 Provide public IDs alongside internal IDs

**Learn the rule.** Use a fast numeric `id` internally and expose a separate `public_id` (UUID or token) to external clients.

**Why it works.**

- Internal joins stay efficient.
- Public APIs avoid predictable, sequential identifiers.
- Rotating external identifiers does not disturb relational integrity.

**Follow the steps.**

1. Keep `id BIGINT UNSIGNED AUTO_INCREMENT` as the primary key.
2. Add `public_id CHAR(27)` or `CHAR(36)` with a unique constraint.
3. Generate the public ID in application code or via a trigger.

```sql
CREATE TABLE project_invitation (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    public_id CHAR(27) NOT NULL,
    project_id BIGINT UNSIGNED NOT NULL,
    issued_at DATETIME NOT NULL,
    UNIQUE KEY ux_project_invitation__public_id (public_id)
);
```

## 8.7 Avoid name collisions with reserved words or vague terms

**Learn the rule.** Do not use bare reserved words or ambiguous nouns. Extend them with clarifiers.

**Why it works.**

- Prevents constant quoting (`SELECT * FROM `order``) and ORM issues.
- Makes cross-team integrations clearer (`status_code`, `member_role`).
- Reduces misunderstandings during handoffs.

**Follow the steps.**

1. Check the MySQL reserved word list before naming.
2. Add context words: `purchase_order`, `member_group`, `project_status_code`.
3. Document approved terminology in your team glossary.

```sql
CREATE TABLE member_role (
    role_code VARCHAR(32) PRIMARY KEY,
    display_label VARCHAR(64) NOT NULL
);
```

**Scaling checklist**

- [ ] Is `utf8mb4` the default everywhere?
- [ ] Are timestamps stored in UTC with `_at` names?
- [ ] Did you choose `TIMESTAMP` vs `DATETIME` deliberately?
- [ ] Do column types and lengths reflect real-world limits?
- [ ] Are new columns `NOT NULL` by default?
- [ ] Do public APIs expose dedicated identifiers?
- [ ] Did you avoid reserved or overloaded words?

If you hit “yes” on each point, your schema is ready for rapid growth.
