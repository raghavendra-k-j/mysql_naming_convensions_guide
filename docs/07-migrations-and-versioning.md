# Chapter 7 – Migrations & Versioning

[← Previous: Views, Routines & Triggers](06-views-routines-triggers.md) • [Back to index](index.md) • [Next chapter →](08-additional-conventions.md)

Schema changes should be as predictable as the schema itself. Use this chapter as a playbook for naming migration files and keeping version history clear.

## 7.1 Timestamped filenames keep order predictable

**Learn the rule.** Start every migration filename with a UTC timestamp in `YYYYMMDDHHMMSS` format, followed by a short slug: `20240718120000_add-project-status.sql`.

**Why it works.**

- Timestamps sort naturally, so every environment runs migrations in the same order.
- CI/CD jobs can detect new files by comparing timestamps.
- Auditors can align database changes with application releases.

**Follow the steps.**

1. Generate the timestamp in UTC.
2. Add an underscore and a hyphen-separated slug (`add-project-status` or `add_project_status`, pick one style and stick to it). This guide uses hyphenated slugs.
3. Keep the slug short but descriptive.
4. Store migrations in a dedicated directory (for example, `db/migrate`).

```text
20240718120000_add-project-status.sql
20240722153000_create-v_project_recent_activity.sql
20240725100000_sp_archive_project.sql
```

## 7.2 One task per migration file

**Learn the rule.** Keep each file focused on a single object or closely related set of changes.

**Why it works.**

- Reviews stay short and clear.
- Rollbacks only touch the failing change.
- Merge conflicts shrink because fewer engineers edit the same file.

**Follow the steps.**

1. Decide what the file is responsible for (create a table, add a column, drop a trigger).
2. Name the slug to match the action (`add-time-entry-billable-flag`).
3. If you need to backfill or clean data, add a new migration with its own timestamp.
4. Mention dependencies in comments when one file must run before another.

```text
20240730090000_add-time-entry-billable-flag.sql
20240730090500_backfill-time-entry-billable-flag.sql
20240730091000_add-time-entry-billable-index.sql
```

## 7.3 Document rollbacks right in the file

**Learn the rule.** Write a short header comment describing the change, its purpose, and how to undo it. When you store rollback scripts separately, reuse the same timestamp.

**Why it works.**

- On-call engineers know exactly how to reverse the change.
- CI/CD pipelines can pair forward and backward files reliably.
- Future teammates understand why the migration exists.

**Follow the steps.**

1. Start each file with comments explaining the forward change and the rollback path.
2. If you keep paired scripts, name the rollback `<timestamp>_<slug>_down.sql`.
3. Test both forward and backward scripts in staging before release.
4. Use safety guards like `IF EXISTS` for destructive actions.

```sql
-- 20240718120000_add-project-status.sql
-- Adds project_status table and links it to project.status_code.
-- Rollback: run 20240718120000_add-project-status_down.sql to drop the table and column.
START TRANSACTION;

CREATE TABLE project_status (
    status_code VARCHAR(32) PRIMARY KEY,
    display_label VARCHAR(64) NOT NULL
);

ALTER TABLE project
    ADD COLUMN status_code VARCHAR(32) NOT NULL DEFAULT 'planning',
    ADD CONSTRAINT fk_project__project_status
        FOREIGN KEY (status_code) REFERENCES project_status (status_code);

COMMIT;
```

```sql
-- 20240718120000_add-project-status_down.sql
-- Rollback for 20240718120000_add-project-status.sql
START TRANSACTION;

ALTER TABLE project
    DROP FOREIGN KEY fk_project__project_status,
    DROP COLUMN status_code;

DROP TABLE project_status;

COMMIT;
```

**Checklist**

- [ ] Does the filename begin with a UTC timestamp?
- [ ] Does the slug describe one clear action?
- [ ] Is the change limited to a single object or tightly related set?
- [ ] Are rollback instructions documented and, if needed, scripted?

If all boxes are checked, your migration naming is ready for production.
