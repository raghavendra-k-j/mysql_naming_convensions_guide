# Chapter 5 – Constraints & Indexes

[← Previous: Table & Column Naming](04-table-and-column-naming.md) • [Back to index](index.md) • [Next chapter →](06-views-routines-triggers.md)

Good table and column names keep data readable, but the story is not complete until constraints and indexes are named just as clearly. When constraint names explain what they guard, error messages become self-explanatory, migrations stay predictable, and on-call engineers can fix problems without guesswork. This chapter walks through each constraint and index pattern so teams can spot issues in seconds instead of hours.

## 5.1 Why Naming Constraints Matters for Debugging

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

## 5.2 Primary Keys (`pk_<table>`)

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

## 5.3 Foreign Keys (`fk_<table>__<column>__<ref_table>`)

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

## 5.4 Unique Constraints (`ux_<table>__<columns>`)

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

## 5.5 Non-Unique Indexes (`ix_<table>__<columns>`)

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

## 5.6 Check Constraints (`ck_<table>__<rule>`)

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

## 5.7 Full-Text Indexes (`ft_<table>__<columns>`)

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

## Wrapping Up

Well-named constraints and indexes act like documentation that lives inside the database. They shorten incident response, prevent duplicate structures, and make migrations safer. Adopt these patterns alongside the table and column rules from Chapter 4, and you will have a schema that explains itself to anyone who reads it.
