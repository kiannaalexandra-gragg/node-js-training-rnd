# M07 — Prisma ORM & Migrations: Training Material

**Week 3 · Days 2-4 · 8 hours**

A "module" in this program means a training stage. When this guide says "ES module" it means a JavaScript
module file. The guide's own steps are called "stages" too, so "the stage" means this module and "Stage 4"
means a step of the guided path below.

## Builds on

M05 (the Express Task API v1, which you upgrade in place) and M06 (the SQL schema and ERD, which you
translate into Prisma). Keep both on hand: your M06 `schema.sql` and `seed.sql` are the input to Stages 4 and 11,
and your M05 error handler, store and Bruno collection are the code you change.

## 1. What you'll build and why

You will replace the Task API's in-memory store with a real database, reached through Prisma ORM 7. You
finish with **Task API v2**:

- the **M06 schema translated into a Prisma schema**,
- **three migrations**: the first creates the tables, the second adds columns, and the third is
  **hand-edited** because Prisma cannot generate it,
- a **service layer** on the typed Prisma client that replaces the in-memory store,
- a list endpoint with **`status` and `q` filters, sorting, and `page`/`pageSize` pagination**, returning
  `{ data, meta }`,
- a **seed script** (`npm run db:seed`),
- **Prisma's error codes mapped** (`P2002`, `P2025`, `P2003`) **into the central error handler you wrote in
  M05**, not a new one,
- a Bruno collection from M05 that is **still green** against v2.

Why this stage matters: in M06 you wrote SQL by hand. An ORM (object-relational mapper) writes it for you, so
a change to your data model becomes a reviewed, replayable migration instead of a hand-run `ALTER TABLE`, and
every query is checked by TypeScript against the model. Migrations are the part that goes wrong in real
teams, which is why two of the three here are about what Prisma cannot do for you.

PostgreSQL, MongoDB and Prisma 8 come up as **concepts only**. SQLite is the only database you touch.

### Where the code goes

All code lives in your build repo, in the `task-api/` folder you made in M05. The Task API is one project
that grows from M05 to M09, so v2 is a change to that folder, not a new one:

```text
task-api/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/            one folder per migration, committed
│   └── seed.ts
├── prisma7.config.ts
├── src/
│   ├── db.ts                  the one PrismaClient
│   ├── services/task-service.ts
│   └── generated/prisma/      the generated client (git-ignored)
└── scripts/check-prisma-errors.ts
```

### How to use this guide

- Work the stages in order. Each ends in a **Done when** checklist you can verify yourself.
- A line starting with `$` is typed in a terminal. Everything else is file content or output.
- Commands and output are from a reference build on macOS with Node 24.22.0 and Prisma 7.10.1. Your
  migration folder names carry your own timestamps, shown here as `<timestamp>`.
- The reference M05 code is shown where you change it. If yours differs (file names, how the store is
  called), make the same change in your shape.
- Imports use `.ts` extensions, as in M04 to M06. The program has not decided this convention (open decision
  P2), so it is this project's choice, and it shapes one generator option in Stage 3.

## 2. Before you start

You need:

- **Node.js 24 LTS.** Run `nvm use` on its own line in your repo root, then check `node --version`.
  Do not pipe `nvm use`: a pipe leaves the shell on whatever version it started with.
- **Your M05 Task API**, merged, with a working Bruno collection, and your **M06 schema and seed**.
- **The `sqlite3` command line tool** (you used it in M06), for looking inside the database.
- **A C/C++ toolchain only if `better-sqlite3` has no prebuilt binary for your machine.** On macOS that is
  the Xcode command line tools. Most machines never need it.
- **A branch habit**: a branch named `feat/module-07/<topic>`, Conventional Commits, and a pull request. You
  do not push to `main`.

Pinned versions that matter in this stage:

| Tool | Version |
| --- | --- |
| Node.js | 24 LTS |
| TypeScript | 7.0 |
| Express | 5.x |
| Prisma ORM | **7.x, always `@7`**: `prisma@7`, `@prisma/client@7`, `npx prisma@7` |
| Database | SQLite, through `better-sqlite3` and the Prisma driver adapter |

**Never run an unversioned `npx prisma` or `npm install prisma`.** The unversioned name now points at
Prisma 8, which was still a release candidate when this guide was written (`npm view prisma dist-tags`
showed `latest` as `8.0.0-rc.24`). Its command line tool does not read `schema.prisma` the way this guide
needs. Every one-off command below is `npx prisma@7 ...`, and `package.json` locks Prisma to `^7`. This is a
fixed rule of the program, not something to ask your trainer about. When you read Prisma's documentation,
use the pages under `/v7/`.

Free tools only. No Docker. No paid licences.

## 3. Guided path

### Stage 1 — Branch, baseline, and the plan

**Goal:** start from an up-to-date `main`, and prove the Bruno collection is green **before** you change
anything, so a red run later is yours.

1. Make sure M05 and M06 are merged, then:

   ```bash
   git switch main
   git pull
   git switch -c feat/module-07/prisma-orm-migrations
   cd task-api
   nvm use
   node --version
   ```

   You branch from `main`, not from an old stage branch.

2. Make sure `.env` exists (copy `.env.example` if not), start the v1 server in a second terminal
   (`npm start`), and run the collection:

   ```bash
   npm run bruno
   ```

   Stop the server when it finishes. Write down the **number of requests** and the **number of tests and
   assertions** it printed: the reference M05 collection had 29 requests. Your count is your baseline, and
   the brief's rule is that every one of those still passes at the end, apart from the three requests
   whose assertions describe the v1 response shape (Stage 13 says which).

3. Skim your M06 `schema.sql` and keep its ERD open. Stage 4 translates it.

**Done when:**

- [ ] `git branch --show-current` prints `feat/module-07/prisma-orm-migrations`.
- [ ] `node --version` prints a 24.x version.
- [ ] `npm run bruno` against the unchanged v1 server prints `PASS`, and you have written down the counts.

### Stage 2 — Install Prisma 7 and the SQLite driver adapter, and lock the version

**Goal:** install the packages and make sure nothing can drift to Prisma 8.

Prisma 7 no longer has a database driver built in. The client needs a **driver adapter**: a small package
that opens the database through a driver you choose. For SQLite that is `@prisma/adapter-better-sqlite3`
plus `better-sqlite3`, the same driver you used in the M06 injection demo.

1. Install, from `task-api/`:

   ```bash
   npm install --save-dev prisma@7 @types/better-sqlite3
   npm install @prisma/client@7 @prisma/adapter-better-sqlite3@7 better-sqlite3@12
   ```

   `better-sqlite3@12` is deliberate. The adapter declares `better-sqlite3@^12.6.0`. If you install
   `better-sqlite3` with no version, npm takes the newest major (13 when this was written), and then there
   are **two copies**: yours at the top and the adapter's own 12.x underneath. The app opens the database
   through the adapter, so the top-level copy is the wrong one.

2. Check there is one copy, and what npm wrote:

   ```bash
   npm ls better-sqlite3
   grep -n '"prisma"\|"@prisma/client"' package.json
   ```

   Every line of `npm ls` should end in the same version (`better-sqlite3@12.11.1` in the reference build,
   `deduped` under the adapter and under `prisma`). `package.json` must show `"prisma": "^7..."` and
   `"@prisma/client": "^7..."`.

3. Read the warnings. `npm install` may print `install-scripts ... not yet covered by allowScripts` for
   `prisma`, `@prisma/engines`, `better-sqlite3`, `esbuild` and others. You do not have to approve anything
   for this stage. Check that the native driver loads:

   ```bash
   node -e "const D=require('better-sqlite3');console.log(new D(':memory:').prepare('select sqlite_version() v').get())"
   npx prisma@7 --version
   ```

   The first prints an object with a `v` such as `3.53.2` (the SQLite bundled in `better-sqlite3@12.11.1`). If it throws, fix the native install before going
   on (see Common pitfalls). The second prints `prisma : 7.x.x`.

4. Add the scripts you will use all stage. In `package.json`, inside `"scripts"`:

   ```json
   "db:generate": "prisma generate",
   "db:migrate": "prisma migrate dev",
   "db:deploy": "prisma migrate deploy",
   "db:seed": "prisma db seed",
   "db:reset": "prisma migrate reset --force",
   "db:studio": "prisma studio --url file://./dev.db",
   "check:errors": "tsx --env-file-if-exists=.env scripts/check-prisma-errors.ts"
   ```

   Scripts run the **locally installed** `prisma` (7.x, from `node_modules`), so they cannot fetch another
   version. `check:errors` is used in Stage 10.

**Done when:**

- [ ] `npm ls better-sqlite3` shows a single version, and `npx prisma@7 --version` prints `7.x.x`.
- [ ] `package.json` locks `prisma` and `@prisma/client` to `^7`.
- [ ] The `better-sqlite3` check prints a SQLite version without an error.

### Stage 3 — `prisma init`, the config file, and the generated client's ignore rule

**Goal:** create the Prisma files, point them at a SQLite file, and keep the generated client out of git.

1. Run init:

   ```bash
   npx prisma@7 init --datasource-provider sqlite
   ```

   It creates `prisma/schema.prisma` and `prisma7.config.ts`, and appends `DATABASE_URL="file:./dev.db"` to
   your `.env`. **Look at `git status` afterwards.** In the reference build it also created
   `.agents/`, `.claude/`, `.windsurf/`, `skills-lock.json` and a `.gitignore` inside `task-api/`. Those are
   documentation folders for AI coding assistants and an extra ignore file. They are not part of this stage,
   and `.claude/` must never be committed. Delete the ones `git status` lists as new and untracked
   (`rm -rf .agents .claude .windsurf` and `rm -f skills-lock.json .gitignore` in the reference build). Do not
   delete a `.gitignore` or `.claude/` that was already in your project before you ran init. The extra
   `.gitignore` matters most: it can make your ignore check in step 4 pass for the wrong reason.

2. **The config file name depends on your Prisma version.** On 7.10 and later it is `prisma7.config.ts`.
   Before 7.10 it was `prisma.config.ts`. Init writes the right name for the version you installed, and the
   Prisma 7 command line tool reads either. Replace its content. Prisma 7 does **not** load `.env` by itself, and
   init's version imports `dotenv`, which you have not installed. Node 24 can load the file:

   ```ts
   // prisma7.config.ts
   import { existsSync } from 'node:fs';
   import { defineConfig, env } from 'prisma/config';

   // Prisma 7 does not load .env by itself. Node 24 can, so no dotenv package is needed. The guard keeps
   // a missing .env (CI sets real environment variables instead) from crashing the CLI.
   if (existsSync('.env')) process.loadEnvFile('.env');

   export default defineConfig({
     schema: 'prisma/schema.prisma',
     migrations: {
       path: 'prisma/migrations',
       // Run by `npx prisma@7 db seed`, which the db:seed script wraps.
       seed: 'tsx prisma/seed.ts',
     },
     datasource: {
       url: env('DATABASE_URL'),
     },
   });
   ```

   Prisma 7 moved the database URL out of `schema.prisma` and into this file. The config is read by the
   **command line tool only**: your running app never reads it, which is why Stage 5 passes the URL to the
   client separately.

