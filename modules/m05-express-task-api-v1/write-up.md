# M05 Write-Up — Express: Routing, Middleware & Error Handling (Task API v1)

> Written from what I captured as I built it: raw notes after each slice, then this document. Written
> so a trainee starting this stage next cohort could follow my path on their own.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo, [`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
in a new top-level folder, `task-api/`, on the branch `feat/module-05/express-task-api-v1`. It is
**not** under `modules/m05-.../` like M01-M04. See "Why it's built this way" for why. The folder:

```text
task-api/
├── package.json
├── package-lock.json
├── tsconfig.json
├── .env.example
├── README.md
├── src/
│   ├── app.ts                       builds and exports the app, never calls listen
│   ├── server.ts                    the only listen, graceful shutdown
│   ├── config.ts                    environment, read once, fails fast
│   ├── errors.ts                    AppError and its subclasses
│   ├── middleware/
│   │   ├── request-id.ts            logger.ts          api-key-auth.ts
│   │   └── not-found.ts             error-handler.ts
│   ├── routes/
│   │   ├── index.ts                 tasks.routes.ts
│   ├── handlers/
│   │   └── tasks.handlers.ts
│   ├── validation/task-validation.ts
│   ├── store/task-store.ts
│   └── types/task.ts
└── bruno/                           the collection: 29 requests in two folders
```

- **Task API v1**, an in-memory Express 5 REST API in TypeScript, with five endpoints under
  `/api/v1/tasks` (list, create, get, patch, delete), plus `GET /health`, which needs no key.
- **A central error handler** that is the only code that builds an error response, and an
  **`AppError` hierarchy** that feeds it: `BadRequestError`, `ValidationError`, `UnauthorizedError`,
  `NotFoundError` and `ConflictError`.
- **Middleware:** a request-ID middleware, a hand-written logger, `helmet`, `cors`, a simple API-key
  check on `/api/v1` followed by `express.json`, a 404 handler and the error handler.
- **Hand-written validation**, one function per input, in `validation/task-validation.ts`.
- **`app.ts` and `server.ts` as two files**, with graceful shutdown on `SIGTERM` and `SIGINT`.
- **Config from environment variables** in `config.ts`, validated once at startup.
- **A Bruno collection** with 7 happy-path requests and 22 error-path requests, with a
  collection-level test that checks the error envelope on every error response.
- **Merged to `main` by pull request.** Five commits on `feat/module-05/express-task-api-v1`, merged as
  [#5](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/5) (see "Checkpoint evidence").

The stack as installed: Node 24.22 (the repo's `.nvmrc` says `24`), `express@5.3.0`,
`typescript@7.0.2`, `tsx@4.23.15`, `helmet@8.3.0`, `cors@2.8.6`, `@types/express@5.0.6`,
`@types/node@24.19.2` and `@usebruno/cli@4.2.1`. `npm run typecheck` exits 0.

## Why it's built this way (key decisions)

### One `task-api/` folder, not one folder per stage

I had to choose between one `task-api/` for M05, M07, M08 and M09, or one folder per module that each
follows the `task-api/` layout. I chose one folder.

- M07 does not start a new project. It adds Prisma to this one, and the arch doc's reference layout
  is a single `task-api/` with `prisma/`, `tests/`, `bruno/` and `src/` side by side.
- Four copies would mean four `package.json` files, four lockfiles and four Bruno collections to keep
  in step, and the migration history could not carry from v2 to v3 to v4. The M07 checkpoint says the
  Bruno collection "still" passes, which only makes sense if it is the same collection.
- A version is still easy to find: each stage is a pull request, so the merge of the M05 branch *is*
  v1. I updated the build repo's `README.md` to say so, and added a line to the root `.env.example`
  index. M06 (SQL, no API) and M10 (a different domain) stay out of `task-api/`.
- I did not create `prisma/`, `tests/` or `jest.config.js`. Git cannot track an empty folder, and
  Jest would need packages M05 does not use. They arrive with M07 and M09.

### The `AppError` hierarchy and the fixed envelope

The wire format is fixed by the stage: `{ error: { code, message, details?, requestId } }`. I wrote the
implementation behind it.

- `AppError` extends `Error` and carries `statusCode`, `code` and an optional `details`. The
  subclasses are thin: each only passes its own status and code to `super`.

  | Class | Status | `code` |
  | --- | --- | --- |
  | `BadRequestError` | 400 | `BAD_REQUEST` |
  | `ValidationError` | 400 | `VALIDATION_ERROR` (the only one with `details`) |
  | `UnauthorizedError` | 401 | `UNAUTHORIZED` |
  | `NotFoundError` | 404 | `NOT_FOUND` |
  | `ConflictError` | 409 | `CONFLICT` |

- **`ConflictError` exists but nothing throws it.** The stage names it as an example class, but it does
  not define any v1 rule that conflicts. A rule such as "titles must be unique" would be a business
  rule I made up, and M07-M09 would have to keep it. So the class is defined and documented, and no
  endpoint uses it. The Bruno collection therefore has no 409 request.
- **Validation is a 400, not a 422.** 422 is in the stage's status-code list, but the arch doc says a
  validation failure becomes a 400 with field-level `details`, and M08's Zod errors will do the same.
- **The error handler decides what a client may see.** It turns anything thrown into an `AppError`.
  Known `AppError`s keep their status and message. Two errors from `express.json` are recognised by
  their `type` (`entity.parse.failed` becomes a 400 `BAD_REQUEST`, `entity.too.large` a 413
  `PAYLOAD_TOO_LARGE`). Any other error that carries a 4xx `status` or `statusCode` (the router's
  malformed-`%` error, an undecodable body) keeps that status with a fixed message: `BAD_REQUEST` /
  `Bad request` for a 400, and `CLIENT_ERROR` / `Request could not be processed` for the rest. Everything
  else is a programmer error and becomes a 500 `INTERNAL_ERROR` whose message is always
  `Internal server error`. The real error goes to `console.error` with the request id, and nowhere else.
  The first version of this had only the two body-parser cases, and a code review showed that was not
  enough (see "What tripped me up").
- **No stack trace in any environment**, not only production. The brief says "no stack traces in
  production responses". Putting them in development responses would be an extra behaviour to explain
  and to get wrong, and I never needed it: the server log has the stack.
- `details` is added with a conditional spread, so it is absent (not `null` or `[]`) unless the error
  has some, and the keys stay in the documented order.

### Middleware order

```text
requestId → requestLogger → helmet → cors → GET /health →
  /api/v1 [apiKeyAuth → express.json → v1Router] → notFound → errorHandler
```

I swapped things on purpose and ran each one. The results are in "What tripped me up". What I took
from them:

- **`express.json` must come before the routes.** After them, `req.body` is `undefined` in the
  handlers, and a perfectly valid create gets a validation error.
- **`express.json` should come after the key check.** My first version had it as a global middleware
  at the top. A code review showed that a caller with no key could then make the server read and parse
  a body (up to its 100 kb limit), and got a 400 or 413 instead of a 401. It now sits in the
  `/api/v1` line after `apiKeyAuth`. Refuse cheaply before working expensively.
- **`errorHandler` must come last.** Express only gives an error to handlers registered *after* the
  place it was thrown. Registered first, it never sees a route's error and Express's built-in HTML
  error page answers instead, with a stack trace in development.
- **`notFound` must come after every route**, or it answers for all of them.
- **`requestId` must come before anything that reads the id.** After the routes, every error says
  `"requestId": "unknown"` (my own fallback string), and every log line starts with `unknown`.
- **Logger before request-ID did not break anything.** I expected it to. It does not, because the
  logger reads the id in the response's `finish` event, long after `requestId` has run. I still keep
  `requestId` first, because it is the one the others depend on, and I would not rely on `finish`
  timing as the reason.
- `helmet` and `cors` sit after the id and the logger so that a request they reject would still have
  an id and a log line.

### The API-key check, and where it sits

I included it because the stage's Scope lists "a simple API-key auth" among the custom middleware, even
though the Build checklist does not. It is on `app.use('/api/v1', apiKeyAuth, express.json(), v1Router)`,
so it is router-level middleware, and `/health` is deliberately outside it.

- The key comes from `API_KEY`, which is required: the server refuses to start without it.
- A missing key and a wrong key give the same 401 message, so the response does not say which it was.
- Both keys are hashed with SHA-256 before `timingSafeEqual`, which throws when its two inputs differ
  in length and which compares in constant time.
- A consequence I accepted: an unknown URL under `/api/v1` is a 401 without a key and a 404 with one,
  because the key is checked before routing. For the same reason malformed JSON without a key is a
  401, not a 400.
- **It does not check how strong the key is.** `API_KEY=a` starts fine, and so does the fake example
  key in `NODE_ENV=production`. I thought about refusing the example value, but any rule about a key's
  content would be a fragile guess, and deployment is out of scope for this stage. The README tells the
  reader to set their own key.

### Rate limiting is researched, not built

`express-rate-limit` is a stretch goal in the stage, and I did not add it to `task-api/`. I did try it in a separate scratch project with `express@5.3.0` and
`express-rate-limit@8.7.1` (peer `express >= 4.11`, installed with 0 vulnerabilities). With `limit: 3`, the fourth request got a `429` with `retry-after=60` and
the body `Too many requests, please try again later.` as **plain text**. That would break the fixed envelope, so using it here would need its `handler` option to call
`next(new AppError(429, ...))` into the central handler, and a new `RATE_LIMITED` code. I tried that too and got a JSON `{"error":{...}}` 429. Both would change the
contract, which is why I left it out of v1. Its default store is per process and in memory, so it resets on restart.

### The success response is the bare resource

The stage fixes the error envelope and the Task fields, but not the shape of a success. I return the
task itself (`GET /:id`, `POST`, `PATCH`) and a bare array for the list. I did not invent a
`{ data, meta }` wrapper. Pagination belongs to M07, and that is where a wrapper would earn its place.
If M07 needs one, that is a change to the contract and should be made on purpose.

### The task type: four fields, no timestamps

The stage fixes the Task's fields at `id`, `title`, `description` (optional) and `status`. My M04
`Task` also had `createdAt` and `updatedAt`. I left them out of v1: nothing in the API uses them, and
the database will own them in M07. The statuses and the `isTaskStatus` guard are carried over from M04
unchanged. The types are copied into `task-api/src/types/task.ts`, not imported from `modules/m04-...`,
so the API does not depend on another stage's folder.

### The `status` filter

`GET /api/v1/tasks?status=<value>` is the only query parameter in v1. I added it because the stage says
to learn "route params and query strings" and because `req.query` is one of the Express 5 changes.

- Accepted values: `todo`, `in_progress`, `done`, case-sensitive.
- No `status`: every task. A valid `status`: only tasks with that status.
- An unknown value, an empty `?status=` and the parameter given twice (`?status=todo&status=done`, which
  Express 5's default parser turns into an array) are all a 400 `VALIDATION_ERROR` with
  `path: "status"`. An invalid *value* is never ignored, because a typo in the value that silently
  returns the full list looks like it worked.
- **Other query keys are ignored**, like unknown body fields: `?Status=done` and `?state=done` are not
  `status`, so they return every task. I first described the rule as "an invalid filter is never
  ignored", which overclaimed, and a code review caught it.
- All three invalid forms have their own Bruno request (15, 16 and 17, the last two also assert
  `details.path`), so the contract and the tests say the same thing.
- Pagination and sorting are not in v1. They are M07's.

### Why validation is hand-written

The stage says so, and the reason holds up once you have written it. By the end I had about 100 lines in
`task-validation.ts` doing four things over and over: check the type, collect a message, remember to
collect rather than stop at the first, and build the `{ path, message }` list. M08 turns all of that
into schemas. If I had started with Zod I would have been using it without knowing what it saves me.

Choices inside it:
- Each parse function takes `unknown` and returns a typed value or throws one `ValidationError` that
  lists every failing field, not only the first.
- Unknown keys in a body are **ignored**, not rejected. The server owns `id`, so a client sending one
  should not break anything. Zod strips unknown keys by default as well, so M08 will not change this.
- `title` is trimmed and must not be empty. I set no maximum length, because a limit would be a rule
  the stage does not give.
- A PATCH with none of `title`, `description`, `status` is a 400, not a silent no-op.
- `:id` must be a UUID, because that is what the store makes. A malformed id is a 400 and a
  well-formed unknown id is a 404.

### The store is async on purpose

Every method of the in-memory store returns a promise, although a `Map` is synchronous. That means the
handlers are already written for a database, so M07 swaps the store and leaves the handlers alone. It
also means a handler really awaits something, which is what makes the Express 5 behaviour (a rejected
promise reaches the error handler) real in this code and not only in a demo.

## How to build it (teach it to the next trainee)

### Build a central error handler that produces one consistent shape

I will use my own request, `GET /api/v1/tasks/not-a-uuid`, to show the reasoning. Six pieces, in the
order I built them, each with the reason it exists.

**1. Decide the shape first, in one type.** Before any handler exists, write down what every error
response looks like. Mine is the fixed envelope. Put it in a type so the compiler checks the one place
that builds it:

```ts
type ErrorBody = {
  error: {
    code: string;
    message: string;
    details?: { path: string; message: string }[];
    requestId: string;
  };
};
```

*Why first:* if the shape is decided at the end, each endpoint will already have invented its own.

**2. Make errors carry their own meaning.** A handler that decides "is this a 404?" by looking at the
message would be fragile. Make the *error* know:

```ts
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

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(404, 'NOT_FOUND', message);
  }
}
```

*Why the long constructor:* `erasableSyntaxOnly` (from M04) rejects `constructor(public statusCode: number)`
because Node's type stripping cannot erase a parameter property. So declare each field, then assign it.
*Why `new.target.name`:* it sets the error's `name` to the subclass, so a log line says
`ValidationError`, not `Error`.

**3. Throw, don't respond.** Everything that can go wrong in a handler *throws* an `AppError` and
returns nothing. My `parseTaskId` throws `new ValidationError([{ path: 'id', message: 'Must be a valid UUID' }])`.
The handler that called it has no `res.status(400)` for this case, and no `try/catch`.

*Why:* in Express 5 a handler that throws, or returns a rejected promise, is passed to the error handler
by Express. So the handler stays about the happy path.

**4. Write the handler, with four parameters.**

```ts
export function errorHandler(err: unknown, req: Request, res: Response, next: NextFunction): void {
  if (res.headersSent) {
    next(err);
    return;
  }

  const appError = toAppError(err);
  const requestId = requestIdOf(res);

  if (appError.statusCode === 500) {
    console.error(`${requestId} ${req.method} ${req.originalUrl} unexpected error:`, err);
  }

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

*Why four parameters:* Express tells an error handler from a normal middleware by counting parameters.
With three, it is treated as ordinary middleware and never receives the error (I tried it: Express's own
HTML error page answered, with a stack trace). `next` stays in the signature for the `headersSent` case,
where the response has already started and the only safe thing is to let Express close the connection.

**5. Turn what is not yours into something safe.** `toAppError` is the gate. An `AppError` passes
through. A client error with a 4xx `status` (a body that does not parse, a bad `%` escape in the URL) is
mapped to its status with a fixed message. *Anything else* becomes
`new AppError(500, 'INTERNAL_ERROR', 'Internal server error')`. The client gets a generic message, and
the real error is logged server-side with the request id. I proved this with a temporary
`throw new Error('boom: connection string postgres://admin:hunter2@db/prod')` in a handler: the response
said only `Internal server error` and the request id, and the log had the full text and stack. I removed
the throw and checked `src` was identical to a backup.

**6. Register it last, after a 404 that goes through it.** The 404 middleware does not respond. It calls
`next(new NotFoundError(...))`, so an unknown URL gets the same envelope as every other error:

```ts
app.use(notFound);
app.use(errorHandler);
```

**How I checked the shape.** I did not trust myself to look at each response. I put one test in the Bruno
collection's `collection.bru`, so it runs after *every* request:

```js
if (res.status >= 400) {
  test("error response uses the fixed envelope", function () {
    const error = res.body.error;
    expect(Object.keys(res.body)).to.eql(["error"]);
    const allowed = error.code === "VALIDATION_ERROR"
      ? ["code", "message", "details", "requestId"]
      : ["code", "message", "requestId"];
    expect(Object.keys(error)).to.eql(allowed);
    expect(error.requestId).to.equal(res.headers["x-request-id"]);
    // ...
  });
}
```

Then I broke it on purpose: I added `stack: appError.stack` to the envelope and ran the collection. It went
from 38/38 tests to 16/38 (it was 31/31 to 12/31 before three requests were added), with
`expected [ Array(4) ] to deeply equal [ 'code', 'message', 'requestId' ]` on every non-validation error. My first version of this check only tested the top-level keys and
would have let the stack through, which is why the second version lists the exact keys.

### The rest, in the order I built it

Run each session from `~/node-js-foundation-training`, `nvm use` on its own line, then check
`node --version` prints v24.

**0. Branch and project.** `git switch main && git pull && git switch -c feat/module-05/express-task-api-v1`,
`mkdir task-api`, a `package.json` with `"type": "module"` and an `engines` field, the M04 `tsconfig.json`
copied unchanged, then `npm install express@5 cors helmet` and
`npm install -D typescript@7 tsx @types/node@24 @types/express@5 @types/cors @usebruno/cli`. Same install-script warning
for `esbuild` as in M04 (`tsx` ran without my approving it).

**1. Config.** `loadConfig(env)` collects every problem and throws one error that lists them. Run with
`PORT=abc NODE_ENV=staging` and no `API_KEY` it printed three bullets, one per problem, rather than
stopping at the first.

**2. The task types, the store, the validators.** The types are the M04 ones minus the timestamps. The store
is a `Map` behind async methods that return copies. Each validator takes `unknown` and collects problems.

**3. Handlers, then routes, as two files.** The handlers know nothing about URLs. They are typed
`Request<Params, ResBody, ReqBody, Query>` with the body and query as `unknown`. `routes/tasks.routes.ts` is
five lines that say which URL reaches which handler. `DELETE` ends with `res.sendStatus(204)`.

**4. `app.ts`, then `server.ts`.** `app.ts` ends with `export const app = express()` and its middleware. No
`listen`. `server.ts` imports it, listens, and on `SIGTERM` or `SIGINT` calls `server.close()` with a 10-second
`unref()`'d fallback timer. Running `npm start` and sending `SIGTERM` printed `SIGTERM received, closing the
server`, then `Server closed`, and exited with code 0.

**5. The Bruno collection.** Plain `.bru` files in `bruno/`, two folders: `Tasks` (the happy path, in order, where
create stores `taskId` in a variable for the later requests) and `Errors`. Run from inside `bruno/`:
`bru run --env local`. With no path it runs the whole collection, folders included (in `@usebruno/cli@4.2.1` I
needed no `-r`).

## Concepts worth explaining

### Express 5 no longer needs a try/catch around async handlers

The Express 5 half of this I ran. The Express 4 half is *(background)*: I did not run Express 4 here. In Express 4, an `async` handler that threw returned a rejected promise, and Express ignored the promise:
nothing was listening for the rejection. On Node 15 and later a rejection nobody handles ends the
process (I checked that part alone: `node -e "Promise.reject(new Error('lost'))"` printed the error and
exited with code 1 on Node 24.22). So one `async` handler that threw could take the whole server down,
and everyone wrapped handlers in `try { ... } catch (e) { next(e) }`, or in a helper that did the same.

Express 5 (through its `router` package) looks at what a handler returns. If it is a promise, Express
attaches a rejection handler that calls `next(err)` for you (I read the code: see "Checkpoint evidence"). A thrown error in an `async` function is
exactly a rejected promise, so it reaches the error handler by itself. I checked it in a small standalone
app: `app.get('/reject', async () => { await Promise.resolve(); throw new Error('rejected in async handler'); })`
answered `500 {"reachedErrorHandler":true,"message":"rejected in async handler"}` with no try/catch in sight.

Two limits, so I do not overclaim. It only covers the promise *the handler returns*: a rejection inside a
callback you did not return (a `setTimeout`, an event handler) still goes nowhere. And it forwards the
error but does not shape it. The error handler still has to turn it into the envelope.

### Operational errors against programmer errors

- An **operational** error is something that can happen in a correct program: a bad request body, a task
  that does not exist, a wrong API key. The client caused it and can fix it, so it gets a specific status
  and message. In my code these are the `AppError` subclasses.
- A **programmer** error is a bug: a null that should not be null, a typo, my `throw new Error('boom ...')`.
  The client cannot fix it, and its message may contain things they must not see. It becomes a generic 500,
  and the detail goes to the log.

That one distinction is why the error handler has a `toAppError` step. `instanceof AppError` is the test for
"operational, safe to describe". Everything else is treated as the second kind.

## What tripped me up

- **A bare `*` is gone, and the error is about a parameter name.** `app.get('*', ...)` in Express 5 throws
  at registration: `Missing parameter name at index 1: *`. The Express 5 form is `/files/*splat`, and
  `req.params.splat` is then an **array** (`/files/a/b/c` gave `["a","b","c"]`), not a string. An optional
  segment is `/books{/:id}`: `/books` left `req.params.id` undefined (my route printed it as `null`) and `/books/7` gave `"7"`.
- **`req.body` is `undefined`, not `{}`, when no parser ran.** A POST with `Content-Type: application/json`
  but no `express.json()` gave `typeof req.body === 'undefined'`. In my API it also happens with no body at
  all: no `Content-Type`, so `express.json()` skips it. Reading `body.title` on `undefined` throws, so every
  parse function starts with a "must be a JSON object" check.
- **`req.query` cannot be assigned.** `req.query = {...}` throws `Cannot set property query of #<IncomingMessage>
  which has only a getter`. Read it, validate it, keep the result in your own variable.
- **A repeated query key is an array.** `?status=todo&status=done` gives `['todo','done']`, not a string, so
  the filter validator has to expect `unknown`, and a plain `typeof === 'string'` guard handles it.
- **Put the error handler first and you get an HTML page with a stack trace.** With `errorHandler` registered
  before the routes, `GET /api/v1/tasks/not-a-uuid` returned `400 Bad Request` as `text/html` with the full
  stack in the page, from Express's built-in handler (`NODE_ENV` was `development`). Declaring the handler
  with three parameters did exactly the same.
- **A middleware swap did not do what I expected.** I put the logger before the request ID expecting log lines
  without an id. It did not (see "Middleware order"): the logger reads the id on `finish`. The swap that
  *did* break the ids was moving `requestId` after the routes, which printed `unknown` in both the logs
  and the envelope.
- **A body-too-large error is a 413, not a 500.** `express.json()` has a 100 kb limit. A 200 kb body without a
  case for `entity.too.large` would have fallen through to my generic 500, which would have blamed the server
  for the client's mistake. It is Bruno request 07, which builds the large body in a pre-request script.
- **The first draft of my collection-level test was too weak.** It checked only the top-level keys. I only
  noticed when I thought about what a leaked stack would look like, and the deliberate `stack:` mutation
  (31/31 down to 12/31) showed the stronger version works.
- **`bru run` has to run from the collection folder.** The collection root is the folder with `bruno.json`. From
  `task-api/`, `bru run -r --env local bruno` printed `You can run only at the root of a collection`. The `bruno`
  npm script is `cd bruno && bru run --env local`.
- **A busy port looked like a clean start.** My first `server.ts` ignored the argument of
  `app.listen`'s callback. A second copy on the same port printed `Task API listening on ...` and then
  exited with code 0. In Express 5 a startup error such as `EADDRINUSE` arrives in that callback and is
  not thrown. It now logs the error and exits 1. A code review found this, and I reproduced it before
  fixing it.
- **Two client errors were a 500.** `GET /api/v1/tasks/%E0%A4%A` (a broken `%` escape) and a POST with
  `Content-Encoding: br` and a body that is not valid brotli both returned `500 INTERNAL_ERROR`, with a
  stack trace in the log. The envelope was right and nothing leaked, but the status blamed the server for
  the client's mistake. My first fix required `expose: true` on the error, and the request with the bad
  `%` escape still returned 500, because the router sets only `status` (400) on that error. The handler
  now keeps any 4xx `status` or `statusCode`. Bruno request 20 covers the bad `%` escape.
- **My Bruno collection could not see either of those.** It had no request that reached them, which is why
  all 26 requests passed. I added requests 20 (bad `%` escape), 21 (unknown route without a key) and 22
  (malformed JSON without a key), and `details.path` assertions on 16 and 17.
- **`tsx watch` restarts by sending the server a `SIGTERM`.** Touching `src/app.ts` under `npm run dev` printed
  `[tsx] change in ./src/app.ts Restarting...`, then `SIGTERM received, closing the server`, then `Server
  closed`, then the new `listening` line. So graceful shutdown runs on every save in development.

## Checkpoint evidence

**v1 merged to `main` by pull request.**
[#5, `feat(m05): add Task API v1, an in-memory Express 5 API with a central error handler`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/5),
from `feat/module-05/express-task-api-v1` into `main`. It was merged on 2026-10-21 (05:40 UTC) with a merge commit, so the five
commits are visible in the history:

```text
$ git log --oneline --graph origin/main -7
*   34dc299 Merge pull request #5 from kiannaalexandra-gragg/feat/module-05/express-task-api-v1
|\
| * 4f00656 docs(m05): describe the task-api layout and the v1 contract
| * 8249f31 test(m05): add Bruno collection
| * cff7339 feat(m05): add task store, validation, routes, app and server
| * bb4d4e9 feat(m05): add config, AppError hierarchy, middleware and central error handler
| * 46199f6 chore(m05): add task-api package.json, tsconfig, lockfile and env example
|/
*   aa0aac0 Merge pull request #4 from kiannaalexandra-gragg/feat/module-04/typescript-on-node
```

`gh pr view 5` reported the state `MERGED`, with the base `main`, the head `feat/module-05/express-task-api-v1`, 56 changed files and
those five commits. The test plan in the PR lists the commands whose results are shown below. The PR also records that `npm audit` reports 12
vulnerabilities, all in the dev-only `@usebruno/cli` tree and none in the production dependencies.

The rest of this section is what I ran on 2026-10-21, on Node 24.22, against `npm start`.

- **Type-check:** `npm run typecheck` (`tsc --noEmit`) exited 0 with no output.
- **Bruno collection passing.** `npm run bruno`:

  ```text
  Status        ✓ PASS
  Requests      29 (29 Passed)
  Tests         38/38
  Assertions    59/59
  ```

  I ran it several times while building. This is the run after the last change to the source and the
  collection. Its first version was 26 requests, 31/31 tests and 53/53 assertions: a code review then
  found three gaps and I added requests 20 to 22 (see "What tripped me up").
- **Every error path returns the exact envelope.** The collection-level test runs after each of the 22
  requests in `Errors/`, and none is skipped. The paths covered, by `code`:
  - `UNAUTHORIZED` 401: missing key, wrong key, an unknown route under `/api/v1` without a key, and
    malformed JSON without a key.
  - `VALIDATION_ERROR` 400, with `details`: create with three bad fields, create without a title, create
    without a body, invalid id, update with an empty body, update with an invalid status, and three bad
    `status` filters.
  - `BAD_REQUEST` 400: malformed JSON, and a malformed `%` escape in the URL.
  - `PAYLOAD_TOO_LARGE` 413: a 200 kb body.
  - `NOT_FOUND` 404: unknown task (get, update, delete), the deleted task, an unknown route under `/api/v1` and
    one outside it.
  - `INTERNAL_ERROR` 500: **not in Bruno.** It needs a handler that throws, and I did not want a debug route
    in the API. I proved it with a temporary throw instead, and the response was
    `{"error":{"code":"INTERNAL_ERROR","message":"Internal server error","requestId":"..."}}` with
    nothing from the thrown error in it. The throw is removed and `src` matches the backup I took before it.
  - `CONFLICT` 409: **no request, because no endpoint throws it.**
- **The collection can fail.** The `stack:` mutation made the run `✗ FAIL`, 7 requests passed and 22 failed,
  16/38 tests. I reverted it.
- **Operations:** `SIGTERM` exits 0 after `Server closed`. A preflight `OPTIONS` with
  `Origin: http://example.test` got `204` with `Access-Control-Allow-Origin: *` and
  `Access-Control-Expose-Headers: X-Request-Id`. `helmet` sets `X-Content-Type-Options: nosniff` and the
  rest, and `X-Powered-By` is absent. An incoming `X-Request-Id: my-trace-1` was reused, and
  `bad id with spaces!` was replaced with a new UUID.
- **A busy port:** a second copy of the server on the same port printed `Could not listen on port 3000:
  Error: listen EADDRINUSE ...` and exited 1.
- **What I did not test:** a request that is still in flight when `SIGTERM` arrives (my handlers answer in
  about a millisecond, so there is none to observe), Windows, and any Node other than 24.

I have not tagged the merge. A `task-api-v1` tag is deferred until the EM has reviewed v1, because pushing a tag publishes it.

**Why Express 5 needs no try/catch around async handlers**, in my own words. I checked the claim three ways before
writing it: the standalone app (`/reject` answered `500 {"reachedErrorHandler":true,...}` with no try/catch), my own handler
check (`/async` throws after an `await` and still came back as the `NOT_FOUND` envelope), and the source of the router that
Express 5 uses, `router@2.2.0`, in `node_modules/router/lib/layer.js`.

An `async` function never throws to its caller. Whatever goes wrong inside it, it hands back a **rejected promise**. In
Express 4 nothing looked at that promise, so nothing handled the rejection, and on current Node that ends the process. That is why every handler needed
its own `try { ... } catch (e) { next(e) }`: the `catch` was the only way to get the error to the error handler.

In Express 5 the router does that part itself. When it calls my handler it keeps what the handler returns, and if that is a
promise it attaches a rejection callback. This is the actual code from `Layer.handleRequest`:

```js
const ret = fn(req, res, next)
if (isPromise(ret)) {
  ret.then(null, function (error) {
    next(error || new Error('Rejected promise'))
  })
}
```

So `throw new NotFoundError(...)` inside my `async` handler becomes a rejected promise, the router sees it, and it calls
`next(error)`, which is exactly what my `catch` would have done. The same function has a `try/catch` around the call for a
handler that throws synchronously, so both kinds reach the error handler. My handlers just `throw`, and `errorHandler` turns
it into the envelope.

What it does **not** do, which I checked by reading the code and not by running it: it only watches the promise the handler
*returns*. A rejection in something I did not return or `await` (a `setTimeout` callback, an event listener, a floating
promise) never goes through that `.then`, so Express cannot catch it. And it only forwards the error. It does not decide the
response: that is still my handler's job, which is why I still needed `toAppError` to stop an unexpected error from leaking.

## What I'd do differently

- I would ask for a code review **before** writing the documents, not after. I wrote the write-up and the
  guide against the first version of the code, and the review's fixes then changed `app.ts`, `server.ts`
  and `error-handler.ts`, so both documents and three embedded code blocks had to be redone.
- I would write the Bruno collection-level envelope test **before** the error handler, and watch it fail first. I
  wrote the handler, saw the curl output looked right, and only then realised my check could not catch a leaked
  stack. Writing the check first would have given me the strong version straight away.
- I would run the middleware-swap experiments while writing `app.ts`, not after. I wrote the order and the
  comment above it first and confirmed them afterwards, and one expectation (logger against request ID) turned
  out wrong.
- I would decide the success-response shape (bare, or `{ data }`) out loud with the trainer before building. The
  stage fixes the error envelope but not that, and I made the call alone. If the fixed contract is meant to
  cover it, the stage should say so.
- On the fixed envelope and Task contract: **they caused no friction.** They saved me the argument I would
  otherwise have had with myself about error shapes. The one gap was the success shape above, and the missing
  409 rule, which I handled by not inventing one.
