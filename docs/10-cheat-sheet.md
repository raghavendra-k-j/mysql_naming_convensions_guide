# Chapter 10 – Cheat Sheet (Quick Reference)

[← Previous: Examples & Patterns in Action](09-examples-and-patterns.md) • [Back to index](index.md) • [Next chapter →](11-conclusion.md)

This quick reference condenses the full guide into a single page. Use it during code review, pair programming, or migration planning when you need fast reminders without rereading every chapter. Each section links the object type to the recommended pattern, explains the "why" in one sentence, and shows a safe example.

## 10.1 Database Names

| Do this | Why | Example |
| --- | --- | --- |
| Use lowercase snake case with product and context | Readers can guess ownership and purpose from the name alone | `billing_service_prod`
| Keep environment suffixes short and predictable | Automation scripts can target the right database safely | `analytics_stg`
| Avoid special characters or camelCase | Tools and shells behave consistently across platforms | `identity_dev`

> **Remember:** Database names change rarely. Choose clarity now to avoid risky renames later.

## 10.2 Table Names

| Rule | Why it helps | Example |
| --- | --- | --- |
| Use singular nouns (`order`, not `orders`) | Singular names avoid clashes with ORMs and make foreign keys obvious | `CREATE TABLE order (...);` |
| Include descriptive nouns | People can read intent without guessing business meaning | `shipping_manifest`
| Add suffixes only when they signal a pattern | Keeps names short while highlighting behaviour | `user_archive`, `tmp_daily_metrics`

## 10.3 Column Name Examples

| Column type | Pattern | Reason |
| --- | --- | --- |
| Primary key | `id` | Every table shares the same join target |
| Foreign key | `<referenced_table>_id` | Reveals the relationship at a glance |
| Timestamp | `<action>_at` | Communicates that the value is a point in time |
| Soft delete | `deleted_at` (`NULL` when active) | Queries can filter active rows simply |
| Amounts | `<thing>_amount_<unit>` or `<thing>_<unit>` | Prevents confusion when multiple currencies or units exist |
| JSON | `<topic>_json` | Signals a flexible payload and keeps structured columns clean |

> **Tip:** Add column comments in migrations when the purpose is not obvious from the name alone.

## 10.4 Constraint & Index Patterns

| Type | Prefix | Example name | Notes |
| --- | --- | --- | --- |
| Primary key | `pk_` | `pk_order` | Usually auto-generated, but naming helps during schema reviews |
| Foreign key | `fk_` | `fk_order_user` | Include both tables so you can spot relationships in error logs |
| Unique constraint | `ux_` | `ux_user_email` | Highlight the business rule being protected |
| Non-unique index | `ix_` | `ix_order_user_id_status` | Order columns from most to least selective |
| Check constraint | `ck_` | `ck_order_status` | Describe the allowed values in words |
| Full-text index | `ft_` | `ft_article_body` | Avoid mixing with non-text columns |

## 10.5 Boolean & Enum Conventions

| Situation | Preferred pattern | Why |
| --- | --- | --- |
| Boolean columns | Prefix with `is_`, `has_`, or `can_` | Makes true/false intent explicit | 
| Status codes | Use lookup tables or enums with meaning in the name | Prevents anonymous values like `status = 3` |
| Feature flags | `is_feature_enabled` or `can_use_feature` | Clear to business and engineering readers |
| Nullable booleans | Avoid when possible; use `NULL` only for "unknown" | Keeps logic simple and reduces three-state bugs |

## 10.6 Migration Filename Example

```
20240718-120000_add-user-preferred-currency.sql
```

- Timestamp (YYYYMMDD-HHMMSS) sorts files automatically.
- Verb-noun description communicates the intent.
- Hyphen-separated words stay readable on every operating system.

> **Field Note:** Agree on the filename pattern in your team charter so new engineers can contribute without asking for reminders.

[← Previous: Examples & Patterns in Action](09-examples-and-patterns.md) • [Back to index](index.md) • [Next chapter →](11-conclusion.md)
