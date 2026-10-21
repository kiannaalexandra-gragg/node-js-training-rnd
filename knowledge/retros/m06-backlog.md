# M06: backlog

Items found in VALIDATE (2026-10-21). The build (`modules/m06-sql-fundamentals-sqlite/` in the build repo) and both documents were checked against the brief's Definition of done, the tasks checklist and the AGENTS.md quality gates.

Checks run: every exercise file on a fresh `schema.sql` plus `seed.sql` database (25 exercises, 6/5/5/4/5 per file); `PRAGMA foreign_key_check` (empty); the ERD columns against `PRAGMA table_info` for all four tables; `mermaid.parse` on the ERD (`er`); `npm run typecheck` and `npm run demo` (both exit 0). The training material's code blocks were also extracted and compared byte for byte with the build files (all in sync), and the guide's own stages 3, 4, 11 and 13 were followed in an empty directory, where the demo exited 0.

VALIDATE fixed three FIX NOWs. Stage 11 left the learner in `injection-demo/`, so Stage 12's `sqlite3 m06.db` commands would have created an empty database there; the guide now says `cd ..` and names the folder. The Mermaid cardinality markers were written for the wrong side of the line and are now given for both sides. Stage 10 and Stage 11 defined an index and the injection mechanism, which the Concepts section explains in full, so both were cut to one line and a pointer. A `schema.sql` comment also pointed at "Exercise 05", which is a file, not an exercise; it now says exercises 24 and 25.

Nothing in this file blocks the build or the two documents.

The `/code-review` of both repos (one reviewer per repo, after the first VALIDATE) found 16 issues, none a correctness or security defect in the SQL or the demo. All were fixed. In the build: exercise file 05 could not be run twice (exercise 25's index was never dropped, so a second run printed `SEARCH` under "without an index" and then failed), so it now has `DROP INDEX IF EXISTS`, and the 04/05 and README headers say to rebuild before every run; exercise 21's deliberate error and exit status 1 are now explained; `demo.ts` printed `BLOCKED` for a fixed query that returned `[]`, so it now asserts controls (the fixed search finds the same tasks as the vulnerable one for `Fix`, and `zzz` finds nothing), checked by breaking `fixed.ts` two ways; `group_concat` order, the `MIN/MAX/AVG` comment, the SQLite 3.15 requirement for row values, and a `schema.sql` header that did not say a re-run wipes data were corrected. In the documents: the Stage 13 `git add` paths did not match the learner's working directory, the quoted keyset plan said `COVERING INDEX` when the exercise's query is not covering (a check of mine had used `SELECT id` only), the `NOT NULL` `ALTER` error only appears on a table with rows, the write-up's demo block was a condensed rewrite and is now the real output, two Concepts essays restated stages and were cut back, and three asides about the author's own build were rewritten as general warnings. The embedded code blocks were re-synced with the build files and the guide was followed again in an empty directory.

## [BACKLOG] Status of the PRs and the checkpoint

- The build repo's PR #6 was merged on 2026-10-21 (merge commit `7f88f42`), and its link and `git log` are in `write-up.md` Checkpoint evidence. As in M05, the author's PR link belongs in `write-up.md` only.
- `write-up.md` has no `GAP:` left. Its last one, the trainer's checkpoint result (random exercises run and explained), was removed rather than filled: `content-standards.md` defines Checkpoint evidence as a merged PR link, a file that exists and an explanation in your own words, and all three are in the write-up. The trainer's check is the trainer's act, and no result has been recorded, because none was given. If the trainer's result should be in the write-up, it has to come from the trainer.
- The content repo (`write-up.md`, `training-material.md`, `tasks.md`, this file) is committed on `feat/module-06/write-up-and-training-material`. Its PR is opened after a final review, which found no critical or high issues and two low ones (this stale line, and the install-warning count), both fixed in the last commit.

## [BACKLOG] Not verified

- The ERD was parsed by Mermaid 11 and matched against the schema, but it was not rendered on GitHub. Look at it on the PR.
- `npm install` printed an `allowScripts` warning for `better-sqlite3` and `esbuild` (npm 11.19.1). `better-sqlite3` loaded anyway through a prebuilt binary. A machine with no prebuilt binary for its platform was not tested.
- Injection is shown with a `LIKE` search through `better-sqlite3`. Binding a column name or an `ORDER BY` term (which a `?` cannot do) is described in the Concepts section but was not built or run.

## [BACKLOG] Decisions made without the EM (EM-owned files, conventions)

- `tasks.md` is EM-authored. Its checkboxes were ticked, as in M05 (commit `513608f`), and nothing else in it changed. The EM may prefer the file left untouched.
- The demo uses `.ts` import specifiers, following M04 and M05. The program's import-extension convention (P2) is still open, and the guide says which one it uses.
- `tasks.id` is `INTEGER PRIMARY KEY`, while M05's in-memory store uses UUID strings. M07 will have to decide how the ids map.
- `better-sqlite3` was chosen over the built-in `node:sqlite` because it is in the pinned stack and is M07's driver. `node:sqlite` was not tried.
