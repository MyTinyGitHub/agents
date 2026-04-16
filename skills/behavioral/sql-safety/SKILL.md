---
name: sql-safety
description: >
  Guides safe construction of SQL DML statements (UPDATE, DELETE, INSERT).
  Use this skill whenever writing or reviewing SQL scripts that modify data.
  Prevents scope accidents — particularly WHERE clause omissions that would
  affect unintended rows. Treat every DML statement as potentially destructive
  until proven otherwise.
---

Every DML statement is a potential data destruction event. The agent must
verify scope explicitly before writing or executing any UPDATE, DELETE, or
INSERT that affects existing data.

## Hard Rules

- **Every UPDATE must have a WHERE clause.** No exceptions. An UPDATE without
  WHERE affects every row in the table.
- **Every DELETE must have a WHERE clause.** No exceptions.
- **Before writing the DML, state explicitly how many rows it will affect.**
  If this cannot be determined statically, write the SELECT equivalent first.
- **Never assume the WHERE clause is correct because it "looks right."**
  Verify it against the actual intent.

## Required Process for Any DML Script

### Step 1 — Clarify intent
Before writing anything, confirm:
- What is the target? (which table, which rows)
- What is the expected row count? (one record, a batch, all records matching condition X)
- Is this a one-time migration or a repeatable operation?

### Step 2 — Write the SELECT first
Always write and show the SELECT equivalent before the DML:

```sql
-- Verify scope first
SELECT id, column_to_update
FROM table_name
WHERE <condition>;

-- Then the actual update
UPDATE table_name
SET column_to_update = new_value
WHERE <condition>;
```

The SELECT makes the scope visible. The developer can run it first and
confirm row count before executing the DML.

### Step 3 — State the expected impact explicitly
Always include a comment above the DML:

```sql
-- Expected: updates 1 row (record id = 42)
UPDATE table_name
SET status = 'ACTIVE'
WHERE id = 42;
```

If the expected count is "all rows" — flag it explicitly and confirm with
the user before proceeding.

### Step 4 — Wrap in a transaction for migrations
For any non-trivial DML, wrap in a transaction:

```sql
BEGIN;

-- Verify scope
SELECT id FROM table_name WHERE <condition>;

-- DML
UPDATE table_name SET column = value WHERE <condition>;

-- Review before committing
COMMIT;
-- or ROLLBACK; if something looks wrong
```

## Scope Verification Checklist

Before presenting any DML to the user, verify:

- [ ] Does the WHERE clause match the stated intent exactly?
- [ ] Could the WHERE clause match more rows than intended?
- [ ] Is there a risk of NULL handling causing unintended matches?
- [ ] Are joins in the WHERE clause narrowing scope correctly?
- [ ] Is the SELECT equivalent included for manual verification?

## Anti-Patterns

- **WHERE-less UPDATE/DELETE** — the single most dangerous SQL mistake
- **Trusting column names** — `WHERE is_deleted = false` looks safe but
  NULL handling may include unexpected rows
- **Implicit scope assumption** — assuming "update the user record" means
  one row without verifying the WHERE uniquely identifies it
- **Skipping the SELECT** — writing the DML directly without a verification
  query forces the developer to reason about scope mentally
- **Confident presentation** — showing a DML script without stating expected
  row count signals false safety
