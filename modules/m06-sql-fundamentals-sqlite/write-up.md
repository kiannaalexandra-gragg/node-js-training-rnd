# M06 Write-Up — SQL Fundamentals with SQLite

> Written from what I captured as I built it: raw notes after each slice, then this document. Written
> so a trainee starting this stage next cohort could follow my path on their own. Your M07 self needs
> this schema and ERD, so it is written like I'll need it again.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo,
[`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
under `modules/m06-sql-fundamentals-sqlite/`, on the branch `feat/module-06/sql-fundamentals-sqlite`.
M06 has no API, so it is a standalone stage folder like M01-M04, not part of `task-api/`.

```text
m06-sql-fundamentals-sqlite/
├── README.md              Mermaid ERD, layout, how to run
├── schema.sql             DDL for users, tasks, tags, task_tags, plus 3 indexes
├── seed.sql               5 users, 24 tasks, 7 tags, 37 task_tags rows
├── exercises/             25 graded queries in 5 files
│   ├── 01-select-filter-sort.sql          exercises 1-6
│   ├── 02-joins.sql                       exercises 7-11
│   ├── 03-aggregates.sql                  exercises 12-16
│   ├── 04-subqueries-and-dml.sql          exercises 17-20
│   └── 05-transactions-and-indexes.sql    exercises 21-25
└── injection-demo/
    ├── package.json, package-lock.json, tsconfig.json
    └── src/  db.ts  vulnerable.ts  fixed.ts  demo.ts
```

- **The schema**: four tables. `users` 1:N `tasks`, and `tasks` M:N `tags` through the junction table
  `task_tags`. `tasks.id`, `title`, `description` and `status` are the fields M05's API contract
  already fixes. `users` and `tags` live only in the database, because M05 exposes only tasks.
- **Seed data** with every id written out, so each exercise gives the same answer on every run.
- **25 exercises**, each tagged easy, medium or hard in its header. The brief asks for 20+.
- **Two transaction exercises that roll back and one that commits** (21, 22, 23).
- **Two `EXPLAIN QUERY PLAN` exercises** that show a scan become a search once an index exists (24, 25).
- **A SQL injection demo** in TypeScript with `better-sqlite3`: a vulnerable task search built by string
  concatenation, two attacks that succeed against it, and the same search with a `?` parameter, where
  the same two attacks return nothing. `npm run demo` exits 0 only if both sides behave that way.
- **A Mermaid ERD** in the module `README.md`.
- **Merged to `main` by pull request.** Four commits on `feat/module-06/sql-fundamentals-sqlite`, merged as
  [#6](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/6) (see "Checkpoint evidence").

The stack as installed: Node 24.22.0 (the repo's `.nvmrc` says `24`; my shell defaulted to 22.23.0, so I
ran `nvm use 24`), the `sqlite3` CLI 3.51.0, `better-sqlite3@12.11.1` (which bundles its own SQLite,
3.53.2), `typescript@7.0.2`, `tsx@4.23.15`, `@types/better-sqlite3@9.6.0` and `@types/node@24.19.2`.
`npm run typecheck` exits 0 in `injection-demo/`.

## Why it's built this way (key decisions)

### The `task_tags` junction table

A task has many tags and a tag labels many tasks. A single column cannot hold that: a `tag` column on
`tasks` would hold one tag, and a comma-separated `tags` text column would break 1NF and make "which
tasks have the `urgent` tag" a string search that no index can help. So the relationship becomes its
own table, with one row per (task, tag) pair.

- **Composite primary key `(task_id, tag_id)`, no surrogate `id`.** A row is identified by the pair.
  The primary key also stops the same tag being attached to the same task twice. I tried it: a second
  `INSERT INTO task_tags VALUES (2, 1)` failed with
  `UNIQUE constraint failed: task_tags.task_id, task_tags.tag_id`. The error says `UNIQUE`, not
  `PRIMARY KEY`, because SQLite builds the primary key on top of a unique index.
- **Both columns are foreign keys with `ON DELETE CASCADE`.** Deleting a task, or a tag, removes only the
  link rows, and no orphan pair is left pointing at nothing. Exercise 20 shows the chain: deleting a user
  cascades to their tasks, which cascades to those tasks' link rows (counts went from 5 users, 24 tasks and
  37 links to 4 users, 19 tasks and 30 links, after one `INSERT`, one `UPDATE` and the `DELETE`).
- **No extra columns on the junction table.** If a link ever needs its own data (who tagged it, when),
  that is the moment to give it a surrogate key or more columns. Nothing in this stage needs it, so I did
  not add it.

### Other schema choices

- **`INTEGER PRIMARY KEY`** for every id. In SQLite that makes the column an alias for the row id, so it
  is assigned automatically. M05's in-memory store uses UUID strings, so M07 will have to decide how those
  ids map. I did not change M05.
- **`tasks.user_id` is `NOT NULL`, with `ON DELETE CASCADE`.** A task with no owner is not a thing in
  this model. Cascade means deleting a user takes their tasks with them. Exercise 23 shows how to avoid
  that when it is not wanted: hand the tasks to someone else in the same transaction first.
- **`CHECK` constraints for `status` and `priority`**, and `CHECK (length(title) > 0)`. The three allowed
  statuses are the same three M05 validates in TypeScript, so the rule now lives in the database as well.
- **Dates as `TEXT`** (`'2026-03-10'` or `'2026-03-10 09:00:00'`). SQLite has no date type. ISO 8601 text
  sorts in date order, so `ORDER BY` and `BETWEEN` work on it.
- **`PRAGMA foreign_keys = ON` at the top of `schema.sql`, `seed.sql` and every exercise file**, because
  SQLite turns it off by default and it applies to one connection only (see "What tripped me up").

### What I indexed

SQLite indexes the primary key automatically and does **not** index foreign key columns. I added three:

| Index | Why |
| --- | --- |
| `idx_tasks_user_id` on `tasks (user_id)` | Every "tasks of this user" query and every join from `users` to `tasks` filters on it |
| `idx_tasks_status` on `tasks (status)` | Filtering by status is the most common list filter in the API |
| `idx_task_tags_tag_id` on `task_tags (tag_id)` | The primary key `(task_id, tag_id)` helps lookups by `task_id` but not by `tag_id` |

I checked the last claim instead of assuming it. `EXPLAIN QUERY PLAN` on `WHERE task_id = 5` gave
`SEARCH task_tags USING COVERING INDEX sqlite_autoindex_task_tags_1 (task_id=?)`, and on `WHERE tag_id = 5`
it gave `SEARCH task_tags USING INDEX idx_task_tags_tag_id (tag_id=?)`. The first one is the primary key's
own index. The second needed an index of its own.

What changes without them is in exercises 24 and 25. With `idx_tasks_status` dropped, the plan for
`WHERE status = 'todo'` was `SCAN tasks`. After `CREATE INDEX`, it was
`SEARCH tasks USING INDEX idx_tasks_status (status=?)`. Exercise 25 did the same for `due_date`
(`SCAN tasks`, then `SEARCH ... USING INDEX idx_tasks_due_date (due_date<?)`). I left `due_date` out of
`schema.sql`: with 24 rows an index changes nothing a person would notice, and I would be indexing for a
query I do not have yet. A scan only hurts when the table is big, and this one is tiny, so these exercises
show the plan changing, not the time.

### Pagination

Exercise 5 pages with `LIMIT 5 OFFSET 5`. Exercise 6 does the same job by keyset: `WHERE (created_at, id) > (?, ?)
ORDER BY created_at, id LIMIT 5`, where the two values come from the last row of the previous page. The
plans differ. Without an index, the keyset query was `SCAN tasks` plus `USE TEMP B-TREE FOR ORDER BY`.
After `CREATE INDEX idx_tasks_created_at_id ON tasks (created_at, id)` it was
`SEARCH tasks USING INDEX idx_tasks_created_at_id (created_at>?)`. It is not a `COVERING INDEX`, because
the exercise also reads `title`, which is not in the index. Selecting only `id` makes the same query plan as
`COVERING INDEX`. So keyset pagination is only fast if there is an index in the sort order, and the `id` in the key is a tie-breaker so two rows
with the same `created_at` cannot be skipped or repeated. I did not keep that index in the schema, for the
same reason as `due_date`.

### The injection demo

- **One function, three payloads.** The vulnerable function is a task search by title,
  `WHERE title LIKE '%${term}%'`. I used a search box on purpose: it is the most natural place for a
  trainee to write string concatenation, and it needs no login flow.
- **The demo checks its own outcome.** The brief says to show the attack succeed and the fix block it, not
  to assert it. `demo.ts` compares the vulnerable result against what proves the attack worked (24 rows
  for the tautology, every user's email for the `UNION`) and the fixed result against zero rows, prints
  `SUCCEEDED` or `BLOCKED`, and exits 1 if either side is wrong.
  After a code review I added controls, because `BLOCKED` only means the fixed query returned nothing: for a
  normal term (`Fix`) the fixed query must find the same tasks as the vulnerable one, and `zzz` must find
  nothing on both sides. I checked that they work by breaking `fixed.ts` on purpose, once so it always
  returns `[]` (the demo then prints `Controls: FAILED` and exits 1) and once so it still concatenates (it
  prints `STILL LEAKING` and exits 1), then restored the file.
- **An in-memory database built from `../schema.sql` and `../seed.sql`**, not a copy of the tables. The
  demo attacks the same tables the exercises use, and it leaves no `.db` file to ignore.
- **`.ts` import specifiers** (`import ... from './fixed.ts'`), the same as M04 and M05. The import
  extension convention is still an open decision for the program (P2). I followed what my own M04 and M05
  already do and did not treat it as settled.
- **`better-sqlite3`**, not Node's built-in `node:sqlite`, because it is in the program's pinned stack and
  is the driver M07's Prisma adapter uses, so it is the one worth getting familiar with.

### Which exercises taught me something I didn't expect

- **Exercise 6.** I wrote a comment saying keyset pagination "starts from an index". When I ran
  `EXPLAIN QUERY PLAN` the comment was wrong for this schema: there is no index on `created_at`, so it
  scanned and sorted. The comment now says what the plan really shows, with and without the index.
- **Exercise 21.** This one made something visible that I had only read about. After the second `INSERT`
  broke the `status` check, a `SELECT COUNT(*)` inside the same transaction returned 25: the first insert
  was still there, waiting. Only `ROLLBACK` brought the count back to 24. A failed statement in the
  `sqlite3` CLI fails by itself and leaves the transaction open. Your code has to decide to roll back.
- **Exercise 13 (a check, not a surprise).** I wrote in the comment that `COUNT(*)` would give Esme 1 and
  `COUNT(t.id)` 0. I ran both before keeping the comment, and got `1` and `0`. A `LEFT JOIN` produces one
  row for a user with no tasks, so `COUNT(*)` counts that row.

## How to build it (teach it to the next trainee)

A guide to designing a junction table for a many-to-many relationship, using `tasks` and `tags`.

### Step 1: Say the relationship in both directions

Ask two questions, one from each side:

- Can one task have more than one tag? Yes (task 5 has `backend`, `bug` and `urgent`).
- Can one tag be on more than one task? Yes (`backend` is on 12 tasks).

"Many" in both directions is many-to-many. If one answer had been "no", it would be one-to-many, and the
foreign key would go on the "many" side with no extra table. That is how `tasks.user_id` works: a task has
one owner, a user has many tasks, so the key sits on `tasks`.

### Step 2: Why a column on either table cannot hold it

A single `tag_id` on `tasks` would give one tag per task. A text column of `'backend,bug,urgent'` cannot be
indexed or constrained, `WHERE tags LIKE '%bug%'` also matches a tag called `debug`, and removing one tag
means rewriting the string. So the link needs a table of its own.

### Step 3: Give the junction table one row per pair

```sql
CREATE TABLE task_tags (
  task_id INTEGER NOT NULL REFERENCES tasks (id) ON DELETE CASCADE,
  tag_id  INTEGER NOT NULL REFERENCES tags (id)  ON DELETE CASCADE,
  PRIMARY KEY (task_id, tag_id)
);
```

- Two foreign keys, one to each side. They carry the relationship.
- The composite primary key is the pair. It is what makes "this task has this tag" something you can only
  say once.
- `ON DELETE CASCADE` on both: deleting a task or a tag removes only the link rows.
- No `id` column. Nothing refers to a single link row, so a surrogate key would be an extra index for no
  benefit.

### Step 4: Index the side the primary key does not cover

The primary key `(task_id, tag_id)` is an index sorted by `task_id` first, so it serves "tags of a task".
"Tasks with a tag" filters on `tag_id` alone, so add `CREATE INDEX idx_task_tags_tag_id ON task_tags (tag_id)`.
Check it with `EXPLAIN QUERY PLAN` and look for `SEARCH`, not `SCAN`.

### Step 5: Query across it

To get tasks with their tags, join through the junction table. Use `LEFT JOIN` if tasks with no tags must
still appear (tasks 6, 12, 19 and 24 have none in my seed data), and `GROUP BY` the task with
`group_concat` to get one row per task:

```sql
SELECT t.id, t.title, group_concat(g.name, ', ') AS tags
FROM tasks AS t
LEFT JOIN task_tags AS tt ON tt.task_id = t.id
LEFT JOIN tags AS g ON g.id = tt.tag_id
GROUP BY t.id;
```

An `INNER JOIN` here would drop the four untagged tasks. Exercise 10 finds exactly them.

## Concepts worth explaining

### Why parameterised queries stop injection

A query built by string concatenation reaches the database as one piece of text. The database parses that
text, and it cannot know which characters the developer wrote and which a user typed. In my demo, the
search term `zzz' OR 1=1 --` turned

```sql
SELECT id, title, status FROM tasks WHERE title LIKE '%zzz%' ORDER BY id
```

into

```sql
SELECT id, title, status FROM tasks WHERE title LIKE '%zzz' OR 1=1 --%' ORDER BY id
```

The quote closed the string, `OR 1=1` is a new condition, and `--` turned the rest of the line into a
comment. The term stopped being data and became part of the program.

A parameterised query sends two things separately: the SQL text with `?` where a value goes, and the
values. The text is parsed and planned first, with the `?` as a hole in the grammar where only a value fits.
The value is attached afterwards, so it cannot add a condition or a quote that closes anything. Fed
`zzz' OR 1=1 --`, the fixed query searched for tasks whose title contains that exact string, and found none.

Two things the parameter does not do. It does not clean the value, because it never needs to. And it
cannot stand in for a table or column name or a keyword, only for a value. A sort column that comes from
the user has to be checked against a list of allowed names instead. I did not build that case in this
stage.

### 1:N against M:N

Where the foreign key goes decides the relationship. One-to-many is a foreign key on the "many" side
pointing at the "one": `tasks.user_id` points at `users.id`. Many-to-many cannot be a foreign key on either
side, because each row would need to point at several rows. So it becomes two one-to-many relationships
meeting in a junction table: `tasks` 1:N `task_tags`, and `tags` 1:N `task_tags`. The junction table is the
only table in the schema whose primary key is made of two foreign keys.

## What tripped me up

- **My first tautology payload proved nothing.** I wrote `' OR 1=1 --` as the term and the vulnerable
  query returned all 24 tasks, and the first version of my demo printed `SUCCEEDED` for it. But with an empty term the query
  reads `LIKE '%'`, which already matches every row. I only caught it while checking my own comment against a
  run of the classic payload `' OR '1'='1`: it also returned 24 rows, and again would have with no attack.
  The fix was a term with a prefix that matches nothing, `zzz' OR 1=1 --`, and a baseline line in the demo
  showing that `zzz` alone returns 0 rows. The 24 rows now can only come from the injected condition.
- **The stacked `DROP TABLE` did not work the way the textbook says.** With `db.prepare(sql).all()`,
  `better-sqlite3` threw `The supplied SQL string contains more than one statement`, and the `tasks` table
  still had 24 rows. That is a driver safeguard, not a fix. When I ran the same vulnerable string through
  `db.exec(...)`, which accepts many statements, the next query failed with `no such table: tasks`, so
  the table was gone. The only thing protecting the first version was which method I called.
- **Foreign keys did nothing until I turned them on.** On a plain `sqlite3 m06.db` connection with no pragma,
  `INSERT INTO tasks (user_id, title) VALUES (999, 'orphan')` succeeded, and the row sat there pointing at
  a user that does not exist. With `PRAGMA foreign_keys = ON` the same statement failed with
  `FOREIGN KEY constraint failed`. The pragma is per connection and is not saved in the file.
- **SQLite did not reject a word in an `INTEGER` column.** After `CREATE TEMP TABLE t(n INTEGER)`,
  inserting `'42'` stored the integer 42 and inserting `'abc'` stored the text `abc`, with no error.
  `typeof(n)` showed `integer` and `text`. A column type in SQLite is an affinity, a preference, not a
  rule. This is why the `CHECK` constraints matter here.
- **`= NULL` matches nothing.** `SELECT count(*) FROM tasks WHERE due_date = NULL` returned `0`. It has
  to be `IS NULL` (exercise 3 returns 5 rows).
- **`LIKE` ignored case and `GLOB` did not.** `WHERE title LIKE '%fix%'` returned 4 rows, and
  `GLOB '*fix*'` returned 0, because the titles say `Fix`.
- **I wrote the wrong row count in a comment.** `seed.sql`'s header first said `task_tags 40`. The real
  count query said 37. I fixed the comment, and now every number in a header comes from a run.
- **A failed statement does not end a transaction** (exercise 21, above).
- **`npm install` printed a warning about install scripts not covered by `allowScripts`** for
  `better-sqlite3` and `esbuild`. I did not approve anything. `better-sqlite3` loaded and ran
  (`select sqlite_version()` returned `3.53.2`), so the warning did not break the install here. I have not
  tested what happens on a machine with no prebuilt binary.
- **I was on the wrong branch to start.** The build repo's local checkout was still on the merged M05
  branch. I switched to `main`, pulled, and made the M06 branch from there, so the M06 work does not sit
  on top of M05's history.

## Checkpoint evidence

The checks below come from real runs on 21 October 2026.

**20+ exercises.** 25 exercises across five files; each file runs against a fresh `schema.sql` plus
`seed.sql` database and prints its exercises under `--- Exercise N` headings. Coverage:

| Required | Exercises |
| --- | --- |
| Filtering | 1, 2, 3, 4 |
| Sorting and pagination | 1, 4, 5, 6 |
| `INNER` and `LEFT` joins | 7, 8, 9, 10, 11, 13, 15 |
| Aggregates with `GROUP BY` / `HAVING` | 12, 13, 14, 15, 16 |
| At least one subquery | 17, 18, 19 |
| DML | 20, 23 |
| A transaction | 21, 22 (rolled back), 23 (committed) |
| Indexes and `EXPLAIN QUERY PLAN` | 24, 25 |

A few real results, to show what running them looked like:

```text
--- Exercise 8: users who own no tasks
id  name
5   Esme Walker

--- Exercise 14: users with at least 2 finished tasks
Ada Lovelace  2     Ben Carter  2     Dev Patel  2

--- Exercise 21: tasks_before 24, tasks_inside_transaction 25, tasks_after 24
--- Exercise 22: tasks_inside_transaction 17, tasks_after_rollback 24
--- Exercise 23: after the COMMIT, Esme Walker owns 6 tasks and user 4 is gone

--- Exercise 24: SCAN tasks  ->  SEARCH tasks USING INDEX idx_tasks_status (status=?)
```

**The ERD matches the schema.** I read `PRAGMA table_info` for each table and compared the column names
and order with the `erDiagram` block in the README using a script: `users`, `tasks`, `tags` and `task_tags`
all matched, and the set of tables matched. `PRAGMA foreign_key_list` showed the three foreign keys the
diagram draws, all `ON DELETE CASCADE`. I parsed the diagram with Mermaid 11 (`mermaid.parse` returned
`diagramType: 'er'`), so it is valid Mermaid.

**The injection demo.** `npm run demo` exits 0 and prints, in full:

```text
Database: 24 tasks, 5 users.

Normal use, term "Fix":
  vulnerable SQL: SELECT id, title, status FROM tasks WHERE title LIKE '%Fix%' ORDER BY id
    rows: 4 -> 5: Fix the duplicate email bug | 9: Fix the broken date picker | 15: Fix the slow list query, ... (+1 more)
    rows: 4 -> 5: Fix the duplicate email bug | 9: Fix the broken date picker | 15: Fix the slow list query, ... (+1 more)

Normal use, term "zzz" (the baseline for the attacks below, no task has this title):
    rows: 0
    rows: 0

Controls: OK (the fixed query must still search, and "zzz" must match nothing)

Attack: Tautology: make the WHERE clause always true
  term: zzz' OR 1=1 --
  vulnerable SQL: SELECT id, title, status FROM tasks WHERE title LIKE '%zzz' OR 1=1 --%' ORDER BY id
  vulnerable: SUCCEEDED
    rows: 24 -> 1: Set up the Express project | 2: Design the tasks table | 3: Write the seed script, ... (+21 more)
  fixed:      BLOCKED
    rows: 0

Attack: UNION: read another table through the search box
  term: zzz' UNION SELECT id, email, name FROM users --
  vulnerable SQL: SELECT id, title, status FROM tasks WHERE title LIKE '%zzz' UNION SELECT id, email, name FROM users --%' ORDER BY id
  vulnerable: SUCCEEDED
    rows: 5 -> 1: ada@example.com | 2: ben@example.com | 3: chloe@example.com, ... (+2 more)
  fixed:      BLOCKED
    rows: 0

Attack: stacked statement, a second statement after the search
  term: '; DROP TABLE tasks; --
  vulnerable SQL: SELECT id, title, status FROM tasks WHERE title LIKE '%'; DROP TABLE tasks; --%' ORDER BY id
  vulnerable: the driver refused: The supplied SQL string contains more than one statement
  tasks table after the attempt: 24 rows
  fixed: 0 rows, the text is only compared, never run
```

`npm run typecheck` exits 0.

**Self-explanation (primary vs foreign key, an index, 1:N vs M:N)**, in my own words:

- A primary key identifies one row of its own table: unique, never null. A foreign key is a column that
  holds another table's primary key, so a row can point at the row it belongs to, and the database can
  refuse a pointer to a row that does not exist.
- An index is a separate sorted structure over one or more columns. It lets the database jump to the
  matching rows instead of reading the whole table, at the cost of extra space and slower writes.
- 1:N puts a foreign key on the "many" side. M:N needs a junction table with a foreign key to each side.

**M06 merged to `main` by pull request.**
[#6, `feat(m06): add SQLite schema, 25 exercises and injection demo`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/6),
from `feat/module-06/sql-fundamentals-sqlite` into `main`. `gh pr view 6` reported the state `MERGED` on
2026-10-21 (07:35 UTC), with those four commits. It was merged with a merge commit, so they are visible in the
history:

```text
$ git log --oneline --graph origin/main -7
*   7f88f42 Merge pull request #6 from kiannaalexandra-gragg/feat/module-06/sql-fundamentals-sqlite
|\
| * fca09b7 docs(m06): add Mermaid ERD and how to run
| * 19ee563 feat(m06): add SQL injection demo and parameterised fix
| * 02eac99 feat(m06): add 25 graded query exercises
| * 87755f8 feat(m06): add users, tasks, tags and task_tags schema and seed data
|/
*   34dc299 Merge pull request #5 from kiannaalexandra-gragg/feat/module-05/express-task-api-v1
|\
| * 4f00656 docs(m05): describe the task-api layout and the v1 contract
```

The test plan in the PR lists the checks whose results are shown above, and two it leaves unchecked: the ERD
was not looked at rendered on GitHub, and a machine with no prebuilt `better-sqlite3` binary was not tried.

## What I'd do differently

- **Run the benign version of every attack first.** The false tautology printed a green `SUCCEEDED`
  that proved nothing. Next time the baseline goes into the demo before the first payload does.
- **Count before I write a number in a comment.** The `task_tags 40` in the seed header was wrong, and I
  only noticed because I ran the count.
- **Check every explanatory comment in the exercises against a plan or a run.** My exercise 6 comment was a
  claim I had not run, and it was wrong.
