# Chapter 3 – Database-Level Conventions

[← Previous: Core Naming Rules](02-core-naming-rules.md) • [Back to index](index.md) • [Next chapter →](04-table-and-column-naming.md)

Before you create tables, choose good names for the databases themselves. This chapter walks you through three simple patterns, each with a short checklist and an example you can copy into MySQL.

## 3.1 Name databases by product and purpose

**Learn the rule.** Use the pattern `<product>_<purpose>`, for example `project_service` or `analytics_reporting`.

**Why it works.**

- Every engineer instantly knows which team owns the data.
- Permissions and monitoring can key off predictable prefixes.
- Backups are easy to map to business areas.

**Follow the steps.**

1. Identify the product, bounded context, or service (such as `project`, `support`, `identity`).
2. Add a short word that describes the job: `service`, `reporting`, `warehouse`, `archive`.
3. Join them with an underscore, all lowercase.
4. Record the name in your internal inventory.

```sql
-- ✅ Clear names
CREATE DATABASE project_service;
CREATE DATABASE project_reporting;

-- ❌ Too vague
CREATE DATABASE db1;
CREATE DATABASE project2;
```

## 3.2 Add environment suffixes consistently

**Learn the rule.** Extend the base name with `_dev`, `_stg`, `_prod`, or whatever suffixes your company standardises.

**Why it works.**

- Connection lists show the environment immediately.
- Deployment scripts can target the right database without manual edits.
- On-call engineers avoid running destructive queries in production by mistake.

**Follow the steps.**

1. Start from the base name in Section 3.1 (`project_service`).
2. Append the environment suffix: `_dev`, `_test`, `_stg`, `_prod`.
3. Keep the suffix ordering identical across services so related names group together.
4. Add automated checks that warn when someone connects to `_prod` from an unapproved network.

```sql
-- ✅ Environment-aware
CREATE DATABASE project_service_dev;
CREATE DATABASE project_service_stg;
CREATE DATABASE project_service_prod;

-- ❌ Inconsistent styles
CREATE DATABASE ProjectServiceDEV;
CREATE DATABASE projectStage;
CREATE DATABASE prod_project_service;
```

## 3.3 Keep a predictable registry

**Learn the rule.** Maintain a simple registry (spreadsheet or Markdown file) that lists every database, its owner, and its purpose.

**Why it works.**

- Automation jobs can verify the database exists before deploying.
- Auditors and support teams know who to ask for access.
- Renames become rare, which keeps connection strings stable.

**Follow the steps.**

1. Create a `docs/database-inventory.md` file or similar shared list.
2. Add a row every time you provision a database: name, owner, purpose, environment.
3. Review the list during quarterly maintenance to retire unused databases.
4. Link the registry from onboarding docs so newcomers find it fast.

```text
Example registry snippet
- project_service_prod → Owned by Project Platform Team → Primary OLTP workload
- project_service_stg  → Owned by Project Platform Team → Staging copy for QA
- project_reporting_prod → Owned by Data Insights Team → BI dashboards
```

> **Tip:** During incidents, responders should be able to open the registry and know exactly which database to inspect. Predictable names turn chaos into a checklist.

---

[← Previous: Core Naming Rules](02-core-naming-rules.md) • [Back to index](index.md) • [Next chapter →](04-table-and-column-naming.md)