3. Set the variable, in `.env` (never committed) and in `.env.example` (committed, fake value):

   ```bash
   # .env.example
   # The SQLite file, relative to task-api/ (required, the server will not start without it).
   DATABASE_URL=file:./dev.db

   # Set to true to print the SQL Prisma runs for every call (optional, default off).
   LOG_QUERIES=false
   ```

   Check that `.env` has one `DATABASE_URL` line, not two.

4. The generated client goes in `src/generated/prisma/`. **Make sure git will ignore it.** Your repo's root
   `.gitignore` should contain `**/generated/prisma`. The `**/` matters: a pattern with a slash in the middle,
   such as `generated/prisma`, only matches from the repo root and would miss `task-api/src/generated/prisma`.
   You will prove the rule in Stage 5, once the folder exists. `*.db` must be ignored too, for `dev.db`.

**Done when:**

- [ ] `prisma/schema.prisma` and `prisma7.config.ts` exist, and `git status` lists no `.agents/`, `.claude/`,
      `.windsurf/`, `skills-lock.json` or extra `.gitignore`.
- [ ] `.env.example` has `DATABASE_URL=file:./dev.db`, and `.env` has it exactly once.
- [ ] You can say why the config file is called `prisma7.config.ts` on your version, and why it is the
      command line tool and not the app that reads it.

### Stage 4 — Translate the M06 schema into Prisma

**Goal:** write `prisma/schema.prisma` so it describes the same data as your M06 schema, deciding on purpose
where Prisma differs.

First decide, then write. These are the translations, with the reasoning:

| M06 (SQL) | Prisma | Why |
| --- | --- | --- |
| `tasks.id INTEGER PRIMARY KEY` | `id String @id @default(uuid())` | The v1 API's contract, its UUID check and the Bruno error cases all use UUID ids. Changing them would change the API. |
| `users` and `tags` integer ids | `Int @id @default(autoincrement())` | The API never exposes them. |
| `task_tags` junction table | `tags Tag[]` and `tasks Task[]` | An **implicit** many-to-many: Prisma creates and manages the join table. Choose an explicit model only if the link needs its own columns. |
| `CHECK (status IN (...))` | `enum TaskStatus` | On SQLite an enum is a plain `TEXT` column with **no** `CHECK`. Prisma enforces it, the database does not. |
| `created_at TEXT DEFAULT (datetime('now'))` | `createdAt DateTime @default(now())` plus `updatedAt DateTime @updatedAt` | A real date type. `@updatedAt` is set by the client on every update. |
| `ON DELETE CASCADE` | `onDelete: Cascade` on the relation | Same rule: deleting a user deletes their tasks. |
| `priority`, `due_date`, `user_id` in `CREATE TABLE` | **added later, in Stages 6 and 7** | The brief asks for a second migration that adds a column and a third that you hand-edit. M05's tasks had none of them. |

So the **first** version of the schema leaves `priority`, `dueDate` and the owner out. Write
`prisma/schema.prisma`:

```prisma
generator client {
  provider               = "prisma-client"
  output                 = "../src/generated/prisma"
  // The repo imports with .ts specifiers (allowImportingTsExtensions), so the generated files must too.
  generatedFileExtension = "ts"
  importFileExtension    = "ts"
  moduleFormat           = "esm"
}

datasource db {
  provider = "sqlite"
}

enum TaskStatus {
  todo
  in_progress
  done
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())
}

model Task {
  id          String     @id @default(uuid())
  title       String
  description String?
  status      TaskStatus @default(todo)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  tags        Tag[]

  @@index([status])
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  tasks Task[]
}
```

The generator options matter. Prisma 7's `prisma-client` generator writes TypeScript **into your project**.
`importFileExtension = "ts"` makes its imports end in `.ts`, so they match the rest of your code. If your
project uses `.js` specifiers instead, set it to `"js"`.

Then format and check it:

```bash
npx prisma@7 format
npx prisma@7 validate
```

`format` aligns the file. `validate` should print `The schema at prisma/schema.prisma is valid`.

**Done when:**

- [ ] `npx prisma@7 validate` prints that the schema is valid.
- [ ] The schema has a `User` 1:N `Task` path planned for Stage 7, an implicit M:N between `Task` and `Tag`,
      an `@id`, a `@default`, a `@unique`, an `@@index` and an `@updatedAt`.
- [ ] You can explain why `Task.id` is a UUID string while `User.id` is an integer.

### Stage 5 — Migration 1: `init`, then generate the client

**Goal:** create the first migration, read the SQL, and build the typed client.

1. Create and apply it:

   ```bash
   npx prisma@7 migrate dev --name init
   ```

   This creates `dev.db`, writes `prisma/migrations/<timestamp>_init/migration.sql`, and applies it. It also
   writes `prisma/migrations/migration_lock.toml`, which pins the provider (`sqlite`): commit it.

2. **Read the file.** A migration is plain SQL, and reading it is how you learn what Prisma decided for you:

   ```bash
   cat prisma/migrations/*_init/migration.sql
   ```

   You should see `CREATE TABLE "User"`, `"Task"`, `"Tag"`, and `"_TagToTask"` (the implicit join table, with
   columns `A` and `B`), unique indexes for `User.email` and `Tag.name`, and an index on `Task.status`.
   Notice two things: `"status" TEXT NOT NULL DEFAULT 'todo'` has no `CHECK`, and `createdAt` is
   `DATETIME ... DEFAULT CURRENT_TIMESTAMP`.

3. **`migrate dev` did not build the client.** Look: `ls src/generated/prisma` fails. Generate it, now and
   after every schema change:

   ```bash
   npm run db:generate
   ls src/generated/prisma
   ```

   You should see `client.ts`, `enums.ts`, `models/` and more. These files are project code, written from your
   schema.

4. Prove the ignore rule, from the **repo root**, on a real path:

   ```bash
   git check-ignore -v task-api/src/generated/prisma/client.ts
   git check-ignore -v task-api/dev.db
   ```

   Each prints the line of a `.gitignore` that matched: `**/generated/prisma` and `*.db`, both from the
   **root** `.gitignore`. If the first one names `task-api/.gitignore`, you still have init's extra file.

5. Make the one client the whole app will share, `src/db.ts`:

   ```ts
   import { PrismaBetterSqlite3 } from '@prisma/adapter-better-sqlite3';
   import { PrismaClient } from './generated/prisma/client.ts';

   // Prisma 7 has no built-in database driver: PrismaClient must be given an adapter. The adapter opens
   // the SQLite file itself, so it needs the URL here as well as in prisma7.config.ts (the config is
   // only read by the CLI, never by the running app).
   //
   // One client for the whole process. Every PrismaClient owns a connection, so a new one per request
   // would open a new connection each time. A module is evaluated once, so exporting the instance from
   // here makes it a singleton without any extra code.
   const url = process.env.DATABASE_URL;
   if (url === undefined || url.trim() === '') {
     throw new Error('DATABASE_URL is required, for example file:./dev.db');
   }

   export const prisma = new PrismaClient({
     adapter: new PrismaBetterSqlite3({ url }),
     // Set LOG_QUERIES=true to print the SQL behind every call: it is how you see what Prisma really
     // runs. Off by default, because a seed of 24 tasks would print a line for every statement.
     // 'error' is left out on purpose: Prisma would print a code frame for every P2025, P2002 and P2003,
     // and those are normal 404s and 409s. A real failure reaches the error handler, which logs it.
     log: process.env.LOG_QUERIES === 'true' ? ['query', 'warn'] : ['warn'],
   });
   ```

6. Also extend `tsconfig.json`, so the type check covers the files outside `src/` that you add later:

   ```json
   "include": ["src", "prisma/seed.ts", "prisma7.config.ts", "scripts"]
   ```

   (A file named in `include` that does not exist yet is fine; the glob simply matches nothing until you
   create it.)

7. Check it compiles:

   ```bash
   npm run typecheck
   ```

**Done when:**

- [ ] `prisma/migrations/<timestamp>_init/migration.sql` exists, with `migration_lock.toml` beside the
      migration folders.
- [ ] `ls src/generated/prisma` lists `client.ts`, and both `git check-ignore -v` commands name the root
      `.gitignore`.
- [ ] `npm run typecheck` exits 0 with the generated client in place.
- [ ] You can say why `src/generated/prisma/` is git-ignored although it is project code, and when it must be
      regenerated.

### Stage 6 — Migration 2: add a column

**Goal:** change the schema, let Prisma generate the migration, and see what it does to a table that exists.

1. In `model Task`, add two fields after `status`:

   ```prisma
     // 1 (high) to 3 (low). The default means existing rows get 2 when this column is added.
     priority    Int        @default(2)
     dueDate     DateTime?
   ```

   `priority` is required **with a default**, so existing rows have a value to take. `dueDate` is optional, so
   existing rows get `NULL`. Either one is safe to add to a table that has data. A required column with **no**
   default is not, which is Stage 7.

2. Create and apply the migration, then read it:

   ```bash
   npx prisma@7 format
   npx prisma@7 migrate dev --name add_priority_due_date
   cat prisma/migrations/*_add_priority_due_date/migration.sql
   ```

   You will not see `ALTER TABLE ... ADD COLUMN`. You will see `-- RedefineTables`: Prisma creates
   `new_Task` with the new shape, copies the rows over with `INSERT INTO "new_Task" ... SELECT ... FROM
   "Task"`, drops `Task`, renames `new_Task`, and re-creates the indexes. That is its general recipe for
   changing a SQLite table. Look at the copy: it lists the old columns only, so `priority` takes its default
   and `dueDate` takes `NULL`.

3. Run the generator, so the client knows the new fields:

   ```bash
   npm run db:generate
   npm run typecheck
   ```

**Done when:**

- [ ] A second folder, `<timestamp>_add_priority_due_date`, exists, and `npx prisma@7 migrate status` prints
      `Database schema is up to date!`.
- [ ] `sqlite3 dev.db ".schema Task"` shows `priority INTEGER NOT NULL DEFAULT 2` and a nullable `dueDate`.
- [ ] You can explain why this column was safe to add with no hand-editing.

### Stage 7 — Migration 3: add a required column to a table that has data (hand-edited)

**Goal:** add `userId` (the owner) as a **required** column, see Prisma refuse, and write the migration yourself.

**Why this one is different.** A required column is `NOT NULL`, so every existing row needs a value the moment
the column exists. Your options are a default, making the column optional now, or **backfilling the value by
hand**. A default only works when one value is right for every old row. Here no value is right until *you*
decide who owns the old tasks. Prisma sees the schema change but cannot know that, so it cannot write the SQL.

