# Chapter 7 – Migrations & Versioning

[← Previous: Views, Routines & Triggers](06-views-routines-triggers.md) • [Back to index](index.md) • [Next chapter →](08-additional-conventions.md)

Naming database objects is only half the story. Teams also need a disciplined way to name the migration files that create, change, and remove those objects. Clear migration names help everyone understand deployment order, audit changes during incidents, and roll back safely when something goes wrong. This chapter explains how to name migration files so large teams can ship changes with confidence.

## 7.1 File Naming → Why Timestamped Names Are Essential for Ordering

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

## 7.2 One Action/Object per File → Why Small, Atomic Changes Matter

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

## 7.3 Rollback & CI/CD Safety

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

[← Previous: Views, Routines & Triggers](06-views-routines-triggers.md) • [Back to index](index.md) • [Next chapter →](08-additional-conventions.md)
