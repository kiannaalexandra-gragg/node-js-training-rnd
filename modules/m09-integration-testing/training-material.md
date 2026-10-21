# M09 — Integration Testing with Jest + Supertest: Training Material

**Week 4 · Days 1-2 · 8 hours**

A "module" in this program means a training stage. When this guide says "ES module" it means a JavaScript
module file. The guide's own steps are called "stages" too, so "the stage" means this module and "Stage 4"
means a step of the guided path below.

## Builds on

M08 (Task API v3: Zod validation, the typed `validate` middleware, the central error handler, Prisma 7 on
SQLite) and M07 before it. You test that same `task-api/` project in place and ship it as **v4**. Keep your
M08 version on `main`, a clean working tree, and your Bruno collection on hand.

## 1. What you'll build and why

You will write a test suite that proves the Task API works end to end, against a real database, and wire it
into CI. You finish with **Task API v4**:

- **25 or more integration tests** that call the real exported Express app and a real SQLite test database,
  covering every endpoint and every error path,
- an **isolated test database**: a separate test database for every run (made from `test.db`), migrated once, cleaned before every test, and refused unless it is really a test database,
- tests for **happy paths, validation failures (400 with `details`), 404, 409, malformed JSON, pagination,
  filtering and sorting, and the middleware** (request-ID header, 404 handler, no stack trace in an error),
- a new endpoint, **`PATCH /api/v1/tasks/:id/complete`**, added **test-first** (red, green, refactor),
- **80% or more coverage** on routes and services, and a suite that passes in **random order**
  (`--randomize`),
- a **GitHub Actions workflow** that runs `npm ci`, `prisma generate`, the type check and the tests, with a
  status badge in the README, and **green CI on a pull request**.

Why this stage matters: so far you checked the API by hand (Bruno, `curl`). That proves it worked once, on
your machine, for the requests you remembered to send. A test suite proves it every time, for every request
you wrote down, and tells you which one broke. Integration tests here use a real database and not mocks,
because the bugs worth catching live in the seams (a query, a migration, a constraint, an error mapping), and
a mock cannot have a bug in a seam it replaces.

### Where the code goes

All code lives in your build repo, in `task-api/`. v4 is a change to that folder, not a new one. One file,
the CI workflow, goes outside it, at the root of the repo:

```text
.github/workflows/ci.yml        NEW   GitHub only reads workflows from the root of the repository
task-api/
├── jest.config.js              NEW
├── .env.test.example           NEW   committed, fake values; your real .env.test is git-ignored
├── tests/
│   ├── global-setup.ts         NEW
│   ├── global-teardown.ts      NEW
│   ├── helpers/
│   │   ├── test-database.ts    NEW
│   │   ├── db.ts               NEW
│   │   ├── factories.ts        NEW
│   │   └── api.ts              NEW
│   └── *.test.ts               NEW   one file per area
├── src/                        a route, a handler, a service method and a schema for /complete
├── package.json                new dev dependencies and test scripts
└── tsconfig.json               "tests" added to include
```

### How to use this guide

Do the stages in order. Each one opens with the part of the brief it covers, states a goal, gives the steps
and the reason for each, and ends with a **Done when** checklist you can verify yourself. Commit at the end
of each stage with a Conventional Commit message (`test(m09): ...`, `feat(m09): ...`, `chore(m09): ...`).
The code shown is the shape that worked: your test names and sample data can differ, but keep the structure.

## 2. Before you start

**Prerequisites.** The M08 Task API on `main` with a clean tree, and its Bruno collection passing.

**Pinned stack.** Node.js 24 LTS (the `.nvmrc` in the repo root), TypeScript 7.0 (types are checked by
`tsc --noEmit`), Express 5, Prisma 7.x (always `npx prisma@7`, never an unversioned `prisma`), SQLite only,
Zod 4, **Jest 30.x**, **Supertest 7.x** and **`@swc/jest`** as the transform. Free tools only: no Docker, no
paid services.

**Import extensions.** The project imports files with their `.ts` extension (`import { app } from
'../src/app.ts'`). Keep that in the tests. Jest finds a file by its real name, so no `moduleNameMapper` is
needed.

**Check Node first.** Run `nvm use` (or put Node 24 first on your `PATH`) and confirm `node -v` prints 24.x
before every session. `better-sqlite3` is a native addon built for one Node version. On Node 22 every test
that touches the database fails at once, with a Prisma error whose text says `better_sqlite3.node was
compiled against a different Node.js version`. It looks like a test bug and is an environment bug.

**Toolchain fallbacks.** `@swc/jest` is the default transform. If it will not run the Prisma-generated
client in ESM mode on your machine, apply these fallbacks **in this order, without mixing them and without
skipping ahead**, and note which one you needed: 1) `ts-jest`, with its documented TypeScript 7 side-by-side
setup; 2) pin TypeScript 6 for the test toolchain only; 3) compile the project as CommonJS by dropping
`"type": "module"`. Stage 2 is where you find out whether you need any.

**Never do these.** Run the tests against `dev.db`. Commit `.env.test`, a `test*.db` file or `coverage/`. Mock the
database. Start a second `npm install` while one is still running.

## 3. Guided path

### Stage 1 — Branch, baseline, and a clean slate

*Traces to the brief:* Setup (the branch → PR → merge habit); Lab step 1 (starting point).

**Goal:** know the API is healthy before you add anything, so a later failure is yours.

1. In the repo root: `nvm use`, then `node -v` (24.x).
2. `git checkout main && git pull && git checkout -b feat/module-09/integration-testing`.
3. In `task-api/`: `npm ci`, `npm run db:generate`, `npm run typecheck` (exit 0).
4. Write down the checksum of your development database, so Stage 3 can prove the tests never touched it:
   `md5 dev.db` on macOS, `md5sum dev.db` on Linux.
5. Keep the deliverable in front of you: 25 or more integration tests covering every endpoint and every
   error path, a passing GitHub Actions workflow, and 80% or more coverage on routes and services.

**Why a baseline:** if the suite you are about to write is red, you must know whether the code or the tests
are at fault. A green typecheck and a recorded checksum remove two suspects.

**Done when:**
- [ ] `node -v` prints a 24.x version in the shell you will use.
- [ ] You are on a branch named `feat/module-09/integration-testing`, and `git status` is clean.
- [ ] `npm run typecheck` exits 0.
- [ ] You have written down the checksum of `dev.db`.

### Stage 2 — Install the toolchain and prove it works before writing tests

