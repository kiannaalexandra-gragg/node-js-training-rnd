# M03 — Node.js Runtime & npm: Training Material

**Week 1 · Days 4-5 · 6 hours**

A "module" in this program means a training stage. "ES module" means a JavaScript module file.

## Builds on

M02 — Modern JavaScript & Async/Await. You reuse `for await`, `async` functions and `AbortController`.
You also need your build repo from M01 and M02, Node 24 pinned with `.nvmrc`, and the branch → pull request
→ merge habit. This stage ends with a pull request and is the **Week 1 Friday gate**.

## 1. What you'll build and why

You will learn how Node.js runs your code, use its core modules, and manage npm dependencies safely.
You finish with:

- a **`log-report` CLI** that streams a ~100 MB log file line by line and writes a JSON report (options read by `parseArgs`),
- a **bare `node:http` server** with three JSON endpoints, with routing, body parsing and error handling written by hand,
- your own **explanation of what Express would remove pain from**,
- hands-on npm basics: versions, the lockfile, `install` vs `ci`, vetting a package, install scripts.

The bare server is meant to hurt a little. You build by hand what Express does for you later, so in M05 you know what you are being given.

**In your repo:** the two programs, their checks and the project files, in `modules/m03-nodejs-runtime-npm/`. The research experiments and the npm lab stay in a scratch folder outside the repo.

| File | Where | Built in |
| --- | --- | --- |
| `package.json`, `.env.example` | your repo, stage folder | Stage 1 |
| `.gitignore`, `.env.example` (two small edits) | your repo, **repo root** | Stage 1 |
| `src/log-report/generate-log.js` | your repo | Stage 3 |
| `src/log-report/aggregate.js`, `aggregate.check.js` | your repo | Stage 4 |
| `src/log-report/log-report.js`, `log-report.check.js` | your repo | Stage 5 |
| `src/http-server/notes-server.js`, `server.js`, `server.check.js` | your repo | Stages 7 and 8 |
| `package-lock.json` | your repo, stage folder | Stage 9 |
| research scripts, the whole-file comparison, the npm lab | scratch folder, not delivered | Stages 2, 6, 9 and 10 |

### How to use this guide

**Start here.** You do not need the same depth from every part.

| Stage | What it is | How to treat it |
| --- | --- | --- |
| 1 | Branch, folders, project files | Core |
| 2 | Research by experiment | Core 1–5. The reference experiments are optional. |
| 3 | Generate the 100 MB log | Core: copy, run, read |
| 4, 5 | Counting, then the streaming CLI | Core: you write it |
| 6 | Measure memory on the full file | Core (the 600 MB step is optional) |
| 7, 8 | Bare server: naive, then hardened | Core: you write Stage 8 |
| 9 | npm in a scratch project | In scope, outside the programs: go through once |
| 10 | Debugging | In scope, outside the programs: short |
| 11 | What a framework would remove | Core: a definition-of-done item |
| 12 | Commit, push, pull request, merge | Core |

**How to read this guide**

- **Guided path** (section 3): what to do, run and check.
- **Concepts** (section 4): a quick glossary of every term, then short essays. Read the event loop essay after Stage 2, the streams essay after Stage 6, and the framework essay after Stage 8.
- **Common pitfalls** (section 5): look here when something misbehaves.
- **Definition of done** (section 6): the final check.

**Stages 4, 5 and 8 are exercises.** Each has *what it must do*, *your turn* (a scaffold, with hints you open one at a time), *self-checks* that fail until your code works, and a *reference implementation* to open after you have tried. Stages 3 and 7 give you code to run, because there the learning is in what you observe.

## 2. Before you start

- **Your build repo** (M01, M02) on `main`, with `.nvmrc` containing `24`, a root `.gitignore` that already ignores `.env`, and a root `.env.example`.
- **Node 24 active.** Run `nvm use` **on its own line** in the repo root and check that `node --version` prints `v24.x`. Do this every session: a new terminal can start on an older Node, and `nvm use` in a folder with no `.nvmrc` does nothing.
- **Tools and space:** internet for npm (Stage 9), about 200 MB of free disk (Stage 6 has an optional 600 MB test), and `curl`, `git` and `gh` (for Stage 12).
- **Pinned:** Node.js 24 LTS and plain JavaScript. No TypeScript (it starts in M04), no Express, no npm packages: you use only built-in `node:` modules. The npm messages quoted in Stage 9 are from npm 11 and may differ in other versions.
- **File convention:** the stage folder has its own `package.json` with `"type": "module"`, so `.js` files are ES modules and imports use the full filename (`./aggregate.js`). The import-extension convention for the TypeScript stages is still an open program decision ("P2"), so this is not its rule.
- **File headers:** start each file with a short comment: `Module 3 output`, which deliverable it belongs to (`deliverable 1 of 2 (log-report CLI)` or `deliverable 2 of 2 (bare node:http server)`), how to run it, and what success looks like.
- **Not needed:** `node:test` and `node:sqlite` exist, but this course uses Jest and Prisma.

Commands are for macOS or Linux and have not been tried on Windows. On Linux, `/usr/bin/time -l` is `/usr/bin/time -v` and `shasum -a 256` is `sha256sum`. Run commands from the stage folder, `~/<repo-name>/modules/m03-nodejs-runtime-npm`, unless a step says otherwise.

## 3. Guided path

### Stage 1 — Create the branch, the stage folders and the project files

**Goal:** a feature branch, a stage folder with one subfolder per program, a `package.json` that makes `.js` files ES modules, a `.env.example`, and two small edits in the repo root.

```bash
cd ~/<repo-name>
nvm use
node --version
git switch main
git switch -c feat/module-03/runtime-and-npm
mkdir -p modules/m03-nodejs-runtime-npm/src/log-report modules/m03-nodejs-runtime-npm/src/http-server
cd modules/m03-nodejs-runtime-npm
```

The stage is laid out by deliverable. Each program gets its own folder, and both share one `package.json` and one lockfile because they use only built-in modules:

```text
m03-nodejs-runtime-npm/
├── package.json
├── .env.example
├── package-lock.json            (Stage 9)
├── data/                        (the generated log, Stage 3, git-ignored)
└── src/
    ├── log-report/
    │   ├── generate-log.js            Stage 3
    │   ├── aggregate.js               Stage 4
    │   ├── aggregate.check.js         Stage 4
    │   ├── log-report.js              Stage 5
    │   └── log-report.check.js        Stage 5
    └── http-server/
        ├── notes-server.js            Stages 7 and 8
        ├── server.js                  Stages 7 and 8
        └── server.check.js            Stage 8
```

Create `package.json` in the stage folder. Every script points into `src/`. The two server scripts load a `.env` from the stage folder, because npm runs scripts from the folder that holds `package.json`:

```json
{
  "name": "m03-nodejs-runtime-npm",
  "version": "1.0.0",
  "private": true,
  "description": "M03: a log-report CLI that streams a large log file, and a bare node:http server",
  "type": "module",
  "engines": {
    "node": ">=24"
  },
  "scripts": {
    "generate-log": "node src/log-report/generate-log.js",
    "report": "node src/log-report/log-report.js",
    "serve": "node --env-file-if-exists=.env src/http-server/server.js",
    "dev": "node --watch --env-file-if-exists=.env src/http-server/server.js",
    "check": "node src/log-report/aggregate.check.js && node src/log-report/log-report.check.js && node src/http-server/server.check.js"
  }
}
```

Create `.env.example` in the stage folder. It documents the one setting and is never loaded itself. Copy it to `.env` to use it:

```text
# Module 3 setting for deliverable 2 of 2 (bare node:http server): copy this file to .env in this folder.
# npm run serve and npm run dev load .env if it exists. Never commit .env.

# The port the notes server listens on. Without it, the server uses 3000.
PORT=3000
```

Now edit two files in the **repo root**. Add these lines to the root `.gitignore` so the 100 MB log and the default report are never committed. The paths name this stage only, so a later stage's `data/` is not hidden:

```text
# M03 generated files: the ~100 MB log and the CLI's default report. Regenerate them.
modules/m03-nodejs-runtime-npm/data/
modules/m03-nodejs-runtime-npm/report.json
```

Replace the root `.env.example`. It is an index, not a file to copy. It points to each module's own `.env.example`:

```text
# An index of the settings each module reads. Do not copy this file.
# Copy the module's own .env.example to .env inside that module's folder, and fill in real values.
# Never commit .env.
#
# M03, modules/m03-nodejs-runtime-npm/.env.example: PORT, the notes server's port (optional, default 3000)
```

**Why:**

- Every stage is delivered as a pull request, never a push to `main`.
- `"type": "module"` makes `.js` files ES modules (`npm init -y` writes `"commonjs"`). `--env-file-if-exists=.env` loads a real `.env` only if you create one.
- One root `.gitignore` covers every stage. Each stage keeps its own `.env.example`, and the root one is only an index.

**Done when:**

- [ ] `node --version` prints a `v24.x` version, and `git status -sb` shows you on `feat/module-03/runtime-and-npm`.
- [ ] `src/log-report/` and `src/http-server/` exist inside the stage folder.
- [ ] `npm pkg get type` prints `"module"`, and `npm run` lists the scripts `generate-log`, `report`, `serve`, `dev` and `check`.
- [ ] `.env.example` exists in the stage folder and contains `PORT=3000`.
- [ ] `git check-ignore -v data/access.log` (from the stage folder) prints the root `.gitignore` rule that matches it.

### Stage 2 — Research by experiment

**Goal:** see how the runtime really behaves before you build on it.

Run **Core 1 to Core 5**. Stages 4 to 8 depend on them. The reference experiments are optional. Keep everything **outside your repo**, in a scratch folder:

```bash
mkdir -p ~/m03-scratch
cd ~/m03-scratch
nvm use 24
node --version
```

This folder has no `.nvmrc`, so a bare `nvm use` does nothing. Check that `node --version` prints `v24.x`. For each experiment, **write your prediction first**, then run it. A wrong prediction is the useful result.

#### Core 1 — What runs first? (`loop-order.mjs`)

```js
console.log('1 sync start');
setTimeout(() => console.log('5 setTimeout 0'), 0);
setImmediate(() => console.log('6 setImmediate'));
process.nextTick(() => console.log('3 nextTick'));
Promise.resolve().then(() => console.log('4 promise microtask'));
console.log('2 sync end');
```

Run `node loop-order.mjs`. Then copy it to `loop-order.cjs` (`cp loop-order.mjs loop-order.cjs`) and run that. Compare the two outputs. Run the `.mjs` file ten times and watch the last two lines.

#### Core 2 — What does one busy thread cost? (`blocking.mjs`)

```js
// A timer that should tick every 100 ms, while something else may hog the thread.
const start = performance.now();
const ticks = [];
const timer = setInterval(() => ticks.push(Math.round(performance.now() - start)), 100);
setTimeout(() => {
  clearInterval(timer);
  console.log('tick times (ms):', ticks.join(', '));
}, 1000);
const mode = process.argv[2];
setTimeout(() => {
  if (mode === 'blocking') {
    const end = Date.now() + 500;
    while (Date.now() < end) {} // keep the one thread busy for 500 ms
    console.log('blocked 500 ms');
  }
}, 250);
```

Run `node blocking.mjs none` and then `node blocking.mjs blocking`. Compare the tick times.

#### Core 3 — The thread pool (`pool.mjs`)

```js
import { pbkdf2 } from 'node:crypto';
const start = performance.now();
let done = 0;
for (let i = 0; i < 8; i++) {
  pbkdf2('pw', 'salt', 300000, 64, 'sha512', () => {
    if (++done === 8) console.log('all 8 done at', Math.round(performance.now() - start), 'ms');
  });
}
```

Run it three times: `node pool.mjs`, `UV_THREADPOOL_SIZE=1 node pool.mjs` and `UV_THREADPOOL_SIZE=8 node
pool.mjs`. Find your CPU count with `node -p "require('node:os').availableParallelism()"`.

#### Core 4 — ES modules against CommonJS (`esm-globals.mjs`)

```js
console.log('typeof require:', typeof require, '| typeof __dirname:', typeof __dirname);
console.log('import.meta.dirname :', import.meta.dirname);
console.log('import.meta.filename:', import.meta.filename);
console.log('top-level await works:', await Promise.resolve('yes'));
```

Run `node esm-globals.mjs`. Then try these:

- Put `const x = require('node:path');` in a `.mjs` file and read the error.
- Make a `.js` file containing `import path from 'node:path'; console.log('ran')` in a folder with no `package.json` (in it or any parent) and run it.
- Run `node -e "console.log(require('fs') === require('node:fs'))"`, then `node -e "require('test')"`, then `node -e "console.log(typeof require('node:test').test)"`.

#### Core 5 — `process`, environment variables and exit codes (`env.mjs`)

```js
console.log('argv slice:', process.argv.slice(2));
console.log('GREETING =', process.env.GREETING, '| PORT =', process.env.PORT, typeof process.env.PORT);
console.log('EMPTY =', JSON.stringify(process.env.EMPTY), '| MISSING =', process.env.MISSING);
```

Create a file called `.env` next to it:

```text
GREETING=hello from file
PORT=4321
# a comment
EMPTY=
```

Run `node --env-file=.env env.mjs a --flag b`. Then run:

- `PORT=9999 node --env-file=.env env.mjs`
- `node --env-file=nope.env env.mjs; echo "exit=$?"`
- `node --env-file-if-exists=nope.env env.mjs`

Finally, print the exit code of each of these:

```bash
node -e "process.exit(3)"; echo $?
node -e "throw new Error('x')" 2>/dev/null; echo $?
node -e "process.exitCode = 4"; echo $?
node -e "Promise.reject(new Error('r'))" 2>/dev/null; echo $?
```

#### What you should see

Results from one reference run on Node 24 (macOS, Apple silicon, 10 cores). Your numbers will differ, but the shapes should match.

| Experiment | Result |
| --- | --- |
| Core 1, `.mjs` | `1, 2, 4 promise microtask, 3 nextTick, 6 setImmediate, 5 setTimeout 0` |
| Core 1, `.cjs` | `1, 2, 3 nextTick, 4 promise microtask, 6, 5` |
| Core 1, ten `.mjs` runs | `setImmediate` first in 9 of 10. Neither order is guaranteed from the main module. |
| Core 2, no blocking | ticks at about 101, 202, 304, 405 ... 911 ms |
| Core 2, blocking | ticks at about 101, 202, then **751**, 852, 953 |
| Core 3, 8 jobs | pool of 1: about 380 ms; default (4): about 111 ms; pool of 8: about 79 ms |
| Core 4 | `typeof require` and `typeof __dirname` are `undefined`; `import.meta.dirname` and `import.meta.filename` print absolute paths; `require` in a `.mjs` gives `ReferenceError: require is not defined in ES module scope`; the `.js` file with `import` and no `package.json` ran; `require('fs') === require('node:fs')` is `true`; `require('test')` fails with `MODULE_NOT_FOUND` but `require('node:test')` works |
| Core 5, `--env-file` | `argv slice: [ 'a', '--flag', 'b' ]`; `PORT = 4321 string`; `EMPTY = ""`; `MISSING = undefined` |
| Core 5, `PORT=9999` | the real environment variable wins over the file |
| Core 5, missing file | `node: nope.env: not found`, exit 9; with `--env-file-if-exists` it runs without it |
| Core 5, exit codes | `3`, `1`, `4`, `1` |

