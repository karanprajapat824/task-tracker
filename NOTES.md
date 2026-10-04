# Bug Fix Notes

## Summary of Changes

- **#1 — AND/OR precedence in search query:** Fixed in:
  - `TaskRepository.searchTasks`
  - `db/queries/search_tasks.sql`
  - `db/oracle/task_search_package.sql`

_(Updated as each bug is fixed.)_

---

## What I Chose Not to Change and Why

_(To be filled in at the end.)_

---

## Biggest Remaining Risk

_(To be filled in at the end.)_

---

## Tools / AI Used

- **Claude Code:** Used to scan the repository, identify potential bugs, and draft fixes.
- Every identified bug was reproduced locally with `curl` **before the fix** and re-checked **after the fix**.

---

# Bug Log

## #1 — Search Query: AND/OR Precedence — FIXED

### Affected Areas

The bug was present in the search logic across:

- `TaskRepository.searchTasks` — backend JPA repository
- `db/queries/search_tasks.sql`
- `db/oracle/task_search_package.sql`
  - COUNT query
  - Cursor query

### Symptom

The search endpoint produced incorrect results in two cases:

1. **Archived tasks appeared in search results** even though archived tasks should be excluded.
2. **The status filter was ignored** whenever the title matched the search query.

### Root Cause

The search condition used `AND` and `OR` without grouping the title and description conditions.

SQL evaluates:

```sql
A AND B OR C AND D
```

as:

```text
(A AND B) OR (C AND D)
```

As a result:

- The `archived` condition only applied to the title branch.
- The `status` condition only applied to the description branch.

Therefore, a task could bypass the `archived` or `status` filters depending on which field matched the search query.

### Reproduction — Before Fix

#### Archived tasks appearing

```bash
curl "localhost:8080/api/tasks?q=endpoint&pageSize=50"
```

**Result:** 5 rows were returned, including task `id=20` with:

```text
archived=true
```

#### Status filter being ignored

```bash
curl "localhost:8080/api/tasks?q=fix&status=DONE&pageSize=50"
```

**Result:** 15 rows were returned, including tasks with:

- `OPEN`
- `IN_PROGRESS`

even though:

```text
status=DONE
```

was specified.

### Fix

Grouped the title and description search conditions so that the `archived` and `status` filters apply to the entire search expression.

Conceptually, the query changed from:

```sql
WHERE archived = FALSE
  AND title LIKE :q
  OR description LIKE :q
  AND (:status IS NULL OR status = :status)
```

to:

```sql
WHERE archived = FALSE
  AND (title LIKE :q OR description LIKE :q)
  AND (:status IS NULL OR status = :status)
```

The same logical fix was applied to the relevant SQL and Oracle package queries.

### Verification — After Fix

#### Search without status filter

```bash
curl "localhost:8080/api/tasks?q=endpoint&pageSize=50"
```

**Result:** 4 rows returned.

- No archived tasks were included.

#### Search with status filter

```bash
curl "localhost:8080/api/tasks?q=fix&status=DONE&pageSize=50"
```

**Result:** 2 rows returned.

- Both results have `status=DONE`.
- No `OPEN` or `IN_PROGRESS` tasks were returned.

---

## Status

**#1 — FIXED AND LOCALLY VERIFIED**

The backend search behavior was reproduced before the change and re-tested after the change using the affected query combinations.