1. **Put real rows in the table**, so you meet the problem the way production would. On an empty table this
   migration would simply work, and teach you nothing:

   ```bash
   sqlite3 dev.db "INSERT INTO Task (id,title,description,status,updatedAt) VALUES
     ('aaaaaaaa-0000-4000-8000-000000000001','Write the schema',NULL,'done',CURRENT_TIMESTAMP),
     ('aaaaaaaa-0000-4000-8000-000000000002','Add the owner column','Needs a backfill','in_progress',CURRENT_TIMESTAMP),
     ('aaaaaaaa-0000-4000-8000-000000000003','Seed the database',NULL,'todo',CURRENT_TIMESTAMP);"
   ```

2. **Change the schema.** Every task has exactly one owner. Add the inverse side to `User`, the foreign key and
   relation to `Task`, and an index:

   ```prisma
   model User {
     // ... existing fields
     tasks     Task[]
   }

   model Task {
     // ... existing fields, after updatedAt
     // Every task has exactly one owner. Deleting the owner deletes their tasks, as in the M06 schema.
     userId      Int
     owner       User       @relation(fields: [userId], references: [id], onDelete: Cascade)
     // ... tags Tag[] stays

     @@index([status])
     @@index([userId])
   }
   ```

3. **Make the migration, but do not apply it.** `--create-only` writes the file and stops, so you can read and
   edit it first:

   ```bash
   npx prisma@7 format
   npx prisma@7 migrate dev --create-only --name add_task_owner
   ```

   Prisma prints:

   ```text
   ⚠️ We found changes that cannot be executed:

     • Step 0 Added the required column `userId` to the `Task` table without a default value. There are 3 rows in this table, it is not possible to execute this step.
   ```

   It still writes the file, with a `Warnings:` comment on top. Open
   `prisma/migrations/<timestamp>_add_task_owner/migration.sql`. The `INSERT INTO "new_Task" ... SELECT`
   copies every column **except** `userId`, and `userId` is `NOT NULL`, so applying it would fail on the first
   row.

4. **Edit it by hand, in two places.**

   *First*, before `-- RedefineTables`, create the owner the rows will point at:

   ```sql
   INSERT INTO "User" ("email", "name", "createdAt")
   VALUES ('owner@example.com', 'Default Owner', CURRENT_TIMESTAMP)
   ON CONFLICT ("email") DO NOTHING;
   ```

   *Second*, in the `INSERT INTO "new_Task"` statement, add `"userId"` to the column list and select the
   owner's id for it. The whole statement becomes:

   ```sql
   INSERT INTO "new_Task" ("createdAt", "description", "dueDate", "id", "priority", "status", "title", "updatedAt", "userId") SELECT "createdAt", "description", "dueDate", "id", "priority", "status", "title", "updatedAt", (SELECT "id" FROM "User" WHERE "email" = 'owner@example.com') FROM "Task";
   ```

   Keep Prisma's `Warnings:` comment, and add a note under it saying what you changed and why, so the next
   reader understands why this file differs from a generated one. The `owner@example.com` user is also the
   owner every task created through the API will get (Stage 8), because v1 has no users.

5. **Apply it and check it:**

   ```bash
   npx prisma@7 migrate dev
   sqlite3 dev.db "SELECT id, title, userId FROM Task; SELECT * FROM User; PRAGMA foreign_key_check;"
   npx prisma@7 migrate dev
   npm run db:generate
   ```

   The first command applies the edited file. All three tasks now show the same `userId`, `User` has the one
   owner, and `PRAGMA foreign_key_check` prints **nothing**, which means no task points at a missing user.
   The second `migrate dev` should say `Already in sync, no schema change or pending migration was found`.

6. **Never edit an applied migration.** Prisma stores a checksum of every applied migration in the database.
   If you change an applied file, `migrate status` can still say "up to date", but the next `migrate dev` says
   the migration "was modified after it was applied" and asks to **reset the database and lose the data**. To
   change the schema again, make a new migration.

**Done when:**

- [ ] Three folders exist in `prisma/migrations/`, and one of them, `..._add_task_owner`, contains your
      hand-written `INSERT INTO "User"` and the `userId` in the copy statement.
- [ ] `npx prisma@7 migrate status` prints `Database schema is up to date!` and
      `sqlite3 dev.db "PRAGMA foreign_key_check;"` prints nothing.
- [ ] You can explain what you had to hand-edit and why `migrate dev` could not generate it.

### Stage 8 — The service layer: replace the in-memory store

**Goal:** move every task query into one module on the typed client, and swap it in for the M05 store.

The M05 store was written as `async` methods on purpose, so the handlers do not change much. Do four things.

1. **Make the types come from the schema.** Replace `src/types/task.ts` so the database owns the shape of a
   task and the two cannot drift:

   ```ts
   import type { Task as TaskRow } from '../generated/prisma/client.ts';
   import { TaskStatus } from '../generated/prisma/enums.ts';

   export type { TaskStatus };

   // One list of allowed values, taken from the schema's enum, used by validation at runtime.
   export const TASK_STATUSES = Object.values(TaskStatus);

   // A task exactly as the database returns it. JSON turns the Date fields into ISO strings on the way out.
   export type Task = TaskRow;

   // What a client sends to create a task. The server owns the id, the owner and the timestamps. status
   // defaults to 'todo' and priority to 2 in the database.
   export type CreateTaskInput = {
     title: string;
     description?: string;
     status?: TaskStatus;
     // 1 (high) to 3 (low).
     priority?: number;
     // null means "no due date".
     dueDate?: Date | null;
   };

   // What a client sends to change a task: any subset of the creatable fields.
   export type UpdateTaskInput = Partial<CreateTaskInput>;

   export function isTaskStatus(value: unknown): value is TaskStatus {
     return typeof value === 'string' && (TASK_STATUSES as readonly string[]).includes(value);
   }

   // The fields a list can be sorted by. A list of names, not any column the client types, so a
   // client cannot sort by something that is not meant to be public.
   export const SORT_FIELDS = ['createdAt', 'title', 'priority', 'dueDate', 'status'] as const;
   export type SortField = (typeof SORT_FIELDS)[number];

   export const SORT_ORDERS = ['asc', 'desc'] as const;
   export type SortOrder = (typeof SORT_ORDERS)[number];

   export const DEFAULT_PAGE_SIZE = 20;
   export const MAX_PAGE_SIZE = 100;

   // GET /api/v1/tasks after validation: every field has a value, either the client's or the default.
   export type ListTasksQuery = {
     status?: TaskStatus;
     // Matches tasks whose title or description contains this text.
     q?: string;
     sortBy: SortField;
     order: SortOrder;
     // 1-based.
     page: number;
     pageSize: number;
   };

   export type PageMeta = {
     page: number;
     pageSize: number;
     // Tasks matching the filters, across all pages.
     total: number;
     totalPages: number;
   };

   export type TaskPage = { data: Task[]; meta: PageMeta };
   ```

   If your M05 types file had other exports, keep them.

2. **Write the service**, `src/services/task-service.ts`. `list` is explained in Stage 9, so read it for the
   shape now and come back:

   ```ts
   import { NotFoundError } from '../errors.ts';
   import { prisma } from '../db.ts';
   import type { Prisma } from '../generated/prisma/client.ts';
   import type {
     CreateTaskInput,
     ListTasksQuery,
     Task,
     TaskPage,
     UpdateTaskInput,
   } from '../types/task.ts';

   // The service holds every database call for tasks, so handlers know nothing about Prisma and a
   // change to the data layer stays in this file. It keeps the shape of the M05 store (list, get,
   // create, update, remove), so the handlers changed very little when the store was swapped out.
   //
   // The API has no users yet, so every task created through it belongs to this one owner. The
   // owner is created by the add_task_owner migration and by the seed script.
   export const DEFAULT_OWNER_EMAIL = 'owner@example.com';

   export const taskService = {
     async list(query: ListTasksQuery): Promise<TaskPage> {
       const { status, q, sortBy, order, page, pageSize } = query;

       // Built piece by piece so only the filters the client sent end up in the query. `contains`
       // becomes SQL LIKE '%q%'. SQLite's LIKE ignores case for ASCII letters, so 'fix' finds 'Fix'.
       const where: Prisma.TaskWhereInput = {};
       if (status !== undefined) where.status = status;
       if (q !== undefined) {
         where.OR = [{ title: { contains: q } }, { description: { contains: q } }];
       }

       // The id is a tie-breaker. Without it, two tasks with the same sort value could swap places
       // between two requests, and a task could appear on two pages or on none.
       const orderBy: Prisma.TaskOrderByWithRelationInput[] = [{ [sortBy]: order }, { id: 'asc' }];

       // One transaction for both reads, so `total` describes the same rows `data` was cut from.
       const [data, total] = await prisma.$transaction([
         prisma.task.findMany({
           where,
           orderBy,
           skip: (page - 1) * pageSize,
           take: pageSize,
         }),
         prisma.task.count({ where }),
       ]);

       return { data, meta: { page, pageSize, total, totalPages: Math.ceil(total / pageSize) } };
     },

     async get(id: string): Promise<Task> {
       const task = await prisma.task.findUnique({ where: { id } });
       if (task === null) throw new NotFoundError(`Task ${id} not found`);
       return task;
     },

     async create(input: CreateTaskInput): Promise<Task> {
       const owner = await prisma.user.findUnique({ where: { email: DEFAULT_OWNER_EMAIL } });
       // Not a client mistake: the database was not migrated or seeded. A plain Error becomes a 500.
       if (owner === null) {
         throw new Error(`Default owner ${DEFAULT_OWNER_EMAIL} is missing: run the migrations`);
       }
       return prisma.task.create({ data: { ...input, userId: owner.id } });
     },

     // update and remove do not look the task up first. Prisma throws a P2025 error when the row does
     // not exist, and the central error handler turns that into the same 404 the handlers used to throw.
     async update(id: string, patch: UpdateTaskInput): Promise<Task> {
       return prisma.task.update({ where: { id }, data: patch });
     },

     async remove(id: string): Promise<void> {
       await prisma.task.delete({ where: { id } });
     },
   };
   ```

   `create` does not copy `req.body` into Prisma. The input already went through your validation, which is
   what the program means by *Zod is the request contract, Prisma types are the persistence contract* (Zod
   itself arrives in M08).

3. **Point the handlers at the service.** In `src/handlers/tasks.handlers.ts`: import `taskService` and the
   `TaskPage` type, delete the `NotFoundError` import (and the `taskStore` import), and change the bodies:

   ```ts
   export async function listTasks(
     req: Request<object, TaskPage, unknown, Record<string, unknown>>,
     res: Response<TaskPage>,
   ): Promise<void> {
     const query = parseListQuery(req.query);
     res.status(200).json(await taskService.list(query));
   }

   export async function createTask(
     req: Request<object, Task, unknown>,
     res: Response<Task>,
   ): Promise<void> {
     const input = parseCreateTask(req.body);
     res.status(201).json(await taskService.create(input));
   }

   export async function getTask(req: Request<IdParams, Task>, res: Response<Task>): Promise<void> {
     const id = parseTaskId(req.params.id);
     res.status(200).json(await taskService.get(id));
   }

   export async function updateTask(
     req: Request<IdParams, Task, unknown>,
     res: Response<Task>,
   ): Promise<void> {
     const id = parseTaskId(req.params.id);
     const patch = parseUpdateTask(req.body);
     // A task that does not exist makes Prisma throw P2025, which the error handler turns into a 404.
     res.status(200).json(await taskService.update(id, patch));
   }

   export async function deleteTask(req: Request<IdParams>, res: Response): Promise<void> {
     const id = parseTaskId(req.params.id);
     await taskService.remove(id);
     res.sendStatus(204);
   }
   ```

   The `if (task === undefined) throw new NotFoundError(...)` lines in `updateTask` and `deleteTask` are gone:
   the missing-row case now arrives as a Prisma error (Stage 10). `getTask` has none either, because the
   service's `get` throws it.

