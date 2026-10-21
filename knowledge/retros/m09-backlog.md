# M09: backlog

Items found while building and writing up M09 (2026-10-21). The build is `task-api/` in the build repo, branch `feat/module-09/integration-testing`; the documents are in `modules/m09-integration-testing/`. The build repo's work is merged: [PR #9](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/9) (11 commits, CI green) went into `main` as `ed7d65b` on 2026-10-21 at 14:12 UTC. The documents are committed on `feat/module-09/write-up-and-training-material` of the content repo and submitted as a pull request to `node-js-training-rnd`.

Checks run (all on Node 24.22.0):
- **The suite:** 88 tests in 9 files (71 integration tests of the API and 17 for the database guards), all passing; `npm run typecheck` exits 0 with `tests/` included. The first version had 71 tests in 8 files.
- **Random order:** seeds 1, 2, 3, 42, 777, 98765, 20261021, one automatic seed (1019317433) on the 63-test suite, 5, 11, 12, 13 on the 70 and 71-test suites, and 1, 2, 3, 5, 42, 777, 20261021 and one automatic seed (-1166006607) on the final 88-test suite. All passed. Three runs started at the same time (seeds 1, 2, 3) all passed too. `tests/tasks.delete.test.ts` gave a different test order under seeds 1 and 2. No ordering dependency was found.
- **Coverage:** routes, handlers, services: 100 / 100 / 100 / 100 (statements, branches, functions, lines). Before the missing-owner test: 98.97 / 95.45 / 100 / 98.97. All of `src/` except the generated client: 95.93 / 87.28 / 93.33 / 95.93 (before that test) and 96.16 / 88.23 / 93.33 / 96.16 on the final run. A raised threshold (101) failed the command with exit 1 while all tests passed.
- **Mutations by hand:** the error handler returning `err.stack` failed 2 tests; removing the id tie-break failed 1; `complete` storing the wrong status failed 4; `resetDb` doing nothing failed 20 of the 63 tests then existing; the database name check disabled failed 9 of the 17 guard tests.
- **`dev.db` untouched:** the MD5 was `6367cdfaf1b61235d07c3aa26d0c4db2` before and after the test runs, after Bruno (run against a throwaway `test-bruno.db` made by hand with `db:deploy` and removed afterwards) and after `DATABASE_URL=file:./dev.db npm test`, which stopped with the `Refusing to touch the database` error.
- **Bruno:** 53 requests, tests 66/66, assertions 104/104 (M08 baseline 47 / 63 / 91 plus the new `Complete/` folder and `Errors/29`, `30`).
- **CI rehearsal, not real CI:** the files git would commit, copied into a clean folder, then `npm ci` (954 packages), `npm run db:generate`, `npm run typecheck`, `npm run test:coverage` with the workflow's environment variables: 9 suites, 88 tests passed after the review fixes, and no database file left in `task-api/`. The guide's own code blocks were also extracted into a clean copy: it typechecks and its 21 tests pass.

## [BACKLOG] Not done

- **Resolved: the merge.** PR #9 was merged as `ed7d65b`. The write-up has the merge commit and `git log --graph`, and the `tasks.md` box is ticked. All 28 boxes in `tasks.md` are ticked now.
- **Resolved: the run on `main`** (run 39614768472) briefly read `in_progress` at the run level although its only job, `Install, generate, type-check, test`, had completed. It is now `completed`, `success`.
- **The feature branch is still on the remote** (`feat/module-09/integration-testing`). The PR was merged without deleting it. Nothing here deletes it.
- **The content repo's PR is not merged yet.** The write-up, the training material, `tasks.md` (all 28 boxes ticked), `coding-standards.md` and the two backlogs are committed on `feat/module-09/write-up-and-training-material`. The EM's review is still to come.
- **Resolved:** green CI (runs 39611518300 at `73a27af` and 39612459807 at `0ccf4ed`, both on `ubuntu-latest` with Node 24.22.0); the `complete` work is three commits (`fc95e0a` red, `e5c2840` green, `2e22647` refactor) and the red state was re-run at its commit (6 of 7 complete tests fail); the M08 comment fix is its own commit (`d5325fc`); the CI job was renamed from `test` to `Install, generate, type-check, test` because the checks list shows one line per job (nothing required the old name: `main` has no branch protection and the repo has no rulesets).
- **The commits were assembled after the build,** from the finished working tree, in the order the work was done. Their dates show when they were committed, not when each file was written.
- **Final self-review against `brief.md`:** VALIDATE and a code review were run (see below). The EM's review is still to come.

## Code review (2026-10-21, one read-only reviewer per repo, before any commit)

Neither reviewer found a critical or high defect. Fixed in the working trees:

- **Build, a shared `test.db` broke when two runs overlapped.** Both reviewers hit it, because they ran Jest at the same time (1 to 12 failures a run, different each time). Each run now has its own database (`test-<pid>-<time>.db`), deleted by a `globalTeardown`. Three simultaneous runs passed. The README, the guide's Common pitfalls and the write-up carry the warning, as the original failure mode.
- **Build, the name guard was crude and only in `globalSetup`.** It is now `testDatabasePath` in `tests/helpers/test-database.ts` (resolved path, `file:` only, query string cut off, `test...db`), used by the setup, the teardown and `resetDb()`, with 17 tests. `file:./dev.db?x=/test.db` and `file:./test.db/../dev.db` are refused.
- **Build, a setup file turned out to be redundant** and was removed (`globalSetup` loads `.env.test` before the test files start). Checked with and without workers.
- **Build, the "moves `updatedAt` forward" test could not fail.** It now seeds `updatedAt` in the past and asserts a strictly later value, in the response and in the stored row.
- **Build, stale headers** (`tasks.handlers.ts` said five handlers; routes, service and schemas did not mention Module 9), the unused `makeUser` (removed), and `*.db-journal`, `*.db-wal`, `*.db-shm` in `.gitignore`.
- **Content, the guide:** a wrong test count in Stage 5, concepts that repeated stage text (flaky tests, red-green-refactor, rewritten to add a model instead), a wrong `--runInBand` cross-reference, asides about the author's build, the CI rehearsal (it must start from the repository root), missing imports in the Stage 5 example, and the README update and Bruno run now marked optional because the brief does not require them. The write-up's all-of-`src/` coverage figures were stale and are updated.

