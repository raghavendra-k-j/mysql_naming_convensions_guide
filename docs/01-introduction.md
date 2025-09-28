# Chapter 1 – Introduction

[← Back to index](index.md) • [Next chapter →](02-core-naming-rules.md)

---

Think of this guide as a W3Schools-style course for naming things in MySQL. We keep the lessons short, show the rules, and immediately follow with examples you can copy.

## 1.1 Why bother with naming rules?

**The problem.** MySQL lets every engineer pick their own style. After a few releases, your schema turns into a puzzle of mixed cases, unexplained acronyms, and duplicated ideas.

**The payoff.** A shared convention removes the guesswork. When every table and column follows the same pattern, you can read new SQL like plain English, reviewers move faster, and automation scripts stop breaking.

**Quick look.**

```sql
-- Random styles slow everyone down
SELECT *
FROM Projects
JOIN projectMembers ON Projects.ID = projectMembers.projectID;

-- Consistent names read like a sentence
SELECT p.id, p.name, tm.full_name
FROM project p
JOIN project_member tm ON tm.project_id = p.id;
```

> **Remember:** you only choose a name once, but the team reads it hundreds of times.

## 1.2 What goes wrong without a standard?

Follow this checklist. If any item matches your schema today, you will benefit from the rules in this guide.

1. **Duplicate words.** `project`, `projects`, and `tbl_project` all exist at once.
2. **Mystery flags.** Columns called `flag_a` or `data1` force guesswork.
3. **Inconsistent casing.** `TaskID` in one table, `task_id` in another.
4. **Hidden relationships.** Foreign keys named `ref1` give no hint about the target table.

Here is the difference in practice:

```sql
-- Painful to read
SELECT flag_a
FROM tbl_project_record
WHERE stat = 'A';

-- Straightforward to read
SELECT is_archived
FROM project_record
WHERE status_code = 'active';
```

## 1.3 How to use this guide

Each chapter follows the same structure:

1. **Explain the rule** in clear English.
2. **Show why it matters** with short bullet points.
3. **Demonstrate the rule** using a small project-tracking schema (tables such as `project`, `team_member`, and `time_entry`).
4. **Give you a quick checklist** so you can apply the idea immediately.

If you are onboarding a teammate, send them this chapter and the cheat sheet (Chapter 10). If you are reviewing code, open the relevant chapter while you check the migration.

---

[← Back to index](index.md) • [Next chapter →](02-core-naming-rules.md)