4. **Delete the old store**: `git rm src/store/task-store.ts`. Close the database on shutdown, in
   `src/server.ts`: `import { prisma } from './db.ts';`, and inside `server.close(...)`'s success path call
   `prisma.$disconnect()` before `process.exit(0)`:

   ```ts
       // No request is in flight any more, so the database connection can be closed safely.
       prisma.$disconnect().then(
         () => {
           console.log('Server closed');
           process.exit(0);
         },
         (disconnectErr: unknown) => {
           console.error('Error while closing the database:', disconnectErr);
           process.exit(1);
         },
       );
   ```

   Do **not** run the type check yet: `parseListQuery` still returns the v1 shape, and Stage 9 changes it.

**Done when:**

- [ ] `src/services/task-service.ts` exists, `src/store/task-store.ts` does not, and nothing imports the store.
- [ ] The five handlers call `taskService`, with no `taskStore` left (`grep -rn taskStore src` prints nothing).
- [ ] You can explain why the handlers needed so few changes, and why `update` and `remove` do not look the
      task up first.

### Stage 9 — The list endpoint: filters, sorting, pagination, `{ data, meta }`

**Goal:** validate and parse the query, and return the page.

The service already builds the query (Stage 8). What is missing is the parsing of what the client sent. The
rule from M05 stays: **a bad value is a 400, never ignored**, so a typo cannot silently return a different list.
In `src/validation/task-validation.ts`, **replace your M05 `parseListQuery` with the one below** (a second function
with that name is a duplicate and will not compile), add the two helpers above it, and add the new names to its imports
(`DEFAULT_PAGE_SIZE`, `MAX_PAGE_SIZE`, `SORT_FIELDS`, `SORT_ORDERS`, `ListTasksQuery`, `SortField`,
`SortOrder`, `TaskStatus`):

```ts
// A query-string value that must be one of a fixed list, or absent (then `fallback`). Anything else
// (an unknown value, an empty `?order=`, or the key given twice, which Express turns into an
// array) is a problem, never ignored, so a typo cannot silently return a different list.
function checkChoice<T extends string>(
  raw: unknown,
  path: string,
  choices: readonly T[],
  fallback: T,
  problems: ErrorDetail[],
): T {
  if (raw === undefined) return fallback;
  const match = choices.find((choice) => choice === raw);
  if (match === undefined) {
    problems.push({ path, message: `Must be one of: ${choices.join(', ')}` });
    return fallback;
  }
  return match;
}

// A query-string value that must be a whole number of 1 or more (and at most `max`, when given), or
// absent (then `fallback`).
function checkWholeNumber(
  raw: unknown,
  path: string,
  max: number | undefined,
  fallback: number,
  problems: ErrorDetail[],
): number {
  if (raw === undefined) return fallback;
  const value = typeof raw === 'string' && /^\d+$/.test(raw) ? Number(raw) : NaN;
  if (!Number.isSafeInteger(value) || value < 1 || (max !== undefined && value > max)) {
    problems.push({
      path,
      message: max === undefined ? 'Must be a whole number of 1 or more' : `Must be a whole number from 1 to ${max}`,
    });
    return fallback;
  }
  return value;
}

// GET /api/v1/tasks accepts these optional query parameters. A bad value is a 400 VALIDATION_ERROR
// that lists every bad parameter, never ignored.
//   status    todo | in_progress | done        only tasks with this status
//   q         text                             tasks whose title or description contains it
//   sortBy    createdAt | title | priority | dueDate | status    (default createdAt)
//   order     asc | desc                       (default desc)
//   page      1, 2, 3 ...                      (default 1)
//   pageSize  1 to MAX_PAGE_SIZE               (default DEFAULT_PAGE_SIZE)
export function parseListQuery(query: unknown): ListTasksQuery {
  const input = isPlainObject(query) ? query : {};
  const problems: ErrorDetail[] = [];

  let status: ListTasksQuery['status'];
  if (input.status !== undefined) {
    if (isTaskStatus(input.status)) status = input.status;
    else problems.push({ path: 'status', message: STATUS_MESSAGE });
  }

  let q: string | undefined;
  if (input.q !== undefined) {
    if (typeof input.q === 'string' && input.q.trim() !== '') q = input.q.trim();
    else problems.push({ path: 'q', message: 'Must be a non-empty string' });
  }

  const sortBy: SortField = checkChoice(input.sortBy, 'sortBy', SORT_FIELDS, 'createdAt', problems);
  const order: SortOrder = checkChoice(input.order, 'order', SORT_ORDERS, 'desc', problems);
  const page = checkWholeNumber(input.page, 'page', undefined, 1, problems);
  const pageSize = checkWholeNumber(
    input.pageSize,
    'pageSize',
    MAX_PAGE_SIZE,
    DEFAULT_PAGE_SIZE,
    problems,
  );

  if (problems.length > 0) throw new ValidationError(problems);

  const parsed: ListTasksQuery = { sortBy, order, page, pageSize };
  if (status !== undefined) parsed.status = status;
  if (q !== undefined) parsed.q = q;
  return parsed;
}
```

Why the service's `list` is built the way it is (read it again now):

- **`where` is built piece by piece**, so a parameter the client did not send adds nothing to the query.
  `q` becomes an `OR` over `title` and `description`, each `contains`.
- **`sortBy` is checked against a list of five names** in `SORT_FIELDS`, so a client cannot sort by a column
  that is not meant to be public.
- **The sort ends with `{ id: 'asc' }`.** If two tasks share a priority, the database may return them in
  either order, and a task could then land on two pages or on none. A unique tie-breaker makes the order
  total.
- **`skip: (page - 1) * pageSize` and `take: pageSize`** are offset pagination, which is what `page` and
  `pageSize` mean. (Keyset pagination, from M06, uses no `page` number.)
- **`findMany` and `count` run in one `$transaction([...])`**, so `total` and `data` come from the same
  snapshot of the table.

Next, accept the two new fields on create and update. Add these two checks above `parseCreateTask`:

```ts
function checkPriority(value: unknown, problems: ErrorDetail[]): number | undefined {
  if (value === undefined) return undefined;
  if (typeof value !== 'number' || !Number.isInteger(value) || value < 1 || value > 3) {
    problems.push({ path: 'priority', message: 'Must be a whole number from 1 (high) to 3 (low)' });
    return undefined;
  }
  return value;
}

// A due date is a calendar date or a full ISO 8601 date-time, as text. `null` clears it.
const ISO_DATE = /^\d{4}-\d{2}-\d{2}(T\d{2}:\d{2}(:\d{2}(\.\d+)?)?(Z|[+-]\d{2}:\d{2}))?$/;

// new Date('2026-02-30') does not fail: JavaScript rolls it over to 2 March, and the wrong date would
// be stored. So the year, month and day in the text are checked against the calendar as well.
function isRealDay(text: string): boolean {
  const [year, month, day] = text.slice(0, 10).split('-').map(Number) as [number, number, number];
  const check = new Date(Date.UTC(year, month - 1, day));
  return (
    check.getUTCFullYear() === year && check.getUTCMonth() === month - 1 && check.getUTCDate() === day
  );
}

function checkDueDate(value: unknown, problems: ErrorDetail[]): Date | null | undefined {
  if (value === undefined || value === null) return value;
  const date =
    typeof value === 'string' && ISO_DATE.test(value) && isRealDay(value)
      ? new Date(value)
      : undefined;
  if (date === undefined || Number.isNaN(date.getTime())) {
    problems.push({
      path: 'dueDate',
      message: 'Must be an ISO 8601 date such as 2026-12-31, or null',
    });
    return undefined;
  }
  return date;
}
```

In `parseCreateTask` and `parseUpdateTask`, call them after `checkStatus`
(`const priority = checkPriority(input.priority, problems); const dueDate = checkDueDate(input.dueDate, problems);`),
copy each defined result into the object you return (`if (priority !== undefined) task.priority = priority;`
and the same for `dueDate`), and in `parseUpdateTask` add `'priority'` and `'dueDate'` to the list of keys
that count as "something was sent".

**One trap in your own error text.** If your M05 collection asserts the exact message of the
"empty PATCH" error, leave that message alone: it is part of the v1 contract your collection checks, even
though `priority` and `dueDate` now also count as a change.

Now start the server and look at the list yourself (you need the owner from Stage 7, and rows from Stage 11 make
it more interesting, so for now create two tasks):

```bash
npm run typecheck
npm start        # in a second terminal
```

```bash
curl -s -H "X-API-Key: example-api-key-change-me" -H 'content-type: application/json' \
  -d '{"title":"curl probe","priority":3,"dueDate":"2026-12-31T10:00:00Z"}' localhost:3000/api/v1/tasks
curl -s -H "X-API-Key: example-api-key-change-me" "localhost:3000/api/v1/tasks?page=99"
curl -s -H "X-API-Key: example-api-key-change-me" "localhost:3000/api/v1/tasks?pageSize=1000"
```

The first returns the task with `"userId":1`, `"priority":3` and `"dueDate":"2026-12-31T10:00:00.000Z"`. The
second returns `{"data":[],"meta":{"page":99,"pageSize":20,"total":...,"totalPages":...}}`: a page past the
end is empty, with the real `total`. The third returns a 400 whose `details` says
`Must be a whole number from 1 to 100`. (Use your own `API_KEY`.)

**Done when:**

- [ ] `npm run typecheck` exits 0.
- [ ] `GET /api/v1/tasks` returns `{ data, meta }` with `page`, `pageSize`, `total` and `totalPages`.
- [ ] `?status=`, `?q=`, `?sortBy=`, `?order=`, `?page=` and `?pageSize=` each work, and a bad value of
      each returns a 400 `VALIDATION_ERROR`.
- [ ] You can explain why the sort has a tie-breaker, and why `total` and `data` share a transaction.

### Stage 10 — Map Prisma's errors in the existing handler

**Goal:** make `P2002`, `P2025` and `P2003` come out as the right status through the handler you already have,
and prove all three.

Prisma reports a failed query as a `PrismaClientKnownRequestError` with a `code`. Without a mapping, they
would all reach your handler as unknown errors and become a 500. Three of the codes mean "the request does not
fit the data":

