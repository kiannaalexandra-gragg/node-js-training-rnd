# M08 — Zod Validation & Type Inference: Training Material

**Week 3 · Days 4-5 · 6 hours**

A "module" in this program means a training stage. When this guide says "ES module" it means a JavaScript
module file. The guide's own steps are called "stages" too, so "the stage" means this module and "Stage 4"
means a step of the guided path below.

## Builds on

M05 (the Task API's error envelope, central error handler and hand-written validation) and M07 (Prisma 7 and
the task service behind it). You change the same `task-api/` project in place: M05's validation is the code
you delete, and M05's envelope is the shape you must not change. Keep your M05 and M07 code and your Bruno
collection on hand.

## 1. What you'll build and why

You will replace every piece of hand-written validation in the Task API with **Zod 4**. You finish with
**Task API v3**:

- **Zod schemas** for the create body, the update body, the list query and the `:id` param,
- a **typed `validate({ params, body, query })` middleware** that runs them before a handler,
- **handler and service input types that come from the schemas** (`z.infer` and `z.output`), with no
  hand-written duplicate left,
- a `ZodError` **mapped into the 400 envelope you fixed in M05**, through the central handler you already have,
- **environment validation** in `config.ts` that stops the server at startup, with every problem listed,
- **no hand-written validation left anywhere**, and a Bruno collection that is still green.

Why this stage matters: TypeScript types are erased when the code runs. A `req.body` typed as
`CreateTaskInput` is still whatever the client sent, and a type cannot refuse it. In M05 you wrote the checks
by hand, and you saw how long they get and how they drift from the types beside them. A schema is a check and
a type at once: one definition says what is accepted, and the compiler derives the type of what comes out.

Zod Mini and JSON Schema export come up as **concepts only**. You do not use either.

### Where the code goes

All code lives in your build repo, in the `task-api/` folder. v3 is a change to that folder, not a new one:

```text
task-api/
├── src/
│   ├── schemas/task.schemas.ts       new: every Zod schema, and the request types derived from them
│   ├── middleware/validate.ts        new: validate({ params, body, query })
│   ├── env.ts                        new: parseEnv, and the database settings
│   ├── config.ts                     changed: the M05 loadConfig becomes a Zod schema
│   ├── db.ts                         changed: reads its settings through env.ts
│   ├── middleware/error-handler.ts   changed: one new branch for ZodError
│   ├── routes/tasks.routes.ts        changed: validate(...) on every route
│   ├── handlers/tasks.handlers.ts    changed: no parsing, typed values from res.locals.valid
│   ├── services/task-service.ts      changed: takes the inferred types
│   ├── types/task.ts                 changed: the hand-written input types are removed
│   └── validation/task-validation.ts deleted
├── bruno/                            changed: requests added for your date and id boundaries (Stage 9)
└── scratch/                          temporary, never committed: your experiments
```

### How to use this guide

- Work the stages in order. Each ends in a **Done when** checklist you can verify yourself.
- A line starting with `$` is typed in a terminal. Everything else is file content or output.
- Commands and output are from a reference build on macOS with Node 24.22.0 and Zod 4.6.7. Your request ids,
  timestamps and counts will differ.
- The reference code is shown where you change it, without each file's header comment. If your M05 and M07
  code differs (file names, how the store is called, what your error classes are named), make the same change
  in your shape.
- The reference messages (`Required`, `Must not be empty`, `Must be one of: todo, in_progress, done`, and so on)
  are the ones the M05 validator and Bruno collection use. **Use your own messages where they differ**: the
  point is to keep what your collection already asserts.
- Imports use `.ts` extensions, as in M04 to M07. The program has not decided this convention (open decision
  P2), so it is this project's choice.

## 2. Before you start

You need:

- **Node.js 24 LTS.** Run `nvm use` on its own line in your repo root, then check `node --version`. Do not pipe
  `nvm use`: a pipe leaves the shell on whatever version it started with. This matters more than usual here:
  `better-sqlite3` is built for the Node version that installed it, and the wrong version breaks every route
  that touches the database (Pitfalls explains how it looks).
- **Your M07 Task API (v2)**, merged, with the Bruno collection green and the seed script working.
- **A branch habit**: a branch named `feat/module-08/<topic>`, Conventional Commits, and a pull request. You do
  not push to `main`.

Pinned versions that matter in this stage:

| Tool | Version |
| --- | --- |
| Node.js | 24 LTS |
| TypeScript | 7.0 |
| Express | 5.x |
| Zod | **4.x**: `npm install zod@4` |
| Prisma ORM | 7.x, always `@7` (`npx prisma@7 ...`). Never an unversioned `prisma`. |
| Database | SQLite |

Zod 4 is not Zod 3. A lot of what you will find online is Zod 3: `z.string().email()`, `z.string().uuid()`,
`message:` options, `error.format()`. Zod 4 has top-level formats (`z.email()`, `z.uuid()`, `z.url()`), one
`error` option for custom messages, and `z.flattenError(error)`. Read the docs under zod.dev for version 4.

Free tools only. No Docker. No paid licences.

## 3. Guided path

### Stage 1 — Branch, baseline, and the inventory

**Goal:** start from an up-to-date `main`, prove the Bruno collection is green **before** you change anything,
and write down every rule the hand-written validation enforces. Those rules are the specification for your
schemas.

*Traces to the brief:* the tasks' Setup (read the brief, the branch → PR → merge habit). The baseline run and the
inventory are this guide's own process, not brief requirements.

1. Make sure M07 is merged, then:

   ```bash
   git switch main
   git pull
   git switch -c feat/module-08/zod-validation
   cd task-api
   nvm use
   node --version
   ```

2. Make sure `.env` exists (copy `.env.example` if not), start the v2 server in a second terminal
   (`npm start`), and run the collection:

   ```bash
   npm run bruno
   ```

   Stop the server when it finishes. Write down the **number of requests, tests and assertions** it printed.
   The reference collection had **36 requests, 49 tests and 76 assertions**. Your counts are your baseline,
   and the stage's rule is that they are the same at the end.

   If many requests return 500, read the server's log before you change anything. If it says
   `ERR_DLOPEN_FAILED` and `NODE_MODULE_VERSION`, your shell is on a different Node version from the one that
   installed `better-sqlite3`. Run `nvm use` again, and check `node --version` prints 24.x. (In the reference
   build, a shell on Node 22.23.0 made 16 of the 36 requests fail before any v3 code existed.)

3. Find every place that validates by hand:

   ```bash
   grep -rnE "if \(!|typeof [a-zA-Z_.]+ [!=]==? '|isNaN|Number\.isInteger|\.trim\(\) [!=]==|\.test\(" src prisma scripts --include='*.ts' --exclude-dir=generated
   ```

   Read the files it points at. In the reference build there were three places: the validation file (252
   lines: `parseCreateTask`, `parseUpdateTask`, `parseTaskId`, `parseListQuery` and their helpers), the
   `loadConfig` function in `config.ts`, and the `DATABASE_URL` check in `db.ts`. There were also hand-written
   input types (`CreateTaskInput`, `UpdateTaskInput`, `ListTasksQuery`) and an `isTaskStatus` guard in
   `types/task.ts`.

4. Write an inventory, in a scratch note, of **every rule and its message**. For each field: is it required,
   what type, what range or format, what is trimmed, what is the default, what is the message. Include what
   the code does with fields it does not know (ignored or rejected), the order the parts of a request are
   checked in, and what the update does with an empty body. You will compare against this in Stage 4.

**Done when:**

- [ ] `git branch --show-current` prints `feat/module-08/zod-validation`.
- [ ] `node --version` prints a 24.x version.
- [ ] `npm run bruno` against the unchanged v2 server prints `PASS`, and you have written down the counts
      (process check, not in the brief).
- [ ] Your inventory lists every rule and message of the validation file, `config.ts` and `db.ts` (process
      check, not in the brief).

### Stage 2 — Install Zod 4 and learn its vocabulary on a scratch file

**Goal:** install Zod 4 and see every feature this stage uses working on small inputs, so the real schemas in
Stage 3 hold no surprises.

*Traces to the brief:* the Setup (`zod@4`) and the Research list (Zod 4 basics, parsing and error formatting,
refinements and transforms, type inference, awareness of Zod Mini and `z.toJSONSchema`).

1. Install and check:

   ```bash
   npm install zod@4
   npm ls zod
   grep '"zod"' package.json
   ```

   `npm ls zod` must show a `4.x` version and `package.json` must show `^4`.

2. Make a scratch folder that is never committed, and a first file:

   ```bash
   mkdir scratch
   ```

   ```ts
   // scratch/tour.ts
   import { z } from 'zod';

   const show = (label: string, r: { success: boolean; data?: unknown; error?: z.ZodError }) =>
     console.log(
       label.padEnd(34),
       r.success
         ? 'OK   ' + JSON.stringify(r.data)
         : 'FAIL ' + JSON.stringify(r.error!.issues.map((i) => `${i.path.join('.') || '(root)'}: ${i.message}`)),
     );
   ```

   Add each block below to the same file, and run `npx tsx scratch/tour.ts`. `safeParse` is explained in
   step 4: for now it is the call that tells you whether the input passed and gives you either the data or the
   problems.

3. **Primitives and string formats.** Zod 4 has the formats at the top level:

   ```ts
   show('z.string() 5', z.string().safeParse(5));
   show('z.int() 1.5', z.int().safeParse(1.5));
   show('z.email() "nope"', z.email().safeParse('nope'));
   show('z.url() "zod.dev"', z.url().safeParse('zod.dev'));
   show('z.uuid() all-1s', z.uuid().safeParse('11111111-1111-1111-1111-111111111111'));
   show('z.guid() all-1s', z.guid().safeParse('11111111-1111-1111-1111-111111111111'));
   ```

   ```text
   z.string() 5                       FAIL ["(root): Invalid input: expected string, received number"]
   z.int() 1.5                        FAIL ["(root): Invalid input: expected int, received number"]
   z.email() "nope"                   FAIL ["(root): Invalid email address"]
   z.url() "zod.dev"                  FAIL ["(root): Invalid URL"]
   z.uuid() all-1s                    FAIL ["(root): Invalid UUID"]
   z.guid() all-1s                    OK   "11111111-1111-1111-1111-111111111111"
   ```

   Notice the last two. `z.uuid()` checks the version and variant digits of a real UUID. `z.guid()` accepts any
   8-4-4-4-12 hex string. You will have to choose between them in Stage 3.

