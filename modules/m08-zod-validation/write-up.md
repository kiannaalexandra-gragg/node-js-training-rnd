# M08 Write-Up — Zod Validation & Type Inference (Task API v3)

> Written from what I captured as I built it: raw notes after each slice, then this document. Written
> so a trainee starting this stage next cohort could follow my path on their own.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo,
[`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
inside `task-api/`, built on the branch `feat/module-08/zod-validation` and merged into `main` by
[PR #8](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/8). The Task API is one
project that grows from M05 to M09, so v3 is a change to `task-api/` in place, not a new folder.

```text
task-api/
├── src/
│   ├── schemas/task.schemas.ts       NEW  every Zod schema, and the request types derived from them
│   ├── middleware/validate.ts        NEW  validate({ params, body, query }), typed, and validOf(res)
│   ├── env.ts                        NEW  parseEnv, and the database settings (DATABASE_URL, LOG_QUERIES)
│   ├── config.ts                     the M05 loadConfig replaced by a Zod schema
│   ├── db.ts                         reads its two settings through env.ts, no hand-written check
│   ├── middleware/error-handler.ts   one new branch: ZodError becomes the 400 envelope
│   ├── routes/tasks.routes.ts        validate(...) on all five routes
│   ├── handlers/tasks.handlers.ts    no parsing: reads typed values through validOf(res)
│   ├── services/task-service.ts      takes the inferred types, and checks them against Prisma's
│   ├── types/task.ts                 the hand-written input types are gone
│   └── validation/task-validation.ts DELETED (252 lines)
├── bruno/                            11 requests added: Errors/25 to 28, and a Dates/ folder of 7
├── README.md                         describes v3, adds a Validation section
└── package.json                      zod ^4.6.7
```

- **Zod schemas for create, update, list query and params.** The create body is a `z.object`. The
  update body is derived from it with `.partial()` plus one `.refine` ("at least one field"). The
  params schema is `{ id }`. The list query has `status`, `q`, `sortBy`, `order`, `page` and
  `pageSize` with their defaults.
- **A typed `validate({ params, body, query })` middleware.** It runs the schemas, passes a failure to
  `next()` as the `ZodError`, and puts the parsed values on `res.locals.valid`. The handlers read them
  through `validOf(res)`, and take the type from the same schema objects the routes pass to `validate`.
  Two guards came out of the code review: a typo'd key in `validate({ ... })` is a compile error, and a
  handler mounted without `validate` fails with a clear message instead of a bare `TypeError`.
- **`ZodError` mapped into the M05 envelope** inside the existing central handler: one `{ path, message }`
  in `details` per field. Nothing else about the envelope changed.
- **Env validation** with Zod, in `config.ts` and `env.ts`, failing fast at startup.
- **Date and id boundaries pinned in Bruno.** Four requests in `Errors/` (hour 24, 29 February in a year
  that is not a leap year, hour 25, and a well-formed id that is not an RFC UUID, which must be a 404) and a
  `Dates/` folder that creates one task and walks its `dueDate` through a leap day, year 0050 (with a read
  back from SQLite), the last second of year 9999, a minutes-only time with an offset, and `null`.
- **No hand-written validation left.** `task-validation.ts`, the `isTaskStatus` guard and the hand-written
  `CreateTaskInput`, `UpdateTaskInput` and `ListTasksQuery` types are deleted. The 252-line validator was
  replaced by a 144-line schemas file.

The stack as installed: Node 24.22.0 (my shell defaulted to 22.23.0, see "What tripped me up"),
`zod@4.6.7`, `express@5.3.0`, `typescript@7.0.2`, `prisma@7.10.1` and `@prisma/client@7.10.1`,
`better-sqlite3@12.11.1`. `npm run typecheck` exits 0.

## Why it's built this way (key decisions)

### Schema design per endpoint

I kept every rule and message v1 and v2 had, so the stage is a change of mechanism, not of behaviour. Each
decision below is one I checked against the old validator (see "Keeping v2's behaviour" in this section).

| Endpoint | Schemas | Notes |
| --- | --- | --- |
| `POST /tasks` | `createTaskBody` | `title` trimmed, `min(1)`. `description` optional. `status` is `z.enum(TaskStatus)`, so the allowed values come from the Prisma enum, not a second list. `priority` is `z.int()` from 1 to 3. `dueDate` is a union of three ISO forms, transformed to a `Date`, and nullable. |
| `PATCH /tasks/:id` | `taskParams`, `updateTaskBody` | The update body is `createTaskBody.partial().refine(...)`, so it is derived, not written again. |
| `GET /tasks/:id`, `DELETE /tasks/:id` | `taskParams` | `id` is `z.guid()`. |
| `GET /tasks` | `listTasksQuery` | Enums with `.default()`, and `page` and `pageSize` as digits-then-number. |

Four choices were not obvious:

- **Unknown body keys are stripped, not rejected** (`z.object`, not `z.strictObject`). v2 ignored them, the
  README documents that, and `{ "id": "evil", "userId": 999 }` in a create body must not reach Prisma.
  Checked over HTTP: a create with `id`, `userId` and `foo` returned 201 with `userId: 1`, a server-made id,
  and no `foo`.
- **The messages are v2's, set through the `error` option.** Zod's own messages read differently
  ("Invalid input: expected string, received undefined"), and the Bruno collection asserts the v2 text
  word for word. A `title` that is missing is `Required`, one that is the wrong type is `Must be a string`,
  and a blank one is `Must not be empty`: a function passed as `error` reads `issue.input` to tell the
  first two apart.
- **`z.guid()` for `:id`, not `z.uuid()`.** v2 accepted any 8-4-4-4-12 hex string. `z.uuid()` also checks
  the version and variant digits. I ran both over the same ids: `11111111-1111-1111-1111-111111111111`
  and `12345678-1234-1234-1234-123456789012` were accepted by v2 and `z.guid()` and rejected by
  `z.uuid()`. With `z.uuid()` such an id would have changed from a 404 to a 400.
- **`page` and `pageSize` check the digits first.** `z.coerce.number()` on its own accepted `"1e2"`, `" 5"`,
  `"5 "`, `"+5"`, `"0x10"`, `"0b11"` and `"1.0"`, all of which v2 rejected (I ran them). So the schema is a string
  that matches `/^\d+$/`, then `Number`, then `z.int().min(1)` (and `.max(100)` for `pageSize`).

### How I handled `req.query` being read-only in Express 5

In Express 5 `req.query` is a getter, so assigning the parsed query back to it does not work. I did not
try to. `validate` writes every parsed part to `res.locals.valid`, and the handlers read from there through
`validOf(res)`: `.query`, `.body`, `.params`. I chose `res.locals` over a `req.valid` property because a
`req.valid` needs a global `declare` merge into Express's `Request` type, and `res.locals` already exists
for "data for the rest of this request". `req.body` and `req.params` are left as the client sent them.

The part that needed care is the type. `validate(schemas)` is generic over the schema object, and returns a
`RequestHandler` whose `Locals` type is `{ valid: { [part]: z.output<schema> } }`. Each route has one
request schema object (`updateTaskRequest = { params: taskParams, body: updateTaskBody }`). The route
passes it to `validate`, and the handler types its `res` with `ValidLocals<typeof updateTaskRequest>`. I
did not have to give the router any explicit generics: Express's `get` and `post` signatures inferred the
`Locals` type from the two functions.

Two limits showed up in review, and I tested both with `tsc` before changing anything:

- **The types do not force a route to call `validate`.** `Response.locals` is `Record<string, any>` joined
  with the handler's `Locals`, so a handler typed for `{ valid: ... }` still compiles when mounted alone
  (`router.get('/x', listTasks)`). Only mounting a handler against the *wrong* schema object is caught. So the
  handlers read through `validOf(res)`, which throws `res.locals.valid is missing: this handler is mounted
  without validate()` if it was never set. Through a throwaway Express app, that mount gave a 500
  `INTERNAL_ERROR` body to the client and that message in the server log. No unvalidated input is passed on.
- **`validate({ qurey: schema })` compiled.** A generic constraint does not apply excess-property checks, and
  `Valid<S>` typed `valid.qurey` as parsed data that never existed, because the loop only visits `params`,
  `body` and `query`. `schemas: S & OnlyKnownParts<S>` (any other key must be `never`) makes it a compile
  error. A scratch file with two typo'd calls (alone and beside a real part) failed to compile where
  expected, and every real use (`validate(getTaskRequest)`, `validate({})`, a custom subset) still compiled.

### The order the parts are checked in

v2's handlers called `parseTaskId` and then `parseUpdateTask`, so a request with a bad id and a bad body
reported only the id. Collecting problems from all three parts at once would have changed that, so
`validate` checks `params`, then `body`, then `query`, and stops at the first part that fails. Inside a
part every bad field is still reported. Over HTTP: `PATCH /tasks/not-a-uuid` with `{"status":"nope"}`
returned only `{ "path": "id", "message": "Must be a valid UUID" }`.

### How a `ZodError` becomes the M05 envelope

`validate` calls `next(zodError)`, and the central handler gets a new branch before the body-parser
branch: `err instanceof ZodError` becomes `new ValidationError(details)`. The envelope code is untouched.
The mapping is `path.join('.')` for `path`, with two rules I had to add:

- **An issue with no path is about the body as a whole**, so its `path` is `body`. That is what v2 sent for
  "Must be a JSON object" and for "At least one of title, description, status is required".
- **One `details` entry per field.** `priority: 1e300` made Zod report two `too_big` issues for the same
  path: one from `z.int()` (above the safe-integer limit) and one from `.max(3)`. v2 reported a field
  once. I kept the first message per path.

### How I kept Zod and Prisma aligned

Zod is the request contract and Prisma's types are the persistence contract. The service now takes the
types derived from the schemas, and makes the compiler check the one against the other:

```ts
const data = { ...input, userId: owner.id } satisfies Prisma.TaskUncheckedCreateInput;
const data = patch satisfies Prisma.TaskUpdateInput;
```

I used `TaskUncheckedCreateInput` rather than the `TaskCreateInput` the brief names as an example, because
the service sets `userId` directly and `TaskCreateInput` wants an `owner` relation object instead. Raw
`req.body` never reaches Prisma: the service only ever receives what a schema produced, with unknown keys
already stripped. The check does its job in some cases: when I renamed `title` to `name` in the schema (a
throwaway change), `tsc --noEmit` failed in `task-service.ts` with `Property 'title' is missing ... but
required in type 'TaskUncheckedCreateInput'`. The handlers themselves did not fail, because they only pass the
value on.

**It does not catch everything, and review made me test where it stops.** With a scratch file and `tsc`:

- Caught: a wrong type on a field both sides have (`priority` as text), and a field Prisma requires that the
  schema stopped producing (the rename above).
- **Not caught: an extra field the schema produces that Prisma has no column for.** A `nickname` field
  compiled cleanly, for both the create and the update. TypeScript checks excess properties only on a fresh
  object literal, and these values are a spread and a variable.
- **Not caught on the update side:** `TaskUpdateInput` has every field optional, so renaming `title` to `name`
  in the update schema still satisfies it. (My rename demonstration above only proves the create side.)

So `satisfies` is a net for type drift and for dropped required fields, not a proof that the contracts match.
M09's tests are where an extra field would show up.

### Env validation, and why the database settings are in their own file

`config.ts` is a Zod schema with the same rules and the same messages as the M05 `loadConfig`: `PORT` 1 to
65535 (default 3000), `NODE_ENV` one of three (default `development`), `API_KEY` not blank, and
`CORS_ORIGIN` as `'*'` or a trimmed list. It fails fast and lists every problem.

The M07 backlog said `DATABASE_URL` and `LOG_QUERIES` were read in `db.ts` and M08 was the natural place to
move them into the config. I did not put them in `config.ts`. `config.ts` runs `loadConfig(process.env)`
when it is imported, and it needs `API_KEY`. `db.ts` is also imported by `prisma/seed.ts` and
`scripts/check-prisma-errors.ts`. If `db.ts` had imported `config.ts`, `npm run db:seed` would have refused
to run without an `API_KEY` it never uses. So the shared `parseEnv` helper and the database schema are in
`src/env.ts`, which has no import-time effect, and `db.ts` calls `loadDbConfig(process.env)`. I ran the
seed and `check:errors` with only `DATABASE_URL` set (no `.env`): both exit 0, as the v2 seed did.

### Keeping v2's behaviour: how I checked it

Matching messages is not the same as matching behaviour, so before deleting the old validator I ran it
against the new schemas over the same inputs, and compared the accepted output or the `details` array.
The script (not committed) had 345 body, params and query cases: whitespace and non-string titles,
priority bounds and non-integers, 60-odd `dueDate` strings, no body and array bodies, update with no
fields, repeated query keys, empty values and `"1e2"`-style numbers. It found, in order:

1. `T10:00Z` (minutes only, with a zone), which v2 accepted and `z.iso.datetime({ offset: true })`
   rejected. Fixed with a third union member, `precision: -1`.
2. Four differences, two inputs each through create and update:
   - **`2026-01-01T24:00:00Z`**: v2 accepted it and parsed it as 2 January. Zod rejects hour 24.
   - **A year before 0100, such as `0050-06-15`**: v2 rejected it (`Date.UTC` reads 0 to 99 as 1900 to
     1999, so v2's calendar check failed). Zod accepts it. I confirmed 0000 to 0099 reject in v2 and 0100
     accepts.

I first accepted both as "implementation quirks". Then I was asked to decide them against the documented
date contract instead, so I did that (next section).

### The two date differences, decided against the contract

The only written contract for `dueDate` is the row in the `task-api` README (written in M07): "ISO 8601
string or `null`. A date (`2026-12-31`) or a date-time, and the day must exist: `2026-02-30` is a 400". The
M05 and M07 briefs and the PRD do not describe the field's format, hours or years (I searched them). So
neither difference is v2 behaviour that the contract promises, and I decided each on its merits:

- **Hour 24 stays rejected.** ISO 8601 does not settle it for me. Older editions allow `24:00:00` for
  midnight only (quoted in
  [W3C mailing-list posts](https://lists.w3.org/Archives/Public/www-voice/2005AprJun/0047.html) on the 1988
  and 2000 editions), and a source describing a later edition says interchange has no end-of-day form. I
  could not read ISO 8601-1:2019 itself, so I do not claim what it says. RFC 3339, the profile most tools
  use, has hours 00 to 23 in its grammar (`time-hour = 2DIGIT ; 00-23`, section 5.6) and says so in words in
  section 5.7: "Although ISO 8601 permits the hour to be "24", this profile of ISO 8601 only allows values
  between "00" and "23"" ([RFC 3339](https://www.rfc-editor.org/rfc/rfc3339)). v2 also *stored the next day* for this input, which
  is a surprising result for a due date. M07's backlog listed it as "not fixed", which is a note, not a
  promise. So: an hour is 00 to 23, and the README now says so. The cost is that a client that sent
  `T24:00:00Z` now gets a 400, and that is written down.
- **Years 0000 to 0099 stay accepted.** The contract says the day must exist, and `0050-06-15` does. v2
  rejected it only because its calendar check used `Date.UTC(0050, ...)`, which treats 0 to 99 as 1900 to
  1999, so it compared against the wrong year. That is a bug, not a rule. ISO 8601's four-digit year is
  `0000` to `9999`, and the README now says that. I checked the storage side too: Bruno `Dates/02` sets
  `0050-06-15` and `Dates/03` reads it back from SQLite as `0050-06-15T00:00:00.000Z`.

A wider boundary comparison against v2 (leap years 1900, 2000, 2024 and 2100, every month end, years
`0000`, `0099`, `0100`, `9999`, `10000`, `-0001`, offsets from `-12:00` to `+24:00`, seconds `59` and `60`,
hours `23` to `25`, minutes `60`): 59 inputs, 53 identical, 6 different, and the 6 are exactly the two
classes above (four years from 0000 to 0099 and the two hour-24 forms). Leap days, month ends, `9999-12-31`,
five-digit years, offsets and minute precision agree with v2.

The same test for the environment: 37 `config` cases (every `PORT`, `NODE_ENV`, `API_KEY` and `CORS_ORIGIN`
edge I could think of), 0 differences in the result or the message. 12 database cases: the same accept and
reject, and the one message difference is the `Invalid environment:` prefix that `DATABASE_URL`'s message
now shares with the others.

### Response shaping

The brief says to use Zod "to shape responses and strip internal fields", and my instruction was to do it
only if the documented response contract needs it. It does not. The README documents `userId` as part of a
task, and Bruno request 02 asserts the exact nine keys including `userId`, so stripping it would break the
documented contract. I added no response schema.

### Other decisions

- `LOG_QUERIES` keeps v2's rule: only the text `true` turns it on. I ran `z.stringbool()`: it is
  case-insensitive, turns on for `true`, `1`, `yes`, `on`, `y` and `enabled`, and rejects other text
  (`treu`, an empty string), so it would have changed both what turns logging on and what stops startup.
- `API_KEY` and `DATABASE_URL` are checked for being blank but kept untrimmed, as v2 did.
- I did not touch the request-ID middleware's regex check on the incoming `X-Request-Id` header. It is
  hand-written validation of a header, but the brief's scope is body, params, query and environment.
- **The UTC year boundary is not enforced.** The four-digit limit is on the text sent. A zone offset moves
  the instant, so `0000-01-01T00:00:00+23:59` is accepted and sent back as `-000001-12-31T00:01:00.000Z`,
  and `9999-12-31T23:59:59-23:59` as `+010000-01-01T23:58:59.000Z`. I confirmed v2 does the same at the top
  end (and refused the lower input only because of the year bug). Rejecting a UTC year outside 0000 to 9999
  would be a behaviour change that needs a decision about the contract, so I changed no code, put a note in the
  README, and listed it as a question for the EM in the backlog.
- Zod Mini and `z.toJSONSchema` are awareness only; I ran each once (see Concepts).

## How to build it (teach it to the next trainee)

### A typed validation middleware for Express 5 and Zod 4

The worked example is the `PATCH /tasks/:id` route, because it needs two parts (the id and the body).

1. **Write one schema per part, and derive instead of repeating.** The create body is written once. The
   update body is `createTaskBody.partial().refine(...)`, so a new field on create is a new optional field
   on update with no second edit.

   ```ts
   export const updateTaskBody = createTaskBody
     .partial()
     .refine((patch) => Object.keys(patch).length > 0, {
       error: 'At least one of title, description, status is required',
     });
   ```

   Why `Object.keys(patch).length`: Zod's output for an optional key that was not sent has no key at all,
   so an empty patch is an empty object.

2. **Group the schemas a route needs into one object, and export it.**

   ```ts
   export const updateTaskRequest = { params: taskParams, body: updateTaskBody };
   ```

   This object is the single thing the route and the handler both refer to, so a handler's type cannot
   name a different schema from the one its route was given. (It cannot make a route call `validate` at all:
   see the second limit in the decision above.)

3. **Write `validate` so it is generic over that object.**

   ```ts
   type Schemas = { params?: z.ZodType; body?: z.ZodType; query?: z.ZodType };
   type OnlyKnownParts<S> = { [K in Exclude<keyof S, keyof Schemas>]: never };

   export type Valid<S extends Schemas> = {
     [K in keyof S]: S[K] extends z.ZodType ? z.output<S[K]> : never;
   };
   export type ValidLocals<S extends Schemas> = { valid: Valid<S> };

   const PARTS = ['params', 'body', 'query'] as const;

   export function validate<S extends Schemas>(
     schemas: S & OnlyKnownParts<S>,
   ): RequestHandler<Request['params'], unknown, unknown, Request['query'], ValidLocals<S>> {
     return (req, res, next) => {
       const valid: Record<string, unknown> = {};
       for (const part of PARTS) {
         const schema = schemas[part];
         if (schema === undefined) continue;
         const result = schema.safeParse(req[part]);
         if (!result.success) {
           next(result.error);
           return;
         }
         valid[part] = result.data;
       }
       res.locals.valid = valid as Valid<S>;
       next();
     };
   }

   export function validOf<V, B>(res: Response<B, { valid: V }>): V {
     const valid: V | undefined = res.locals.valid;
     if (valid === undefined) {
       throw new Error('res.locals.valid is missing: this handler is mounted without validate()');
     }
     return valid;
   }
   ```

   Why `safeParse` and `next(result.error)`: `parse` throws a `ZodError` (I checked), but `safeParse` returns
   the failure as a value, which suits a loop that must stop at the first failing part. Why
   `res.locals`: see the decision above.

4. **Turn the `ZodError` into the envelope in the central handler, not in `validate`.** Add one branch to
   `toAppError`: `if (err instanceof ZodError) return new ValidationError(zodErrorToDetails(err));`, where
   the mapper joins each issue's `path`, calls an empty path `body`, and keeps the first message per path.

5. **Put `validate` on the route, and type the handler from the same object.**

   ```ts
   tasksRouter.patch('/:id', validate(updateTaskRequest), updateTask);

   export async function updateTask(
     _req: Request,
     res: Response<Task, ValidLocals<typeof updateTaskRequest>>,
   ): Promise<void> {
     const { params, body } = validOf(res);
     res.status(200).json(await taskService.update(params.id, body));
   }
   ```

   `params.id` is a `string` and `body` is the update type, both because the schemas said so.

6. **Delete the old code in the same step.** Once a route has `validate`, remove the `parse*` call in its
   handler. When the last handler is done, delete the file, and let `tsc --noEmit` list anything that still
   imports it.

7. **Prove it.** `npm run typecheck`, then the Bruno collection, then a grep for the patterns the old code
   used (`if (!`, `typeof x !==`, `.trim() ===`, `isNaN`, `.test(`).

## Concepts worth explaining

### `z.input` against `z.output`: one schema, two types

A schema that only checks (a string, a number) has the same type going in and coming out. A schema that
changes the value does not, and Zod keeps both. `listTasksQuery` takes `{ page: '2' }` and produces
`{ page: 2, pageSize: 20, sortBy: 'createdAt', order: 'desc' }`. I checked with the compiler that
`z.input<typeof listTasksQuery>` wants `page` as text and rejects `2`, while `z.output` has `page` as a
number and rejects `'2'`. The same is true of `dueDate` (text in, `Date` out). The rule I use: a handler
and a service always take the **output** type, because that is what the schema gave them, and the input
type only matters for something that builds a request. `z.infer` is the same as `z.output`. Getting this
wrong is easy: a hand-written `CreateTaskInput` with `dueDate?: string` would compile, and be wrong the
moment a handler used it.

### A `ZodError` is a list of issues, and the envelope is a view of it

A `ZodError` has `issues`, and each issue has a `code`, a `path` (an array of keys, empty for the value as
a whole) and a `message`. Everything else is a formatter over that list. For the same bad body
(`{ title: '  ', priority: 4.5, status: 'x', dueDate: '2026-02-30' }`):

- `z.flattenError(error)` gives `{ formErrors: [], fieldErrors: { title: [...], status: [...], ... } }`: one
  level, an array of messages per field. It is the best fit for a flat form, and it has no place for a
  nested path.
- `z.treeifyError(error)` gives `{ errors: [], properties: { title: { errors: [...] }, ... } }`, which
  follows the shape of the data. It is the one for nested objects and arrays.
- `z.prettifyError(error)` gives text (`✖ Must not be empty` then `→ at title`), for a person reading a
  terminal. I used it nowhere in the API, and the env messages are written by hand.

The envelope's `details` is none of these: it is `[{ path, message }]` with a dotted path, so I map the
`issues` list myself. That is a decision about who the reader is: `details` is for a client program that
wants a field name and a message, in an order that matches the schema's field order, one per field.
`treeifyError` showed one more thing worth knowing: for `dueDate` (a union) it listed the messages of all
three union members (`Invalid ISO date`, `Invalid ISO datetime`, ...) instead of mine, because the custom
message is on the union's own issue. The issues list is the thing to map, not the tree.

### Awareness only: Zod Mini and `z.toJSONSchema`

Neither is used in v3. `import * as z from 'zod/mini'` is a function-style API (`.check(minLength(1))` in
place of `.min(1)`) that Zod offers for places where bundle size matters. I only ran one schema to see it
reject an empty string, and I did not measure its size. `z.toJSONSchema(schema)` turns a schema into JSON Schema: for
`z.object({ title: z.string().min(1), priority: z.int().min(1).max(3).optional() })` it printed
`"required":["title"]`, `"minLength":1`, `"minimum":1`, `"maximum":3` and `"additionalProperties":false`. That is
the starting point for generating OpenAPI documentation from the same schemas, and I did not take it
further.

## What tripped me up

- **The baseline Bruno run failed 16 of 36 requests before I had changed anything.** Create and list
  returned 500. The server log said `ERR_DLOPEN_FAILED`: `better_sqlite3.node` "was compiled against a
  different Node.js version using NODE_MODULE_VERSION 137. This version of Node.js requires
  NODE_MODULE_VERSION 127". My shell was on Node 22.23.0, and the repo pins 24 in `.nvmrc`. Putting Node
  24.22.0 first on `PATH` and rerunning gave 36 of 36. If I had not run the baseline first, I would have
  blamed the 500s on my own changes.
- **`z.coerce.number()` does not pipe from a string.** Piping a string schema into `z.coerce.number().pipe(...)`
  was a type error: `Argument of type 'ZodPipe<ZodCoercedNumber<unknown>, ZodInt>' is not assignable to
  parameter of type '$ZodType<any, string, ...>'`, because a coerced number's input type is `unknown`, not
  `string`. After the digits check, `.transform(Number)` does the same job and types correctly.
- **`z.coerce.number()` is looser than it looks.** See the decision above: `"1e2"`, `" 5"` and `"0x10"` all
  parse. I only found it because I compared against the old function instead of reading the Zod docs and
  assuming.
- **`z.iso.datetime()` insists on seconds.** `2026-01-01T10:00Z` failed, while
  `z.iso.datetime({ offset: true, precision: -1 })` accepted it and rejected `10:00:00Z`, so the schema
  needed both.
- **I assumed a duplicate issue and was wrong.** I added the first-message-per-path rule expecting `4.5` to
  report twice. It did not: `z.int()` stops the checks after it fails. I only kept the rule because I then
  ran the 345 cases without it: `1e300`, `9007199254740992` and a `pageSize` of `9007199254740993` do
  report twice, and the comparison shows 9 differences without the rule against 4 with it.
- **My `PORT` message echoed `"NaN"` instead of `"abc"`.** An `error` function's `issue.input` is the value
  at that step, and after coercion that is the number. I moved the check onto the string (`refine`, calling
  `z.int().min(1).max(65535)` on `Number(text)`) so the message shows what was typed.
- **Importing `config.ts` is not free.** It evaluates `loadConfig(process.env)` on import, so a test script
  that imported it threw `API_KEY is required` straight away. That is the fail-fast working, and it is also
  why the database settings could not live there.

## Checkpoint evidence

All of the following is from `task-api/` on Node 24.22.0. Most of it was captured on the branch
`feat/module-08/zod-validation` before the merge. The last part, "After the merge", was run on a clean clone
of `main`.

**v3 merged by pull request.** [PR #8](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/8),
"feat(m08): add Zod validation and type inference for Task API v3", from `feat/module-08/zod-validation` into
`main` of my own repo. It has 6 commits (29 files changed, 766 insertions, 423 deletions) and was merged by
me on 2026-10-21 at 11:40 UTC, as a merge commit, `4deec36`. `git log --graph --oneline origin/main` afterwards:

```text
*   4deec36 Merge pull request #8 from kiannaalexandra-gragg/feat/module-08/zod-validation
|\
| * 579b52e docs(m08): describe Task API v3
| * d047c80 test(m08): pin the date and id boundaries in Bruno
| * c51d344 feat(m08): validate the environment with Zod
| * 40b730b refactor(m08): replace the hand-written validation with the schemas
| * 71dcd54 feat(m08): add the Zod schemas, the validate middleware and the ZodError mapping
| * eaaff82 chore(m08): add Zod 4
|/
*   3604be1 Merge pull request #7 from kiannaalexandra-gragg/feat/module-07/prisma-orm-migrations
```

The commits carry no `Co-Authored-By` trailer, and `main` was not pushed to directly: the branch went up
first, and the PR was opened from it.

**`tsc --noEmit` proving handler types come from `z.infer`.**

- `npm run typecheck` exits 0.
- The input types are defined only by `z.output` on a schema:
  `grep -rnE "(type|const) (CreateTaskInput|UpdateTaskInput|ListTasksQuery)" src` returns three lines, all
  in `src/schemas/task.schemas.ts`, each `export type X = z.output<typeof ...>`, and `grep -rnE "interface |isTaskStatus|parseCreateTask|parseUpdateTask|parseTaskId|parseListQuery" src
  prisma scripts` (generated client excluded) finds nothing.
- The throwaway rename (`title` to `name` in `createTaskBody`) made `tsc` fail in the service:
  `src/services/task-service.ts(63,49): error TS2741: Property 'title' is missing in type '{ name: string; ... userId: number; }' but required in type 'TaskUncheckedCreateInput'`. I reverted it, and `typecheck` exits 0 again.
- The compiler also confirmed `z.input` against `z.output` on a scratch file (since deleted): `page: '2'` is
  accepted as input and `page: 2` is an error, and the reverse on the output.

**No manual `if (!body.title)`-style checks remain.**

```text
$ grep -rnE "if \(!|typeof [a-zA-Z_.]+ [!=]==? '|isNaN|Number\.isInteger|Number\.isSafeInteger|\.trim\(\) [!=]==|\.test\(" src prisma scripts --include='*.ts' --exclude-dir=generated
src/config.ts:35:      .refine((key) => key.trim() !== '', { error: API_KEY_MESSAGE }),
src/env.ts:10:  if (!result.success) {
src/env.ts:35:      .refine((url) => url.trim() !== '', { error: DATABASE_URL_MESSAGE }),
src/middleware/request-id.ts:15:  const id = incoming !== undefined && SAFE_ID.test(incoming) ? incoming : randomUUID();
src/middleware/request-id.ts:25:  return typeof id === 'string' ? id : 'unknown';
src/middleware/validate.ts:42:      if (!result.success) {
src/middleware/error-handler.ts:28:  if (typeof err === 'object' && err !== null && 'type' in err && typeof err.type === 'string') {
src/middleware/error-handler.ts:39:  if (typeof err !== 'object' || err === null) return undefined;
src/middleware/error-handler.ts:44:    if (typeof value === 'number' && value >= 400 && value < 500) return value;
scripts/check-prisma-errors.ts:76:  if (!ok) failed = true;
```

None is a check on a request body, param or query. Two are Zod rules (`.refine` keeping the key untrimmed),
`result.success` and `ok` are results, the error-handler lines are type guards on a thrown value, and
`request-id.ts` is the header regex from M05 that I left (see decisions).

**Invalid input never reaches Prisma.** I started the server with `LOG_QUERIES=true`, which prints the SQL
behind every Prisma call, and counted `prisma:query` lines in the log. After five invalid requests (a
create with every field wrong, a create with no JSON content type, a `PATCH` with a bad id, a `PATCH` with
an empty body, and a list with five bad parameters) the count was **0**. After one valid create it was 2
(the owner lookup and the insert). The one valid create had `id`, `userId` and `foo` in its body, and the
stored task had a server-made id and `userId: 1`.

**A 400 with `details` in the exact M05 envelope** (the create with every field wrong; `requestId` shortened):

```json
{"error":{"code":"VALIDATION_ERROR","message":"Request validation failed","details":[{"path":"title","message":"Must not be empty"},{"path":"status","message":"Must be one of: todo, in_progress, done"},{"path":"priority","message":"Must be a whole number from 1 (high) to 3 (low)"},{"path":"dueDate","message":"Must be an ISO 8601 date such as 2026-12-31, or null"}],"requestId":"<id>"}}
```

**The Bruno collection against v3.** The 36 requests I had at the v2 baseline are unchanged and still pass.
I then added 11 for the boundary decisions above (`Errors/25` to `28` and the `Dates/` folder), and ran the
whole collection again.

| | Requests | Tests | Assertions | Status |
| --- | --- | --- | --- | --- |
| v2 baseline (Node 24.22.0) | 36 passed | 49/49 | 76/76 | PASS |
| v3, the original 36 requests | 36 passed | 49/49 | 76/76 | PASS |
| v3, with the 11 boundary requests | 47 passed | 63/63 | 91/91 | PASS |

The new requests assert: hour 24, a non-leap 29 February and hour 25 are each a 400 with
`{ path: "dueDate", message: "Must be an ISO 8601 date such as 2026-12-31, or null" }`; a leap day gives
`2028-02-29T00:00:00.000Z`; `0050-06-15` is stored and read back; `9999-12-31T23:59:59Z` is stored;
`2026-01-01T10:00+08:00` (minutes only, with an offset) is stored as `2026-01-01T02:00:00.000Z`; `null`
clears it; and `11111111-1111-1111-1111-111111111111` is a 404.
The server log for the v3 run had no `unexpected error` line. The collection's `collection.bru` asserts the
envelope (exactly `error`, with `code`, `message`, `details` only on `VALIDATION_ERROR`, and a `requestId`
equal to the `X-Request-Id` header) after every request that returns a 4xx, which is every request in
`Errors/`.

**Fail fast, and the seed.** With no `.env` and the environment set by hand:

| Command | Result |
| --- | --- |
| server, `PORT=abc NODE_ENV=staging API_KEY=" "` | exit 1, one message listing all three: `PORT must be an integer from 1 to 65535, got "abc"`, `NODE_ENV must be one of development, production, test, got "staging"`, `API_KEY is required and must not be empty` |
| server, no `API_KEY` | exit 1 |
| server, no `DATABASE_URL` | exit 1, `DATABASE_URL is required, for example file:./dev.db` |
| `tsx prisma/seed.ts`, only `DATABASE_URL` | `Seeded: users 6, tasks 24, tags 7, task_tags 37`, exit 0 |
| `tsx scripts/check-prisma-errors.ts`, only `DATABASE_URL` | three `PASS`, exit 0 |
| `tsx prisma/seed.ts`, no `DATABASE_URL` | exit 1 |

`npm run db:seed` and `npm run check:errors` (the real entry points, with `.env`) gave the same results.
I did not run `migrate reset`: nothing in this stage needed it.

**Malformed ids are a 400, and well-formed ids with no task are a 404.** `GET /api/v1/tasks/<id>` against
the running server, by `curl`:

| `id` | Status |
| --- | --- |
| `not-a-uuid`, `123` | 400 |
| `11111111-1111-1111-1111-111111111111` (version digit 1, variant digit 1: not an RFC UUID) | 404 |
| `12345678-1234-1234-1234-123456789012` | 404 |
| `AAAAAAAA-AAAA-4AAA-8AAA-AAAAAAAAAAAA` (uppercase) | 404 |
| `11111111-1111-4111-8111-111111111111` (a valid UUID, no such task) | 404 |
| `00000000-0000-4000-8000-000000000007` (a seeded task) | 200 |

With `z.uuid()` the second and third rows would have been 400. Bruno `Errors/08` (400) and `Errors/28` (404)
keep the line in the collection.

**Behaviour against v2.** 345 cases, 4 differences, both decided against the date contract (see above). 37 `config` cases, 0
differences. 12 database cases, accept and reject identical.

**After the merge: the checks re-run on `main`.** I cloned `main` fresh (HEAD `4deec36`) into an empty
directory, copied `.env.example` to `.env` and followed the README, on Node 24.22.0:

| Step | Result |
| --- | --- |
| `npm install`, `npm run db:generate` | clean |
| `npm run db:deploy` | 3 migrations found, all applied |
| `npm run db:seed` | `Seeded: users 6, tasks 24, tags 7, task_tags 37` |
| `npm run typecheck` | exit 0 |
| `npm run check:errors` | 3 `PASS` |
| `grep` for `parseCreateTask`, `parseUpdateTask`, `parseTaskId`, `parseListQuery`, `isTaskStatus` in `src` | 0 matches, and `src/validation/` does not exist |
| `npm start`, then `npm run bruno` | **47 requests passed, tests 63/63, assertions 91/91, PASS** |
| SIGTERM | `Server closed`, and no `unexpected error` line in the server log |

So the merged `main` gives the same result as the branch did before the merge.

## What I'd do differently

- **Write the comparison script before the schemas.** I wrote the schemas, then the script, and the script
  changed the schemas four times. Writing the cases first, from the old validator's branches, would have made
  the schemas a response to a list.
- **Check `node --version` before the baseline, not after the baseline failed.** The `better-sqlite3`
  mismatch cost one confusing run and nothing else, but it only looked like a v2 bug because I had not
  looked at the shell.
- **Decide where the database settings live at the start.** I found the seed problem by reading what imports
  `db.ts`, which was luck. A one-line list of "who imports this file" before moving the settings would have
  been the first step, not the third.
- **Settle the date contract before comparing, not after.** I treated the two date differences as quirks to
  accept, and only read what the contract actually said when I was asked to. The README's "ISO 8601" did not
  decide hour 24, so the decision needed outside sources and a sentence in the README. Writing the accepted
  range down first (forms, hours, years, calendar) would have turned both into test cases from the start,
  and the Bruno requests that pin them would not have been an afterthought.
