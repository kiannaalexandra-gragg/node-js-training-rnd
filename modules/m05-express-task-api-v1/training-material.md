# M05 — Express: Routing, Middleware & Error Handling: Training Material

**Week 2 · Days 3-5 · 12 hours**

A "module" in this program means a training stage. When this guide says "ES module" it means a JavaScript
module file. The guide's own steps are called "stages" too, so "the stage" means this module and "Stage 4"
means a step of the guided path below.

## Builds on

M04 (TypeScript on Node). The **Task domain types** you modelled there become an actual API here, and the
project setup (`tsconfig.json`, `npm run typecheck`, `tsx`) carries over. This is also where the **Task API**
starts: it carries forward through M07, M08 and M09 as v1, v2, v3 and v4.

## 1. What you'll build and why

You will build a well-structured REST API with **Express 5** in **TypeScript**, with **one consistent error
model**. You finish with **Task API v1**: an in-memory Express REST API at `/api/v1/tasks` with:

- **request-ID and logging middleware**,
- a **central error handler** that turns every error into one fixed JSON shape,
- **hand-written validation**, on purpose, so that Zod in M08 has something to replace,
- a **committed Bruno collection** that tests every endpoint and every error path,
- an `app.ts` that builds and exports the app and a separate `server.ts` that is the only file calling `listen`,
- the whole thing delivered by a **merged pull request**. This is the **Week 2 Friday gate**.

Why this stage matters: every later stage builds on what you write now. M07 swaps the in-memory store for a
database, M08 swaps your hand-written validation for Zod, and M09 tests the app you export from `app.ts`
without opening a port. A clean error model and a clean split between *building* the app and *running* it
are what make those later steps small.

### The contract you must implement

These two things are **fixed for the whole program**. You implement them. You do not redesign them.