#### Reference experiments (optional)

Optional one-liners. The second column shows what the reference run showed.

| Try | What it showed |
| --- | --- |
| `node -p "const o={d:new Date(0),m:new Map([[1,2]])}; o.self=o; const c=structuredClone(o); [c.d instanceof Date, c.m instanceof Map, c.self===c]"` | `[ true, true, true ]`. `JSON.stringify(o)` on the same object throws a `TypeError`, and `structuredClone({f(){}})` throws a `DataCloneError`. |
| `node -p "const u=new URL('https://user:pw@example.com:8080/a/b?x=1#frag'); [u.hostname,u.port,u.pathname,u.searchParams.get('x'),u.hash]"` | the parts, split correctly |
| `node --watch script.mjs`, then edit the file | `Change detected in '<path>'`, `Restarting 'script.mjs'` and a new pid |
| `node -e "const {EventEmitter}=require('node:events'); new EventEmitter().emit('error', new Error('no listener'))"` | it **throws**: an `'error'` event with no listener is an exception |
| `node:timers/promises`: `await sleep(50)`; abort a `sleep(5000, null, { signal })` | about 51 ms; the aborted one rejects with `AbortError`, code `ABORT_ERR` |
| `node:util`: `promisify(fs.readFile)`; `parseArgs({ args: ['-v','--name','x','f.txt'], options: { v: {type:'boolean',short:'v'}, name: {type:'string'} }, allowPositionals: true })` | `{ values: { v: true, name: 'x' }, positionals: ['f.txt'] }`; `parseArgs` throws on a flag it does not know |
| `node:path`: `path.join('a','..','b','c.txt')`, `path.extname('a/b.tar.gz')`; `node:os`: `os.platform()`, `os.availableParallelism()` | `b/c.txt`, `.gz`, then your platform and core count |
| `crypto.randomUUID()` | 36 characters, and the 15th (index 14) is `4` |
| `fetch`, `URL`, `AbortController`, `structuredClone` | all four are globals, so no `import` is needed |
| `node:stream/promises`: `node -e "const {pipeline}=require('node:stream/promises'),{Readable,Transform,Writable}=require('node:stream'); let out=''; pipeline(Readable.from(['a','b','c']), new Transform({transform(c,e,cb){cb(null,String(c).toUpperCase())}}), new Writable({write(c,e,cb){out+=c;cb()}})).then(()=>console.log(out))"` | `ABC`. See Concepts for what a stream is. |

`node:test`, `node:sqlite` and the global `URLPattern` also exist on Node 24. You do not need them: this course uses Jest and Prisma, and you hand-write routing in Stage 8.

**Done when:**

- [ ] You ran Core 1 to Core 5 in `~/m03-scratch`.
- [ ] You can explain the Core 1 order: why `nextTick` and the promise callback swap places between `.cjs` and `.mjs`, and why `setTimeout(0)` and `setImmediate` have no fixed order.
- [ ] You can explain Core 2 and Core 3: why one blocked thread stalls every request, and why pool size changed the `pbkdf2` times.
- [ ] You can explain why `require` and `__dirname` are missing in an ES module, what replaces `__dirname`, and what the `node:` prefix does (name a module that needs it).

### Stage 3 — Generate the log file

**Goal:** a synthetic access log of about 100 MB, one record per line: the input for the CLI.

Create `src/log-report/generate-log.js` with the code below. Do this even if your trainer also gives you a log: the Stage 5 self-checks run this script to make a 300 MB log.

It is a data tool, not the thing you are learning, so copy it, but read it: the way it **writes** mirrors how the CLI will **read**.

The exact numbers in this guide (file size, line count, SHA-256 values, 2,200 malformed lines) are for the file this script writes with default settings. If you use a trainer's file instead, skip those checks and use your own totals.

```js
// Module 3 support file: generates the practice log that the log-report CLI reads (deliverable 1 of 2).
// Run from modules/m03-nodejs-runtime-npm: npm run generate-log
// Options: --size-mb 5 (default 100), --seed 2 (default 1), --out /tmp/small.log (default data/access.log)
// Success: "Wrote 2231981 lines, 100.0 MB, to <path>/data/access.log in ~2s (seed 1)".
//
// One record per line: `<ISO timestamp> <METHOD> <path> <status> <duration ms>`. The same seed always
// writes the same file, so everyone's report can be compared. About one line in a thousand is
// deliberately garbled, so the CLI has to cope with bad input.
import { createWriteStream } from 'node:fs';
import { mkdir } from 'node:fs/promises';
import path from 'node:path';
import { once } from 'node:events';
import { finished } from 'node:stream/promises';
import { parseArgs } from 'node:util';

let values;
try {
  ({ values } = parseArgs({
    options: {
      out: { type: 'string', short: 'o', default: path.join(import.meta.dirname, '..', '..', 'data', 'access.log') },
      'size-mb': { type: 'string', default: '100' },
      seed: { type: 'string', default: '1' },
    },
  }));
} catch (error) {
  // parseArgs throws on an unknown flag or a missing value. Turn that into a usage error, exit 2.
  console.error(`generate-log: ${error.message}`);
  process.exit(2);
}

const sizeMb = Number(values['size-mb']);
const seed = Number(values.seed);
if (!Number.isFinite(sizeMb) || sizeMb <= 0 || !Number.isInteger(seed)) {
  console.error('generate-log: --size-mb must be a positive number and --seed an integer');
  process.exit(2);
}
const targetBytes = Math.round(sizeMb * 1024 * 1024);

// A small, fast, seedable random number generator (mulberry32). Math.random cannot be seeded.
function makeRandom(initial) {
  let state = initial >>> 0;
  return () => {
    state = (state + 0x6d2b79f5) >>> 0;
    let t = state;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const random = makeRandom(seed);

// Pick one entry from a list of [value, weight] pairs.
function weighted(entries) {
  const total = entries.reduce((sum, [, weight]) => sum + weight, 0);
  let roll = random() * total;
  for (const [value, weight] of entries) {
    roll -= weight;
    if (roll < 0) return value;
  }
  return entries.at(-1)[0];
}

const ROUTES = [
  [['GET', '/'], 20],
  [['GET', '/api/tasks'], 25],
  [['POST', '/api/tasks'], 8],
  [['GET', '/api/tasks/1'], 6],
  [['GET', '/api/tasks/2'], 5],
  [['GET', '/api/tasks/3'], 4],
  [['PUT', '/api/tasks/1'], 3],
  [['DELETE', '/api/tasks/2'], 2],
  [['GET', '/api/users'], 7],
  [['GET', '/api/users/me'], 6],
  [['POST', '/api/login'], 5],
  [['POST', '/api/logout'], 2],
  [['GET', '/health'], 9],
  [['GET', '/static/app.js'], 6],
  [['GET', '/static/app.css'], 5],
  [['GET', '/favicon.ico'], 3],
  [['GET', '/admin'], 1],
  [['GET', '/api/reports'], 2],
];
const STATUSES = [
  [200, 70],
  [201, 5],
  [204, 3],
  [301, 2],
  [304, 6],
  [400, 3],
  [401, 2],
  [404, 5],
  [500, 2.5],
  [503, 1.5],
];

const START_MS = Date.UTC(2026, 0, 1);
let clock = START_MS;

function makeLine() {
  clock += Math.floor(random() * 60);
  const timestamp = new Date(clock).toISOString();
  const [method, route] = weighted(ROUTES);
  const status = weighted(STATUSES);
  const durationMs = 1 + Math.floor(random() * random() * 900);
  const line = `${timestamp} ${method} ${route} ${status} ${durationMs}`;
  if (random() < 0.001) {
    // Garbled records: cut short, wrong field type, or not a log line at all.
    return weighted([
      [line.slice(0, 20 + Math.floor(random() * 15)), 1],
      [`${timestamp} ${method} ${route} OK ${durationMs}`, 1],
      ['-- connection reset by peer --', 1],
    ]);
  }
  return line;
}

await mkdir(path.dirname(values.out), { recursive: true });
const output = createWriteStream(values.out);

let bytes = 0;
let lines = 0;
const startedAt = performance.now();
while (bytes < targetBytes) {
  // Build roughly 64 KB at a time, so the file is written in chunks, not line by line.
  const chunk = [];
  let chunkBytes = 0;
  while (chunkBytes < 64 * 1024 && bytes + chunkBytes < targetBytes) {
    const line = makeLine() + '\n';
    chunk.push(line);
    chunkBytes += line.length; // every character here is ASCII, so length equals bytes
    lines++;
  }
  bytes += chunkBytes;
  // write() returns false when its internal buffer is full. Wait for 'drain' before writing more,
  // otherwise the whole file piles up in memory, which is the thing this lab is about avoiding.
  if (!output.write(chunk.join(''))) await once(output, 'drain');
}
output.end();
await finished(output);

const seconds = ((performance.now() - startedAt) / 1000).toFixed(1);
console.log(`Wrote ${lines} lines, ${(bytes / 1024 / 1024).toFixed(1)} MB, to ${values.out} in ${seconds}s (seed ${seed})`);
```

Run it, and then run it a second time to a different path:

```bash
npm run generate-log
npm run generate-log -- --out /tmp/m03-second.log
ls -l data/access.log
wc -l data/access.log
shasum -a 256 data/access.log /tmp/m03-second.log
head -3 data/access.log
```

**Why it is written this way:**

- Each line is `<ISO timestamp> <METHOD> <path> <status> <duration ms>`, for example `2026-01-01T00:00:00.037Z GET / 200 856`. About one line in a thousand is garbled on purpose.
- A **seeded** generator (`Math.random` cannot be seeded) gives the same bytes every run, so reports can be compared.
- It **streams its output** in ~64 KB chunks and waits for `'drain'` when `write()` returns `false`.
- `parseArgs` throws on an unknown flag, so it catches that and exits `2`. Try `node src/log-report/generate-log.js --nope; echo $?`.

**Done when:**

- [ ] `ls -l data/access.log` shows a size of `104857613` bytes (about 100 MB).
- [ ] `wc -l data/access.log` prints a line count of `2231981`.
- [ ] Both `shasum` lines print the same SHA-256, `e28d11f85bdaef172e5e6ce3ea4413f7b145649399e3dde05f696d689fadecd8`.
- [ ] `node src/log-report/generate-log.js --nope; echo $?` prints one error line and then `2`.
- [ ] `git status` does not list `data/`.
- [ ] You can explain what the `drain` wait in the generator protects against.

### Stage 4 — Count the lines (`aggregate.js`)

**Goal:** turn a stream of log lines into counts, using memory that does not grow with the number of lines, as long as the number of distinct paths stays small.

This half of the CLI knows nothing about files, flags or exit codes. That keeps it easy to check with a plain array.

**What it must do**

- `parseLine(line)` returns `{ timestamp, method, path, status, durationMs }` for a good line, else `null`. A good line has exactly five space-separated fields:
  - a timestamp that `Date.parse` accepts,
  - an upper-case method (`/^[A-Z]+$/`),
  - a path starting with `/`,
  - a status from 100 to 599 (`/^[1-5]\d\d$/`), kept as a string,
  - a duration made only of digits, returned as a number.
- `aggregateLines(lines, { top = 5 })` takes **any async iterable of lines** (array, async generator or `readline`) and returns a promise for an object with: `totalLines`, `parsedLines`, `malformedLines`, `firstTimestamp`, `lastTimestamp`, `byStatus`, `byStatusClass` (keys like `2xx`), `byMethod`, `topPaths` (`{ path, count }`, busiest first, ties by path, at most `top`), `averageDurationMs` (one decimal, `0` if nothing parsed) and `maxDurationMs`.
- A blank line is skipped (neither good nor bad). A line that fails `parseLine` counts in `totalLines` and `malformedLines` only. `parsedLines` is `totalLines - malformedLines`.
- Keys of `byStatus`, `byStatusClass` and `byMethod` come out sorted, so the same input gives the same report. Empty input gives `totalLines: 0`, `firstTimestamp: null` and `topPaths: []`.
- It must **not keep lines or parsed records**. Only counters.

**Your turn.** Create `src/log-report/aggregate.js`. The constants are given:

```js
const METHOD = /^[A-Z]+$/;
const STATUS = /^[1-5]\d\d$/;
const DIGITS = /^\d+$/;

// Returns the fields of one log line, or null when the line is not a well-formed record.
export function parseLine(line) {
  // TODO: split the line, check every field, and return the record or null
}

export async function aggregateLines(lines, { top = 5 } = {}) {
  // TODO: loop over the lines with `for await`, count what you see, and build the report.
  // Keep counters (numbers and Maps), never the lines themselves.
}
```

<details>
<summary>Hint 1</summary>

`for await (const line of lines)` works on an array as well as a stream, so one function serves your checks and the real file. Use one `Map` per thing you count and a helper that adds one to a key. A `Map`'s size depends on how many *different* keys exist, not how many lines you read.

</details>

<details>
<summary>Hint 2</summary>

Turn a `Map` into a sorted plain object with `Object.fromEntries([...map].sort(...))`. For `topPaths`, sort by count descending and decide what breaks ties, or two runs could differ.

</details>

<details>
<summary>Hint 3</summary>

The status class is the first character of the status plus `xx`. `firstTimestamp` is set once, by the first good record (`??=` helps), and `lastTimestamp` is overwritten by every good record.

</details>

**Self-checks.** Create `src/log-report/aggregate.check.js` and run it with `node src/log-report/aggregate.check.js`. It fails until your code works, and its messages say what is wrong.

<details>
<summary>The self-check file, aggregate.check.js (open it, copy it, then run it)</summary>

