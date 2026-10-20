# M03 Write-Up — Node.js Runtime & npm

> Written from what I captured as I built it: raw notes after each slice, then this document. Written
> so a trainee starting this stage next cohort could follow my path on their own.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo, [`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
in `modules/m03-nodejs-runtime-npm/`, on the branch `feat/module-03/runtime-and-npm`, merged to `main` by
[pull request #3](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/3) (see Checkpoint
evidence). The module folder is laid out by deliverable:

```text
m03-nodejs-runtime-npm/
├── package.json
├── package-lock.json
├── data/                      (the generated log, git-ignored)
└── src/
    ├── log-report/
    │   ├── generate-log.js
    │   ├── aggregate.js            aggregate.check.js
    │   └── log-report.js           log-report.check.js
    └── http-server/
        ├── notes-server.js
        ├── server.js               server.check.js
```

- **A log generator**, `generate-log.js`. The trainer was meant to hand out the ~100 MB log file. I wrote
  the generator instead, so the next cohort can make the same file themselves. It is seeded, so the same
  seed always gives byte-identical output.
- **The `log-report` CLI**, `log-report.js` plus `aggregate.js`. It streams a log file line by line,
  counts what it sees, and writes a JSON report. Options come from `parseArgs`.
- **The bare `node:http` server**, `notes-server.js` plus `server.js`. Three JSON endpoints on an
  in-memory notes store: `GET /notes`, `POST /notes` and `GET /notes/:id`. Routing, body parsing and
  error handling are all written by hand.
- **Three check files**, each next to the code it checks: 27 `node:assert` checks in total (6 + 9 + 12).
  All 27 pass.
- **Project files**: `package.json` (`"type": "module"`, `engines` for Node 24, scripts),
  `package-lock.json` and `.env.example` (documenting `PORT`). The module has no `.gitignore`: the repo
  root's `.gitignore` ignores the generated `data/` folder and `report.json`. The root `.env.example` is an
  index that points to each module's own example.

The research experiments (event loop, ESM, `--env-file`, `npm` and so on) were throwaway scripts in a
scratch folder outside the repo. Their results are in step 8 of "How to build it".

Run everything from `modules/m03-nodejs-runtime-npm/`, on Node 24:

```bash
nvm use
npm run generate-log   # about 100 MB into data/access.log, in ~2 s
npm run report         # writes report.json
npm run serve          # server on http://localhost:3000
npm run check          # all 27 checks
```

Environment: Node v24.22.0 and npm 11.19.1 (`.nvmrc` pins Node 24), macOS 26.5, Apple silicon, 10 CPU
cores, zsh. My default shell started on Node 22, so I had to run `nvm use` in each session.

## Why it's built this way (key decisions)

**Streaming, and why this approach**

- The CLI reads with `createReadStream(input, { encoding: 'utf8', signal })` wrapped in
  `readline.createInterface({ input, crlfDelay: Infinity })`, and `aggregateLines` consumes it with
  `for await (const line of lines)`. I chose `readline` because it hands me one finished line at a time,
  and a stream only produces more data as fast as the loop asks for it. I did not benchmark alternatives
  such as a hand-written `Transform` that splits lines.
- `aggregateLines` accepts any async iterable of lines, so the checks feed it a plain array or an async
  generator. It keeps only counters. Its memory depends on how many distinct statuses, methods and paths
  exist (a few dozen), never on how many lines the file has.
- I split `aggregate.js` (counting) from `log-report.js` (flags, signals, exit codes, writing the file).
  The counting is easy to check with an array, and the CLI is checked by running it as a child process.

**What would have broken if I had loaded the file whole**

I measured it with a throwaway script (not in the repo) that does `readFile(path, 'utf8')`, splits on
`\n`, then runs the same `aggregateLines`:

| Run | Result |
| --- | --- |
| Streaming CLI, 100 MB file | peak 84.7 to 85.8 MB, about 0.95 s |
| Whole-file version, 100 MB file | peak **821 MB**, 1.26 s, about 9.7 times the streaming run |
| Whole-file version, 100 MB file, `--max-old-space-size=64` | `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory`, exit 134 |
| Whole-file version, 600 MB file | `RangeError: Invalid string length`, because `buffer.constants.MAX_STRING_LENGTH` is 536,870,888 (about 512 MiB) |
| Streaming CLI, 600 MB file, `--max-old-space-size=32` | succeeded, 5.7 s |

So whole-file reading costs about ten times the file size in memory at 100 MB, and past about 512 MiB it
fails outright whatever RAM the machine has. The streaming version finishes a 600 MB file under a 32 MB
heap cap.

The streaming run is not perfectly flat: peak memory was about 85 MB for the 100 MB file and about 117 MB
for the 600 MB file. I do not think that is retained data, because the 600 MB run still passed under the
32 MB cap. `--trace-gc` showed 649 Scavenges and 0 Mark-Compact collections uncapped, and 475 Scavenges
and 174 Mark-Compact collections under the cap. My reading is that the extra memory is short-lived buffers
V8 has not yet given back, but I only measured the collection counts, not that cause. A bare `node -e 0`
already peaks at about 43.7 MiB.

**CLI decisions**

- Options: `--input/-i`, `--output/-o`, `--top/-n`, `--stats` and `--help/-h`. The default input is
  `data/access.log` at the module root, found with `import.meta.dirname`. The default output is
  `report.json` in the current directory.
- Exit codes: 0 for success, 1 for a failed run (such as a missing input), 2 for a bad command line, 130
  for SIGINT and 143 for SIGTERM. I set `process.exitCode` instead of calling `process.exit()`, so output
  is flushed.
- The report stays deterministic on purpose, so elapsed time and peak memory go to stderr behind `--stats`
  and not into the JSON. Every full run of the default file gives the same report hash.
- A signal aborts the read stream through an `AbortController` and writes no report. A second signal
  re-raises itself (see "What tripped me up", item 2).
- A malformed line is counted, not fatal. Blank lines are skipped and counted as neither good nor bad.
- A missing input and an output folder that does not exist are both `ENOENT`, so the CLI remembers which
  step it was in: `cannot read <input>` or `cannot write <output>`, exit 1. A code review found that my
  first version blamed a read for a failed write.

**The log format and the generator**

- One record per line: `<ISO timestamp> <METHOD> <path> <status> <duration ms>`, for example
  `2026-01-01T00:00:00.037Z GET / 200 856`. About one line in a thousand is deliberately garbled, so the
  CLI has a real failure case to cope with.
- `Math.random` cannot be seeded, so the generator uses a small seeded generator (mulberry32). I generated
  the file twice and the SHA-256 matched both times.
- The generator also streams: about 64 KB per chunk, and it waits for `drain` when `write()` returns
  `false`. Without that wait, the whole file would pile up in memory while the disk catches up.
- Status codes are independent of routes, so you will see `POST /api/tasks 200`. That is fine for
  synthetic data.

**The bare server: routing and body parsing**

I wrote a naive version first on purpose, saw what broke (see "What tripped me up"), and then added only
what the failures asked for.

- *Routing.* The path is split into segments and compared by hand. `route(method, segments)` returns a
  handler or throws: a 404 for an unknown path and a 405 with an `Allow` header for a known path with the
  wrong method. Empty segments are dropped, so `/notes` and `/notes/` are the same route. Node 24 has a
  global `URLPattern`; I hand-rolled the matching because the brief asks for it.
- *Body parsing.* `readJsonBody` checks `Content-Type` (415), refuses a declared `Content-Length` over
  1 MiB (413), counts bytes as chunks arrive for bodies with no length (413), and turns a `JSON.parse`
  failure into a 400. Bodies are collected as `Buffer` chunks and decoded once, so a multi-byte character
  split across two chunks is not corrupted.
- *Errors.* Handlers throw an `HttpError` (status, message, headers). One `try/catch` around the whole
  request handler turns it into `{ "error": { "status", "message" } }`. Anything else becomes a logged
  500 with a generic message.
- *Validation.* `text` must be a non-empty string of at most 1000 characters. I use `body?.text`, because
  a body of `null` is valid JSON and would otherwise throw a `TypeError`.
- *Structure.* `notes-server.js` exports `createNotesServer()` and never calls `listen`. `server.js` reads
  `PORT`, listens, and shuts down on `SIGINT` and `SIGTERM`. The checks listen on port 0.
- *Ids and URLs.* Ids come from `crypto.randomUUID()`. The path is parsed from a fixed prefix
  (``new URL(`http://localhost${req.url}`)``), because the `Host` header is client input. It is a prefix
  and not a base: with `new URL(req.url, base)` a target starting with `//` is read as a host, and a code
  review found that `GET //` then threw and answered 500. A target that does not start with `/` is a 400.
  I never decode the id, so a malformed `%` sequence cannot throw.
- *Shutdown.* `server.close()` stops new connections and, on Node 24, closes idle keep-alive ones. A
  connection that is busy when the signal arrives only goes idle after its response, so `server.js` also
  runs a 100 ms `closeIdleConnections()` sweep until the server has closed. A code review reproduced the
  problem: a POST in flight at `SIGTERM` was answered with a 201, and then the process waited for the 5 s
  timer and exited 1. The shutdown check now sends `SIGTERM` while a request is half-sent.

**Other decisions**

- *`package.json` lives in the M03 folder, with `"type": "module"` and `.js` files.* This stage is plain
  JavaScript, so it does not settle the open P2 question (`.ts` versus `.js` import extensions). M02 used
  `.mjs` files because the repo had no `package.json`, and that does not carry over.
- *No dependencies.* Everything uses `node:` modules. The lockfile has only the root entry, and `npm
  audit` reports 0 vulnerabilities.
- *Scripts load the module's own `.env` if it exists* (`node --env-file-if-exists=.env`). npm runs a script
  from the folder that holds `package.json`, so that is `modules/m03-nodejs-runtime-npm/.env`. The server
  falls back to port 3000 in code. The build repo's README says each stage that needs configuration ships
  its own `.env.example`, so the example sits with the code that reads `PORT`.
- *The root `.env.example` is an index, and there is one `.gitignore`, at the root.* The root
  `.gitignore` already ignores `.env` in any folder, and I added the two generated paths for this module
  only, so a `data/` folder in a later module is not hidden by accident. One `.gitignore` can cover every
  module, and one `.env.example` cannot, because each module needs different variables.
- *The checks fail fast.* A code review found checks that could hang. Child processes are now awaited with
  a timeout, every request in `server.check.js` has a 5 s limit, the server check fails at once if
  `server.js` exits before it is listening, and the stuck-read check asserts `signalCode` as well as
  `exitCode`. The signal checks wait 0.7 s before signalling, on a 300 MB log (so the run lasts several
  seconds), because the CLI prints nothing the check could wait for.
- *Layout by deliverable.* I first built the module flat, then moved it into `src/log-report/` and
  `src/http-server/`, because they are the two deliverables the brief asks for. They share one
  `package.json` and one lockfile, since both use only `node:` modules. After the move, the default log
  path had to change (`import.meta.dirname` is now two folders down).
- *Every file opens with a short header in the M01 and M02 style:* `Module 3 output`, which of the two
  deliverables it belongs to, how to run it and what success looks like (or "Nothing to run here"). The two
  longest files also have `// ---- ... ----` section dividers, as `fs-refactor.mjs` does in M02.

## How to build it (teach it to the next trainee)

I use the streaming CLI as the worked example, because the easy version of it quietly loads the whole
file. The other steps follow in the order I did them.

Start each session with `cd ~/node-js-foundation-training/modules/m03-nodejs-runtime-npm`, run `nvm
use` on its own line, and check that `node --version` prints v24.

**0. Branch and folder.** `git switch main`, `git switch -c feat/module-03/runtime-and-npm`, `mkdir -p
modules/m03-nodejs-runtime-npm`. Every stage is delivered from a feature branch.

**1. Folders, `package.json`, `.env.example`, root `.gitignore` and `.env.example`.**
- *Do:* `mkdir -p src/log-report src/http-server`. Write a `package.json` with `"private": true`,
  `"type": "module"`, `"engines": { "node": ">=24" }` and the scripts `generate-log`, `report`, `serve`,
  `dev` and `check`, each pointing into `src/`, plus an `.env.example` containing `PORT=3000`. In the repo
  root, add the two generated paths to `.gitignore` and make `.env.example` an index.
- *Why:* `npm init -y` in npm 11 wrote `"type": "commonjs"`, so the module type has to be set by hand.
  `data/` is 100 MB and reproducible, so it is regenerated, never committed.

**2. The generator, `generate-log.js`.**
- *Goal:* a reproducible log of about 100 MB without holding it in memory.
- *Do:* `parseArgs` for `--out`, `--size-mb` and `--seed`; a seeded `random()`; routes and statuses picked
  from weighted lists; lines built into about 64 KB chunks; `output.write(chunk)`, and when it returns
  `false`, `await once(output, 'drain')`.
- *Result:* `npm run generate-log` printed `Wrote 2231981 lines, 100.0 MB ... in 1.8s (seed 1)`. The file
  is 104,857,613 bytes with SHA-256 `e28d11f85bdaef172e5e6ce3ea4413f7b145649399e3dde05f696d689fadecd8`.

**3. The counting, `aggregate.js`.**
- *Goal:* fold a stream of lines into counts, with no knowledge of files or flags.
- *Do:* `parseLine(line)` splits on spaces and returns the fields or `null` (five fields, a parseable
  timestamp, an upper-case method, a path starting with `/`, a status from 100 to 599, a non-negative
  whole duration). `aggregateLines(lines, { top })` loops `for await`, skips `''`, counts a `null` parse
  as malformed, and updates `Map` counters for status, status class, method and path, plus a duration sum
  and maximum and the first and last timestamps. It returns plain objects with sorted keys and the `top`
  busiest paths (by count, then path).
- *Why:* nothing in it grows with the number of lines, which is the whole point.
- *Check:* `aggregate.check.js` has 6 checks.

**4. The CLI, `log-report.js` (the worked example).**
- *Goal:* stream the file, honour the options and signals, and write the report.
- *Do:* parse the options in a `try/catch`, then run:
  ```js
  const controller = new AbortController();
  const source = createReadStream(input, { encoding: 'utf8', signal: controller.signal });
  // readline hands over one line at a time, so the file never has to fit in memory.
  const lines = readline.createInterface({ input: source, crlfDelay: Infinity });
  const counts = await aggregateLines(lines, { top: topCount });
  await writeFile(output, JSON.stringify({ source: path.basename(input), ...counts }, null, 2) + '\n');
  ```
  Validate `--top` as a whole number of at least 1 (exit 2), map `ENOENT` to a one-line message (exit 1),
  and install `SIGINT` and `SIGTERM` handlers that set `process.exitCode` and abort the controller. On a
  second signal, remove the handlers and re-send the signal.
- *Why:* the usual first idea is `readFile` then `split('\n')`. That works on a 1 MB sample and fails
  quietly in memory on the real file, which is why the check has to use the full file.
- *Result:* on the 100 MB file, `npm run report -- --stats` printed `Wrote report.json: 2231981 lines,
  2200 malformed` and `stats: 0.93s elapsed, peak memory 84.8 MB (maxRSS)`.
- *Measure memory two ways.* `--stats` prints `process.resourceUsage().maxRSS` (in KB). For a second
  opinion, `/usr/bin/time -l node src/log-report/log-report.js ...` on macOS prints `maximum resident set
  size` in bytes (89,128,960, which is 84.99 MiB). They agree. On Linux, `/usr/bin/time -v` prints the
  same figure in KB.
- *Check:* `log-report.check.js` has 9 checks. It runs the real CLI as a child process and covers the
  report, `--top`, `--stats`, a missing input and output folder, an unknown flag, a bad `--top`, `--help`,
  SIGINT on a 300 MB generated file (exit 130, no report) and a second SIGINT on a stuck read.

**5. The bare server, first version.**
- *Do:* the first thing that comes to mind in `notes-server.js`: `http.createServer(async (req, res) =>
  { ... })`, an `if` per method and path, `for await (const chunk of req) body += chunk`, `JSON.parse`,
  `res.writeHead` and `res.end(JSON.stringify(...))`. `server.js` reads `PORT` and calls `listen`.
- *Result:* the happy path worked (`POST /notes` gave 201 and `GET /notes` listed the note). Two things
  were wrong at once: an unknown id returned `200` with an empty body, and one malformed JSON body killed
  the whole process (see "What tripped me up", items 3 and 4).

**6. The bare server, hardened.** In this order, because each failure asked for it:
1. An `HttpError` class and a `sendJson(res, status, body, headers)` helper.
2. One `try/catch` around the whole handler that sends the `HttpError`, or a logged 500.
3. `route(method, segments)` with 404 and 405 (`Allow`).
4. `readJsonBody(req)` with 415, 413 (by length and by counting) and 400.
5. Validation of `text`, and a `Location` header on the 201.
6. In `server.js`: validate `PORT`, handle the `listen` error event, and shut down on `SIGINT` and
   `SIGTERM` with `server.close()`, a 100 ms `closeIdleConnections()` sweep and a 5 s forced-exit timer.

*Check:* `server.check.js` has 12 checks: eleven over real HTTP on `server.listen(0)`, and one that spawns
`server.js` with `PORT=0`, sends `SIGTERM` while a request is half-sent, finishes it, and expects a 201 and
a prompt exit 0.

**7. Run all the checks and try the failure cases by hand.** `npm run check` exited 0 with 27 `ok` lines.
With the server running: `curl -s -i -X POST localhost:3000/notes -H 'Content-Type: application/json' -d
'{"text": oops'` gave 400 with the server still up, `curl -i localhost:3000/notes/nope` gave a 404 JSON
body, and `curl -i -X DELETE localhost:3000/notes` gave 405 with `Allow: GET, POST`.

**8. Research by experiment (scratch folder, outside the repo).**
Results are from Node v24.22.0 and npm 11.19.1.

*Event loop and thread pool*
- Ordering of sync code, `setTimeout(0)`, `setImmediate`, `process.nextTick` and a resolved promise: in a
  `.mjs` file `1 sync start, 2 sync end, 4 promise microtask, 3 nextTick, 6 setImmediate, 5 setTimeout 0`;
  in a `.cjs` file `1, 2, 3 nextTick, 4 promise microtask, 6, 5`. Over ten ESM runs `setImmediate` came
  first nine times, so that pair has no guaranteed order from the main module.
- Blocking: a `setInterval` every 100 ms ticked at 101, 202, 304 ... 911 ms. With a 500 ms busy loop
  started at 250 ms, ticks were at 101, 202, 751, 852 and 953.
- Thread pool: eight `crypto.pbkdf2` jobs (300,000 iterations, sha512) finished at 380 ms with
  `UV_THREADPOOL_SIZE=1`, 111 ms by default (4 threads) and 79 ms with 8.

*Modules, environment and flags*
- In an ES module `typeof require` and `typeof __dirname` are `undefined`; `import.meta.dirname` and
  `import.meta.filename` work, and so does top-level `await`. `require()` there throws `ReferenceError:
  require is not defined in ES module scope`.
- A `.js` file with `import` syntax and no `package.json` ran on Node 24, which detected the syntax
  itself. `require('./lib.mjs')` of an ES module also worked.
- `require('fs') === require('node:fs')`, but `require('test')` fails with `MODULE_NOT_FOUND` while
  `require('node:test')` works.
- `node --env-file=.env`: `PORT` arrived as the string `"4321"`, a quoted value containing `#` stayed
  whole, an empty `EMPTY=` was `""`, and a real environment variable beat the file. A missing file stops
  with `node: nope.env: not found`, exit 9. `--env-file-if-exists` runs without it.
- `node --watch` restarted a script on edit. `node --run <script>` runs a `package.json` script with no
  `> ...` header.
- Exit codes: `process.exit(3)` gave 3, an uncaught `throw` 1, `process.exitCode = 4` gave 4, an unhandled
  rejection 1. `process.argv.slice(2)` gave `['a', '--flag', 'b']`.
- `structuredClone` kept a `Date`, a `Map` and a circular reference; `JSON.stringify` of the same object
  threw a `TypeError`; cloning a function threw a `DataCloneError`.

*Core modules*
- `events`: `emit('error')` with no listener throws. `timers/promises`: `sleep(50)` took about 51 ms, and aborting a
  `sleep(5000)` rejected with `AbortError`, code `ABORT_ERR`.
- `stream`: `pipeline(Readable.from(['a','b','c']), upperCaseTransform, sink)` produced `ABC`. A file
  stream's default `highWaterMark` measured 65,536 bytes, and `highWaterMark: 64` turned a small script
  into 47 chunks.
- `path.join('a','..','b','c.txt')` is `b/c.txt`; `path.extname('a/b.tar.gz')` is `.gz`; `randomUUID()`
  returns 36 characters with `4` at index 14.
- `node:test` and `node:sqlite` both exist on Node 24.22.0. This course uses Jest and Prisma instead.

*Debugging*
- `node --inspect=9333` printed `Debugger listening on ws://127.0.0.1:9333/<id>`, and
  `http://127.0.0.1:9333/json/list` returned a target.
- **Over the inspector protocol.** I could not click through VS Code from a terminal, so I attached to the
  inspector endpoint directly with a throwaway script (Node 24's global `WebSocket`). I ran `node
  --inspect=127.0.0.1:9229 src/http-server/server.js` with `PORT=0` and set a breakpoint on line 78 of
  `notes-server.js`, `const note = notes.get(id);` in `getNote`. A `POST /notes` did not pause. A `GET
  /notes/<id>` paused at once, with top frame `getNote` at 78:18. Evaluating in that frame gave `id`,
  `notes.size` and `req.method`, but `note` gave `ReferenceError: Cannot access 'note' from debugger`,
  because the line had not run yet. After one step over, `note.text` was `debug me`. After I removed the
  breakpoint the same `GET` returned `200` with no pause. `--inspect-brk` printed nothing until a debugger
  attached, and the first pause was `Break on start` at line 1.
- **In VS Code itself** (1.142.0, built-in `js-debug` 1.140.0). I created `.vscode/launch.json` with the
  attach snippet (`"request": "attach"`, `"port": 9229`, `"skipFiles": ["<node_internals>/**"]`, `"type":
  "node"`), ran **Attach**, set the same breakpoint and sent a `POST` and then a `GET`. The root
  `.gitignore` ignores `.vscode/`, so `launch.json` stays local. A screenshot of the paused session showed:
  - Call Stack: `getNote` (78:18) and two `<anonymous>` frames, with Node's own frames hidden by
    `skipFiles`. The line was highlighted yellow with a red breakpoint dot.
  - Variables: `Local: getNote` (`id`, `req`, `res`, `this = undefined`, and no `note` yet), `Closure
    (createNotesServer)` (including `notes = Map(5)`), `Module` and `Global`. After one step over (F10)
    `note` appeared in `Local`.
  - Debug Console: `id`, `req.url` and `notes.size` printed (5, because five notes had been created in
    that run), and `note` failed exactly as in the protocol session.
  - While paused, the `GET`'s `curl` waited with no output, and my next `POST` waited too: a paused
    debugger stops the whole single-threaded server until you continue.
  - After removing the breakpoint, `curl --max-time 5` showed the `POST` returning `201` in about 1 ms and
    the `GET` returning `200` in about 0.6 ms. Nothing paused.
- `console.table` draws a table, `console.error` writes to stderr, `console.dir(obj, { depth: 0 })` prints
  `{ a: [Object] }`, `console.time`/`timeEnd` print `t: 0.002ms`, `console.group` indents, and
  `console.assert` prints only when the condition is false. A stack trace from `c()`
  called by `b()` called by an `async a()` after an `await` listed `c`, `b` and `a`. `new Error('outer', {
  cause: e })` keeps the original as `cause`.

*npm* (in a scratch project, with `semver` as the example package)
- `npm init -y` wrote `"type": "commonjs"`, `"main": "index.js"`, an `ISC` licence and a failing `test`
  script.
- **Vetting before installing**, with `npm view semver ...`: version 7.8.5, licence ISC, four maintainers,
  only dev scripts (no `preinstall`, `install` or `postinstall`), no dependencies, 101,065 bytes unpacked
  in 53 files, a SLSA provenance attestation, created 2011, last modified 2026-09-09.
- `npm install semver@7.5.0` added 3 packages (`semver`, plus `lru-cache` and `yallist` that I never asked
  for) and wrote `"semver": "^7.5.0"`. It immediately reported "1 high severity vulnerability".
  `package-lock.json` (lockfileVersion 3) recorded each package's exact version, `resolved` URL and
  `integrity` hash.
- `npm audit` named it: `semver 7.0.0 - 7.5.1, Severity: high, ... Regular Expression Denial of Service`
  (exit 1). `npm outdated` showed Current 7.5.0, Wanted 7.8.5, Latest 7.8.5 (exit 1). `npm ls --all` shows
  `semver` then `lru-cache` then `yallist`.
- **Ranges** against 0.0.3 0.0.4 0.2.3 0.2.9 0.3.0 7.5.0 7.5.9 7.6.0 7.8.5 8.0.0: `^7.5.0` matched 7.5.0,
  7.5.9, 7.6.0 and 7.8.5; `~7.5.0` matched 7.5.0 and 7.5.9; `^0.2.3` matched 0.2.3 and 0.2.9; `^0.0.3`
  matched only 0.0.3.
- `npm update` left `package.json` at `^7.5.0` but moved the lockfile to 7.8.5 and removed `lru-cache` and
  `yallist`. The lockfile diff was exactly the `version`, `resolved` and `integrity` lines. `npm audit`
  then found 0 vulnerabilities.
- **`ci` against `install`.** With `package.json` edited to `^6.0.0` and a lockfile at 7.8.5, `npm ci`
  refused (`code EUSAGE ... Invalid: lock file's semver@7.8.5 does not satisfy semver@6.3.1`, exit 1). With
  the two in sync, `npm ci` rebuilt `node_modules` in about half a second.
- `npx semver 7.5.0 -r "^7"` ran the local binary and printed `7.5.0`. `npm install --save-dev semver` on a
  project where it was a normal dependency moved it; `npm ci --omit=dev` then installed nothing.
- **Lifecycle scripts.** A local package whose `postinstall` writes a file: `npm install` printed an
  `install-scripts` warning and the script **still ran**, as me. `--ignore-scripts` did not run it, and
  `--strict-allow-scripts` stopped with `ESTRICTALLOWSCRIPTS` (exit 1). Both settings default to `false`.
- **`engines`.** `"node": ">=99"` gave only an `EBADENGINE` warning and exit 0. With `--engine-strict` the
  install failed with exit 1.

**9. Commit, push, open the pull request and merge it.**
- *Do:* run the checks again, then commit in three groups, staging files by name or folder:
  ```bash
  git add modules/m03-nodejs-runtime-npm/package.json modules/m03-nodejs-runtime-npm/package-lock.json \
    modules/m03-nodejs-runtime-npm/.env.example .gitignore .env.example
  git commit -m "chore(m03): add package.json, lockfile, env example and ignore rules"
  git add modules/m03-nodejs-runtime-npm/src/log-report
  git commit -m "feat(m03): add log generator and streaming log-report CLI"
  git add modules/m03-nodejs-runtime-npm/src/http-server
  git commit -m "feat(m03): add bare node:http notes server"
  git push -u origin feat/module-03/runtime-and-npm
  gh pr create --repo kiannaalexandra-gragg/node-js-foundation-training --base main \
    --head feat/module-03/runtime-and-npm --title "feat(m03): add log-report CLI and bare node:http server" \
    --body "<Summary, Changes, Test plan>"
  ```
  The PR body has Summary, Changes and Test plan sections, and the test plan lists only checks I had just
  run. I opened the PR, and it was merged on GitHub afterwards.
- *Why:* every stage is delivered from a branch and a pull request, and `--repo`, `--base` and `--head` are
  passed explicitly so the PR cannot land in a repo I did not mean. The commits carry no `Co-Authored-By`
  trailer.
- *Result:* PR #3, merged with a merge commit (see Checkpoint evidence). `data/` and `.env` stayed out of
  the commits because the `.gitignore` rules cover them.

## Concepts worth explaining

**Streams and backpressure**

A stream moves data in chunks, so a program can start on the first chunk while the rest is still on disk,
and never has to hold all of it. A file stream reads 64 KiB at a time by default (measured:
`readableHighWaterMark` 65,536). A `highWaterMark` of 64 bytes turned a 3 KB script into 47 chunks, so the
chunk size is a setting and not a property of the file.

The part I would have missed is that a stream has two sides, and the fast side must wait for the slow one.
In the generator, `output.write(chunk)` returns `false` once its internal buffer is full. That is the
signal to stop and wait for `'drain'`. If I ignore it, every chunk I produce stays in the buffer, and my
"streaming" generator quietly becomes the whole-file-in-memory program it was meant to avoid. Reading has
the same rule in the other direction: `for await (const line of lines)` only asks `readline` for the next
line when the loop body is done, so the read stream is paused until the counting catches up. That is why
the CLI's memory stayed near 85 MB for a 100 MB file and it finished a 600 MB file under a 32 MB heap cap,
when the whole-file version needed 821 MB for 100 MB and failed outright for 600 MB.

**The event loop and the thread pool**

Node runs your JavaScript on one thread. That is fine because most of what a server does is wait (for a
disk, a socket or a timer), and the waiting is handed to the operating system or to libuv's thread pool.
The cost is that anything that keeps the one thread busy stops everything else. My experiment showed it: a
timer set to fire every 100 ms went quiet for 500 ms during a busy loop and fired again at 751 ms, so every
request on a real server would have waited too.

Some work cannot be done by the operating system without blocking, such as `fs` calls and
`crypto.pbkdf2`. libuv runs those on a pool of threads, 4 by default. Eight identical `pbkdf2` jobs took
380 ms with a pool of 1, 111 ms with the default 4 (two waves of four) and 79 ms with 8. I believe my stuck
`/dev/stdin` read was holding one of those threads, which would explain why even `process.exit()` hung, but
I did not prove that.

One more surprise: ordering inside one tick depends on the module type. In CommonJS, `process.nextTick` ran
before a promise callback. In an ES module the promise callback ran first. Between `setTimeout(0)` and
`setImmediate`, `setImmediate` won in nine of ten runs, but I would not rely on either order.

## What tripped me up

1. **`parseArgs` throws on an unknown flag.** `node src/log-report/generate-log.js --nope` printed a full
   `ERR_PARSE_ARGS_UNKNOWN_OPTION` stack trace and exited 1. `parseArgs` is strict by default, so I wrapped
   it in `try/catch`, printed one line, and exited 2 (a usage error).
2. **My SIGINT check hung, and so did my first fix.**
   - I tested Ctrl+C by pointing the CLI at `/dev/stdin` fed by a pipe that never sends data, then sending
     `SIGINT`. The check never finished, and the command hit the 120 s tool timeout. I had to `pkill` it.
   - Installing a signal handler replaces Node's default "exit now" behaviour, and `abort()` only asks a
     stream to close. My reading is that a stream cannot finish closing while its read is pending on a pipe
     that never delivers. That is my reasoning, not something I confirmed in Node's source.
   - My first fix, `process.exit(code)` on a second signal, hung too: a probe showed the handler ran twice
     and called `process.exit(130)`, yet the process was alive 0.7 s later. I think a thread-pool read that
     never returns blocks the exit, but I did not read libuv's source to confirm that.
   - The fix that worked: on the second signal, `process.removeAllListeners(signal)` and
     `process.kill(process.pid, signal)`, so Node's default handler ends the process at once. The shell
     sees 130, and a child process sees `code null, signal 'SIGINT'`.
   - My first probe also signalled the wrong process: `pgrep -f probe2.mjs` matched the `zsh -c` wrapper
     whose command line contained the script name. Use `$!` or `pgrep -x node`.
3. **The first bare server crashed on bad JSON.** A POST with `{"text": oops` killed the whole process
   (`SyntaxError: Unexpected token 'o'`, then `curl` exit 7 on the next request). An `async` request
   handler's rejection is not caught by `http.createServer`, so it becomes an unhandled rejection and Node
   exits. One bad request took the service down for everyone.
4. **An unknown id returned `200` with an empty body.** `notes.get()` gave `undefined`,
   `JSON.stringify(undefined)` is `undefined`, and `res.end(undefined)` sends nothing. There was no 404 and
   no JSON.
5. **`fetch` could not read my 413.** I answered an oversized body with `Connection: close`, and the check
   crashed with `TypeError: fetch failed ... Error: write EPIPE`, because the server hung up while the
   client was still sending. `curl` had shown the 413 fine. Removing `Connection: close` fixed it, since
   Node discards the unread body itself once the response is sent, but only when the `Content-Length` is
   known. For a chunked body cut off mid-stream I tested both ways: five runs with `Connection: close` gave
   a 413 to both a `fetch` and an `http.request` client every time; without it `fetch` timed out in two of
   five runs and one `http.request` got `ECONNRESET`. So the chunked 413 keeps `Connection: close` and the
   declared-length 413 does not. I did not find out why.
6. **`HTTP/1.1 100 Continue` arrives before the 413.** Node answers `Expect: 100-continue` by itself before
   my handler runs, so the client is told to send the body and is then refused. Refusing earlier needs
   `server.on('checkContinue')`. I did not do that.
7. **`GET //notes` gave 404, and `GET //` gave 500.** `new URL('//notes', 'http://localhost')` reads a
   leading `//` as a host, so the pathname is `/`. A code review later found the worse case: `new URL('//',
   'http://localhost')` throws `ERR_INVALID_URL`, so `GET //` answered 500 and logged a stack trace. I now
   parse from a fixed prefix (see "Why it's built this way"), and `//notes` routes as `/notes`.
8. **Process-wrapper surprises.**
   - `kill -TERM` on the `node --run serve` process left the server running as an orphan on port 3000, and
     the next test failed with `EADDRINUSE`. Re-tested, signalling only the process I started: `node --run
     serve` left the server running for both `SIGTERM` and `SIGINT`, while `npm run serve` and a plain `node`
     shut down cleanly. `SIGINT` to the whole process group, which is what Ctrl+C does (I used `perl -e
     'setpgrp(0,0); exec @ARGV'`), left no server behind for either. With `npm run` my shutdown log line was
     not captured in that last case, and I did not find out why.
   - Counting leftovers with `grep server.js` also matched VS Code helper processes. I needed a precise
     pattern.
9. **Start-up failures printed stack traces.** A second server on the same port printed a 20-line
   `EADDRINUSE` trace, and `PORT=abc` printed an `ERR_SOCKET_BAD_PORT` `RangeError`. Worse, `PORT=` (an
   empty value) gave `Number('') = 0`, a valid port, so the server started on a random port and my test
   loop hung. Fixes: a `listen` error handler, a hand-written `PORT` check (exit 2), and `||` instead of
   `??`.
10. **The wrong Node version, twice.** My default shell was on Node 22. I ran `nvm use` from a folder with
    no `.nvmrc` (`No .nvmrc file found`) and then reported `URLPattern` as `undefined`. It is a global on
    Node 24 (re-run in the repo gave `function`).
11. **I misread two of my own measurements.** I printed `readableHighWaterMark` of a stream I had created
    with `highWaterMark: 64` and labelled it the default (the real default is 65,536). And I expected a
    flat memory line for the streaming run and got 85 MB at 100 MB and 117 MB at 600 MB. It is not retained
    data (see "Why it's built this way").
12. **npm settings surprised me.** `npm init -y` writes `"type": "commonjs"`, so it has to be changed to
    `"module"` by hand. And npm warned about install scripts but ran them anyway, until I passed
    `--strict-allow-scripts`.
13. **A code review found problems my checks could not see.** Each was reproduced before I fixed it.
    - Shutdown with a request in flight exited 1 after 5 s (see "Why it's built this way"). My first fix
      set a flag that added `Connection: close` to responses, and it failed: the request had started
      before the signal, so the flag was still false. I replaced it with the sweep.
    - Three checks could hang instead of fail: the stuck-read check's `exitCode === null` assertion also
      passes for a process killed by a signal, the server check waited forever for a "listening" line if
      `server.js` exited at start-up, and an empty request handler made `fetch` wait for its default
      timeout. I proved each fix by running the check against code made wrong on purpose.
    - A failed output write was reported as "cannot read".
14. **At a breakpoint, `note` could not be read.** Paused on `const note = notes.get(id);`, evaluating
    `note` gave `ReferenceError: Cannot access 'note' from debugger`. The line had not run yet, so the
    `const` was still unassigned. I had to step over once before `note.text` worked.

**Where the bare server hurt, and what I expect Express to remove**

I have not used Express yet, so this is a list of what I wrote by hand and what I expect it to take over. I
will check each against Express 5 in M05.

- *Routing.* Splitting the path into segments, matching `:id` by hand, and building 404 and 405 (with
  `Allow`) myself. I expect a router to do the matching and the 405.
- *Body parsing.* Collecting chunks, checking `Content-Type`, counting bytes against a limit, and mapping a
  `JSON.parse` failure to 400. I expect a built-in JSON parser to cover this.
- *Errors.* The one `try/catch` that stops a thrown error in an `async` handler from killing the process,
  plus the `HttpError` mapping. The solution design says Express 5 sends rejected promises to the error
  handler automatically.
- *Response plumbing.* `writeHead`, `Content-Length` and `JSON.stringify` on every response, wrapped in
  `sendJson`.
- *Easy to miss.* Nothing in `node:http` stopped me returning `200` with an empty body, or crashing on bad
  input. Safe behaviour was something I had to remember to write each time.

## Checkpoint evidence

**Both programs merged to `main` by pull request.** One PR carries both programs:
[#3, `feat(m03): add log-report CLI and bare node:http server`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/3),
from `feat/module-03/runtime-and-npm` into `main`. It was merged on 2026-10-20 (13:00 UTC) with a merge
commit, so the three commits are visible in the history:

```text
$ git log --oneline --graph origin/main -5
*   f26878b Merge pull request #3 from kiannaalexandra-gragg/feat/module-03/runtime-and-npm
|\
| * a9e66dc feat(m03): add bare node:http notes server
| * 8a9bbba feat(m03): add log generator and streaming log-report CLI
| * 82d0284 chore(m03): add package.json, lockfile, env example and ignore rules
|/
*   30b6a9e Merge pull request #2 from kiannaalexandra-gragg/feat/module-02/async-toolkit
```

`gh pr view 3` reported the state `MERGED` with the base `main` and those three commits. The brief's lab
says "both merged PRs" in one place and "merged to `main` through a pull request" in another. I used one PR
for both programs, the same shape as M02, and the EM can confirm whether two were expected.

**The CLI's memory behaviour on the full log file.** Re-captured on the final code, after the folder
restructure and the review fixes, on the default file (2,231,981 lines, 104,857,613 bytes, 2,200 malformed
lines):

```text
$ /usr/bin/time -l node src/log-report/log-report.js --stats -o /tmp/m03-final.json
Wrote /tmp/m03-final.json: 2231981 lines, 2200 malformed
stats: 0.93s elapsed, peak memory 84.9 MB (maxRSS)
        0.95 real         0.94 user         0.03 sys
            89210880  maximum resident set size
```

| Measurement | Value |
| --- | --- |
| Streaming CLI, 100 MB file, three runs | 84.7, 85.1 and 85.8 MB by `--stats`; 89,014,272 to 90,144,768 bytes by `time -l` |
| Whole-file version, same file | 821 MB (861,044,736 bytes) |
| Streaming CLI, 1 MB file | 62,685,184 bytes |
| `node -e 0` | 45,776,896 bytes |
| Streaming CLI, 600 MB file | 117.0 MB, 5.74 s |
| Streaming CLI, 600 MB file, `--max-old-space-size=32` | passed |
| Whole-file version, 600 MB file | `RangeError: Invalid string length` |

Final report totals: `byStatusClass` 2xx 1,739,888, 3xx 177,482, 4xx 222,785, 5xx 89,626; `byMethod` DELETE
37,242, GET 1,855,099, POST 281,381, PUT 56,059; busiest paths `/api/tasks` 618,344, `/` 375,134,
`/api/tasks/1` 168,786, `/health` 168,724, `/api/tasks/2` 131,371. The report's SHA-256 starts
`0eb16fc1e6b84dd1` and was identical in every full run.

**The bare server's three endpoints responding**, on the final code (captured again on 2026-10-20, after the
restructure and the review fixes) with `npm run serve`:

```text
$ curl -s -i -X POST localhost:3000/notes -H 'Content-Type: application/json' -d '{"text":"remember the milk"}'
HTTP/1.1 201 Created
Location: /notes/9a7cc1c4-c680-4766-997b-2344ee3e7d62
{"id":"9a7cc1c4-c680-4766-997b-2344ee3e7d62","text":"remember the milk","createdAt":"2026-10-20T08:26:32.431Z"}

$ curl -s localhost:3000/notes
[{"id":"9a7cc1c4-c680-4766-997b-2344ee3e7d62","text":"remember the milk","createdAt":"2026-10-20T08:26:32.431Z"}]

$ curl -s localhost:3000/notes/9a7cc1c4-c680-4766-997b-2344ee3e7d62
{"id":"9a7cc1c4-c680-4766-997b-2344ee3e7d62","text":"remember the milk","createdAt":"2026-10-20T08:26:32.431Z"}
```

**The checks:** `npm run check` exited 0 with 27 `ok` lines (6 + 9 + 12) and no skips.

**My explanation of what Express would remove pain from:** see "Where the bare server hurt" under "What
tripped me up".

**Not yet covered:** running any of this on Windows (the SIGINT check uses `/dev/stdin`, and I only used
macOS and zsh); how long the stage took; and anyone else following this path cold.

## What I'd do differently

- Wrap the request handler in one `try/catch` before writing a single route. The crash on bad JSON and the
  empty-body `200` both came from the first version having no such place.
- Write `server.on('error')` and the `PORT` check on day one. Three separate start-up failures printed stack
  traces before I handled them.
- Test the "stuck read" signal case with a real long file first and only then reach for `/dev/stdin`. I
  lost two long runs to the hung check.
- Keep a precise way to find my own processes (`$!`, or a full `pgrep` pattern) from the start. I wasted a
  probe and a test run on matching the wrong process.
- Measure the default instead of printing a number I had just set myself.
