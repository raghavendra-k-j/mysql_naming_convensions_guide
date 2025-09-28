# Chapter 3 – Database-Level Conventions

[← Previous: Core Naming Rules](02-core-naming-rules.md) • [Back to index](index.md) • [Next chapter →](04-table-and-column-naming.md)

This chapter focuses on the database names themselves. Before you create tables, you should decide how to label each database so every tool, environment, and teammate understands its purpose at a glance.

## 3.1 Naming Databases by Product and Context

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

## 3.2 Handling Multiple Environments (dev, stg, prod) Safely

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

## 3.3 Why Database Names Should Be Predictable

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

[← Previous: Core Naming Rules](02-core-naming-rules.md) • [Back to index](index.md) • [Next chapter →](04-table-and-column-naming.md)