```js
// Module 3 output, deliverable 1 of 2 (log-report CLI): self-checks for aggregate.js.
// Run from modules/m03-nodejs-runtime-npm: node src/log-report/aggregate.check.js
// Success: 6 "ok - ..." lines and no error.
import assert from 'node:assert/strict';
import { aggregateLines, parseLine } from './aggregate.js';

async function check(name, fn) {
  await fn();
  console.log(`ok - ${name}`);
}

const GOOD = '2026-01-01T00:00:00.037Z GET /api/tasks 200 856';

await check('parseLine reads a well-formed record', () => {
  assert.deepEqual(parseLine(GOOD), {
    timestamp: '2026-01-01T00:00:00.037Z',
    method: 'GET',
    path: '/api/tasks',
    status: '200',
    durationMs: 856,
  });
});

await check('parseLine returns null for every kind of bad line', () => {
  const bad = [
    '',
    '-- connection reset by peer --',
    '2026-01-01T00:00:00.037Z GET /api/tasks 200', // a field is missing
    '2026-01-01T00:00:00.037Z GET /api/tasks OK 856', // status is not a number
    '2026-01-01T00:00:00.037Z GET /api/tasks 700 856', // not an HTTP status class
    'not-a-date GET /api/tasks 200 856',
    '2026-01-01T00:00:00.037Z get /api/tasks 200 856', // method must be upper case
    '2026-01-01T00:00:00.037Z GET api/tasks 200 856', // path must start with /
    '2026-01-01T00:00:00.037Z GET /api/tasks 200 -5', // negative duration
    `${GOOD} extra`,
  ];
  for (const line of bad) assert.equal(parseLine(line), null, `expected null for "${line}"`);
});

const SAMPLE = [
  '2026-01-01T00:00:00.001Z GET /a 200 10',
  '2026-01-01T00:00:00.002Z GET /b 404 20',
  '2026-01-01T00:00:00.003Z POST /a 201 30',
  '',
  'garbage',
  '2026-01-01T00:00:00.004Z GET /b 500 40',
  '2026-01-01T00:00:00.005Z GET /c 200 50',
];

await check('aggregateLines counts statuses, classes, methods and paths', async () => {
  const report = await aggregateLines(SAMPLE);
  assert.equal(report.totalLines, 6, 'the blank line is skipped, the garbage line counts');
  assert.equal(report.malformedLines, 1);
  assert.equal(report.parsedLines, 5);
  assert.deepEqual(report.byStatus, { 200: 2, 201: 1, 404: 1, 500: 1 });
  assert.deepEqual(report.byStatusClass, { '2xx': 3, '4xx': 1, '5xx': 1 });
  assert.deepEqual(report.byMethod, { GET: 4, POST: 1 });
  assert.equal(report.firstTimestamp, '2026-01-01T00:00:00.001Z');
  assert.equal(report.lastTimestamp, '2026-01-01T00:00:00.005Z');
  assert.equal(report.averageDurationMs, 30);
  assert.equal(report.maxDurationMs, 50);
});

await check('topPaths is ordered by count then path, and limited by top', async () => {
  const report = await aggregateLines(SAMPLE, { top: 2 });
  assert.deepEqual(report.topPaths, [
    { path: '/a', count: 2 },
    { path: '/b', count: 2 },
  ]);
});

await check('aggregateLines accepts an async generator, not only an array', async () => {
  async function* lines() {
    for (let i = 0; i < 1000; i++) yield GOOD;
    yield 'garbage';
  }
  const report = await aggregateLines(lines());
  assert.equal(report.totalLines, 1001);
  assert.equal(report.byStatus['200'], 1000);
  assert.equal(report.malformedLines, 1);
});

await check('an empty input gives an empty report, not a crash', async () => {
  const report = await aggregateLines([]);
  assert.equal(report.totalLines, 0);
  assert.equal(report.firstTimestamp, null);
  assert.equal(report.averageDurationMs, 0);
  assert.deepEqual(report.topPaths, []);
});
```

</details>

<details>
<summary>Reference implementation (open it after you have tried)</summary>

```js
// Module 3 output, deliverable 1 of 2 (log-report CLI): the counting half, `parseLine` and `aggregateLines`.
// Nothing to run here. log-report.js uses it, and aggregate.check.js tests it.
// It knows nothing about files, flags or exit codes, so it can be checked with a plain array.

const METHOD = /^[A-Z]+$/;
const STATUS = /^[1-5]\d\d$/;
const DIGITS = /^\d+$/;

// Returns the fields of one log line, or null when the line is not a well-formed record.
export function parseLine(line) {
  const parts = line.split(' ');
  if (parts.length !== 5) return null;
  const [timestamp, method, path, status, duration] = parts;
  if (Number.isNaN(Date.parse(timestamp))) return null;
  if (!METHOD.test(method) || !path.startsWith('/')) return null;
  if (!STATUS.test(status) || !DIGITS.test(duration)) return null;
  return { timestamp, method, path, status, durationMs: Number(duration) };
}

function increment(map, key) {
  map.set(key, (map.get(key) ?? 0) + 1);
}

// Turn a Map of counts into a plain object with its keys in a fixed order.
function toSortedObject(map) {
  return Object.fromEntries([...map].sort(([a], [b]) => (a < b ? -1 : a > b ? 1 : 0)));
}

// Consumes any async iterable of lines, one at a time, and keeps only the counters. Memory use
// depends on how many distinct statuses, methods and paths exist, never on how many lines there are.
export async function aggregateLines(lines, { top = 5 } = {}) {
  let totalLines = 0;
  let malformedLines = 0;
  let durationSum = 0;
  let maxDurationMs = 0;
  let firstTimestamp = null;
  let lastTimestamp = null;
  const byStatus = new Map();
  const byStatusClass = new Map();
  const byMethod = new Map();
  const byPath = new Map();

  for await (const line of lines) {
    if (line === '') continue; // a blank line is not a record, good or bad
    totalLines++;
    const record = parseLine(line);
    if (record === null) {
      malformedLines++;
      continue;
    }
    increment(byStatus, record.status);
    increment(byStatusClass, `${record.status[0]}xx`);
    increment(byMethod, record.method);
    increment(byPath, record.path);
    durationSum += record.durationMs;
    if (record.durationMs > maxDurationMs) maxDurationMs = record.durationMs;
    firstTimestamp ??= record.timestamp;
    lastTimestamp = record.timestamp;
  }

  const parsedLines = totalLines - malformedLines;
  const topPaths = [...byPath]
    .sort(([pathA, countA], [pathB, countB]) => countB - countA || (pathA < pathB ? -1 : 1))
    .slice(0, top)
    .map(([path, count]) => ({ path, count }));

  return {
    totalLines,
    parsedLines,
    malformedLines,
    firstTimestamp,
    lastTimestamp,
    byStatus: toSortedObject(byStatus),
    byStatusClass: toSortedObject(byStatusClass),
    byMethod: toSortedObject(byMethod),
    topPaths,
    averageDurationMs: parsedLines === 0 ? 0 : Math.round((durationSum / parsedLines) * 10) / 10,
    maxDurationMs,
  };
}
```

</details>

**Compare with your version.**

- It never stores a line, so its memory does not grow with the number of lines, as long as the number of distinct paths stays small. It takes an async iterable, so the streaming lives one level up.
- Sorted keys and tie-breaking by path make the report identical every run.

**Done when:**

- [ ] `node src/log-report/aggregate.check.js` prints six `ok - ...` lines and no error.
- [ ] `aggregateLines` keeps counters only (nothing in your code grows with the number of lines, only with the number of distinct paths), and you can explain why it takes an async iterable and not a file name.

### Stage 5 — The streaming CLI (`log-report.js`)

**Goal:** a command-line program that streams the log into `aggregateLines`, reads options with `parseArgs`, exits with sensible codes, and writes the JSON report.

**What it must do**

| Option | Meaning |
| --- | --- |
| `-i`, `--input <file>` | the log to read. Default: `data/access.log` at the module root, two folders up from the script (use `import.meta.dirname`). |
| `-o`, `--output <file>` | the report to write. Default: `report.json` in the current directory. |
| `-n`, `--top <count>` | how many paths `topPaths` lists. Default `5`. A whole number of at least 1. |
| `--stats` | when done, print elapsed time and peak memory to **stderr** (not into the report). |
| `-h`, `--help` | print the usage and exit 0. |

| Situation | Exit code | Message goes to |
| --- | --- | --- |
| report written | 0 | stdout: `Wrote <file>: <n> lines, <m> malformed` |
| input file missing | 1 | stderr, one line: `cannot read <file>: no such file or directory`, no stack trace, no report file |
| output folder missing | 1 | stderr, one line: `cannot write <file>: no such file or directory`, no stack trace |
| unknown flag, or bad `--top` | 2 | stderr, the problem (a flag error also shows the usage) |
| stopped by Ctrl+C (`SIGINT`) | 130 | stderr: `stopped by SIGINT, no report written` |
| stopped by `SIGTERM` | 143 | stderr: same, naming the signal |

- The report is `{ source: <the input file's name>, ...counts }`, written as `JSON.stringify(report, null, 2)` plus a newline. It must be identical every run, so timing and memory go to stderr.
- It must **stream**: read in chunks, hand over one line at a time, never read the whole file into one string or array.
- A signal must stop the read cleanly and write no report. A second signal must end the process at once, however stuck it is.

**Your turn.** Create `src/log-report/log-report.js`. The imports, the usage text and the `fail` helper are given:

```js
import { createReadStream } from 'node:fs';
import { writeFile } from 'node:fs/promises';
import path from 'node:path';
import readline from 'node:readline';
import { parseArgs } from 'node:util';
import { aggregateLines } from './aggregate.js';

const USAGE = `Usage: node src/log-report/log-report.js [options]

Options:
  -i, --input <file>   log file to read      (default: data/access.log at the module root)
  -o, --output <file>  report file to write  (default: report.json in the current directory)
  -n, --top <count>    how many paths to list in topPaths (default: 5)
      --stats          print elapsed time and peak memory to stderr when done
  -h, --help           show this help`;

function fail(message, code) {
  console.error(`log-report: ${message}`);
  process.exitCode = code;
}

// TODO 1: read the options with parseArgs. It throws on an unknown flag: catch that and fail(..., 2).
// TODO 2: for --help, print USAGE and stop. Otherwise run the program.

async function run({ input, output, top, stats }) {
  // TODO 3: validate `top` (a whole number >= 1), or fail(..., 2).
  // TODO 4: make an AbortController and handle SIGINT and SIGTERM (see the hints).
  // TODO 5: stream `input` into aggregateLines, write the report, print the message,
  //         and print the stats line when --stats is set.
  // TODO 6: turn errors into exit codes: an abort, a missing file (ENOENT: the input or the output?), anything else.
}
```

<details>
<summary>Hint 1</summary>

`createReadStream(input, { encoding: 'utf8', signal })` reads in chunks (64 KiB by default). `readline.createInterface({ input: stream, crlfDelay: Infinity })` turns the stream into an async iterable of lines that `aggregateLines` can consume directly.

</details>

<details>
<summary>Hint 2</summary>

Set `process.exitCode = n` and let the program finish. `process.exit(n)` can cut off output that has not been written yet. For stats, `process.resourceUsage().maxRSS` is peak memory in **kilobytes**, and `performance.now()` gives elapsed milliseconds.

</details>

<details>
<summary>Hint 3</summary>

`process.on('SIGINT', ...)` replaces Node's default "exit now", so you must exit yourself. Call `controller.abort(signalName)` and set the exit code (130 for `SIGINT`, 143 for `SIGTERM`). An error with `error.name === 'AbortError'` means the read was stopped, and `controller.signal.reason` holds the name you passed.

Test Ctrl+C on a read that is *stuck*, not only one that works. In testing, a read stuck on a pipe that never sends data could not be stopped politely, and even `process.exit()` did not end the process. So a second Ctrl+C should remove your handler (`process.removeAllListeners(name)`) and send the same signal to yourself (`process.kill(process.pid, name)`). Node's default handler then ends the process at once.

</details>

<details>
<summary>Hint 4</summary>

A missing input file and a missing output folder are both `ENOENT`, and `error.path` names the path that failed. How will your `catch` know whether it was reading or writing when the error happened?

</details>

**Self-checks.** Create `src/log-report/log-report.check.js`. Run `node src/log-report/log-report.js --help` first to see your usage text, then run the checks. They start the real CLI as a child process, so they test what a user would see. One check generates a 300 MB log, so the run takes about ten seconds.

<details>
<summary>The self-check file, log-report.check.js (open it, copy it, then run it)</summary>

