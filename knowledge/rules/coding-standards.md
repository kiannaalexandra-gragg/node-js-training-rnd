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