| Code | Means | Becomes |
| --- | --- | --- |
| `P2002` | a unique constraint failed (a duplicate) | 409 `CONFLICT` |
| `P2025` | the record the operation needs does not exist | 404 `NOT_FOUND` |
| `P2003` | a foreign key constraint failed | 409 `CONFLICT` |

**P2003 is a choice** (the program says 409 or 400). 409 fits because the request was well formed and conflicts
with the rows that exist: an owner that does not exist, or one that still has tasks. Whichever you pick, say
why.

1. In `src/middleware/error-handler.ts`, add the imports
   (`import { AppError, ConflictError, NotFoundError } from '../errors.ts';` and
   `import { Prisma } from '../generated/prisma/client.ts';`), then add this function above `toAppError`:

   ```ts
   // Prisma reports a failed query as a PrismaClientKnownRequestError with a code. Three of the codes
   // mean "the request does not fit the data", so they map to client errors. The messages are fixed:
   // Prisma's own text holds a code frame of your source and the field names, so it is never sent.
   //   P2002  a unique constraint failed (a duplicate)        -> 409 CONFLICT
   //   P2025  the record the operation needs does not exist   -> 404 NOT_FOUND
   //   P2003  a foreign key constraint failed                 -> 409 CONFLICT
   // P2003 is 409 here, not 400: the request was well-formed, and it conflicts with the rows that exist
   // (an owner that does not exist, or one that still has tasks). Any other code is a bug or an outage.
   function prismaErrorToAppError(err: Prisma.PrismaClientKnownRequestError): AppError | undefined {
     switch (err.code) {
       case 'P2002':
         return new ConflictError('A record with the same unique value already exists');
       case 'P2025':
         return new NotFoundError('Resource not found');
       case 'P2003':
         return new ConflictError('The request refers to, or depends on, a record that conflicts with it');
       default:
         return undefined;
     }
   }
   ```

2. In `toAppError`, right after the `AppError` line, add:

   ```ts
     if (err instanceof Prisma.PrismaClientKnownRequestError) {
       const mapped = prismaErrorToAppError(err);
       if (mapped !== undefined) return mapped;
     }
   ```

   There is no new handler and no new middleware. Anything Prisma throws that is not one of the three codes
   falls through to the 500 branch, so an outage is never reported as a client mistake.

3. **Prove all three.** P2025 is reachable from the API (update or delete an id that does not exist), but
   nothing a client can send to the v1 routes collides with a unique column or sets a foreign key. So P2002 and
   P2003 need a script that makes the **real** errors and sends them through the **real** handler.
   `scripts/check-prisma-errors.ts`:

   ```ts
   import express from 'express';
   import type { AddressInfo } from 'node:net';
   import { prisma } from '../src/db.ts';
   import { errorHandler } from '../src/middleware/error-handler.ts';
   import { requestId } from '../src/middleware/request-id.ts';

   // P2002, P2025 and P2003 cannot be reached through the v1 routes: the task endpoints have no unique
   // column a client can collide with and no foreign key a client can set. So this script makes the
   // REAL errors, by running real failing queries against the real database, and sends each one through
   // the REAL error handler by way of a throwaway Express app. Nothing in the mapping is faked.
   //
   // Each case below says which query fails and why. `expected` is what the mapping must produce.
   type Case = {
     name: string;
     code: string;
     expected: { status: number; code: string };
     run: () => Promise<unknown>;
   };

   const cases: Case[] = [
     {
       name: 'duplicate email',
       code: 'P2002',
       expected: { status: 409, code: 'CONFLICT' },
       // users.email is UNIQUE and the default owner already has this address.
       run: () => prisma.user.create({ data: { email: 'owner@example.com', name: 'Duplicate' } }),
     },
     {
       name: 'update a task that does not exist',
       code: 'P2025',
       expected: { status: 404, code: 'NOT_FOUND' },
       // update() needs the row to exist, and no task has this id.
       run: () =>
         prisma.task.update({
           where: { id: '99999999-9999-4999-8999-999999999999' },
           data: { title: 'x' },
         }),
     },
     {
       name: 'task for an owner that does not exist',
       code: 'P2003',
       expected: { status: 409, code: 'CONFLICT' },
       // tasks.userId is a foreign key, and no user has this id. userId is written directly (an
       // "unchecked" create), so Prisma does not look the user up first and the database refuses it.
       run: () => prisma.task.create({ data: { title: 'orphan', userId: 999_999 } }),
     },
   ];

   const app = express();
   app.use(requestId);
   app.get('/case/:index', async (req) => {
     const found = cases[Number(req.params.index)];
     if (found === undefined) throw new Error('No such case');
     await found.run();
   });
   app.use(errorHandler);

   const server = app.listen(0);
   const { port } = server.address() as AddressInfo;

   let failed = false;
   for (const [index, c] of cases.entries()) {
     // Prisma's own error comes first, to show the code the mapping is keyed on.
     const raw = await c.run().then(
       () => 'no error',
       (err: unknown) => (err as { code?: string }).code ?? 'not a Prisma error',
     );

     const res = await fetch(`http://127.0.0.1:${port}/case/${index}`);
     const body = (await res.json()) as { error: { code: string; message: string } };
     const ok =
       raw === c.code && res.status === c.expected.status && body.error.code === c.expected.code;
     if (!ok) failed = true;
     console.log(
       `${c.code} ${c.name}: Prisma code ${raw} -> HTTP ${res.status} ${body.error.code} "${body.error.message}" ${ok ? 'PASS' : 'FAIL'}`,
     );
   }

   server.close();
   await prisma.$disconnect();
   process.exit(failed ? 1 : 0);
   ```

   Run it, writing the output to a file so a pipe cannot hide the exit status:

   ```bash
   npm run check:errors > check.log 2>&1; echo "exit=$?"; grep -n "PASS\|FAIL" check.log; rm check.log
   ```

   Expect `exit=0` and three `PASS` lines, and nothing else in the log. (Prisma's own `error` log is left off
   in `db.ts` on purpose. With it on, each real failure printed a long code frame, and a normal 404 would too.)

**Done when:**

- [ ] `npm run check:errors` exits 0 and prints `PASS` for `P2002` (409), `P2025` (404) and `P2003` (409).
- [ ] The mapping lives in `error-handler.ts`, in the same `toAppError` as your M05 cases, and no second
      handler exists.
- [ ] You can explain why the messages are fixed text instead of Prisma's, and why an unmapped Prisma code
      still becomes a 500.

### Stage 11 — The seed script

**Goal:** load the M06 sample data through Prisma, so a rebuilt database has something in it.

The seed is explicit: `npm run db:seed` runs it, and nothing runs it for you. It uses `upsert`, which finds a
row by a unique value and creates it only if it is missing, so running it twice creates no duplicates.
Two facts shape it:

- **Users and tags have no hand-written ids.** Migration 3 already created the default owner as user 1, so
  your first M06 user cannot be user 1 as well. Find rows by their unique `email` or `name`.
- **Task ids are UUIDs**, built from your M06 task number, so they never change between runs:
  task 7 is always `00000000-0000-4000-8000-000000000007`.

The pattern, `prisma/seed.ts` (shown with three tasks; **port all of your own M06 `seed.sql`** into the
`users`, `tags` and `tasks` arrays):

```ts
import { prisma } from '../src/db.ts';
import { DEFAULT_OWNER_EMAIL } from '../src/services/task-service.ts';
import type { TaskStatus } from '../src/types/task.ts';

const users = [
  { email: 'ada@example.com', name: 'Ada Lovelace', createdAt: '2026-01-05 09:00:00' },
  { email: 'ben@example.com', name: 'Ben Carter', createdAt: '2026-01-12 10:30:00' },
  // ... the rest of your M06 users
] as const;

const tags = ['backend', 'frontend', 'bug', 'docs', 'urgent', 'chore', 'research'] as const;
type TagName = (typeof tags)[number];

type SeedTask = {
  n: number;
  owner: (typeof users)[number]['email'];
  title: string;
  description: string | null;
  status: TaskStatus;
  priority: number;
  dueDate: string | null;
  createdAt: string;
  tags: TagName[];
};

const ada = 'ada@example.com';

const tasks: SeedTask[] = [
  { n: 1, owner: ada, title: 'Set up the Express project', description: 'Install Express 5 and TypeScript', status: 'done', priority: 1, dueDate: '2026-03-10', createdAt: '2026-03-01 09:00:00', tags: ['backend', 'chore'] },
  { n: 2, owner: ada, title: 'Design the tasks table', description: 'Columns, types and constraints', status: 'done', priority: 1, dueDate: '2026-03-12', createdAt: '2026-03-02 09:30:00', tags: ['backend', 'docs'] },
  { n: 6, owner: ada, title: 'Tidy the README', description: null, status: 'todo', priority: 3, dueDate: null, createdAt: '2026-03-08 16:00:00', tags: [] },
  // ... the rest of your M06 tasks, with their tags
];

// M06 wrote timestamps as 'YYYY-MM-DD HH:MM:SS' text in UTC. A Date needs the T and the Z.
const utc = (text: string): Date => new Date(`${text.replace(' ', 'T')}Z`);
const taskId = (n: number): string => `00000000-0000-4000-8000-${String(n).padStart(12, '0')}`;

async function main(): Promise<void> {
  // The owner of tasks created through the API. The migration creates it too, so on a migrated
  // database this finds it and changes nothing.
  await prisma.user.upsert({
    where: { email: DEFAULT_OWNER_EMAIL },
    create: { email: DEFAULT_OWNER_EMAIL, name: 'Default Owner' },
    update: {},
  });

  for (const user of users) {
    await prisma.user.upsert({
      where: { email: user.email },
      create: { email: user.email, name: user.name, createdAt: utc(user.createdAt) },
      update: {},
    });
  }

  for (const name of tags) {
    await prisma.tag.upsert({ where: { name }, create: { name }, update: {} });
  }

  for (const task of tasks) {
    await prisma.task.upsert({
      where: { id: taskId(task.n) },
      create: {
        id: taskId(task.n),
        title: task.title,
        description: task.description,
        status: task.status,
        priority: task.priority,
        dueDate: task.dueDate === null ? null : new Date(task.dueDate),
        createdAt: utc(task.createdAt),
        // A nested write: Prisma finds the owner and the tags and links them in the same operation.
        owner: { connect: { email: task.owner } },
        tags: { connect: task.tags.map((name) => ({ name })) },
      },
      update: {},
    });
  }

  const [userCount, taskCount, tagCount, links] = await Promise.all([
    prisma.user.count(),
    prisma.task.count(),
    prisma.tag.count(),
    prisma.$queryRaw<{ n: bigint }[]>`SELECT COUNT(*) AS n FROM "_TagToTask"`,
  ]);
  console.log(
    `Seeded: users ${userCount}, tasks ${taskCount}, tags ${tagCount}, task_tags ${links[0]?.n}`,
  );
}

main()
  .catch((err: unknown) => {
    console.error('Seed failed:', err);
    process.exitCode = 1;
  })
  .finally(() => prisma.$disconnect());
