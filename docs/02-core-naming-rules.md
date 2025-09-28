# Chapter 2 – Core Naming Rules

[← Previous: Introduction](01-introduction.md) • [Back to index](index.md) • [Next chapter →](03-database-level-conventions.md)

This chapter is the heart of the guide. Each section follows a W3Schools rhythm: learn the rule, see why it matters, then copy a working example. All examples use a simple project-tracking schema (`project`, `team_member`, `time_entry`) so the patterns stay consistent throughout the guide.

## 2.1 Use `lower_snake_case` everywhere

**Rule.** Write every identifier—databases, tables, columns, indexes, routines—in lowercase snake case.

**Why it matters.**

- MySQL treats names differently on Linux versus Windows. One style avoids "works on my machine" bugs.
- Snake case keeps multi-word names readable without quoting.
- Lowercase names integrate smoothly with ORMs, CLI tools, and environment variables.

**How to apply.**

1. Convert each word to lowercase.
2. Join the words with underscores.
3. Avoid hyphens or spaces because they force backticks.

```sql
-- ✅ Follows the rule
SELECT total_minutes
FROM time_entry
WHERE project_id = 12;

-- ❌ Breaks the rule (camelCase and mixed case table)
SELECT TotalMinutes
FROM TimeEntry
WHERE ProjectID = 12;
```

## 2.2 Prefer full words over cryptic abbreviations

**Rule.** Spell out business terms. Use abbreviations only when the whole team knows the meaning (such as `id`, `ip`, `url`).

**Why it matters.**

- Full names are self-documenting and search-friendly.
- New teammates understand the schema without a translation guide.
- Analytics tools and dashboards show useful labels automatically.

**How to apply.**

1. Write the real-world phrase: `billing_address`, not `bill_addr`.
2. If the word feels long, add it to a glossary so the team stays aligned.
3. Be consistent. If you pick `due_date`, use it everywhere (not `deadline` in one table and `due` in another).

```sql
-- ✅ Clear intent
ALTER TABLE project
    ADD COLUMN kickoff_date DATE NOT NULL;

-- ❌ Requires tribal knowledge
ALTER TABLE proj
    ADD COLUMN kd DATE NOT NULL;
```

## 2.3 Handle acronyms deliberately

**Rule.** When an acronym is unavoidable (for example, `API`, `SKU`, `VAT`), write it in lowercase inside the snake case and document it once.

**Why it matters.**

- Consistent casing keeps search results reliable (`vat_rate`, not `VatRate`).
- Documented acronyms reduce onboarding time.
- Avoiding random uppercase letters prevents quoting issues.

**How to apply.**

1. Treat the acronym as a regular word: `api_client_id`, `vat_rate`.
2. Add a short description to your team glossary.
3. If two acronyms collide (`id` vs `team_id`), keep both words for clarity.

```sql
-- ✅ Balanced use of acronyms
CREATE TABLE integration_endpoint (
    id BIGINT UNSIGNED PRIMARY KEY,
    api_key CHAR(36) NOT NULL,
    vat_rate DECIMAL(5,2) NOT NULL
);

-- ❌ Mixed casing and unexplained short forms
CREATE TABLE integration (
    ID BIGINT PRIMARY KEY,
    ApiKey CHAR(36) NOT NULL,
    v_rate DECIMAL(5,2) NOT NULL
);
```

## 2.4 Keep table names singular

**Rule.** Name tables with singular nouns (`project`, `project_member`) so each row reads like one entity.

**Why it matters.**

- Joins sound natural: `project JOIN project_member` instead of `projects JOIN project_members`.
- Singular words avoid awkward plural forms (`person` vs `people`).
- ORMs and scaffolding tools usually expect singular table names.

**How to apply.**

1. Use the entity name in singular form: `task`, `task_note`.
2. For link tables, combine the two nouns alphabetically: `project_task` (not `task_project`).
3. Document intentional exceptions such as `analytics` or `settings` so nobody "fixes" them later.

```sql
-- ✅ Reads cleanly
SELECT p.name, tm.full_name
FROM project p
JOIN project_member tm ON tm.project_id = p.id;

-- ❌ Harder to parse
SELECT projects.name, team_members.full_name
FROM projects
JOIN team_members ON team_members.project_id = projects.id;
```

## 2.5 Skip noise prefixes like `tbl_` and `col_`

**Rule.** The object type is already obvious. Keep the meaningful word front and centre.

**Why it matters.**

- `tbl_project` wastes space and hides the actual topic.
- Renaming later requires touching every query.
- Dashboards and schema explorers sort alphabetically, so noise prefixes group unrelated tables together.

**How to apply.**

1. Name tables by subject (`project_status`).
2. Name columns by attribute (`archived_at`).
3. Reserve suffixes for useful context (`_history`, `_log`).

```sql
-- ✅ Focused names
CREATE TABLE project_status (
    project_id BIGINT UNSIGNED PRIMARY KEY,
    status_code VARCHAR(32) NOT NULL,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ❌ Noise everywhere
CREATE TABLE tbl_project_status (
    col_project_id BIGINT UNSIGNED PRIMARY KEY,
    col_status VARCHAR(32) NOT NULL,
    col_updated TIMESTAMP NOT NULL
);
```

## 2.6 Avoid reserved words and MySQL keywords

**Rule.** Never use a reserved word alone (`order`, `group`, `select`). If you truly need the word, add context.

**Why it matters.**

- Reserved words require backticks in every query.
- New MySQL versions can add keywords and break your schema.
- Many tools refuse to generate code for reserved names.

**How to apply.**

1. Check the official MySQL reserved word list before naming.
2. Add a meaningful suffix or prefix: `purchase_order`, `status_code`.
3. Lint migrations with tooling (`sqlfluff`, `schemalint`) to catch mistakes early.

```sql
-- ✅ Safe alternative
CREATE TABLE purchase_order (
    id BIGINT UNSIGNED PRIMARY KEY,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ❌ Needs quoting forever
CREATE TABLE `order` (
    `id` BIGINT UNSIGNED PRIMARY KEY
);
```

**Quick checklist before you name anything**

- [ ] Is it lower_snake_case?
- [ ] Are the words spelled out?
- [ ] Are acronyms documented and lowercase?
- [ ] Is the table name singular?
- [ ] Are there any pointless prefixes?
- [ ] Did you avoid reserved words?

Tick every box and you are ready for the next chapter.