```js
// Module 3 output, deliverable 1 of 2 (log-report CLI): end-to-end checks that run the real CLI as a child process.
// Run from modules/m03-nodejs-runtime-npm: node src/log-report/log-report.check.js
// Success: 9 "ok - ..." lines and no error. It generates a 300 MB log, so it takes about 10 seconds.
import assert from 'node:assert/strict';
import { execFile, spawn } from 'node:child_process';
import { mkdtemp, readFile, rm, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import path from 'node:path';
import { once } from 'node:events';
import { promisify } from 'node:util';

const run = promisify(execFile);
const CLI = path.join(import.meta.dirname, 'log-report.js');

async function check(name, fn) {
  await fn();
  console.log(`ok - ${name}`);
}

// Resolves when the child has ended. The timeout makes a child that never ends fail the check
// instead of hanging the whole run.
const waitForClose = (child) => once(child, 'close', { signal: AbortSignal.timeout(15_000) });

// execFile rejects on a non-zero exit code. Turn that into a result so checks can assert on it.
async function runCli(args) {
  try {
    const { stdout, stderr } = await run(process.execPath, [CLI, ...args]);
    return { code: 0, stdout, stderr };
  } catch (error) {
    return { code: error.code, stdout: error.stdout, stderr: error.stderr };
  }
}

const dir = await mkdtemp(path.join(tmpdir(), 'm03-log-report-'));
try {
  const input = path.join(dir, 'in.log');
  const output = path.join(dir, 'out.json');
  await writeFile(
    input,
    [
      '2026-01-01T00:00:00.001Z GET /a 200 10',
      '2026-01-01T00:00:00.002Z GET /b 404 20',
      'garbage',
      '2026-01-01T00:00:00.003Z POST /a 201 30',
      '',
    ].join('\n'),
  );

  await check('writes a JSON report with the right counts and exits 0', async () => {
    const result = await runCli(['-i', input, '-o', output]);
    assert.equal(result.code, 0);
    const report = JSON.parse(await readFile(output, 'utf8'));
    assert.equal(report.source, 'in.log');
    assert.equal(report.totalLines, 4);
    assert.equal(report.malformedLines, 1);
    assert.deepEqual(report.byStatus, { 200: 1, 201: 1, 404: 1 });
    assert.deepEqual(report.topPaths, [
      { path: '/a', count: 2 },
      { path: '/b', count: 1 },
    ]);
  });

  await check('--top limits topPaths', async () => {
    const result = await runCli(['-i', input, '-o', output, '--top', '1']);
    assert.equal(result.code, 0);
    assert.equal(JSON.parse(await readFile(output, 'utf8')).topPaths.length, 1);
  });

  await check('--stats prints elapsed time and peak memory to stderr', async () => {
    const result = await runCli(['-i', input, '-o', output, '--stats']);
    assert.equal(result.code, 0);
    assert.match(result.stderr, /^stats: [\d.]+s elapsed, peak memory [\d.]+ MB/m);
  });

  await check('a missing input or an unwritable output exits 1, names which one, and writes no report', async () => {
    const missingOutput = path.join(dir, 'never.json');
    const noInput = await runCli(['-i', path.join(dir, 'nope.log'), '-o', missingOutput]);
    assert.equal(noInput.code, 1);
    assert.match(noInput.stderr, /cannot read .*nope\.log: no such file or directory/);
    await assert.rejects(readFile(missingOutput), { code: 'ENOENT' });

    const noFolder = path.join(dir, 'no-such-folder', 'out.json');
    const noOutput = await runCli(['-i', input, '-o', noFolder]);
    assert.equal(noOutput.code, 1);
    assert.match(noOutput.stderr, /cannot write .*out\.json: no such file or directory/);
  });

  await check('an unknown flag exits 2 and shows the usage', async () => {
    const result = await runCli(['--nope']);
    assert.equal(result.code, 2);
    assert.match(result.stderr, /Unknown option '--nope'/);
    assert.match(result.stderr, /Usage: node src\/log-report\/log-report\.js/);
  });

  await check('a bad --top exits 2', async () => {
    for (const bad of ['0', '-3', 'abc', '1.5']) {
      const result = await runCli(['-i', input, '-o', output, '--top', bad]);
      assert.equal(result.code, 2, `--top ${bad}`);
    }
  });

  await check('--help prints the usage and exits 0', async () => {
    const result = await runCli(['--help']);
    assert.equal(result.code, 0);
    assert.match(result.stdout, /Usage: node src\/log-report\/log-report\.js/);
  });

  await check('SIGINT stops a run in progress: exit 130 and no report', async () => {
    // A 300 MB log takes a few seconds to read, so a signal at 0.7 s lands well inside the run. The wait
    // gives the child time to start and install its handlers (the CLI prints nothing we could wait for).
    const bigLog = path.join(dir, 'big.log');
    await run(process.execPath, [path.join(import.meta.dirname, 'generate-log.js'), '--size-mb', '300', '-o', bigLog]);
    const interruptedOutput = path.join(dir, 'interrupted.json');
    const child = spawn(process.execPath, [CLI, '-i', bigLog, '-o', interruptedOutput], {
      stdio: ['ignore', 'ignore', 'pipe'],
    });
    const closed = waitForClose(child);
    let stderr = '';
    child.stderr.on('data', (chunk) => (stderr += chunk));
    await new Promise((resolve) => setTimeout(resolve, 700));
    child.kill('SIGINT');
    const [code, signal] = await closed;
    assert.equal(code, 130, `got code ${code}, signal ${signal}: null and SIGINT means the signal arrived before the handler existed`);
    assert.match(stderr, /stopped by SIGINT/);
    await assert.rejects(readFile(interruptedOutput), { code: 'ENOENT' });
  });

  if (process.platform === 'win32') {
    console.log('skip - the stuck-read check needs /dev/stdin, which Windows does not have');
  } else {
    await check('a second SIGINT forces the exit when a read is stuck', async () => {
      // Reading /dev/stdin from a pipe that never sends data is a read that cannot finish.
      const child = spawn(process.execPath, [CLI, '-i', '/dev/stdin', '-o', path.join(dir, 'stuck.json')], {
        stdio: ['pipe', 'ignore', 'ignore'],
      });
      const closed = waitForClose(child);
      try {
        await new Promise((resolve) => setTimeout(resolve, 700));
        child.kill('SIGINT');
        await new Promise((resolve) => setTimeout(resolve, 700));
        // A process killed by a signal has exitCode null too, so check signalCode as well.
        assert.ok(child.exitCode === null && child.signalCode === null, 'the first SIGINT alone must not end a stuck read');
        child.kill('SIGINT');
        // The second signal ends the process through Node's default handler, so it dies "by SIGINT"
        // (no exit code of its own). A shell reports that as 130.
        const [code, signal] = await closed;
        assert.equal(code, null);
        assert.equal(signal, 'SIGINT');
      } finally {
        child.kill('SIGKILL'); // never leave a stuck child behind if an assertion above failed
      }
    });
  }
} finally {
  await rm(dir, { recursive: true, force: true });
}
```

</details>

<details>
<summary>Reference implementation (open it after you have tried)</summary>

```js
// Module 3 output, deliverable 1 of 2 (log-report CLI): streams a log file line by line into a JSON report.
// Run from modules/m03-nodejs-runtime-npm: npm run report -- --stats   (add --help to see every option)
// Success: "Wrote report.json: 2231981 lines, 2200 malformed" for the default generated log, plus a
// `stats:` line with the elapsed time and peak memory.
// Exit codes: 0 report written, 1 the run failed (for example the input is missing),
// 2 bad command line, 130 stopped by SIGINT (Ctrl+C), 143 stopped by SIGTERM.
import { createReadStream } from 'node:fs';
import { writeFile } from 'node:fs/promises';
import path from 'node:path';
import readline from 'node:readline';
import { parseArgs } from 'node:util';
import { aggregateLines } from './aggregate.js';

// ---- Options and usage --------------------------------------------------------------------------
const USAGE = `Usage: node src/log-report/log-report.js [options]

Options:
  -i, --input <file>   log file to read      (default: data/access.log at the module root)
  -o, --output <file>  report file to write  (default: report.json in the current directory)
  -n, --top <count>    how many paths to list in topPaths (default: 5)
      --stats          print elapsed time and peak memory to stderr when done
  -h, --help           show this help`;

function fail(message, code) {
  console.error(`log-report: ${message}`);
  process.exitCode = code;
}

let values;
try {
  ({ values } = parseArgs({
    options: {
      input: { type: 'string', short: 'i', default: path.join(import.meta.dirname, '..', '..', 'data', 'access.log') },
      output: { type: 'string', short: 'o', default: 'report.json' },
      top: { type: 'string', short: 'n', default: '5' },
      stats: { type: 'boolean', default: false },
      help: { type: 'boolean', short: 'h', default: false },
    },
  }));
} catch (error) {
  fail(`${error.message}\n\n${USAGE}`, 2);
}

if (values?.help) {
  console.log(USAGE);
} else if (values) {
  await run(values);
}

// ---- The run: stream the file, count, write the report ------------------------------------------
async function run({ input, output, top, stats }) {
  const topCount = Number(top);
  if (!Number.isInteger(topCount) || topCount < 1) {
    return fail(`--top must be a whole number of at least 1, got "${top}"`, 2);
  }

  // One AbortController lets a signal stop the read stream cleanly instead of killing the process.
  // Installing a handler replaces Node's default "exit now" behaviour, so Ctrl+C must never be
  // allowed to leave the process stuck. In testing, a read blocked on a pipe that never sends data
  // could not be abandoned politely, and even process.exit() did not end the process. So a second
  // signal removes our handler and re-sends the signal, and Node's default ends the process at once.
  const controller = new AbortController();
  const stop = (signalName, code) => () => {
    if (controller.signal.aborted) {
      process.removeAllListeners(signalName);
      process.kill(process.pid, signalName);
      return;
    }
    process.exitCode = code;
    controller.abort(signalName);
  };
  process.on('SIGINT', stop('SIGINT', 130));
  process.on('SIGTERM', stop('SIGTERM', 143));

  const startedAt = performance.now();
  let writing = false; // tells a missing input from a missing output folder, which are both ENOENT
  try {
    const source = createReadStream(input, { encoding: 'utf8', signal: controller.signal });
    // readline hands over one line at a time and only reads as fast as we consume, so the 100 MB
    // file never has to exist in memory all at once.
    const lines = readline.createInterface({ input: source, crlfDelay: Infinity });
    const counts = await aggregateLines(lines, { top: topCount });

    const report = { source: path.basename(input), ...counts };
    writing = true;
    await writeFile(output, JSON.stringify(report, null, 2) + '\n');
    console.log(`Wrote ${output}: ${counts.totalLines} lines, ${counts.malformedLines} malformed`);

    if (stats) {
      const seconds = ((performance.now() - startedAt) / 1000).toFixed(2);
      const peakMb = (process.resourceUsage().maxRSS / 1024).toFixed(1);
      console.error(`stats: ${seconds}s elapsed, peak memory ${peakMb} MB (maxRSS)`);
    }
  } catch (error) {
    if (error.name === 'AbortError') {
      console.error(`log-report: stopped by ${controller.signal.reason}, no report written`);
    } else if (error.code === 'ENOENT') {
      fail(`cannot ${writing ? 'write' : 'read'} ${error.path}: no such file or directory`, 1);
    } else {
      fail(error.message, 1);
    }
  }
}
```

</details>

**Compare with your version.**

- The streaming is three lines: `createReadStream`, `readline.createInterface`, `aggregateLines`. A signal aborts it through the `AbortController`, so nothing half-finished is written.
- The second-signal escape exists because a stuck read could not be stopped politely. `--stats` goes to stderr so the report stays identical every run.
- A missing input and a missing output folder are both `ENOENT`, so the program remembers which step it was in.

**Run it on the full file:**

```bash
npm run report -- --stats
node -p "const r = require('./report.json'); [r.totalLines, r.malformedLines, r.byStatusClass]"
shasum -a 256 report.json
```

The first command should print `Wrote report.json: 2231981 lines, 2200 malformed` and a `stats:` line with a time under about a second on the reference machine and a peak memory figure. Then try the failure cases:

```bash
node src/log-report/log-report.js -i nope.log; echo "exit=$?"
node src/log-report/log-report.js --nope; echo "exit=$?"
node src/log-report/log-report.js --top 0; echo "exit=$?"
node --run report -- -n 3 --stats
```

**Done when:**

- [ ] `node src/log-report/aggregate.check.js && node src/log-report/log-report.check.js` prints 15 `ok - ...` lines and no error.
- [ ] `npm run report -- --stats` on the full file prints `Wrote report.json: 2231981 lines, 2200
      malformed`, and `report.json` has a SHA-256 of
      `0eb16fc1e6b84dd1f0e5cc6097034f84f645a69ad221bb002a9fd4fd43354cfa` (for the file `generate-log.js`
      writes with its default settings).
- [ ] A missing input or an output folder that does not exist exits with `1`, an unknown flag or `--top 0`
      exits with `2`, and none of them leaves a stack trace or a report file.
- [ ] You can explain, step by step, what happens to the 100 MB file from `createReadStream` to the counters, and why a second Ctrl+C must end the process by a different route from the first.

### Stage 6 — Check the memory behaviour on the full file

**Goal:** prove with numbers that your CLI streams, by comparing it with the easy version that does not.

On a small file both versions give the same answer, so correctness on a sample proves nothing. Only the full file shows the difference.

**Step 1 — Measure your CLI.** Two measurements, which should roughly agree:

```bash
npm run report -- --stats                                  # prints maxRSS, in MB
/usr/bin/time -l node src/log-report/log-report.js         # macOS: look for "maximum resident set size" (bytes)
/usr/bin/time -v node src/log-report/log-report.js         # Linux: look for "Maximum resident set size" (KB)
```

**Step 2 — Write the easy version** in your scratch folder (`~/m03-scratch/whole-file.mjs`, not in the repo). It loads the whole file into one string, splits it, and reuses your `aggregateLines`:

```js
// The "easy" version, for comparison only: read the whole file, then split it into lines.
import { readFile } from 'node:fs/promises';

const [, , logPath, aggregatePath] = process.argv;
const { aggregateLines } = await import(aggregatePath);

const text = await readFile(logPath, 'utf8');
const lines = text.split('\n');
const counts = await aggregateLines(lines);
console.log(`whole-file: ${counts.totalLines} lines, ${counts.malformedLines} malformed`);
console.log(`peak memory ${(process.resourceUsage().maxRSS / 1024).toFixed(0)} MB (maxRSS)`);
```

It takes the log path and your `src/log-report/aggregate.js` path as arguments. From your stage folder:

```bash
node ~/m03-scratch/whole-file.mjs data/access.log "$PWD/src/log-report/aggregate.js"
```

**Step 3 — Take away the memory.** `--max-old-space-size` (in MB) caps V8's old-generation heap. Run both programs under a small cap:

```bash
node --max-old-space-size=32 src/log-report/log-report.js --stats
node --max-old-space-size=64 ~/m03-scratch/whole-file.mjs data/access.log "$PWD/src/log-report/aggregate.js"; echo "exit=$?"
```

**Step 4 (optional, 600 MB) — find the hard limit.**

```bash
node src/log-report/generate-log.js --size-mb 600 --out /tmp/m03-600.log
node src/log-report/log-report.js -i /tmp/m03-600.log --stats -o /tmp/m03-600-report.json
node ~/m03-scratch/whole-file.mjs /tmp/m03-600.log "$PWD/src/log-report/aggregate.js"
node -p "require('node:buffer').constants.MAX_STRING_LENGTH"
```

**What the reference run showed** (macOS, Apple silicon, Node 24; your sizes will differ, the shape should not):

| Run | Result |
| --- | --- |
| streaming CLI, 100 MB file | peak about 85 MB, under 1 s |
| whole-file version, 100 MB file | peak about 820 MB: about eight times the file and about ten times the streaming run (this peak can vary between runs) |
| streaming CLI, `--max-old-space-size=32` | still succeeds |
| whole-file version, `--max-old-space-size=64` | `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory`, exit 134 |
| streaming CLI, 600 MB file | succeeds, peak about 117 MB |
| whole-file version, 600 MB file | `RangeError: Invalid string length` |
| `MAX_STRING_LENGTH` | `536870888`, about 512 MiB |

The streaming peak is not perfectly flat: about 85 MB for the 100 MB file and about 117 MB for the 600 MB file.

- It does not look like kept data: the 600 MB run still passed under a 32 MB heap cap.
- With `node --trace-gc src/log-report/log-report.js -i /tmp/m03-600.log -o /tmp/x.json` the reference run counted 649 short Scavenge collections and 0 Mark-Compact ones without a cap, and 475 and 174 under the 32 MB cap, where it still finished.
- The likely cause is short-lived buffers that V8 has not yet returned to the operating system, but the reference run only counted collections.
- A bare `node -e 0` already peaks at about 44 MB, so most of the baseline is Node itself.

**Done when:**

- [ ] You ran your CLI and the whole-file version on the full file, and wrote down both peak-memory
      figures.
- [ ] Under `--max-old-space-size=32` your CLI finishes the full file, and the whole-file version fails
      under a larger cap (64 MB) with a heap out-of-memory error.
- [ ] You can explain why a small sample would not show which version streams, using your own two numbers, and why a file above about 512 MiB can never be read into one string, however much memory the machine has.

### Stage 7 — The bare server, first try: write the obvious version and watch it fail

**Goal:** a working `node:http` server with three JSON endpoints, written the way most people first write it, so you can see where it goes wrong.

The endpoints, on an in-memory store of notes:

| Request | Meaning |
| --- | --- |
| `GET /notes` | list every note |
| `POST /notes` with a JSON body `{ "text": "..." }` | create a note |
| `GET /notes/:id` | read one note |

A note is `{ id, text, createdAt }`. The `id` comes from `crypto.randomUUID()`.

Create `src/http-server/notes-server.js` with this deliberately naive version. It exports a function that builds the server and never calls `listen`, so a script and tests can both start it:

