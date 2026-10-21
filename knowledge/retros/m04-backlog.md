# M04: backlog

Items found in VALIDATE and in the `/code-review` of both repos (2026-10-21). The `/code-review` reported 17 findings (7 in the
build repo, 10 in the documents) and all 17 were fixed before the commit: `applyTaskUpdate` now names each field instead of
spreading, `toError` guards `String()`, `readEnv` trims and accepts only whole digits, two type-level assertions and four runtime
asserts were added (each confirmed to fail when the old code is put back), the two M02 timing bounds were widened, the `demo` script
is a single Node process, and in the guide Stage 5 names both mentions of the old file name, Stage 8 no longer runs a script that needs a
Stage 9 file and shows the error message it teaches, Stage 10 times the binaries directly, the Concepts essays no longer restate the
stages, and the file names, legend and small slips were corrected. Every learner-facing command that changed was re-run from a clean
state. One FIX NOW was fixed earlier in VALIDATE: the Stage 3 "Done when" said
errors appear only on lines marked `// error`, but five error lines carried no marker (Core 3 lines 15-16, Core 4
`Pt`, Core 5 `_wrong`, Core 6 `import`). They are now marked, the wording names the `never` lines too, and the
compiler output was re-run and is identical. A last review before this repo's PR found no critical or high findings and four text
problems, all fixed: this file described the documents as uncommitted, the M04 Next step called the Task API field list an open
decision although the M05 brief fixes it, the size figures below were stale, and one Concepts sentence contradicted its own table.
Nothing in this file blocks the build or the two documents.

The build repo's PR #4 was merged on 2026-10-21 (merge commit `aa0aac0`). Its link and `git log` are in `write-up.md` Checkpoint
evidence, and the last `tasks.md` box is ticked. As in M03, the author's PR link belongs in `write-up.md` only: `content-standards.md`
says it is not carried into `training-material.md`.

## [BACKLOG] The brief's wording differs from what was observed (EM-owned files, not edited)

`brief.md` and `tasks.md` are EM-authored, so they were not changed. For the EM to confirm:

- **Native type stripping on Node 24.** The brief calls it awareness-level because full stability arrives with Node 26.
  On Node v24.22.0 it ran all 40 checks with no flag and no warning. The reference build did not check Node's documented
  stability label, only the behaviour.
- **"Strict by default" as a 6 → 7 change.** TypeScript 6.0.3 already reported the same errors as 7.0.2 for a config with no
  `strict`. What did change is that legacy options went from deprecated (`TS5107` / `TS5101`) to removed (`TS5108` /
  `TS5102`), and the programmatic API (`createProgram`, `transpileModule`) is no longer exported from the main entry.
- **`types` is required.** In 7.0.2 `@types/*` packages are not picked up automatically: without `"types": ["node"]` the
  reference project gave 33 errors. The brief's `tsconfig` essentials list does not mention `types`.

## [BACKLOG] `CreateTaskInput` is a deny-list, and `Task` has two fields the API does not

`CreateTaskInput` uses `Omit`, so a new `Task` field becomes client-writable by default (shown with a `dueDate`
experiment in the write-up and the Concepts essay). `Pick` would fail safe. The choice was left as built and documented,
not changed after the fact. Also, the M05 brief ("Resolved: the Task API contract") fixes the API's fields as `id`, `title`,
`description?` and `status`. `createdAt` and `updatedAt` were added in M04 so there were server-owned fields to omit. They are
beyond the contract, so M05 must decide whether `Task` keeps them and whether the API exposes them. The M04 Next step says so.
`arch-summary.md` still lists the Task API spec (P3) as "Proposed", which disagrees with the M05 brief. `coding-standards.md`
says the brief wins, and the EM may want to bring the summary into line.

## [BACKLOG] The training material is untested on Windows

Commands are for macOS or Linux shells. Specific risks: `ln -s` (needs developer mode or admin rights on Windows),
`sed '/"types"/d'`, `grep -nE`, `printf` with `\n`, `echo '...' > file` quoting, `cp -R`, and the `for f in ...; do ...; done`
loop. `nvm use` is `nvm-windows` there and reads `.nvmrc` differently. The solution design says trainers dry-run each
module on every OS in the cohort.

## [BACKLOG] The guided exercises have not been tried by someone who did not build the stage

Stages 4, 6 and 7 are exercises with a scaffold, hints and a collapsed reference. In VALIDATE the scaffolds were confirmed
to type-check as given, the references and all three check files were re-run (typecheck exit 0, 40 checks under `tsx`,
40 under native Node, a clean `npm ci`), and the commands in Stages 2, 3, 8 and 9 were run in a copy of the project
with the output compared to the guide. Stage 10's comparisons ran earlier in a scratch folder with equivalent
commands. Only its API check and its timing step (now with the binaries called directly) were re-run as written, so its
`~/ts6-probe` install and config steps are the least-verified part. What has not been tested is whether a learner can finish from the hints. Run a
cold read with someone who did not build it.

## [BACKLOG] The 8-hour scope has not been timed, and the guide is long

The brief allocates 8 hours across two days. The build was not timed. `training-material.md` is 2,431 lines (about 18,900
words including code) with 11 stages, three of them full exercises and one (Stage 3) of six practice files. About 790 of
those lines sit in 18 collapsed `<details>` blocks (the three check files, the hints and the reference implementations), and a
"How to use this guide" table marks which stages are core. `write-up.md` is 670 lines. Record how long the first cohort
takes per stage. Stage 10 (TypeScript 6 against 7) needs a network install of `typescript@6` and is marked in-scope but
shorter, so it is the first candidate to trim.

## [BACKLOG] Version-specific compiler messages and background claims

The quoted compiler messages and error codes are from TypeScript 7.0.2 (and 6.0.3 for Stage 10), `tsx` 4.23.15,
`@types/node` 24.19.2 and Node v24.22.0 on macOS. Wording can change in other patch versions. The Concepts section marks
as *(background)* the points the steps do not prove: `skipLibCheck` and `allowImportingTsExtensions` behaviour,
`JSON.parse` being typed as `any`, why a stripper leaves a type-only import behind, that the native compiler checks in
parallel, and which tools to use for what. The EM should check the marked points against the TypeScript and Node docs. The
write-up itself contains only what was run, and says so where it did not run something (Node's stability label, a real
`tsc` build of the project, Jest and M09).

## [BACKLOG] Candidate additions to `knowledge/rules/coding-standards.md` (not made)

Two constraints came out of the build that the standards do not state: `types: ["node"]` must be listed explicitly under
TypeScript 7, and `erasableSyntaxOnly` plus `verbatimModuleSyntax` plus `.ts` specifiers are what let native type
stripping run the code. They were not added, because the open decision P2 covers import extensions and editing the rules
was outside this stage's plan. Raise them with the EM when P2 is settled.
