# Chapter 2 – Core Naming Rules

[← Previous: Introduction](01-introduction.md) • [Back to index](index.md) • [Next chapter →](03-database-level-conventions.md)

This chapter explains the everyday rules that keep a MySQL schema readable. Each rule is written in simple English, but the ideas go deep so your team can apply them with confidence.

## 2.1 Case & Style → Why lower_snake_case Wins

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

## 2.2 Language → Why Full Words Beat Abbreviations

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

## 2.3 Acronyms & Abbreviations → How to Use Them When Necessary

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

## 2.4 Singular vs Plural Table Names → Why Singular Is Less Ambiguous

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

## 2.5 Avoiding Noise Prefixes (tbl_, col_) → Why They Harm More Than Help

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

## 2.6 Reserved Words → Why They Create Long-Term Bugs

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

[← Previous: Introduction](01-introduction.md) • [Back to index](index.md) • [Next chapter →](03-database-level-conventions.md)
