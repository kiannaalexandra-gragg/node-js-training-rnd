# M05: backlog

Items found in VALIDATE and in the `/code-review` of both repos (2026-10-21). The build (`task-api/` in the build repo) and both documents were
checked against the brief's Definition of done, the tasks checklist and the AGENTS.md quality gates. `npm run typecheck` and `npm run bruno` were re-run on
Node 24.22 against a freshly started server after every change: the final result is 29 of 29 requests, 38/38 tests and 59/59 assertions (the first version
was 26 requests, 31/31 and 53/53).

VALIDATE fixed two FIX NOWs: the Concepts essays on the `app.ts`/`server.ts` split and on graceful shutdown restated Stage 9, and `task-api/README.md` now
says the Bruno collection must be run whole. The `/code-review` (one reviewer per repo) found 7 issues in the build repo and 9 in the documents. All were
fixed or recorded below. In the build: `server.ts` now checks the error that `app.listen`'s callback receives (a busy port printed "listening" and exited 0),
the central handler keeps any 4xx `status` or `statusCode` instead of turning a bad `%` escape or an undecodable body into a 500 (a first fix that required
`expose: true` missed the router's error and was corrected), `express.json()` moved after the key check, Bruno gained requests 20 to 22 and `details.path`
assertions on 16 and 17, and the README wording was narrowed. In the documents: they had been written against the first version of the code, so the three
changed files were re-embedded and every count, table and experiment was brought into line. The Express 4 explanation was also wrong (it said the request
hangs: on Node 15 and later an unhandled rejection ends the process, checked on 24.22), Stage 10 experiment 3 quoted one message for two requests, the
Concepts essays on the pipeline and on operational errors restated stages, the `status` filter rationale overclaimed, two "Done when" lines could not be
checked as written, the migration guide had no URL, Stage 11 never showed a `patch`, a `delete` or `isArray`, and two small ordering slips were fixed.
Nothing in this file blocks the build or the two documents. The build repo's PR #5 was merged on 2026-10-21 (merge commit `34dc299`), and its link and
`git log` are in `write-up.md` Checkpoint evidence. As in M03 and M04, the author's PR link belongs in `write-up.md` only. The Express 5 explanation in the
write-up was drafted from the experiments and the router source and was accepted as the trainee's own. The `task-api-v1` tag is deferred until the EM has reviewed v1.

## [BACKLOG] Decisions made without the EM (contract gaps, EM-owned files not edited)

`brief.md` and `tasks.md` are EM-authored, so they were not changed. For the EM to confirm:

- **Success-response shape.** The brief fixes the error envelope and the Task's fields, not a success wrapper. v1 returns the bare resource and a bare
  array. If M07's pagination needs `{ data, meta }`, that changes the contract and the Bruno assertions in the same PR.
- **`ConflictError` has no use.** The brief names it as an example class but defines no v1 rule that conflicts. It is defined and documented, nothing throws
  it, and there is no 409 request in Bruno. The EM may want a rule, or to drop the example from the brief.
- **422 against 400.** The brief's status list includes 422, but the arch doc maps validation failures to 400 and M08's Zod errors do the same. v1 uses 400
  for every validation error and never produces a 422.
- **Three behaviours added by the review fixes**, for the EM to confirm: a new code `CLIENT_ERROR` (4xx other than 400 and 413, message `Request could not
  be processed`) for client errors that Express reports without a code of its own, `BAD_REQUEST` also covering a malformed `%` escape, and the key being
  checked before the body is parsed (malformed JSON without a key is a 401). The brief fixes the envelope but leaves the set of codes to the trainee, so none
  of these changes the contract.
- **API-key auth.** It is in the brief's Scope ("custom middleware") but not in the tasks.md Build checklist. It was built (router-level on `/api/v1`, `/health`
  left open), with the key from a required `API_KEY` variable.