```

Notes on the choices:

- **`connect`, not `create`, for the owner and tags.** `connect` links to a row that must already exist and
  fails if it does not (P2025), so a typo in an email stops the seed instead of inventing a user.
- **The counts are printed from real queries**, so you can compare them with your M06 `seed.sql` header. The
  reference build, with the M06 data (5 users, 24 tasks, 7 tags, 37 links) plus the default owner, printed
  `Seeded: users 6, tasks 24, tags 7, task_tags 37`.
- **`$queryRaw` with a tagged template** is how you count the join table, which has no model of its own. A
  value placed in `${...}` is sent as a parameter, not pasted into the SQL: the M06 lesson on injection
  applies to Prisma's raw queries too.

Run it **twice**:

```bash
npm run db:seed
npm run db:seed
```

Both runs must print the same counts.

**Look at the data in Prisma Studio.** Studio is a browser view of your tables. Start it from `task-api/`:

```bash
npm run db:studio
```

It prints `Prisma Studio is running at: http://localhost:5555` and opens your browser (add
`-- --browser none` to stop that, and `-- --port 5600` to change the port). The sidebar lists your tables
(`_prisma_migrations`, `_TagToTask`, `Tag`, `Task`, `User`). Open `Task`: you should see your seeded tasks,
25 to a page, with `priority`, `status`, `dueDate` and `userId`. Open `_TagToTask` to see the join table that
Prisma manages (columns `A` and `B`), and `_prisma_migrations` to see one row per applied migration. Stop it with
Ctrl+C. The script passes `--url file://./dev.db` because Studio does not accept the `file:./dev.db` form (see
Common pitfalls), so it must be run from `task-api/`.

**Done when:**

- [ ] `npm run db:seed` prints your counts, and a second run prints the same counts.
- [ ] `npm run db:studio` serves Studio, and its `Task` table shows your seeded tasks.
- [ ] `npm run typecheck` exits 0 with `prisma/seed.ts` in place.
- [ ] You can explain what `upsert` does, and why the seed finds users by email and not by id.

### Stage 12 — A tour of the Prisma Client, on a throwaway database

**Goal:** use every client call the brief lists once, on a copy of the database, so the service you wrote in
Stage 8 stops being the only way you know the client.

Do this on a database you can throw away, never on `dev.db`:

```bash
export DATABASE_URL=file:./tour.db
npx prisma@7 migrate deploy
npx tsx prisma/seed.ts
```

(`export` lasts for this terminal session only. Close the terminal, or `unset DATABASE_URL`, when you finish.)
Then write a scratch file, `scripts/tour.ts`, that you will **not** commit, and run it with
`npx tsx scripts/tour.ts`:

```ts
import { prisma } from '../src/db.ts';
import { Prisma } from '../src/generated/prisma/client.ts';

const show = (label: string, value: unknown) => console.log(`\n## ${label}\n${JSON.stringify(value, null, 1)}`);

// create, with a nested write: the task, its owner link and two tags in one call
const created = await prisma.task.create({
  data: {
    title: 'Tour: nested write',
    owner: { connect: { email: 'ada@example.com' } },
    tags: { connectOrCreate: [
      { where: { name: 'backend' }, create: { name: 'backend' } },
      { where: { name: 'tour' }, create: { name: 'tour' } },
    ] },
  },
  include: { tags: true },
});
show('create + nested write (include tags)', { title: created.title, tags: created.tags.map((t) => t.name) });

// createMany
const many = await prisma.task.createMany({
  data: [
    { title: 'Tour A', userId: 1, priority: 1 },
    { title: 'Tour B', userId: 1, priority: 3 },
  ],
});
show('createMany', many);

// findUnique / findFirst
show('findUnique by id', (await prisma.task.findUnique({ where: { id: created.id } }))?.title);
show('findUnique by unique email', (await prisma.user.findUnique({ where: { email: 'ben@example.com' } }))?.name);
show('findFirst todo, priority 1, oldest due', (await prisma.task.findFirst({
  where: { status: 'todo', priority: 1 }, orderBy: { dueDate: 'asc' },
}))?.title);

// where: AND / OR / NOT / contains / in / gte
const filtered = await prisma.task.findMany({
  where: {
    AND: [{ priority: { gte: 2 } }, { status: { in: ['todo', 'in_progress'] } }],
    OR: [{ title: { contains: 'doc' } }, { description: { contains: 'doc' } }],
    NOT: { title: { contains: 'Tour' } },
  },
  select: { title: true, priority: true, status: true },
  orderBy: { title: 'asc' },
});
show('AND/OR/NOT/contains/in/gte', filtered);

// skip/take
const pageTwo = await prisma.task.findMany({ orderBy: [{ createdAt: 'asc' }, { id: 'asc' }], skip: 5, take: 3, select: { title: true } });
show('skip 5 take 3', pageTwo.map((t) => t.title));

// select vs include
const selected = await prisma.task.findFirstOrThrow({ where: { id: created.id }, select: { title: true, owner: { select: { name: true } } } });
const included = await prisma.task.findFirstOrThrow({ where: { id: created.id }, include: { owner: true } });
show('select (only what you name)', selected);
show('include (all task fields + the relation)', Object.keys(included));

// update / upsert / delete
const updated = await prisma.task.update({ where: { id: created.id }, data: { status: 'done' } });
show('update', { status: updated.status, updatedAtChanged: updated.updatedAt > created.updatedAt });
const up1 = await prisma.tag.upsert({ where: { name: 'tour-upsert' }, create: { name: 'tour-upsert' }, update: {} });
const up2 = await prisma.tag.upsert({ where: { name: 'tour-upsert' }, create: { name: 'tour-upsert' }, update: {} });
show('upsert twice gives the same row', { same: up1.id === up2.id });
const deleted = await prisma.task.deleteMany({ where: { title: { startsWith: 'Tour A' } } });
show('deleteMany', deleted);
const removed = await prisma.tag.delete({ where: { name: 'tour-upsert' } });
show('delete returns the row it removed', { name: removed.name });

// count and groupBy
show('count where status todo', await prisma.task.count({ where: { status: 'todo' } }));
show('groupBy status', await prisma.task.groupBy({ by: ['status'], _count: { _all: true }, _avg: { priority: true }, orderBy: { status: 'asc' } }));

// $transaction: batch and interactive
const [before, after] = await prisma.$transaction([
  prisma.task.count(),
  prisma.tag.count(),
]);
show('$transaction batch', { tasks: before, tags: after });
try {
  await prisma.$transaction(async (tx) => {
    await tx.task.create({ data: { title: 'Tour: rolled back', userId: 1 } });
    const inside = await tx.task.count({ where: { title: 'Tour: rolled back' } });
    show('interactive, inside the transaction', { rolledBackRowVisibleInside: inside });
    throw new Error('abort on purpose');
  });
} catch (err) {
  show('interactive, after the throw', { message: (err as Error).message, rowsLeft: await prisma.task.count({ where: { title: 'Tour: rolled back' } }) });
}

// $queryRaw with a tagged template: the value is a parameter, not part of the SQL text
const term = 'Tour%';
show('$queryRaw tagged template', await prisma.$queryRaw`SELECT title FROM "Task" WHERE title LIKE ${term} ORDER BY title`);

// Types
const where = { status: 'todo' } satisfies Prisma.TaskWhereInput;
const select = { id: true, title: true } satisfies Prisma.TaskSelect;
const row = await prisma.task.findFirstOrThrow({ where, select });
type TitleOnly = Prisma.TaskGetPayload<{ select: typeof select }>;
const typed: TitleOnly = row;
show('satisfies + GetPayload', Object.keys(typed));
// The same idea for include: the result type is the task plus the relation you included.
type WithOwner = Prisma.TaskGetPayload<{ include: { owner: true } }>;
const withOwner: WithOwner = included;
show('GetPayload with include', { ownerName: withOwner.owner.name });
const input: Prisma.TaskCreateInput = { title: 'typed', owner: { connect: { id: 1 } } };
show('TaskCreateInput compiles', Object.keys(input));