```js
import http from 'node:http';
import { randomUUID } from 'node:crypto';

export function createNotesServer() {
  const notes = new Map();

  return http.createServer(async (req, res) => {
    const url = new URL(req.url, `http://${req.headers.host}`);

    if (req.method === 'GET' && url.pathname === '/notes') {
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify([...notes.values()]));
      return;
    }

    if (req.method === 'POST' && url.pathname === '/notes') {
      let body = '';
      for await (const chunk of req) body += chunk;
      const data = JSON.parse(body);
      const note = { id: randomUUID(), text: data.text, createdAt: new Date().toISOString() };
      notes.set(note.id, note);
      res.writeHead(201, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(note));
      return;
    }

    if (req.method === 'GET' && url.pathname.startsWith('/notes/')) {
      const note = notes.get(url.pathname.slice('/notes/'.length));
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(note));
      return;
    }

    res.writeHead(404);
    res.end();
  });
}
```

And a minimal `src/http-server/server.js` that starts it (you will improve it in Stage 8):

```js
import { createNotesServer } from './notes-server.js';

const port = Number(process.env.PORT ?? 3000);
createNotesServer().listen(port, () => {
  console.log(`notes server listening on http://localhost:${port}`);
});
```

Start it in one terminal with `npm run serve`. Without a `.env` file, Node prints `.env not found. Continuing without it.`, which is normal. In a second terminal, run these in order:

```bash
curl -s -i -X POST localhost:3000/notes -H 'Content-Type: application/json' -d '{"text":"first note"}'
curl -s localhost:3000/notes
curl -s -i localhost:3000/notes/does-not-exist
curl -s -i -X POST localhost:3000/notes -H 'Content-Type: application/json' -d '{"text": oops'
curl -s --max-time 3 localhost:3000/notes; echo "curl exit=$?"
```

**What you will see:**

- **Commands 1 and 2 work:** a `201` with the note as JSON, then a list containing it.
- **Command 3 (unknown id) answers `HTTP/1.1 200 OK` with an empty body.** No 404, no JSON. `notes.get()` returned `undefined`, `JSON.stringify(undefined)` is `undefined`, and `res.end(undefined)` sends nothing.
- **Command 4 (malformed JSON) gets no answer, and the server process dies.** Its terminal shows `SyntaxError: Unexpected token 'o', "{"text": oops" is not valid JSON` and the line `Node.js v24...`. Command 5 then prints `curl exit=7` (connection refused).
- **Why:** an `async` handler's rejection is not caught by `http.createServer`. It becomes an unhandled rejection and Node exits. One bad request took the service down for everyone.

Write down what else you notice: what the code trusts, what it forgets to check, what the client has to guess. That list is your answer to Stage 11.

**Done when:**

- [ ] `npm run serve` prints `notes server listening on http://localhost:3000`, `POST /notes` returns `201` with a JSON note, and `GET /notes` lists it.
- [ ] You reproduced both failures: `GET /notes/does-not-exist` answers `200` with an empty body, and the
      malformed-JSON `POST` kills the server (the next `curl` exits with `7`).
- [ ] You can explain why the malformed-JSON request kills the whole server and not just that one request.

### Stage 8 — Harden the server by hand, and start it properly

**Goal:** replace the naive handler with one that routes, parses bodies and reports errors correctly, without a framework, and give `server.js` proper start-up and shutdown.

Work in the order the failures asked for: make errors safe first, then routing, then body parsing.

**What `notes-server.js` must do**

| Request | Response |
| --- | --- |
| `GET /notes` | `200`, a JSON array |
| `POST /notes` with a valid body | `201`, the note as JSON, and a `Location: /notes/<id>` header |
| `GET /notes/:id`, known id | `200`, the note |
| `GET /notes/:id`, unknown id | `404` and a JSON error |
| any other path, including odd ones such as `//` and `//x/notes` | `404` and a JSON error |
| a known path with the wrong method (for example `DELETE /notes`) | `405`, a JSON error, and an `Allow` header listing the allowed methods (`GET, POST` for `/notes`, `GET` for `/notes/:id`) |
| `POST /notes` with malformed JSON | `400`, and **the server keeps running** |
| `POST /notes` with a `Content-Type` that is not JSON | `415` |
| `POST /notes` whose `text` is missing, not a string, blank, or over 1000 characters, or whose body is `null` or an array or empty | `400` |
| `POST /notes` with a body over 1 MiB, declared by `Content-Length` or arriving in chunks without one | `413` |

- Every error response has the same shape: `{ "error": { "status": <number>, "message": "<text>" } }`.
- Every JSON response carries `Content-Type: application/json; charset=utf-8` and a `Content-Length`.
- `/notes`, `/notes/` and `//notes` are the same route (empty path segments are ignored).
- An unexpected exception becomes a logged `500` with a generic message, never the exception text.

**What `server.js` must do**

- **`PORT`:** read it from the environment, default `3000`. It is a string, so convert and check it by hand: a whole number from 0 to 65535, else print one line and exit `2`. An empty `PORT=` must fall back to 3000, not become port 0.
- **Start-up message:** print `notes server listening on http://localhost:<the real port>`. With `PORT=0` the OS picks a free port, so ask the server for it.
- **`listen` failure** (for example the port is in use): print one line (`Port 3000 is already in use`) and exit `1`, with no stack trace.
- **`SIGINT` or `SIGTERM`:** log `<signal> received, closing the server`, stop accepting connections, let requests in flight finish (even one that is half-sent), close keep-alive connections as they go idle, and exit `0` when the last one is done. If that takes more than five seconds, exit `1`. Hint: `server.close()` closes idle connections, but a busy one only becomes idle after its response.

**Your turn.** Rewrite `src/http-server/notes-server.js`. The error class and the response helper are given:

```js
import http from 'node:http';
import { randomUUID } from 'node:crypto';

const MAX_BODY_BYTES = 1024 * 1024;
const MAX_TEXT_LENGTH = 1000;

class HttpError extends Error {
  constructor(status, message, headers = {}) {
    super(message);
    this.status = status;
    this.headers = headers;
  }
}

function sendJson(res, status, body, headers = {}) {
  const payload = JSON.stringify(body);
  res.writeHead(status, {
    'Content-Type': 'application/json; charset=utf-8',
    'Content-Length': Buffer.byteLength(payload),
    ...headers,
  });
  res.end(payload);
}

// TODO 1: readJsonBody(req): check Content-Type (415), refuse a declared Content-Length over the
//         limit (413), count bytes as chunks arrive (413), then parse (400 on bad JSON).

export function createNotesServer() {
  const notes = new Map();

  // TODO 2: the three handlers: listNotes, createNote (validate `text`, 201 + Location), getNote (404).

  // TODO 3: route(method, segments): match by hand, return a handler, throw 405 (with Allow) or return null.

  return http.createServer(async (req, res) => {
    // TODO 4: parse the path, split it into segments, find the handler (or 404), run it,
    //         and make sure EVERY failure ends up as a JSON response from one place.
  });
}
```

<details>
<summary>Hint 1</summary>

A throw inside an `async` handler is not caught by `http.createServer`; you saw it kill the process. Where must one `try/catch` go so that every route, and every helper they call, ends in a JSON response from a single place?

</details>

<details>
<summary>Hint 2</summary>

The body arrives as `Buffer` chunks. Keep them in an array, count bytes as they arrive, and decode once with `Buffer.concat(chunks).toString('utf8')`. Why is that safer than `body += chunk`? (Think of a multi-byte character cut across two chunks.) Also, `JSON.parse('null')` is valid and returns `null`: what does `body.text` do, and what does `body?.text` do?

</details>

<details>
<summary>Hint 3</summary>

Split the path with `pathname.split('/').filter(Boolean)`: `/notes/abc` becomes `['notes', 'abc']`. A known path with an unsupported method is a 405, not a 404. One small object per path shape, mapping methods to handlers, gives you both the dispatch and the `Allow` list.

</details>

<details>
<summary>Hint 4</summary>

Two traps to test on purpose.

**URLs.** `new URL(req.url, base)` needs a base, and the `Host` header is client input, so use a fixed one. Try `new URL('//', 'http://localhost')` and `new URL('//notes', 'http://localhost')`: a leading `//` is read as a host. What would a fixed *prefix* (`http://localhost` followed by `req.url`) do instead?

**Too-big bodies.** A refused body has not finished sending. For a declared `Content-Length`, `Connection: close` can cut the client off before it reads your answer, so leave the connection alone and let Node discard the unread body. A chunked body is only known to be too big after part of it is read: try that path with and without `Connection: close`, several times in a row, and compare what different clients get.

</details>

**Self-checks.** Create `src/http-server/server.check.js` and run `node src/http-server/server.check.js`. It starts your server on a free port (`server.listen(0)`) and sends real requests, each with a 5 s limit, so a handler that never answers fails with a `TimeoutError` instead of hanging. It then starts `server.js` as a child process, sends `SIGTERM` while a request is half-sent, finishes the request, and expects a `201` and a prompt exit `0`. So write `server.js` first (below), then run the checks.

<details>
<summary>The self-check file, server.check.js (open it, copy it, then run it)</summary>

```js
// Module 3 output, deliverable 2 of 2 (bare node:http server): checks for the routes and error paths over
// real HTTP, then a graceful shutdown.
// Run from modules/m03-nodejs-runtime-npm: node src/http-server/server.check.js
// Success: 12 "ok - ..." lines and no error.
import assert from 'node:assert/strict';
import { spawn } from 'node:child_process';
import { once } from 'node:events';
import http from 'node:http';
import net from 'node:net';
import path from 'node:path';
import { setTimeout as sleep } from 'node:timers/promises';
import { createNotesServer } from './notes-server.js';

async function check(name, fn) {
  await fn();
  console.log(`ok - ${name}`);
}

const server = createNotesServer();
server.listen(0); // port 0: let the operating system pick a free port
await once(server, 'listening');
const base = `http://localhost:${server.address().port}`;

// Every request gets a 5 s limit, so a handler that never answers fails the check instead of hanging it.
const fetchTimed = (url, options = {}) => fetch(url, { ...options, signal: AbortSignal.timeout(5000) });

const postJson = (body, headers = {}) =>
  fetchTimed(`${base}/notes`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', ...headers },
    body: typeof body === 'string' ? body : JSON.stringify(body),
  });

try {
  let created;

  await check('POST /notes creates a note: 201, JSON, and a Location header', async () => {
    const response = await postJson({ text: 'first note' });
    assert.equal(response.status, 201);
    assert.match(response.headers.get('content-type'), /^application\/json/);
    created = await response.json();
    assert.match(created.id, /^[0-9a-f-]{36}$/);
    assert.equal(created.text, 'first note');
    assert.equal(response.headers.get('location'), `/notes/${created.id}`);
  });

  await check('GET /notes lists the notes', async () => {
    const response = await fetchTimed(`${base}/notes`);
    assert.equal(response.status, 200);
    assert.deepEqual(await response.json(), [created]);
  });

  await check('GET /notes/:id returns one note, and a trailing slash or a leading // works too', async () => {
    for (const suffix of ['', '/']) {
      const response = await fetchTimed(`${base}/notes/${created.id}${suffix}`);
      assert.equal(response.status, 200);
      assert.deepEqual(await response.json(), created);
    }
    assert.equal((await fetchTimed(`${base}//notes`)).status, 200);
  });

  await check('an unknown id is a 404 with a JSON error, not an empty 200', async () => {
    const response = await fetchTimed(`${base}/notes/nope`);
    assert.equal(response.status, 404);
    assert.deepEqual(await response.json(), { error: { status: 404, message: 'No note with id nope' } });
  });

  await check('an unknown route is a 404 with a JSON error, including odd paths like // that once gave a 500', async () => {
    for (const unknown of ['/nothing', '//', '//x/notes', '//a:b/']) {
      const response = await fetchTimed(`${base}${unknown}`);
      assert.equal(response.status, 404, unknown);
      assert.equal((await response.json()).error.status, 404, unknown);
    }
  });

  await check('a known path with the wrong method is a 405 with an Allow header', async () => {
    const onCollection = await fetchTimed(`${base}/notes`, { method: 'DELETE' });
    assert.equal(onCollection.status, 405);
    assert.equal(onCollection.headers.get('allow'), 'GET, POST');
    const onItem = await fetchTimed(`${base}/notes/${created.id}`, { method: 'POST' });
    assert.equal(onItem.status, 405);
    assert.equal(onItem.headers.get('allow'), 'GET');
  });

  await check('malformed JSON is a 400 and the server keeps serving', async () => {
    const response = await postJson('{"text": oops');
    assert.equal(response.status, 400);
    assert.equal((await response.json()).error.message, 'Body is not valid JSON');
    assert.equal((await fetchTimed(`${base}/notes`)).status, 200);
  });

  await check('a wrong Content-Type is a 415', async () => {
    const response = await postJson('hello', { 'Content-Type': 'text/plain' });
    assert.equal(response.status, 415);
  });

  await check('a body without a usable "text" is a 400, including null, arrays and empty text', async () => {
    for (const body of [{}, { text: 42 }, { text: '   ' }, { text: 'x'.repeat(1001) }, 'null', '[1]', '']) {
      const response = await postJson(body);
      assert.equal(response.status, 400, `body ${JSON.stringify(body)}`);
    }
  });

  await check('a body over 1 MiB is a 413 by its Content-Length', async () => {
    const response = await postJson({ text: 'a'.repeat(2 * 1024 * 1024) });
    assert.equal(response.status, 413);
  });

  await check('a chunked body over 1 MiB is a 413 too, counted as it arrives', async () => {
    const status = await new Promise((resolve, reject) => {
      const request = http.request(`${base}/notes`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Transfer-Encoding': 'chunked' },
      });
      request.on('response', (response) => {
        resolve(response.statusCode);
        response.resume();
      });
      // The server may close the connection while we are still writing, which is fine here.
      request.on('error', (error) => (error.code === 'ECONNRESET' || error.code === 'EPIPE' ? null : reject(error)));
      request.write('{"text":"');
      request.write('a'.repeat(600 * 1024));
      request.write('a'.repeat(600 * 1024));
      request.end('"}');
    });
    assert.equal(status, 413);
  });
} finally {
  server.closeAllConnections();
  server.close();
}