Still open after the review:
- **The guard checks `process.env.DATABASE_URL`, and Prisma keeps the URL it was built with at import.** If a test changed the variable after the import, the two could differ. No test does that, and the guard test shows the check runs first, but a guard that read the URL `db.ts` actually used would be stricter. `db.ts` is not part of this stage.
- **A run killed with SIGKILL leaves its `test-<pid>-<time>.db` behind.** It is git-ignored. There is no cleanup of old ones.
- **`kill %1` stopping the Bruno server in an interactive shell was not confirmed** (a non-interactive run has no job control). The guide says to stop the server by its process id otherwise.
- **The brief's "snapshots used sparingly" and `rejects`/`toThrow` are only a row each in the guide's quick reference, with no stage that uses them.** Neither is a `tasks.md` task.
- **A stray `task-api/test-review.db`** a reviewer left behind was removed (git-ignored; no other `test-*.db` is left).

## [BACKLOG] Questions for the EM

- **There is no way to get a 409 from this API.** The brief and `tasks.md` ask for tests of "409 on unique violations". Users and tags have no routes, and `Task` has no unique column, so no request can cause a P2002 or P2003. I covered the error handler's mapping with a test-only Express app (the real `requestId` middleware and the real `errorHandler`, routes that make real Prisma failures on `test.db`, nothing mocked). Is that what the brief means? If a unique endpoint is expected, it is not in v3.
- **`ci.yml` location.** The architecture document's project layout puts `.github/workflows/ci.yml` inside `task-api/`. GitHub only reads `.github/workflows/` at the repository root, so the build repo has it at the root with `defaults.run.working-directory: task-api`. The layout in the architecture document (and the copy in `knowledge/rules/arch-summary.md`) needs the same correction. Not edited: they are source documents.
- **The architecture document's `jest.config.js` and `tests/` sit inside `task-api/`**, which matches what was built. Only the workflow differs.
- **`test` script naming:** the brief's script is `test`. Two more were added (`test:coverage`, `test:random`) so CI and the guide can call coverage and random order by name. Confirm these extra scripts are acceptable.
- **Completing a task that is already done is a 200**, decided while writing the red tests. The brief does not say. A 409 would also be defensible.

## [BACKLOG] Not verified / limits

- **Random order was checked with a dozen seeds, in one file at a time for the order check.** It was not verified that `--randomize` also reorders test files, only tests inside a file.
- **`coverageProvider: 'v8'` was chosen, and the default (Babel/istanbul) provider was never tried** with `@swc/jest`, so the guide cannot say whether it would have worked.
- **The explanation of why `jest.mock` does nothing under ESM** (the call is not moved above the imports) was not tested. The two outcomes were: `jest.mock` after a static import left the real function in place with no error or warning, and `jest.unstable_mockModule` plus a dynamic `import()` returned the mock.
- **The `.ts` import specifiers worked with no `moduleNameMapper`** (open decision P2 in `AGENTS.md`). That was tested on macOS with Node 24.22.0 only, not on Windows or Linux.
- **`--runInBand` is the only parallelism option tried.** One database per worker with `JEST_WORKER_ID` was described in the guide and not built.
- **The M08 backlog asked for a regression test for the four `dueDate` differences.** The create tests pin two of the boundaries (`2026-02-30` and hour 24 are 400s) and the date-only form (midnight UTC). Year 0050, year 9999 with an offset, and the minutes-only form are still only in Bruno (`Dates/`). The extra-field case that `satisfies` cannot catch has no test either.
- **`logger.ts` (74%) and `env.ts` (79%) are the least covered files in `src/`.** Neither is a route, handler or service, so neither is in the gate.

## [BACKLOG] Things that went wrong while building (for the next cohort)

- **A second `npm install` while the first was stuck.** The first install (Jest, Supertest, swc) hung with no output for over ten minutes. After it was killed and a second install finished, the first install's process exited with code 0 and `node_modules/jest` was missing. A plain `npm install` repaired it (237 packages, 1 second). Package versions in the lockfile were checked against `HEAD`: 285 packages added, none removed, none changed.
- **Node 22 ran the tests once.** A command chain stopped before the `PATH` export and `npm test` ran on Node 22.23.0: all 63 tests failed in `resetDb` with `better_sqlite3.node was compiled against a different Node.js version using NODE_MODULE_VERSION 137`. The `engine-strict` or startup-check question from the M08 backlog is still open and is now more concrete: a test run on the wrong Node gives 63 red tests that look like a test bug.
- **A wrong guess at an error message** (the 404 handler's message includes `/api/v1`) made 1 of 63 tests fail on the first full run. The test was wrong.
- **Three of seven first-run tests for `complete` passed with no route.** They were rewritten before any code was written (see the write-up).