await prisma.$disconnect();
```

What the reference build printed, so you know what to look for (your numbers follow your own seed data):

| Call | Result in the reference build |
| --- | --- |
| `create` with a nested `connectOrCreate` | one task, with tags `backend` and `tour` |
| `createMany` | `{ "count": 2 }` |
| `findUnique` by `email` | `"Ben Carter"` (`findUnique` takes only a unique field) |
| `findFirst` todo, priority 1, `orderBy dueDate asc` | `"Tour A"`, **a task with no due date** (see Concepts) |
| `AND`/`OR`/`NOT`/`contains`/`in`/`gte` | two tasks, "Document the junction table" and "Write the API usage docs" |
| `skip: 5, take: 3` | three titles, the sixth to eighth oldest |
| `select` against `include` | `select` returned only `title` and `owner.name`. `include` returned **every task field** plus `owner`. |
| `update` | the status changed, and `updatedAt` moved forward by itself (`@updatedAt`) |
| `upsert`, twice | the same row both times |
| `count` and `groupBy` | `count` of `todo` was 13. `groupBy status` returned `_count` and `_avg.priority` per status. |
| batch `$transaction([...])` | both counts, taken together |
| interactive `$transaction(async (tx) => ...)` | the row was **visible inside**, and **gone after the throw** (`rowsLeft: 0`) |
| `$queryRaw` with a tagged template | the two `Tour%` titles, `${term}` sent as a parameter |
| `delete` | the row it removed, `{ "name": "tour-upsert" }` (`deleteMany` returned `{ "count": 1 }`) |
| `satisfies` and `GetPayload` | the type has exactly `id` and `title`, so the compiler knows the shape that `select` produced |
| `GetPayload` with `include` | the type is the task plus its `owner`, so `withOwner.owner.name` compiles |

Check that the types compile **before** you delete the scratch file. `tour.ts` is in `scripts/`, so the type
check covers it, and a clean run proves that `satisfies`, both `GetPayload` forms and `TaskCreateInput` compile:

```bash
npm run typecheck
```

Then see the SQL Prisma writes: run the tour again with `LOG_QUERIES=true npx tsx scripts/tour.ts`. Every call
now prints `prisma:query` lines first. For example, the seed script's `count` calls print a line like
``prisma:query SELECT COUNT(*) AS `_count$_all` FROM (SELECT `main`.`User`.`id` FROM `main`.`User` WHERE 1=1 LIMIT ? OFFSET ?) AS `sub` ``
(the values are sent separately, so the SQL has `?` where they go). Find the SQL for a nested write and a
`groupBy` in the tour's output too, and compare them with the call that produced them. Delete the scratch file
and the database when you finish, and unset the variable:

```bash
rm -f scripts/tour.ts tour.db tour.db-journal
unset DATABASE_URL
```

**Done when:**

- [ ] The tour runs to the end and you can point at the output of `create`, `createMany`, `findUnique`,
      `findFirst`, `findMany` with `AND`/`OR`/`NOT`/`contains`/`in`/`gte`, `orderBy`, `skip`/`take`,
      `select` and `include`, a nested write, `update`, `upsert`, `delete`, `count`, `groupBy`, both kinds of
      `$transaction`, and `$queryRaw`.
- [ ] `scripts/tour.ts` and `tour.db` are deleted, and `git status` does not list them.
- [ ] You can explain `select` against `include` and what the transaction callback rolled back.

### Stage 13 — Update the Bruno collection, then prove the rebuild

**Goal:** make the collection pass against v2, and prove that `migrate reset` plus the seed rebuilds the
database from nothing.

1. **Update only the requests that describe the v1 response shape.** The responses changed in two ways: a
   task has more fields, and the list is wrapped in `{ data, meta }`. In the reference collection that
   touched three requests, and no error request:

   - *Create task* (`02`): the keys it asserts are now `createdAt`, `description`, `dueDate`, `id`,
     `priority`, `status`, `title`, `updatedAt` and `userId`, and `priority` is `2`, `dueDate` is `null`.
   - *List tasks* (`03`): assert `res.body.data` is an array, `res.body.meta.page` is `1` and
     `res.body.meta.pageSize` is `20`, and the created task's id is in `res.body.data`.
   - *List tasks filtered by status* (`04`): read `res.body.data` instead of `res.body`.

   Every other request, **all of the error requests included**, must pass unchanged. That is the point of the
   brief's "still green".

2. **Add requests for what is new**, so v2's list is covered too. A `Listing/` folder that creates its own task,
   so it passes on an empty database as well as a seeded one:
   *create a task to find* (title `Listing probe zebra`, `priority` 1, `dueDate` `2026-12-31`; save its id),
   *search by text* (`?q=ZEBRA`, which proves case-insensitive matching), *sort by priority*
   (`?sortBy=priority&order=asc&pageSize=100`, asserting the priorities never go down), *a page of one*
   (`?pageSize=1`, asserting `meta.totalPages` equals `meta.total`), and *delete the task to find*. Add two error
   requests: `?page=0&pageSize=1000`, asserting a 400 whose `details` names both `page` and `pageSize`, and a
   create with `"dueDate": "2026-02-30"`, asserting a 400 on `dueDate` (without the calendar check in Stage 9,
   that date is accepted and stored as 2 March).

3. **Rebuild from nothing.** `migrate reset` drops the database and replays every migration on an empty one,
   so it tests the one thing a normal `migrate dev` never does: your hand-edited migration on a table with no
   rows.

   ```bash
   npm run db:reset
   sqlite3 dev.db "SELECT count(*) FROM User; SELECT count(*) FROM Task; SELECT count(*) FROM Tag;"
   npm run db:seed
   ```

   Expect `Database reset successful`, then `1`, `0`, `0` (the owner, from your migration, and nothing else:
   **a reset does not run the seed**), then your seed counts. Now do the whole cycle a **second time** (`npm run db:reset`, then
   `npm run db:seed`): the counts must be the same, and you should also run the seed once more without a reset,
   where they must not change either. If you use an AI coding agent, note that Prisma refuses to run `migrate reset` when it detects one,
   until the person who owns the database consents. That is by design: it deletes all data.

4. Run the whole collection against the rebuilt database:

   ```bash
   npm start        # in a second terminal
   npm run bruno
   ```

   The reference run printed `Status ✓ PASS`, `Requests 36 (36 Passed)`, `Tests 49/49`, `Assertions 76/76`
   (the 29 from v1, with 3 edited, plus 5 in `Listing/` and 2 new error requests). Stop the server with
   Ctrl+C and check that it prints `Server closed`.

5. Also prove `migrate deploy` on a database that does not exist yet:

   ```bash
   DATABASE_URL=file:./deploy-check.db npx prisma@7 migrate deploy
   sqlite3 deploy-check.db "select count(*) from User; select count(*) from _prisma_migrations;"
   rm -f deploy-check.db deploy-check.db-journal
   ```

   It creates the file and applies the three migrations in order. `User` has `1` row (the owner your
   migration made) and `_prisma_migrations` has `3`.

**Done when:**

- [ ] `npm run db:reset` then `npm run db:seed` rebuilds the database, and a second reset-and-seed gives the
      same counts. (Brief: *`migrate reset` plus the seed script rebuilds the database from scratch, every time.*)
- [ ] `npm run bruno` against that database prints `PASS`, with at least as many requests as your Stage 1
      baseline. (Brief: *the Bruno collection from M05 is still green against v2.*)
- [ ] `migrate deploy` built a new database from the migrations alone.

### Stage 14 — Explain it, document it, self-review, and open the pull request

**Goal:** be able to say the difference between the two migrate commands, tidy the repo, and deliver it.

1. **Explain `migrate dev` against `migrate deploy` without notes.** Write your own answer, using what you saw
   in Stages 5, 7 and 13. The model to check yourself against: `dev` is for your machine and **writes**: it
   turns a schema change into a new migration, applies it, and offers to wipe the database if an applied
   migration was edited (Stage 7). `deploy` is for CI and production and **only applies**: it replays the
   committed migrations in order and writes none. Because it never works SQL out from the schema, a server runs
   only the files that were reviewed in a pull request.

2. **Update `task-api/README.md`** for v2: the setup commands (`npm install`, copy `.env`, `npm run db:generate`,
   `npm run db:deploy`, `npm run db:seed`), the new scripts, `DATABASE_URL` and `LOG_QUERIES`, the list query
   parameters, the new task fields, `CONFLICT` and `NOT_FOUND` for the Prisma codes, and the three migrations
   (say which one is hand-edited and why). A fresh clone has no generated client, so the README must tell the
   next person to run `npm run db:generate`.

3. **Check the version lock and the one-offs:**

   ```bash
   grep -n '"prisma"\|"@prisma/client"' package.json
   npm ls prisma @prisma/client
   ```

   Both must show `^7` in `package.json` and `7.x.x` installed. Look through your shell history and your
   README for a bare `npx prisma`: there must be none.

4. **Run everything once more, and check what git would commit:**

   ```bash
   npm run typecheck
   git status
   ```

   `git status` must not list `dev.db`, `src/generated/prisma/`, `.env`, `tour.db`, `node_modules`, `.agents/`,
   `.claude/`, `.windsurf/` or `skills-lock.json`. It must list `prisma/migrations/` (all three, plus
   `migration_lock.toml`).

5. **Commit in small, labelled pieces**, from the repo root, with Conventional Commits, for example:

   ```bash
   git add task-api/package.json task-api/package-lock.json task-api/.env.example task-api/prisma7.config.ts task-api/tsconfig.json
   git commit -m "chore(m07): add Prisma 7, the SQLite adapter and the Prisma config"
   git add task-api/prisma/schema.prisma task-api/prisma/migrations
   git commit -m "feat(m07): add the Prisma schema and three migrations, one hand-edited"
   git add task-api/src task-api/scripts
   git commit -m "feat(m07): replace the in-memory store with a Prisma task service"
   git add task-api/prisma/seed.ts
   git commit -m "feat(m07): add the seed script"
   git add task-api/bruno
   git commit -m "test(m07): update the Bruno collection for v2"
   git add task-api/README.md README.md
   git commit -m "docs(m07): describe Task API v2"
   ```

   If `git add` ever says `pathspec ... did not match any files`, you are in a different folder from the one
   the path starts in. Check with `pwd`. (`git rm` already staged the store's deletion; it commits with the
   `src` piece.)

6. Push the branch and open a pull request into your own repo's `main`, then merge it after review. The
   merge of this branch **is** Task API v2.

**Done when:**

- [ ] You can explain `migrate dev` against `migrate deploy` in your own words, unaided.
- [ ] `package.json` locks `prisma` and `@prisma/client` to `^7`, and no command you ran or documented is a bare
      `npx prisma`.
- [ ] Every box in section 6 is ticked, and `git status` is clean.
- [ ] A pull request for the branch exists and `main` is untouched until it is merged.

## 4. Concepts you need

### Quick reference

| Idea | One line |
| --- | --- |
| Migration | A numbered SQL file; the folder of them is the history of your schema |
| `migrate dev` | Development: turns schema changes into a new migration and applies it |
| `migrate deploy` | CI and production: applies the committed migrations, creates none |
| `migrate reset` | Development only: drops the database, replays every migration |
| `migrate status` | Compares the migration folder with what the database says it applied |
| `db push` | Prototyping only: forces the database to match the schema with no migration file |
| `generate` | Writes the typed client from `schema.prisma`; run it after install and every schema change |
| Driver adapter | The package that opens the database for Prisma Client in Prisma 7 |
| `select` / `include` | Name the fields you want / add a relation to all the fields |
| P2002 / P2025 / P2003 | Unique violation / record not found / foreign key violation |

### What an ORM is for, and when to use raw SQL

An ORM maps tables to typed objects. In return for the SQL it hides, you get queries the compiler checks, one
schema file that describes your data, and a migration history. The costs are the ones you can already guess
from M06: you cannot see the SQL unless you ask (set `LOG_QUERIES=true`), a slow query is easy to write by
accident, and unusual queries (a report with several joins and a window function) are awkward to express.
Drop to raw SQL (`$queryRaw`) for those, and when you do, use the **tagged template**, which sends `${value}`
as a parameter. The raw call returns untyped rows, so you give the result a type yourself. The ORM does not
replace SQL knowledge: it is the reason to have it.

### The migration history as a mental model

Three things must agree: the **folder** `prisma/migrations/`, the **table** `_prisma_migrations` inside the
database (one row per applied migration, with a checksum), and `schema.prisma`. Each command is a different
question about them:

- **`migrate dev`** asks "does `schema.prisma` match the history?" and, if not, writes the difference as a new
  folder and applies it. If an applied file's checksum no longer matches, it cannot go on and offers a reset.
- **`migrate status`** asks only "which folders has this database applied?", so it is not a check that the
  history is intact.
- **`migrate deploy`** applies the folders the database has not applied yet and stops there. Run on a database
  file that does not exist, it replays the whole chain (Stage 13).
- **`migrate reset`** discards the database and then does the same replay, so it is the way to re-test the
  **whole chain from nothing** on your development database, including a hand-edited file on an empty table.
- **`db push`** skips the folder entirely: it changes the database to match the schema and records nothing.
  That is quick for a prototype and useless for a team, because nobody else can replay what you did. It was
  not used in this build.

**Migration conflicts in a team** (reasoning, not something this build ran). Migrations are ordered by their
timestamped folder name. If two branches each add a migration from the same starting point, both folders merge
into `main` and each was generated without knowing about the other. The usual repair, before either is applied
anywhere shared, is to rebase, delete the migration folder that belongs to your branch, and run
`migrate dev` again so Prisma regenerates it on top of the other one. Any development database that already
applied your deleted migration now disagrees with the folder, so expect `migrate dev` to ask for a reset there.
Once a migration has been applied to a shared database, you never edit or delete it: you add another.

### Implicit against explicit many-to-many, and `onDelete`

An **implicit** many-to-many (`tags Tag[]` on `Task`, `tasks Task[]` on `Tag`) lets Prisma own the join table,
`_TagToTask`, with two columns, `A` and `B`. You cannot add a column to it, and you cannot name it. An
**explicit** many-to-many is a model of your own (`TaskTag` with two relations), which you pick as soon as the
link needs data of its own (who tagged it, when) or a name you control. The `onDelete` option on a relation is
what the database does to the child when the parent goes: `Cascade` deletes the children (what you used for
the owner), `Restrict` refuses the delete while children exist, `SetNull` clears the foreign key (so the field
must be optional), and `NoAction` leaves it to the database. The choice changes which Prisma error you meet:
a refused delete is a P2003.

### The generated client, and why its types follow your queries

`generate` writes real TypeScript into `src/generated/prisma/`: one type per model, one for each enum, and
input types for every call, such as `Prisma.TaskWhereInput` and `Prisma.TaskCreateInput`. The type of a result
depends on the arguments you pass. `select: { id: true, title: true }` returns `{ id, title }`, not a `Task`.
`Prisma.TaskGetPayload<{ select: typeof select }>` names that type so you can reuse it, and
`satisfies Prisma.TaskSelect` on the `select` object checks it is valid **without widening its type** (a plain
annotation would make the compiler forget which fields you chose). Because the files are written from the
schema, they are stale the moment the schema changes: the compiler will not warn you until you regenerate.

### `select` against `include`

`include` adds relations **on top of** all the scalar fields. `select` replaces the default: only what you
name comes back, and you can reach into a relation inside it. Use `select` when you want a small payload or
want to avoid sending a column to a caller; use `include` when you need the whole row and its relation. They
cannot be combined at the same level.

### Offset pagination, and what `NULL` does to a sort

`skip`/`take` is offset pagination: the database counts past `skip` rows every time, so deep pages get slower,
and rows inserted between two requests shift the pages. Keyset pagination (M06, exercise 6) avoids both but
has no page number, so the brief's `page`/`pageSize` contract needs offset. Whatever you pick, a sort needs a
unique tie-breaker.

Sorting also has a quiet rule: in SQLite, **`NULL` sorts before every value in ascending order**. In the
reference build `ORDER BY dueDate ASC` listed the tasks with **no** due date first, and the endpoint's
`?sortBy=dueDate&order=asc` does the same. If "no date" should sort last, say so in your API's contract, or
change the query.

### Enums, constraints and who enforces a rule

A rule can live in the database, in Prisma, or in your validation. M06's `CHECK (status IN ...)` was in the
database. Prisma's `enum` on SQLite is **text with no `CHECK`**, so only Prisma and your validation enforce the
three values. A client that writes to the file with raw SQL can store anything. The more places that enforce a
rule, the harder it is to break, and the more places you must keep in step.

### Why the error handler maps codes and never forwards Prisma's text

The `message` of a Prisma error is built for the developer, not the client: in the reference build it held the
path of the source file that made the call, a code frame of your own source, and the field names. The handler
matches on the stable **code** instead, as it already does for body-parser errors. The code also decides blame:
P2025 and P2002 mean the *request* did not fit the data (4xx), while a connection error, a timeout or any other
code means *you* have a problem (5xx), so unmapped codes stay 500.

### The concept-only topics (no lab)

You are not asked to build any of these. Know the shape of the answer.

- **PostgreSQL.** The schema's `datasource` provider becomes `postgresql`, the client is built with
  `@prisma/adapter-pg` (`new PrismaPg({ connectionString })`) in the same place `db.ts` builds the SQLite one,
  and the connection string points at a server instead of a file. Because a server has many clients, the
  adapter's connection pool matters (it uses the `pg` package's defaults, which Prisma's documentation notes
  differ from Prisma ORM 6's), and hosts such as Neon and Supabase give a pooled URL for the running app and a
  direct one for the command line tool. PostgreSQL also has richer types than SQLite (for example `jsonb`),
  and enforces your enums and constraints in the database. (Prisma 7 documentation, PostgreSQL quickstart.)
- **MongoDB.** A document database stores nested JSON-like documents in collections, with no joins and no
  foreign keys, so "a task with its tags" is usually one document with the tags inside it. Prisma models
  embedded documents as composite types and uses `ObjectId` ids. There is no Prisma Migrate for MongoDB: the
  schema is applied with `db push`. The Prisma 7 pages disagreed on whether Prisma 7 supports MongoDB at all,
  so check the current documentation before relying on it.
- **Prisma 8.** It was a release candidate when this was written. Prisma's "coming from Prisma ORM 7" page
  describes `schema.prisma` becoming a contract file, the `generator` block going away, `generate` becoming
  `emit`, `migrate dev` and `migrate deploy` being replaced by other commands, and queries written as chains
  (`.where(...).all()`). That page uses PostgreSQL throughout and does not describe SQLite, so this program
  stays on Prisma 7 and locks to `^7`. Prisma says Prisma 7 keeps receiving fixes and security updates for 18
  months after Prisma 8 is generally available.

Prisma 7 pages for the two databases you set up or compared in this stage:
[SQLite quickstart](https://www.prisma.io/docs/v7/prisma-orm/quickstart/sqlite) and
[PostgreSQL quickstart](https://www.prisma.io/docs/v7/prisma-orm/quickstart/postgresql). Use `/v7/` pages only.

## 5. Common pitfalls

- **An unversioned `npx prisma` or `npm install prisma`.** It resolves to Prisma 8, whose command line tool
  does not read your `schema.prisma`. Use `npx prisma@7`, install `prisma@7` and `@prisma/client@7`, and lock
  both to `^7`.
- **Two copies of `better-sqlite3`.** Installing it with no version takes the newest major, but the Prisma
  adapter pins its own. Install `better-sqlite3@12`, then check `npm ls better-sqlite3` shows one version.
  If the native module will not load at all, install your platform's build tools (the Xcode command line
  tools on macOS), or use `@prisma/adapter-libsql`, which changes the install commands in Stage 2 and the
  adapter in `db.ts`. An `allowScripts` warning during install did not break the reference build.
- **`prisma init` writes more than a schema.** It also created AI-assistant skill folders (`.agents/`,
  `.claude/`, `.windsurf/`), `skills-lock.json` and an extra `.gitignore`. Delete them. The extra `.gitignore`
  can make your ignore check pass for the wrong reason, so prove the **root** rule with
  `git check-ignore -v` from the repo root.
- **Forgetting `prisma generate`.** `migrate dev` did not run it in 7.10, so a fresh clone, a fresh install or
  a schema change leaves you with a missing or stale client. Run `npm run db:generate` after install, after
  every schema change, and in CI.
- **Prisma 7 does not read `.env`.** Without `process.loadEnvFile` (or `dotenv`) in the config file, the command
  line tool sees no `DATABASE_URL`. The running app has its own check in `db.ts`.
- **A required column with no default on a table that has rows.** Prisma refuses, and the file it still writes
  would fail. Use `--create-only`, read the SQL, and write the backfill yourself. Test it on a table with rows
  **and** on an empty one (`migrate reset`).
- **`migrate dev` on a database you cannot lose.** It may offer a reset. `migrate reset` and a modified
  applied migration both end in "all data will be lost". Never point them at production. `migrate deploy` is
  the one for a server.
- **Editing an applied migration.** `migrate status` still says "up to date". `migrate dev` is the command
  that notices, and then asks to wipe the database. Add a new migration instead.
- **Believing a reset seeds the database.** In Prisma 7.10 it does not. Run `npm run db:seed` yourself.
- **An enum on SQLite is not a `CHECK`.** Do not rely on the database to reject a bad status.
- **Every column change on SQLite is a table rebuild.** Expect `new_Task`, a copy, `DROP TABLE` and a rename in
  the generated SQL, and remember the copy statement is the part you edit.
- **`orderBy dueDate asc` lists undated tasks first** in SQLite. A `findFirst` for "oldest due" can return a
  task with no due date.
- **`new Date('2026-02-30')` is not invalid.** JavaScript rolls an impossible day over to the next month, so a
  validator that only checks the pattern and `Number.isNaN(date.getTime())` stores 2 March and returns 201.
  Check the year, month and day against the calendar too (`isRealDay` in Stage 9), and test the dates you
  would not normally try: `2026-02-30`, `2026-04-31`, `2026-02-29`.
- **`LIKE` wildcards in a search term.** `contains` becomes `LIKE '%q%'`, and a `%` or `_` inside `q` is a
  wildcard, so `?q=%25` matches every task. The value is still a parameter, so it is not injection, but you
  cannot search for a literal `%` or `_` this way. Know it, document it, and use `$queryRaw` with an `ESCAPE`
  clause if the feature ever needs it.
- **P2002 and P2003 can be unreachable from your routes.** If no endpoint can trigger a code, a Bruno request
  cannot prove its mapping. Make the real error in a script and send it through the real handler.
- **Prisma Studio says it does not support `file:./dev.db`.** The message is
  `Prisma Studio is not supported for the "file:./dev.db" protocol`. Studio reads the protocol as everything
  before `://`, so the single-colon `file:` form that `DATABASE_URL` uses is rejected. Use
  `--url file://./dev.db` (as the `db:studio` script does) from inside `task-api/`. Do not write `file://dev.db`
  or `file:///dev.db`: they point at `/dev.db` at the root of your disk, and Studio then reports that it
  created a new empty file there.