await check('server.js closes cleanly on SIGTERM, even with a request still in flight', async () => {
  const child = spawn(process.execPath, [path.join(import.meta.dirname, 'server.js')], {
    env: { ...process.env, PORT: '0' },
    stdio: ['ignore', 'pipe', 'inherit'],
  });
  const closed = once(child, 'close', { signal: AbortSignal.timeout(15_000) });
  let socket;
  try {
    // Wait for the "listening" line, which also tells us the port. Fail if the server exits or is silent.
    let output = '';
    const port = await new Promise((resolve, reject) => {
      child.stdout.on('data', (chunk) => {
        output += chunk;
        const match = output.match(/listening on http:\/\/localhost:(\d+)/);
        if (match) resolve(Number(match[1]));
      });
      child.once('close', (code) => reject(new Error(`server.js exited with code ${code} before it was listening`)));
      setTimeout(() => reject(new Error('server.js was not listening after 5 s')), 5000).unref();
    });

    // Start a request and leave it half-sent, so it is still in flight when the signal arrives.
    const body = '{"text":"in flight"}';
    let response = '';
    socket = net.connect(port, 'localhost');
    socket.on('data', (chunk) => (response += chunk));
    socket.write(
      `POST /notes HTTP/1.1\r\nHost: localhost\r\nContent-Type: application/json\r\n` +
        `Content-Length: ${Buffer.byteLength(body)}\r\n\r\n${body.slice(0, 8)}`,
    );
    await sleep(200);
    const signalledAt = performance.now();
    child.kill('SIGTERM');
    await sleep(200);
    socket.write(body.slice(8)); // finish the request after the server has been told to stop

    const [code] = await closed;
    assert.equal(code, 0, 'exit code 1 after about 5 s means shutdown timed out waiting on a connection');
    assert.ok(performance.now() - signalledAt < 3000, 'it should exit once the request is answered, not wait for the 5 s timer');
    assert.match(response, /^HTTP\/1\.1 201/, 'the request in flight must still be answered');
    assert.match(output, /SIGTERM received/);
  } finally {
    socket?.destroy();
    child.kill('SIGKILL');
  }
});
```

</details>

**`server.js`:** write it from the requirements above. Reference:

<details>
<summary>Reference implementation of server.js (open it after you have tried)</summary>

```js
// Module 3 output, deliverable 2 of 2 (bare node:http server): starts the notes server.
// Run from modules/m03-nodejs-runtime-npm: npm run serve   (npm run dev restarts it when a file changes)
// Success: prints "notes server listening on http://localhost:3000". Stop it with Ctrl+C.
import { createNotesServer } from './notes-server.js';

// Environment variables are always strings, so PORT has to be converted and checked by hand.
// `||` and not `??`: a line like `PORT=` in .env gives "", and Number("") is 0, which would make
// the operating system pick a random port.
const port = Number(process.env.PORT || 3000);
if (!Number.isInteger(port) || port < 0 || port > 65535) {
  console.error(`PORT must be a whole number from 0 to 65535, got "${process.env.PORT}"`);
  process.exit(2);
}

const server = createNotesServer();

// listen() reports a failure (such as a port already in use) as an 'error' event. With no listener,
// that event is thrown and prints a long stack trace.
server.on('error', (error) => {
  console.error(error.code === 'EADDRINUSE' ? `Port ${port} is already in use` : error.message);
  process.exitCode = 1;
});

server.listen(port, () => {
  // Ask the server for its port: with PORT=0 the operating system picks a free one.
  console.log(`notes server listening on http://localhost:${server.address().port}`);
});

// Shut down cleanly: stop accepting connections, let requests already in flight finish, then exit.
function shutdown(signalName) {
  console.log(`${signalName} received, closing the server`);
  server.close((error) => {
    if (error) console.error(error);
    process.exit(error ? 1 : 0);
  });
  // close() also closes idle keep-alive connections, but a connection that is busy when the signal
  // arrives only goes idle after its response, and an idle one would then hold close() open. So keep
  // closing idle connections until the server has closed.
  setInterval(() => server.closeIdleConnections(), 100).unref();
  // If a request never finishes, do not wait forever.
  setTimeout(() => {
    console.error('shutdown timed out, forcing exit');
    process.exit(1);
  }, 5000).unref();
}
process.once('SIGINT', () => shutdown('SIGINT'));
process.once('SIGTERM', () => shutdown('SIGTERM'));
```

</details>

<details>
<summary>Reference implementation of notes-server.js (open it after you have tried)</summary>

```js
// Module 3 output, deliverable 2 of 2 (bare node:http server): three JSON endpoints and no framework.
// Nothing to run here. server.js starts it, and server.check.js tests it.
// Routing, body parsing, error handling and response helpers are all written by hand:
//   GET  /notes       list every note
//   POST /notes       create a note from a JSON body { "text": "..." }
//   GET  /notes/:id   read one note
import http from 'node:http';
import { randomUUID } from 'node:crypto';

// ---- Helpers: errors, responses and reading the body --------------------------------------------
const MAX_BODY_BYTES = 1024 * 1024;
const MAX_TEXT_LENGTH = 1000;

class HttpError extends Error {
  constructor(status, message, headers = {}) {
    super(message);
    this.status = status;
    this.headers = headers;
  }
}

function sendJson(res, status, body, headers = {}) {
  const payload = JSON.stringify(body);
  res.writeHead(status, {
    'Content-Type': 'application/json; charset=utf-8',
    'Content-Length': Buffer.byteLength(payload),
    ...headers,
  });
  res.end(payload);
}

// Read the request body, which arrives as a stream of chunks, and parse it as JSON.
async function readJsonBody(req) {
  const type = req.headers['content-type'] ?? '';
  if (!type.toLowerCase().startsWith('application/json')) {
    throw new HttpError(415, 'Content-Type must be application/json');
  }
  // Refuse an oversized body from its declared length before reading any of it. (No `Connection:
  // close` here: with it, fetch could fail with EPIPE before it could read the 413.)
  if (Number(req.headers['content-length']) > MAX_BODY_BYTES) {
    throw new HttpError(413, `Body is larger than ${MAX_BODY_BYTES} bytes`);
  }
  // A chunked body has no declared length, so count the bytes as they arrive too. Here `Connection:
  // close` is needed: without it the 413 was unreliable for some clients (fetch timed out in 2 of 5 runs).
  const chunks = [];
  let size = 0;
  for await (const chunk of req) {
    size += chunk.length;
    if (size > MAX_BODY_BYTES) {
      throw new HttpError(413, `Body is larger than ${MAX_BODY_BYTES} bytes`, { Connection: 'close' });
    }
    chunks.push(chunk);
  }
  try {
    return JSON.parse(Buffer.concat(chunks).toString('utf8'));
  } catch {
    throw new HttpError(400, 'Body is not valid JSON');
  }
}

// ---- The server: handlers, routing and the one error handler ------------------------------------
export function createNotesServer() {
  const notes = new Map();

  const listNotes = (req, res) => sendJson(res, 200, [...notes.values()]);

  async function createNote(req, res) {
    const body = await readJsonBody(req);
    if (typeof body?.text !== 'string' || body.text.trim() === '' || body.text.length > MAX_TEXT_LENGTH) {
      throw new HttpError(400, `"text" must be a non-empty string of at most ${MAX_TEXT_LENGTH} characters`);
    }
    const note = { id: randomUUID(), text: body.text, createdAt: new Date().toISOString() };
    notes.set(note.id, note);
    sendJson(res, 201, note, { Location: `/notes/${note.id}` });
  }

  function getNote(req, res, id) {
    const note = notes.get(id);
    if (!note) throw new HttpError(404, `No note with id ${id}`);
    sendJson(res, 200, note);
  }

  // Match a method and path to a handler by hand: split the path into segments and compare them.
  function route(method, segments) {
    if (segments[0] !== 'notes' || segments.length > 2) return null;
    const allowed = segments.length === 1 ? { GET: listNotes, POST: createNote } : { GET: getNote };
    const handler = allowed[method];
    if (!handler) throw new HttpError(405, `${method} is not allowed here`, { Allow: Object.keys(allowed).join(', ') });
    return (req, res) => handler(req, res, segments[1]);
  }

  return http.createServer(async (req, res) => {
    try {
      // Parse the path from a fixed prefix: the Host header is client input and must not decide whether
      // parsing throws. A prefix, not a base: with `new URL(req.url, base)`, a target that starts with //
      // is read as a host, so `GET //` threw and answered 500.
      if (!req.url.startsWith('/')) throw new HttpError(400, 'Request target must start with /');
      let pathname;
      try {
        ({ pathname } = new URL(`http://localhost${req.url}`));
      } catch {
        throw new HttpError(400, 'Bad request target');
      }
      // Dropping empty segments makes /notes, /notes/ and //notes the same route.
      const segments = pathname.split('/').filter(Boolean);
      const handler = route(req.method, segments);
      if (!handler) throw new HttpError(404, `No route for ${pathname}`);
      await handler(req, res);
    } catch (error) {
      // The one place every failure ends up. Without it, a thrown error in this async function
      // becomes an unhandled rejection and takes the whole process down.
      if (res.headersSent) return res.destroy();
      if (error instanceof HttpError) {
        sendJson(res, error.status, { error: { status: error.status, message: error.message } }, error.headers);
      } else {
        console.error(error);
        sendJson(res, 500, { error: { status: 500, message: 'Internal Server Error' } });
      }
    }
  });
}
```

</details>

**Compare with your version.**

- One `try/catch` around the whole handler is the central error handler. Handlers throw an `HttpError` and never write errors themselves.
- Routing is "split the path, compare the pieces". A 405 falls out of knowing a path exists but lacks a method.
- Body size is checked twice (declared `Content-Length`, then counted chunks). Only the chunked refusal sends `Connection: close`.
- Shutdown sweeps idle connections every 100 ms, because a connection busy at the signal goes idle later.
- `notes-server.js` never calls `listen`, so the checks can use port 0. Express apps want the same split.

**Try it by hand** (start it with `npm run serve`; `npm run dev` restarts it when you edit a file):

```bash
curl -s -i -X POST localhost:3000/notes -H 'Content-Type: application/json' -d '{"text": oops'
curl -s localhost:3000/notes
curl -s -i localhost:3000/notes/nope
curl -s -i -X DELETE localhost:3000/notes
curl -s -i -X POST localhost:3000/notes -H 'Content-Type: text/plain' -d 'hello'
```

Expected: command 1 answers `400` and the next request still works, 3 is a JSON `404`, 4 is a `405` with `Allow: GET, POST`, and 5 is a `415`. Then stop the server with Ctrl+C and read its last log line. Also try a second copy on the same port, and `PORT=abc node src/http-server/server.js; echo $?`.

**Done when:**

- [ ] `node src/http-server/server.check.js` prints twelve `ok - ...` lines and no error.
- [ ] `npm run check` prints 27 `ok - ...` lines in total (6, then 9, then 12) and exits with `0`.
- [ ] With `npm run serve` running, `curl` shows all three endpoints responding (`POST /notes` gives `201`, `GET /notes` lists the note, `GET /notes/<that id>` returns it), and the malformed-JSON `POST` gets a `400` JSON error with the server still up.
- [ ] Starting a second server on a busy port, and `PORT=abc`, each print one line and exit (with `1` and
      `2`), not a stack trace.
- [ ] You can explain why `notes-server.js` exports a function and `server.js` is the file that calls
      `listen`.

### Stage 9 — npm, versions and supply-chain hygiene

**Goal:** learn what `package.json` and the lockfile each record, how `install`, `update`, `outdated`, `audit` and `ci` behave, and how to check a package before it runs code on your machine.

Your programs use no packages, so do the experiments in a scratch project with the small, well-known `semver` package.

```bash
mkdir -p ~/m03-scratch/npmlab
cd ~/m03-scratch/npmlab
nvm use 24
npm init -y
```

`npm init -y` writes a `package.json`. Notice `"type": "commonjs"`, `"main": "index.js"` and a `test` script that fails on purpose.

#### Step 1 — Vet a package before you install it

Nothing is installed by these commands. They only read the registry.

```bash
npm view semver version dist-tags license repository.url maintainers --json
npm view semver@latest scripts dependencies dist.unpackedSize dist.fileCount dist.attestations.provenance.predicateType time.modified --json
npm view semver time --json
```

For each field, ask what it tells you:

| Field | Question |
| --- | --- |
| `repository.url`, `maintainers` | Is it the project you think it is, and who can publish? |
| `time` | How old is it, and how often does it release? |
| `scripts` | Any `preinstall`, `install` or `postinstall`? They run at install time. |
| `dependencies`, `dist.unpackedSize`, `dist.fileCount` | How much comes with it? Does the size fit its job? |
| `dist.attestations...` | Is there build provenance? (It helps, but is not proof of safety.) |
| `license` | Can you use it? |

For `semver` the reference run showed: licence ISC, four maintainers, only dev scripts (**no** `preinstall`, `install` or `postinstall`), no dependencies, 101,065 bytes in 53 files, a SLSA provenance attestation, created in 2011, and releases only days apart.

Ask the question that comes before all of these: **do you need the package at all?** Everything in your two programs came from `node:` modules.

#### Step 2 — Install an old version on purpose, then audit it

```bash
npm install semver@7.5.0
npm pkg get dependencies
npm ls
npm ls --all
npm audit; echo "exit=$?"
npm outdated; echo "exit=$?"
```

What you will see:

- `package.json` records `"semver": "^7.5.0"` (`npm install` adds a caret by default).
- It added **3** packages and reported a vulnerability at once.
- `npm ls` shows the top level only. `npm ls --all` shows packages you did not ask for: `semver` needs `lru-cache`, which needs `yallist`.
- In `package-lock.json`, `node_modules/semver` and `node_modules/lru-cache` each record an exact `version`, a `resolved` URL and an `integrity` hash.
- `npm audit` names the vulnerable range (`semver 7.0.0 - 7.5.1`, severity high, a regular-expression denial of service).
- `npm outdated` shows **Current** (installed from the lockfile), **Wanted** (newest your range allows) and **Latest** (newest published).
- `audit` and `outdated` both exit `1` when they find something, so a CI job can fail on them.

#### Step 3 — Ranges

Create `ranges.cjs` (the `.cjs` extension makes it CommonJS, so `require` works) and run `node ranges.cjs`:

```js
const semver = require('semver');
const versions = ['0.0.3', '0.0.4', '0.2.3', '0.2.9', '0.3.0', '7.5.0', '7.5.9', '7.6.0', '7.8.5', '8.0.0'];
for (const range of ['^7.5.0', '~7.5.0', '^0.2.3', '~0.2.3', '^0.0.3']) {
  console.log(range.padEnd(8), 'matches:', versions.filter((v) => semver.satisfies(v, range)).join(' '));
}
```

Predict each line first. The reference run printed:

```text
^7.5.0   matches: 7.5.0 7.5.9 7.6.0 7.8.5
~7.5.0   matches: 7.5.0 7.5.9
^0.2.3   matches: 0.2.3 0.2.9
~0.2.3   matches: 0.2.3 0.2.9
^0.0.3   matches: 0.0.3
```

#### Step 4 — `update`, and the lockfile diff

```bash
cp package-lock.json lock.before.json
npm update
npm pkg get dependencies
diff <(grep -E '"(version|resolved|integrity)"' lock.before.json) <(grep -E '"(version|resolved|integrity)"' package-lock.json)
npm audit
```

- `npm update` kept `package.json` at `^7.5.0` but moved the lockfile to the newest version that range allows (7.8.5 in the reference run), and removed two packages it no longer needs.
- The `diff` is a lockfile review in miniature: every changed line is a `version`, a `resolved` URL or an `integrity` hash.
- `npm audit` then reports no vulnerabilities.

#### Step 5 — `ci`, development dependencies, `npx` and `engines`

```bash
npm pkg set dependencies.semver="^6.0.0"
rm -rf node_modules
npm ci; echo "exit=$?"
ls node_modules
```

The lockfile holds 7.x but `package.json` now asks for `^6.0.0`, so `npm ci` refuses (`npm ci can only install packages when your package.json and package-lock.json ... are in sync`, naming `lock file's semver@7.8.5 does not satisfy semver@6.3.1`; your version numbers will differ). It exits `1` and installs nothing. Put it back, then try development dependencies and `npx`:

```bash
npm pkg set dependencies.semver="^7.5.0"
npm ci
npx semver 7.5.0 -r "^7"
npm uninstall semver
npm install -D semver
npm pkg get dependencies devDependencies
rm -rf node_modules
npm ci --omit=dev
ls node_modules
```

- `npm ci` installs exactly what the lockfile says, in well under a second here. *(Background:)* it never edits the lockfile, while `npm install` may.
- `npx semver ...` ran the `semver` program from your local `node_modules` and printed `7.5.0`. *(Background:)* for a package you have **not** installed, `npx` downloads it and runs its code, so be careful.
- After `npm install -D` the package sits under `devDependencies`. `npm ci --omit=dev` (a production install) then installs nothing, so `node_modules` does not exist. Development dependencies are for tools: test runners, linters, type checkers.

Last, `engines`:

```bash
mkdir -p ~/m03-scratch/eng && cd ~/m03-scratch/eng
echo '{"name":"eng","version":"1.0.0","engines":{"node":">=99"}}' > package.json
npm install; echo "exit=$?"
npm install --engine-strict; echo "exit=$?"
```

An unmet `engines` range is only an `EBADENGINE` warning (exit `0`) until you pass `--engine-strict`, which
turns it into a failure (exit `1`).

#### Step 6 — Install scripts: code that runs as you

Make a package that looks harmless but has a `postinstall` script, and a project that depends on it. Create
`~/m03-scratch/scripts-lab/harmless-looking/package.json`:

```json
{ "name": "harmless-looking", "version": "1.0.0", "scripts": { "postinstall": "node -e \"require('fs').writeFileSync(process.env.INIT_CWD + '/PROOF.txt', 'postinstall ran as ' + require('os').userInfo().username + ' in ' + process.cwd())\"" } }
```

```bash
cd ~/m03-scratch/scripts-lab
mkdir app && cd app && npm init -y
npm install ../harmless-looking
cat PROOF.txt
```

You will see an `npm warn install-scripts` message about `harmless-looking@1.0.0`, and **the script ran
anyway**: `PROOF.txt` names your own user account. Now try the controls, clearing the project between tries:

```bash
rm -rf node_modules package-lock.json PROOF.txt && npm pkg delete dependencies
npm install --ignore-scripts ../harmless-looking; ls PROOF.txt
rm -rf node_modules package-lock.json PROOF.txt && npm pkg delete dependencies
npm install --strict-allow-scripts ../harmless-looking; echo "exit=$?"; ls PROOF.txt
npm config ls -l | grep -iE "script"
```

- `--ignore-scripts` installs without running the script, so `PROOF.txt` is never written.
- `--strict-allow-scripts` makes npm refuse (`ESTRICTALLOWSCRIPTS`, exit `1`) because the install script is not approved.
- By default, `ignore-scripts` and `strict-allow-scripts` are both `false` (see the `npm config` listing).
- npm 11 also has `npm install-scripts ls` and `approve` to manage approvals. This lab does not go further.

#### Step 7 — Apply it to your own stage folder

```bash
cd ~/<repo-name>/modules/m03-nodejs-runtime-npm
npm install
npm audit; echo "exit=$?"
rm -rf node_modules
npm ci
```

- Your project has no dependencies, so `npm install` creates a `package-lock.json` with only the root entry (and the `engines` range). `npm audit` finds `0 vulnerabilities`, and `npm ci` works.
- **Do not skip this step:** Stage 12 commits that lockfile.
- When you review a lockfile diff in a pull request, look for: packages you did not ask for, a `resolved` URL that is not the registry you expect, an `integrity` hash that changed without a version change, and a large jump in package count.

**Done when:**

- [ ] You ran Steps 1 to 6 in `~/m03-scratch` and can show the output of `npm audit`, `npm outdated` and
      the failing `npm ci`.
- [ ] `ls package-lock.json` in your stage folder succeeds, and `npm audit` there exits with `0`.
- [ ] You can explain what `package.json` records and what `package-lock.json` records, and why `npm ci`
      refused when they disagreed.
- [ ] You can explain what `^7.5.0`, `~7.5.0` and `^0.2.3` each allow, and why a caret on a `0.x` version
      is narrower.
- [ ] You can list what you would check on a package page before installing it, and explain what an install script can do and what `--ignore-scripts` changed in Step 6.

### Stage 10 — Debugging: `console`, stack traces and `--inspect`

**Goal:** know the tools for finding out what a program is doing, and how to read an error.

**Stack traces.** Create `~/m03-scratch/trace.mjs`:

```js
function c() { throw new Error('boom') }
function b() { c() }
async function a() { await Promise.resolve(); b() }
a().catch((e) => {
  console.log(e.stack);
  console.log('cause demo:', new Error('outer', { cause: e }).cause.message);
});
```

Run it and read the stack:

- The first `at` line is where the error was thrown (`c`). Each line below is its caller (`b`, then `a`), with a `file:///...:line:column` location (ES modules show file URLs).
- The lines stop at `a`: after an `await`, the stack only knows the function that resumed.
- `cause` keeps the original error when you wrap it in a new one.

**`console`.** Try each of these in `~/m03-scratch/console-demo.mjs`, one at a time:

```js
console.table([{ a: 1, b: 'x' }, { a: 2, b: 'y' }]);
console.error('this goes to stderr');
console.dir({ a: { b: { c: { d: 1 } } } }, { depth: 0 });
console.time('t');
console.timeEnd('t');
console.group('g');
console.log('inside');
console.groupEnd();
console.assert(1 === 2, 'assert prints only when the condition is false');
```

**The inspector.** Run `node --inspect=9333 -e "setTimeout(() => {}, 30000)"` in one terminal. It prints `Debugger listening on ws://127.0.0.1:9333/<id>`. In another terminal, `curl -s localhost:9333/json/list` shows the debugging target, the endpoint a debugger connects to. `node --inspect-brk script.mjs` does the same but pauses before the first line, so you can set breakpoints first.

**A real session in VS Code.** Start your Stage 8 server with the inspector on, from the stage folder:

```bash
node --inspect src/http-server/server.js
```