4. **`parse` against `safeParse`, and objects.** `parse` returns the data or **throws** a `ZodError`.
   `safeParse` never throws: it returns `{ success: true, data }` or `{ success: false, error }`.

   ```ts
   const User = z.object({ name: z.string(), age: z.int().optional() });
   show('z.object, extra key', User.safeParse({ name: 'Ada', extra: 1 }));
   show('z.strictObject, extra key', z.strictObject({ name: z.string() }).safeParse({ name: 'Ada', extra: 1 }));
   show('z.object, not an object', User.safeParse([]));
   ```

   ```text
   z.object, extra key                OK   {"name":"Ada"}
   z.strictObject, extra key          FAIL ["(root): Unrecognized key: \"extra\""]
   z.object, not an object            FAIL ["(root): Invalid input: expected object, received array"]
   ```

   `z.object` **strips** keys it does not know: they are in the input and not in the output. `z.strictObject`
   **rejects** them.

5. **Arrays, enums, literals and unions.**

   ```ts
   show('array, bad item', z.array(z.string()).safeParse(['a', 2]));
   show('enum, bad', z.enum(['todo', 'done']).safeParse('later'));
   show('literal, bad', z.literal('v3').safeParse('v4'));
   show('union, true', z.union([z.int(), z.string()]).safeParse(true));
   const Shape = z.discriminatedUnion('kind', [
     z.object({ kind: z.literal('a'), x: z.int() }),
     z.object({ kind: z.literal('b'), y: z.string() }),
   ]);
   show('discriminated, b wrong', Shape.safeParse({ kind: 'b', y: 5 }));
   show('discriminated, unknown', Shape.safeParse({ kind: 'c' }));
   ```

   ```text
   array, bad item                    FAIL ["1: Invalid input: expected string, received number"]
   enum, bad                          FAIL ["(root): Invalid option: expected one of \"todo\"|\"done\""]
   literal, bad                       FAIL ["(root): Invalid input: expected \"v3\""]
   union, true                        FAIL ["(root): Invalid input"]
   discriminated, b wrong             FAIL ["y: Invalid input: expected string, received number"]
   discriminated, unknown             FAIL ["kind: Invalid discriminator value. Expected 'a' | 'b'"]
   ```

   Read the paths: the array error is at `1` (the index), and a discriminated union reports the problem inside
   the branch it picked, where a plain union can only say `Invalid input`.

6. **Optional, nullable, default, and `z.coerce`.**

   ```ts
   const O = z.object({ a: z.string().optional(), b: z.string().nullable(), c: z.string().default('dflt') });
   show('b null, c missing', O.safeParse({ b: null }));
   show('b missing', O.safeParse({}));
   show('coerce "42"', z.coerce.number().safeParse('42'));
   show('coerce "abc"', z.coerce.number().safeParse('abc'));
   show('coerce ""', z.coerce.number().safeParse(''));
   show('coerce "1e2"', z.coerce.number().safeParse('1e2'));
   show('coerce null', z.coerce.number().safeParse(null));
   ```

   ```text
   b null, c missing                  OK   {"b":null,"c":"dflt"}
   b missing                          FAIL ["b: Invalid input: expected string, received undefined"]
   coerce "42"                        OK   42
   coerce "abc"                       FAIL ["(root): Invalid input: expected number, received NaN"]
   coerce ""                          OK   0
   coerce "1e2"                       OK   100
   coerce null                        OK   0
   ```

   `optional()` allows a missing key, `nullable()` allows `null`, `default()` fills in a missing value. And
   `z.coerce.number()` is just `Number(input)` before the check, so `""` and `null` are `0` and `"1e2"` is
   100. Keep that in mind for the query string.

7. **Refinements, transforms and pipes.**

   ```ts
   const Range = z
     .object({ from: z.int(), to: z.int() })
     .refine((r) => r.from <= r.to, { error: 'from must not be after to', path: ['to'] });
   show('refine, cross-field', Range.safeParse({ from: 3, to: 2 }));
   show('refine, type error first', Range.safeParse({ from: 'x', to: 2 }));
   show('trim + lowercase', z.string().trim().toLowerCase().safeParse('  HeLLo '));
   show('pipe, 2nd stage fails', z.string().transform(Number).pipe(z.int()).safeParse('7.5'));
   ```

   ```text
   refine, cross-field                FAIL ["to: from must not be after to"]
   refine, type error first           FAIL ["from: Invalid input: expected number, received string"]
   trim + lowercase                   OK   "hello"
   pipe, 2nd stage fails              FAIL ["(root): Invalid input: expected int, received number"]
   ```

   A `refine` is a custom rule, and it only runs when the value already passed its type checks (the second
   line). `transform` changes the value. `pipe` feeds the output of one schema into another.

8. **Deriving schemas, and custom messages.**

   ```ts
   const Task = z.object({ id: z.string(), title: z.string(), done: z.boolean() });
   const Create = Task.omit({ id: true });
   const Update = Create.partial();
   const Summary = Task.pick({ id: true, title: true });
   const Extended = Task.extend({ tags: z.array(z.string()) });
   console.log(Object.keys(Create.shape), Object.keys(Update.shape), Object.keys(Extended.shape));
   show('Update, empty object', Update.safeParse({}));

   const title = z.string({ error: (i) => (i.input === undefined ? 'Required' : 'Must be a string') });
   show('error fn, missing', z.object({ t: title }).safeParse({}));
   show('error fn, wrong type', z.object({ t: title }).safeParse({ t: 5 }));
   show('check message', z.string().min(3, { error: 'Too short' }).safeParse('ab'));
   ```

   ```text
   [ 'title', 'done' ] [ 'title', 'done' ] [ 'id', 'title', 'done', 'tags' ]
   Update, empty object               OK   {}
   error fn, missing                  FAIL ["t: Required"]
   error fn, wrong type               FAIL ["t: Must be a string"]
   check message                      FAIL ["(root): Too short"]
   ```

   `omit`, `pick`, `partial` and `extend` make a new schema from an old one, so a new field is one edit.
   Custom messages go in the `error` option: either a string, or a function that receives the issue (here it
   reads `issue.input`, which is `undefined` when the key was not sent).

9. **Reading errors.** A `ZodError` holds a list called `issues`. Three helpers format it:

   ```ts
   const Body = z.object({
     title: z.string().trim().min(1, { error: 'Must not be empty' }),
     priority: z.int({ error: 'Must be 1 to 3' }).min(1).max(3),
   });
   const r = Body.safeParse({ title: '  ', priority: 4.5 });
   if (!r.success) {
     console.log(JSON.stringify(z.flattenError(r.error)));
     console.log(JSON.stringify(z.treeifyError(r.error)));
     console.log(z.prettifyError(r.error));
   }
   ```

   ```text
   {"formErrors":[],"fieldErrors":{"title":["Must not be empty"],"priority":["Must be 1 to 3"]}}
   {"errors":[],"properties":{"title":{"errors":["Must not be empty"]},"priority":{"errors":["Must be 1 to 3"]}}}
   ✖ Must not be empty
     → at title
   ✖ Must be 1 to 3
     → at priority
   ```

   `flattenError` is one level (good for flat forms), `treeifyError` follows the data's shape (good for
   nested data), and `prettifyError` is text for a person. None of them is the `{ path, message }` list your
   envelope needs: Stage 6 builds that from `error.issues`.

10. **Keep `scratch/` out of git.** `git status` shows it as untracked: never `git add` it, and delete the
    folder before Stage 10.

**Done when:**

- [ ] `npm ls zod` shows a `4.x` version, and `package.json` shows `^4`.
- [ ] `npx tsx scratch/tour.ts` runs and prints a line for every block above.
- [ ] You can explain, in your own words, what `z.object` does with an unknown key and what `z.strictObject`
      does.
- [ ] You can explain the difference between `parse` and `safeParse`.
- [ ] You can say what `z.coerce.number()` returns for `""` and for `"1e2"`, and why that matters for a query
      string.

### Stage 3 — Write the schemas: create, update, params, list query

**Goal:** one file, `src/schemas/task.schemas.ts`, that holds every rule from your inventory as a schema, and
exports the types derived from them.

*Traces to the brief:* Lab step 1 (schemas for create, update, list query and params) and the scope's type
inference item (`z.infer`, `.pick`, `.omit`, `.partial`, `.extend`).

Build it in pieces. The final file is at the end of this stage.

1. **Start with the shared pieces.** Take the status values from the generated Prisma enum, so there is one
   list of them, not two:

   ```ts
   import { z } from 'zod';
   import { TaskStatus } from '../generated/prisma/enums.ts';

   export const TASK_STATUSES = Object.values(TaskStatus);
   const STATUS_MESSAGE = `Must be one of: ${TASK_STATUSES.join(', ')}`;
   const PRIORITY_MESSAGE = 'Must be a whole number from 1 (high) to 3 (low)';

   export const SORT_FIELDS = ['createdAt', 'title', 'priority', 'dueDate', 'status'] as const;
   export const SORT_ORDERS = ['asc', 'desc'] as const;
   export const DEFAULT_PAGE_SIZE = 20;
   export const MAX_PAGE_SIZE = 100;

   const orRequired = (typeMessage: string) => (issue: { input?: unknown }) =>
     issue.input === undefined ? 'Required' : typeMessage;
   ```

   `orRequired` is the function from Stage 2, step 8: a missing key says `Required`, and a key that is there
   with the wrong type says the type message. Copy the constants (`SORT_FIELDS`, `DEFAULT_PAGE_SIZE`, and so
   on) from `types/task.ts` into this file, because the schemas are now what defines them. Leave the originals
   where they are for now, since the old validator still imports them: you remove them from `types/task.ts`
   in Stage 7.

