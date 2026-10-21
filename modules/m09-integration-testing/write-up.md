# M09 Write-Up — Integration Testing with Jest + Supertest (Task API v4)

> Written from what I captured as I built it: raw notes after each slice, then this document. Written
> so a trainee starting this stage next cohort could follow my path on their own. The M10 capstone
> has no reference material except what is written here.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo,
[`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
inside `task-api/`, on the branch `feat/module-09/integration-testing`. The Task API is one project that
grows from M05 to M09, so v4 is a change to `task-api/` in place, not a new folder.

The pull request is [PR #9](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/9), merged into `main` on 2026-10-21 at 14:12 UTC as merge commit
`ed7d65b`. It has 11 commits: the M08 comment fix, the dependencies, the test database,
the tests for the existing endpoints, the failing `complete` tests, the route, the refactor, the Bruno
requests, the workflow, the docs, and a rename of the CI job. CI was green on it and on `main` after the
merge (see Checkpoint evidence).

```text
.github/workflows/ci.yml            NEW  npm ci, generate, type-check, test with coverage (repo root)
.gitignore (repo root)              *.db-journal, *.db-wal and *.db-shm added beside *.db
task-api/
├── jest.config.js                  NEW  ESM, @swc/jest, globalSetup and globalTeardown, coverage scope and threshold
├── .env.test.example               NEW  the test settings, fake values (the real .env.test is git-ignored)
├── tests/
│   ├── global-setup.ts             NEW  loads .env.test, makes this run's own database, runs prisma migrate deploy on it
│   ├── global-teardown.ts          NEW  deletes this run's database
│   ├── helpers/test-database.ts    NEW  testDatabasePath (the guard) and perRunDatabaseUrl
│   ├── helpers/db.ts               NEW  resetDb (beforeEach, with its own guard) and disconnectDb (afterAll)
│   ├── helpers/factories.ts        NEW  makeTask, makeTasks, defaultOwner
│   ├── helpers/api.ts              NEW  authed() Supertest agent and anonymous()
│   ├── tasks.create.test.ts        NEW  17 tests
│   ├── tasks.read.test.ts          NEW   3 tests
│   ├── tasks.update.test.ts        NEW   7 tests
│   ├── tasks.delete.test.ts        NEW   4 tests
│   ├── tasks.list.test.ts          NEW  17 tests
│   ├── tasks.complete.test.ts      NEW   7 tests, written before the route
│   ├── middleware.test.ts          NEW  14 tests
│   ├── error-mapping.test.ts       NEW   2 tests
│   └── test-database.test.ts       NEW  17 tests, for the database guards
├── src/routes/tasks.routes.ts      PATCH /:id/complete
├── src/handlers/tasks.handlers.ts  completeTask
├── src/schemas/task.schemas.ts     completeTaskRequest
├── src/services/task-service.ts    complete(); and the M08 comment fix (see below)
├── bruno/                          Complete/ (4 requests), Errors/29 and 30
├── tsconfig.json                   "tests" added to include, so typecheck covers the tests
├── package.json                    jest, @jest/globals, @swc/core, @swc/jest, supertest, @types/supertest; test scripts
└── README.md                       v4, a Testing section, the badge
```

- **88 tests in 9 files** (71 integration tests plus 17 for the database guards), against the real exported
  Express app (Supertest, no port) and a real SQLite database. Nothing in the data layer is mocked.
- **A test database that is new for every run and cannot be `dev.db`.** `.env.test` names `file:./test.db`.
  A Jest `globalSetup` turns that into `test-<pid>-<time>.db`, runs `prisma migrate deploy` on it and sets
  `DATABASE_URL` for the test files, and a `globalTeardown` deletes it. `resetDb()` empties the tables in
  `beforeEach`. The name is checked twice, in the setup and again in `resetDb()`, on the resolved file path.
- **`PATCH /api/v1/tasks/:id/complete`**, added test-first in red, green and refactor.
- **A GitHub Actions workflow** at the repo root, and a status badge in both READMEs.
- **Review fixes** (after a read-only code review, see "Other decisions"): the per-run database, the second
  guard, a stricter `updatedAt` test, header comments, and the journal files in `.gitignore`.
- **The M08 follow-up fix**: the comment in `src/services/task-service.ts` that said a schema field
  Prisma cannot store "stops compiling" now says what `satisfies` really catches and does not catch.

The stack as installed: Node 24.22.0 (my shell defaulted to 22.23.0, see "What tripped me up"),
`jest@30.5.2`, `supertest@7.3.1`, `@swc/jest@0.2.39`, `@swc/core@1.16.18`, `@types/supertest@7.2.1`,
with `prisma@7.10.1`, `typescript@7.0.2`, `better-sqlite3@12.11.1` from before. `npm run typecheck`
exits 0.

## Why it's built this way (key decisions)

### Why this test-database strategy, and what breaks without the cleanup

The strategy is the one the architecture document fixes: a separate `test.db`, `prisma migrate deploy` once
in `globalSetup`, the tables emptied in `beforeEach`, and `--runInBand`. One refinement came later, from a code
review: the file is new for every run (see below).

I ran the experiment to see what breaks, instead of assuming. I made `resetDb()` do nothing and ran the
suite: 20 of the 63 tests that existed then failed (43 passed), in 4 of 7 files.
What failed was never the code under test. The create tests that check "nothing was stored" saw rows left
by an earlier file. Every list test saw tasks it did not make (an "empty list" test found some, the total
was wrong, the order included strangers). A delete test that checks the other tasks survive counted the
wrong number. A foreign-key test that checks nothing was stored found rows. I restored `resetDb()` and the
suite went back to 63 of 63. The point I take from it: without cleanup the result depends on which test ran
before, which is the definition of a flaky test, and it would have passed or failed differently on a
different machine.

Three other choices follow from the same idea:

- **One writer.** SQLite lets one connection write at a time, and the tests write constantly, so the `test`
  script uses `--runInBand` (one process, files one after another). The brief's alternative, one database
  file per worker with `JEST_WORKER_ID`, would be faster and I did not need it: the whole suite runs in
  about 1.4 seconds without coverage.
- **One database per run, not one shared `test.db`.** My first version used a fixed `test.db`, as the
  architecture document says. A code review found the weakness, and the review itself showed it: two
  reviewers ran Jest at the same time (I had started them in parallel), and the runs on the shared file
  failed 1 to 12 tests at a time, in different places each run, with counts and lists that were wrong. The
  same suite passed 15 of 15 on a private file. `--runInBand` only orders the files inside one process. So
  `globalSetup` now builds `test-<pid>-<time>.db` from the configured name, and `globalTeardown` deletes it.
  I then started three runs at once with three seeds: all passed, and no file was left behind. A run killed
  with SIGKILL cannot run the teardown, so it can leave a file behind (git-ignored, safe to delete).
- **A guard on the database name, in two places.** A test run deletes every row. `prisma7.config.ts` also
  loads `.env`, which holds `DATABASE_URL=file:./dev.db`, so a missing `.env.test` must stop the run and not
  fall back to real data. The first version only checked that the file name started with `test`, once, in
  `globalSetup`. The review pointed out two holes: a URL like `file:./dev.db?x=/test.db` has a base name that
  looks right, and `resetDb()` trusted that the setup had run. The guard now lives in
  `tests/helpers/test-database.ts`, checks the resolved path (a `file:` URL, the query string cut off first,
  the name `test...` ending in `.db`) and throws on anything unexpected. It runs in `globalSetup`, in
  `globalTeardown` (so only a test database is ever deleted) and at the start of `resetDb()`. It has 17
  tests of its own, and I checked they fail when the guard is weakened: with the name check disabled, 9 of
  them failed. I checked the MD5 of `dev.db` (`6367cdfa...`) before and after: unchanged.

One limit of that guard: `resetDb()` checks `process.env.DATABASE_URL`, but Prisma keeps the URL it was built
with when `db.ts` was imported, so a test that changes the variable afterwards only proves the check runs
first, not that Prisma moved. I did not find a way for the two to differ in this suite.

I also found that the setup file I first wrote to load `.env.test` into each test file was not needed:
`globalSetup` loads it before Jest starts the test files, and they inherit what it set. I took the setup file
out and ran the suite with and without workers: 71 of 71 both times. So the setup is one file less.

### Which toolchain fallback I needed: none

`@swc/jest` worked with TypeScript 7, ESM, the Prisma-generated client (`.ts` files in
`src/generated/prisma/`), the `better-sqlite3` addon and the project's `.ts` import specifiers on the first
try. A one-test smoke file that imported `app` and `prisma` and called `GET /health` passed (1 test,
0.314 s). So no fallback was applied and none of the three (ts-jest, TypeScript 6, CommonJS) was tried. The
only output besides the result was Node's `ExperimentalWarning: VM Modules`, which is expected with
`--experimental-vm-modules`. The `.ts` specifiers needed no `moduleNameMapper`, which is the P2 convention
the architecture document proposes: Jest's resolver finds a file named `task-service.ts` literally.

### What writing `PATCH /:id/complete` test-first changed

I wrote seven tests before any of the route, handler or service existed. Three things came out of that.

1. **The behaviour was decided before the code.** Completing a task that is already done had to be either a
   200 or a 409, and writing the test forced me to pick. I picked 200 (completing twice leaves the same
   state, so a client can safely retry). With the handler open first I would have written whatever the
   `update` call did and called that the design.
2. **The first red run was weaker than it looked.** Seven tests ran: 4 failed and 3 passed with no route.
   The three passing ones passed for the wrong reason. "Unknown id gives 404" passed because the 404 handler
   answered for a route that did not exist. "Only the named task is completed" passed because nothing ran
   at all. "No key gives 401" passed because the key check runs before routing, so it is correct, but it
   proves nothing about the new route. I fixed the first two before writing any code: the 404 test now
   asserts the message `Resource not found` (Prisma's P2025 mapped by the error handler) and not `Route not
   found: ...`, and the second asserts that the target did become `done`. The second red run was 6 failed, 1
   passed, and the one that passes is the 401, for the reason above.
3. **The handler stayed tiny.** Green was a schema (`{ params: taskParams }`, reused), a six-line handler,
   `prisma.task.update({ where: { id }, data: { status: 'done' } })` and one route line, and the whole suite
   was 70 of 70. Refactor then removed the second place that writes a task: `complete(id)` now calls
   `taskService.update(id, { status: 'done' })`, so there is one write path and the missing-task 404 is the
   same code. The tests did not change and stayed green, which is what a refactor is.

### Other decisions

- **No endpoint can produce a 409, so I tested the mapping a different way.** The brief asks for 409s on
  unique violations. Users and tags have no routes, and tasks have no unique column, so no request to this
  API can make a P2002 or a P2003. I built a two-route Express app inside `tests/error-mapping.test.ts`
  with the real request-id middleware and the real `errorHandler`; one route inserts a second user with the
  default owner's email and the other inserts a task for a user id that does not exist. Both are real
  Prisma failures against `test.db`, and both come out as `409 CONFLICT` with the fixed message. Nothing is
  mocked. This is listed as a question for the EM in the backlog.
- **One mock, at a boundary.** Forcing a 500 needs something to fail. The test
  `jest.spyOn(taskService, 'list').mockRejectedValue(new Error(...))` replaces the service method, not the
  database, and `restoreMocks: true` in the Jest config puts it back after each test. The test asserts the
  body has no stack, no file path and no message from the error, and that the server log got the detail
  with the request id.
- **A test that takes the default owner away.** The 500 for "the database was not migrated" lives in
  `task-service.ts` (the owner lookup), and it was the only uncovered line in routes, handlers and services
  (97.64 percent of that file). A test now deletes the owner, posts, and expects a generic 500 and no row.
  `resetDb()` puts the owner back with an upsert, so a test that removes it on purpose cannot leave the next
  one without it, in any order.
- **Coverage is measured where the brief says, and I report the rest.** `collectCoverageFrom` is
  `src/routes`, `src/handlers` and `src/services` with a threshold of 80 for statements, branches,
  functions and lines, and `coverageProvider: 'v8'`. With all of `src/` (minus the generated client) the
  figures were lower: 96.16 percent statements and lines, 88.23 branches and 93.33 functions on the final
  run (the first measurement, before the missing-owner test, was 95.93 / 87.28 / 93.33), and the weakest
  files were `env.ts` (79 percent, the failure path of `parseEnv`) and `logger.ts` (74 percent, which does not
  log when `NODE_ENV=test`). Those are not routes or services, so they are not in the gate.
- **The workflow lives at the repo root.** The architecture document's layout draws `ci.yml` inside
  `task-api/`, but GitHub only reads `.github/workflows/` at the root of the repository, and this repo has
  `task-api/` as a subfolder. The workflow uses `defaults.run.working-directory: task-api`. Logged in the
  backlog.
- **CI runs the coverage script, not the bare one.** The step is `npm run test:coverage`, so the 80 percent
  threshold fails the job. I checked that a failed threshold fails the command: with the threshold raised to
  101 percent all 71 tests passed and `npm test -- --coverage` still exited 1 with `Coverage for lines
  (100%) does not meet "global" threshold (101%)`.
- **A code review changed the suite.** Two read-only reviewers found no critical or high defect. I fixed:
  the per-run database and the second guard (above); the "moves `updatedAt` forward" test, which used
  `>=` against the task's own seeded `updatedAt` and so could not fail, and now seeds both timestamps in the
  past and asserts the stored value is strictly later; a stale "five handlers" header comment and three
  headers that did not mention Module 9; an unused `makeUser` factory, which I removed (no test needed a
  second user); and `*.db-journal`, `*.db-wal` and `*.db-shm`, which `*.db`
  does not match, in `.gitignore`.
- **The M08 follow-up.** The comment on `satisfies` in `create` said a schema field "Prisma cannot store"
  makes the line stop compiling. M08's own guide says an extra field is not caught (excess properties are
  not checked on a spread). The comment now says a shared field with a type Prisma does not accept, or a
  required Prisma field the schema stopped producing, stops compiling, and that an extra field Prisma has no
  column for is not caught. The matching comment in `task.schemas.ts` had already been corrected: that file
  has no "stops compiling" text.

## How to build it (teach it to the next trainee)

### An isolated test database for integration tests

The goal is that a test can write, delete and count rows without caring what the test before it did, without
caring whether another test run is going on, and without any chance of touching the database you develop
against. Five pieces do it. I will use the one example that shows the reasoning: "after creating a task, the
list has exactly one task".

**1. Name a test database in a file that is not committed.** `.env.test` holds `DATABASE_URL=file:./test.db`,
a fake `API_KEY`, and `NODE_ENV=test`. A committed `.env.test.example` shows the shape. `test.db` is a name
to start from, not the file the tests use.

**2. Write down what counts as a test database, once.** `tests/helpers/test-database.ts` exports
`testDatabasePath(url)`: the URL must start with `file:`, a query string is cut off, the path is resolved,
and the file name must be `test...` ending in `.db`. Anything else throws. Resolving first matters:
`file:./dev.db?x=/test.db` and `file:./test.db/../dev.db` both end up refused. Both the setup and the cleanup
call it, so there is one rule and not two that can drift apart. It has its own tests.

**3. Give each run its own database and migrate it.** `tests/global-setup.ts` runs once in the Jest parent
process before any test file. It loads `.env.test` (`process.loadEnvFile` never replaces a variable that is
already set, so a value from the shell or CI wins), builds `test-<pid>-<time>.db` from the configured name
(the guard runs first, so a `DATABASE_URL` copied from `.env` is refused here), sets `DATABASE_URL` to it,
and runs `npx prisma@7 migrate deploy`. `migrate deploy` applies the committed migrations and creates none,
so the file gets the same tables and the same default owner as `dev.db`, which matters because `create` looks
that owner up. The test files start after this and inherit the variable, which is also why `config.ts` and
`db.ts` see it when they are imported. `tests/global-teardown.ts` deletes the file and its journal files.

**4. Empty the tables before every test, in an order the foreign keys allow, and check again first.**
`resetDb()` calls `testDatabasePath` before it deletes anything, then removes tasks, then tags, then users
other than the default owner, and puts the owner back if a test removed it. The order is the foreign keys: a
task points at its owner, so deleting users first would fail or cascade away the rows you meant to count.

**5. Make rows through a factory, not through the API you are testing.** `makeTask({ priority: 1 })` writes
straight to the database with Prisma, so a failing "get a task" test means "get" is broken, not that "create"
was. Anything a test depends on is passed in, so a test about ordering passes the dates and never relies on
the clock.

Then each file starts the same way: `beforeEach(resetDb)` and `afterAll(disconnectDb)`. The example test:

```ts
it('lists the task that was just created', async () => {
  await authed().post(TASKS).send({ title: 'Only one' });

  const res = await authed().get(TASKS);

  expect(res.body.meta.total).toBe(1);
});
```

It can only pass if the table was empty when it started, and only the cleanup makes that true. Without it
the answer depends on which file ran first. And it can only be trusted if the database is the run's own:
with a file shared between runs, a second run's cleanup can empty the table in the middle of the first run's
test.

## Concepts worth explaining

### Why integration tests here use a real database instead of mocks

A mock of the data layer checks that the code calls the mock the way the author imagined it would be called.
It cannot tell you that the query is wrong, that the migration does not create the column, that SQLite sorts
`NULL` first, that a unique constraint rejects the second row, or that Prisma throws a P2025 which the error
handler turns into a 404. All five of those were real behaviours in this API, and a mock would have returned
whatever I told it to. The two tests that exercise Prisma errors (`error-mapping.test.ts`) only mean
something because the failures are real. The price is speed and setup: a real database needs a migration
step and a cleanup, which is the guide above, and the suite here still runs in about 1.4 seconds. Mocking is
for the edges I do not control or cannot trigger, like forcing the service to throw so the 500 path can be
seen, and that is the only spy on application code in the suite (the other `spyOn` calls only silence `console.error`).

### Red, green, refactor, and the red that lies

The cycle is: write a test that fails for the right reason, make it pass with the least code, then improve
the code with the tests as a safety net. The part nobody warns about is "the right reason". My first red run
had three tests that passed with no route at all, so they were not specifying anything yet. A red test is
only useful if it fails because the behaviour is missing, so look at the failure message, not only at the
count. Refactor means the tests do not change: if I have to edit a test to keep it green, I am changing
behaviour, not structure.

## What tripped me up

- **`npm install` hung, and a stray process came back later.** The first install of Jest, Supertest and
  swc, started in the background, produced no output for over ten minutes (the last line in npm's log was a
  24-second tarball fetch). I killed it with `pkill`, started a second install in the foreground, and it finished
  in 3 minutes. Then the first install's own `npm` process exited with code 0 and a later command found
  `node_modules/jest` missing: two installs had been writing into the same `node_modules`. A plain
  `npm install` reconciled it ("added 237 packages in 1s"). Lesson: confirm the old process is really gone
  (`pgrep -fl npm`) before you start another install.
- **Node 22 failed every test at once.** A command chain with `&&` stopped before the `PATH` export, so
  `npm test` ran on Node 22.23.0 and all 63 tests failed in `resetDb` with a Prisma error whose text says
  `better_sqlite3.node was compiled against a different Node.js version using NODE_MODULE_VERSION 137. This
  version of Node.js requires ...`. Same code on Node 24.22.0: all green. The failure is in the first
  database call of every test, which looks like a test bug and is an environment bug.
- **I guessed an error message wrong.** My first full run was 62 of 63. The failing test expected
  `Route not found: GET /nothing-here`, and the real message is `Route not found: GET /api/v1/nothing-here`,
  because the 404 handler is mounted on the app, so `req.path` is the full path. The test was wrong, not the
  API, and I fixed the test. Read the code before asserting an exact string.
- **A test that passes is not always a test that works.** See the red-run story above. A test that cannot
  fail is worse than no test, because it reports confidence.
- **Two runs on one `test.db` failed in different places each time.** While two reviewers ran the suite at
  the same time, 1 to 12 tests failed per run, different ones each run, one with `Received: 4` where the test
  expected nothing stored. The code was fine: the second run's cleanup emptied the tables during the first
  run's test. The same suite passed 15 of 15 on a private file. This is why each run now has its own database.
- **The PR showed one check called `test`.** I asked why CI "only has test". The workflow ran all four steps
  (npm ci, generate, type-check, test) but they were four steps of one job named `test`, and the checks list
  shows one line per job. Anyone reading the PR would think only the tests ran. I renamed the job to
  `Install, generate, type-check, test` and kept it as one job. First I checked that nothing depended on the old
  name: `main` had no branch protection (the API answers 404) and the repo has no rulesets, so no required
  status check named `test` existed.
- **No ordering dependency showed up.** The brief expects `--randomize` to expose something. In my suite it
  did not: seeds 1, 2, 3, 42, 777, 98765, 20261021, one random seed (1019317433), and 11, 12 and 13 after the
  last test was added all gave every test passing. I checked that the flag really reorders by reading the
  test order for `tasks.delete.test.ts` under seed 1 and seed 2 (different order). The reasons it stayed
  clean are probably the cleanup in `beforeEach`, explicit dates in the sort tests, and a factory counter
  that only makes names unique. I cannot claim more than that, and I cannot claim the suite is free of
  ordering dependencies, only that these seeds found none.
- **Mocking: `jest.mock` does nothing in ESM and does not say so.** I wrote `jest.mock('../../src/services/
  task-service.ts', factory)` after a static import, ran it, and the real `list` function ran; no error,
  no warning. `jest.unstable_mockModule` with a dynamic `await import(...)` afterwards returned the mock.
  The usual explanation is that `jest.mock` relies on being moved above the `require` calls, and ES module
  imports are resolved before any code runs, so there is nothing to move it above. I did not test that
  explanation, only the two outcomes.
- **The 409 gap** (see Other decisions): the brief asks for 409 tests on unique violations and the API has
  no way to cause one.

## Checkpoint evidence

All evidence below is from my own runs on Node 24.22.0 in `task-api/`.

- **At least 25 integration tests covering all endpoints and every error path.** 88 tests in 9 files, 71 of
  them integration tests of the API: create 17, list 17, middleware 14, update 7, complete 7, delete 4, read
  3, error mapping 2, and 17 for the database guards (`test-database.test.ts`). Every
  endpoint (`GET /health`, list, create, get, update, delete, complete) has a happy path and its error
  paths: 400 with `details`, 401, 404, 413, malformed JSON, 409 (through the error handler), and a 500.
- **Mutation checks, to show the tests can fail.** I broke the code three ways and restored it each time.
  An error handler that returns the stack failed 2 tests. Removing the `id` tie-break from the list failed 1
  test. Making `complete` set `in_progress` failed 4 tests. With cleanup removed, 20 of 63 failed. With the
  database name check disabled, 9 of the 17 guard tests failed.
- **The suite passes with `--randomize`.** `npm run test:random` (seed 1019317433) gave 63 of 63 before
  `complete`, and seeds 11, 12 and 13 gave 71 of 71 after. After the review fixes (88 tests), seeds 1, 2, 3, 5,
  42, 777, 20261021 and one automatic seed (-1166006607) all gave 88 of 88.
- **Runs at the same time no longer collide.** Three runs started together with seeds 1, 2 and 3 all passed
  88 of 88, and `ls *.db` afterwards showed only `dev.db`.
- **80 percent or more coverage on routes and services.** `npm run test:coverage`:
  `All files | 100 | 100 | 100 | 100` over `src/handlers`, `src/routes` and `src/services` (statements,
  branches, functions, lines), re-measured after the review fixes. Before the missing-owner test it was
  98.97 / 95.45 / 100 / 98.97, with `task-service.ts` lines 57-58 uncovered.
- **`PATCH /tasks/:id/complete` added test-first.** The saved red run: `Tests: 6 failed, 1 passed, 7 total`,
  with `Expected: 200 Received: 404` on the happy paths and `Expected: 400 Received: 404` for a bad id. Green:
  70 of 70. Refactor: `complete` delegates to `update`, 70 of 70 again. In `git log` they are three commits,
  `fc95e0a` (the failing tests), `e5c2840` (the route) and `2e22647` (the refactor). I checked each one in a
  temporary worktree: at `fc95e0a` the suite is `Tests: 6 failed, 82 passed, 88 total` (the complete file is
  6 failed and 1 passed), and at `e5c2840` and `2e22647` all 88 pass. The commits were put together from the
  finished working tree, in the order the work was done, so their dates show when I committed, not when each
  file was first written.
- **Bruno still passes and covers the new route.** Against a server on a throwaway
  database made by hand (`DATABASE_URL=file:./test-bruno.db npm run db:deploy`, removed afterwards):
  `53 (53 Passed)`, tests `66/66`, assertions `104/104` (the M08 baseline was 47, 63 and 91; the new
  `Complete/` folder and `Errors/29`, `30` add 6, 3 and 13).
- **Green CI on a pull request.** [PR #9](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/9) has a green check. The first run,
  [39611518300](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/actions/runs/39611518300), at `73a27af`, was `ubuntu-latest` with Node v24.22.0 taken from `.nvmrc`: the steps
  Install (953 packages added), Generate the Prisma client, Type-check and Test all succeeded, with 9 suites,
  88 tests and 100 percent coverage on the measured files, in 36 seconds. After I renamed the job, run
  [39612459807](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/actions/runs/39612459807) at `0ccf4ed` passed in 34 seconds under the check name
  `Install, generate, type-check, test`. Before the first push I had rehearsed the same steps in a clean copy of
  the files git tracks (no `.env`, `.env.test`, `dev.db`, `node_modules` or generated client): same result.
  The rehearsal and the runner agreed, so I found nothing that differs between them. The push to `main` after the
  merge ran the same workflow ([run 39614768472](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/actions/runs/39614768472),
  merge commit `ed7d65b`): its one job, `Install, generate, type-check, test`, finished with every step
  successful, and the run's own status is `completed`, conclusion `success`. (A few minutes after the merge the
  API still showed the run as `in_progress` although its only job had finished. It settled by itself.)
- **Merged into `main`.** [PR #9](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/9) was merged on 2026-10-21 at 14:12:06 UTC with merge commit `ed7d65b`.
  The history on `main`, from `git log --oneline --graph`:

  ```text
  *   ed7d65b Merge pull request #9 from kiannaalexandra-gragg/feat/module-09/integration-testing
  |\
  | * 0ccf4ed ci(m09): name the CI job for everything it runs
  | * 73a27af docs(m09): describe Task API v4 and add the CI badge
  | * 415cff0 ci(m09): run typecheck and the tests on every pull request
  | * f10fdd1 test(m09): pin the complete route in Bruno
  | * 2e22647 refactor(m09): complete a task through update
  | * e5c2840 feat(m09): complete a task
  | * fc95e0a test(m09): add failing tests for completing a task
  | * 732ae8f test(m09): cover every endpoint and error path with integration tests
  | * aa8ab36 test(m09): add an isolated test database with two guards
  | * 581fc13 chore(m09): add Jest 30, Supertest 7 and @swc/jest
  | * d5325fc docs(m08): correct the satisfies comment in the task service
  |/
  *   4deec36 Merge pull request #8 from kiannaalexandra-gragg/feat/module-08/zod-validation
  ```

  I then cloned the merged `main` fresh and ran the workflow's steps with its environment variables on
  Node 24.22.0, with no `.env`, no `.env.test` and no `dev.db`: `npm ci`, `npm run db:generate`,
  `npm run typecheck` (exit 0), `npm run test:coverage` (9 suites, 88 tests, 100 percent on routes, handlers
  and services), and `npm test -- --randomize --seed=77` (88 of 88, no database file left behind).
- **`dev.db` is unchanged.** Its MD5 was `6367cdfaf1b61235d07c3aa26d0c4db2` before and after every run above,
  including the run with `DATABASE_URL=file:./dev.db` (stopped with `Refusing to touch the database`) and the
  mutation that disabled the name check.
- **The training material's code runs.** I extracted the guide's code blocks (config, guard, setup, teardown,
  helpers, the read test, the guard tests including the `resetDb` one, the 409 test) into a clean copy of the
  project: it typechecks and the 21 tests in it pass.
- **The M08 follow-up fix** is commit `d5325fc`, on its own, the first commit of [PR #9](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/9), and it is on `main` now.

## What I'd do differently

- **Start with the red run's failure messages.** I would write the first failing test, read why it fails, and
  only then write the rest. I lost no time to it, but I only noticed the three tests that passed for the
  wrong reason because I read the output line by line.
- **Run the environment check first, every time.** `node -v` as the first line of every command that
  touches the database would have saved the Node 22 run. A startup check in the project (an `engines` value
  that `npm` enforces with `engine-strict`) is still an open question from M08.
- **Never start a second `npm install`.** Wait for the first, or prove it is gone.
- **Decide the 409 question before building.** I would ask the EM, before writing `error-mapping.test.ts`,
  whether a test-only app is the intended way to cover those two error codes, or whether the brief expects an
  endpoint the API does not have yet.
- **Commit as I go, not at the end.** I ran red, green and refactor in that order, but I only committed
  afterwards, from the finished working tree. The three commits show the right order and the suite results I
  checked at each one, and they cannot show when I wrote each file. Committing the failing tests before
  touching `src` is the only way to make that visible in the history without anyone taking my word for it.
- **Build the database per run from the start.** A fixed `test.db` worked until two runs overlapped, and it
  needed a code review plus two parallel reviewers to show it.
- **For the capstone:** reuse this shape (`tests/global-setup.ts` and `global-teardown.ts`, `tests/helpers/`,
  `resetDb` in `beforeEach`, the guard on the database name checked twice, a Supertest agent, `restoreMocks`) and write the workflow
  file first, so CI is green from the first PR and not discovered at the end.