- **A bare `PrismaClient` per request.** Build one client in `db.ts` and import it everywhere.
- **Query logging on by default.** A seed prints one line per statement. Keep logging opt-in.
- **Committing the database, the generated client or `.env`.** `git status` must not list `dev.db`,
  `src/generated/prisma/` or `.env`. Commit `.env.example` with fake values only.

## 6. Check yourself against the definition of done

Tick every box. Each one is something you can show or say.

- [ ] `migrate reset` plus the seed script rebuilds the database from scratch, every time. You ran it more
      than once and the counts matched.
- [ ] The Bruno collection from M05 is still green against v2 (`npm run bruno` prints `PASS`).
- [ ] You can explain `migrate dev` against `migrate deploy` in your own words, without notes.
- [ ] `P2002`, `P2025` and `P2003` each map to the right status code through your existing M05 error handler,
      not a new one (`npm run check:errors` exits 0).
- [ ] `package.json` locks `prisma` and `@prisma/client` to `^7`, and every one-off command uses
      `npx prisma@7`, never a bare `npx prisma`.
- [ ] The deliverable exists: the M06 schema translated into Prisma, **at least two migrations** and one of
      them **hand-edited** via `--create-only`, a service layer replacing the in-memory store, a list with
      `status`/`q` filters, sorting and `page`/`pageSize` pagination returning `{ data, meta }`, a seed script,
      and Prisma errors mapped into the existing error handler.
- [ ] You used every client call in the brief's list at least once (Stage 12), and `npm run typecheck` exits 0.

## 7. Next step

M08 replaces your hand-written validation with Zod 4, and the type of every request body comes from
`z.infer`. Prisma stays the persistence contract: invalid input must never reach it. Keep your migrations and
your service layer. They are what the Zod-validated routes call.
