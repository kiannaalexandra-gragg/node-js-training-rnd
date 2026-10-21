# M07: backlog

Items found in VALIDATE (2026-10-21). The build (`task-api/` in the build repo, branch `feat/module-07/prisma-orm-migrations`) and both documents were checked against the brief's Definition of done, the tasks checklist and the AGENTS.md quality gates.

Checks run:
- **A fresh copy of the project** (tracked and new files only: no `node_modules`, `dev.db`, generated client or `.env`), set up by following the README: `npm install`, `npm run db:generate`, `npm run db:deploy`, `npm run db:seed`, `npm run typecheck`, `npm start`, `npm run bruno`, `npm run check:errors`. Results: typecheck failed with `Cannot find module './generated/prisma/client.ts'` until `db:generate`, then exited 0; deploy applied 3 migrations; seed printed `users 6, tasks 24, tags 7, task_tags 37`; Bruno `36 (36 Passed)`, tests 49/49, assertions 76/76; `check:errors` exited 0 with three `PASS`; the server printed `Server closed` on SIGTERM.
- **The guide's Stage 2 install commands** in an empty project: one `better-sqlite3@12.11.1`, `prisma@7.10.1`, SQLite `3.53.2`.
- **Embedded code against the build files**: `db.ts`, `task-service.ts`, `check-prisma-errors.ts` and the validation blocks match line for line. Header comments are left out of the guide on purpose, and `tasks.handlers.ts` is shown as the changed functions only.
- **Edge cases by `curl`** against a running server: dates (`2026-02-30`, `2026-04-31`, `2026-02-29`, a leap day, an offset date-time), `?q=%25`, `?q=_`, `?page=9007199254740991`, `?page=1000000000000000`, `?pageSize=1000`.
- **AGENTS.md gates**: `brief.md` and `write-up-template.md` untouched (only `tasks.md` ticks), the write-up has its seven sections in order, the training material has its six sections plus Builds on and Next step, no bare `npx prisma` outside warnings, no secrets (only the fake `example-api-key-change-me`), exactly one `graphify-out/`.
- **Prisma Studio** served on SQLite and was opened in a browser (24 `Task` rows).

