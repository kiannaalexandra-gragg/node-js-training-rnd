# Coding Standards

Standards for **any code that appears in this repo's content**: snippets in a `write-up.md` or
`training-material.md`, and the code a trainee builds for a stage. This repo holds teaching content,
not an application, so these standards exist to keep what we teach correct and consistent.

Source: `docs/arch-docs/Node.js Developer Training Program — Solution Design.md` and each module's
`brief.md`. When they disagree, the module's `brief.md` wins for that stage.

## Pinned stack

| Layer | Version | Rule |
| --- | --- | --- |
| Runtime | Node.js 24 LTS | Decided (see root `README.md`). One version per cohort, pinned in `.nvmrc`. |
| Language | JavaScript ES2023–ES2025 in Week 1; TypeScript 7.0 from M04 | Everything taught runs natively on Node 24+. |
| Web framework | Express 5.x | `npm i express@5`. |
| Data access | Prisma ORM 7.x | Always `@7`: `prisma@7`, `@prisma/client@7`, `npx prisma@7`. Never an unversioned `prisma`; it can resolve to Prisma 8, which does not read `schema.prisma` for this stack. Link `/v7/` docs only. |
| Database | SQLite via `better-sqlite3` and the Prisma driver adapter | The only database used hands-on. PostgreSQL and MongoDB are concepts only. |
| Validation | Zod 4.x | `npm i zod@4`. |
| Test runner | Jest 30.x | `@swc/jest` transform, ESM mode. |
| HTTP testing | Supertest 7.x | Calls the exported app, so no port is needed. |
| Dev runner | `tsx` | Used by the `dev` and `start` scripts. |
| API client | Bruno | Collections are plain text, committed in `bruno/`. |

Free tools only. No Docker, no paid licences. If content mentions a version, it must match this table.

## Language and typing

- Plain modern JavaScript in Week 1 (M01–M03). TypeScript from M04 onward.
- TypeScript runs in **strict** mode. Types are checked by `tsc --noEmit` (`npm run typecheck`), not
  by the Jest transform, which only strips types.
- Prefer `unknown` over `any`. If `any` appears in a snippet, the text must say why.
- ES modules (`"type": "module"`) throughout.
- **Open decision (P2):** `.ts` import extensions with `allowImportingTsExtensions` versus `.js`
  specifiers. Not settled by the briefs. Do not present either as the program's convention. Follow
  the stage's `brief.md` and say which you used.

## Express 5

- Rejected promises in async handlers reach the error handler automatically. Do not wrap handlers
  in `try/catch` just to forward errors.
- `app.ts` builds and exports the app and never calls `listen`. `server.ts` listens and shuts down
  gracefully on `SIGTERM`.
- Do not teach removed Express 4 patterns: wildcard paths such as `*`, and `res.json(status, body)`.
  Use `res.status(n).json(body)`.
- Middleware order matters and must be explainable. State what breaks if two middleware swap.
- The central error handler has four parameters and is registered last. With three, or registered
  before the routes, Express's built-in handler answers instead: an HTML page with a stack trace
  when `NODE_ENV` is `development`. Prove an order claim by swapping the lines and running it.
  Swapping the logger and the request-ID middleware changes nothing, because the logger reads the
  id on `finish`. Check the cheap rejection before the expensive step: the API-key check comes
  before `express.json()`.
- Never put a stack or `err.message` of an unexpected error in a response. Map it to a generic 500
  and log the detail with the request id. A client error that a library marks with a 4xx `status`
  or `statusCode` keeps that status, with a fixed message. Check the error `app.listen`'s callback
  receives, and exit 1 on it.

## Errors