*Traces to the brief:* Setup (Jest, Supertest, `@swc/jest`; "confirm the transform works in ESM mode before
writing any real tests"); the toolchain fallback order.

**Goal:** a one-test smoke run that imports your real `app` and touches the real Prisma client under Jest.

1. Install, once, and wait for it to finish (a second install running at the same time can leave
   `node_modules` half-written):

   ```bash
   cd task-api
   npm install --save-dev jest@30 @jest/globals@30 supertest@7 @types/supertest @swc/core @swc/jest
   ```

2. Add the scripts to `package.json`. The `test` script is exactly this, so it works on every operating
   system:

   ```json
   "test": "node --experimental-vm-modules node_modules/jest/bin/jest.js --runInBand",
   "test:coverage": "npm test -- --coverage",
   "test:random": "npm test -- --randomize"
   ```

   `--experimental-vm-modules` is what lets Jest load ES modules. `--runInBand` runs the test files one
   after another in one process, so two files never write to the database at once (Stage 3 says why).

3. Add `"tests"` to `include` in `tsconfig.json`, so `npm run typecheck` checks the tests too:
   `"include": ["src", "tests", "prisma/seed.ts", "prisma7.config.ts", "scripts"]`.

4. Create `jest.config.js`. The project is `"type": "module"`, so this is an ES module:

   ```js
   // Module 9 output: the Jest configuration for the integration tests.
   export default {
     testEnvironment: 'node',
     roots: ['<rootDir>/tests'],
     testMatch: ['**/*.test.ts'],

     // Jest treats .js files as ESM because of "type": "module". Tell it .ts files are ESM too.
     extensionsToTreatAsEsm: ['.ts'],

     // @swc/jest strips the types and leaves import and export alone. It does not check types:
     // `npm run typecheck` does that, locally and in CI.
     transform: {
       '^.+\\.ts$': [
         '@swc/jest',
         {
           jsc: { parser: { syntax: 'typescript' }, target: 'es2024' },
           module: { type: 'es6' },
         },
       ],
     },

     // Fresh spies for every test, so no test inherits another's.
     restoreMocks: true,
   };
   ```

   (You add `globalSetup`, `globalTeardown` and the coverage settings in Stages 3 and 10.)

5. Write a throwaway `tests/smoke.test.ts`:

   ```ts
   import { describe, expect, it } from '@jest/globals';
   import request from 'supertest';
   import { app } from '../src/app.ts';
   import { prisma } from '../src/db.ts';

   describe('smoke', () => {
     it('serves /health and reads the database', async () => {
       const res = await request(app).get('/health');
       expect(res.status).toBe(200);
       expect(await prisma.user.count()).toBeGreaterThanOrEqual(1);
       await prisma.$disconnect();
     });
   });
   ```

   It needs environment variables the app reads at import (`API_KEY`, `DATABASE_URL`), so it cannot pass
   yet. Stage 3 supplies them, and you run it at the end of that stage.

**Why describe, it, expect come from `@jest/globals`:** the globals Jest injects have no TypeScript types
unless you add more packages. Importing them is explicit, typed and works the same in ESM.

**Why `request(app)` and no port:** `app.ts` builds and exports the app and never calls `listen` (`server.ts`
does that). Supertest starts the app on a temporary port for each request and closes it, so tests do not
fight over port 3000.

**Done when:**
- [ ] `npm ls jest supertest @swc/jest` shows Jest 30.x, Supertest 7.x and `@swc/jest`.
- [ ] `npm run typecheck` exits 0 with `tests/` included.
- [ ] `jest.config.js` exists and `npm test` starts Jest (the smoke test is allowed to fail until Stage 3).
- [ ] You can say in one sentence why `jest.config.js` has `extensionsToTreatAsEsm`.

### Stage 3 — The isolated test database

*Traces to the brief:* Scope, "Test database strategy" (a separate `test.db` via `.env.test`, a Jest
`globalSetup` that runs `prisma migrate deploy`, `--runInBand`); Lab step 1.

**Goal:** tests that can only ever touch a test database, which is created, migrated and removed
automatically, and which no other test run can touch.

The reasoning, in order. Tests write and delete constantly, so they need their own database. That database
needs the same tables and the default owner as the real one. A mistake in configuration must not be able to
point the tests, which delete every row, at your development data. And two test runs at the same time (two
terminals, an editor's test runner) must not share a file, or each run's cleanup wipes the other's rows and
the tests fail in different places each time.

1. **The settings.** Create `.env.test.example` (committed, fake values):

   ```text
   # Settings for the tests: copy this file to .env.test. Never commit .env.test.
   API_KEY=test-api-key
   DATABASE_URL=file:./test.db
   NODE_ENV=test
   LOG_QUERIES=false
   ```

   Then `cp .env.test.example .env.test`. Confirm Git ignores the real one:
   `git check-ignore -v .env.test` prints a rule. The `.gitignore` must also cover `*.db`, and the journal
   files SQLite leaves beside a database while it writes or after a crash: `*.db-journal`, `*.db-wal` and
   `*.db-shm`. `*.db` does not match those. Add them if they are missing.

   `test.db` here is a **name to start from**, not the file the tests will use. Step 3 turns it into a
   different file for every run.

2. **The rule for what counts as a test database.** One small file holds it, so the setup and the cleanup use
   the same check. `tests/helpers/test-database.ts`:

   ```ts
   import { basename, resolve } from 'node:path';

   // A test run deletes every row, so the database must be checked by what it really is. The check is on
   // the resolved file path: it must be a `file:` URL, a query string is cut off first (otherwise
   // `file:./dev.db?x=/test.db` would look like a file called test.db), the path is resolved before its
   // name is read (so `file:./test.db/../dev.db` is dev.db), and the name must be `test...` and end in `.db`.
   // Anything unexpected throws: when in doubt, do not wipe the database.
   export function testDatabasePath(url: string | undefined): string {
     if (url === undefined || !url.startsWith('file:')) {
       throw new Error(`Refusing to touch the database: DATABASE_URL must be a file: URL such as file:./test.db, got "${String(url)}"`);
     }
     const path = resolve(url.slice('file:'.length).split('?')[0] ?? '');
     if (!/^test[\w.-]*\.db$/.test(basename(path))) {
       throw new Error(`Refusing to touch the database: ${path} is not a test database (its name must start with "test" and end in ".db")`);
     }
     return path;
   }

   // The database one test run uses: the configured name with the process id and the time added, so
   // test.db becomes test-48213-1761043200000.db. Two runs at the same time get two files.
   export function perRunDatabaseUrl(configuredUrl: string | undefined, pid: number, now: number): string {
     testDatabasePath(configuredUrl);
     const [path = '', query] = (configuredUrl as string).split('?');
     const withRun = path.replace(/\.db$/, `-${pid}-${now}.db`);
     return query === undefined ? withRun : `${withRun}?${query}`;
   }
   ```

3. **Build the run's database once.** `tests/global-setup.ts`:

   ```ts
   import { execFileSync } from 'node:child_process';
   import { existsSync } from 'node:fs';
   import { perRunDatabaseUrl } from './helpers/test-database.ts';

   // Runs once, in the Jest parent process, so it loads the test settings itself.
   export default function globalSetup(): void {
     if (existsSync('.env.test')) process.loadEnvFile('.env.test');

     // The configured name is checked first (a DATABASE_URL copied from .env is refused here), then this
     // run gets its own file. Test files start after this, so they inherit the new value.
     process.env.DATABASE_URL = perRunDatabaseUrl(process.env.DATABASE_URL, process.pid, Date.now());

     // migrate deploy applies the committed migrations and creates none. It creates the file, with the
     // same tables and the same default owner as dev.db.
     execFileSync('npx', ['prisma@7', 'migrate', 'deploy'], { env: process.env, stdio: 'pipe' });
   }
   ```

   `process.loadEnvFile` never replaces a variable that is already set, so a value from the shell or from CI
   wins over `.env.test`. `prisma7.config.ts` also loads `.env` (your development settings) and does not
   replace a variable that is already set either, so the per-run `DATABASE_URL` is the one `migrate deploy` uses.

4. **Delete it when the run ends.** `tests/global-teardown.ts`:

   ```ts
   import { rmSync } from 'node:fs';
   import { testDatabasePath } from './helpers/test-database.ts';

   // Runs once after the last test file, in the same process as globalSetup, and also when tests fail.
   // It cannot run if the process is killed, so a crashed run can leave a file behind: it is git-ignored
   // and safe to delete by hand.
   export default function globalTeardown(): void {
     const path = testDatabasePath(process.env.DATABASE_URL);
     for (const suffix of ['', '-journal', '-wal', '-shm']) rmSync(`${path}${suffix}`, { force: true });
   }
   ```

   Register both in `jest.config.js`, after `transform`:

   ```js
   // Runs once, in the Jest parent process, before any test file: loads .env.test, gives the run its own
   // database and migrates it. Test files start after it and inherit what it set.
   globalSetup: '<rootDir>/tests/global-setup.ts',
   // Runs once, after the last test file: deletes this run's database.
   globalTeardown: '<rootDir>/tests/global-teardown.ts',
   ```

   There is no separate "setup file" for the environment. `globalSetup` runs before Jest starts the test files,
   and they inherit the variables it set, `API_KEY` and `DATABASE_URL` included. That matters because
   `config.ts` and `db.ts` read the environment when they are imported.

5. **Why `--runInBand` is still there.** Each run has its own database, but inside one run every test file
   shares it, and SQLite lets one connection write at a time. The `test` script runs the files one after
   another, so they never write at once. (The Concepts section compares the alternative, a database per
   worker.)

6. **Test the rule itself.** The guard is the one thing that protects your data, so it gets tests.
   `tests/test-database.test.ts`:

   ```ts
   import { describe, expect, it } from '@jest/globals';
   import { resolve } from 'node:path';
   import { perRunDatabaseUrl, testDatabasePath } from './helpers/test-database.ts';

   describe('testDatabasePath', () => {
     it.each([['file:./test.db'], ['file:./test-48213-1761043200000.db'], ['file:./test.db?connection_limit=1']])(
       'accepts %s',
       (url) => {
         expect(testDatabasePath(url)).toMatch(/test[\w.-]*\.db$/);
       },
     );

     it.each([
       ['the development database', 'file:./dev.db'],
       ['a name that only contains test', 'file:./mytest.db'],
       ['a name that does not end in .db', 'file:./test.sqlite'],
       ['a query string that looks like a path', 'file:./dev.db?x=/test.db'],
       ['a path that walks back to dev.db', 'file:./test.db/../dev.db'],
       ['a folder called test around dev.db', 'file:./test/dev.db'],
       ['a server URL', 'postgresql://localhost/test'],
       ['an empty path', 'file:'],
       ['nothing at all', undefined],
     ])('refuses %s', (_name, url) => {
       expect(() => testDatabasePath(url)).toThrow(/Refusing to touch the database/);
     });

     it('resolves the path before it reads the name', () => {
       expect(testDatabasePath('file:./test.db')).toBe(resolve('test.db'));
     });
   });

   describe('perRunDatabaseUrl', () => {
     it('adds the process id and the time, and keeps the folder and the extension', () => {
       expect(perRunDatabaseUrl('file:./test.db', 48213, 1761043200000)).toBe('file:./test-48213-1761043200000.db');
     });

     it('refuses a database that is not a test database, before it builds a name', () => {
       expect(() => perRunDatabaseUrl('file:./dev.db', 1, 1)).toThrow(/Refusing to touch the database/);
     });
   });
   ```

   (Stage 4 adds one more test to this file, for `resetDb`.)

7. **Run the smoke test.** `npm test`. It should pass. While it runs, a file like
   `test-<number>-<number>.db` exists in `task-api/`, and when the run ends it is gone: `ls *.db` shows only
   `dev.db`.

8. **Prove `dev.db` was not touched.** Run the checksum command from Stage 1 again and compare.

9. **Prove the guard works end to end.** Run `DATABASE_URL=file:./dev.db npm test`. Expect the `Refusing to
   touch the database` error and no tests run. (A variable set in the shell wins over `.env.test`, which is
   exactly the situation the guard protects against.)

**Done when:**
- [ ] `npm test` passes, and afterwards `ls *.db` shows no `test-` file: the run's database was deleted.
- [ ] `git status` does not list `.env.test`, and `git check-ignore -v test-1-2.db test-1-2.db-journal` shows
      a rule for each.
- [ ] The checksum of `dev.db` is the same as at the end of Stage 1.
- [ ] `DATABASE_URL=file:./dev.db npm test` stops with the `Refusing to touch the database` error.
- [ ] The guard's tests pass, and you can say why `file:./dev.db?x=/test.db` is refused.
- [ ] You can explain why each run has its own database file, and why `--runInBand` is still used.

### Stage 4 — Cleanup, factories and the first real test

*Traces to the brief:* Scope, "Test database strategy" (clean tables in `beforeEach` in foreign-key-safe
order, factories and seed helpers, `afterAll(() => prisma.$disconnect())`); Lab step 1.

**Goal:** every test starts from the same empty state, and builds the rows it needs without using the API
it is testing.

1. **`tests/helpers/db.ts`:**

   ```ts
   import { prisma } from '../../src/db.ts';
   import { DEFAULT_OWNER_EMAIL } from '../../src/services/task-service.ts';
   import { testDatabasePath } from './test-database.ts';

   // Run in beforeEach. The order is the foreign keys: a task points at its owner, so tasks go first,
   // then tags, then users. Deleting a task or a tag also removes its rows in the tag join table.
   // The default owner stays, because `create` needs it. The upsert puts it back if a test deleted it.
   export async function resetDb(): Promise<void> {
     // The second guard. The setup already checked, but a wipe must not depend on the setup having run
     // (another Jest config, or this helper called on its own). Nothing is deleted unless the database
     // this process uses is a test database.
     testDatabasePath(process.env.DATABASE_URL);

     await prisma.task.deleteMany();
     await prisma.tag.deleteMany();
     await prisma.user.deleteMany({ where: { email: { not: DEFAULT_OWNER_EMAIL } } });
     await prisma.user.upsert({
       where: { email: DEFAULT_OWNER_EMAIL },
       update: {},
       create: { email: DEFAULT_OWNER_EMAIL, name: 'Default Owner' },
     });
   }

   // Run in afterAll: the client holds an open connection to the test database.
   export async function disconnectDb(): Promise<void> {
     await prisma.$disconnect();
   }
   ```

2. **`tests/helpers/factories.ts`:**

   ```ts
   import { prisma } from '../../src/db.ts';
   import type { Prisma, Task, User } from '../../src/generated/prisma/client.ts';
   import { DEFAULT_OWNER_EMAIL } from '../../src/services/task-service.ts';

   let counter = 0;

   export async function defaultOwner(): Promise<User> {
     return prisma.user.findUniqueOrThrow({ where: { email: DEFAULT_OWNER_EMAIL } });
   }

   // Anything a test depends on is passed in, so the test shows it. A sort test passes the dates.
   export async function makeTask(overrides: Partial<Prisma.TaskUncheckedCreateInput> = {}): Promise<Task> {
     counter += 1;
     const owner = await defaultOwner();
     return prisma.task.create({ data: { title: `Task ${counter}`, userId: owner.id, ...overrides } });
   }

   // SQLite has one writer, so these are made one after another, not in parallel.
   export async function makeTasks(
     count: number,
     build: (i: number) => Partial<Prisma.TaskUncheckedCreateInput> = () => ({}),
   ): Promise<Task[]> {
     const tasks: Task[] = [];
     for (let i = 0; i < count; i += 1) tasks.push(await makeTask(build(i)));
     return tasks;
   }
   ```

3. **`tests/helpers/api.ts`:**

   ```ts
   import request from 'supertest';
   import { app } from '../../src/app.ts';

   export const TASKS = '/api/v1/tasks';

   // An agent remembers what it is told to send, for every request it makes. This API has no cookies,
   // so the agent carries the API key: no test repeats the header.
   export function authed() {
     return request.agent(app).set('X-API-Key', process.env.API_KEY ?? '');
   }

   // A request with no key, for the tests about authentication.
   export function anonymous() {
     return request(app);
   }
   ```

4. **The first real test**, `tests/tasks.read.test.ts`. Delete `tests/smoke.test.ts`; this replaces it:

   ```ts
   import { afterAll, beforeEach, describe, expect, it } from '@jest/globals';
   import { authed, TASKS } from './helpers/api.ts';
   import { disconnectDb, resetDb } from './helpers/db.ts';
   import { makeTask } from './helpers/factories.ts';

   beforeEach(async () => {
     await resetDb();
   });
   afterAll(async () => {
     await disconnectDb();
   });

   describe('GET /api/v1/tasks/:id', () => {
     it('returns the task', async () => {
       // Arrange
       const task = await makeTask({ title: 'Find me', priority: 1 });
       // Act
       const res = await authed().get(`${TASKS}/${task.id}`);
       // Assert
       expect(res.status).toBe(200);
       expect(res.body).toMatchObject({ id: task.id, title: 'Find me', priority: 1, status: 'todo' });
     });

     it('answers 404 NOT_FOUND for an id that does not exist', async () => {
       const res = await authed().get(`${TASKS}/00000000-0000-4000-8000-000000000000`);
       expect(res.status).toBe(404);
       expect(res.body.error.code).toBe('NOT_FOUND');
     });

     it('answers 400 for an id that is not a UUID', async () => {
       const res = await authed().get(`${TASKS}/not-a-uuid`);
       expect(res.status).toBe(400);
       expect(res.body.error.details).toContainEqual({ path: 'id', message: 'Must be a valid UUID' });
     });
   });
   ```

   Every test follows **Arrange-Act-Assert**: build the rows, make one request, check the result.
   `toMatchObject` checks the fields you name and ignores the rest (the generated `createdAt`, for example).

5. **Test the second guard.** Add one more test to `tests/test-database.test.ts`: when `DATABASE_URL`
   does not name a test database, `resetDb()` must throw before it deletes anything. Add the imports
   `afterAll`, `afterEach` (from `@jest/globals`), `prisma` (`../src/db.ts`), `disconnectDb` and `resetDb`
   (`./helpers/db.ts`) and `makeTask` (`./helpers/factories.ts`), then:

   ```ts
   afterAll(async () => {
     await disconnectDb();
   });

   describe('resetDb refuses a database that is not a test database', () => {
     const real = process.env.DATABASE_URL;
     afterEach(() => {
       process.env.DATABASE_URL = real;
     });

     it('throws before it deletes anything', async () => {
       const task = await makeTask({ title: 'must survive' });
       process.env.DATABASE_URL = 'file:./dev.db';

       await expect(resetDb()).rejects.toThrow(/Refusing to touch the database/);

       expect(await prisma.task.findUnique({ where: { id: task.id } })).not.toBeNull();
       process.env.DATABASE_URL = real;
       await resetDb();
     });
   });
   ```

   Changing `process.env.DATABASE_URL` after the app was imported does not move Prisma to another file: the
   client keeps the URL it was built with. The test only proves that the check runs first.

6. **Prove the cleanup matters.** This needs more than one test file, so do it at the end of Stage 5, once
   the create, update and delete files exist. Temporarily make `resetDb()` do nothing (comment out its
   body) and run `npm test`. A number of tests in other files will fail, even though the code under test is
   fine: they find rows left by an earlier file. Put the body back. You have just seen what a flaky test is
   made of: the result depends on what ran before.

**Done when:**
- [ ] `tests/helpers/db.ts`, `factories.ts` and `api.ts` exist, and `npm run typecheck` exits 0.
- [ ] `npm test` runs `tasks.read.test.ts` with 3 passing tests.
- [ ] The `resetDb` guard test passes: with a `DATABASE_URL` that is not a test database, `resetDb()` throws
      and the row you made is still there.
- [ ] You can answer: why are tasks deleted before users in `resetDb()`?

### Stage 5 — Happy paths for every endpoint

*Traces to the brief:* Lab step 2 (happy paths); Build, "Write tests for the happy paths across every
endpoint".

**Goal:** one passing test file per endpoint that proves the success case, including what is stored.

Each file starts like `tasks.read.test.ts`: the `@jest/globals` import, the helper imports,
`beforeEach(resetDb)` and `afterAll(disconnectDb)`. A file also imports what its own tests use: the create
file below needs `prisma` (`../src/db.ts`) and `defaultOwner` (`./helpers/factories.ts`) as well. Write these,
each as its own `it`:

| File | Happy-path tests |
| --- | --- |
| `tasks.create.test.ts` | 201 and the shape (`title`, `description: null`, `status: 'todo'`, `priority: 2`, `dueDate: null`, `userId` is the default owner's id) **and the row exists** (`prisma.task.findUnique`); every field the client sets is kept; a date-only `dueDate` comes back as midnight UTC (`2026-12-31T00:00:00.000Z`); the title is trimmed; server-owned fields (`id`, `userId`, `createdAt`) in the body are ignored |
| `tasks.update.test.ts` | PATCH changes only the fields sent; `updatedAt` moves forward and `createdAt` does not; `dueDate: null` clears the date |
| `tasks.delete.test.ts` | 204 with an empty `res.text`, and the row is gone; the other tasks are untouched |
| `GET /health` (in `middleware.test.ts`, Stage 8) | 200 `{ status: 'ok' }` with no API key |

Worked example, create. The top of `tasks.create.test.ts`:

```ts
import { afterAll, beforeEach, describe, expect, it } from '@jest/globals';
import { prisma } from '../src/db.ts';
import { authed, TASKS } from './helpers/api.ts';
import { disconnectDb, resetDb } from './helpers/db.ts';
import { defaultOwner } from './helpers/factories.ts';

beforeEach(async () => {
  await resetDb();
});
afterAll(async () => {
  await disconnectDb();
});

describe('POST /api/v1/tasks', () => {
  // the tests go here, indented one level
});
```

and the first test:

```ts
it('creates a task with defaults and stores it', async () => {
  const owner = await defaultOwner();

  const res = await authed().post(TASKS).send({ title: 'Write the tests' });

  expect(res.status).toBe(201);
  expect(res.body).toMatchObject({
    title: 'Write the tests',
    description: null,
    status: 'todo',
    priority: 2,
    dueDate: null,
    userId: owner.id,
  });
  const stored = await prisma.task.findUnique({ where: { id: res.body.id } });
  expect(stored?.title).toBe('Write the tests');
});
```

**Why check the database as well as the response:** a handler can answer 201 and store nothing. The
response is what the API says, the row is what it did.

**Why `.send(...)`, `.set(...)`, `.query(...)`:** `.send(obj)` sends a JSON body and sets the content type,
`.set(name, value)` sets a header, `.query({ a: 1 })` builds the query string. Supertest chains return a
promise-like, so `await` runs the request.

**Done when:**
- [ ] `npm test` passes with the create, read, update and delete files in place.
- [ ] The create, update and delete tests each assert the stored row, not only the response.
- [ ] No test builds its rows by calling the endpoint it is testing (use the factories).
- [ ] You ran the no-cleanup experiment from Stage 4 step 6, saw tests fail in files you had not changed, and
      restored `resetDb()`.

### Stage 6 — Error paths: 400, 404, 409, malformed JSON

*Traces to the brief:* Lab step 2 (validation failures, 404s, 409s, malformed JSON); Build, validation
failures with `details`, 404s, 409s, malformed JSON.

**Goal:** every error response the API can give is pinned by a test, including that nothing was stored.

1. **Validation, with `test.each`.** One table, one test body. Put these in `tasks.create.test.ts`:

   ```ts
   describe('rejects a bad body with 400 and details', () => {
     it.each([
       ['no title', {}, 'title', 'Required'],
       ['an empty title', { title: '' }, 'title', 'Must not be empty'],
       ['an unknown status', { title: 'x', status: 'blocked' }, 'status', 'Must be one of: todo, in_progress, done'],
       ['a priority above 3', { title: 'x', priority: 4 }, 'priority', 'Must be a whole number from 1 (high) to 3 (low)'],
       ['an impossible date', { title: 'x', dueDate: '2026-02-30' }, 'dueDate', 'Must be an ISO 8601 date such as 2026-12-31, or null'],
       ['hour 24', { title: 'x', dueDate: '2026-12-31T24:00:00Z' }, 'dueDate', 'Must be an ISO 8601 date such as 2026-12-31, or null'],
     ])('%s', async (_name, body, path, message) => {
       const res = await authed().post(TASKS).send(body);

       expect(res.status).toBe(400);
       expect(res.body.error).toMatchObject({ code: 'VALIDATION_ERROR' });
       expect(res.body.error.details).toContainEqual({ path, message });
       expect(await prisma.task.count()).toBe(0);
     });
   });
   ```

   Add more rows (a blank title, a title that is a number, a fractional priority). Add one test that sends
   `{ title: '', status: 'nope', priority: 9 }` and checks that **all three** paths are in `details`: the API
   reports every bad field, not the first. The messages are the ones your M08 schemas produce: read yours
   and copy them.

2. **404.** `GET` and `PATCH` and `DELETE` of a well-formed id that does not exist: status 404,
   `error.code` `NOT_FOUND`. Also: deleting the same task twice (the second is 404), and a task that was
   deleted cannot be fetched.

3. **400 for ids and bodies.** A non-UUID id on get, update and delete. `PATCH` with `{}` (at least one field
   is required) and with `{ status: 'finished' }` (and check the stored status did not change). `PATCH` with
   `{ description: null }` is a 400 (`Must be a string`) in this API, while `{ dueDate: null }` is allowed.

4. **Malformed JSON and a body that is too large**, in `middleware.test.ts`:

   ```ts
   it('answers 400 BAD_REQUEST to malformed JSON', async () => {
     const res = await authed().post(TASKS).set('Content-Type', 'application/json').send('{"title":');
     expect(res.status).toBe(400);
     expect(res.body.error).toMatchObject({ code: 'BAD_REQUEST', message: 'Request body is not valid JSON' });
   });

   it('answers 413 to a body over the size limit', async () => {
     const res = await authed().post(TASKS).send({ title: 'x'.repeat(200_000) });
     expect(res.status).toBe(413);
     expect(res.body.error.code).toBe('PAYLOAD_TOO_LARGE');
   });
   ```

   Sending a string with `Content-Type: application/json` is how you send JSON that is not valid JSON: if
   you passed an object, Supertest would serialise it correctly.

5. **409, and why it needs a different trick.** Check your routes: no endpoint can make a unique-constraint
   or foreign-key violation (users and tags have no routes, tasks have no unique column). So there is no
   request that produces a 409. The error handler's mapping (Prisma `P2002` and `P2003` become `409
   CONFLICT`) is still code that must work, so test it with the real parts and a tiny app of your own.
   `tests/error-mapping.test.ts`:

   ```ts
   import { afterAll, beforeEach, describe, expect, it } from '@jest/globals';
   import express from 'express';
   import request from 'supertest';
   import { prisma } from '../src/db.ts';
   import { errorHandler } from '../src/middleware/error-handler.ts';
   import { requestId } from '../src/middleware/request-id.ts';
   import { DEFAULT_OWNER_EMAIL } from '../src/services/task-service.ts';
   import { disconnectDb, resetDb } from './helpers/db.ts';

   // The real request-id middleware and the real error handler, and routes whose Prisma calls really
   // fail on the test database. Nothing is mocked: the 409 comes from a real P2002 or P2003.
   const app = express();
   app.use(requestId);
   app.get('/duplicate-user', async (_req, res) => {
     await prisma.user.create({ data: { email: DEFAULT_OWNER_EMAIL, name: 'A second owner' } });
     res.sendStatus(201);
   });
   app.get('/task-for-missing-user', async (_req, res) => {
     await prisma.task.create({ data: { title: 'Orphan', userId: 999_999 } });
     res.sendStatus(201);
   });
   app.use(errorHandler);

   beforeEach(async () => {
     await resetDb();
   });
   afterAll(async () => {
     await disconnectDb();
   });

   describe('Prisma errors become a 409 CONFLICT', () => {
     it('maps a unique violation (P2002) and does not echo Prisma', async () => {
       const res = await request(app).get('/duplicate-user');

       expect(res.status).toBe(409);
       expect(res.body.error).toMatchObject({
         code: 'CONFLICT',
         message: 'A record with the same unique value already exists',
       });
       expect(JSON.stringify(res.body)).not.toContain(DEFAULT_OWNER_EMAIL);
     });

     it('maps a foreign-key violation (P2003) and stores nothing', async () => {
       const res = await request(app).get('/task-for-missing-user');

       expect(res.status).toBe(409);
       expect(await prisma.task.count()).toBe(0);
     });
   });
   ```

   Express 5 passes a rejected promise from a handler to the error handler, so the routes need no
   `try/catch`.

**Done when:**
- [ ] A `test.each` table drives at least six validation cases, each asserting the status, the
      `VALIDATION_ERROR` code, the `{ path, message }` in `details`, and that nothing was stored.
- [ ] 404 is tested for get, update and delete, and 400 for a non-UUID id.
- [ ] Malformed JSON is a tested 400 (`BAD_REQUEST`) and an oversized body a tested 413.
- [ ] A test produces a real 409 for a unique violation and for a foreign-key violation, with nothing mocked.
- [ ] You can explain why no endpoint test can cover the 409 here.

### Stage 7 — Pagination, filtering, searching and sorting

*Traces to the brief:* Lab step 3; Build, "pagination/filter/sort behavior on the list endpoint".

**Goal:** the list endpoint's `data` and `meta` are right, for pages, filters and every sort.

`tests/tasks.list.test.ts`. Two rules make these tests reliable: make **exactly** the rows the test needs, and
**pass explicit values** for whatever the order depends on, so the clock never decides.

| Behaviour | How to arrange it | What to assert |
| --- | --- | --- |
| Empty list | no rows | `{ data: [], meta: { page: 1, pageSize: 20, total: 0, totalPages: 0 } }` |
| Default page | `makeTasks(25)` | `data` has 20 rows; `meta` is `{ page: 1, pageSize: 20, total: 25, totalPages: 2 }` |
| Every task on exactly one page | `makeTasks(25)`, request pages 1, 2, 3 with `pageSize=10` | the ids from all pages are 25 and all different |
| A page past the end | 3 tasks, `page=5` | 200, `data: []`, the true `total` |
| Status filter | 3 `todo`, 2 `done`, filter `done` | 2 rows, all `done`, `meta.total` is 2 |
| Search | one match in the title, one in the description in different case, one no match | matches only, title and description both searched, case ignored |
| Filter plus search | combine both | only rows that satisfy both |
| Default sort | three tasks with explicit `createdAt` dates | newest first |
| `sortBy=title`, `sortBy=priority&order=desc` | explicit titles and priorities | the expected order |
| `sortBy=dueDate&order=asc` | one task with a date, one without, one earlier | tasks with no due date come **first** (SQLite sorts `NULL` first) |
| Tie-break | six tasks with the same priority | the same order on two requests, and ids ascending |
| Bad query | `status=urgent&sortBy=password&order=up&page=0&pageSize=101` | 400, and `details` lists every one of the five paths |

Worked example, a sort that does not depend on the clock:

```ts
it('sorts newest first by default', async () => {
  await makeTask({ title: 'old', createdAt: new Date('2026-01-01T00:00:00Z') });
  await makeTask({ title: 'new', createdAt: new Date('2026-03-01T00:00:00Z') });
  await makeTask({ title: 'mid', createdAt: new Date('2026-02-01T00:00:00Z') });

  const res = await authed().get(TASKS);

  expect(res.body.data.map((task: { title: string }) => task.title)).toEqual(['new', 'mid', 'old']);
});
```

Also add a `test.each` for query values the API must refuse: `page=1e2`, `page=-1`, `pageSize=0`, `q=` (a
blank).

**Why the tie-break test matters:** if two rows have the same sort value and the order is not fixed, a task
can move between two requests, appear on two pages, or on none. The service adds `id` as a second sort key.
The test proves it, and (Stage 10) you will break it on purpose to see the test fail.

**Done when:**
- [ ] `tasks.list.test.ts` covers an empty list, paging (including no task on two pages), a page past the
      end, the status filter, search, every sort field you test, the `NULL`-first case and the tie-break.
- [ ] A bad query returns 400 and the test checks that every bad parameter is listed.
- [ ] No sort test relies on the order rows were inserted or on the current time.

### Stage 8 — Middleware behaviour and the error handler

*Traces to the brief:* Lab step 3 (middleware behaviour); Build, "request-ID header, the 404 handler, and
that the error handler never leaks a stack trace"; Scope, mocking only at boundaries.

**Goal:** the cross-cutting behaviour is pinned: it holds for every route, so it needs its own tests.

`tests/middleware.test.ts`, one `describe` per concern:

- **`GET /health`:** 200 without a key.
- **API key:** no key and a wrong key are both 401 `UNAUTHORIZED` with the **same** message
  (`Missing or invalid API key`), so the response does not say which it was. Malformed JSON **without** a key
  is a 401, not a 400: the key is checked before the body is read.
- **Request ID:** a generated id (36 characters) on a success, and on a 401; the id in `error.requestId`
  equals the `X-Request-Id` header; a safe incoming id is reused; one with unsafe characters is replaced
  (`'bad id;<script>'`).
- **404 handler:** an unknown route under `/api/v1` (with a key) and outside it both give the `NOT_FOUND`
  envelope. The message is the full path, `Route not found: GET /api/v1/nothing-here`. Read the code before
  you assert an exact string.
- **No leak.** Force a failure the API did not expect and check what the client gets. A spy replaces one
  service method for one test:

  ```ts
  import { jest } from '@jest/globals';
  import { taskService } from '../src/services/task-service.ts';

  it('turns an unexpected failure into a generic 500 with no stack and no cause', async () => {
    const secret = 'ECONNRESET at /srv/secret/path.ts:12';
    jest.spyOn(taskService, 'list').mockRejectedValue(new Error(secret));
    const logged = jest.spyOn(console, 'error').mockImplementation(() => undefined);

    const res = await authed().get(TASKS);

    expect(res.status).toBe(500);
    expect(res.body.error).toMatchObject({ code: 'INTERNAL_ERROR', message: 'Internal server error' });
    const text = JSON.stringify(res.body);
    expect(text).not.toContain('secret');
    expect(text).not.toContain('.ts:');
    expect(text).not.toMatch(/\n\s+at /);
    expect(res.body.error).not.toHaveProperty('stack');
    // The detail goes to the server log, with the request id so the two can be matched.
    expect(logged).toHaveBeenCalledWith(expect.stringContaining(res.body.error.requestId), expect.any(Error));
  });
  ```

  This is the only mock in the suite that touches application code, and it replaces the **service**, not the
  database. `restoreMocks: true` in `jest.config.js` puts both spies back after the test. The `console.error`
  spy only keeps the output quiet.

- **The 500 for a database that was not migrated:** the service throws a plain error when the default owner
  is missing. Test it: delete the owner (`prisma.user.delete`), `POST` a task, expect a generic 500, no
  `owner@example.com` text in the body, and no row stored. `resetDb()` puts the owner back.

**Done when:**
- [ ] Tests cover the request-ID header (generated, reused, replaced), the 401 for a missing and a wrong key,
      the 404 handler inside and outside the API, malformed JSON and the 413.
- [ ] A test proves a failing request answers with a generic 500 that has no stack, no file path and no text
      from the thrown error, and that the log line carries the request id.
- [ ] You can say why the spy is on the service and not on Prisma.

### Stage 9 — `PATCH /tasks/:id/complete`, test-first

*Traces to the brief:* Lab step 4; Build, "Add `PATCH /tasks/:id/complete` test-first"; Definition of done,
"added test-first (red → green → refactor), not after the fact".

**Goal:** three separate steps, in this order, each one verified. The order is the lesson.

**RED: write the tests, run them, read why they fail.** Create `tests/tasks.complete.test.ts` before you touch
`src`. The behaviour to pin, and decide before you code:

| Test | Expectation |
| --- | --- |
| A `todo` task | 200, `status: 'done'`, and the stored row says `done` |
| An `in_progress` task | 200, `done`, and the other fields (`title`, `priority`) unchanged |
| A task that is **already** done | 200 again (completing twice leaves the same state, so a retry is safe) |
| Two tasks, complete one | the other is untouched **and** the target did become `done` |
| An id that does not exist | 404, `NOT_FOUND`, message `Resource not found` |
| A non-UUID id | 400 with `details` for `id` |
| No API key | 401, and the task is unchanged |

Run `npm test -- tests/tasks.complete`. **Read the failures, not just the count.** With no route, requests
fall through to the 404 handler, so the happy-path tests fail with `Expected: 200 Received: 404`, and the
bad-id test with `Expected: 400 Received: 404`. Now check the tests that **pass** in red: any test that
passes when the feature does not exist is not testing the feature. Typical offenders:

- "Unknown id gives 404" passes because the **404 handler** answers a route that does not exist. Make it
  specific: assert the message is `Resource not found` (a missing task, mapped from Prisma's `P2025`), not
  `Route not found: ...`.
- "Only the named task is completed" passes because nothing ran. Also assert the target **did** become
  `done`.
- "No key gives 401" passes because the key check runs before routing. That one is honest, and it stays.

Fix those tests, run again, and expect a count like `6 failed, 1 passed` with the one pass being the 401.
Commit: `test(m09): add failing tests for completing a task`.

**GREEN: the least code that passes.** Four small additions, none of them new ideas:

```ts
// src/schemas/task.schemas.ts: the id is the only input, so the params schema is reused
export const completeTaskRequest = { params: taskParams };

// src/services/task-service.ts
async complete(id: string): Promise<Task> {
  return prisma.task.update({ where: { id }, data: { status: 'done' } });
},

// src/handlers/tasks.handlers.ts (import the type of completeTaskRequest like the other handlers)
export async function completeTask(
  _req: Request,
  res: Response<Task, ValidLocals<typeof completeTaskRequest>>,
): Promise<void> {
  res.status(200).json(await taskService.complete(validOf(res).params.id));
}

// src/routes/tasks.routes.ts
tasksRouter.patch('/:id/complete', validate(completeTaskRequest), completeTask);
```

The route file also imports `completeTask` and `completeTaskRequest`, like the other routes. A missing task
needs no code: `update` throws Prisma's `P2025` and your M05/M07 error handler already maps
it to the 404. Run the whole suite: all tests pass. `npm run typecheck` exits 0. Commit:
`feat(m09): complete a task`.

**REFACTOR: improve the code, change no test.** Look for duplication that the passing tests now let you
remove safely. Here, `complete` is an update of the status, so let it go through `update`:

```ts
async complete(id: string): Promise<Task> {
  return taskService.update(id, { status: 'done' });
},
```

One place writes a task, and the missing-task 404 is the same code path. Run the suite: still green, and you
did not edit a test. If you had to, you changed behaviour and it is not a refactor. If nothing needs
refactoring, say so and do not invent a commit. Commit: `refactor(m09): complete a task through update`.

**Done when:**
- [ ] `git log --oneline` shows the test commit **before** the feature commit, then the refactor commit
      (or you have written down that nothing needed refactoring).
- [ ] You saved the red output: the failing run has `Received: 404` for the happy paths.
- [ ] You can name which tests passed in your first red run, and why a passing red test is a warning.
- [ ] The whole suite passes after the green step and again after the refactor, without editing a test.
- [ ] You can explain which decision writing the test first forced (completing a done task: 200 or 409).

### Stage 10 — Coverage, random order, and trying to break it

*Traces to the brief:* Scope, "Coverage (`--coverage`, thresholds); flaky tests"; Lab, "80%+ coverage on
routes/services, and the suite passing with `--randomize`"; Definition of done, 80% coverage and random order.

**Goal:** measured coverage on the right files, a suite that passes in any order, and evidence the tests can
fail.

1. **Coverage, scoped.** Add to `jest.config.js`:

   ```js
   collectCoverageFrom: ['src/routes/**/*.ts', 'src/handlers/**/*.ts', 'src/services/**/*.ts'],
   coverageProvider: 'v8',
   coverageThreshold: {
     global: { statements: 80, branches: 80, functions: 80, lines: 80 },
   },
   ```

   Run `npm run test:coverage`. Read the table: the **Uncovered Line #s** column names the lines no test
   ran. Write a test for each one that is a real behaviour (not for a line that is only there for the
   compiler). In this project the line that is easy to miss is the service's "default owner is missing" error, which
   is why Stage 8 has a test for it.

   Check the gate works: `npm test -- --coverage --coverageThreshold='{"global":{"lines":101}}'` passes every
   test and still exits non-zero with `Coverage for lines (100%) does not meet "global" threshold (101%)`.
   That exit code is what lets CI fail on coverage.

2. **Random order.** `npm run test:random`. Jest prints the **seed**. Run it several times, and with seeds
   you choose: `npm test -- --randomize --seed=42`. To see the flag work, run one file with two seeds and
   compare the order of the tests. A failure under one seed is reproduced with the same `--seed`.

   If a test fails only in random order, it depends on something another test did. Look for the causes listed
   under "What makes a test flaky" in Concepts, and fix the cause. Never drop the flag: the brief calls that
   out as a bug, not a setting.

3. **Mutation by hand, to prove the tests can fail.** A suite that cannot fail proves nothing. Break the code
   on purpose, run the tests, then undo each change:
   - make the 500 case return `err.stack` in the message in `error-handler.ts`: the no-leak test must fail;
   - remove the `{ id: 'asc' }` tie-break from the list query: the tie-break test must fail;
   - make `complete` store a different status: the complete tests must fail.

   If a change does not make any test fail, you have found a gap: write the test.

**Done when:**
- [ ] `npm run test:coverage` reports 80% or more for statements, branches, functions and lines on routes,
      handlers and services, and exits 0.
- [ ] The threshold check above exits non-zero while all tests pass.
- [ ] `npm run test:random` passes with at least five different seeds, and you ran one file under two seeds
      and saw a different order.
- [ ] You broke at least three behaviours on purpose, saw a test fail for each, and restored the code
      (`git diff` shows none of the mutations).
- [ ] You can explain in your own words what a coverage percentage does and does not tell you.

### Stage 11 — The GitHub Actions workflow and the badge

*Traces to the brief:* Scope, "CI"; Lab step 5; Build, the workflow and the badge; Definition of done,
"Green CI on a pull request".

**Goal:** every pull request runs the checks you ran by hand, on a clean machine.

1. Create `.github/workflows/ci.yml` at the **root of the repository** (not in `task-api/`: GitHub only reads
   workflows from there). Look up the current major versions of the two actions, for example with
   `gh api repos/actions/checkout/releases/latest --jq .tag_name`, and use those:

   ```yaml
   name: CI

   on:
     pull_request:
     push:
       branches: [main]

   # The workflow only reads the repo, so it gets no more than that.
   permissions:
     contents: read

   jobs:
     # One job: the repo is installed once and every step sees the same files. The name is what the pull
     # request's checks list shows (one line per job, not per step), so it says everything the job runs.
     ci:
       name: Install, generate, type-check, test
       runs-on: ubuntu-latest
       # The Task API lives in task-api/, so every `run` step starts there.
       defaults:
         run:
           working-directory: task-api
       # The same settings .env.test holds locally. They are fake: the test database is a throwaway
       # file that `prisma migrate deploy` creates inside the job, one per run.
       env:
         NODE_ENV: test
         API_KEY: ci-test-api-key
         DATABASE_URL: file:./test.db
       steps:
         - uses: actions/checkout@v7

         # The Node version comes from the .nvmrc at the repo root, so CI and a laptop agree.
         - uses: actions/setup-node@v7
           with:
             node-version-file: .nvmrc
             cache: npm
             cache-dependency-path: task-api/package-lock.json

         - name: Install
           run: npm ci

         # The generated Prisma client is git-ignored, so a fresh checkout does not have it.
         - name: Generate the Prisma client
           run: npm run db:generate

         # Jest only strips types, so tsc is what checks them.
         - name: Type-check
           run: npm run typecheck

         # globalSetup gives the run its own test database and migrates it, then Jest runs. The 80% threshold fails this step.
         - name: Test
           run: npm run test:coverage
   ```

   The order is the brief's: install, generate, type-check, test. They are four steps of one job. The
   checks list on a pull request shows one line per **job**, so a job called `test` would look as if only the
   tests ran. Name the job for everything it does, and read the steps inside the job's log.

2. **Rehearse CI locally before you push.** Copy only the files Git would commit into an empty folder, so
   anything your machine has and CI will not (a `.env`, `dev.db`, a generated client, `node_modules`) is gone,
   then run the four steps with the workflow's variables:

   Run the first command **from the root of the repository** (the folder that holds `.github/` and `.nvmrc`,
   not from `task-api/`): `git ls-files` lists paths relative to the folder you run it in, and from
   `task-api/` it would leave out the workflow and the Node version file.

   ```bash
   git ls-files -co --exclude-standard | rsync -a --files-from=- . /path/to/ci-copy/
   cd /path/to/ci-copy/task-api
   NODE_ENV=test API_KEY=ci-test-api-key DATABASE_URL=file:./test.db \
     sh -c 'npm ci && npm run db:generate && npm run typecheck && npm run test:coverage'
   ```

   If this fails, CI will fail, and you can fix it without waiting for a runner. If it passes, CI can still
   differ (the operating system and the build of `better-sqlite3`), so it is a rehearsal, not proof.

3. **The badge.** Add this line under the title of `README.md` in the repo root, and of `task-api/README.md`
   (replace the account and repository with yours):

   ```markdown
   [![CI](https://github.com/<you>/<repo>/actions/workflows/ci.yml/badge.svg)](https://github.com/<you>/<repo>/actions/workflows/ci.yml)
   ```

   The badge shows the state of the workflow, so it has nothing to show until the workflow has run.

4. *Optional, and not required by the brief:* bring `task-api/README.md` up to date: v4, the
   `PATCH .../complete` row in the endpoints table, a Testing section (settings, a database per run,
   `--runInBand`, `--randomize`, coverage), the new scripts, and the layout. Include a warning not to point
   `DATABASE_URL` at a fixed file by hand: it was the reason for the per-run database.

**Done when:**
- [ ] `.github/workflows/ci.yml` is at the repository root and has the four steps in the brief's order.
- [ ] The local rehearsal in a clean copy passes: install, generate, type-check, test with coverage.
- [ ] Both READMEs have the badge, with your own account and repository in the URL.
- [ ] You can say why the workflow sets `API_KEY` and `DATABASE_URL` instead of relying on a `.env.test`.

### Stage 12 — Verify against the definition of done, and open the pull request

*Traces to the brief:* Verify; Definition of done (green CI, 80% coverage, random order, test-first
`complete`); the branch → PR → merge habit.

**Goal:** prove each box with a command, then let CI prove it on a clean machine.

1. **Count the tests.** `npm test` prints `Tests: N passed`. N is at least 25, and every endpoint
   (`GET /health`, list, create, get, update, delete, complete) has a happy path and its error paths.
2. **Coverage and order.** `npm run test:coverage` (80%+) and `npm run test:random` (green, several seeds).
3. **Typecheck.** `npm run typecheck` exits 0. Jest does not check types; this does.
4. *Optional, and not required by the brief:* check that the old checks still pass. Bruno is part of the
   project: add requests for the new route (create a task, complete it, complete it again, delete it, plus a
   404 and a 400), then run the collection against a server that uses a throwaway database (not `dev.db`, so
   your development data is not touched). The test runs create and delete their own database, so make one
   for this by hand:

   ```bash
   DATABASE_URL=file:./test-bruno.db npm run db:deploy
   API_KEY=<the apiKey in bruno/environments/local.bru> DATABASE_URL=file:./test-bruno.db PORT=3000 \
     npx tsx src/server.ts &
   npm run bruno
   kill %1                       # in an interactive shell; otherwise stop the server by its process id
   rm -f test-bruno.db*
   ```

5. **Development data untouched.** The checksum of `dev.db` matches Stage 1.
6. **`git status`.** You should see your source changes, `tests/`, `jest.config.js`, `.env.test.example`,
   `.github/`, the README changes and the lockfile. You must **not** see `.env.test`, a `test*.db` file or
   `coverage/`.
7. **Commit, push, open the pull request**, then wait for CI. Conventional Commits, no direct push to `main`,
   and the PR goes to your own repository:

   ```bash
   git push -u origin feat/module-09/integration-testing
   gh pr create --repo <you>/<repo> --title "feat(m09): integration tests, CI and PATCH /tasks/:id/complete" --body "..."
   gh pr checks --repo <you>/<repo> --watch
   ```

   Open the Actions tab on a failure and read the step that failed. Reproduce it with the rehearsal from
   Stage 11, fix it, push again.
8. Self-review against every Definition of done box (section 6), then merge when CI is green and your
   reviewer approves.

**Done when:**
- [ ] The pull request shows a **green** CI check.
- [ ] `npm test` reports 25 or more passing tests, and `npm run test:coverage` reports 80%+ on routes and
      services.
- [ ] `npm run test:random` passes.
- [ ] `git log` shows the `complete` tests committed before the code that makes them pass.
- [ ] `.env.test`, any `test*.db` file and `coverage/` are not in the pull request.

## 4. Concepts you need

### Quick reference

| You want to... | Use |
| --- | --- |
| Group tests, name one test | `describe('...', () => {})`, `it('...', async () => {})` |
| Run code around tests | `beforeAll`, `afterAll`, `beforeEach`, `afterEach` |
| The same test over a table | `it.each([[...], [...]])('%s', (a, b) => {})` |
| Check a value | `expect(x).toBe(1)` (same value), `toEqual` (deep), `toMatchObject` (named fields), `expect.objectContaining` |
| Check an array has an item | `toContainEqual({ ... })` |
| Check a promise | `await expect(p).rejects.toThrow('text')` |
| Compare with a stored copy | `expect(x).toMatchSnapshot()`: use sparingly, because a changed snapshot is easy to accept without reading it. Prefer asserting the fields you care about |
| A fake function or a spy | `jest.fn()`, `jest.spyOn(obj, 'method')` |
| One request | `request(app).get(url).set(h, v).query(q).send(body)` |
| A reusable header or cookie jar | `request.agent(app).set(h, v)` |
| Run in random order, reproduce | `--randomize`, `--seed=<n>` |
| Coverage | `--coverage`, `coverageThreshold` |

### The testing pyramid, and where integration tests sit

A **unit test** runs one function with its neighbours replaced: fast, precise about where a bug is, blind to
anything between the pieces. An **end-to-end test** drives the whole running system the way a user does
(a browser, a deployed server): slow and fragile, but it proves the pieces fit. An **integration test** sits
between: several real parts together, here the router, the middleware, the handler, Prisma and SQLite, with
the network and the process boundary removed. The pyramid says to have many of the cheap ones at the bottom
and few of the expensive ones at the top. For a small API whose logic lives in the seams (validation, a
query, an error mapping), integration tests are the cheap ones, which is why this stage has no unit tests of
the service in isolation. What to test in an API: every route's success, every error it can send, and the
behaviour every route shares.

### Why a real database and not a mock

A mock checks that your code calls the mock the way you imagined. It cannot know the query is wrong, that
the migration lacks the column, that SQLite sorts `NULL` first, that a unique index rejects the second row, or
that Prisma throws `P2025` for a missing row. Those are the bugs an API actually has. With a real database
the test fails for the same reasons production would. The price is a setup step and a cleanup (Stages 3-4),
and slower tests, which for SQLite in a file is small. Mock only at **boundaries you do not control or cannot
trigger**: a payment provider, the clock, a failure you cannot cause on purpose. Forcing the service to throw
in Stage 8 is the example here, and it is the exception, not the pattern.

### What makes a test flaky

A flaky test passes and fails with no change to the code. Almost every cause is one of three. **Shared
state**: rows, files, counters or environment variables one test leaves for the next, and also **another
test run** using the same database file. **Time**: a test that compares to "now", sorts by a creation time
that two rows share, or sleeps and hopes. **Order**: a test that passes only because another ran first.
Isolation (clean tables before each test, a database per run), explicit values (pass the dates you sort by)
and `--randomize` are the three defences, in that order of importance. One limit of random order is worth
knowing: a clean run under a few seeds shows that those seeds found nothing, not that no dependency exists.
Another: it reorders tests, and it cannot find a dependency on something outside the tests, such as a file
another process is using.

### Red, green, refactor

Each step of the cycle buys a different proof, which is why the order matters. **Red** proves the test can
fail, so a later green means something: it is the test's own test. **Green** proves the code satisfies what
the test says, and nothing more, so the code stays as small as the specification. **Refactor** proves the tests
protect the design and not only the behaviour of one version of the code: you change the structure, and the
tests, unchanged, say whether you broke anything. Write the test after the code and you lose the first proof,
and the test tends to describe what the code does instead of what it should do. That is also why the
decisions (here: what to do with a task that is already done) get made when the test is written, while they
are still choices.

### What a coverage number tells you

Coverage is the share of lines, branches and functions that ran while the tests ran. It tells you what was
**not** run, which is useful: a line no test touched is untested for certain. It does not tell you that what
ran was **checked**: a test that calls an endpoint and asserts nothing covers every line it reaches. So read
the report as a list of unvisited code, not as a score, and pair it with a way to prove tests can fail, such
as breaking the code on purpose (Stage 10). A threshold is a floor that stops coverage drifting down in CI.
It is not a target to game. This guide uses `coverageProvider: 'v8'`, which reads what the JavaScript engine
ran instead of instrumenting the code through the transform. Whether the default provider (Babel and
istanbul) also works with `@swc/jest` is something you can check yourself.

### Mocking in ESM

In CommonJS, `jest.mock('./module')` works because Jest moves it above the `require` calls (the usual explanation). ES module imports
are resolved before any code runs, so a `jest.mock` call under a static import changes nothing, and it does
not warn you: the real module simply runs. Run this to see it, with a scratch test: call `jest.mock` on a
module you also import at the top, and log which function you got. The ESM way is `jest.unstable_mockModule`,
called **before** a dynamic `await import(...)` of the thing under test:

```ts
jest.unstable_mockModule('../src/services/task-service.ts', () => ({ taskService: { list: () => 'mocked' } }));
const { taskService } = await import('../src/services/task-service.ts');
```

Prefer `jest.spyOn(object, 'method')`, as in Stage 8, when the thing is a method on an object you can import:
it needs no module mocking at all.

### One writer, one run: `--runInBand` and a database per run

Two different collisions are possible, and the setup handles each one differently. **Inside one run**, Jest
normally runs test files in parallel in several processes, and SQLite takes a lock for every write, so two
files writing to one database would collide (or clean each other's rows). `--runInBand` runs the files one
after another in one process and removes that. The alternative is one database file per worker: build
`test-<n>.db` from `JEST_WORKER_ID`, migrate each, and parallel files become safe. It is faster on a large
suite and more setup, and a suite that runs in seconds does not need it. **Between runs**, two test processes
at the same time (two terminals, an editor's test runner, a watcher) would share one fixed file, and each
run's cleanup would wipe the other's rows. A database per run fixes that, and it costs one file name built
from the process id and the time, and a teardown that deletes it.

## 5. Common pitfalls

- **Running on the wrong Node.** On Node 22, every database test fails with `better_sqlite3.node was compiled
  against a different Node.js version`. Check `node -v` before you debug anything else, and especially after a
  command chain that stopped early.
- **Two installs at once.** If an `npm install` seems stuck, make sure the process is really gone
  (`pgrep -fl npm`) before you start another. Two can write the same `node_modules`, and one of them may
  finish minutes later and leave packages missing.
- **Tests that pass without the feature.** Read the red run. A 404 test that passes before the route exists is
  answered by the 404 handler.
- **Guessing an exact message.** Open the code and copy the real string. The 404 handler's message is the full
  path (`/api/v1/...`), not the path inside the router.
- **Pointing the tests at `dev.db`.** The guard in `globalSetup` and in `resetDb` exists for this. Keep both.
- **Two test runs on one database file.** If `DATABASE_URL` names a fixed file and a second test process starts
  (another terminal, an editor's test runner, a watcher), the two runs wipe each other's rows and tests fail
  in different places each time, with counts and lists that look wrong. That is the failure the per-run
  database prevents: do not point the tests at a fixed file by hand, and if you see this, look for a second
  Jest process (`pgrep -fl jest`).
- **Asserting on the clock or on insertion order.** Pass explicit dates and values, and give ties a defined
  order.
- **Trusting only the response.** Assert the stored row for writes, and `count` for "nothing was stored".
- **No 409 route.** Do not invent an endpoint to get a 409. Test the error handler's mapping with a small
  app and real Prisma failures.
- **`jest.mock` under ESM does nothing, silently.** Use `jest.spyOn` or `jest.unstable_mockModule`.
- **Putting `ci.yml` in `task-api/`.** GitHub will not run it. It belongs in `.github/workflows/` at the root.
- **Committing `.env.test`, a `test*.db` file or `coverage/`.** Run `git status` before every commit.
- **Dropping `--randomize` when it fails.** A random-order failure is a real dependency between tests. Fix it.

## 6. Check yourself against the definition of done

- [ ] **Green CI on a pull request.** The PR's CI check is green: install, generate, type-check, test.
- [ ] **80% or more coverage on routes and services.** `npm run test:coverage` reports it and exits 0.
- [ ] **The suite passes in random order.** `npm run test:random` is green under several seeds. If it is not,
      you found an order dependency to fix, not a flag to avoid.
- [ ] **`PATCH /tasks/:id/complete` was added test-first.** `git log` shows the failing tests committed
      before the code that makes them pass, then the refactor.
- [ ] **25 or more integration tests**, covering every endpoint and every error path, against the real app
      and a real SQLite test database, with nothing in the data layer mocked.
- [ ] **A separate test database for every run**, migrated by `globalSetup`, deleted by `globalTeardown`, cleaned
      in `beforeEach` in foreign-key-safe order, disconnected in `afterAll`, and unable to touch `dev.db`: the
      name is checked in the setup and again in `resetDb()`.
- [ ] **The toolchain note.** You can state whether `@swc/jest` worked as is or which fallback you needed,
      and why.
- [ ] **The status badge** is in the README.
- [ ] **You can explain** why integration tests here use a real database, what makes a test flaky, and the
      red-green-refactor cycle, each in your own words.

## 7. Next step

M10 is the capstone. It has no reference material except what you write, so keep this stage's pieces as a
starting kit for your own project: `jest.config.js`, `tests/global-setup.ts` with its name guard,
`tests/helpers/` (`resetDb`, factories, the Supertest agent), the workflow file, and the habit of writing the
failing test first. Put the workflow in from your first pull request, so CI is green from the start and not
discovered at the end.
