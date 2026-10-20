# M03: backlog

Items found in VALIDATE and in the `/code-review` of both repos (2026-10-20). The Medium and Low-medium review
findings, and the stale curl evidence in the write-up, were fixed before the commit. PR #3 was merged on
2026-10-20, and the VS Code debugger session was captured from real screenshots, so there are no unresolved
`GAP:` markers left in `write-up.md` or `training-material.md`. What remains is below, and none of it blocks
the build or the documents.

## [BACKLOG] The training material is untested on Windows

The commands are for macOS or Linux shells. Specific risks: `/usr/bin/time -l` (macOS) against `-v` (Linux)
was not tried on Linux; `shasum -a 256`, `wc -l`, `head`, `ls`, and bash/zsh process substitution
(`diff <(...) <(...)`) have no direct Windows equivalent; the stuck-read self-check uses `/dev/stdin` and is
skipped on Windows; and the `node -p` one-liners use double quotes. The solution design says trainers dry-run
each module on every OS in the cohort.

## [BACKLOG] The guided exercises have not been tried by someone who did not build the stage

Stages 4, 5 and 8 are exercises with a scaffold, up to four hints, self-checks and a collapsed reference.
The references and checks were verified by running them (27 checks pass from a clean `npm ci`), the Stage 8
scaffold was confirmed to fail fast (a `TimeoutError` after 5 s) when the request handler never answers, and the
documented numbers (file size, line count, both SHA-256 values, exit codes, the 32 MB and 64 MB heap-cap
behaviour) were re-checked in VALIDATE. What has not been tested is whether a learner can finish from the
hints. Run a cold read with someone who did not build it.

## [BACKLOG] The 6-hour scope has not been timed, and the guide is long

The brief allocates 6 hours. The build was not timed. `training-material.md` is about 2,430 lines (down from
2,706; about 10,900 words outside code, down from about 13,100 after a second readability pass that left every code block
unchanged) with 12 stages, three of them full exercises, a ~100 MB data step, an optional 600 MB step and an npm lab.
About 890 of those lines sit in collapsed `<details>` blocks (the three self-check files, the hints and the
reference implementations), so the visible path is shorter than the line count. A "Start here" table in section 1
marks each stage Core or in-scope-but-shorter, Stage 9 was cut from 10 steps to 7 and Stage 10 to about half, and
`write-up.md` went from 751 to about 620 lines. A further cut would mean removing code a learner needs to finish alone.
Record how long the first cohort takes per stage, and consider moving Stage 9 (npm lab) or the optional
experiments out if the total does not fit.

## [BACKLOG] The Concepts section contains background claims that were not measured

`training-material.md` Concepts is marked *(background)* where it states standard Node and npm knowledge that
the build did not prove: the four stream kinds, `pipeline` destroying streams on failure, the event-loop phase
order, why ES modules reverse the `nextTick` and promise order, which work uses the thread pool, the semver
promise, the supply-chain threat model, and where Node fits. The write-up itself contains only what was
measured. The EM should check the marked points against the Node and npm documentation.

## [BACKLOG] `node --run` does not forward signals to the process it starts

Observed on Node 24.22.0 / npm 11.19.1: sending `SIGTERM` or `SIGINT` to the `node --run serve` process alone
left the server running (orphaned, port still bound). `npm run` and a plain `node` shut down cleanly, and
Ctrl+C (a signal to the whole group) was clean for both runners. The cause was not investigated, and the
documents state only what was observed. Worth checking before the guide recommends `node --run` for anything
long-running.

## [BACKLOG] Node 24 detects ES module syntax in a `.js` file with no `package.json`

A `.js` file with `import` syntax ran as an ES module, silently, with no `package.json` anywhere above it. A
`.js` file with no `import`/`export` ran as CommonJS. `--no-experimental-detect-module` restores the old
failure. This does not make M02's `.mjs` choice wrong, since `.mjs` is explicit and works on any Node version.
It is relevant to the open question in the M02 backlog about introducing `"type": "module"` earlier. M03
uses a `package.json` with `"type": "module"` in its own folder, which satisfies `coding-standards.md`, and it
does not settle P2 (`.ts` against `.js` import extensions).