Every error response from every endpoint uses the fixed envelope defined in M05's `brief.md`:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [{ "path": "title", "message": "Required" }],
    "requestId": "<from the request-ID middleware>"
  }
}
```

`details` appears only on validation errors. No stack traces in production responses (M05 sends none in any environment). Every error
reaches one central handler. `AppError` subclasses set their own status, a `ZodError` becomes a
400 with field-level `details`, and Prisma `P2002`, `P2025` and `P2003` map to 409, 404 and 409/400.

## Data and validation

- **Zod is the request contract. Prisma types are the persistence contract.** Raw `req.body` never
  reaches Prisma.
- Handler input types come from `z.infer`, not hand-written duplicates.
- Validate environment variables once at startup in `config.ts` and fail fast.
- Keep the settings a script needs (`DATABASE_URL`) out of `config.ts`. It validates the whole environment
  when it is imported, so a seed script that imported it would need `API_KEY` too. Put them in a module with
  no import-time effect, as `env.ts` does.
- Zod 4 is not Zod 3: use `z.email()`, `z.uuid()`, `z.url()`, the `error` option for messages, and
  `z.flattenError`. Read the version 4 documentation.
- Do not assign parsed request data back to `req.query`: it is read-only in Express 5. Put it on `res.locals`
  (or a typed `req.valid`, which needs a global Express type merge) and state which one the project uses.
  Derive the handler's type from the same schema object the route passes to `validate`, with `z.output`.
  The compiler cannot tell whether a route has `validate` in front of a handler, so read the parsed data
  through a helper that throws when it is missing, and make `validate` refuse keys other than `params`,
  `body` and `query` at compile time.
- A `z.object` strips unknown keys, `z.strictObject` rejects them, and `z.looseObject` keeps them. For an API
  that writes to a database, never keep them. State which one a schema uses, and why.
- `z.coerce.number()` is `Number(input)`: it accepts `"1e2"`, `" 5"` and `"0x10"`, and turns `""` and `null`
  into `0`. For a query string, check the digits (`z.string().regex(/^\d+$/)`), then `.transform(Number)`,
  then `z.int()`. `.pipe(z.coerce.number())` after a string schema does not type-check, because a coerced
  number's input type is `unknown`.
- `z.uuid()` checks the version and variant digits and `z.guid()` does not. Switching from a hand-written
  8-4-4-4-12 check to `z.uuid()` turns some ids that were a 404 into a 400.
- `z.iso.datetime({ offset: true })` needs seconds: add a member with `precision: -1` for a time that stops at
  the minute. `z.iso.datetime` also rejects hour 24, and `z.iso.date()` checks the day against the calendar
  (`2026-02-30` fails), so no hand-written calendar check is needed. A zone offset can still move a date at the
  edge of the year range outside it, so say in the README whether the limit is on the text or on the instant.
- State a date input's accepted forms and range in the README, decide each boundary against that contract
  instead of against the old code, and pin it with a Bruno request. A boundary the old code got right by
  accident or wrong by accident is not part of the contract.
- One field can produce more than one Zod issue (`priority: 1e300` fails both `z.int()` and `.max(3)`). Map
  `error.issues` to one `details` entry per path, the first message winning, and give an issue with an empty
  path the path `body`.
- `satisfies Prisma.<Input>` on a schema's output catches a wrong type on a shared field and a required Prisma
  field the schema stopped producing. It does not catch an extra field Prisma has no column for (excess
  properties are only checked on a fresh object literal), and it cannot catch a rename in an update input,
  where every field is optional. Do not describe it as proof that the two contracts match.
- When replacing hand-written validation with a schema, run both on the same inputs before deleting the old
  code. Matching messages does not mean matching behaviour, and Bruno only covers the inputs someone wrote a
  request for.
- `db.ts` exports one `PrismaClient` singleton built with the driver adapter.
- The generated Prisma client is git-ignored. Run `prisma generate` after install and after every
  schema change.
- In Prisma 7.10, `migrate dev` does not run `generate`, and `migrate reset` does not run the seed. Run
  `prisma generate` and `prisma db seed` explicitly, and say so in any stage that depends on them.
- A SQLite `enum` is plain `TEXT` with no `CHECK`, so Prisma and validation enforce it and the database
  does not.
- `prisma studio` on SQLite needs a URL with `://` (`--url file://./dev.db`): it reads everything before
  `://` as the protocol, so the `file:./dev.db` form is rejected.
- `migrate status` does not notice an applied migration that was edited afterwards; `migrate dev` does.
- A date from a client is not valid just because `new Date(text)` is not `Invalid Date`: JavaScript rolls
  `2026-02-30` over to 2 March. Check the day against the calendar too, and test impossible dates.
- Install `better-sqlite3` at the major version the Prisma adapter declares (`@12`), or npm installs a
  second copy beside the adapter's own.

## Testing

- Integration tests use a real SQLite test database (`test.db`), not mocks of the data layer.
- `prisma migrate deploy` runs in `globalSetup`. Tables are cleaned in `beforeEach`.
- Tests must pass in random order (`--randomize`) and run with `--runInBand`.

## Repo hygiene

- `.gitignore` covers `node_modules`, `.env`, `*.db`, the generated Prisma client, `dist`, and
  `coverage`.
- Ignore the generated Prisma client as `**/generated/prisma`. The client is generated in
  `src/generated/prisma/`, and a pattern with a slash in the middle (`generated/prisma`) only
  matches from the repo root, so it would miss it. Prove an ignore rule with
  `git check-ignore -v <real path>`, not a file at the repo root.
- Commit `.env.example`. Never commit `.env` or `.env.test`. Never put a real secret in an example.
- Conventional Commits for every commit.
- Proposed, pending EM confirmation (see `knowledge/retros/m05-backlog.md`): the Task API is one
  project, `task-api/` at the build repo's root, shared by M05, M07, M08 and M09, and each stage is a
  PR that changes it in place. Other stages live in `modules/mNN-<name>/`.