**The error envelope.** Every error response, from every endpoint, has exactly this shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [{ "path": "title", "message": "Required" }],
    "requestId": "<from your request-ID middleware>"
  }
}
```

`details` appears **only on validation errors**. `code` is a stable, machine-readable string per error type.
You choose the exact set of codes you need. No stack traces in responses.

**The Task and the API's scope.** `tasks` is the only resource exposed in v1 to v4:

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | Made by the server |
| `title` | string | Required |
| `description` | string | Optional |
| `status` | `"todo"` \| `"in_progress"` \| `"done"` | |

`users` and `tags` exist later in the database schema (M06) but are **not** routes in this program.

### Where the code goes: one `task-api/` folder

M01-M04 each had a folder under `modules/`. **The Task API does not.** M05, M07, M08 and M09 all build the
same project, one version at a time, so it lives in one top-level folder, `task-api/`, in your build repo. Each
version is a pull request: the merge of this stage's branch is v1. M07 will add Prisma to this same folder
instead of copying it. This guide therefore says `task-api/` where M04's said `modules/m04-.../`.

| File | Where | Built in |
| --- | --- | --- |
| `package.json`, `package-lock.json`, `tsconfig.json`, `.env.example` | `task-api/` | Stage 1 |
| `src/config.ts` | `task-api/src/` | Stage 3 |
| `src/errors.ts` | `task-api/src/` | Stage 4 |
| `src/types/task.ts`, `src/store/task-store.ts`, `src/validation/task-validation.ts` | `task-api/src/` | Stage 5 |
| `src/middleware/request-id.ts`, `logger.ts`, `api-key-auth.ts`, `not-found.ts` | `task-api/src/middleware/` | Stage 6 |
| `src/middleware/error-handler.ts` | `task-api/src/middleware/` | Stage 7 |
| `src/handlers/tasks.handlers.ts`, `src/routes/tasks.routes.ts`, `src/routes/index.ts` | `task-api/src/` | Stage 8 |
| `src/app.ts`, `src/server.ts` | `task-api/src/` | Stage 9 |
| `bruno/` (the collection) | `task-api/` | Stage 11 |
| `README.md` (and a `README.md` edit at the repo root) | `task-api/`, repo root | Stage 12 |
| `scratch/` (Stage 2's practice file) | `task-api/`, **not committed** | Stage 2 |

### How to use this guide

| Stage | What it is | How to treat it |
| --- | --- | --- |
| 1 | Branch, folder, project files, install | Core |
| 2 | Express 5 changes, by experiment | Core: run it, read the output |
| 3 | `config.ts` | Core |
| 4 | The `AppError` hierarchy | Core: you write it |
| 5 | Task types, the in-memory store, hand-written validation | Core |
| 6 | Request-ID, logger, API-key and 404 middleware | Core |
| 7 | The central error handler | Core: you write it |
| 8 | Handlers and routes | Core |
| 9 | `app.ts` and `server.ts`, then run it | Core |
| 10 | Prove the middleware order, and prove the 500 path | Core |
| 11 | The Bruno collection | Core |
| 12 | The gate, commit, push, pull request, merge | Core |

- **Guided path** (section 3): what to do, run and check.
- **Concepts** (section 4): a quick reference, then short essays. Read each essay after the stage named beside it.
- **Common pitfalls** (section 5): look here when something misbehaves.
- **Definition of done** (section 6): the final check.

**Stages 4 and 7 are exercises.** Each has *what it must do*, *your turn* (a scaffold that already
type-checks, with hints you open one at a time), and a *reference implementation* to open after you have
tried. The other stages give you the code to type and the reason for it, because there the learning is in
what you observe.

**Stretch, not required:** `express-rate-limit`. Do it only if you finish early. Nothing in the Definition of
done depends on it. The Concepts section has a short note on how it behaves under Express 5, and what to watch for.

## 2. Before you start

- **Your build repo** (M01-M04) on `main`, with `.nvmrc` containing `24`, and a root `.gitignore` that ignores
  `node_modules/` and `.env`.
- **Your M04 `tsconfig.json`** and your Task types. Stage 1 copies the config. Stage 5 gives you a version of the
  Task types to use, so a missing M04 file does not block you.
- **Node 24 active.** Run `nvm use` **on its own line** in the repo and check that `node --version` prints
  `v24.x`. Do this every session. A new terminal can start on an older Node, and `nvm use | tail` in a pipe does
  not switch the version at all.
- **Tools:** internet for npm, plus `git` and `gh` (Stage 12), and `curl`. Bruno is used from the command line
  through the `@usebruno/cli` package you install in Stage 1. The free Bruno desktop app can open the same
  `bruno/` folder, and was not used in the reference build.
- **Two terminals** from Stage 9 on: one runs the server and stays running, the other sends requests.
- **Pinned:** Node.js 24 LTS; Express 5.x (the reference build used 5.3.0); TypeScript 7.0 (7.0.2); `@types/node@24`;
  `tsx` as the dev runner. No Prisma, Zod or Jest yet. The messages quoted here are from those versions, and
  wording can differ in other patch versions.
- **Import extensions, an open decision.** Whether TypeScript projects import `./file.ts` or `./file.js` is still an
  open program decision ("P2"). **This guide uses `.ts` specifiers with `allowImportingTsExtensions`, as M04 did, and
  that is its choice for this stage, not the program's rule.**
- **File headers:** start each file you write with a short comment: `Module 5 output`, what the file is, how to run it (or "Nothing to run here" and who uses it), and
  what success looks like. The reference files below have them, and so does the Bruno collection (in a `docs` block).
- **No removed Express 4 patterns.** A bare `*` path and the old status-argument signatures such as `res.json(obj, status)` are gone in Express 5. Use `/*splat`
  and `res.status(n).json(body)`. Stage 2 shows what happens if you try.
- **No stack traces in any response, and no secrets in any file you commit.** The `.env.example` you write has
  fake values, and your real `.env` is git-ignored.

Commands are for macOS or Linux and have not been tried on Windows (`kill -TERM` and `lsof` in particular do not
exist there). Run commands from `~/<repo-name>/task-api` unless a step says otherwise.

## 3. Guided path

### Stage 1 — Create the branch, the folder and the project files

**Goal:** a `task-api/` folder with the packages installed, a config that type-checks, and a first commit, on a
branch of its own.

1. Switch to Node 24, on its own line, and check it:
   ```bash
   cd ~/<repo-name>
   nvm use
   node --version        # v24.x
   ```
2. Branch and folder:
   ```bash
   git switch main && git pull
   git switch -c feat/module-05/express-task-api-v1
   mkdir task-api && cd task-api
   ```
3. Create `package.json`. The `bruno` script runs from inside the collection folder, because `bru run` only works at
   the root of a collection:

   ```json
   {
     "name": "task-api",
     "version": "1.0.0",
     "private": true,
     "description": "Task API: an Express 5 REST API in TypeScript, built up from M05 (v1) to M09 (v4)",
     "type": "module",
     "engines": {
       "node": ">=24"
     },
     "scripts": {
       "dev": "tsx watch --env-file-if-exists=.env src/server.ts",
       "start": "tsx --env-file-if-exists=.env src/server.ts",
       "typecheck": "tsc --noEmit",
       "bruno": "cd bruno && bru run --env local"
     },
     "dependencies": {
       "cors": "^2.8.6",
       "express": "^5.3.0",
       "helmet": "^8.3.0"
     },
     "devDependencies": {
       "@types/cors": "^2.8.19",
       "@types/express": "^5.0.6",
       "@types/node": "^24.19.2",
       "@usebruno/cli": "^4.2.1",
       "tsx": "^4.23.15",
       "typescript": "^7.0.2"
     }
   }
   ```

   - `"type": "module"` makes `.ts` files ES modules. Set it by hand: `npm init -y` writes `"commonjs"`.
   - `dev` restarts the server when a file changes, `start` runs it once. Both load `.env` if it exists.
   - `typecheck` is the **gate** from M04: `tsc --noEmit` checks every type and writes nothing.
4. Copy M04's `tsconfig.json` unchanged (it is `strict`, `nodenext`, with `allowImportingTsExtensions`,
   `erasableSyntaxOnly`, `verbatimModuleSyntax` and `noUncheckedIndexedAccess`), or write it:

   ```json
   {
     "compilerOptions": {
       "target": "es2024",
       "lib": ["es2024"],
       "types": ["node"],
       "module": "nodenext",
       "moduleResolution": "nodenext",
       "strict": true,
       "noUncheckedIndexedAccess": true,
       "verbatimModuleSyntax": true,
       "erasableSyntaxOnly": true,
       "allowImportingTsExtensions": true,
       "noEmit": true,
       "skipLibCheck": true
     },
     "include": ["src"]
   }
   ```

   Express's types need `"types": ["node"]`, which is already there. Nothing else changes for Express.
5. Install. Check what is current first:
   ```bash
   npm view express dist-tags.latest        # 5.x
   npm install express@5 cors helmet
   npm install -D typescript@7 tsx @types/node@24 @types/express@5 @types/cors @usebruno/cli
   npm ls --depth=0
   ```
   - `express@5` is pinned to the major on purpose: an unversioned `express` could one day resolve to a different major.
   - `@types/express` and `@types/cors` give TypeScript the shapes of those packages, which ship as plain JavaScript.
   - In the reference build `npm ls` showed `express@5.3.0`, `cors@2.8.6`, `helmet@8.3.0`, `typescript@7.0.2`, `tsx@4.23.15`,
     `@types/express@5.0.6`, `@types/node@24.19.2` and `@usebruno/cli@4.2.1`. npm also warned that `esbuild` (a
     dependency of `tsx`) has a `postinstall` script it had not approved. `tsx` ran without it. Read an install-script
     warning, don't approve it blindly.
6. Write `.env.example`. It documents every setting, with **fake** values, and you copy it to a real `.env` that git
   ignores:

   ```bash
   # Module 5 settings for Task API v1: copy this file to .env in this folder.
   # npm run dev and npm start load .env if it exists. Never commit .env. The values below are fake examples.

   # The port the API listens on (optional, default 3000).
   PORT=3000

   # development, production or test (optional, default development).
   NODE_ENV=development

   # The API key every /api/v1 request must send in the X-API-Key header (required, the server will not start without it).
   API_KEY=example-api-key-change-me

   # Origins allowed by CORS, comma-separated (optional, default * which allows any origin).
   CORS_ORIGIN=*
   ```

   ```bash
   cp .env.example .env
   ```
7. Add one line to the **repo root's** `.env.example` (the index of every stage's settings, from M03 and M04):
   ```text
   # M05, task-api/.env.example: PORT (optional, default 3000), NODE_ENV (optional, default development), API_KEY (required, the key /api/v1 requests send), CORS_ORIGIN (optional, default *)
   ```
8. Prove git will ignore the right things, with a real path (a pattern containing a slash only matches from where the
   `.gitignore` is):
   ```bash
   git check-ignore -v .env node_modules/x dist/x test.db
   ```
   Each line should name a rule in the root `.gitignore`. If one prints nothing, add the rule before you commit.
9. Commit:
   ```bash
   cd ..
   git add task-api/package.json task-api/package-lock.json task-api/tsconfig.json task-api/.env.example .env.example
   git commit -m "chore(m05): add task-api package.json, tsconfig, lockfile and env example"
   cd task-api
   ```

**Done when:**

- [ ] `node --version` prints `v24.x`, and `git branch --show-current` prints `feat/module-05/express-task-api-v1`.
- [ ] `npm ls --depth=0` lists `express@5.x`, `typescript@7.x` and `@types/node@24.x`.
- [ ] `git check-ignore -v .env` names a rule, and `git status` shows no `.env` and no `node_modules/`.

### Stage 2 — The Express 5 changes, by experiment

**Goal:** see, with your own eyes, each Express 5 change the stage asks you to learn, so that you recognise it when
it bites later. The official Express 5 migration guide (https://expressjs.com/en/guide/migrating-5.html) is the reference for the full list. Read it after this
stage.

Run `mkdir -p scratch`, then create `scratch/express5.ts`. It is practice only and you will not commit it. It starts its own tiny app on port 3199, makes six
requests to it, and prints what came back:

```ts
import express from 'express';
import type { NextFunction, Request, Response } from 'express';

// 1. A path with a bare `*` (the Express 4 wildcard).
try {
  express().get('*', (_req, res) => res.send('x'));
  console.log('1. get("*"): registered fine');
} catch (e) {
  console.log('1. get("*") threw:', (e as Error).message.split('\n')[0]);
}

const app = express();
// 2. The Express 5 wildcard, named.
app.get('/files/*splat', (req, res) => { res.json({ splat: req.params.splat }); });
// 3. Optional segment with braces.
app.get('/books{/:id}', (req, res) => { res.json({ id: req.params.id ?? null }); });
// 4. No body parser ran: req.body.
app.post('/nobody', (req, res) => { res.json({ bodyType: typeof req.body, isUndefined: req.body === undefined }); });
// 5. Assigning req.query.
app.get('/query', (req, res) => {
  try { (req as unknown as { query: unknown }).query = { a: '1' }; res.json({ assigned: true }); }
  catch (e) { res.json({ assigned: false, error: (e as Error).message }); }
});
// 6. An async handler that rejects, with no try/catch.
app.get('/reject', async () => { await Promise.resolve(); throw new Error('rejected in async handler'); });
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => { res.status(500).json({ reachedErrorHandler: true, message: err.message }); });

const server = app.listen(3199, async () => {
  const get = async (path: string, init?: RequestInit) => { const r = await fetch(`http://localhost:3199${path}`, init); return `${r.status} ${await r.text()}`; };
  console.log('2. /files/a/b/c ->', await get('/files/a/b/c'));
  console.log('3. /books ->', await get('/books'), '| /books/7 ->', await get('/books/7'));
  console.log('4. POST /nobody ->', await get('/nobody', { method: 'POST', headers: { 'content-type': 'application/json' }, body: '{"a":1}' }));
  console.log('5. /query ->', await get('/query'));
  console.log('6. /reject ->', await get('/reject'));
  server.close();
});
```

Run it:

```bash
npx tsx scratch/express5.ts
```

In the reference build it printed:

```text
1. get("*") threw: Missing parameter name at index 1: *; visit https://git.new/pathToRegexpError for info
2. /files/a/b/c -> 200 {"splat":["a","b","c"]}
3. /books -> 200 {"id":null} | /books/7 -> 200 {"id":"7"}
4. POST /nobody -> 200 {"bodyType":"undefined","isUndefined":true}
5. /query -> 200 {"assigned":false,"error":"Cannot set property query of #<IncomingMessage> which has only a getter"}
6. /reject -> 500 {"reachedErrorHandler":true,"message":"rejected in async handler"}
```

Read each line against the Express 5 change it shows:

| Line | What it shows | Consequence for the code you write |
| --- | --- | --- |
| 1 | A bare `*` path no longer exists. The error talks about a *parameter name*, because `*` now has to be named. | Never write `app.get('*')`. Use `/*splat`, or `app.use(handler)` with no path for "everything". |
| 2 | `/*splat` matches any depth, and `req.params.splat` is an **array**, not a string. | Join it if you need a path string. |
| 3 | `{/:id}` makes a segment optional. When it is absent, `req.params.id` is `undefined`. (The route printed it as `null` because of the `?? null`.) | Handle `undefined`. |
| 4 | With no body parser, `req.body` is `undefined`, not `{}`. | Never read `req.body.title` without checking `req.body` first. Your validators will start by checking this. |
| 5 | `req.query` is a read-only getter. | Read it and validate it. Do not assign to it or write to it to "clean" it. |
| 6 | A rejected promise from an async handler reached the error handler with **no try/catch and no `next(err)`**. | This is the answer to the Definition of done's last question. Stage 7 and the Concepts section explain it. |

Also in the Express 5 migration guide (read there, not tried here): the signatures that took a status as an extra argument are removed, namely `res.json(obj, status)`,
`res.send(body, status)` and `res.send(status)` (use `res.sendStatus(status)` for the last), as is `app.del()` (use `app.delete()`). Always use `res.status(n).json(body)`.
The guide also says `/*splat` does **not** match the bare root of its prefix (`/files`, in line 2's example), and that `/{*splat}` matches both *(from the guide, not run here)*.

**Done when:**

- [ ] You ran `scratch/express5.ts` and the six lines match the table above (the wording of line 1 may differ).
- [ ] You can say, without looking, what changed for `*`, `req.body`, `req.query` and a rejected async handler.
- [ ] `scratch/` is not in your commit (you have not run `git add` on it).

### Stage 3 — Read the environment once, and fail fast (`config.ts`)

**Goal:** one file that reads every environment variable, checks it, and refuses to let the server start if
something is wrong.

*Why:* a server that cannot work should say so when it starts, with every problem listed at once, not fail on the
first request with a confusing error. Values are also checked here **once**, so no other file reads `process.env`.
This is hand-written on purpose: in M08 you replace it with a Zod schema, and you will see what the schema saves
you.

Create `src/config.ts`:

```ts
// Module 5 output: the environment (PORT, NODE_ENV, API_KEY, CORS_ORIGIN), read once at startup.
// Nothing to run here. Imported by app.ts, server.ts and two middleware. Success: a bad value stops the server at startup, with every problem listed.

// Reads the environment once, at startup, and fails fast: a server that cannot start properly
// should say so immediately, not fail on the first request. Hand-written here on purpose; M08
// replaces it with a Zod schema.

const NODE_ENVS = ['development', 'production', 'test'] as const;
type NodeEnv = (typeof NODE_ENVS)[number];

export type Config = {
  port: number;
  nodeEnv: NodeEnv;
  apiKey: string;
  // '*' allows any origin. Otherwise the list of allowed origins.
  corsOrigin: '*' | string[];
};

export function loadConfig(env: NodeJS.ProcessEnv): Config {
  const problems: string[] = [];

  const portText = env.PORT ?? '3000';
  const port = Number(portText);
  if (!Number.isInteger(port) || port < 1 || port > 65535) {
    problems.push(`PORT must be an integer from 1 to 65535, got "${portText}"`);
  }

  const nodeEnvText = env.NODE_ENV ?? 'development';
  const nodeEnv = NODE_ENVS.find((value) => value === nodeEnvText);
  if (nodeEnv === undefined) {
    problems.push(`NODE_ENV must be one of ${NODE_ENVS.join(', ')}, got "${nodeEnvText}"`);
  }

  const apiKey = env.API_KEY ?? '';
  if (apiKey.trim() === '') {
    problems.push('API_KEY is required and must not be empty');
  }

  const corsText = (env.CORS_ORIGIN ?? '*').trim();
  const corsOrigin =
    corsText === '*'
      ? '*'
      : corsText
          .split(',')
          .map((origin) => origin.trim())
          .filter((origin) => origin !== '');

  if (problems.length > 0 || nodeEnv === undefined) {
    throw new Error(`Invalid environment:\n- ${problems.join('\n- ')}`);
  }

  return { port, nodeEnv, apiKey, corsOrigin };
}

export const config = loadConfig(process.env);
```

- `PORT` and `NODE_ENV` have defaults. `API_KEY` has none: a server with no key must not start, or `/api/v1` would be
  open (or locked, depending on how you wrote the check).
- The function collects **every** problem and throws one error that lists them. Reporting one at a time makes you
  restart the server once per mistake.
- `loadConfig(env)` takes the environment as a parameter, so a test in M09 can call it with a made-up one.
  `export const config = loadConfig(process.env)` is what the rest of the app imports.
- Values are `Number(...)`-checked with `Number.isInteger`. Do not use `Number(x)` alone as validation: `Number(' ')` is `0`.
- `CORS_ORIGIN=*` means any origin. Otherwise it is a comma-separated allow-list.

Check it, including the failure. The first command should print nothing and exit 0, the second should fail with three
bullets:

```bash
npm run typecheck && echo "typecheck exit 0"
env -u API_KEY PORT=abc NODE_ENV=staging npx tsx -e "import('./src/config.ts')" 2>&1 | head -8
```

In the reference build the second one printed a stack frame pointing at `config.ts` and then:

```text
Error: Invalid environment:
- PORT must be an integer from 1 to 65535, got "abc"
- NODE_ENV must be one of development, production, test, got "staging"
- API_KEY is required and must not be empty
```

**Done when:**

- [ ] `npm run typecheck` exits 0.
- [ ] Loading the config with a bad `PORT`, a bad `NODE_ENV` and no `API_KEY` fails once, with all three problems listed.

### Stage 4 — The `AppError` hierarchy (`errors.ts`)

**Goal:** a base `AppError` and small subclasses, so that an error carries its own HTTP status and its own
`code`.

*Why:* the central error handler (Stage 7) must not guess what kind of error it received. If a `NotFoundError` knows it
is a 404 with the code `NOT_FOUND`, the handler only has to read those two properties.

**What it must do**

- `AppError extends Error`, with `statusCode`, `code`, `message` and an optional `details` list of `{ path, message }`.
  Its `name` is the subclass's name.
- Subclasses, each passing its own status and code to `super`, and each with a sensible default message:

  | Class | Status | `code` | Takes |
  | --- | --- | --- | --- |
  | `BadRequestError` | 400 | `BAD_REQUEST` | a message |
  | `ValidationError` | 400 | `VALIDATION_ERROR` | the `details` list, and a message |
  | `UnauthorizedError` | 401 | `UNAUTHORIZED` | a message |
  | `NotFoundError` | 404 | `NOT_FOUND` | a message |
  | `ConflictError` | 409 | `CONFLICT` | a message |

- **`ValidationError` is the only one with `details`.**
- **No parameter properties.** `constructor(public statusCode: number)` is rejected by `erasableSyntaxOnly`. Declare the field,
  assign it in the constructor.
- `ConflictError` is part of the hierarchy, but **no v1 endpoint throws it**: the contract defines no rule that conflicts, and
  you should not invent one.

**Your turn.** Create `src/errors.ts` from this scaffold. It type-checks as it stands. Replace each `TODO`:

```ts
// Module 5 output: the AppError hierarchy.
// Nothing to run here. Handlers and validation throw these, and middleware/error-handler.ts turns them into the error envelope.

export type ErrorDetail = { path: string; message: string };

// TODO: AppError extends Error. Declare statusCode, code and details as fields (no parameter properties),
// assign them in the constructor, and set `this.name` to the subclass's name.
export class AppError extends Error {
  constructor(message: string) {
    super(message);
  }
}

// TODO: BadRequestError (400, BAD_REQUEST), ValidationError (400, VALIDATION_ERROR, with details),
// UnauthorizedError (401, UNAUTHORIZED), NotFoundError (404, NOT_FOUND), ConflictError (409, CONFLICT).
```

<details>
<summary>Hint 1: the base class</summary>

The base needs `statusCode`, `code`, `message` and optional `details`. Declare each as a `readonly` field above the constructor, call
`super(message)`, then assign. `new.target.name` inside a constructor is the name of the class that was actually constructed, so it gives
each subclass its own `name` without repeating it.

</details>

<details>
<summary>Hint 2: the subclasses</summary>

Each subclass is a constructor with a default message that calls `super(status, code, message)`. `ValidationError` takes `details` first
and the message second (with a default), because a validation error without details makes no sense.

</details>

<details>
<summary>Reference implementation, <code>errors.ts</code> (open it after you have tried)</summary>

```ts
// Module 5 output: the AppError hierarchy.
// Nothing to run here. Handlers and validation throw these, and middleware/error-handler.ts turns them into the error envelope.

// The AppError hierarchy. Every error the API means to send to a client is an AppError: it
// carries its own HTTP status and a stable, machine-readable code, so the central error handler
// never has to guess. Anything that is not an AppError is a programmer error (a bug), and the
// handler turns it into a generic 500.
//
// Written without parameter properties on purpose: erasableSyntaxOnly (from the M04 tsconfig)
// rejects them, because Node's type stripping cannot erase them.

export type ErrorDetail = { path: string; message: string };

export class AppError extends Error {
  readonly statusCode: number;
  readonly code: string;
  readonly details: ErrorDetail[] | undefined;

  constructor(statusCode: number, code: string, message: string, details?: ErrorDetail[]) {
    super(message);
    this.name = new.target.name;
    this.statusCode = statusCode;
    this.code = code;
    this.details = details;
  }
}

export class BadRequestError extends AppError {
  constructor(message = 'Bad request') {
    super(400, 'BAD_REQUEST', message);
  }
}

// The only error that carries `details`: one entry per field that failed.
export class ValidationError extends AppError {
  constructor(details: ErrorDetail[], message = 'Request validation failed') {
    super(400, 'VALIDATION_ERROR', message, details);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') {
    super(401, 'UNAUTHORIZED', message);
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(404, 'NOT_FOUND', message);
  }
}

// Defined for the hierarchy. v1 has no rule that conflicts, so no endpoint throws it yet.
export class ConflictError extends AppError {
  constructor(message = 'Request conflicts with the current state') {
    super(409, 'CONFLICT', message);
  }
}
```

</details>

Check it:

```bash
npm run typecheck && echo "typecheck exit 0"
npx tsx -e "import { NotFoundError, ValidationError } from './src/errors.ts'; const e = new NotFoundError('x'); console.log(e.name, e.statusCode, e.code, e instanceof Error); console.log(new ValidationError([{ path: 'title', message: 'Required' }]).details)"
```

In the reference build the last two lines printed `NotFoundError 404 NOT_FOUND true` and `[ { path: 'title', message: 'Required' } ]`.

**Done when:**

- [ ] `npm run typecheck` exits 0, and your file has no `public` or `private` in a constructor parameter list.
- [ ] `new NotFoundError('x')` has `name` `NotFoundError`, `statusCode` 404 and `code` `NOT_FOUND`, and is an `instanceof Error`.
- [ ] Only `ValidationError` accepts `details`.

### Stage 5 — The Task types, the in-memory store and hand-written validation

**Goal:** the three pieces that sit behind the handlers: the Task types, a store that holds tasks in memory, and
validation that turns untrusted input into typed values or a `ValidationError`.

#### The Task types (`src/types/task.ts`)

These are your M04 Task types with one change: **no `createdAt` and no `updatedAt`.** The contract's Task has four fields
(`id`, `title`, `description`, `status`), nothing in v1 uses timestamps, and in M07 the database will own them. Copy
the types into this folder rather than importing them from M04's folder, so the API does not depend on another stage.

```ts
// Module 5 output: the Task types for Task API v1.
// Nothing to run here. Type-check with `npm run typecheck`.

// The Task domain for Task API v1, carried over from M04. The program's API contract fixes the
// fields at id, title, description (optional) and status, so the M04 timestamps are left out until
// M07, where the database owns them.

// One list of allowed values is the source of truth: the type is derived from it, so the runtime
// array (used by validation) and the compile-time union cannot drift apart.
export const TASK_STATUSES = ['todo', 'in_progress', 'done'] as const;
export type TaskStatus = (typeof TASK_STATUSES)[number];

export type Task = {
  readonly id: string;
  title: string;
  description?: string;
  status: TaskStatus;
};

// What a client sends to create a task. The server owns the id, and status defaults to 'todo'.
export type CreateTaskInput = Omit<Task, 'id' | 'status'> & Partial<Pick<Task, 'status'>>;

// What a client sends to change a task: any subset of the creatable fields.
export type UpdateTaskInput = Partial<CreateTaskInput>;

export function isTaskStatus(value: unknown): value is TaskStatus {
  return typeof value === 'string' && (TASK_STATUSES as readonly string[]).includes(value);
}
```


- One list, `TASK_STATUSES`, is the source of truth. The type is *derived* from it, so the array that validation loops over
  and the union the compiler checks cannot disagree.
- `CreateTaskInput` and `UpdateTaskInput` are derived from `Task`, not written out again.

#### The store (`src/store/task-store.ts`)

```ts
// Module 5 output: the in-memory task store.
// Nothing to run here. Used by the handlers. Success: tasks survive between requests and are gone after a restart.
import { randomUUID } from 'node:crypto';
import type { CreateTaskInput, Task, TaskStatus, UpdateTaskInput } from '../types/task.ts';

// The in-memory store: a Map that lives as long as the process. Restarting the server empties it.
//
// Every method is async even though a Map is synchronous. The handlers are written against a
// promise-returning store now, so M07 can swap in Prisma without touching them, and a rejected
// promise really does reach the error handler through Express 5 (no try/catch needed).
const tasks = new Map<string, Task>();

export const taskStore = {
  async list(status?: TaskStatus): Promise<Task[]> {
    const all = [...tasks.values()];
    return (status === undefined ? all : all.filter((task) => task.status === status)).map(
      (task) => ({ ...task }),
    );
  },

  async get(id: string): Promise<Task | undefined> {
    const task = tasks.get(id);
    return task === undefined ? undefined : { ...task };
  },

  async create(input: CreateTaskInput): Promise<Task> {
    const task: Task = {
      id: randomUUID(),
      title: input.title,
      status: input.status ?? 'todo',
    };
    if (input.description !== undefined) task.description = input.description;
    tasks.set(task.id, task);
    return { ...task };
  },

  async update(id: string, patch: UpdateTaskInput): Promise<Task | undefined> {
    const existing = tasks.get(id);
    if (existing === undefined) return undefined;
    const updated: Task = { ...existing, ...patch };
    tasks.set(id, updated);
    return { ...updated };
  },

  async remove(id: string): Promise<boolean> {
    return tasks.delete(id);
  },
};
```

- It is a `Map` that lives as long as the process. **Restarting the server empties it.**
- **Every method is `async` even though a `Map` is synchronous.** The handlers are then already written for a database, so M07
  swaps this file and leaves the handlers alone. It also means a handler truly `await`s something, which is what makes the Express 5
  behaviour from Stage 2 (line 6) real in your code.
- Methods return **copies** (`{ ...task }`), so a caller cannot change a stored task by mutating what it was given.
- `update` merges only the fields the validator kept, so `undefined` never blanks a field.

#### Validation (`src/validation/task-validation.ts`)

This is hand-written **on purpose**. Every check below is code that M08 deletes when Zod arrives. Write it, and notice how much of it
is the same four things repeated: check the type, collect a message, keep going instead of stopping at the first problem, build the
`{ path, message }` list.

```ts
// Module 5 output: hand-written request validation (M08 replaces it with Zod).
// Nothing to run here. Used by the handlers. Success: bad input throws one ValidationError that lists every problem.
import { ValidationError, type ErrorDetail } from '../errors.ts';
import {
  TASK_STATUSES,
  isTaskStatus,
  type CreateTaskInput,
  type TaskStatus,
  type UpdateTaskInput,
} from '../types/task.ts';

// Hand-written validation, on purpose. Every check below is the kind of code M08 deletes when Zod
// arrives. Each function takes `unknown` (nothing from the network is trusted), collects every
// problem it finds instead of stopping at the first, and throws one ValidationError.
//
// Fields that are not part of the contract (an `id` or `foo` in the body) are ignored, not
// rejected: the server owns the id, and extra keys do no harm.

const STATUS_MESSAGE = `Must be one of: ${TASK_STATUSES.join(', ')}`;
// The ids this API hands out are UUIDs, so anything else cannot match a task.
const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

function isPlainObject(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null && !Array.isArray(value);
}

// Express 5 leaves req.body undefined when no body parser ran (no Content-Type: application/json),
// so "no body at all" is the first thing to handle.
function requireObject(body: unknown): Record<string, unknown> {
  if (!isPlainObject(body)) {
    throw new ValidationError([{ path: 'body', message: 'Must be a JSON object' }]);
  }
  return body;
}

function checkTitle(value: unknown, required: boolean, problems: ErrorDetail[]): string | undefined {
  if (value === undefined) {
    if (required) problems.push({ path: 'title', message: 'Required' });
    return undefined;
  }
  if (typeof value !== 'string') {
    problems.push({ path: 'title', message: 'Must be a string' });
    return undefined;
  }
  if (value.trim() === '') {
    problems.push({ path: 'title', message: 'Must not be empty' });
    return undefined;
  }
  return value.trim();
}

function checkDescription(value: unknown, problems: ErrorDetail[]): string | undefined {
  if (value === undefined) return undefined;
  if (typeof value !== 'string') {
    problems.push({ path: 'description', message: 'Must be a string' });
    return undefined;
  }
  return value;
}

function checkStatus(value: unknown, problems: ErrorDetail[]): TaskStatus | undefined {
  if (value === undefined) return undefined;
  if (!isTaskStatus(value)) {
    problems.push({ path: 'status', message: STATUS_MESSAGE });
    return undefined;
  }
  return value;
}

export function parseCreateTask(body: unknown): CreateTaskInput {
  const input = requireObject(body);
  const problems: ErrorDetail[] = [];

  const title = checkTitle(input.title, true, problems);
  const description = checkDescription(input.description, problems);
  const status = checkStatus(input.status, problems);

  if (problems.length > 0 || title === undefined) throw new ValidationError(problems);

  const task: CreateTaskInput = { title };
  if (description !== undefined) task.description = description;
  if (status !== undefined) task.status = status;
  return task;
}

export function parseUpdateTask(body: unknown): UpdateTaskInput {
  const input = requireObject(body);
  const problems: ErrorDetail[] = [];

  const title = checkTitle(input.title, false, problems);
  const description = checkDescription(input.description, problems);
  const status = checkStatus(input.status, problems);

  // A PATCH that changes nothing is a mistake, not a no-op.
  const sent = ['title', 'description', 'status'].some((key) => input[key] !== undefined);
  if (!sent) {
    problems.push({ path: 'body', message: 'At least one of title, description, status is required' });
  }
  if (problems.length > 0) throw new ValidationError(problems);

  const patch: UpdateTaskInput = {};
  if (title !== undefined) patch.title = title;
  if (description !== undefined) patch.description = description;
  if (status !== undefined) patch.status = status;
  return patch;
}

export function parseTaskId(id: string): string {
  if (!UUID.test(id)) {
    throw new ValidationError([{ path: 'id', message: 'Must be a valid UUID' }]);
  }
  return id;
}

// GET /api/v1/tasks accepts one optional filter, ?status=todo|in_progress|done.
//   - no `status`: every task.
//   - a valid `status`: only the tasks with that status.
//   - anything else (an unknown value, an empty `?status=`, or `status` given twice): a 400
//     VALIDATION_ERROR with path "status". An invalid filter is never ignored, so a typo cannot
//     silently return the unfiltered list.
export function parseListQuery(query: unknown): { status?: TaskStatus } {
  const raw = isPlainObject(query) ? query.status : undefined;
  if (raw === undefined) return {};
  if (!isTaskStatus(raw)) {
    throw new ValidationError([{ path: 'status', message: STATUS_MESSAGE }]);
  }
  return { status: raw };
}
```

Rules to keep, and why:

- **Every function takes `unknown`.** Nothing from the network is trusted, and `unknown` forces you to check before you use it.
- **Collect every problem, then throw one `ValidationError`.** A client fixing a form wants all the errors at once.
- **No body is a validation error, not a crash.** In Express 5 `req.body` is `undefined` when no parser ran (Stage 2, line 4), so
  `requireObject` runs first.
- **Fields outside the contract are ignored**, not rejected. The server owns `id`, so a client sending one breaks nothing.
- `title` is trimmed and must not be empty. There is **no maximum length**: the contract gives none, and inventing one would be a rule the
  program did not ask for.
- A PATCH with none of `title`, `description`, `status` is a 400. A malformed `:id` is a 400 and a well-formed unknown one is a 404.

**The `status` filter, as a contract.** `GET /api/v1/tasks?status=<value>` is the only query parameter in v1:

| Request | Result |
| --- | --- |
| no `status` | every task |
| `?status=todo`, `?status=in_progress` or `?status=done` | only tasks with that status (case-sensitive) |
| `?status=bogus` | 400 `VALIDATION_ERROR`, `details: [{ path: "status", message: "Must be one of: todo, in_progress, done" }]` |
| `?status=` (empty) | the same 400 |
| `?status=todo&status=done` (twice) | the same 400. Express gives an **array**, which is not a valid value. |

An invalid *value* is never ignored. A typo in the value that silently returned the full list would look like it worked. **Other query parameters are
ignored**, like unknown body fields: `?Status=done` and `?state=done` are not `status`, so they return every task. Pagination and sorting are **not** in v1.
They arrive in M07.

Check the validators without a server. The script prints what each call returns or throws:

```bash
npm run typecheck && echo "typecheck exit 0"
npx tsx -e "
import { parseCreateTask, parseUpdateTask, parseListQuery, parseTaskId } from './src/validation/task-validation.ts';
const show = (label, fn) => { try { console.log(label, '->', JSON.stringify(fn())); } catch (e) { console.log(label, '-> ' + e.name, JSON.stringify(e.details)); } };
show('create valid', () => parseCreateTask({ title: '  Buy milk ', id: 'ignored' }));
show('create bad', () => parseCreateTask({ title: '', description: 5, status: 'nope' }));
show('create undefined body', () => parseCreateTask(undefined));
show('update empty', () => parseUpdateTask({}));
show('id bad', () => parseTaskId('not-a-uuid'));
show('filter none', () => parseListQuery({}));
show('filter ok', () => parseListQuery({ status: 'done' }));
show('filter twice', () => parseListQuery({ status: ['todo', 'done'] }));
"
```

In the reference build it printed:

```text
create valid -> {"title":"Buy milk"}
create bad -> ValidationError [{"path":"title","message":"Must not be empty"},{"path":"description","message":"Must be a string"},{"path":"status","message":"Must be one of: todo, in_progress, done"}]
create undefined body -> ValidationError [{"path":"body","message":"Must be a JSON object"}]
update empty -> ValidationError [{"path":"body","message":"At least one of title, description, status is required"}]
id bad -> ValidationError [{"path":"id","message":"Must be a valid UUID"}]
filter none -> {}
filter ok -> {"status":"done"}
filter twice -> ValidationError [{"path":"status","message":"Must be one of: todo, in_progress, done"}]
```

**Done when:**

- [ ] `npm run typecheck` exits 0.
- [ ] The eight lines above match yours. In particular, the bad create reports **three** problems, not one.
- [ ] No file here imports `zod` or any validation library.

### Stage 6 — Request-ID, logger, API-key and 404 middleware

**Goal:** four small middleware, each doing one job, ready to be put in order in Stage 9.

A **middleware** is a function `(req, res, next)`. Express runs them in the order they are registered. Each one either answers the request
(ending the chain), calls `next()` to pass it on, or calls `next(err)` to jump straight to the error handler. Stage 9 and the Concepts
section explain why the order matters. For now, write each one.

#### Request ID (`src/middleware/request-id.ts`)

```ts
// Module 5 output: request-ID middleware.
// Nothing to run here. Registered first in app.ts. Success: every response, error or not, carries an X-Request-Id header.
import { randomUUID } from 'node:crypto';
import type { NextFunction, Request, Response } from 'express';

// Gives every request an id, so one request can be followed through the logs and matched to the
// error a client received. It must run first: the logger and the error handler both read the id.
//
// An id sent by a caller is reused only when it is short and made of safe characters. Anything
// else is replaced, so a caller cannot push arbitrary text into our logs and headers.
const SAFE_ID = /^[A-Za-z0-9._-]{1,128}$/;

export function requestId(req: Request, res: Response, next: NextFunction): void {
  const incoming = req.get('x-request-id');
  const id = incoming !== undefined && SAFE_ID.test(incoming) ? incoming : randomUUID();
  res.locals.requestId = id;
  res.setHeader('X-Request-Id', id);
  next();
}

// res.locals is where Express keeps per-request values for later middleware. It is typed loosely,
// so this one function is the only place that reads the id back and gives it a string type.
export function requestIdOf(res: Response): string {
  const id: unknown = res.locals.requestId;
  return typeof id === 'string' ? id : 'unknown';
}
```

- It gives every request an id, stores it in `res.locals` (Express's place for per-request values), and sends it back in the
  `X-Request-Id` header. That `requestId` is what the error envelope contains, so a client can quote it and you can find the matching log line.
- An id from the caller is reused **only if it is short and made of safe characters**. Anything else is replaced, so a caller cannot put
  arbitrary text into your logs and headers.
- `requestIdOf(res)` is the one place that reads the id back, and gives it a `string` type. `res.locals` is loosely typed.

#### Logger (`src/middleware/logger.ts`)

```ts
// Module 5 output: request logging middleware.
// Nothing to run here. Registered in app.ts. Success: the server prints one line per request.
import type { NextFunction, Request, Response } from 'express';
import { config } from '../config.ts';
import { requestIdOf } from './request-id.ts';

// One line per request: id, method, URL, status and how long it took. It logs on the response's
// 'finish' event, because the status code does not exist yet when the middleware first runs.
export function requestLogger(req: Request, res: Response, next: NextFunction): void {
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    if (config.nodeEnv === 'test') return;
    const ms = Number(process.hrtime.bigint() - start) / 1_000_000;
    console.log(
      `${requestIdOf(res)} ${req.method} ${req.originalUrl} ${res.statusCode} ${ms.toFixed(1)}ms`,
    );
  });
  next();
}
```

- It logs on the response's `finish` event, because the status code does not exist yet when the middleware first runs. This also means it
  reads the id **after** the request ID middleware has run, whatever order the two are registered in.
- It prints nothing when `NODE_ENV` is `test`, so M09's test output stays readable. It writes one line per request:
  `<requestId> GET /api/v1/tasks 200 1.2ms`.

#### API key (`src/middleware/api-key-auth.ts`)

```ts
// Module 5 output: the simple API-key check for /api/v1.
// Nothing to run here. Registered in app.ts. Success: a request without a valid X-API-Key header gets a 401.
import { createHash, timingSafeEqual } from 'node:crypto';
import type { NextFunction, Request, Response } from 'express';
import { config } from '../config.ts';
import { UnauthorizedError } from '../errors.ts';

// A simple API-key check, nothing more: it is not user authentication. The caller sends the key in
// the X-API-Key header.
//
// Both keys are hashed first so timingSafeEqual always compares two buffers of the same length (it
// throws on different lengths), and the comparison time does not leak how much of the key matched.
function sameKey(received: string, expected: string): boolean {
  const a = createHash('sha256').update(received).digest();
  const b = createHash('sha256').update(expected).digest();
  return timingSafeEqual(a, b);
}

export function apiKeyAuth(req: Request, _res: Response, next: NextFunction): void {
  const key = req.get('x-api-key');
  if (key === undefined || !sameKey(key, config.apiKey)) {
    // The same message for a missing and a wrong key, so it does not say which one it was.
    next(new UnauthorizedError('Missing or invalid API key'));
    return;
  }
  next();
}
```

- It is a **simple key check, not user authentication.** There are no users or sessions.
- Missing and wrong keys get the **same** message, so the response does not say which it was.
- Both keys are hashed before `timingSafeEqual`, which throws on inputs of different lengths and compares in constant time. A plain `===`
  can leak, through timing, how many leading characters were right.
- It calls `next(new UnauthorizedError(...))`. It does not write a response itself: the error handler does.

#### Not found (`src/middleware/not-found.ts`)

```ts
// Module 5 output: the 404 middleware for unmatched routes.
// Nothing to run here. Registered after every route in app.ts. Success: an unknown URL gets the NOT_FOUND envelope.
import type { NextFunction, Request, Response } from 'express';
import { NotFoundError } from '../errors.ts';

// Runs only when no route answered. It does not respond itself: it hands a NotFoundError to the
// central error handler, so an unknown URL gets the same envelope as every other error.
export function notFound(req: Request, _res: Response, next: NextFunction): void {
  next(new NotFoundError(`Route not found: ${req.method} ${req.path}`));
}
```

- It runs only if nothing answered. It does not respond itself: it hands a `NotFoundError` to the error handler, so an unknown URL gets the
  same envelope as every other error.

```bash
npm run typecheck && echo "typecheck exit 0"
```

**Done when:**

- [ ] `npm run typecheck` exits 0.
- [ ] None of the four files writes a response body. They only call `next` (the request-ID one also sets a header).
- [ ] You can say why `apiKeyAuth` hashes both keys before comparing them.

### Stage 7 — The central error handler (`error-handler.ts`)

**Goal:** the **one** function that builds an error response. Every error in the app, however it was thrown, ends here and leaves in the
fixed envelope.

*Why one place:* if each endpoint builds its own error JSON, they drift. With one handler there is one shape, one place to fix it, and one
place that decides what a client is allowed to see.

**What it must do**

- Have **four parameters** `(err, req, res, next)`. Express tells an error handler from an ordinary middleware by counting parameters.
  With three it is never given the error.
- Turn anything thrown into an `AppError`:
  - an `AppError` passes through unchanged,
  - a JSON body that does not parse (`express.json` throws an error whose `type` is `entity.parse.failed`) becomes a 400 `BAD_REQUEST`
    with the message `Request body is not valid JSON`,
  - a body over the size limit (`type` `entity.too.large`) becomes a 413 `PAYLOAD_TOO_LARGE` with `Request body is too large`,
  - any **other client error** that Express, its router or body-parser reports with a 4xx `status` (or `statusCode`) keeps that status. Examples:
    a malformed `%` escape in a URL (the router sets `status` 400) and a body whose `Content-Encoding` cannot be decoded. A 400 becomes
    `BAD_REQUEST` with the message `Bad request`, and any other 4xx becomes `CLIENT_ERROR` with `Request could not be processed`. The messages are
    fixed, not the error's own text, which can echo what the client sent,
  - **everything else is a programmer error** and becomes a 500 `INTERNAL_ERROR` with the message `Internal server error` and **nothing
    from the original error in it**.
- For a 500, log the real error server-side, with the request ID. A 500's cause goes to the log and nowhere else.
- Send `{ error: { code, message, details?, requestId } }` with the error's status. `details` is present **only when the error has some**.
  No stack trace, ever.
- If `res.headersSent` is already true, call `next(err)` and stop: the response has started and you cannot send another.

**Your turn.** Create `src/middleware/error-handler.ts` from this scaffold. It type-checks as it stands. Replace each `TODO`:

```ts
// Module 5 output: the central error handler.
// Nothing to run here. Registered last in app.ts. Success: every error response has the fixed envelope, and `npm run bruno` passes.
import type { NextFunction, Request, Response } from 'express';
import { AppError } from '../errors.ts';
import { requestIdOf } from './request-id.ts';

type ErrorBody = {
  error: {
    code: string;
    message: string;
    details?: { path: string; message: string }[];
    requestId: string;
  };
};

// TODO: toAppError(err: unknown): AppError. An AppError passes through, the two body-parser errors are
// recognised by their `type` property, and anything else becomes the generic 500.
function toAppError(err: unknown): AppError {
  throw new Error('TODO');
}

// TODO: the handler. Four parameters. Handle headersSent, convert, log a 500, build the ErrorBody, send it.
export function errorHandler(err: unknown, req: Request, res: Response, next: NextFunction): void {
  throw new Error('TODO');
}
```

<details>
<summary>Hint 1: recognising a body-parser error</summary>

`err` is `unknown`, so you cannot read `err.type`. Narrow first: it must be a non-null object, `'type' in err`, and `typeof err.type ===
'string'`. Put that in a small helper that returns the `type` string or `undefined`, then `switch` on it.

</details>

<details>
<summary>Hint 2: the other client errors</summary>

The router's malformed-`%` error has a numeric `status` of 400 and no `statusCode`, and body-parser's errors have both. Write a small helper that reads
whichever of the two properties exists (checking `'status' in err` first, because `err` is `unknown`) and returns it when it is a number from 400 to 499.
If you also required `expose: true` here, the malformed-`%` URL would still be a 500: the router's error does not set it.

</details>

<details>
<summary>Hint 3: keeping <code>details</code> out when there are none</summary>

`{ a, b, ...(cond && { c }) }` adds `c` only when `cond` is true, because spreading `false` adds nothing. Use it so that `details` is
absent, not `undefined` or `[]`, and the keys stay in the documented order (`code`, `message`, `details`, `requestId`).

</details>

<details>
<summary>Hint 4: what to log, and where</summary>

Only the 500 case needs a log: the other errors are the client's doing and the logger already recorded their status. Log
`${requestId} ${req.method} ${req.originalUrl} unexpected error:` and then the original `err`, so the stack is in the log. Never put `err` in the
response.

</details>

<details>
<summary>Reference implementation, <code>error-handler.ts</code> (open it after you have tried)</summary>

```ts
// Module 5 output: the central error handler.
// Nothing to run here. Registered last in app.ts. Success: every error response has the fixed envelope, and `npm run bruno` passes.
import type { NextFunction, Request, Response } from 'express';
import { AppError } from '../errors.ts';
import { requestIdOf } from './request-id.ts';

// The one place that builds an error response. Express knows this is an error handler because it
// declares four parameters, so `next` must stay in the signature even when it is only used for the
// headers-already-sent case below.
//
// The body is always { error: { code, message, details?, requestId } }. `details` is present only
// when the error has some (validation errors). Nothing here ever puts a stack trace or the text of
// an unexpected error into the response.

type ErrorBody = {
  error: {
    code: string;
    message: string;
    details?: { path: string; message: string }[];
    requestId: string;
  };
};

// body-parser (inside express.json) throws plain errors with a `type` we can recognise.
function bodyParserType(err: unknown): string | undefined {
  if (typeof err === 'object' && err !== null && 'type' in err && typeof err.type === 'string') {
    return err.type;
  }
  return undefined;
}

// Express, its router and body-parser mark an error as the client's fault with a 4xx `status` (the
// router's malformed %-escape error) or `statusCode` as well (body-parser's, from http-errors). A
// bad %-escape in a URL and a body that does not decompress are two examples. Returns that status,
// or undefined for anything else.
function clientErrorStatus(err: unknown): number | undefined {
  if (typeof err !== 'object' || err === null) return undefined;
  const candidates: unknown[] = [];
  if ('status' in err) candidates.push(err.status);
  if ('statusCode' in err) candidates.push(err.statusCode);
  for (const value of candidates) {
    if (typeof value === 'number' && value >= 400 && value < 500) return value;
  }
  return undefined;
}

// Turns anything thrown into an AppError. Known client errors keep their meaning. Everything else
// is a programmer error and becomes a generic 500 whose message says nothing about the cause.
function toAppError(err: unknown): AppError {
  if (err instanceof AppError) return err;
  switch (bodyParserType(err)) {
    case 'entity.parse.failed':
      return new AppError(400, 'BAD_REQUEST', 'Request body is not valid JSON');
    case 'entity.too.large':
      return new AppError(413, 'PAYLOAD_TOO_LARGE', 'Request body is too large');
  }
  // Any other client error keeps its 4xx status. The message is fixed, not the error's own text,
  // which can echo what the client sent.
  const status = clientErrorStatus(err);
  if (status !== undefined) {
    return status === 400
      ? new AppError(400, 'BAD_REQUEST', 'Bad request')
      : new AppError(status, 'CLIENT_ERROR', 'Request could not be processed');
  }
  return new AppError(500, 'INTERNAL_ERROR', 'Internal server error');
}

export function errorHandler(err: unknown, req: Request, res: Response, next: NextFunction): void {
  // Once the response has started we cannot send a new one: let Express close the connection.
  if (res.headersSent) {
    next(err);
    return;
  }

  const appError = toAppError(err);
  const requestId = requestIdOf(res);

  // The full detail of a bug goes to the server log, with the request id, and nowhere else.
  if (appError.statusCode === 500) {
    console.error(`${requestId} ${req.method} ${req.originalUrl} unexpected error:`, err);
  }

  // The spread adds `details` only when there are some, and keeps the keys in the documented order.
  const body: ErrorBody = {
    error: {
      code: appError.code,
      message: appError.message,
      ...(appError.details !== undefined && { details: appError.details }),
      requestId,
    },
  };
  res.status(appError.statusCode).json(body);
}
```


</details>

**Check it before the app exists.** Make a throwaway file that registers only the request-ID middleware and your handler, throws each kind of
error from a route, and prints the responses. Create `scratch/handler-check.ts`:

```ts
import express from 'express';
import { NotFoundError, ValidationError } from '../src/errors.ts';
import { errorHandler } from '../src/middleware/error-handler.ts';
import { requestId } from '../src/middleware/request-id.ts';

const app = express();
app.use(requestId);
app.get('/not-found', () => { throw new NotFoundError('Task 42 not found'); });
app.get('/invalid', () => { throw new ValidationError([{ path: 'title', message: 'Required' }]); });
app.get('/async', async () => { await Promise.resolve(); throw new NotFoundError('thrown after an await'); });
app.get('/bug', () => { throw new Error('boom: secret detail that must not leak'); });
app.use(errorHandler);

const server = app.listen(3198, async () => {
  for (const path of ['/not-found', '/invalid', '/async', '/bug']) {
    const res = await fetch(`http://localhost:3198${path}`);
    console.log(path, res.status, await res.text());
  }
  server.close();
});
```

```bash
npm run typecheck && echo "typecheck exit 0"
npx tsx scratch/handler-check.ts
```

In the reference build the four responses were (request IDs differ every run, and the `/bug` stack trace in your terminal comes from the
`console.error` and is the **server log**, not the response):

```text
/not-found 404 {"error":{"code":"NOT_FOUND","message":"Task 42 not found","requestId":"..."}}
/invalid 400 {"error":{"code":"VALIDATION_ERROR","message":"Request validation failed","details":[{"path":"title","message":"Required"}],"requestId":"..."}}
/async 404 {"error":{"code":"NOT_FOUND","message":"thrown after an await","requestId":"..."}}
/bug 500 {"error":{"code":"INTERNAL_ERROR","message":"Internal server error","requestId":"..."}}
```

Read the third line twice: that route is `async`, throws **after an `await`**, and has no try/catch and no `next(err)`. It still reached your
handler. That is Express 5 doing the work a try/catch wrapper used to do. The fourth line is the important safety result: the text `boom:
secret detail that must not leak` appears in your terminal (the log) and **not** in the response.

Then commit this slice (leave `scratch/` out):

```bash
cd ..
git add task-api/src/config.ts task-api/src/errors.ts task-api/src/middleware
git commit -m "feat(m05): add config, AppError hierarchy, middleware and central error handler"
cd task-api
```

**Done when:**

- [ ] `npm run typecheck` exits 0, and `errorHandler` has four parameters.
- [ ] The four responses match, and `details` appears in the validation response and **only** there.
- [ ] The word `boom` (or whatever you threw) is **not** in the `/bug` response body.
- [ ] You can explain why the `/async` route reached the handler with no try/catch.

### Stage 8 — Handlers and routes

**Goal:** the five endpoints, split in two: *handlers* (what to do) and *routes* (which URL calls which handler).

*Why split:* a handler knows nothing about URLs, so it can be tested, moved or reused. The route file reads like a table of contents of
the API.

#### The handlers (`src/handlers/tasks.handlers.ts`)

```ts
// Module 5 output: the five task handlers (list, create, get, update, delete).
// Nothing to run here. tasks.routes.ts maps URLs to them. Success: `npm run bruno` passes against a running server.
import type { Request, Response } from 'express';
import { NotFoundError } from '../errors.ts';
import { taskStore } from '../store/task-store.ts';
import type { Task } from '../types/task.ts';
import {
  parseCreateTask,
  parseListQuery,
  parseTaskId,
  parseUpdateTask,
} from '../validation/task-validation.ts';

// Handlers hold the logic and know nothing about URLs: tasks.routes.ts decides which URL reaches
// which handler. They are typed Request<Params, ResBody, ReqBody, Query>. The body and the query
// are `unknown` because they come from the network, and the parse functions turn them into
// something typed.
//
// There is no try/catch anywhere in this file. In Express 5 a handler that throws, or returns a
// rejected promise, is passed to the error handler for us.

type IdParams = { id: string };

export async function listTasks(
  req: Request<object, Task[], unknown, { status?: unknown }>,
  res: Response<Task[]>,
): Promise<void> {
  const { status } = parseListQuery(req.query);
  res.status(200).json(await taskStore.list(status));
}

export async function createTask(
  req: Request<object, Task, unknown>,
  res: Response<Task>,
): Promise<void> {
  const input = parseCreateTask(req.body);
  res.status(201).json(await taskStore.create(input));
}

export async function getTask(req: Request<IdParams, Task>, res: Response<Task>): Promise<void> {
  const id = parseTaskId(req.params.id);
  const task = await taskStore.get(id);
  if (task === undefined) throw new NotFoundError(`Task ${id} not found`);
  res.status(200).json(task);
}

export async function updateTask(
  req: Request<IdParams, Task, unknown>,
  res: Response<Task>,
): Promise<void> {
  const id = parseTaskId(req.params.id);
  const patch = parseUpdateTask(req.body);
  const task = await taskStore.update(id, patch);
  if (task === undefined) throw new NotFoundError(`Task ${id} not found`);
  res.status(200).json(task);
}

export async function deleteTask(req: Request<IdParams>, res: Response): Promise<void> {
  const id = parseTaskId(req.params.id);
  const removed = await taskStore.remove(id);
  if (!removed) throw new NotFoundError(`Task ${id} not found`);
  res.sendStatus(204);
}
```

- **Typing:** `Request<Params, ResBody, ReqBody, Query>` and `Response<ResBody>`. The params are `{ id: string }`. The body and the query are
  `unknown` because they come from the network, and the `parse...` functions turn them into something typed. `ResBody` is the success body
  (errors do not go through `res`, they are thrown).
- **There is no try/catch anywhere in this file.** A throw, or a rejected promise, reaches the error handler by itself (Stage 2 and 7).
- **Status codes:** 200 for reads and updates, 201 for create, 204 with `res.sendStatus(204)` for delete (a 204 has no body).
- A `NotFoundError` message includes the id, which the client sent, so it tells them what was not found.
- **The success response is the bare resource**: the task itself, or an array for the list. The contract fixes the *error* envelope and the
  Task fields, not a success wrapper, so nothing here invents a `{ data, meta }`. Pagination, in M07, is where a wrapper would earn its place.

#### The routes (`src/routes/`)

```ts
// Module 5 output: the /api/v1/tasks routes (URLs only, no logic).
// Nothing to run here. Mounted by routes/index.ts.
import express from 'express';
import {
  createTask,
  deleteTask,
  getTask,
  listTasks,
  updateTask,
} from '../handlers/tasks.handlers.ts';

// URLs only. Mounted at /api/v1/tasks by routes/index.ts, so these paths are relative to that.
export const tasksRouter = express.Router();

tasksRouter.get('/', listTasks);
tasksRouter.post('/', createTask);
tasksRouter.get('/:id', getTask);
tasksRouter.patch('/:id', updateTask);
tasksRouter.delete('/:id', deleteTask);
```

```ts
// Module 5 output: the /api/v1 router.
// Nothing to run here. Mounted by app.ts.
import express from 'express';
import { tasksRouter } from './tasks.routes.ts';

// Everything under /api/v1. A later version of the API would get its own router next to this one.
export const v1Router = express.Router();

v1Router.use('/tasks', tasksRouter);
```

- `express.Router()` makes a mini-app that holds routes. These paths are **relative**: `routes/index.ts` mounts `tasksRouter` at `/tasks`, and
  `app.ts` (Stage 9) mounts the v1 router at `/api/v1`, so `/` here is `/api/v1/tasks`.
- A later version of the API would get its own router next to `v1Router`. That is what the `/api/v1` prefix is for: the version is part of the URL,
  so a v2 can exist beside it.

```bash
npm run typecheck && echo "typecheck exit 0"
```

**Done when:**

- [ ] `npm run typecheck` exits 0.
- [ ] No handler has a `try` statement, a `catch` clause or a `res.status(4xx)` call. Every failure is a `throw`.
- [ ] Route files contain URLs and handler names only, and handler files contain no URLs.

### Stage 9 — `app.ts` and `server.ts`, then run it

**Goal:** the app, built and exported in one file, and the server that runs it in another. Then the first real requests.

*Why two files:* `app.ts` builds the app and **never calls `listen`**. `server.ts` is the **only** file that does. Importing `app.ts`
therefore starts nothing and opens no port. In M09 a test will import the app and call it directly with Supertest, with no port, no
clash between test files, and nothing to shut down *(background: M09 does this, this stage does not)*.

**What `app.ts` must do**

- Create and **export** the app. No `listen`.
- Register these, **in an order you can defend**: your request-ID middleware, your logger, `helmet()`, `cors(...)`, a `GET /health` route that needs
  no key, then one `app.use('/api/v1', ...)` that runs your API-key middleware, then `express.json()`, then the v1 router, and finally your 404
  middleware and your error handler. The body parser is inside the `/api/v1` line, after the key check, so a caller without a key is refused
  before the server reads their body.
- `cors` takes its origin from `config.corsOrigin`, and exposes the `X-Request-Id` header so a browser can read it.

**Before you open the reference, write down the order** you would register them in, and one sentence for each neighbour pair about what
breaks if they swap. Stage 10 tests your answers.

<details>
<summary>Hint: four rules that fix almost the whole order</summary>

1. A middleware only affects what comes **after** it. Anything that reads `req.body` must come after `express.json()`.
2. Express gives an error only to error handlers registered **after** the place it was thrown. So the error handler is last, and the 404
   middleware (which throws a `NotFoundError`) comes just before it, after every route.
3. Anything that reads the request ID must come after the middleware that sets it. Anything that should cover *every* request, even one a
   later middleware rejects, should come early.
4. Refuse cheaply before working expensively: a missing key should be rejected before the server buffers and parses a request body.

</details>

<details>
<summary>Reference implementation, <code>app.ts</code> (open it after you have written your order)</summary>

```ts
// Module 5 output: builds and exports the Express app (middleware, routes, 404 and error handler). It never calls listen.
// Nothing to run here. server.ts starts it. Success: `npm run typecheck` exits 0 and, with the server running, GET /health answers {"status":"ok"}.
import cors from 'cors';
import express from 'express';
import helmet from 'helmet';
import { config } from './config.ts';
import { apiKeyAuth } from './middleware/api-key-auth.ts';
import { errorHandler } from './middleware/error-handler.ts';
import { requestLogger } from './middleware/logger.ts';
import { notFound } from './middleware/not-found.ts';
import { requestId } from './middleware/request-id.ts';
import { v1Router } from './routes/index.ts';

// Builds and exports the app. It never calls listen: server.ts does that, so a test can import
// this file and call the app directly without opening a port.
export const app = express();

// Order matters: a request flows through these top to bottom, and the first one that responds ends
// it. Request id and logger come first so that every request, even one rejected by a later step,
// has an id and a log line. The 404 handler comes after every route, and the error handler comes
// last, because Express only passes an error to handlers registered after the place it was thrown.
app.use(requestId);
app.use(requestLogger);
app.use(helmet());
app.use(
  cors({
    origin: config.corsOrigin,
    // Browsers may only read response headers a server exposes.
    exposedHeaders: ['X-Request-Id'],
  }),
);

// Open on purpose: a health check must work without a key.
app.get('/health', (_req, res) => {
  res.status(200).json({ status: 'ok' });
});

// The key is checked once, for everything under /api/v1, and before the body is read: a caller
// without a key gets a 401, and cannot make the server buffer and parse a body. express.json comes
// before the routes, or req.body would be undefined in the handlers.
app.use('/api/v1', apiKeyAuth, express.json(), v1Router);

app.use(notFound);
app.use(errorHandler);
```


</details>

`/health` is **outside** the key check on purpose: a health check has to work without a key. The key is checked once, for everything under
`/api/v1`, and **before** the body is read. Two consequences: an unknown URL under `/api/v1` is a **401 without a key and a 404 with one**, because the
key is checked before routing, and malformed JSON without a key is a **401, not a 400**, because the parser never runs.

**What `server.ts` must do**

- Import the app and call `listen` on `config.port`.
- Check the error that `listen`'s callback receives. In Express 5 a startup failure such as a busy port arrives there instead of being thrown. If
  you ignore it, the server prints "listening" and then exits 0, which looks like a clean run. Log it and exit 1.
- On `SIGTERM` (what a process manager sends to say "stop") and `SIGINT` (Ctrl+C), **shut down gracefully**: stop accepting connections,
  let requests already in flight finish, then exit with code 0.
- If a connection never ends, don't wait forever: force exit 1 after a timeout, with a timer that does not keep the process alive on its own.
- Do it only once, even if the signal arrives twice.

<details>
<summary>Reference implementation, <code>server.ts</code> (open it after you have tried)</summary>

```ts
// Module 5 output: the only file that calls listen, with graceful shutdown on SIGTERM and SIGINT.
// Run from task-api/: npm run dev   (restarts on change; npm start runs it once)
// Success: prints "Task API listening on http://localhost:3000 (development)". Ctrl+C prints "Server closed" and exits 0.
import { app } from './app.ts';
import { config } from './config.ts';

// The only file that calls listen.
const server = app.listen(config.port, (err) => {
  // Express 5 passes a startup failure (for example EADDRINUSE) to this callback instead of
  // throwing. Ignoring it would print "listening" and then exit 0, which looks like a clean run.
  if (err) {
    console.error(`Could not listen on port ${config.port}:`, err);
    process.exit(1);
  }
  console.log(`Task API listening on http://localhost:${config.port} (${config.nodeEnv})`);
});

// Graceful shutdown: stop taking new connections, let requests already in flight finish, then
// exit. A process manager sends SIGTERM when it wants the server to stop. SIGINT is Ctrl+C.
let shuttingDown = false;

function shutdown(signal: string): void {
  if (shuttingDown) return;
  shuttingDown = true;
  console.log(`${signal} received, closing the server`);

  // server.close stops listening and calls back once every open connection has ended.
  server.close((err) => {
    if (err) {
      console.error('Error while closing the server:', err);
      process.exit(1);
    }
    console.log('Server closed');
    process.exit(0);
  });

  // If a connection never ends, do not wait forever. unref() so this timer alone does not keep
  // the process alive.
  setTimeout(() => {
    console.error('Shutdown timed out, forcing exit');
    process.exit(1);
  }, 10_000).unref();
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```


</details>

`server.close(callback)` stops listening and calls back once every open connection has ended, which is what makes the shutdown
graceful. `setTimeout(...).unref()` is the safety net that does not itself keep the process alive.

#### Run it

Use two terminals, both in `task-api/` with Node 24 active. Terminal A:

```bash
npm run typecheck && echo "typecheck exit 0"
npm run dev
```

It should print `Task API listening on http://localhost:3000 (development)`. In terminal B, with the key from your `.env`:

```bash
KEY=example-api-key-change-me
curl -s localhost:3000/health
curl -s -X POST localhost:3000/api/v1/tasks -H "x-api-key: $KEY" -H 'content-type: application/json' -d '{"title":"first","description":"d"}'
curl -s localhost:3000/api/v1/tasks -H "x-api-key: $KEY"
curl -s localhost:3000/api/v1/tasks
curl -s -X POST localhost:3000/api/v1/tasks -H "x-api-key: $KEY" -H 'content-type: application/json' -d '{"title":"","status":"nope","description":5}'
curl -s -X POST localhost:3000/api/v1/tasks -H "x-api-key: $KEY"
curl -s -X POST localhost:3000/api/v1/tasks -H "x-api-key: $KEY" -H 'content-type: application/json' -d '{bad'
curl -s localhost:3000/api/v1/tasks/not-a-uuid -H "x-api-key: $KEY"
curl -s localhost:3000/api/v1/tasks/11111111-1111-4111-8111-111111111111 -H "x-api-key: $KEY"
curl -s "localhost:3000/api/v1/tasks?status=bogus" -H "x-api-key: $KEY"
curl -s localhost:3000/nope
curl -s "localhost:3000/api/v1/tasks/%E0%A4%A" -H "x-api-key: $KEY"
curl -s -X POST localhost:3000/api/v1/tasks -H 'content-type: application/json' -d '{bad'
```

In the reference build the responses were (ids and request IDs differ for you):

```text
{"status":"ok"}
{"id":"2383d480-0f84-4edc-b184-0508d11cce94","title":"first","status":"todo","description":"d"}
[{"id":"2383d480-0f84-4edc-b184-0508d11cce94","title":"first","status":"todo","description":"d"}]
{"error":{"code":"UNAUTHORIZED","message":"Missing or invalid API key","requestId":"63ead1f3-..."}}
{"error":{"code":"VALIDATION_ERROR","message":"Request validation failed","details":[{"path":"title","message":"Must not be empty"},{"path":"description","message":"Must be a string"},{"path":"status","message":"Must be one of: todo, in_progress, done"}],"requestId":"8bdf6079-..."}}
{"error":{"code":"VALIDATION_ERROR","message":"Request validation failed","details":[{"path":"body","message":"Must be a JSON object"}],"requestId":"df7dea68-..."}}
{"error":{"code":"BAD_REQUEST","message":"Request body is not valid JSON","requestId":"4471a4b0-..."}}
{"error":{"code":"VALIDATION_ERROR","message":"Request validation failed","details":[{"path":"id","message":"Must be a valid UUID"}],"requestId":"b1c8c21e-..."}}
{"error":{"code":"NOT_FOUND","message":"Task 11111111-1111-4111-8111-111111111111 not found","requestId":"faa792c9-..."}}
{"error":{"code":"VALIDATION_ERROR","message":"Request validation failed","details":[{"path":"status","message":"Must be one of: todo, in_progress, done"}],"requestId":"5d2e5cf7-..."}}
{"error":{"code":"NOT_FOUND","message":"Route not found: GET /nope","requestId":"2a593499-..."}}
{"error":{"code":"BAD_REQUEST","message":"Bad request","requestId":"a083f8f6-..."}}
{"error":{"code":"UNAUTHORIZED","message":"Missing or invalid API key","requestId":"..."}}
```

The last two matter. A URL with a broken `%` escape is a **400** (the router throws a `URIError` with `status` 400, which your handler keeps), not a 500. And malformed
JSON sent **without** a key is a **401**: the key is checked before the body is parsed, so `{bad` is never read.

Terminal A printed one line per request, `<requestId> <METHOD> <url> <status> <ms>`. Compare a `requestId` from a response with the matching log line.
Add `-i` to a curl to see the `X-Request-Id` header carry the same value.

Check the operations side, in terminal B:

```bash
curl -s -i localhost:3000/health | grep -i "x-content-type\|x-frame\|x-powered-by"
curl -s -i -X OPTIONS localhost:3000/api/v1/tasks -H 'Origin: http://example.test' -H 'Access-Control-Request-Method: POST' -H 'Access-Control-Request-Headers: x-api-key,content-type' | grep -i "^HTTP\|access-control"
curl -s -i localhost:3000/health -H 'X-Request-Id: my-trace-1' | grep -i "^x-request-id"
curl -s -i localhost:3000/health -H 'X-Request-Id: bad id with spaces!' | grep -i "^x-request-id"
```

In the reference build: `helmet` set `X-Content-Type-Options: nosniff` and `X-Frame-Options: SAMEORIGIN`, and `X-Powered-By` was absent. The preflight
answered `204 No Content` with `Access-Control-Allow-Origin: *`, the allowed methods and `Access-Control-Expose-Headers: X-Request-Id`. The
incoming id `my-trace-1` was reused, and the unsafe one was replaced with a new UUID.

Now the shutdown. In terminal B, `kill -TERM $(lsof -ti :3000)`. In terminal A you should see:

```text
SIGTERM received, closing the server
Server closed
```

and the process exits 0. `Ctrl+C` in terminal A does the same with `SIGINT`. (With `npm run dev`, every save you make restarts the server the
same way: the reference build printed `[tsx] change in ./src/app.ts Restarting...`, then the two lines above, then the new `listening` line.)
Check the startup failure too: with the server running in terminal A, start a second copy on the same port in terminal B with `PORT=3000 npm start`. In the
reference build it printed `Could not listen on port 3000: Error: listen EADDRINUSE: address already in use :::3000` and a stack, and exited **1**. Before the
callback checked its argument, the same command printed `Task API listening on ...` and exited 0.

Only the shutdown of an *idle* server was checked in the reference build: a request still in flight when the signal arrives was not tried, because the handlers
answer in about a millisecond.

Commit this slice (leave `scratch/` out):

```bash
cd ..
git add task-api/src/types task-api/src/store task-api/src/validation task-api/src/handlers task-api/src/routes task-api/src/app.ts task-api/src/server.ts
git commit -m "feat(m05): add task store, validation, routes, app and server"
cd task-api
```

**Done when:**

- [ ] `npm run typecheck` exits 0, and **only** `server.ts` calls `listen`.
- [ ] The thirteen curl responses match, and every error body has `code`, `message` and `requestId`, plus `details` on validation errors and only there.
- [ ] `kill -TERM` printed `SIGTERM received...` and `Server closed`, and the server exited. A second server on a busy port exited 1.
- [ ] You wrote your middleware order down before opening the reference, and you can say why each neighbour pair is in that order.

### Stage 10 — Prove the order matters, and prove the 500 path

**Goal:** break your app on purpose, one change at a time, and watch the thing you predicted. A rule you have only read is easy to forget. A
rule you have seen fail is not. You will also prove that an unexpected error never leaks.

Start from a **clean tree**, so every experiment can be undone with one command:

```bash
git status --short          # should list only scratch/ (or nothing)
```

For each experiment below: edit the file, run the server on a spare port in terminal A, send the request from terminal B, read what
happened, stop the server, then restore:

```bash
# terminal A
PORT=3301 npm start
# terminal B
KEY=example-api-key-change-me
# afterwards, terminal A: Ctrl+C. Then:
git restore src/app.ts src/middleware src/handlers      # restores whatever you changed
git status --short
```

| # | Change | Request | What the reference build saw |
| --- | --- | --- | --- |
| 1 | Take `express.json()` out of the `/api/v1` line, so it reads `app.use('/api/v1', apiKeyAuth, v1Router);`, and register it on its own **after** that line: `app.use(express.json());` (just before `app.use(notFound)`) | `curl -s -X POST localhost:3301/api/v1/tasks -H "x-api-key: $KEY" -H 'content-type: application/json' -d '{"title":"valid"}'` | A **400** `VALIDATION_ERROR`, `details: [{"path":"body","message":"Must be a JSON object"}]`, for a perfectly valid body: `req.body` is `undefined` in the handler because the parser had not run yet. |
| 2 | Move `app.use(errorHandler)` to **before** the `/api/v1` line | `curl -s -i localhost:3301/api/v1/tasks/not-a-uuid -H "x-api-key: $KEY"` | `400 Bad Request` with `Content-Type: text/html` and a whole stack trace in the page, from **Express's built-in handler** (the `NODE_ENV` was `development`). Your handler never saw the error. |
| 3 | Move `app.use(notFound)` to **before** `app.get('/health', ...)` | `curl -s localhost:3301/health` and `curl -s localhost:3301/api/v1/tasks -H "x-api-key: $KEY"` | Both answer `NOT_FOUND`: `Route not found: GET /health` and `Route not found: GET /api/v1/tasks`. The 404 middleware answered for routes that exist, because it came first. |
| 4 | Move `app.use(requestId)` to **after** the routes (just before `app.use(notFound)`) | `curl -s localhost:3301/api/v1/tasks/not-a-uuid -H "x-api-key: $KEY"` | The envelope's `requestId` is `"unknown"`, and every log line starts with `unknown`: nothing had set the id when the handler ran. |
| 5 | Swap the first two lines, so the **logger is before `requestId`** | `curl -s localhost:3301/health` | **Nothing broke.** The log line still had the id. The logger reads it on the response's `finish` event, after `requestId` ran. If you predicted a failure here, your prediction was wrong, and the useful thing to take away is *why*. |
| 6 | In `error-handler.ts`, change the signature to three parameters, `(err: unknown, req: Request, res: Response): void`, and add `const next = (_e: unknown) => undefined;` as the first line of the function so the rest still compiles | the same `not-a-uuid` request as 2 | The same **HTML page with a stack trace** as experiment 2. Express counts parameters, and with three it is not an error handler. |

Compare with the order you wrote down in Stage 9. Which predictions did you get wrong, and why?

**The 500 path.** Bruno cannot trigger a 500 without a route that fails on purpose, and you must **not** leave a debug route in the API. So
prove it with a temporary throw and then remove it. In `src/handlers/tasks.handlers.ts`, add this as the **first line** of `listTasks`:

```ts
throw new Error('boom: connection string postgres://admin:hunter2@db/prod');
```

(The connection string is a fake. It stands for the kind of detail an unexpected error can carry.) Run `PORT=3301 npm start`, then:

```bash
curl -s -i localhost:3301/api/v1/tasks -H "x-api-key: $KEY"
```

In the reference build the response was `HTTP/1.1 500 Internal Server Error` with the body

```text
{"error":{"code":"INTERNAL_ERROR","message":"Internal server error","requestId":"8b4cb948-b011-484c-b7c4-3f27279da151"}}
```

and **nothing from the thrown error was in it.** The terminal running the server printed the same request ID, the words `unexpected error:`, the
full message (`boom: connection string ...`) and the stack. That is the design: the log has everything, the client has a generic message and an id to
quote.

**Remove the throw, and prove it is gone:**

```bash
git restore src/handlers/tasks.handlers.ts
git status --short
grep -rn "boom\|hunter2" src && echo "LEFTOVER, remove it" || echo "clean"
```

`git status --short` must list nothing from `src/`, and the last line must print `clean`.

**Done when:**

- [ ] You ran experiments 1 to 6, saw each result, and restored the files after each (`git status --short` shows no change in `src/`).
- [ ] You can say, for each of 1-4, what the rule is (what must come before or after what) in one sentence.
- [ ] You saw a `500` with only `Internal server error` in the response and the detail in the server log, then **removed the throw**:
      the `grep` printed `clean`.

### Stage 11 — The Bruno collection

**Goal:** a committed collection that tests every endpoint and every error path, and that **fails when something is wrong**. A test that
can't fail proves nothing.

**Bruno** is a free API client whose collections are **plain text files** in a folder, so they live in git and diff like code. A
collection is a folder with a `bruno.json`. Each request is a `.bru` file. You installed the command-line runner, `@usebruno/cli`, in Stage 1,
so you run it from the terminal. (The Bruno desktop app can open the same folder, but it was not used in the reference build.)

Create the folders:

```bash
mkdir -p bruno/environments bruno/Tasks bruno/Errors
```

**Collection files**

`bruno/bruno.json` marks the folder as a collection:

```json
{
  "version": "1",
  "name": "Task API v1",
  "type": "collection",
  "ignore": ["node_modules", ".git"]
}
```

`bruno/environments/local.bru` holds the variables. `apiKey` is the fake example key from `.env.example`, so a fresh copy of `.env.example` works with no
editing. `missingId` is a well-formed UUID that no task has:

```bru
vars {
  baseUrl: http://localhost:3000
  apiKey: example-api-key-change-me
  missingId: 11111111-1111-4111-8111-111111111111
}
```

`bruno/collection.bru` applies to **every** request. It sends the key as a header, and its `tests` block runs after every response. **One test here checks
the error envelope on every error response in the collection.** You do not repeat it per request:

```bru
headers {
  x-api-key: {{apiKey}}
}

tests {
  // Runs after every request in the collection. Any error response, from any endpoint, must use
  // the fixed envelope, and its requestId must be the one the server put in the X-Request-Id header.
  if (res.status >= 400) {
    test("error response uses the fixed envelope", function () {
      const error = res.body.error;
      expect(Object.keys(res.body)).to.eql(["error"]);
      // Exactly these keys: no stack trace or other internals can ride along.
      const allowed = error.code === "VALIDATION_ERROR"
        ? ["code", "message", "details", "requestId"]
        : ["code", "message", "requestId"];
      expect(Object.keys(error)).to.eql(allowed);
      expect(error.code).to.be.a("string").and.not.equal("");
      expect(error.message).to.be.a("string").and.not.equal("");
      expect(error.requestId).to.be.a("string").and.equal(res.headers["x-request-id"]);
      if (error.code === "VALIDATION_ERROR") {
        expect(error.details).to.be.an("array").that.is.not.empty;
      } else {
        expect(error).to.not.have.property("details");
      }
    });
  }
}

docs {
  Module 5 output: the Bruno collection for Task API v1.
  Run from task-api/ with the server running: npm run bruno
  Success: Status PASS, every request passed. Tasks/ is the happy path, in order. Errors/ has one request per error path.
}
```

- It checks the *exact set of keys*, not only that some keys exist. A response with an extra `stack` key must fail.
- `error.requestId` must equal the `X-Request-Id` header, so the id in the body is the real one.
- `details` must be present and non-empty on `VALIDATION_ERROR`, and **absent** on every other code.

`bruno/Tasks/folder.bru` and `bruno/Errors/folder.bru` name the folders and fix their order (`seq: 1` and `seq: 2`):

```text
meta {
  name: Tasks
  seq: 1
}
```

**A happy-path request** (`bruno/Tasks/02 Create task.bru`). `assert` is a short list of checks, `tests` is JavaScript with `expect`, and
`vars:post-response` saves a value from the response for the requests after it. Here it keeps the new task's `id` as `taskId`:

```bru
meta {
  name: 02 Create task
  type: http
  seq: 2
}

post {
  url: {{baseUrl}}/api/v1/tasks
  body: json
  auth: none
}

body:json {
  {
    "title": "Write the Bruno collection",
    "description": "One request per endpoint"
  }
}

assert {
  res.status: eq 201
  res.body.title: eq Write the Bruno collection
  res.body.status: eq todo
}

tests {
  test("returns the task with a server-made id and only the contract fields", function () {
    expect(res.body.id).to.be.a("string");
    expect(Object.keys(res.body).sort()).to.eql(["description", "id", "status", "title"]);
  });
}

vars:post-response {
  taskId: res.body.id
}
```

**An error-path request** (`bruno/Errors/03 Create with invalid fields.bru`):

```bru
meta {
  name: 03 Create with invalid fields
  type: http
  seq: 3
}

post {
  url: {{baseUrl}}/api/v1/tasks
  body: json
  auth: none
}

body:json {
  {
    "title": "",
    "description": 5,
    "status": "nope"
  }
}

assert {
  res.status: eq 400
  res.body.error.code: eq VALIDATION_ERROR
}

tests {
  test("lists every failing field, in details", function () {
    expect(res.body.error.message).to.equal("Request validation failed");
    expect(res.body.error.details).to.eql([
      { path: "title", message: "Must not be empty" },
      { path: "description", message: "Must be a string" },
      { path: "status", message: "Must be one of: todo, in_progress, done" },
    ]);
  });
}
```

Three more shapes you will need. A `patch` and a `delete` are the same layout with a different block name. An array response is checked with the `isArray`
assertion, and a variable saved earlier is read back with `bru.getVar`:

```bru
meta {
  name: 03 List tasks
  type: http
  seq: 3
}

get {
  url: {{baseUrl}}/api/v1/tasks
  body: none
  auth: none
}

assert {
  res.status: eq 200
  res.body: isArray
}

tests {
  test("includes the task just created", function () {
    expect(res.body.map((task) => task.id)).to.include(bru.getVar("taskId"));
  });
}
```

```bru
meta {
  name: 06 Update task
  type: http
  seq: 6
}

patch {
  url: {{baseUrl}}/api/v1/tasks/{{taskId}}
  body: json
  auth: none
}

body:json {
  {
    "status": "in_progress"
  }
}

assert {
  res.status: eq 200
  res.body.id: eq {{taskId}}
  res.body.status: eq in_progress
}

tests {
  test("leaves the fields that were not sent unchanged", function () {
    expect(res.body.title).to.equal("Write the Bruno collection");
    expect(res.body.description).to.equal("One request per endpoint");
  });
}
```

```bru
meta {
  name: 07 Delete task
  type: http
  seq: 7
}

delete {
  url: {{baseUrl}}/api/v1/tasks/{{taskId}}
  body: none
  auth: none
}

assert {
  res.status: eq 204
}
```

Three requests need a script, because they need something a plain request cannot say. In a `script:pre-request` block, `req` is the request about to be
sent:

```bru
meta {
  name: 01 Missing API key
  type: http
  seq: 1
}

get {
  url: {{baseUrl}}/api/v1/tasks
  body: none
  auth: none
}

assert {
  res.status: eq 401
  res.body.error.code: eq UNAUTHORIZED
}

script:pre-request {
  req.deleteHeader("x-api-key");
}
```

```bru
meta {
  name: 07 Create with a body that is too large
  type: http
  seq: 7
}

post {
  url: {{baseUrl}}/api/v1/tasks
  body: json
  auth: none
}

body:json {
  {}
}

assert {
  res.status: eq 413
  res.body.error.code: eq PAYLOAD_TOO_LARGE
}

script:pre-request {
  req.setBody({ title: "x".repeat(200 * 1024) });
}
```

Malformed JSON cannot be sent as a `json` body (Bruno would parse it first), so use a `text` body with the JSON content type:

```bru
meta {
  name: 06 Create with malformed JSON
  type: http
  seq: 6
}

post {
  url: {{baseUrl}}/api/v1/tasks
  body: text
  auth: none
}

headers {
  content-type: application/json
}

body:text {
  {bad
}

assert {
  res.status: eq 400
  res.body.error.code: eq BAD_REQUEST
}

tests {
  test("is not a validation error, so it has no details", function () {
    expect(res.body.error.message).to.equal("Request body is not valid JSON");
  });
}
```

**Build the rest the same way.** The `Tasks` folder is the happy path, **in order**, because later requests use the `taskId` that request 02 saved. The
`Errors` folder has one request per error path. Every request uses `auth: none` (the key comes from `collection.bru`), the URL starts with `{{baseUrl}}`, and
every error request asserts the status and `res.body.error.code`.

| Folder, `seq` | Request | Expect |
| --- | --- | --- |
| Tasks 1 | `GET /health` | 200, `res.body.status` is `ok` |
| Tasks 2 | `POST /api/v1/tasks` with a `title` and `description` | 201, `status` is `todo`, keys are exactly `description, id, status, title`, saves `taskId` |
| Tasks 3 | `GET /api/v1/tasks` | 200, an array that includes `taskId` (read it with `bru.getVar("taskId")`) |
| Tasks 4 | `GET /api/v1/tasks?status=todo` | 200, every returned task has status `todo` |
| Tasks 5 | `GET /api/v1/tasks/{{taskId}}` | 200, the same `id` |
| Tasks 6 | `PATCH /api/v1/tasks/{{taskId}}` with `{"status":"in_progress"}` | 200, status changed, `title` and `description` unchanged |
| Tasks 7 | `DELETE /api/v1/tasks/{{taskId}}` | 204 |
| Errors 1 | no key (`req.deleteHeader("x-api-key")`) | 401 `UNAUTHORIZED` |
| Errors 2 | wrong key (`req.setHeader("x-api-key", "not-the-key")`) | 401 `UNAUTHORIZED` |
| Errors 3 | create with `title: ""`, `description: 5`, `status: "nope"` | 400 `VALIDATION_ERROR`, three `details` entries, in that order |
| Errors 4 | create with no `title` | 400 `VALIDATION_ERROR`, `[{path: "title", message: "Required"}]` |
| Errors 5 | create with **no body** (`body: none`) | 400 `VALIDATION_ERROR`, `[{path: "body", message: "Must be a JSON object"}]` |
| Errors 6 | create with malformed JSON | 400 `BAD_REQUEST` |
| Errors 7 | create with a 200 KB title (set in a pre-request script) | 413 `PAYLOAD_TOO_LARGE` |
| Errors 8 | `GET /api/v1/tasks/not-a-uuid` | 400 `VALIDATION_ERROR`, `[{path: "id", ...}]` |
| Errors 9 | `GET /api/v1/tasks/{{missingId}}` | 404 `NOT_FOUND` |
| Errors 10 | `GET /api/v1/tasks/{{taskId}}` (the task deleted above) | 404 `NOT_FOUND` |
| Errors 11 | `PATCH /api/v1/tasks/{{missingId}}` with `{"status":"done"}` | 404 `NOT_FOUND` |
| Errors 12 | `PATCH /api/v1/tasks/{{taskId}}` with `{}` | 400 `VALIDATION_ERROR`, `[{path: "body", message: "At least one of title, description, status is required"}]` |
| Errors 13 | `PATCH /api/v1/tasks/{{missingId}}` with `{"status":"finished"}` | 400 `VALIDATION_ERROR` on `status` (validation runs **before** the lookup) |
| Errors 14 | `DELETE /api/v1/tasks/{{missingId}}` | 404 `NOT_FOUND` |
| Errors 15 | `GET /api/v1/tasks?status=bogus` | 400 `VALIDATION_ERROR` on `status` |
| Errors 16 | `GET /api/v1/tasks?status=` | 400 `VALIDATION_ERROR`, `details` is `[{path: "status", ...}]` |
| Errors 17 | `GET /api/v1/tasks?status=todo&status=done` | 400 `VALIDATION_ERROR`, the same `details` |
| Errors 18 | `GET /api/v1/nope` (with the key) | 404 `NOT_FOUND` |
| Errors 19 | `GET /nope` | 404 `NOT_FOUND` |
| Errors 20 | `GET /api/v1/tasks/%E0%A4%A` (a broken `%` escape; write the URL exactly so) | 400 `BAD_REQUEST`, message `Bad request`: a client error, not a 500 |
| Errors 21 | `GET /api/v1/nope` with **no key** (`req.deleteHeader`) | 401 `UNAUTHORIZED`: the key is checked before routing |
| Errors 22 | `POST /api/v1/tasks` with malformed JSON (a `text` body) and **no key** | 401 `UNAUTHORIZED`, message `Missing or invalid API key`: the key is checked before the body is parsed |

That is 7 + 22 = 29 requests. Two rows deliberately have **no** request: there is **no 409**, because no v1 endpoint throws `ConflictError`, and there is **no 500**,
because Bruno cannot trigger one without a debug route (Stage 10 proved it another way).

**Run it.** Start the server in terminal A (`npm start`) and run the collection in terminal B:

```bash
npm run bruno
```

In the reference build:

```text
Status        ✓ PASS
Requests      29 (29 Passed)
Tests         38/38
Assertions    59/59
```

(`bru run` only works at the root of a collection. The `npm run bruno` script does `cd bruno` for you. From `task-api/`, `bru run ... bruno` printed
`You can run only at the root of a collection`. In `@usebruno/cli` 4.2.1 no `-r` was needed to include the folders.)

**Prove the collection can fail.** In `src/middleware/error-handler.ts`, add one line inside the `error:` object of the response body:
`stack: appError.stack,`. Restart the server and run `npm run bruno` again. In the reference build it went to `✗ FAIL`: 7 requests passed, 22 failed, tests
16/38, with `expected [ Array(4) ] to deeply equal [ 'code', 'message', 'requestId' ]` on each non-validation error. **Remove the line**
(`git restore src/middleware/error-handler.ts`) and run it once more. It must pass again.

Commit:

```bash
cd ..
git add task-api/bruno
git commit -m "test(m05): add Bruno collection"
cd task-api
```

**Done when:**

- [ ] `npm run bruno` prints `✓ PASS` with every request passed (29 if you built the same set).
- [ ] The collection failed when you leaked a `stack`, and passes again now that it is removed.
- [ ] `git status --short` shows nothing under `src/`, and `bruno/` is committed.
- [ ] Every one of the 22 error requests is covered by the collection-level envelope test, and none sets its own copy of it.

### Stage 12 — The gate, then commit, push, pull request and merge

**Goal:** prove the deliverable from a clean state, document it, and deliver it as a pull request, with nothing pushed straight to `main`. This is
**v1**.

#### Step 1 — Write `task-api/README.md` and update the repo's

A reader who opens `task-api/` cold should be able to run it. The README needs: what the project is and which version it is at, the setup commands
(`nvm use`, `npm install`, `cp .env.example .env`, `npm run dev`), the scripts, the configuration table, the endpoint table (with which need the key), the
Task fields, the `status` filter contract from Stage 5, the error envelope and the table of `code`s with their statuses, and the folder layout. Say that
`prisma/`, `tests/` and `jest.config.js` do not exist yet and arrive in M07 and M09.

Then edit the **repo root's** `README.md`. It still says every stage lives under `modules/`. Add the exception, and why:

- Say that the Task API lives in `task-api/`, one project for M05, M07, M08 and M09, because each of those stages **changes the same project**
  instead of building a new one.
- Say that each version is a pull request, and that M06 (no API) and M10 (a different domain) are not part of it.
- Update the layout diagram, so `task-api/` appears next to `modules/`.

#### Step 2 — Re-run the gate

```bash
node --version
rm -rf node_modules && npm ci
npm run typecheck && echo "typecheck exit 0"
npm start        # terminal A
npm run bruno    # terminal B
git status -sb
```

**Why:** `npm ci` proves the lockfile alone rebuilds the project, and the PR's test plan should list results you have just seen. `git status -sb`
should show the branch, the root `README.md` modified, and `task-api/README.md` and `scratch/` untracked, with **no** `node_modules/` and no `.env`.

#### Step 3 — Delete the practice files and commit the docs

```bash
rm -rf scratch
cd ..
git add task-api/README.md README.md
git commit -m "docs(m05): describe the task-api layout and the v1 contract"
git log --oneline main..HEAD
```

`git log` should show your five commits: `chore`, `feat`, `feat`, `test` and `docs`, all `(m05)`.

#### Step 4 — Push the branch and open the pull request

```bash
git push -u origin feat/module-05/express-task-api-v1
gh pr create --repo <user>/<repo-name> --base main --head feat/module-05/express-task-api-v1 \
  --title "feat(m05): add Task API v1, an in-memory Express 5 API with a central error handler" \
  --body "<your description>"
```

Replace `<user>` and `<repo-name>` with your GitHub username and repo name. The description has three sections:

- **Summary:** why the stage exists and what it adds, in one short paragraph. Say that this is v1 of the Task API and that it lives in `task-api/`.
- **Changes:** one bullet per notable change (project setup, config, `AppError` hierarchy, middleware, central error handler, routes and handlers,
  `app.ts`/`server.ts`, Bruno collection, README).
- **Test plan:** one checked item per command you ran in Step 2, with what it printed (for example "`tsc --noEmit` exit 0", "`npm run bruno`: 29 of 29 requests
  passed", "SIGTERM prints `Server closed` and exits 0"). Never tick a box for something you did not run.

#### Step 5 — Merge it and look at the history

Merge the PR on GitHub with a **merge commit**, not a squash, so your commits stay visible. Then:

```bash
git fetch origin
git log --oneline --graph origin/main -8
```

The graph should show a `Merge pull request #<number> from ...` commit on `main` with your five `...(m05): ...` commits under it. GitHub writes that
message, so it is not a Conventional Commit. That is expected. **That merge is v1.**

**Done when:**

- [ ] `npm run typecheck` exits 0 after `npm ci` from a clean `node_modules`, and `npm run bruno` passes against a freshly started server.
- [ ] `git log --oneline main..HEAD` shows five commits (`chore`, `feat`, `feat`, `test`, `docs`), and no `scratch/`, `.env` or `node_modules/` path is in
      any of them (`git show --stat` lists none).
- [ ] A pull request from `feat/module-05/express-task-api-v1` into `main` exists on your build repo, with Summary, Changes and Test plan sections.
- [ ] The PR is merged, and `git log --oneline --graph origin/main -8` shows its merge commit with your five commits under it.

## 4. Concepts you need

Start with the **quick reference** if a term is new. The **essays** below it explain the bigger ideas. Read each essay after the stage named beside it.

How far to trust each statement:

- **Unmarked:** you ran or measured it in a guided step.
- **Marked *(background)*:** standard Express, HTTP and Node knowledge that the steps do not prove.
- **Marked *(reference build)*:** seen while building this guide, but not a guided step.

Check the marked points against the Express, MDN and Node docs before relying on them.

### Quick reference

| Term | Meaning |
| --- | --- |
| **Middleware** | A function `(req, res, next)` that runs for a request. It answers, calls `next()` to continue, or calls `next(err)` to jump to the error handler. |
| **Pipeline** | The registered middleware and routes, in order. A request flows through them top to bottom until something answers. |
| **Error-handling middleware** | A middleware with **four** parameters `(err, req, res, next)`. Express gives it errors and only errors. |
| **Router** | A mini-app from `express.Router()` that holds routes and is mounted at a path with `app.use`. |
| **Operational error** | An expected failure in a correct program (bad input, not found, wrong key). Becomes a specific 4xx. |
| **Programmer error** | A bug. Becomes a generic 500, with the detail in the log only. |
| **`AppError`** | This project's operational error: it carries its own status, `code` and optional `details`. |
| **Envelope** | The fixed JSON shape every error response has. |
| **Idempotent** | Doing it twice has the same effect as doing it once. `GET`, `PUT` and `DELETE` are. `POST` is not. `PATCH` need not be. |
| **Fail fast** | Check settings at startup and refuse to run with bad ones, instead of failing later on a request. |
| **Graceful shutdown** | On `SIGTERM`, stop taking new requests, let current ones finish, then exit. |
| **Preflight** | The `OPTIONS` request a browser sends before a cross-origin request that uses non-simple headers. CORS answers it. |

### HTTP and REST refresher (after Stage 1)

*(background, except where marked)* This is what the stage's HTTP list means, in the order you meet it in this API.

- **Methods and what they mean.** `GET` reads and must not change anything. `POST` creates (here: `POST /api/v1/tasks`). `PATCH` changes some fields of one
  resource (here), whereas `PUT` replaces the whole resource. `DELETE` removes. **Idempotency:** repeating `DELETE /tasks/1` leaves the same state (the
  task is gone), though the second reply is a 404 here, while repeating `POST` makes a second task.
- **Resource naming.** Nouns, plural, no verbs in the path: `/tasks`, `/tasks/:id`, not `/getTask`. The *method* is the verb.
- **Versioning in the URL.** `/api/v1` lets `/api/v2` exist beside it later without breaking existing clients.
- **Status codes in this API,** and when each is chosen:

  | Code | Meaning | Where it appears here |
  | --- | --- | --- |
  | 200 | OK | reads and updates |
  | 201 | Created | `POST /tasks` |
  | 204 | No Content (no body at all) | `DELETE /tasks/:id` |
  | 400 | The request is malformed or invalid | validation errors, bad JSON |
  | 401 | Not authenticated (who are you?) | missing or wrong API key |
  | 403 | Authenticated but not allowed (you may not) | **not produced in v1**: there are no roles |
  | 404 | No such resource | unknown task or route |
  | 409 | Conflicts with current state | **defined, not produced in v1** (`ConflictError`) |
  | 413 | Body too large | over `express.json`'s default 100 kb limit *(reference build)* |
  | 422 | Understood but semantically invalid | **not used**: this program returns validation failures as 400 |
  | 500 | Server bug | any unexpected error |

- **401 against 403:** 401 says "I don't know who you are", 403 says "I know who you are, and no". A missing API key is the first.
- **Headers and JSON.** `Content-Type: application/json` tells `express.json` to parse the body. Without it, `req.body` stays `undefined` (Stage 2).
  `X-Request-Id` and `X-API-Key` are custom headers.
- **Pagination conventions.** A list that can grow needs paging: either `?page=2&limit=20` or a cursor. **v1 has none**, because the in-memory list is short and
  paging is M07's concern. The point for now is that the list endpoint's *shape* is a bare array and would change if you added a `{ data, meta }` wrapper, which is why that is
  a decision to make on purpose.

### The middleware pipeline and why order matters (after Stage 9)

Stage 6 defined middleware, and Stages 9 and 10 put yours in order and broke it on purpose. Two things sit underneath that, and you can read both in the
code of the `router` package that Express 5 uses (`router@2.2.0` in the reference build, `node_modules/router/lib/layer.js`) *(reference build)*:

- **Express tells the kinds of function apart by their parameter count.** `Layer.handleRequest` skips any function with more than three parameters
  (`fn.length > 3`), and `Layer.handleError` skips any function whose count is not exactly four (`fn.length !== 4`). So an error handler written with
  three parameters is never given an error, which is what Stage 10's experiment 6 showed.
- **`next(err)` is a jump, not a call.** Once an error is passed on, the router walks forward through what is registered *after* that point and gives it
  only to functions with four parameters. Everything else in between is skipped. That is why the error handler is last, and why one registered before the
  routes never sees their errors.

One more rule of order that is a judgement and not a mechanism: **refuse cheaply before working expensively.** The API-key check is a string comparison, and
reading and parsing a body costs memory and time, so the key check comes first. Put the parser before it and a caller with no key can still make the
server buffer up to 100 kb per request.

There are three levels at which you attach middleware, and this project uses all three:

| Level | Example here | Applies to |
| --- | --- | --- |
| App-level | `app.use(requestId)`, `app.use(helmet())` | every request |
| Router-level | `app.use('/api/v1', apiKeyAuth, express.json(), v1Router)` | requests under `/api/v1` |
| Route-level | `router.get('/:id', getTask)` where you could put a middleware before `getTask` | one route |

**Built-in and third-party middleware** *(background, apart from what you ran)*: `express.json()` parses JSON bodies (you used it). `express.urlencoded()`
parses form bodies and `express.static()` serves files, and neither is needed for a JSON API. `helmet` and `cors` are third-party and you used both.
`morgan` and `pino-http` are ready-made request loggers. This project wrote its own one-line logger so the request ID is in it and so you see how a
`finish`-event logger works. In a real project either library is fine.

**Why the request ID goes first.** A request ID is useful only if everything that handles a request can see it. Putting it first means even a request
that a later middleware rejects has an ID in its log line and in its error response. Experiment 4 showed what happens otherwise.

### Operational errors against programmer errors (after Stage 7)

An **operational** error is something that can happen in a correct program: a wrong API key, a task that does not exist, a body that is not valid JSON. The
client caused it and can fix it, so it gets a specific status and a message worth reading. In this project these are the `AppError` subclasses.

A **programmer** error is a bug: reading a property of `undefined`, a typo, your temporary `throw new Error('boom ...')`. The client cannot fix it, and its
message may contain things they must never see (a connection string, a file path, a stack). It becomes a generic 500, and the detail goes to the log.

That one distinction is why `toAppError` exists in the handler. `err instanceof AppError` means "operational, safe to describe". Everything else is treated
as the second kind **by default**, so a bug can never accidentally leak: you have to opt a message in by throwing an `AppError`, not opt it out.

**Why not "no stack traces in production only"?** The stage's requirement is no stack traces in *production* responses. This project sends none in any
environment: a stack in development responses is one more behaviour to get wrong, and the server log has it anyway.

### Express 5 and async handlers (after Stage 7)

*(Stage 2 line 6 and Stage 7's `/async` route are your evidence.)* In Express 4 *(background: not run here)*, a handler that returned a rejected promise was
ignored by Express: nothing was listening for the rejection. On Node 15 and later, a rejection that nobody handles ends the process. You can see that
on its own: `node -e "Promise.reject(new Error('lost'))"` printed the error and exited with code 1 on Node 24.22 *(reference build)*. So in Express 4 one
`async` handler that threw could take the whole server down, and people wrapped every handler in `try { ... } catch (e) { next(e) }`, or in a helper that
did it for them.

Express 5 looks at what a handler **returns**. If it is a promise, Express attaches a rejection handler that calls `next(err)`. A `throw` inside an `async`
function is exactly a rejected promise, so it reaches the error handler by itself. You can read the code that does it, in the `router` package Express 5
uses (`router@2.2.0` in the reference build). It appears in your stack traces as `router/lib/layer.js`:

```bash
sed -n 142,170p node_modules/router/lib/layer.js
```

In `Layer.handleRequest` the router calls your handler in a `try`, so a synchronous `throw` goes to `next(err)`, and it keeps the return value. If that is a
promise it runs `ret.then(null, function (error) { next(error || new Error('Rejected promise')) })`. That one line is the try/catch you no longer write
*(reference build)*.

Two limits, so you do not over-claim:

- It covers **the promise the handler returns.** A rejection inside a callback you did not return (a `setTimeout`, an event listener, a floating promise you did
  not `await`) goes nowhere, and Express cannot see it.
- It **forwards** the error. It does not **shape** it. Turning the error into the envelope is still your handler's job.

**The answer to the Definition of done's question**, in short: because Express 5 turns a rejected promise returned by a handler into `next(err)` for you, so a
thrown or awaited-and-failed error reaches the central handler without a try/catch. Put it in your own words before you tick that box.

### Where a line of code belongs: `app.ts` or `server.ts` (after Stage 9)

Stage 9 gave the reason for the split. This is the rule for using it, which you will need every time you add something in M07-M09.

- **If it is about what the API does** (a middleware, a route, how an error is shaped), it goes in `app.ts` or a file `app.ts` imports.
- **If it is about the process** (which port, signals, timeouts, exit codes), it goes in `server.ts`.
- **The dependency runs one way.** `server.ts` imports `app.ts`, never the reverse. That is what keeps `app.ts` free of side effects, so a test or a
  script can import it without anything starting.

A related rule: **read the environment once.** `config.ts` is the only file that touches `process.env`, and it fails fast. Everything else imports the
checked `config`. In M08 this file becomes a Zod schema and nothing that imports it changes.

### What graceful shutdown does and does not promise (after Stage 9)

Stage 9 built it and showed it exit. These are the limits to know before you rely on it.

- **Why `SIGTERM`.** A process manager (and a container runtime) asks a server to stop with `SIGTERM` *(background)*. If the process is still running after a grace
  period, it is killed with `SIGKILL`, which no program can catch. So the timeout in `server.ts` should be shorter than the manager's grace period, or your own
  forced exit never gets to run.
- **Why `SIGINT` too.** It is what Ctrl+C sends. Handling both means the shutdown you test by hand in development is the one that runs in production.
- **The exit code is part of the contract.** 0 means a clean stop and 1 means it was forced or `close` failed, and a process manager reads that.
- **It only waits for requests it can see.** `server.close` waits for open connections to end. It cannot wait for work you started and did not tie to a request,
  such as a timer or a background job *(background)*. In M07 you will also have a database connection to close in the same place.
- **`tsx watch` uses it.** It restarts by stopping the server first, so graceful shutdown also runs on every save in development *(reference build)*.

### Stretch: `express-rate-limit` (not required, not built in the reference build)

*(Reference build: tried in a separate scratch project with `express@5.3.0` and `express-rate-limit@8.7.1`, not added to `task-api/`.)* A rate limiter counts requests per client
and refuses the ones over a limit, which protects a public endpoint from being hammered. The package is middleware, so it goes in your pipeline like `helmet` does. Its only
peer requirement is `express >= 4.11`, and it installed cleanly next to Express 5.

```js
app.use(rateLimit({ windowMs: 60_000, limit: 3, standardHeaders: 'draft-7', legacyHeaders: false }));
```

With a limit of 3, the fourth request in the window printed, in the reference build:

```text
1 200 ratelimit=limit=3, remaining=2, reset=60 retry-after=null {"ok":true}
3 200 ratelimit=limit=3, remaining=0, reset=60 retry-after=null {"ok":true}
4 429 ratelimit=limit=3, remaining=0, reset=60 retry-after=60 Too many requests, please try again later.
```

Two things to know before you add it to this API:

- **The default 429 is plain text, not your envelope.** That would break "every error response uses the fixed envelope". The package has a `handler` option that replaces
  its response, so the fix is to pass the error on to your central handler: `handler: (_req, _res, next, options) => next(new AppError(options.statusCode, 'RATE_LIMITED', '...'))`.
  In the reference build that gave a 429 with a JSON `{"error":{...}}` body. `RATE_LIMITED` is a new `code` you would be adding, so the Task API contract's error table would
  grow by one, and the `Errors/` collection would need a request that trips it.
- **The default store is in memory, per process** *(from the package's documentation)*. Restarting the server resets the counts, and with several server processes each keeps its
  own, so a client can get more than the limit in total. A shared store is the fix, and is out of scope here. Behind a proxy, the client's address is also the proxy's unless
  Express's `trust proxy` setting is configured *(background, not tried)*.

## 5. Common pitfalls

| Mistake | What happens | Do this |
| --- | --- | --- |
| `app.get('*', ...)` | Express 5 throws at registration: `Missing parameter name at index 1: *` | `/*splat`, or `app.use(handler)` with no path |
| Treating `req.params.splat` as a string | It is an array: `/files/a/b/c` gives `["a","b","c"]` | Join it |
| Reading `req.body.title` with no check | `req.body` is `undefined` when no parser ran, so it throws. Also true when the request has no body | Check that `req.body` is an object first |
| `req.query = {...}` | Throws: `Cannot set property query ... which has only a getter` | Read it, validate it, keep your own variable |
| Typing `req.query.status` as `string` | `?status=a&status=b` gives an array | Take it as `unknown` and validate |
| `nvm use \| tail -2` | Prints `Now using node v24`, but `node -v` still says 22: a pipe runs `nvm use` in a subshell | `nvm use` on its own line, then `node --version` |
| `npm init -y` | Writes `"type": "commonjs"` | Set `"type": "module"` by hand |
| `constructor(public statusCode: number)` in an error class | `erasableSyntaxOnly` error | Declare the field, assign it in the constructor |
| An error handler with three parameters | Express treats it as ordinary middleware. The built-in **HTML** error page, with a stack trace, answers instead | Four parameters, even if you do not use `next` |
| Registering the error handler before the routes | Same HTML page: it never sees the routes' errors | Register it last |
| `express.json()` before `apiKeyAuth` | A caller with no key can make the server buffer and parse a body up to 100 kb, and malformed JSON gets a 400 instead of a 401 | Put it after the key check, in the same `/api/v1` line |
| Ignoring the error that `app.listen`'s callback receives | A busy port prints "listening" and exits 0 | Log it and `process.exit(1)` |
| Mapping only the client errors you thought of | `GET /api/v1/tasks/%E0%A4%A` is a 500, and so is an undecodable body | Keep any 4xx `status` or `statusCode`, with a fixed message. Do not require `expose` |
| `express.json()` after the routes | `req.body` is `undefined` in the handlers, so a valid body is a validation error | Before the routes |
| `notFound` before the routes | Every URL is a 404, including `/health` | After the last route, before the error handler |
| `requestId` after the routes or the error handler | `"requestId": "unknown"` in every envelope and log line | First |
| No case for `entity.too.large` | A body over 100 kb is a generic 500, which blames the server for the client's mistake | Map the two body-parser errors in `toAppError` |
| Putting `stack` or `err.message` in the response | Leaks internals. Bruno's key-set test fails, if you wrote it that strictly | Log it, do not send it |
| A collection test that only checks "there is an `error` key" | A leaked `stack` passes | List the exact keys, and break it on purpose once |
| Leaving the temporary `throw` in a handler | The endpoint is broken and may be committed | `git restore` it, then `grep -rn boom src` |
| Throwing in a callback instead of returning the promise | Express 5 cannot see it. Nothing reaches the error handler | `await` it inside an `async` handler |
| `bru run` from `task-api/` | `You can run only at the root of a collection` | `cd bruno`, or `npm run bruno` |
| An `.env.example` with a real key | A secret in git history | Fake values only. The real `.env` is ignored |
| Inventing a 409 rule so `ConflictError` has a use | A business rule the contract never asked for, which M07-M09 must keep | Define the class, document it, throw it nowhere |
| Inventing a `{ data, meta }` response wrapper | The contract fixes the error envelope, not a success one. A wrapper is a decision to make on purpose | Return the resource. Decide a wrapper with the trainer, in M07 |
| `git add .` in a folder with `scratch/` | Practice files get committed | Stage by name or folder, and `rm -rf scratch` before the last commit |
| Windows: `kill -TERM`, `lsof` | Not available | Use Ctrl+C for `SIGINT`. The commands here are untried on Windows |

## 6. Check yourself against the definition of done

Tick each one only when you have just verified it.

- [ ] **v1 is merged by pull request.** A PR from `feat/module-05/express-task-api-v1` into `main` exists and is merged with a merge commit, and
      `git log --oneline --graph origin/main -8` shows it.
- [ ] **Every error path returns the exact envelope shape.** Every error response, not one, is `{ error: { code, message, details?, requestId } }`: the 22 `Errors/`
      requests in Bruno all pass the collection-level envelope test, `details` appears only on `VALIDATION_ERROR`, `requestId` equals the `X-Request-Id` header, and
      there is no stack in any response (you saw the 500 in Stage 10 and the `stack` mutation fail in Stage 11).
- [ ] **The Bruno collection passes.** `npm run bruno` against a running server prints `✓ PASS` with all requests passed, and the collection is committed in
      `task-api/bruno/`.
- [ ] **You can explain why Express 5 no longer needs a try/catch wrapper around async handlers.** Without looking, you can say what Express 5 does with a promise a handler
      returns, what that means for a `throw` in an `async` function, and name one case it does **not** cover.
- [ ] Supporting evidence for the deliverable: `app.ts` exports the app and never calls `listen`; `server.ts` is the only file that does, and shuts down on `SIGTERM`; the
      `AppError` hierarchy and a single central handler produce the envelope; request-ID and logging middleware exist; validation is hand-written (no `zod` in
      `package.json`); the store is in memory; there is a 404 handler, a health endpoint, CORS and an API-key check.

## 7. Next step

If every box in section 6 is ticked, this stage is done. This is the **Week 2 Friday gate**.

1. Keep the repo and carry the same workflow forward: a branch named `feat/module-NN/<topic>`, Conventional Commits and a pull request. Never push to `main`
   directly.
2. Keep `task-api/`. **You are not starting a new project in M07.** It carries forward as v2, v3 and v4: M07 replaces the in-memory store with a database
   through Prisma, M08 replaces your hand-written validation and `config.ts` with Zod, and M09 tests the app you export from `app.ts`.
3. Move on to **M06 — SQL Fundamentals with SQLite** (Week 3 · Days 1-2). It does not change the API: it designs the relational schema (`users`, `tasks`, `tags`,
   `task_tags`) that M07 turns into Prisma. M06's code lives under `modules/`, not in `task-api/`.
4. Two decisions for later: the contract fixes the Task's four fields and the error envelope, but not a success-response wrapper or a pagination shape. Pagination and
   sorting arrive with M07, and that is where to settle both.