2. **The create body.** Every field, in the order your validator reported them, with the messages your
   collection asserts:

   ```ts
   const DUE_DATE_MESSAGE = 'Must be an ISO 8601 date such as 2026-12-31, or null';
   const dueDate = z
     .union(
       [
         z.iso.date(),
         z.iso.datetime({ offset: true }),
         z.iso.datetime({ offset: true, precision: -1 }),
       ],
       { error: DUE_DATE_MESSAGE },
     )
     .transform((text) => new Date(text))
     .nullable();

   export const createTaskBody = z.object(
     {
       title: z
         .string({ error: orRequired('Must be a string') })
         .trim()
         .min(1, { error: 'Must not be empty' }),
       description: z.string({ error: 'Must be a string' }).optional(),
       status: z.enum(TaskStatus, { error: STATUS_MESSAGE }).optional(),
       priority: z
         .int({ error: PRIORITY_MESSAGE })
         .min(1, { error: PRIORITY_MESSAGE })
         .max(3, { error: PRIORITY_MESSAGE })
         .optional(),
       dueDate: dueDate.optional(),
     },
     { error: 'Must be a JSON object' },
   );
   ```

   Why each choice is made this way:

   - `z.object` (strip), because the M05 validator ignored fields it did not know, and an `id` or `userId` in
     a body must never reach Prisma. It does not reach it here either: it is dropped.
   - `.trim().min(1)` in that order: the trim runs first, so `"   "` is empty.
   - The `{ error: 'Must be a JSON object' }` on the object itself is the message when the body is not an
     object at all (no body, an array, a string). It has no field, so its path is empty.
   - `dueDate` is a text date or date-time, and `null` clears it. The union has three members because Zod
     draws a line the validator did not: `z.iso.datetime` wants seconds, and `precision: -1` is the form with
     minutes only (`2026-01-01T10:00Z`). `z.iso.date` checks the day against the calendar, so `2026-02-30` is
     refused instead of rolling over to 2 March the way `new Date('2026-02-30')` does. `.transform` turns the
     text into a `Date`, because that is what Prisma wants.
   - `.optional()` goes **after** the `.transform` and `.nullable()`: it says the key may be missing, and
     `null` is a different thing (it clears the date).

3. **The update body, derived.**

   ```ts
   export const updateTaskBody = createTaskBody
     .partial()
     .refine((patch) => Object.keys(patch).length > 0, {
       error: 'At least one of title, description, status is required',
     });
   ```

   `.partial()` makes every field optional. The `refine` is the "an empty PATCH is a mistake" rule: a key that
   was not sent is **not in the output at all**, so an empty patch is an empty object. Because the refine only
   runs when the fields passed, `{ "title": "" }` reports the title problem and not also "at least one".

4. **The params.**

   ```ts
   export const taskParams = z.object({ id: z.guid({ error: 'Must be a valid UUID' }) });
   ```

   `z.guid()`, not `z.uuid()`: your M05 validator accepted any 8-4-4-4-12 hex string, and `z.uuid()` also
   refuses ids with an invalid version digit (Stage 4 shows what that changes). See **A schema is a statement of what you accept**
   in Concepts for how to choose.

5. **The list query.** Everything in a query string is text, so a number needs more than `z.number()`:

   ```ts
   function wholeNumber(max?: number) {
     const message =
       max === undefined ? 'Must be a whole number of 1 or more' : `Must be a whole number from 1 to ${max}`;
     let size = z.int({ error: message }).min(1, { error: message });
     if (max !== undefined) size = size.max(max, { error: message });
     return z
       .string({ error: message })
       .regex(/^\d+$/, { error: message })
       .transform(Number)
       .pipe(size);
   }

   const choice = <T extends readonly [string, ...string[]]>(values: T) =>
     z.enum(values, { error: `Must be one of: ${values.join(', ')}` });

   export const listTasksQuery = z.object({
     status: z.enum(TaskStatus, { error: STATUS_MESSAGE }).optional(),
     q: z
       .string({ error: 'Must be a non-empty string' })
       .trim()
       .min(1, { error: 'Must be a non-empty string' })
       .optional(),
     sortBy: choice(SORT_FIELDS).default('createdAt'),
     order: choice(SORT_ORDERS).default('desc'),
     page: wholeNumber().default(1),
     pageSize: wholeNumber(MAX_PAGE_SIZE).default(DEFAULT_PAGE_SIZE),
   });
   ```

   Why `wholeNumber` is a string, a regex, `Number` and then an integer check, instead of
   `z.coerce.number().int().min(1)`: Stage 2 showed that `z.coerce.number()` reads `"1e2"` as 100 and `""` as
   0. A query string of `?page=1e2` is a typo, and your M05 validator rejected it. The digits check says
   "only digits", and then the number check does the range. Why `.transform(Number)` and not
   `z.coerce.number()` after the regex: a coerced number's input type is `unknown`, and `.pipe` from a string
   will not type-check (Pitfalls shows the error).

   A key that appears twice (`?status=a&status=b`) arrives as an array, and an array is not a string or an
   enum value, so it fails the schema. `.default()` fills in a missing key, and the **output** has a value for
   every key with a default.

6. **One request object per route, and the derived types.** The route and the handler will both refer to the
   same object, so a handler's type cannot name a different schema from the one its route was given:

   ```ts
   export const listTasksRequest = { query: listTasksQuery };
   export const createTaskRequest = { body: createTaskBody };
   export const getTaskRequest = { params: taskParams };
   export const updateTaskRequest = { params: taskParams, body: updateTaskBody };
   export const deleteTaskRequest = { params: taskParams };

   export type CreateTaskInput = z.output<typeof createTaskBody>;
   export type UpdateTaskInput = z.output<typeof updateTaskBody>;
   export type ListTasksQuery = z.output<typeof listTasksQuery>;
   export type SortField = (typeof SORT_FIELDS)[number];
   export type SortOrder = (typeof SORT_ORDERS)[number];
   ```

   These types are **not written by hand**: `z.output<typeof createTaskBody>` is whatever the schema produces.
   See **`z.input` and `z.output`** in Concepts for why `output` is the one a handler uses.

7. `npm run typecheck` must still exit 0: the old validator and the old types are still in place, and nothing
   uses the new file yet.

**Done when:**

- [ ] `src/schemas/task.schemas.ts` exists and exports `createTaskBody`, `updateTaskBody`, `taskParams` and
      `listTasksQuery` (the create, update, params and list-query schemas the brief requires).
- [ ] `npm run typecheck` exits 0.
- [ ] `CreateTaskInput`, `UpdateTaskInput` and `ListTasksQuery` in that file are `z.output<...>` types, not
      object types you wrote.
- [ ] You can explain why the update schema is built from the create schema, and why it is `.partial()` plus
      a `refine` and not a second object written out.

### Stage 4 — Prove the schemas behave like the validator they replace

**Goal:** before you delete anything, run the old validator and the new schemas on the same inputs, and find
every input they answer differently.

*Traces to the brief:* the Definition of done's "invalid input never reaches Prisma: a Zod schema rejects it
first, every time" and "400 responses carry field-level `details`", and Lab step 3 (remove every hand-written
check, not alongside). The comparison is how you show the schemas reject what the old checks rejected before you
delete them. The brief does not name the method.

Why now: the brief's rule is that the hand-written code must **disappear**, not sit beside the Zod layer. You
cannot compare once it is gone. And "the messages match" is not "the behaviour matches": the Bruno collection
covers the inputs you thought of, and a schema can differ on the ones you did not.

1. Write the comparison in your scratch folder:

   ```ts
   // scratch/compare.ts
   import type { z } from 'zod';
   import {
     parseCreateTask,
     parseListQuery,
     parseTaskId,
     parseUpdateTask,
   } from '../src/validation/task-validation.ts';
   import {
     createTaskBody,
     listTasksQuery,
     taskParams,
     updateTaskBody,
   } from '../src/schemas/task.schemas.ts';

   type Detail = { path: string; message: string };
   type Outcome = { ok: true; value: unknown } | { ok: false; details: Detail[] };

   // Dates become text, so two equal dates compare equal.
   const norm = (v: unknown): unknown =>
     JSON.parse(JSON.stringify(v, (_k, x) => (x instanceof Date ? `Date(${x.toISOString()})` : x)));

   // Key order does not matter, so a canonical form with the keys sorted is compared.
   const canon = (v: unknown): string =>
     JSON.stringify(v, (_k, x) =>
       x && typeof x === 'object' && !Array.isArray(x)
         ? Object.fromEntries(Object.entries(x).sort(([a], [b]) => a.localeCompare(b)))
         : x,
     );

   function oldRun(fn: () => unknown): Outcome {
     try {
       return { ok: true, value: norm(fn()) };
     } catch (e) {
       return { ok: false, details: (e as { details: Detail[] }).details };
     }
   }

   // The mapping Stage 6 builds into the error handler: one entry per path, an empty path is the source.
   function newRun(schema: z.ZodType, input: unknown, root: string): Outcome {
     const r = schema.safeParse(input);
     if (r.success) return { ok: true, value: norm(r.data) };
     const seen = new Set<string>();
     const details: Detail[] = [];
     for (const issue of r.error.issues) {
       const path = issue.path.length === 0 ? root : issue.path.join('.');
       if (seen.has(path)) continue;
       seen.add(path);
       details.push({ path, message: issue.message });
     }
     return { ok: false, details };
   }

   let total = 0;
   let diffs = 0;
   function compare(label: string, o: Outcome, n: Outcome): void {
     total++;
     if (canon(o) === canon(n)) return;
     diffs++;
     console.log(`DIFF ${label}\n   v2: ${JSON.stringify(o)}\n   v3: ${JSON.stringify(n)}`);
   }

   const bodies: [string, unknown][] = [
     ['no body', undefined], ['null', null], ['array', []], ['string', 'x'], ['empty object', {}],
     ['title ok', { title: 'Buy milk' }], ['title padded', { title: '  Buy milk  ' }],
     ['title empty', { title: '' }], ['title blanks', { title: '   ' }], ['title number', { title: 5 }],
     ['title null', { title: null }], ['description null', { title: 't', description: null }],
     ['status bad', { title: 't', status: 'nope' }], ['status upper', { title: 't', status: 'TODO' }],
     ['priority 0', { title: 't', priority: 0 }], ['priority 4', { title: 't', priority: 4 }],
     ['priority 1.5', { title: 't', priority: 1.5 }], ['priority "2"', { title: 't', priority: '2' }],
     ['priority 1e300', { title: 't', priority: 1e300 }],
     ['dueDate date', { title: 't', dueDate: '2026-12-31' }],
     ['dueDate 30 Feb', { title: 't', dueDate: '2026-02-30' }],
     ['dueDate minutes only', { title: 't', dueDate: '2026-01-01T10:00Z' }],
     ['dueDate no zone', { title: 't', dueDate: '2026-01-01T10:00:00' }],
     ['dueDate offset', { title: 't', dueDate: '2026-01-01T10:00:00+08:00' }],
     ['dueDate hour 24', { title: 't', dueDate: '2026-01-01T24:00:00Z' }],
     ['dueDate year 0050', { title: 't', dueDate: '0050-06-15' }],
     ['dueDate null', { title: 't', dueDate: null }], ['dueDate number', { title: 't', dueDate: 5 }],
     ['unknown keys', { title: 't', id: 'x', userId: 9, foo: 1 }],
     ['everything wrong', { title: 5, description: 5, status: 'x', priority: 'x', dueDate: 'x' }],
   ];
   for (const [label, body] of bodies) {
     compare(`[create] ${label}`, oldRun(() => parseCreateTask(body)), newRun(createTaskBody, body, 'body'));
     compare(`[update] ${label}`, oldRun(() => parseUpdateTask(body)), newRun(updateTaskBody, body, 'body'));
   }
   compare('[update] only unknown keys', oldRun(() => parseUpdateTask({ foo: 1 })), newRun(updateTaskBody, { foo: 1 }, 'body'));
   compare('[update] dueDate null only', oldRun(() => parseUpdateTask({ dueDate: null })), newRun(updateTaskBody, { dueDate: null }, 'body'));

   const ids = [
     '00000000-0000-4000-8000-000000000007', 'AAAAAAAA-AAAA-4AAA-8AAA-AAAAAAAAAAAA',
     '11111111-1111-1111-1111-111111111111', '12345678-1234-1234-1234-123456789012',
     'not-a-uuid', '', '123',
   ];
   for (const id of ids) {
     const n = newRun(taskParams, { id }, 'params');
     compare(`[params] ${JSON.stringify(id)}`, oldRun(() => parseTaskId(id)), n.ok ? { ok: true, value: id } : n);
   }

   const queries: [string, unknown][] = [
     ['empty', {}], ['unknown keys', { foo: 'bar' }], ['status ok', { status: 'todo' }],
     ['status empty', { status: '' }], ['status twice', { status: ['todo', 'done'] }],
     ['q padded', { q: '  fix  ' }], ['q blanks', { q: '   ' }], ['q empty', { q: '' }],
     ['sortBy bad', { sortBy: 'userId' }], ['order upper', { order: 'ASC' }],
     ...['1', '007', '0', '-1', '+5', '1.5', '1.0', '1e2', ' 5', '5 ', '', '0x10', '9007199254740993'].map(
       (p): [string, unknown] => [`page ${JSON.stringify(p)}`, { page: p }],
     ),
     ...['100', '101', '1e2', ''].map((p): [string, unknown] => [`pageSize ${JSON.stringify(p)}`, { pageSize: p }]),
     ['page twice', { page: ['1', '2'] }],
     ['everything wrong', { status: 'x', q: ' ', sortBy: 'x', order: 'x', page: '0', pageSize: '101' }],
   ];
   for (const [label, q] of queries) {
     compare(`[query] ${label}`, oldRun(() => parseListQuery(q)), newRun(listTasksQuery, q, 'query'));
   }

   console.log(`\n${total} cases, ${diffs} differences`);
   ```