It prints `Debugger listening on ws://127.0.0.1:9229/<id>`, then its own `listening` line, and `Debugger attached.` once a debugger connects. Create `.vscode/launch.json` in the repo root (the repo's `.gitignore` ignores `.vscode/`) with the attach snippet VS Code ships:

```json
{
  "version": "0.2.0",
  "configurations": [
    { "name": "Attach", "port": 9229, "request": "attach", "skipFiles": ["<node_internals>/**"], "type": "node" }
  ]
}
```

Start **Attach** from the Run and Debug panel, put a breakpoint on the line `const note = notes.get(id);` in
`getNote` (line 78 of the reference `notes-server.js`), and send:

```bash
curl -s -X POST localhost:3000/notes -H 'Content-Type: application/json' -d '{"text":"debug me"}'
curl -s localhost:3000/notes/<the id from the first reply>
```

Check your session against the reference run:

- The `POST` does not pause (it never reaches `getNote`). The `GET` pauses on the breakpoint. Call Stack shows `getNote` first, and Node's own frames are hidden by `skipFiles`.
- In the paused frame, evaluate `id`, `req.method`, `req.url` and `notes.size` in the Debug Console (Shift+Cmd+Y on macOS). `notes.size` is however many notes you created since the server started.
- `note` fails with `ReferenceError: Cannot access 'note' from debugger`, because the line has not run yet. Press **F10** (step over) once, and `note.text` prints `'debug me'`.
- While paused, **the whole server is paused**: the `GET`'s `curl` waits, and so would any other request. Press Continue (F5) and they finish in order. If a request seems to hang, look for a yellow line first.
- Remove the breakpoint (click its red dot) and the next `GET` returns at once. `curl --max-time 5` makes a hang show as a timeout.
- `node --inspect-brk src/http-server/server.js` prints nothing until a debugger attaches, then pauses before line 1 with the reason "Break on start".

What the Debug Console showed in the reference run, at that pause (the first line is the server's own earlier log line):

```text
notes server listening on http://localhost:3000
id
'35b1ed99-4c78-4ca5-8371-6b7bf96ff3a2'
req.url
'/notes/35b1ed99-4c78-4ca5-8371-6b7bf96ff3a2'
notes.size
5
note
Uncaught ReferenceError ReferenceError: Cannot access 'note' from debugger
    at eval (repl:1:1)
```

**Done when:**

- [ ] `node trace.mjs` printed a stack with the frames `c`, `b` and `a`, and you can say which line is
      where the error was thrown and why `a` is the last frame.
- [ ] You ran every `console` method above and can say which stream `console.error` writes to, and why
      that matters when you pipe a program's output.
- [ ] `curl -s localhost:9333/json/list` returned a target while `node --inspect` was running.
- [ ] With the Stage 8 server running under `--inspect` and a debugger attached, a `GET /notes/<id>` paused
      on your breakpoint in `getNote`, and you read `id` and `req.url` in the paused frame.
- [ ] You can explain why `note` cannot be read on that line and can after one step over, and the difference between `--inspect` and `--inspect-brk`.

### Stage 11 — Explain what a framework would remove

**Goal:** say, from your own code, what Express would take off your hands. This is a required part of the definition of done.

Open `notes-server.js` and `server.js`. For each area below, write one or two sentences: what you wrote by hand, what went wrong before you wrote it (Stage 7 shows you), and what you expect a framework to do instead.

- **Routing:** matching a method and a path, pulling `:id` out of the URL, choosing 404 or 405.
- **Body parsing:** how the body arrived, what you checked before trusting it, where the size limit and content-type check went.
- **Error handling:** the single `try/catch`, why the first version crashed on bad JSON, what `HttpError` stands in for.
- **Responses:** the work repeated on every response (status, headers, `Content-Length`, serialising).
- **What the bare server let you get wrong:** the empty `200`, the missing `Location` header, the unbounded body. What would a framework make easy, and what would it still leave to you?

You have not used Express yet, so write expectations, and check each against Express 5 in M05.

**Done when:**

- [ ] You have written the explanation, covering routing, body parsing, error handling and response
      helpers, each tied to a specific piece of your own code.

### Stage 12 — Commit, push, open the pull request and merge it

**Goal:** deliver the stage as M01 and M02 did: a branch, Conventional Commits and a pull request, with nothing pushed straight to `main`. This is the Week 1 Friday gate.

Run every command from your stage folder, with Node 24 active (`nvm use` on its own line).

#### Step 1 — Re-run the deliverables

```bash
npm run check
npm run report -- --stats
npm run serve        # in another terminal, then curl the three endpoints and stop the server
git status -sb
```

**Why:** the PR's test plan should list results you have just seen. `git status -sb` should show the stage branch, the root `.gitignore` and `.env.example` modified, and the stage folder untracked, with **no** `data/`, `report.json` or `.env` (the root `.gitignore` hides them).

#### Step 2 — Commit in one group per concern

Stage files **by name or by folder**, not with `git add .`. `package-lock.json` comes from Stage 9, Step 7: if `ls package-lock.json` fails, run that step first. The two repo-root files are two folders up:

```bash
git add package.json package-lock.json .env.example ../../.gitignore ../../.env.example
git commit -m "chore(m03): add package.json, lockfile, env example and ignore rules"
git add src/log-report
git commit -m "feat(m03): add log generator and streaming log-report CLI"
git add src/http-server
git commit -m "feat(m03): add bare node:http notes server"
```

**Why this grouping:** project files first, because both programs rely on them. Each program shares a commit with its checks, because the checks prove it works.

#### Step 3 — Push the branch

```bash
git push -u origin feat/module-03/runtime-and-npm
```

#### Step 4 — Open the pull request

```bash
gh pr create --repo <user>/<repo-name> --base main --head feat/module-03/runtime-and-npm \
  --title "feat(m03): add log-report CLI and bare node:http server" \
  --body "<your description>"
```

Replace `<user>` and `<repo-name>` with your GitHub username and repo name. The description has three sections:

- **Summary:** why the stage exists and what it adds, in one short paragraph.
- **Changes:** one bullet per notable change (generator, CLI, server, checks).
- **Test plan:** one checked item per command you ran in Step 1, with what it printed (for example "27 `ok -` lines and exit 0", the CLI's line count and peak memory, the three `curl` status codes). Never tick a box for something you did not run.

#### Step 5 — Merge it and look at the history

Merge the PR on GitHub with a **merge commit**, not a squash, so your three commits stay visible. Then:

```bash
git fetch origin
git log --oneline --graph origin/main -8
```

The graph should show a `Merge pull request #<number> from ...` commit on `main` with your three `...(m03): ...` commits under it. GitHub writes that message, so it is not a Conventional Commit. That is expected.

**Done when:**

- [ ] `git log --oneline main..HEAD` on the stage branch shows three commits (`chore(m03)`, `feat(m03)`,
      `feat(m03)`), and `git status -sb` shows nothing left to commit.
- [ ] A pull request from `feat/module-03/runtime-and-npm` into `main` exists on your build repo, and its
      description has Summary, Changes and Test plan sections.
- [ ] The PR is merged, and `git log --oneline --graph origin/main -8` shows its merge commit with your
      three commits under it.

## 4. Concepts you need

Start with the **quick reference** if a term is new. The **essays** below it explain the bigger ideas. Read each essay after the stage named beside it.

How far to trust each statement:

- **Unmarked:** you ran or measured it in a guided step.
- **Marked *(background)*:** standard Node and npm knowledge that the steps do not prove.
- **Marked *(reference build)*:** seen while building this guide, but not a guided step.

Check the marked points against the Node and npm docs before relying on them.

### Quick reference

Read the whole table as *(background)*, except where a row points at something you ran in a stage.

| Concept | In one line | Used in |
| --- | --- | --- |
| **What Node is** *(background)* | V8 runs your JavaScript, libuv gives the event loop and thread pool, core APIs reach the machine. | Stage 2 |
| **Event loop** | The single thread's loop that runs the next ready callback. | Stage 2 |
| **Blocking and non-blocking** | Blocking calls stall the thread. Non-blocking calls return at once and deliver later. | Stage 2 |
| **Thread pool** | libuv helper threads (4 by default) for `fs`, `crypto.pbkdf2` and similar. | Stage 2 |
| **When Node fits** *(background)* | Good when mostly waiting (APIs). Poor for long calculations. | Stage 2 |
| **ES modules and CommonJS** | `import`/`export` against `require`. `"type": "module"` picks the first. | Stages 1, 2 |
| **`import.meta.dirname`, `.filename`** | ES-module replacements for `__dirname` and `__filename`. | Stages 2, 3, 5 |
| **The `node:` prefix** | Marks a built-in, never an npm package. | All stages |
| **`node:fs`, `node:fs/promises`** | Files: streams (`createReadStream`) and promises (`readFile`, `writeFile`, `mkdir`). | Stages 3, 5, 6 |
| **`node:path`** | Builds file paths the same on every OS (`join`, `basename`). | Stages 3, 5 |
| **`node:os`** | Machine facts: `platform()`, `availableParallelism()`, `tmpdir()`. | Stages 2, 5 |
| **`node:events`** | `EventEmitter` and `once()`. An `'error'` event with no listener throws. | Stages 2, 3, 8 |
| **`node:stream`** | Chunked data with backpressure (see the essay). | Stages 3, 5, 6 |
| **`node:http`** | `createServer` gives you `req` and `res` streams, nothing more. | Stages 7, 8 |
| **`node:crypto`** | Hashes, random bytes, `randomUUID()`. | Stages 2, 7 |
| **`node:timers/promises`** | An awaitable `sleep()` that a signal can cancel. | Stages 2, 8 |
| **`node:util`** | `parseArgs` (options) and `promisify` (callbacks to promises). | Stages 2, 3, 5 |
| **`process.argv`** | Command-line arguments: `process.argv.slice(2)`. | Stage 2 |
| **`process.env`** | Environment variables, always strings. | Stages 2, 8 |
| **Exit codes** | 0 is success. Here: 1 failed run, 2 bad usage, 130 and 143 stopped by `SIGINT` and `SIGTERM` (128 plus the signal number *(background)*). | Stages 2, 5, 8 |
| **Signals** | `SIGINT` (Ctrl+C) and `SIGTERM` ask a process to stop. Your handler replaces the default. | Stages 5, 8 |
| **`.env`, `--env-file`** | `KEY=value` lines loaded into `process.env`. Real variables win. Never commit `.env`. | Stages 1, 2, 8 |
| **`--watch`, `--run`** | Restart on change. Run a `package.json` script without npm. | Stages 2, 5, 8 |
| **Globals** | `fetch`, `URL`, `AbortController`, `structuredClone`: no import needed. | Stages 2, 5, 8 |
| **`package.json`** | The manifest: `type`, `engines`, `scripts`, `dependencies`, `devDependencies`. | Stages 1, 9 |
| **Semver ranges** | `^7.5.0` up to, not including, 8.0.0. `~7.5.0` patches only. | Stage 9 |
| **The lockfile** | `package-lock.json`: the exact versions and hashes installed. | Stage 9 |
| **`install`, `ci`, `update`, `outdated`, `audit`** | Add. Install exactly from the lockfile. Update in range. List newer. List vulnerabilities. | Stage 9 |
| **`npx`** | Runs a package's command. *(Background:)* downloads it first if not installed. | Stage 9 |
| **`dependencies`, `devDependencies`, `engines`** | Run. Build and test. Supported Node versions (a warning only). | Stage 9 |
| **Lifecycle scripts** | `postinstall` and friends run on your machine at install time. | Stage 9 |
| **`console`, stack traces** | `table`, `error` (stderr), `dir`, `time`, `group`, `assert`. Traces read from the throw site down. | Stage 10 |
| **`--inspect`** | The endpoint a debugger attaches to. `--inspect-brk` pauses at line 1. | Stage 10 |
| **`node:test`, `node:sqlite`** | Built in. This course uses Jest and Prisma. | Awareness |

### The event loop and the thread pool (after Stage 2)

**In short:** your JavaScript runs on one thread, so anything slow on that thread stalls everything.

- **One thread, many connections.** A server mostly *waits* (socket, disk, timer). Node hands the waiting to the operating system or helper threads, and the **event loop** runs whichever callback is ready next.
- **Blocking stalls everyone.** A long loop, `readFileSync`, a huge `JSON.parse` or a regex that backtracks badly (the Stage 9 `semver` advisory) stops the thread. Your Core 2 timer showed it.
- **The thread pool** runs work the OS cannot do without blocking, such as most `fs` calls and parts of `crypto` *(background)*. It has 4 threads by default, and Core 3 showed pool size changing the `pbkdf2` times.
- **Ordering** *(background)*: each turn runs timers, then I/O, then `setImmediate`, with `process.nextTick` and promise callbacks in between. Core 1 showed `nextTick` and promises swapping places between CommonJS and ES modules, so never depend on their order.

**Mental model:** one cashier (the thread) with helpers in the back room (the pool). One customer who holds the cashier for ten minutes holds up the whole queue.

**Where Node fits** *(background)*: mostly-waiting work (JSON APIs, proxies, real-time connections). Poorly: long calculations per request, unless moved to a worker thread or another process.

### Streams and backpressure (after Stage 6)

**In short:** a stream handles data a chunk at a time, and the fast side must wait for the slow side.

- **Chunks.** *(Background:)* Node has four kinds: *Readable* (file read), *Writable* (file write, HTTP response), *Duplex* (socket) and *Transform* (changes data in flight). An HTTP request is a Readable, which is why your server collected its body from chunks.
- **`highWaterMark`** is the buffer size at which a stream says "enough for now". For a file stream the default is 64 KiB.
- **Backpressure.** `write()` returns `false` when the buffer is full: stop and wait for `'drain'`. Ignore it and nothing fails, but everything piles up in memory and your "streaming" program becomes a load-everything program.

Three ways to respect it (that `pipeline` destroys every stream on failure is *background*):

| Style | How backpressure is handled | Use it when |
| --- | --- | --- |
| `pipeline(a, b, c)` from `node:stream/promises` | for you, between streams, and it destroys all of them if one fails | moving bytes from one stream to another |
| `for await (const chunk of readable)` | by pulling: the next chunk is requested only when the loop body finishes | processing items in your own code, as the CLI does with lines |
| `write()` and `'drain'` by hand | by you | producing data yourself, as the generator does |

- **Why the CLI stayed small:** pulling. While `aggregateLines` was busy, `readline` did not ask the file for more, so the read paused. Stage 6 shows the low memory, not the pausing.
- **Trade-off:** you see the data once, in order, so anything that needs all of it at once (sorting a whole file) needs another plan.
- **Why the whole-file version needs far more** (the bytes, a string, then one string per line, all at once) is reasoning, not something Stage 6 measured.

**Mental model:** a pipe with a tap and a bucket. Backpressure is closing the tap when the bucket is full.

### ES modules and CommonJS (after Stage 2)

**In short:** Node has two module systems, and `"type"` in `package.json` picks the one your `.js` files use.

| | CommonJS | ES modules |
| --- | --- | --- |
| Syntax | `require()`, `module.exports` | `import`, `export` |
| Loading | synchronous, as the program runs | analysed before it runs |
| `__dirname`, `__filename` | yes | no: use `import.meta.dirname` and `import.meta.filename` |
| Top-level `await` | no | yes |
| Import specifier | extension optional | must include the extension (`./aggregate.js`) |

*(Background, apart from what you saw in Core 4.)*

- **Choosing for `.js`:** the nearest `package.json`. `"type": "module"` is ES modules, `"commonjs"` or nothing is CommonJS. `.mjs` is always ES modules and `.cjs` always CommonJS.
- **Node 24 guessed:** `import` syntax in a `.js` file with no `package.json` ran, and `require()` of an ES module worked *(reference build)*. Do not lean on it: set `"type"`. Your M02 files used `.mjs` for the same reason.
- **The `node:` prefix:** `fs` and `node:fs` are the same module, but some built-ins (`node:test`) work only with the prefix, so use it everywhere.

### Semver, ranges and the lockfile (after Stage 9)

**In short:** `package.json` says what you will accept. The lockfile says what you actually got.

**A version is `MAJOR.MINOR.PATCH`** *(background)*: a patch fixes bugs, a minor adds without breaking, a major may break. A **range** is the set of versions you accept:

| Range | Accepts |
| --- | --- |
| `^7.5.0` | 7.5.0 up to, not including, 8.0.0 |
| `~7.5.0` | patch updates only, up to, not including, 7.6.0 |
| `^0.2.3` | 0.2.x only (below 1.0.0 the minor number acts as the major) |
| `^0.0.3` | 0.0.3 only |

Ranges make `npm install` depend on *when* you run it, so the lockfile records the outcome: every package in the tree (including ones you never asked for) with its exact version, source URL and integrity hash.

| File | Records | Edited by |
| --- | --- | --- |
| `package.json` | what you will accept (ranges) | you, and `npm install <pkg>` |
| `package-lock.json` | what you got (exact versions, hashes) | npm, whenever the tree changes |

- **`npm ci` trusts the lockfile alone**, so CI and deployments use it and the lockfile is committed.
- **`devDependencies`** build and test. A production install (`--omit=dev`) skips them.
- **`engines`** only warns on a mismatch unless you ask npm to be strict.

### Supply-chain hygiene (after Stage 9)

**In short:** installing a package runs someone else's code with your permissions, so make it a deliberate choice.

**How it goes wrong** *(background)*: it was malicious from the start (often a "typosquat" of a popular name), a good package is taken over, or a later version turns bad. Code can run when you install (lifecycle scripts, as in Stage 9), import or call it.

**How to make it deliberate:**

1. **Prefer not to add it.** Is there a `node:` module, or is it ten lines you can write?
2. **Vet it, and count what arrived** (Stage 9, Steps 1 and 2).
3. **Review lockfile diffs** like code: new packages, changed sources, hashes that changed without a version change.
4. **Control install scripts** (Stage 9, Step 6). The default only warns.
5. **Use `npm ci` in automation** and `npm audit` as a gate. An audit finds *known* problems only, not a malicious package nobody has reported.

### What a framework takes over (after Stage 8)

**In short:** a framework handles the chores every HTTP server repeats, but not the decisions.

- **It takes over** what you wrote up in Stage 11: routing, reading the request, failing safely and writing the response. Check Express against that list in M05 (the solution design says Express 5 forwards rejected promises in async handlers to the error handler).
- **It gives you a place for** logging, authentication and request ids that apply to many routes. Your bare server had nowhere for that.
- **It does not teach you HTTP:** status codes and headers are still your decisions.
- **It does not decide what valid input is.** Checking `text` is still your job (Module 8 swaps your hand-written check for a validation library).
- **Underneath,** Node's `http` answered `Expect: 100-continue` before your handler could refuse a large body *(reference build)*. A framework sits on the same module, so that is probably still there *(background)*.

## 5. Common pitfalls

| Mistake | What happens | Do this |
| --- | --- | --- |
| `nvm use` in a folder with no `.nvmrc` | You stay on an older Node | `nvm use` on its own line in the repo root, then check `node --version` |
| `npm init -y` | Writes `"type": "commonjs"` | Set `"module"` by hand |
| Testing the CLI on a small sample | A load-everything version passes too | Measure peak memory on the full file |
| `parseArgs` with an unknown flag | It throws a stack trace | `try/catch`, one line, exit `2` |
| Ctrl+C tested only on a working read | A stuck read could not be stopped politely, and `process.exit()` did not end the process | Test a stuck read. On a second signal, remove your handler and re-send the signal |
| `async` handler with no `try/catch` | One throw kills the server | One `try/catch` around the whole handler |
| `res.end(JSON.stringify(undefined))` | An empty `200` | Check "not found" first |
| `body += chunk`, `JSON.parse('null')` | Split characters corrupt. `null.text` throws | Keep `Buffer` chunks, decode once, use `body?.text` |
| `new URL(req.url, base)` | A `//` target is read as a host, and `GET //` answers `500` | Fixed prefix: ``new URL(`http://localhost${req.url}`)`` |
| `Connection: close` on a `413` | Declared length: `fetch` may get `EPIPE` before it reads the answer. Chunked without it: flaky (`fetch` timed out in 2 of 5 runs) | Off for a declared length, on for chunked. Test repeatedly |
| Shutdown tested with no request in flight | It misses a half-sent request: the process waits for the 5 s timer and exits `1` | Send the signal while a request is in flight |
| A check with no timeout | A silent handler hangs the check | Time-limit requests and child processes |
| `Number(process.env.PORT)`, no `listen` error handler | `abc` is `NaN`, `PORT=` is `0` (random), a busy port prints a stack trace | Check by hand, use `\|\|` for the default, add `server.on('error')` |
| `.env` in the wrong folder | It is ignored: the path is relative to the stage folder | Copy `.env.example` to `.env` in the stage folder |
| Signalling `node --run serve` alone | The server keeps running on the port | Ctrl+C (whole group), or `lsof -i :3000` |
| `npm install` of a package with an install script | npm warns, but the script **runs** | `npm view <pkg> scripts` first, then `--ignore-scripts` |
| `npm ci` after hand-editing `package.json` | It refuses | `npm install`, review the diff, commit both |

## 6. Check yourself against the definition of done

Tick each one only when you have just verified it.

- [ ] **Both programs are merged to `main` through a pull request.** `git log --oneline --graph origin/main
      -8` shows the merge commit of your PR with your commits under it, and the PR contains the `log-report`
      CLI and the bare `node:http` server.
- [ ] **The CLI genuinely streams the file rather than loading it fully into memory first.** On the full file
      (about 100 MB, 2,231,981 lines for the generated one), the CLI finishes under `node --max-old-space-size=32`, its peak
      memory is a fraction of the whole-file version's, and you have both figures written down.
- [ ] **You can explain, from the bare `node:http` server you just wrote, what specifically a framework like
      Express would remove pain from.** You can name routing, body parsing, error handling and response
      helpers, and point to the lines of your own code for each.
- [ ] Supporting evidence for the first two: `npm run check` prints 27 `ok - ...` lines and exits with `0`,
      and `npm audit` in the stage folder reports `0 vulnerabilities`.

## 7. Next step

If every box in section 6 is ticked, this stage is done, and so is Week 1.

1. Keep the repo and carry the same workflow forward: a branch named `feat/module-NN/<topic>`,
   Conventional Commits and a pull request. Never push to `main` directly.
2. Move on to **M04 — TypeScript Basics & TypeScript on Node.js** (Week 2 · Days 1-2). It builds on
   this stage and on M02: you port your **M02 async toolkit** to TypeScript, so have that file and its
   checks on hand, and keep Node 24 pinned with `.nvmrc`. Everything from M04 onward is TypeScript, so this
   was the last stage in plain JavaScript.
3. Keep your explanation of what Express would remove pain from. **M05 — Express** (Week 2) is where you hand
   routing, body parsing and error handling to a framework, and your bare `node:http` server is what you
   will compare it with.