## [BACKLOG] `npm install` warns about install scripts but still runs them

With npm 11.19.1, installing a local package with a `postinstall` printed an `npm warn install-scripts`
message and ran the script anyway. `--ignore-scripts` and `--strict-allow-scripts` both prevented it, and
`ignore-scripts` and `strict-allow-scripts` are `false` by default. The training material teaches this as
observed behaviour. The EM may want to decide whether the program recommends a project-level `.npmrc` setting
for later modules.

## [BACKLOG] Graph community labels for M03 are unreliable

`graphify query` places the M03 training material under the "M02 — Modern JavaScript & Async/Await" and "M10 —
Capstone Project" communities, and the M03 brief under "M04 — TypeScript Basics...". The edges are sensible
(each file connects to its own headings), but the labels do not match the module folders. This is the same
issue as the M02 backlog entry. Use edges and file paths until the graph is rebuilt.

## Open findings from the code review of `node-js-foundation-training` (all Low)

- **A client aborting mid-body logs a stack trace and tries a 500.** `notes-server.js`: `for await (const chunk of
  req)` throws `Error: aborted` (`ECONNRESET`), which is not an `HttpError`, so the handler logs it as a server bug
  and writes a 500 to a dead socket. Skip the log when `req.destroyed` or the code is `ECONNRESET`.
- **`parseLine` accepts non-timestamps.** `aggregate.js`: `Date.parse` accepts `1`, `2026`, `Jan-1` and
  `2026-02-31T00:00:00Z`, so a report can contain `"firstTimestamp": "1"`. An ISO regex would fix it, and no check
  covers it.
- **The "memory does not depend on file size" claim has two holes.** A 100 MB file with no newline made
  `readline` buffer the whole line (peak 416 MB, against 85 MB for the normal log), and `byPath` grows with the
  number of distinct paths. State the assumption in the guide, or cap line length and distinct paths.
- **`generate-log.js` with a bad `--out`** (a directory or an unwritable path) ends in an uncaught `EISDIR` dump
  instead of the one-line, exit-2 message its other bad arguments get.
- **Smaller server details.** The notes `Map` is unbounded and `GET /notes` is unpaginated. `HEAD /notes` returns
  405. The content-type check (`startsWith('application/json')`) also accepts `application/jsonp`. The server
  binds all interfaces while its log line says `localhost`. The chunked and declared-length 413 paths differ on
  `Connection: close` on purpose (see the write-up), and the reason the chunked path needs it was not found.
- **Very low.** The default `report.json` is relative to the current directory, so running the CLI from the repo
  root leaves a `report.json` that the root `.gitignore` does not cover (the header says to run it from the
  module folder). A signal that arrives after reading has finished but before or during `writeFile` still
  produces the report and sets exit code 130.

## Open findings from the code review of `node-js-training-rnd` (all Low)

Fixed in the "improve both files" pass (2026-10-20): the Concepts preamble now separates `(background)` from
`(reference build)` points and marks the `npm ci`, `npx`, `require()` of an ES module and `Expect: 100-continue`
claims; the `pipeline` row in the Stage 2 reference table is now a one-liner that runs; the Stage 11 Done-when
checks that were not in the brief, and Stage 2's "prediction written down" check, are gone; "Before you start" now
names the M01 root `.gitignore` and `.env.example`, and Stage 12 says Stage 9 Step 7 creates the lockfile; the
"Node 22" asides and the two author-mistake pitfall rows are removed.

- **The brief says "Both merged PRs" (plural), and Stage 12 opens one PR with three commits.** The EM should
  confirm which is expected, because the Friday-gate evidence differs.
- **One Done-when check is still not in the brief.** The fourth item in section 6 (27 `ok` lines and a clean
  `npm audit`) is kept, now labelled "Supporting evidence for the first two". The brief's Definition of done has
  three items.
- **The new Concepts quick reference is mostly `(background)`.** Its rows for the core APIs, `process`, flags,
  `package.json` and the npm commands state standard Node and npm knowledge that the guided steps only partly
  exercise. The EM should check them against the Node and npm documentation, like the marked points above.