2. Run it from `task-api/`:

   ```bash
   npx tsx scratch/compare.ts
   ```

   It prints a `DIFF` block for each input the two answer differently, and a count. Add cases until you stop
   thinking of new ones: dates are the richest source (leap days, other zones, hour 24, very early years),
   then numbers at their limits. The reference run used 345 cases.

3. **For each difference, decide: a bug in the schema, or a difference you accept.** A bug is fixed in the
   schema and the script rerun. An accepted difference is written down with its reason. These are the ones the
   reference run found, so you know what they look like:

   - **A schema bug, fixed.** `2026-01-01T10:00Z` (minutes only) was accepted by the old validator and rejected
     by `z.iso.datetime({ offset: true })`. The fix is the third member of the `dueDate` union
     (`precision: -1`). If you still have two members, this is your `DIFF`.
   - **A trap you avoided.** Swap `z.guid()` for `z.uuid()` in `taskParams` and rerun: ids such as
     `11111111-1111-1111-1111-111111111111` flip from "valid" to "Must be a valid UUID", so a request that was
     a 404 becomes a 400. Swap `page` for `z.coerce.number().int().min(1).default(1)` and you get a `DIFF` for most bad pages
     (Zod's default messages are not yours), and among them six that flip from rejected to **accepted**:
     `"+5"`, `"1.0"`, `"1e2"`, `" 5"`, `"5 "` and `"0x10"`. Put them back.
   - **Differences to decide.** Two inputs still differ, and both are in `dueDate`: `2026-01-01T24:00:00Z`
     (hour 24) was accepted by the old validator and parsed as 2 January, and Zod rejects it. And a year before
     0100, such as `0050-06-15`, was rejected by the old validator (its calendar check used
     `Date.UTC(year, ...)`, which reads a year from 0 to 99 as 1900 to 1999) and Zod accepts it.

   **How to decide a difference: against your documented contract, not against the old code.** Find what your
   README (or your M05 and M07 documentation) says the field accepts. If it says something explicit, follow
   it, even when that means making the schema accept or refuse something the Zod default does not. If it is
   silent, decide on the merits and then **write the answer into the contract**, so it stops being silent. The
   reference README said only "ISO 8601: a date or a date-time, and the day must exist". Both differences
   were therefore decisions, and they went like this:

   - **Hour 24 is rejected.** ISO 8601's older editions allow `24:00:00` for midnight only, later guidance
     discourages it, and RFC 3339 (the profile most tools use) has hours 00 to 23. The old validator also stored
     the *next day*, a surprising result for a due date. The README now says "the hour is 00 to 23". A client
     that sent hour 24 gets a 400, and the README says that too.
   - **Years 0000 to 0099 are accepted.** The contract says the day must exist, and `0050-06-15` does. The old
     validator refused it by accident (the `Date.UTC` reading above). ISO 8601's year has four digits,
     `0000` to `9999`, and the README now says so.

   One boundary the date schemas do **not** settle: a zone offset moves the instant, so a date at the edge of
   the year range can leave it. In the reference build `0000-01-01T00:00:00+23:59` was accepted and sent back as
   `-000001-12-31T00:01:00.000Z` (a signed six-digit year), and `9999-12-31T23:59:59-23:59` as
   `+010000-01-01T23:58:59.000Z`, which the old validator did too. The reference build changed nothing for it and
   wrote it in the README as an open question about the contract. Decide yours, and write it down.

   If your contract had promised hour 24, you would put it back, with a Zod rule and not a hand-written
   parser, and the Bruno request in Stage 9 would assert it. Your list of differences will depend on what your
   M05 validator did and what your README says. Whatever you decide, it goes in your README in Stage 10, and
   Stage 9 pins it in Bruno.

4. Keep `scratch/compare.ts` until Stage 7. The old validator has to exist for it to run.

**Done when:**

- [ ] `npx tsx scratch/compare.ts` runs and prints a `cases, N differences` line.
- [ ] Every `DIFF` is either fixed in the schema or decided against your documented contract and written down
      with a reason.
- [ ] You can explain why `z.guid()` and the digits-then-number check were chosen over `z.uuid()` and
      `z.coerce.number()`.

### Stage 5 — The typed `validate` middleware

**Goal:** a function that takes the schemas for a route, runs them before the handler, and gives the handler
typed results, working around Express 5's read-only `req.query`.

*Traces to the brief:* Lab step 2 (build the typed `validate` middleware, working around `req.query` being
read-only) and the scope's "Zod + Express" item.

1. Create `src/middleware/validate.ts`:

   ```ts
   import type { Request, RequestHandler, Response } from 'express';
   import type { z } from 'zod';

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

2. Read it for the reasons behind each choice:

   - **Why the results go on `res.locals`, not back on the request.** In Express 5, `req.query` is a getter,
     so `req.query = parsed` does not work. `res.locals` is the object Express gives every request for data
     that should live as long as the request does. `req.body` and `req.params` are left as the client sent
     them, so the only place a handler gets validated data from is one place.
   - **Why the types are generic.** `Valid<S>` maps each schema you pass to its `z.output`, under the same
     name. If a route passes `{ params, body }`, the handler gets `valid.params` and `valid.body`, and
     `valid.query` does not exist.
   - **Why the parts are checked in the order `params`, `body`, `query`, and the first failing part stops.**
     That is the order a v2 handler checked them in: a request with a bad id **and** a bad body reported only
     the id. If your M05 handlers checked in a different order, put the parts in that order. Inside a part,
     every bad field is still reported, because one `safeParse` reports all of them.
   - **Why a failure is passed to `next(result.error)` and not turned into a response here.** The central error
     handler is the one place that builds an error response. `validate` hands it the `ZodError`, like any other
     error.
   - **Why `OnlyKnownParts`.** The loop only visits `params`, `body` and `query`. Without this type, a typo
     such as `validate({ qurey: schema })` compiles, `Valid<S>` types `valid.qurey` as parsed data, and the
     schema is never run. With it, the typo is a compile error.
   - **Why `validOf`, and what the types cannot do.** The handler's `res` type names a request schema object,
     which stops a handler from being typed against a schema other than the one its route got. It cannot stop a
     route from leaving `validate` out: Express accepts a handler mounted on its own. So handlers read
     through `validOf`, which throws a clear error if `res.locals.valid` was never set. The client gets a
     generic 500 and your log says exactly what is wrong, and unvalidated input is never passed on quietly.

3. `npm run typecheck` must exit 0. Then check that the typo guard works, with a throwaway file:

   ```ts
   // src/_scratch.ts: delete it as soon as you have run the check
   import express from 'express';
   import { z } from 'zod';
   import { validate } from './middleware/validate.ts';
   import { getTaskRequest } from './schemas/task.schemas.ts';

   const r = express.Router();
   r.get('/ok', validate(getTaskRequest));
   // @ts-expect-error qurey is not params, body or query
   r.get('/bad', validate({ qurey: z.object({}) }));
   ```

   `npx tsc --noEmit` must exit 0: it means the line below the `@ts-expect-error` really was an error (an
   unused `@ts-expect-error` is itself an error). Delete `src/_scratch.ts`. The runtime half is seen in
   Stage 7, once there are handlers to mount.

**Done when:**

- [ ] `src/middleware/validate.ts` exists and exports `validate`, `Valid`, `ValidLocals` and `validOf`.
- [ ] `npm run typecheck` exits 0.
- [ ] The throwaway file compiled with the typo line refused, and you deleted it (process check, not in the
      brief).
- [ ] You can explain why the parsed query is on `res.locals` and not on `req.query`.
- [ ] You can explain why a request with a bad id and a bad body reports only the id.

### Stage 6 — Map `ZodError` into the M05 envelope

**Goal:** a `ZodError` reaching your central error handler comes out as the same 400 `VALIDATION_ERROR`
envelope as before, with one `{ path, message }` in `details` per field.

*Traces to the brief:* Lab step 4 (map `ZodError` output into the M05 envelope's `details` array) and the
Definition of done's envelope item.

1. In `src/middleware/error-handler.ts`, import `ZodError` from `zod`, and `ValidationError` and `ErrorDetail`
   from your errors file (use your own class names). Add the mapping function above `toAppError`:

   ```ts
   function zodErrorToDetails(err: ZodError): ErrorDetail[] {
     const seen = new Set<string>();
     const details: ErrorDetail[] = [];
     for (const issue of err.issues) {
       const path = issue.path.length === 0 ? 'body' : issue.path.join('.');
       if (seen.has(path)) continue;
       seen.add(path);
       details.push({ path, message: issue.message });
     }
     return details;
   }
   ```

2. Add one branch to `toAppError`, before the Prisma and body-parser branches:

   ```ts
   if (err instanceof AppError) return err;
   if (err instanceof ZodError) return new ValidationError(zodErrorToDetails(err));
   ```

   The rest of the handler is unchanged: it already builds
   `{ error: { code, message, details?, requestId } }` from an error that carries `details`.

3. Why the mapper is written this way:

   - **`issue.path.join('.')`.** A path is an array of keys: `['title']`, or `['tags', 1]` for an array item.
     The envelope has always had dotted text.
   - **An empty path becomes `body`.** An issue about the value as a whole (not an object, or "at least one
     field") has no field to name. v2 called it `body`, so `body` stays. Only the body can produce one, because
     Express always gives `req.params` and `req.query` as objects.
   - **One entry per path.** Zod can report the same field twice. `priority: 1e300` fails `z.int()` (it is above
     the safe-integer limit) **and** `.max(3)`, and both are `too_big` at the path `priority`. The first
     message wins, so the client gets one problem per field. Delete the `seen` check and rerun Stage 4's script
     to see it: in the reference run it was 9 differences without it and 4 with it.
   - **The order of `details`** follows the order of the schema's keys, which is why Stage 3 wrote the create
     fields in the order your validator reported them.

4. `npm run typecheck` must exit 0. You will see this branch work in Stage 7.

**Done when:**

- [ ] `npm run typecheck` exits 0.
- [ ] `error-handler.ts` has one `instanceof ZodError` branch that returns your validation error, and the
      envelope-building code at the bottom of the file is unchanged.
- [ ] You can explain why `details` is built from `error.issues` and not from `z.flattenError(error)`.

### Stage 7 — Wire the routes, derive the types, align with Prisma, delete the old code

**Goal:** every route runs `validate`, every handler reads typed values, and the hand-written validation and
the hand-written input types are **gone**, not left beside the new ones.

*Traces to the brief:* Lab step 3 (remove every hand-written check as you wire Zod in), the scope's "Zod +
Prisma" item (never pass raw `req.body` to Prisma) and the deliverable's "handler types proven to come from
`z.infer`".

1. **Routes.** Add `validate(...)` before each handler, passing the request object from Stage 3:

   ```ts
   tasksRouter.get('/', validate(listTasksRequest), listTasks);
   tasksRouter.post('/', validate(createTaskRequest), createTask);
   tasksRouter.get('/:id', validate(getTaskRequest), getTask);
   tasksRouter.patch('/:id', validate(updateTaskRequest), updateTask);
   tasksRouter.delete('/:id', validate(deleteTaskRequest), deleteTask);
   ```

   with these imports at the top:

   ```ts
   import { validate } from '../middleware/validate.ts';
   import {
     createTaskRequest,
     deleteTaskRequest,
     getTaskRequest,
     listTasksRequest,
     updateTaskRequest,
   } from '../schemas/task.schemas.ts';
   ```

2. **Handlers.** Replace the parsing with a read through `validOf(res)`. The type of `res` comes from the same
   object the route passed to `validate`:

   ```ts
   import type { Request, Response } from 'express';
   import { validOf, type ValidLocals } from '../middleware/validate.ts';
   import type {
     createTaskRequest,
     deleteTaskRequest,
     getTaskRequest,
     listTasksRequest,
     updateTaskRequest,
   } from '../schemas/task.schemas.ts';
   import { taskService } from '../services/task-service.ts';
   import type { Task, TaskPage } from '../types/task.ts';

   export async function listTasks(
     _req: Request,
     res: Response<TaskPage, ValidLocals<typeof listTasksRequest>>,
   ): Promise<void> {
     res.status(200).json(await taskService.list(validOf(res).query));
   }

   export async function createTask(
     _req: Request,
     res: Response<Task, ValidLocals<typeof createTaskRequest>>,
   ): Promise<void> {
     res.status(201).json(await taskService.create(validOf(res).body));
   }

   export async function getTask(
     _req: Request,
     res: Response<Task, ValidLocals<typeof getTaskRequest>>,
   ): Promise<void> {
     res.status(200).json(await taskService.get(validOf(res).params.id));
   }

   export async function updateTask(
     _req: Request,
     res: Response<Task, ValidLocals<typeof updateTaskRequest>>,
   ): Promise<void> {
     const { params, body } = validOf(res);
     res.status(200).json(await taskService.update(params.id, body));
   }

   export async function deleteTask(
     _req: Request,
     res: Response<never, ValidLocals<typeof deleteTaskRequest>>,
   ): Promise<void> {
     await taskService.remove(validOf(res).params.id);
     res.sendStatus(204);
   }
   ```

   The handlers no longer name `req.body`, `req.params` or `req.query` at all, and the `unknown` annotations
   from M05 are gone. You did not have to give the router any generics: `get`, `post` and the others infer the
   `Locals` type from `validate(...)` and the handler.

   Notice what the compiler does **not** check: a handler mounted with no `validate` in front of it still
   compiles. In the reference build, a scratch route `app.get('/x', listTasks)` returned a 500 with the generic
   `INTERNAL_ERROR` body, and the server log said `res.locals.valid is missing: this handler is mounted
   without validate()`. That is `validOf` doing its job.

3. **The types and the service.** Reduce `types/task.ts` to what is not derived from a schema. The input
   types, the constants and the `SortField` and `SortOrder` aliases now live in
   `src/schemas/task.schemas.ts`, so delete them here along with `isTaskStatus` and `TASK_STATUSES`. Keep
   the Prisma-derived `Task` and the page types. Two details compile only one way: the `TaskStatus` import
   becomes `import type`, because nothing here uses it as a value any more (`verbatimModuleSyntax` rejects a
   value import that is only re-exported as a type), and `export type { TaskStatus }` stays if your seed script
   imports it from here. The reference result:

   ```ts
   import type { Task as TaskRow } from '../generated/prisma/client.ts';
   import type { TaskStatus } from '../generated/prisma/enums.ts';

   export type { TaskStatus };

   export type Task = TaskRow;

   export type PageMeta = {
     page: number;
     pageSize: number;
     // Tasks matching the filters, across all pages.
     total: number;
     totalPages: number;
   };

   export type TaskPage = { data: Task[]; meta: PageMeta };
   ```

   In `task-service.ts`, import the three input types from the schemas file and the rest from `types/task.ts`:

   ```ts
   import type { CreateTaskInput, ListTasksQuery, UpdateTaskInput } from '../schemas/task.schemas.ts';
   import type { Task, TaskPage } from '../types/task.ts';
   ```

   Then make the compiler check Zod against Prisma:

   ```ts
   const data = { ...input, userId: owner.id } satisfies Prisma.TaskUncheckedCreateInput;
   return prisma.task.create({ data });
   ```

   ```ts
   const data = patch satisfies Prisma.TaskUpdateInput;
   return prisma.task.update({ where: { id }, data });
   ```

   `satisfies` checks that the value fits the Prisma type without changing the value's own type. Why
   `TaskUncheckedCreateInput`: it is the variant that takes `userId` directly, which is how your service sets
   the owner. The "checked" `TaskCreateInput` wants an `owner` relation object instead. What this does and does
   not catch is in **Zod is the request contract, Prisma is the persistence contract** in Concepts: read it
   before you rely on it. And raw `req.body` is never passed on: the service receives what the create schema
   produced, with unknown keys already dropped.

4. **Delete the old validator.** Delete `src/validation/task-validation.ts` (and the folder, if it is empty).
   Then:

   ```bash
   npm run typecheck
   ```

   Anything that still imported the old file or the removed types shows up as an error. Fix each one by
   importing from the schemas. (`scratch/compare.ts` will now fail to compile: it has done its job, so delete
   it.)

5. **Run the collection.** Start the server (`npm start`), then:

   ```bash
   npm run bruno
   ```

   It must print `PASS` with the **same counts** you wrote down in Stage 1. If a request fails, the response
   tells you which message or status changed: fix the schema (or the envelope mapping), not the Bruno
   assertion, unless the contract itself changed on purpose and you wrote that down in Stage 4.

**Done when:**

- [ ] `npm run typecheck` exits 0, and `src/validation/` no longer exists.
- [ ] `grep -rnE "parseCreateTask|parseUpdateTask|parseTaskId|parseListQuery|isTaskStatus" src` finds nothing.
- [ ] `grep -rnE "(type|const) (CreateTaskInput|UpdateTaskInput|ListTasksQuery)" src` finds only the
      `z.output` lines in `src/schemas/task.schemas.ts`.
- [ ] `npm run bruno` prints `PASS`, with the same counts as your Stage 1 baseline (process check, not in the
      brief).
- [ ] You can explain what `satisfies Prisma.TaskUncheckedCreateInput` checks, and why raw `req.body` never
      reaches Prisma.

### Stage 8 — Environment validation, without breaking the seed

**Goal:** the environment is checked by a Zod schema at startup, the process refuses to start on a bad value and
lists every problem, and the seed script and the other database scripts still run without an API key.

*Traces to the brief:* Lab step 5 (env validation in `config.ts` that fails fast on startup). Keeping the seed
script working is this guide's own process.

1. **Look at who imports what before you move anything.** `config.ts` runs `loadConfig(process.env)` when it is
   imported, and it needs `API_KEY`. In the reference build, `db.ts` also read `DATABASE_URL` and `LOG_QUERIES`,
   and `db.ts` is imported by the server, by `prisma/seed.ts` and by `scripts/check-prisma-errors.ts`. If
   `db.ts` imported `config.ts`, then `npm run db:seed` would refuse to run without an `API_KEY` it never
   uses. So the database settings go in a file with **no import-time effect**.

2. **`src/env.ts`:** the shared helper and the database settings:

   ```ts
   import { z } from 'zod';

   export function parseEnv<S extends z.ZodType>(schema: S, env: NodeJS.ProcessEnv): z.output<S> {
     const result = schema.safeParse(env);
     if (!result.success) {
       const seen = new Set<string>();
       const lines: string[] = [];
       for (const issue of result.error.issues) {
         const key = issue.path.join('.');
         if (seen.has(key)) continue;
         seen.add(key);
         lines.push(issue.message);
       }
       throw new Error(`Invalid environment:\n- ${lines.join('\n- ')}`);
     }
     return result.data;
   }

   const DATABASE_URL_MESSAGE = 'DATABASE_URL is required, for example file:./dev.db';

   export const dbEnvSchema = z
     .object({
       DATABASE_URL: z
         .string({ error: DATABASE_URL_MESSAGE })
         .refine((url) => url.trim() !== '', { error: DATABASE_URL_MESSAGE }),
       LOG_QUERIES: z
         .string()
         .default('false')
         .transform((text) => text === 'true'),
     })
     .transform((env) => ({ databaseUrl: env.DATABASE_URL, logQueries: env.LOG_QUERIES }));

   export type DbConfig = z.output<typeof dbEnvSchema>;

   export function loadDbConfig(env: NodeJS.ProcessEnv): DbConfig {
     return parseEnv(dbEnvSchema, env);
   }
   ```

   `parseEnv` throws **one** error that lists every problem, so a person fixes them all in one go. The
   `refine` keeps the value untrimmed (it only checks that it is not blank), and `LOG_QUERIES` keeps the
   old rule that only the text `true` turns logging on. Match whatever your v2 code did here.

3. **`src/db.ts`:** replace the hand-written `DATABASE_URL` check and the `LOG_QUERIES` comparison:

   ```ts
   import { loadDbConfig } from './env.ts';

   const { databaseUrl, logQueries } = loadDbConfig(process.env);

   export const prisma = new PrismaClient({
     adapter: new PrismaBetterSqlite3({ url: databaseUrl }),
     log: logQueries ? ['query', 'warn'] : ['warn'],
   });
   ```

4. **`src/config.ts`:** replace `loadConfig` with a schema that has the same rules and the same messages:

   ```ts
   import { z } from 'zod';
   import { parseEnv } from './env.ts';

   const NODE_ENVS = ['development', 'production', 'test'] as const;

   const portRule = z.int().min(1).max(65535);
   const PORT_MESSAGE = (issue: { input?: unknown }): string =>
     `PORT must be an integer from 1 to 65535, got "${String(issue.input)}"`;
   const API_KEY_MESSAGE = 'API_KEY is required and must not be empty';

   const envSchema = z
     .object({
       PORT: z
         .string()
         .default('3000')
         .refine((text) => portRule.safeParse(Number(text)).success, { error: PORT_MESSAGE })
         .transform(Number),
       NODE_ENV: z
         .enum(NODE_ENVS, {
           error: (issue) => `NODE_ENV must be one of ${NODE_ENVS.join(', ')}, got "${String(issue.input)}"`,
         })
         .default('development'),
       API_KEY: z
         .string({ error: API_KEY_MESSAGE })
         .refine((key) => key.trim() !== '', { error: API_KEY_MESSAGE }),
       CORS_ORIGIN: z
         .string()
         .default('*')
         .transform((text) => {
           const trimmed = text.trim();
           return trimmed === '*'
             ? ('*' as const)
             : trimmed
                 .split(',')
                 .map((origin) => origin.trim())
                 .filter((origin) => origin !== '');
         }),
     })
     .transform((env) => ({
       port: env.PORT,
       nodeEnv: env.NODE_ENV,
       apiKey: env.API_KEY,
       corsOrigin: env.CORS_ORIGIN,
     }));

   export type Config = z.output<typeof envSchema>;

   export function loadConfig(env: NodeJS.ProcessEnv): Config {
     return parseEnv(envSchema, env);
   }

   export const config = loadConfig(process.env);
   ```

   Two details you will otherwise trip on. The port is checked as **text** (`refine` on the string, with
   `portRule` run on `Number(text)`), so the message can show what was typed: if you coerce first, the error
   function's `issue.input` is already the number, and `PORT=abc` is reported as `got "NaN"`. And `Config` is
   `z.output` of the schema: the type of the config object is derived like every other type in this stage.

5. **Prove startup fails fast, with `.env` out of the way.** `env -i` starts a command with an empty
   environment, so only what you set is there (it also drops `PATH`, so pass it):

   ```bash
   $ env -i PATH="$PATH" DATABASE_URL=file:./dev.db PORT=abc NODE_ENV=staging API_KEY=" " npx tsx src/server.ts
   $ echo $?
   ```

   The reference output was one error listing all three problems (`PORT must be an integer from 1 to 65535,
   got "abc"`, `NODE_ENV must be one of development, production, test, got "staging"`, `API_KEY is required
   and must not be empty`), and `1` as the exit code. The server must not print "listening". Repeat with no
   `API_KEY` at all, and with no `DATABASE_URL`: each must exit 1 with its own message.

6. **Prove the database scripts do not need an API key:**

   ```bash
   $ env -i PATH="$PATH" DATABASE_URL=file:./dev.db npx tsx prisma/seed.ts
   $ env -i PATH="$PATH" DATABASE_URL=file:./dev.db npx tsx scripts/check-prisma-errors.ts
   $ env -i PATH="$PATH" npx tsx prisma/seed.ts
   $ npm run db:seed
   ```

   The first prints the seed counts and exits 0. The second prints three `PASS` lines. The third (no
   `DATABASE_URL`) must exit 1 with the `DATABASE_URL is required` message. The last is the real entry point,
   with your `.env`.

**Done when:**

- [ ] The server started with a bad `PORT`, a bad `NODE_ENV` and a blank `API_KEY` exits with code 1, before it
      listens, and one message lists all three problems (the brief's env validation that fails fast).
- [ ] `npx tsx prisma/seed.ts` with only `DATABASE_URL` set exits 0, and `npm run db:seed` still works
      (process check, not in the brief).
- [ ] `config.ts` and `db.ts` contain no hand-written `if` check of an environment value.
- [ ] You can explain why the database settings are in `env.ts` and not in `config.ts`.

### Stage 9 — Verify against the definition of done

**Goal:** collect the evidence for each box of the definition of done, and fix whatever it shows.

*Traces to the brief:* "You build and capture" (`tsc --noEmit`, no manual checks) and the Definition of done.
The id and date-boundary requests are this guide's own process.

1. **Handler types come from `z.infer`, proven with `tsc`.**

   ```bash
   npm run typecheck
   ```

   It exits 0. Then break a schema on purpose, to see the types follow it. In `createTaskBody`, rename `title`
   to `name`, and run `npm run typecheck` again. It must fail, in the service:

   ```text
   src/services/task-service.ts(63,49): error TS2741: Property 'title' is missing in type '{ name: string; ... userId: number; }' but required in type 'TaskUncheckedCreateInput'.
   ```

   The schema, the derived type and Prisma's type are tied together. **Undo the rename**, and check
   `npm run typecheck` exits 0 again.

2. **No manual checks remain.** Run the Stage 1 grep again:

   ```bash
   grep -rnE "if \(!|typeof [a-zA-Z_.]+ [!=]==? '|isNaN|Number\.isInteger|Number\.isSafeInteger|\.trim\(\) [!=]==|\.test\(" src prisma scripts --include='*.ts' --exclude-dir=generated
   ```

   Read every line it prints. Each must be something other than a check on a request body, param, query or
   environment value. In the reference build, what was left was `if (!result.success)` (a result, in `env.ts` and
   `validate.ts`), two `.refine((x) => x.trim() !== '')` (Zod rules that check a value is not blank and keep it
   untrimmed), type guards on a caught error in the error handler, `if (!ok)` in a script, and a regex in the
   request-id middleware that checks an incoming `X-Request-Id` header. That last one is hand-written validation
   of a header, which is outside this stage's scope (body, params, query, environment), so it stayed. Also
   check that nothing reads `req.body`, `req.query` or `req.params` outside `validate.ts`:

   ```bash
   grep -rnE "req\.(body|query|params)" src --include='*.ts' --exclude-dir=generated
   ```

   Only comments should match.

3. **Invalid input never reaches Prisma.** Start the server with query logging on, which prints the SQL behind
   every Prisma call:

   ```bash
   $ LOG_QUERIES=true npm start > /tmp/m08-server.log 2>&1 &
   ```

   (The `&` runs it in the background, so you can type the next commands in the same terminal.)

   Send invalid requests, using your own `API_KEY`:

   ```bash
   K='x-api-key: <your API_KEY>'; B=http://localhost:3000/api/v1/tasks
   req() { curl -s -w '  -> HTTP %{http_code}\n' "$@"; }
   req -X POST $B -H "$K" -H 'content-type: application/json' -d '{"title":"  ","priority":9,"status":"x","dueDate":"2026-02-30"}'
   req -X POST $B -H "$K" -d 'x'
   req -X PATCH $B/not-a-uuid -H "$K" -H 'content-type: application/json' -d '{"status":"nope"}'
   req -X PATCH $B/11111111-1111-4111-8111-111111111111 -H "$K" -H 'content-type: application/json' -d '{}'
   req -g "$B?status=todo&status=done&page=1e2&pageSize=101&sortBy=userId" -H "$K"
   ```

   (`req` is just `curl` that ends each response with its HTTP status on its own line, so the next command's
   output does not run into it.) Every one returns `HTTP 400`. Now count the SQL the server logged:

   ```bash
   grep -c "prisma:query" /tmp/m08-server.log
   ```

   It must print `0`. Then send **one valid** create, with extra keys the schema does not know:

   ```bash
   RESPONSE=$(curl -s -X POST $B -H "$K" -H 'content-type: application/json' -d '{"title":"  Stripped  ","id":"evil","userId":999,"foo":1}')
   echo "$RESPONSE"
   grep -c "prisma:query" /tmp/m08-server.log
   ```

   The create returns 201 with a title of `Stripped` (trimmed), a server-made `id`, and your default owner's
   `userId`: the `id`, `userId` and `foo` the client sent are gone. The count is now greater than 0 (2 in the
   reference build: the owner lookup and the insert).

   Now clean up, **by the exact id of the task you made** and never by a search: a search such as `?q=` can match
   seeded tasks (a search for `proto` matches "Prototype the tag filter"). Take the id from the response, delete
   that one task, and stop the server:

   ```bash
   ID=$(echo "$RESPONSE" | sed -E 's/.*"id":"([^"]+)".*/\1/')
   echo "$ID"
   curl -s -o /dev/null -w '%{http_code}\n' -X DELETE "$B/$ID" -H "$K"
   pkill -f "src/server.ts"
   ```

   The `DELETE` prints `204`. Check that `echo "$ID"` printed a UUID before you run the `DELETE`: an empty `$ID`
   would send the request to the list URL.

4. **The 400 carries `details` in the exact envelope.** The first request above returned, in the reference
   build (with `requestId` shortened):

   ```json
   {"error":{"code":"VALIDATION_ERROR","message":"Request validation failed","details":[{"path":"title","message":"Must not be empty"},{"path":"status","message":"Must be one of: todo, in_progress, done"},{"path":"priority","message":"Must be a whole number from 1 (high) to 3 (low)"},{"path":"dueDate","message":"Must be an ISO 8601 date such as 2026-12-31, or null"}],"requestId":"<id>"}}
   ```

   Check yours: `code`, `message`, `details` as an array of `{ path, message }`, and `requestId`, and no other
   key. Also check the other three: a body that is not JSON gives `{ "path": "body", "message": "Must be a JSON
   object" }`, a bad id with a bad body reports **only** `id`, and an empty `PATCH` gives
   `{ "path": "body", "message": "At least one of ..." }`.

5. **Check the id boundary, and pin your decisions in Bruno.** A malformed id must be a 400 and an id with the
   right shape but no task must be a 404. Check both against the running server:

   ```bash
   for id in not-a-uuid 123 11111111-1111-1111-1111-111111111111 12345678-1234-1234-1234-123456789012 AAAAAAAA-AAAA-4AAA-8AAA-AAAAAAAAAAAA; do
     printf '%-40s ' "$id"; curl -s -o /dev/null -w '%{http_code}\n' "$B/$id" -H "$K"
   done
   ```

   The reference build printed `400` for the first two and `404` for the other three, including the one that is
   not an RFC UUID (that is what `z.guid()` is for).

   Then give each boundary you decided in Stage 4, and each id boundary, a request in the Bruno collection, so
   the decision lives in a test and not only in the README. A rejected input is an `Errors/` request:

   ```text
   meta {
     name: 25 Create with hour 24
     type: http
     seq: 25
   }

   post {
     url: {{baseUrl}}/api/v1/tasks
     body: json
     auth: none
   }

   body:json {
     {
       "title": "Hour 24",
       "dueDate": "2026-01-01T24:00:00Z"
     }
   }

   assert {
     res.status: eq 400
     res.body.error.code: eq VALIDATION_ERROR
   }

   tests {
     test("rejects the date and names dueDate", function () {
       expect(res.body.error.details).to.eql([
         { path: "dueDate", message: "Must be an ISO 8601 date such as 2026-12-31, or null" },
       ]);
     });
   }
   ```

   An **accepted** boundary creates data, so it needs a clean-up. Put those in a folder of their own
   (`bruno/Dates/` with a `folder.bru` that has `seq: 4`): a first request creates a task and saves its id with
   `vars:post-response { dateId: res.body.id }`, the next ones `PATCH {{baseUrl}}/api/v1/tasks/{{dateId}}`
   with a different `dueDate` each and assert the `dueDate` that comes back, a `GET` reads one back from the
   database, and the last request deletes the task. The reference build had these, and each asserts the value
   Prisma returns:

   | Request | `dueDate` sent | Result |
   | --- | --- | --- |
   | `Errors/25` | `2026-01-01T24:00:00Z` | 400 |
   | `Errors/26` | `2026-02-29` (2026 is not a leap year) | 400 |
   | `Errors/27` | `2026-01-01T25:00:00Z` | 400 |
   | `Errors/28` | a `GET` of `11111111-1111-1111-1111-111111111111` | 404 `NOT_FOUND` |
   | `Dates/01` | `2028-02-29` (create) | `2028-02-29T00:00:00.000Z` |
   | `Dates/02`, `03` | `0050-06-15`, then a `GET` | `0050-06-15T00:00:00.000Z`, read back from SQLite |
   | `Dates/04` | `9999-12-31T23:59:59Z` | `9999-12-31T23:59:59.000Z` |
   | `Dates/05` | `2026-01-01T10:00+08:00` (minutes only) | `2026-01-01T02:00:00.000Z` |
   | `Dates/06` | `null` | `null` |

   Change these to match your own Stage 4 decisions: if you kept hour 24, `Errors/25` becomes a `Dates/`
   request that expects the next day.

6. **The whole collection, once more.**

   ```bash
   npm run typecheck
   npm run bruno
   ```

   Both must pass. The Bruno counts are the ones from Stage 1 plus the requests you added in step 5: the
   reference build went from 36 requests, 49 tests and 76 assertions to **47 requests, 63 tests and 91
   assertions**, and the original 36 were untouched. Check the server's log for `unexpected error` (there must
   be none).

**Done when:**

- [ ] `npm run typecheck` exits 0, and the temporary rename made it fail inside `task-service.ts`.
- [ ] Your grep for hand-written checks shows nothing that validates a request body, param, query or
      environment value, and you can say what each remaining line is.
- [ ] After five invalid requests, `grep -c "prisma:query"` on the server log printed `0`.
- [ ] A 400 response from your server has the M05 envelope with `details` as `{ path, message }` entries.
- [ ] A malformed id (`not-a-uuid`) returns 400, and a well-formed id with no task, including one that is not
      an RFC UUID, returns 404 (process check, not in the brief).
- [ ] Each date boundary you decided in Stage 4 has a Bruno request that asserts it (process check, not in the
      brief).
- [ ] `npm run bruno` prints `PASS`, and the original requests still pass with the same counts as your
      Stage 1 baseline (process check, not in the brief).

### Stage 10 — Document, self-review, and open the pull request

**Goal:** leave the project describing itself correctly, and deliver v3 the way you delivered the earlier
stages.

*Traces to the brief:* the Definition of done's "v3 merged by pull request". The README and the commit layout are
this guide's own process.

1. **Update `task-api/README.md` for v3.** It must say: the current version is v3 and that validation is Zod;
   a short **Validation** section (every route runs `validate`, the parsed values are in `res.locals.valid`,
   the parts are checked `:id`, body, query and the first failing part is the only one reported, unknown fields
   are dropped, a `ZodError` becomes the 400 envelope); the layout (`schemas/`, `env.ts`, `validate.ts`, no
   `validation/`); which variables are read in `config.ts` and which in `env.ts`, and why; and **every
   difference from v2 you accepted in Stage 4**, so the next person is not surprised.

2. **Check the dependency and what git would commit:**

   ```bash
   grep -n '"zod"' package.json
   npm run typecheck
   git status
   ```

   `package.json` must show `^4`. Delete `scratch/` before you run `git status`, so it cannot be added by
   accident. `git status` must not list `scratch/`, `dev.db`, `src/generated/prisma/`, `.env` or
   `node_modules`. It must list `src/schemas/`, `src/env.ts`, `src/middleware/validate.ts`, the new
   `bruno/` requests (`bruno/Dates/` and the files you added under `bruno/Errors/`) and the deletion of
   `src/validation/task-validation.ts`.

3. **Self-review against the brief**, once, top to bottom: the schemas for create, update, list query and
   params; the typed `validate` middleware; the env validation; handler types from `z.infer`; no hand-written
   validation anywhere; invalid input never reaching Prisma; 400s with `details` in the M05 envelope. Section 6
   below is the list.

4. **Commit in small, labelled pieces**, from the repo root, with Conventional Commits, for example:

   ```bash
   git add task-api/package.json task-api/package-lock.json
   git commit -m "chore(m08): add Zod 4"
   git add task-api/src/schemas task-api/src/middleware/validate.ts task-api/src/middleware/error-handler.ts
   git commit -m "feat(m08): add the Zod schemas, the validate middleware and the ZodError mapping"
   git add task-api/src/routes task-api/src/handlers task-api/src/services task-api/src/types task-api/src/validation
   git commit -m "refactor(m08): replace the hand-written validation with the schemas"
   git add task-api/src/env.ts task-api/src/config.ts task-api/src/db.ts
   git commit -m "feat(m08): validate the environment with Zod"
   git add task-api/bruno
   git commit -m "test(m08): pin the date and id boundaries in Bruno"
   git add task-api/README.md
   git commit -m "docs(m08): describe Task API v3"
   ```

   If `git add` says `pathspec ... did not match any files`, you are in a different folder from the one the
   path starts in: check with `pwd`. `git add` on the deleted `src/validation` path records the deletion.

5. Push the branch and open a pull request into your own repo's `main`, then merge it after review. The merge
   of this branch **is** Task API v3, and it closes Week 3.

**Done when:**

- [ ] `package.json` locks `zod` to `^4`, and `git status` shows none of the files listed in step 2 as things
      you would commit.
- [ ] `README.md` describes v3 and lists the differences you accepted (process check, not in the brief).
- [ ] A pull request for the branch exists, and `main` is untouched until it is merged (the brief's "v3 merged
      by pull request").

## 4. Concepts you need

### Quick reference

| You want | Use |
| --- | --- |
| Check and get the data, never throw | `schema.safeParse(x)`: `{ success, data }` or `{ success, error }` |
| Check and get the data, throw on failure | `schema.parse(x)` throws a `ZodError` |
| The type a handler receives | `z.output<typeof schema>` (the same as `z.infer`) |
| The type a caller must send | `z.input<typeof schema>` |
| A schema from another | `.pick`, `.omit`, `.partial`, `.extend` |
| A custom message | the `error` option: a string, or a function of the issue |
| Text in, number out | digits check, then `Number`, then `z.int()`. Not `z.coerce` alone. |
| The problems, as a list | `error.issues` (each: `code`, `path`, `message`) |

### `z.input` and `z.output`: one schema, two types

A schema that only checks has the same type going in and coming out. A schema that **changes** the value does
not, and Zod keeps both. Your list-query schema takes `{ page: '2' }` and produces
`{ page: 2, pageSize: 20, sortBy: 'createdAt', order: 'desc' }`. The compiler agrees: `z.input<typeof
listTasksQuery>` wants `page` as text and rejects `2`, and `z.output` has `page` as a number and rejects `'2'`.
`dueDate` is the same: text in, `Date` out.

The rule: **code that receives validated data takes the output type**, because that is what the schema
produced (defaults filled in, text turned into numbers and dates, unknown keys gone). The input type only
matters to code that **builds** a request, such as a test, or a client written in TypeScript. `z.infer` is
just `z.output`. This is also why a hand-written `CreateTaskInput` is a risk even when it compiles: if it says
`dueDate?: string`, a handler can use it as text, and the schema has already turned it into a `Date`.

### Where a custom message lives, and why you map `issues`

Stage 2 showed three formatters and Stage 6 mapped the issues by hand. What is worth knowing is *why* the
list of `issues` and not a formatter is the thing to map. A custom message sits on the issue that **owns** it.
A union's `error` is on the union's own issue, and the issues of its members carry their own default
messages. `z.treeifyError` walks both, so for the `dueDate` union it listed `Invalid ISO date`,
`Invalid ISO datetime` and `Invalid ISO datetime`, not the one message you set. `error.issues` held yours. So
a formatter can show you text you never wrote, and a mapper over `issues` is the one place you decide what the
client reads. The path is the other half: it is an array of keys (`['title']`, or `['tags', 1]` for the second
item of an array) and empty for the value as a whole, which is why an empty path needs a name of your choosing.

### A schema is a statement of what you accept

Every schema answers three questions per field: what happens to what it does not know, how text becomes a
value, and how strict a format is. Stages 2 and 3 settle each for this API. The idea worth keeping is what
comes after. **Tightening or loosening any of them changes what your API accepts, even when every message
matches**, so it is a change to the contract, and it should be decided against the contract you have
published, not against what the old code happened to do. There is also a third option for unknown keys that
this API does not use: `z.looseObject` keeps them. For an API that writes to a database that is how a client
sets `id` or an `isAdmin` flag, which is called mass assignment, and it is why stripping or rejecting are the
only safe choices.

### Zod is the request contract, Prisma is the persistence contract

They describe two different things. The request contract is what a **client** may send, and it can have rules
the database never sees (a trimmed title, `at least one field`, a date as text). The persistence contract is
what the **database** can store. Between them is a conversion, which is the service: the create schema produces
a `Date` for `dueDate`, and Prisma takes a `Date`.

The risk is the two drifting apart without anyone noticing: a new Prisma column nobody can set, or a schema
field that Prisma has no column for. `satisfies` makes the compiler check part of the join, and the reference
build tested exactly which part, with a scratch file and `tsc`:

- **Caught:** a wrong type on a field both sides have (a `priority` that is text where Prisma wants a number),
  and a field Prisma *requires* that the schema no longer produces (`title` renamed to `name` in the create
  body: `Property 'title' is missing`).
- **Not caught:** an extra field the schema produces that Prisma has no column for. TypeScript checks excess
  properties only on a fresh object literal, and the value here is a spread (`{ ...input, userId }`) or a
  variable. A schema field `nickname` compiled without a complaint.
- **Not caught on the update side:** `TaskUpdateInput` makes every field optional, so renaming `title` to `name`
  in the update schema still satisfies it.
- **Not caught in the other direction:** a Prisma column no schema can set.

So `satisfies` is a net for type drift and for dropped required fields, and not a proof that the two contracts
match. The rest is review, and tests (M09).

The same boundary applies on the way out. A schema can shape a response and strip internal fields. Whether you
need that depends on the contract you have already published. In this stage the README documents `userId` as
part of a task and the Bruno collection asserts it, so removing it would have broken the contract, and no
response schema was added.

### The concept-only topics (no lab)

These come up in practice and you do not use them here.

- **Zod Mini** (`import * as z from 'zod/mini'`) is a function-style version of the same library, written as
  `z.string().check(z.minLength(1))` rather than `z.string().min(1)`. Zod offers it for places where the size of
  the code you ship matters, such as a browser bundle. A server has no such pressure, so there is no reason to
  use it in this API.
- **`z.toJSONSchema(schema)`** turns a schema into JSON Schema. For
  `z.object({ title: z.string().min(1), priority: z.int().min(1).max(3).optional() })` it produced
  `"type":"object"`, `"required":["title"]`, `"minLength":1`, `"minimum":1`, `"maximum":3` and
  `"additionalProperties":false`. That is a starting point for generating API documentation from the same
  schemas that validate the requests, so the documentation cannot drift from the code. You do not need it for
  this stage.

## 5. Common pitfalls

- **Many routes return 500 before you have changed anything.** The server log says `ERR_DLOPEN_FAILED` and
  `NODE_MODULE_VERSION`. `better-sqlite3` was built for another Node version than the shell is running. Run
  `nvm use`, check `node --version` prints 24.x, and run your baseline **first**, so a red run is not blamed on
  your changes.
- **Copying Zod 3 code from the internet.** `z.string().email()`, `z.string().uuid()` and `message:` are Zod 3.
  In Zod 4 use `z.email()`, `z.uuid()` and the `error` option, and read the documentation for version 4.
- **`z.coerce.number()` on a query string.** It accepts `"1e2"`, `" 5"`, `"0x10"` and `"1.0"`, and turns `""`
  and `null` into `0`. Check the digits first.
- **Piping a string into `z.coerce.number()`.** `.pipe(z.coerce.number())` after `z.string()` fails to compile:
  `Argument of type 'ZodPipe<ZodCoercedNumber<unknown>, ZodInt>' is not assignable to parameter of type
  '$ZodType<any, string, ...>'`. A coerced number's input is `unknown`, not `string`. After a digits check,
  `.transform(Number)` does the job.
- **`z.uuid()` rejecting ids your API handed out in the past.** It checks the version and variant digits. If
  your M05 validator accepted any hex shape, `z.guid()` keeps the behaviour, and `z.uuid()` turns some 404s into
  400s.
- **`z.iso.datetime()` and times without seconds.** `2026-01-01T10:00Z` fails the default, and
  `precision: -1` accepts it and then rejects `10:00:00Z`. If your validator took both, use both in a union.
- **Assuming a number reports one problem.** `priority: 1e300` made Zod report two `too_big` issues for the same
  field. Without the one-entry-per-path rule, your `details` repeats a field.
- **Messages in the environment error that show the coerced value.** If you coerce `PORT` first, the error
  function sees the number and `PORT=abc` says `got "NaN"`. Check the text, then convert.
- **Importing `config.ts` from something that does not need it.** It validates the environment when it is
  imported, so anything that imports it needs an `API_KEY` and the rest. Keep the settings a script needs
  (`DATABASE_URL`) somewhere with no import-time effect, or your seed script stops working without them.
- **Assigning the parsed query to `req.query`.** In Express 5 it is a read-only getter. Put parsed values on
  `res.locals`.
- **A `refine` that does not run.** It runs only after the fields passed their own checks (the Stage 2 tour
  showed it). A cross-field rule will not report beside a type error, and a rule you expected to see is not
  broken: fix the type error first.
- **Leaving the old validator beside the new one.** The brief's rule is that the hand-written code disappears,
  and `tsc` will not tell you about code that still compiles. Delete the file, then let the compiler list what
  broke.
- **Trusting `grep` to be silent.** The patterns find legitimate lines too (a `.trim() !== ''` inside a `refine`,
  a type guard on an error). Read each hit and name what it is, rather than aiming for zero.
- **Deciding a date rule by what the old code did.** The old validator accepted hour 24 and refused year 0050,
  and neither was something the README promised. Compare against your **documented** contract: follow it where
  it is explicit, and where it is silent, decide and write the answer into the README, then pin it in Bruno.
  Reproducing an accident, or tightening a rule without saying so, are both silent changes to your API.
- **Comparing only the messages.** Bruno passes when the messages and statuses you asserted match. Inputs
  nobody wrote a request for (hour 24, a year before 0100, `"0x10"`) are where a schema and the code it replaces
  differ. Run both on the same inputs.
- **Committing `scratch/`, `.env` or the database.** `git status` must not list them.

## 6. Check yourself against the definition of done

Tick every box. Each one is something you can show or say.

- [ ] **v3 is merged by pull request**, from a `feat/module-08/...` branch, into `main`.
- [ ] **No manual `if (!body.title)`-style checks remain anywhere in the codebase.** You ran the grep, read every
      hit, and each is something other than a check on request or environment input.
- [ ] **Invalid input never reaches Prisma.** A schema rejects it first, every time: five invalid requests
      against a server logging queries left the log with no `prisma:query` line.
- [ ] **400 responses carry field-level `details` in the exact envelope shape fixed in M05**: `error` with
      `code`, `message`, `details` (an array of `{ path, message }`) and `requestId`, and nothing else. Zod's
      output was mapped into `details`, and the shape did not change.
- [ ] **The deliverable exists**: schemas for create, update, list query and params; a typed
      `validate({ params, body, query })` middleware; environment validation in `config.ts` that fails fast
      on startup; and handler types that come from `z.infer`, proven with `tsc --noEmit`.
- [ ] `npm run typecheck` exits 0 (it proves the handler types come from the schemas).
- [ ] You can explain, without notes: why TypeScript types do not validate a request, what the difference is
      between `z.input` and `z.output`, and why the parsed query is not put back on `req.query`.

Process checks of this guide, **not required by the brief** (run them anyway): `npm run bruno` prints `PASS`
with the counts you wrote down in Stage 1 (plus the requests you added in Stage 9), every difference from v2
is decided and written in your README, and the seed script and `npm run check:errors` run with only
`DATABASE_URL` set.

This is the **Week 3 Friday gate.**

## 7. Next step

M09 adds integration tests to this API with Jest 30 and Supertest 7. Keep the schemas, the `validate`
middleware and the envelope exactly as they are: the tests will assert the envelope you fixed in M05, the 400s
you checked here and the status codes of every route, against a real SQLite test database. The
`z.input` types are the ones a test uses to build a request body.