VALIDATE fixed one FIX NOW:
- **`dueDate: "2026-02-30"` was accepted and stored as 2 March.** `new Date()` rolls an impossible day over, and the pattern check did not catch it. Added `isRealDay` to the validation (also in the training material's Stage 9 block), `Errors/24` to the Bruno collection, and a pitfall, a coding-standards bullet and a write-up entry. Re-checked with `curl`: the four impossible dates return 400, the three valid ones return 201.

## [BACKLOG] Not fixed

- **`?q=` treats `%` and `_` as LIKE wildcards.** `?q=%25` matches all 24 tasks. Prisma's `contains` does not escape them. A fix needs `$queryRaw` with an `ESCAPE` clause. It is documented in the README, the write-up and the guide's pitfalls.
- **`?sortBy=dueDate&order=asc` lists undated tasks first** (SQLite sorts `NULL` first). Documented in the README and the guide. Prisma's `nulls: 'last'` option was not tried.
- **`2026-12-31T24:00:00Z` rolls over to 1 January 2027.** It is valid ISO 8601, so it is accepted.
- **The empty-PATCH message still says "title, description, status".** Bruno `Errors/12` asserts it word for word, and `priority` and `dueDate` now count as a change too.
- **`prisma/seed.ts` and `scripts/` are only type-checked because `tsconfig.json` lists them.** The `include` was widened for this; the program may prefer a separate tsconfig.
- **`DATABASE_URL=file:./dev.db` is relative to the working directory**, so starting the server from another folder would open another file. `db:studio`'s `file://./dev.db` has the same dependence. Both are documented as "run from `task-api/`".

## [BACKLOG] Not verified

- **A machine with no prebuilt `better-sqlite3` binary**, and the `@prisma/adapter-libsql` fallback. `npm install` printed `allowScripts` warnings for six packages; nothing was approved and the driver loaded anyway.
- **Windows.** `db:studio` uses `file://./dev.db` and the guide's commands are shell commands for macOS and Linux. The relative `file://` form was only run on macOS.
- **A migration conflict in a team PR.** The guide describes the repair as reasoning and says it was not run.
- **The concept-only topics** (PostgreSQL, MongoDB, Prisma 8) come from the Prisma documentation and were not run. The Prisma 7 pages disagreed on whether Prisma 7 supports MongoDB. The Prisma 8 page does not describe SQLite.
- **`db push`** was not run.
- **Prisma Studio was looked at, not exercised.** No row was edited or inserted through it.
- The guide was **not** followed stage by stage by someone who did not build it. The fresh-copy run covers the README flow, not every stage.

## [BACKLOG] Decisions made without the EM (EM-owned files, conventions)

- `tasks.md` is EM-authored. 33 of its 34 checkboxes were ticked, as in M05 and M06, and nothing else in it changed. The EM may prefer the file left untouched. "Continue the branch → PR → merge habit" stays unticked until the PRs exist.
- **P2003 is 409, not 400**, and the message is fixed text. (The brief says "409 or 400".)
- `Task.id` stays a UUID string and `User` and `Tag` use integer ids. The M06 backlog had left the id mapping to M07.
- Every task created through the API belongs to one default owner, `owner@example.com`, because v1 has no users. The hand-edited migration creates it.
- Bruno `Tasks` 02, 03 and 04 were changed for the v2 response shape, and the `Listing/` folder and `Errors/23` and `24` were added. No other request changed.
- **The URL prefix stays `/api/v1` although the response of `GET /api/v1/tasks` changes shape** (an array becomes `{ data, meta }`, and a task gains fields). "v2" is the project iteration, and the M05 brief fixes `/api/v1`. The M07 definition of done also requires the M05 Bruno collection to stay green, and every request in it uses `/api/v1`. Putting a breaking change under the same URL version is only reasonable here because the API has no clients besides its own collection. This was raised after the PR was opened. The README now says so. **Question for the EM:** should a change like this ever move to `/api/v2`, and if so should M08 and M09 do the same?
- `.ts` import specifiers, with the generator's `importFileExtension = "ts"` to match. The program's convention (P2) is still open.
- `prisma init` created AI-assistant folders (`.agents/`, `.claude/`, `.windsurf/`), `skills-lock.json` and a `.gitignore` in `task-api/`. They were deleted. The guide tells the learner to delete them.
- `migrate reset` was run twice (once during APPLY, once after the code review), each time with the owner's explicit consent given to Prisma's AI-agent guard. Both gave the same counts.

## [BACKLOG] Status of the PRs and the checkpoint

- **Build repo:** committed as seven Conventional Commits (no `Co-Authored-By` trailer) on `feat/module-07/prisma-orm-migrations`, pushed, and opened as [PR #7](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/7) against the owner's own repo. It was MERGED by the owner on 2026-10-21 09:53 UTC (merge commit `3604be1`). A fresh clone of the pushed branch passed the README flow (typecheck 0, Bruno 36/36, `check:errors` 3 PASS).
- **Content repo:** nothing is committed. `feat/module-07/write-up-and-training-material` holds the two new documents, the `tasks.md` ticks, the `coding-standards.md` edit and this file, all uncommitted, and no PR exists yet.
- `write-up.md` has no `GAP:` left. Its Checkpoint evidence has the PR #7 link and the `git log --graph` of `main` after the merge. As in M05 and M06, the author's PR link belongs in `write-up.md` only, not in the training material. The trainer's own checkpoint check is recorded as "not recorded", as in M06. The checks were not re-run on `main` after the merge.
- The "explain `migrate dev` against `migrate deploy` in your own words" text in Checkpoint evidence was drafted from the runs, and the trainee then confirmed it as their own (2026-10-21). On that review two claims that went beyond the runs were cut back: `migrate dev` "offers to wipe the database if the history does not match" became "when an applied migration was edited" (the only case seen), and `migrate deploy` "never resets" became "writes none, and I never saw it ask for a reset". `deploy` was not run against a database with an edited migration, so what it does then is not claimed. The same edit is in the training material's Stage 14.

## [BACKLOG] Code review (2026-10-21, one reviewer per repo, before any commit)

The review found no critical or high issue. Fixed in the working trees:

- **Build:** Prisma's own `error` log printed a code frame for every normal 404 or 409, in the server log and 6 times in `check:errors`. `db.ts` now logs `warn` only (`query` and `warn` with `LOG_QUERIES=true`); `check:errors` prints three lines. The README said all config is read in `config.ts`; it now names the `DATABASE_URL` exception. The README seed counts, the `db:deploy` note about dev dependencies, `sortBy=title` being case-sensitive, and `description` not being clearable were corrected or added. Bruno `Listing/03` now asserts that the task from step 01 is on the page (Tests 49/49). An error-handler comment claimed Prisma's text can name the values sent; only a code frame and field names were seen, so it was reworded here and in the guide.
- **Content:** Stage 9 now says to replace the M05 `parseListQuery` (pasting a second one is a duplicate). Concepts no longer repeat Stage 7's required-column walkthrough or Stage 10's mapping comment. The claim that `migrate reset` is the only command that replays the chain from nothing was wrong (Stage 13 does it with `deploy`) and is fixed. The tour now covers `delete` and `GetPayload` with `include` (the brief's scope lists both), and the extracted guide block was run verbatim. The tour clean-up order, the `rm` instruction in Stage 3, the LOG_QUERIES example's source, Stage 13's second reset, the migration-conflict note about a dev database, and the `coding-standards.md` bullets (split, no M07 provenance, and the unverified claim that `migrate reset` does not run `generate` removed) were corrected.
- **Evidence:** the `tasks.md` tick for "rebuilds from scratch, every time" was backed by one reset. It is now backed by two.

Not fixed, left as notes:

- **`sortBy=title` is case-sensitive and `sortBy=status` is alphabetical**, and a `description` cannot be cleared. Both are now documented in the README, and neither was changed.
- **`DATABASE_URL` and `LOG_QUERIES` are read in `db.ts`, not `config.ts`.** The M05 config is replaced by Zod in M08, which is the natural place to move them.
