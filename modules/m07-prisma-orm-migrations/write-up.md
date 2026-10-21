# M07 Write-Up — Prisma ORM & Migrations (Task API v2)

> Written from what I captured as I built it: raw notes after each slice, then this document. Written
> so a trainee starting this stage next cohort could follow my path on their own.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo,
[`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
inside `task-api/`, on the branch `feat/module-07/prisma-orm-migrations`. The Task API is one project
that grows from M05 to M09, so v2 is a change to `task-api/` in place, not a new folder.

```text
task-api/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   │   ├── 20261021082413_init/migration.sql
│   │   ├── 20261021082429_add_priority_due_date/migration.sql
│   │   ├── 20261021082448_add_task_owner/migration.sql      hand-edited, made with --create-only
│   │   └── migration_lock.toml
│   └── seed.ts
├── prisma7.config.ts                  Prisma CLI config (datasource URL, paths, seed command)
├── src/
│   ├── db.ts                          the one PrismaClient, with the better-sqlite3 adapter
│   ├── services/task-service.ts       replaces store/task-store.ts, which is deleted
│   ├── handlers/tasks.handlers.ts     calls the service, returns { data, meta } from the list
│   ├── validation/task-validation.ts  list query, priority and dueDate added
│   ├── middleware/error-handler.ts    maps P2002, P2025, P2003 inside the M05 handler
│   ├── types/task.ts                  Task types now come from the generated client
│   ├── server.ts                      disconnects the client on shutdown
│   └── generated/prisma/              the generated client (git-ignored)
├── scripts/check-prisma-errors.ts     makes real P2002, P2025 and P2003 errors
└── bruno/                             3 requests updated, 7 added (Listing/ folder, Errors/23 and 24)
```

- **The schema**: `User`, `Task` and `Tag`, with `User` 1:N `Task` and an implicit many-to-many between
  `Task` and `Tag`. The M06 `task_tags` junction table is the join table `_TagToTask`, which Prisma
  creates and manages.
- **Three migrations.** `init` creates the tables. `add_priority_due_date` adds two columns.
  `add_task_owner` adds a required `userId` to a table that already had rows, and it is the hand-edited
  one.
- **A service layer** (`taskService`: `list`, `get`, `create`, `update`, `remove`) with the same shape
  as the M05 store, so the handlers changed by a few lines.
- **`GET /api/v1/tasks`** filters by `status` and `q`, sorts by `sortBy` and `order`, pages by `page`
  and `pageSize`, and returns `{ data, meta: { page, pageSize, total, totalPages } }`.
- **A seed script**, `npm run db:seed`, loading the same 5 users, 24 tasks, 7 tags and 37 task-tag
  links as the M06 `seed.sql`.
- **Prisma errors mapped inside the M05 handler**: P2002 to 409, P2025 to 404, P2003 to 409. The
  handler is the same function. It gained one branch.
- **`task-api/README.md`** now describes v2: setup, scripts, the list query, the errors and the
  migrations.

The stack as installed: Node 24.22.0 (my shell defaulted to 22.23.0, so I ran `nvm use 24` first),
`prisma@7.10.1` and `@prisma/client@7.10.1` (`npx prisma@7 --version` printed `7.10.1`, TypeScript
`7.0.2`, "Query Compiler: enabled"), `@prisma/adapter-better-sqlite3@7.10.1`, `better-sqlite3@12.11.1`
(SQLite 3.53.x), `@types/better-sqlite3@9.6.0`, `typescript@7.0.2`. `npm run typecheck` exits 0.

## Why it's built this way (key decisions)

### The schema, translated from M06

M06's `schema.sql` and the Prisma schema describe the same data, but they do not map one to one.

| M06 (SQL) | M07 (Prisma) | Why |
| --- | --- | --- |
| `tasks.id INTEGER PRIMARY KEY` | `id String @id @default(uuid())` | M05's contract, its UUID check (`parseTaskId`) and every Bruno error case already use UUID ids. Switching to integers would change the contract. M06's backlog had left this decision for M07. |
| `users.id`, `tags.id` integers | `Int @id @default(autoincrement())` | Users and tags are never exposed by the API, so there is nothing to keep stable. |
| `task_tags` junction table | `tags Tag[]` on `Task` and `tasks Task[]` on `Tag` | An implicit many-to-many. Prisma creates `_TagToTask` itself. I gave up the right to add columns to the link (who tagged it, when). M06 said that is the moment to switch to an explicit model, and nothing here needs it. |
| `CHECK (status IN (...))` | `enum TaskStatus` | On SQLite an enum is a plain `TEXT` column. The generated SQL has **no** `CHECK`, so Prisma enforces the three values and the database does not. M06's database-level rule is weaker here. |
| `priority`, `due_date`, `user_id` all in one `CREATE TABLE` | added in migrations 2 and 3 | The brief asks for a second migration adding a column, and a hand-edited one. M05 had no priority, due date or owner, so adding them in order is what really happened to this API. |
| `created_at TEXT DEFAULT (datetime('now'))` | `createdAt DateTime @default(now())`, `updatedAt DateTime @updatedAt` | Prisma has a real date type, and `@updatedAt` is filled in by the client on every update. |
| `ON DELETE CASCADE` on `user_id` | `onDelete: Cascade` on the `owner` relation | Same rule as M06: deleting a user deletes their tasks. |
| three indexes | `@@index([status])`, `@@index([userId])` | `_TagToTask` gets its own indexes from Prisma. I did not add `dueDate`, for the M06 reason: 24 rows prove nothing. |

**The API's tasks need an owner, but the API has no users.** M06's `tasks.user_id` is `NOT NULL`, and
v1 has no user concept or login. I kept the rule and gave the API one default owner,
`owner@example.com`. The migration creates it, the seed script creates it if it is missing, and the service
creates every task under it. If it is missing, `create` throws a plain `Error`, which the handler turns
into a 500, because that means the database was not migrated, which is not the client's mistake.

### What I hand-edited in the `--create-only` migration, and why `migrate dev` couldn't generate it

After migration 2, I inserted three tasks with `sqlite3`, then added a required `userId` to `Task` in
the schema and ran `npx prisma@7 migrate dev --create-only --name add_task_owner`. Prisma refused to
plan it:

```text
⚠️ We found changes that cannot be executed:

  • Step 0 Added the required column `userId` to the `Task` table without a default value. There are 3 rows in this table, it is not possible to execute this step.
```

It still wrote the file, with a warning comment at the top and SQL whose `INSERT ... SELECT` copied
every column except `userId`. Applying that would fail on the three rows. Prisma can see the schema
change, but it cannot know which user the existing tasks should belong to, and that is a business
decision, not a schema fact. I made two changes by hand:

1. **Before** the table redefinition, create the owner the rows will point at:

   ```sql
   INSERT INTO "User" ("email", "name", "createdAt")
   VALUES ('owner@example.com', 'Default Owner', CURRENT_TIMESTAMP)
   ON CONFLICT ("email") DO NOTHING;
   ```

2. In the `INSERT INTO "new_Task" ... SELECT`, add `"userId"` to the column list and select the
   owner's id for it:

   ```sql
   (SELECT "id" FROM "User" WHERE "email" = 'owner@example.com')
   ```

Then `npx prisma@7 migrate dev` applied it. Afterwards all three tasks had `userId` 1,
`PRAGMA foreign_key_check` printed nothing, and a second `migrate dev` said
`Already in sync, no schema change or pending migration was found.`

### How I mapped P2002, P2025 and P2003 onto the M05 handler

I did not add a handler. `toAppError` in `error-handler.ts` already turns anything thrown into an
`AppError`, so Prisma's errors became one more case in it. If the error is a
`Prisma.PrismaClientKnownRequestError`, a small function maps its `code`:

| Prisma code | Means | Becomes |
| --- | --- | --- |
| `P2002` | a unique constraint failed | 409 `CONFLICT` |
| `P2025` | the record the operation needs does not exist | 404 `NOT_FOUND` |
| `P2003` | a foreign key constraint failed | 409 `CONFLICT` |

- **P2003 is 409, not 400.** The brief says "409 or 400". The request was well formed and it conflicts
  with the rows that exist (an owner that does not exist, or one that still has tasks), which is what
  409 means. This is my choice, not the program's.
- **The messages are fixed.** Prisma's own text names tables and columns. The handler never puts it in
  a response, the same rule M05 set for unexpected errors.
- **Any other Prisma code falls through** to the generic 500, so an outage or a bug is not dressed up
  as a client error.
- `ConflictError`, defined in M05 and thrown by nothing until now, is finally used.
- **`update` and `remove` no longer look the task up first.** Prisma throws P2025 when the row is
  missing, so the handler's `if (task === undefined) throw new NotFoundError(...)` lines are gone.
  `get` still uses `findUnique`, which returns `null`, and throws `NotFoundError` itself.

### The list endpoint

- **`skip`/`take` (offset) pagination, not keyset.** M06 exercise 6 showed keyset. The brief's contract
  is `page` and `pageSize`, which is offset by definition.
- **The sort always ends with `{ id: 'asc' }`** as a tie-breaker, so two tasks with the same priority
  cannot swap places between requests.
- **`findMany` and `count` run in one `$transaction([...])`**, so `total` describes the same rows that
  `data` was cut from.
- **`sortBy` is checked against a list of five names**, not passed through, so a client cannot sort by
  a column that is not meant to be public.
- **A bad query value is a 400, never ignored.** Same rule as v1's `status` filter. `details` lists
  every bad parameter in one response.
- **`pageSize` is capped at 100.** A larger value is a 400, not a silent cap.
- **`q`** matches the title or the description with `contains`. SQLite's `LIKE` ignores case for ASCII
  letters, so `?q=ZEBRA` found a title written `zebra`.

### Other decisions

- **`prisma7.config.ts`, not `prisma.config.ts`.** Installed Prisma is 7.10.1, which is 7.10 or later,
  and `prisma init` generated the 7.10 name on its own.
- **`.env` is loaded with `process.loadEnvFile`**, guarded by `existsSync`. `prisma init` wrote
  `import "dotenv/config"`, but `dotenv` is not installed and Node 24 can load the file itself.
- **`DATABASE_URL` is read twice**: by the CLI through the config file, and by `src/db.ts` for the
  running app. The app never reads the config file.
- **Query logging is opt-in** (`LOG_QUERIES=true`). My first version logged queries whenever
  `NODE_ENV=development`, and a seed of 24 tasks printed several hundred lines of SQL.
- **The types come from the schema.** `Task` in `types/task.ts` is the generated model type, and
  `TASK_STATUSES` is `Object.values()` of the generated enum, so the list `validation` checks and the
  database cannot disagree.
- **Bruno: the v1 requests that asserted the v1 shape changed.** 02 asserted exactly four keys, and 03
  and 04 asserted `isArray`. v2 adds fields and wraps the list, so those three assertions moved with the
  contract. Every Errors request is unchanged except one added. I added a `Listing/` folder that
  creates its own task, searches for it, sorts, pages and deletes it, so it passes on an unseeded
  database too.
- **The empty-PATCH message still says "title, description, status".** Bruno request 12 asserts that
  exact text. `priority` and `dueDate` now also satisfy the "at least one field" rule, so the message
  is incomplete. I left it so the v1 error contract stays green.

## How to build it (teach it to the next trainee)

### Adding a required column to a table that already has data

Example: every task must have an owner (`userId`), and the `Task` table already holds three rows.

**1. Understand what a required column means for existing rows.** A required column is `NOT NULL`.
Every existing row needs a value for it the moment the column exists. There are only three ways to get
one: give the column a default, make it optional now and tighten it later, or backfill the value by
hand in the migration. A default only works if one value is right for every old row. Here no value is
"right" until you decide who owns old tasks, so the choice is yours, not Prisma's.

**2. Put some real rows in the table, so you see the problem the way production would.** On an empty
table the migration just works, and teaches nothing. I used `sqlite3`:

```bash
sqlite3 dev.db "INSERT INTO Task (id,title,status,updatedAt) VALUES ('aaaaaaaa-0000-4000-8000-000000000001','Write the schema','done',CURRENT_TIMESTAMP);"
```

(I inserted three rows the same way.)

**3. Change the schema.** Add the field and the relation, and the inverse side on `User`:

```prisma
model Task {
  // ...
  userId Int
  owner  User @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId])
}