- **Where the code lives.** The brief does not say. M01-M04 used `modules/mNN-<name>/` in the build repo. The Task API went in one top-level `task-api/`
  folder (the arch doc's reference layout) for M05, M07, M08 and M09, and the build repo's README says why. The brief's "Builds on" and M07's text could name
  the folder.
- **Timestamps.** M04's Next step asked whether `createdAt` and `updatedAt` stay on the Task and whether the API exposes them. In v1 they are left out, because
  the contract's Task has four fields and nothing uses them. M07's database can own them.

## [BACKLOG] Behaviour not covered by the brief, left as built

- **A wrong method on a real path is a 404, not a 405.** `PUT /api/v1/tasks/:id` returns `NOT_FOUND` through the 404 middleware. The brief does not list 405, and
  adding it would add an error code the contract does not define.
- **Unknown query keys are ignored too.** `?Status=done` returns every task. Only the *value* of `status` is validated. The README says so.
- **The API key's strength is not checked.** `API_KEY=a` starts, and so does the fake example key with `NODE_ENV=production`. A rule about a key's content
  would be a fragile heuristic and deployment is out of scope, so the README tells the reader to set their own key.
- **Errors Node itself rejects never reach Express** (a malformed header line, oversized headers). Node answers with a bare 400 and no JSON, so "every error
  response" is true for what Express produces. The README says so. A `server.on('clientError', ...)` handler would make it literal.
- **`description` cannot be cleared.** `{"description": null}` is a 400 and `""` is accepted. The contract does not say how to clear an optional field.
- **`CORS_ORIGIN=","` or an empty list allows no origin**, and a trailing slash on an origin never matches. Both are minor and unhandled.
- **Unknown body fields are ignored**, not rejected. This matches Zod's default in M08, and the choice is documented in the README, the write-up and the guide.
- **The 500 path has no Bruno request.** Proving it needs a handler that throws, and a debug route in the API was ruled out. It is proved by a temporary throw
  in Stage 10, with a `grep` that the throw was removed.
- **Graceful shutdown was checked only on an idle server.** A request still in flight when `SIGTERM` arrives was not tried, because every handler answers in
  about a millisecond. A deliberately slow handler would be needed to test it.
- **`express-rate-limit` was researched in a scratch project, not built.** The default 429 is plain text, so adding it needs a `handler` that routes into the central
  error handler and a new `RATE_LIMITED` code, which grows the contract. The EM should decide whether a later stage adds it.
- **Not tried:** Windows (`kill -TERM` and `lsof` have no equivalent in the guide), and the Bruno desktop app.
- **Install-script warnings.** `npm install` warned about `esbuild`, and `npm ci` listed `esbuild`, `fsevents` and `protobufjs` as install scripts not covered
  by `allowScripts`. The project ran without approving any. The guide tells the learner to read the warning and not approve it blindly.

## [BACKLOG] Size

`training-material.md` is about 2,300 lines, close to M04's. Stages 4 and 7 are exercises with hints, and every other stage gives the code. If the EM wants it
shorter, the Stage 11 request table could drop the reference files for requests that follow the pattern of one already shown.

## [BACKLOG] `npm audit` reports 12 vulnerabilities, all in the dev-only Bruno CLI

On a clean `npm ci` of `task-api/` (2026-10-21), `npm audit` reported 12 vulnerabilities (4 moderate, 8 high), and `npm audit --omit=dev` reported 0. Every one is in the
dependency tree of `@usebruno/cli@4.2.1` (a dev dependency): `axios`, `@faker-js/faker`, `form-data`, `js-yaml`, `csv-parse` and Bruno's own packages. Express, `cors`,
`helmet` and the app's own code are not affected. `npm audit fix --force` was **not** run: it would change or break the pinned Bruno CLI that runs the collection. The
Bruno CLI runs only on the developer's machine against `localhost`, so the exposure is small, but a trainer should know the guide's install step will print the warning.
Options for the EM: accept it as dev-only, pin a newer `@usebruno/cli` once one clears the advisories, or document the Bruno desktop app as the way to run the collection.

## [BACKLOG] The program's rules list only one of the removed `res.json` signatures (shared files, not edited)

`coding-standards.md` (line 44), the arch doc's pre-flight checklist (item 9), the PRD (`docs/prd/nodejs-developer-training.md`) and `knowledge/rules/arch-summary.md` name
`res.json(status, body)` as the removed Express 4 pattern to look for in lab solutions. That form did exist in Express 4 and is gone in Express 5, so the rule is not wrong. It
is incomplete: the Express 5 migration guide (read on 2026-10-21) also lists `res.json(obj, status)`, with the status as the **second** argument, and likewise
`res.send(body, status)`, `res.send(status)` and `app.del()`. Express 4's `res.json` accepted both argument orders as deprecated forms. M05's training material follows the guide.
When the EM updates the pre-flight grep, it should **add** `res.json(obj, status)` to the pattern and keep `res.json(status, body)`, not swap one for the other.
`AGENTS.md` itself has no `res.json` wording.