model User {
  // ...
  tasks Task[]
}
```

**4. Make the migration file without applying it.** `--create-only` writes the SQL and stops, so you
can read it before the database changes:

```bash
npx prisma@7 migrate dev --create-only --name add_task_owner
```

Prisma prints the warning shown above and writes the file. Do not run it yet.

**5. Read the file.** Prisma uses the same recipe for every column change on SQLite: create
`new_Task` with the new shape, copy the rows across with `INSERT INTO "new_Task" ... SELECT ... FROM
"Task"`, drop the old table, rename the new one. The copy lists every column except `userId`, and
`userId` is `NOT NULL`, so it would fail on the first row.

**6. Edit the SQL by hand: create the value, then use it.** The owner must exist before any task can
point at it, so the new `INSERT INTO "User"` goes at the top, and the `userId` column and a
`(SELECT "id" FROM "User" WHERE "email" = ...)` go into the copy (the two edits are shown above).
Keep Prisma's warning comment and add a note saying what you changed and why, so the next person
understands why this file differs from a generated one.

**7. Apply it and check it:**

```bash
npx prisma@7 migrate dev
sqlite3 dev.db "SELECT id, title, userId FROM Task; PRAGMA foreign_key_check;"
npx prisma@7 migrate dev        # must now say: Already in sync
npm run db:generate             # the client must learn the new field; migrate dev does not do it
```

All three rows had `userId` 1, and the foreign key check printed nothing.

**8. Never edit it again once it is applied.** Every applied migration's checksum is stored in the
database. To change the schema again, make a new migration. I tested this on a throwaway database: I
added one comment line to an applied migration, `migrate status` still said `Database schema is up to
date!`, and `migrate dev` said `The migration ... was modified after it was applied. We need to reset
the SQLite database`. I restored the file, and the hash matched the original.

**9. Test it from scratch, not just on the database you migrated.** `migrate reset` throws the data
away and replays all migrations on an empty database, where the backfill copies zero rows and the
owner is still created. That is the run that proves the hand-edited file works in both cases.

## Concepts worth explaining

### `migrate dev` against `migrate deploy`

`migrate dev` is for the developer's machine, and it **writes**: it compares `schema.prisma` with the
migration history, creates a new migration folder when the schema has changed, applies it, and if an
applied migration has been edited it asks to reset the database and delete the data (I saw that
message when I added a comment line to an applied migration on a throwaway database). `migrate
deploy` is for CI and production, and it **only applies**: it takes the migrations already committed
in `prisma/migrations/`, applies the ones not yet applied, in order, and writes no new migration.
Because it never works SQL out from the schema, running it on a server executes only the files that
were reviewed in a pull request. I ran `deploy` against a database file that did not exist: it created
the file, applied all three migrations in order, and left 1 row in `User` (the default owner, from
migration 3) and 3 rows in `_prisma_migrations`. I did not run `deploy` against a database whose
applied migration had been edited, so I cannot say what it does then. In my own words: `dev` turns my
schema edits into reviewed SQL, `deploy` replays that reviewed SQL somewhere else, and I must never run
`dev` against a database I cannot lose.

### What the generated client is

`npx prisma@7 generate` reads `schema.prisma` and writes a folder of TypeScript, here
`src/generated/prisma/`: `client.ts`, `enums.ts`, `models/Task.ts` and others. It is project code,
not a library. Everything that makes `prisma.task.create()` know the shape of a `Task` (the field
names, the enum values, the `Prisma.TaskWhereInput` type) is in those files, and they are written from
the schema, so the file is out of date the moment the schema changes. Because it is generated, it is
git-ignored, which means a fresh clone does not have it. `migrate dev` did **not** generate it for me:
after my first migration `src/generated/prisma/` did not exist, and `generate` took 19 ms. So the rule
is: generate after `npm install`, after every schema change, and in CI.

### The concept-only topics (read, not built)

These were not hands-on, so they come from the Prisma 7 documentation and not from my own runs.

- **ORM trade-offs.** An ORM maps tables to objects, gives typed queries and keeps schema and code in
  step. In return it hides the SQL, so a slow query is easy to write by accident (which is what
  `LOG_QUERIES=true` is for), and complex reporting queries are awkward. Dropping to raw SQL
  (`$queryRaw`, used in my seed for `COUNT(*)`) is still right for those. I used it with a tagged
  template, which sends the value as a parameter, as in M06.
- **PostgreSQL.** Change `provider` to `"postgresql"`, use `@prisma/adapter-pg` with
  `new PrismaPg({ connectionString })`, and the client is built the same way as `db.ts`. The docs say
  driver adapters use the `pg` package's defaults, so connection and idle timeouts differ from what
  Prisma ORM 6 used, and that hosts such as Neon and Supabase give a pooled URL for the running app and
  a direct URL for the CLI. Postgres also has richer types (for example `jsonb`) than SQLite.
- **MongoDB.** A document database stores nested documents and has no joins or foreign keys. Prisma's
  schema models embedded documents as composite types and uses `ObjectId` ids, and it has no Migrate:
  the schema is applied with `db push`. The Prisma 7 documentation I read was inconsistent on whether
  Prisma 7 supports MongoDB at all (one page says support is "coming soon"), so check before relying
  on it.
- **Prisma 8.** It was a release candidate when I checked (`npm view prisma dist-tags` showed `latest`
  as `8.0.0-rc.24`, and `npx prisma@7 migrate dev` printed "Update available 7.10.1 -> 8.0.0-rc.24").
  The Prisma documentation's "coming from Prisma ORM 7" page lists: `schema.prisma` becomes a contract
  file, the `generator` block goes, `generate` becomes `emit`, `migrate dev` and `migrate deploy` are
  replaced, and queries use chained calls (`.where(...).all()`). That page uses PostgreSQL examples
  throughout and does not say what SQLite support looks like, so I cannot confirm it, which is why this
  program stays on 7. The program's solution design says Prisma 7 stays supported for 18 months after
  Prisma 8 is generally available (its pre-cohort checklist, item 2).

## What tripped me up

- **Two copies of `better-sqlite3`.** `npm install better-sqlite3` took the newest, `^13.0.3`, but
  `@prisma/adapter-better-sqlite3@7.10.1` depends on `better-sqlite3@^12.6.0`. `npm ls` showed 13.0.3 at
  the top and 12.11.1 nested under the adapter and under `prisma`. The app opens the database through
  the adapter, so the 13 at the top was not the copy that mattered. I ran `npm install better-sqlite3@12`
  and `npm ls` then showed one copy, 12.11.1. M06 already used 12.11.1.
- **`npm install` warned about install scripts** (`prisma`, `@prisma/engines`, `better-sqlite3`,
  `esbuild`, `fsevents`, `protobufjs`: "not yet covered by allowScripts"). I did not approve anything.
  `better-sqlite3` loaded (`select sqlite_version()` printed `3.53.4` on the 13.0.3 copy I had at that
  point, and `3.53.2` on 12.11.1 later), and `prisma --version` ran, so the warning did not break this
  install.
- **`prisma init` made more than the brief says.** Besides `prisma/schema.prisma` and `prisma7.config.ts`
  it appended `DATABASE_URL` to my `.env`, and it created `.agents/`, `.claude/`, `.windsurf/`,
  `skills-lock.json` and a `task-api/.gitignore`. The first folders hold documentation for AI coding
  assistants. None of that is part of the stage, and `.claude/` must not be committed (AGENTS.md). My
  attempt to remove them from the shell was refused, so I deleted them by hand (see "Status of the remaining items"
  below). The extra `.gitignore` also changed my check: `git check-ignore -v` on the
  generated client matched `task-api/.gitignore`, so it no longer proves the repo's root rule
  `**/generated/prisma` works.
- **`migrate dev` did not generate the client.** After `migrate dev --name init`,
  `src/generated/prisma/` was not there. Anything importing it would have failed until I ran
  `npx prisma@7 generate`.
- **An enum on SQLite is not a `CHECK`.** The generated SQL for `status` was
  `"status" TEXT NOT NULL DEFAULT 'todo'`. M06's `CHECK (status IN (...))` is not in it.
- **On SQLite, adding a column is a table rebuild.** Even migration 2, with only a defaulted column and
  a nullable one, made `migrate dev` write `new_Task`, copy, `DROP TABLE`, rename. It is correct, and it is also why the
  hand-edited migration had to change the copy statement.
- **`migrate reset` refused to run for me.** Prisma detected it was started by an AI coding agent and
  stopped with a message that says the agent must get the user's explicit consent and then pass it in
  `PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION`. I asked, got consent for that one file (`dev.db`), and
  reran it.
- **`migrate reset` does not run the seed.** In 7.10.1 the reset printed `Database reset successful`,
  and the tables were empty: 1 user (the migration's owner), 0 tasks, 0 tags. I had to run
  `npm run db:seed` myself, which is why the checkpoint says "plus the seed script".
- **The first seed run flooded the terminal.** Logging every query while `NODE_ENV=development` printed
  a long run of `prisma:query` lines, one per statement. I made it opt-in with `LOG_QUERIES=true`.
- **`orderBy dueDate asc` puts tasks with no due date first.** SQLite sorts `NULL` before every value
  in ascending order. `SELECT title, dueDate FROM Task ORDER BY dueDate ASC LIMIT 3` returned "Tidy the
  README", "Archive the old branches" and "Compare SQLite and Postgres", all with an empty `dueDate`.
  In my tour, `findFirst` "oldest due" returned a task that had no due date at all. I did not change
  the endpoint, so `?sortBy=dueDate&order=asc` lists undated tasks first.
- **`dueDate: "2026-02-30"` was accepted and stored as 2 March.** My validator checked the text against
  an ISO 8601 pattern and then `new Date(value)`, and `Number.isNaN(date.getTime())` was false because
  JavaScript does not reject an impossible day, it rolls it over. In Node 24, `new Date('2026-02-30')`,
  `'2026-04-31'`, `'2026-02-29'` and `'2026-02-30T10:00:00Z'` all became the next month. A `curl` returned
  201 with `"dueDate":"2026-03-02T00:00:00.000Z"`. I only found it while validating the stage, by sending
  the dates I would not normally think of. The fix, `isRealDay`, builds the date again from the year,
  month and day in the text and compares them. After it, `2026-02-30`, `2026-04-31`, `2026-02-29` and
  `2026-02-30T10:00:00Z` return 400, and `2028-02-29`, `2026-12-31` and `2026-06-15T10:30:00+08:00` return
  201. I added `Errors/24` to Bruno so it stays fixed. `2026-12-31T24:00:00Z` still rolls over to
  1 January 2027, which is valid ISO 8601, so I left it.
- **`?q=%` and `?q=_` return every task.** `contains` becomes `LIKE '%q%'`, and `%` and `_` are LIKE
  wildcards that Prisma does not escape, so `?q=%25` (a literal `%`) matched all 24 tasks. It is not an
  injection, since the value is still a parameter, but searching for a literal `%` or `_` cannot work.
  I did not fix it. See "Status of the remaining items".
- **Prisma's own `error` log printed a code frame for every normal 404 and 409.** My first `db.ts` set
  `log: ['warn', 'error']`. In the `check:errors` output each of the three cases printed a
  `prisma:error` block (file path, the failing call, the Prisma message) twice, and a review reproduced it
  for a plain `PATCH` on an unknown id, which correctly returned 404. The central handler already logs
  genuine 500s with the request id, so I removed `'error'` from the log levels: `log` is `['warn']`, or
  `['query', 'warn']` with `LOG_QUERIES=true`, and `check:errors` now prints only its three `PASS` lines.
- **P2002 and P2003 cannot happen through the v1 routes.** Nothing a client can send collides with a
  unique column or sets a foreign key. So I could not prove them with Bruno alone. I wrote
  `scripts/check-prisma-errors.ts`, which runs a real failing query for each code, sends the error
  through the real error handler on a throwaway Express app, and prints the status.
- **Prisma Studio rejected my SQLite URL, until I wrote it with `://`.** `npx prisma@7 studio --port
  5599 --browser none` printed `Prisma Studio is not supported for the "file:./tour.db" protocol.`
  I got the same message with `--url file:./tour.db` and with the `npx -p better-sqlite3 -p prisma@7
  prisma studio --url ...` form from the documentation, and I wrongly removed my `db:studio` script. I
  then read how the CLI parses the URL (`node_modules/prisma/build/cli.js`): it takes everything before
  `://` as the protocol. `file:./tour.db` has no `://`, so the whole string was looked up as a protocol
  name, and `file` (which Studio does support, using Node's built-in `node:sqlite`) never matched.
  What I tried, each with Studio started in the background and probed with `curl`:
  `--url "file://$PWD/dev.db"` returned HTTP 200. `--url file://./dev.db` returned HTTP 200 too, and
  found the existing file without creating a new one. `--url file://dev.db` and `--url file:///dev.db`
  failed: they resolve to `/dev.db` at the root of the disk, and Studio printed `Database file at
  "/dev.db" was not found. A new file was created.` (no `/dev.db` was left behind). So the script is
  `"db:studio": "prisma studio --url file://./dev.db"`, run from `task-api/`.

## Checkpoint evidence

The checks below come from real runs on 21 October 2026, against the stack above.

**`migrate reset` plus the seed script rebuilds the database from scratch.**

```text
$ npm run db:reset        # prisma migrate reset --force
Applying migration `20261021082413_init`
Applying migration `20261021082429_add_priority_due_date`
Applying migration `20261021082448_add_task_owner`

Database reset successful

$ sqlite3 dev.db "SELECT count(*) FROM User; SELECT count(*) FROM Task; SELECT count(*) FROM Tag;"
1 / 0 / 0           (users / tasks / tags: the reset does not seed)

$ npm run db:seed
Seeded: users 6, tasks 24, tags 7, task_tags 37

$ npm run db:seed         # a second time, with no reset
Seeded: users 6, tasks 24, tags 7, task_tags 37
```

The counts are the M06 data (5 users, 24 tasks, 7 tags, 37 links) plus the default owner. That was the
first cycle: one reset with the consent described above, then the seed twice to show it creates no
duplicates. After the code review I ran the whole cycle a second time from the same `dev.db`, again with
the owner's consent for that one run: `migrate reset` printed `Database reset successful`, the tables
held `1 / 0 / 0` again, `npm run db:seed` printed `Seeded: users 6, tasks 24, tags 7, task_tags 37`, a
further seed printed the same line, `PRAGMA foreign_key_check` printed nothing, `_prisma_migrations` had
3 rows and `migrate status` said `Database schema is up to date!`. So the rebuild gave identical counts
on two separate resets.

**The Bruno collection from M05 is still green against v2.** With the server running against that
freshly reset and seeded database, `npm run bruno` printed:

```text
📊 Execution Summary
│ Status        │     ✓ PASS     │
│ Requests      │ 36 (36 Passed) │
│ Tests         │     49/49      │
│ Assertions    │     76/76      │
```

v1 had 29 requests (7 in `Tasks`, 22 in `Errors`). The 36 are those 29, with `Tasks` 02, 03 and 04
edited for the v2 shape, plus 5 in `Listing/` and `Errors/23` and `24`. I first ran the README flow in a
fresh copy of the project (tracked and new files copied into an empty folder: no `node_modules`, no
`dev.db`, no generated client): `npm install`, `npm run db:generate`, `npm run db:deploy`,
`npm run db:seed`, then `npm start` and `npm run bruno`. That run gave 36 requests, 48 tests and 76
assertions. In that copy `npm run typecheck` failed with `Cannot find module
'./generated/prisma/client.ts'` until I ran `db:generate`, then exited 0. The code review later made me
add one assertion to `Listing/03`, which is why the numbers above say 49 tests: they come from the
`git clone` of the committed branch described under "The pull request" below, and from a run on my own
`dev.db`.

**`P2002`, `P2025` and `P2003` each map through the existing handler.** `npm run check:errors` exited 0
and printed:

```text
P2002 duplicate email: Prisma code P2002 -> HTTP 409 CONFLICT "A record with the same unique value already exists" PASS
P2025 update a task that does not exist: Prisma code P2025 -> HTTP 404 NOT_FOUND "Resource not found" PASS
P2003 task for an owner that does not exist: Prisma code P2003 -> HTTP 409 CONFLICT "The request refers to, or depends on, a record that conflicts with it" PASS
```

P2025 is also exercised by Bruno `Errors` 09, 11, 14 (get, update and delete an unknown id), which
expect 404 `NOT_FOUND`, and they passed in the run above.

**`migrate dev` against `migrate deploy`, in my own words.** `migrate dev` is the command I run while
developing: it turns a change to `schema.prisma` into a new migration, applies it, and offered to
wipe the database when I edited a migration that was already applied. `migrate deploy` is the command
CI and production run: it applies only the migrations that are already committed, in order, and
writes none. Evidence: `deploy` against a new file built the whole database from the three committed
migrations, and I never saw it ask for a reset. I confirm this explanation as my own.

**`package.json` locks Prisma to `^7`.**

```text
$ grep -n '"prisma"\|"@prisma/client"' package.json
25:    "@prisma/client": "^7.10.1",
37:    "prisma": "^7.10.1",
```

Every one-off command I ran used `npx prisma@7`. The `db:*` scripts in `package.json` call `prisma`,
the locally installed 7.10.1, not `npx`, so they never fetch another version. At the time I checked,
the unversioned `prisma` was `8.0.0-rc.24`.

**Other checks I ran.** `npm run typecheck` exits 0. SIGTERM printed `SIGTERM received, closing the
server` then `Server closed`. A list with `?page=99` returned `"data":[]` with `total` 24 and
`totalPages` 2, and `?q=fix&pageSize=3` returned 3 tasks with `total` 4.

**Prisma Studio shows the data.** With the seeded `dev.db`, `npm run db:studio -- --port 5555 --browser
none` printed `Prisma Studio is running at: http://localhost:5555` and `curl` got HTTP 200. I opened it in
a browser. The sidebar listed the tables `_prisma_migrations`, `_TagToTask`, `Tag`, `Task` and `User`, and
the `Task` table showed 25 rows per page of the seeded tasks: ids `00000000-0000-4000-8000-0000000000NN`,
the M06 titles and descriptions, `priority`, `status`, `dueDate`, `createdAt`, and `userId` (2 to 5 for
the seeded tasks). I stopped it again afterwards. I did not edit any rows in Studio.

**M07 merged to `main` by pull request.**
[#7, `feat(m07): add Prisma schema, migrations and service for Task API v2`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/7),
from `feat/module-07/prisma-orm-migrations` into `main`. `gh pr view 7` reported the state `MERGED` on
2026-10-21 (09:53 UTC), with seven commits (the last one was added after I was asked why the URL prefix is
still `/api/v1`). It was merged with a merge commit, so the commits are visible in the history, and the merge
is Task API v2:

```text
$ git log --oneline --graph origin/main -9
*   3604be1 Merge pull request #7 from kiannaalexandra-gragg/feat/module-07/prisma-orm-migrations
|\
| * 9455a0a docs(m07): explain why the URL prefix stays /api/v1
| * 7021e02 docs(m07): describe Task API v2
| * 41613b6 test(m07): update the Bruno collection for v2
| * 1a55a76 feat(m07): add the seed script
| * fe538a5 feat(m07): replace the in-memory store with a Prisma task service
| * b72935a feat(m07): add the Prisma schema and three migrations, one hand-edited
| * 15e12e4 chore(m07): add Prisma 7, the SQLite adapter and the Prisma config
|/
*   7f88f42 Merge pull request #6 from kiannaalexandra-gragg/feat/module-06/sql-fundamentals-sqlite
```

Before I pushed, I made a `git clone` of the committed branch from my local repo into an empty folder and
followed the README: install, `db:generate`,
`db:deploy`, `db:seed`, then `npm run typecheck` (exit 0), `npm run bruno` (36 of 36 requests, 49/49 tests,
76/76 assertions) and `npm run check:errors` (exit 0, three `PASS` lines).

**Status of the remaining items (the open ones are not results):**

- Closed: the merge. PR #7 is merged (see "M07 merged to `main` by pull request" above). I did not re-run
  the checks on `main` after the merge. The ones above were run on the branch and on a local clone of the committed
  branch before I pushed.
- Not recorded: the trainer's own checkpoint check. It is the trainer's act, not part of my evidence, and
  no result has been given to me.
- Known and not fixed: `?q=` treats `%` and `_` as wildcards (see "What tripped me up"), and
  `?sortBy=dueDate&order=asc` lists undated tasks first.
- Closed: Prisma Studio. See the Studio check under "Other checks I ran".
- Closed: I deleted the `prisma init` leftovers (`.agents/`, `.claude/`, `.windsurf/`,
  `skills-lock.json`, `.gitignore`) by hand. From the repo root,
  `git check-ignore -v task-api/src/generated/prisma/client.ts` now prints
  `.gitignore:12:**/generated/prisma`, so the root rule works on its own. `task-api/dev.db` matches
  `*.db` and `task-api/.env` matches `.env`, and `git status` lists only files that belong to the stage.

## What I'd do differently

- **Install `better-sqlite3@12` on the first command.** Taking the newest cost me two installs and a
  `npm ls`, because the adapter pins a major version of its own. Next time I check the adapter's
  dependency range first.
- **Run `prisma init` in an empty folder first**, to see what it writes before it writes into a
  project that already has a `.env` and a `.gitignore`.
- **Write the check script before the endpoints.** I only noticed that P2002 and P2003 were unreachable
  from the API while mapping them, and the script is what finally proved them.
- **Decide how undated tasks sort** before shipping `sortBy=dueDate`, instead of finding the `NULL`
  ordering during the tour.
