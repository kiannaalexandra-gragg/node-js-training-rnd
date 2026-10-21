# M04 Write-Up — TypeScript Basics & TypeScript on Node.js

> Written from what I captured as I built it: raw notes after each slice, then this document. Written
> so a trainee starting this stage next cohort could follow my path on their own.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo, [`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
in `modules/m04-typescript-on-node/`, on the branch `feat/module-04/typescript-on-node`, merged to `main` by
[pull request #4](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/4) (see Checkpoint
evidence). The module folder:

```text
m04-typescript-on-node/
├── package.json
├── package-lock.json
├── tsconfig.json
├── .env.example
└── src/
    ├── toolkit/
    │   ├── async-toolkit.ts          async-toolkit.check.ts
    ├── domain/
    │   ├── task.ts                   domain.check.ts
    │   └── result.ts
    ├── env.ts                        env.check.ts
    └── demo.ts
```

- **The M02 toolkit, ported to TypeScript**, in one file, `async-toolkit.ts`: `sleep`, `retry<T>`,
  `withTimeout<T>`, `mapLimit<T, R>`, and the two error classes `RetryError` and `TimeoutError`. The behaviour
  is the M02 behaviour. Only the types are new.
- **The Task domain**, in `domain/task.ts`: `Task`, `TaskStatus` (a union, not an `enum`), `CreateTaskInput`
  and `UpdateTaskInput` (both derived from `Task` with utility types), a user-defined type guard
  `isTaskStatus`, a `STATUS_LABELS` table checked with `satisfies`, and two small functions, `createTask` and
  `applyTaskUpdate`, so the types are used by something.
- **`Result<T, E>`**, in `domain/result.ts`: a discriminated union, with `ok`, `err`, `toError`, `attempt`,
  `mapResult` and `unwrapOr`.
- **Typed `process.env`**, in `env.ts`: `readEnv()` returns a `Result<Env, string>` instead of letting
  `string | undefined` leak into the program. `demo.ts` ties it together: it saves five tasks with
  `mapLimit`, retries a flaky save with `retry`, and turns a timeout into an error `Result` with
  `withTimeout` and `attempt`.
- **Three check files**, each next to the code it checks: 40 `node:assert` checks in total (23 + 14 + 3). The
  23 are the 23 M02 checks ported. All 40 pass. The domain checks also contain *type-level* checks that are
  verified by `npm run typecheck`, not by running the file.
- **Project files**: `package.json` (`"type": "module"`, `engines` for Node 24, scripts `typecheck`, `demo`,
  `check`, `check:native`), `tsconfig.json`, `package-lock.json` and an `.env.example` documenting
  `TOOLKIT_DELAY_MS`. I also added one line to the repo root's `.env.example`, which is an index of each
  module's settings, as M03 did.

The experiments (native type stripping failure cases, `unknown` against `any`, TypeScript 6 against 7, the
import-extension conventions, and so on) were throwaway files in a scratch folder outside the repo. Their
results are in "Why it's built this way" and "What tripped me up".

Run everything from `modules/m04-typescript-on-node/`, on Node 24:

```bash
nvm use
npm install
npm run typecheck      # tsc --noEmit: zero errors
npm run check          # all 40 checks, run by tsx
npm run check:native   # the same 40 checks, run by node itself (native type stripping)
npm run demo           # the toolkit and the Task types together
```

Environment: Node v24.22.0 and npm 11.19.1 (`.nvmrc` pins Node 24), TypeScript 7.0.2, `tsx` 4.23.15,
`@types/node` 24.19.2, macOS 26.5, Apple silicon, zsh. My default shell starts on Node 22.23.0, so I run
`nvm use` in each session.

## Why it's built this way (key decisions)

**The project: `tsconfig.json`, option by option**

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

I checked what most options do by removing or breaking them, so those reasons are things I saw. `noEmit` and
`skipLibCheck` are the exceptions: I describe their standard behaviour and did not test them.

- `strict`: kept explicit for readability. TypeScript 7.0.2 is strict when the option is absent (a config with
  no `strict` at all still reported `TS7006` and `TS2322`).
- `module` and `moduleResolution: nodenext`: Node's own rules for ES modules, so the compiler agrees with the
  runtime about file extensions. (`--showConfig` shows that `module: nodenext` alone already implies
  `moduleResolution: nodenext`.)
- `noUncheckedIndexedAccess`: makes `list[0]` a `T | undefined`. It caused the one real error in the toolkit
  port (see `mapLimit` below).
- `verbatimModuleSyntax`: forces `import type` for types. It is also what makes native type stripping safe
  (see "What tripped me up", item 5).
- `erasableSyntaxOnly`: rejects TypeScript syntax that has to be *rewritten* (`enum`, constructor parameter
  properties). The compiler reported `TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled`
  for my test `enum`. It keeps the code runnable by native type stripping.
- `noEmit`: `tsc` is only a checker here. `tsx` and `node` run the code.
- `skipLibCheck`: skips type-checking the `.d.ts` files of dependencies, so only my code is checked.
- `types: ["node"]`: **required** in 7.0.2. With it removed, `console` and `process` were "Cannot find name"
  (`TS2584`, `TS2591`) even though `@types/node` was installed, so `@types/*` packages are not picked up
  automatically. There was a second, stranger error too: after `if (!env.ok) { ...; process.exit(1); }` the
  compiler no longer narrowed `env` to the ok variant. Without `@types/node` it does not know `process.exit`
  returns `never`, so it cannot tell that the code after the `if` is only reached when `env.ok` is true.
- `target` and `lib: es2024`: `Promise.withResolvers` (used by `withTimeout`) is in the ES2024 library. With
  `lib: ["es2022"]` the compiler said `TS2550: Property 'withResolvers' does not exist on type
  'PromiseConstructor'`. With no `target` and no `lib` at all the project also compiled, so I kept them for
  clarity, not necessity.
- `allowImportingTsExtensions`: see the next decision.

**The import-extension convention (the open decision "P2")**

I used `.ts` specifiers (`import { sleep } from './async-toolkit.ts'`) with `allowImportingTsExtensions`. This is
my choice for this stage and I do not present it as the program's convention. What I measured, with
`strict`, `nodenext`, `verbatimModuleSyntax` and `erasableSyntaxOnly` on:

| Specifier in the source | `tsc --noEmit` | `tsx` | `node file.ts` (native) |
| --- | --- | --- | --- |
| `'./lib.ts'` | error `TS5097` unless `allowImportingTsExtensions` is on, then passes | runs | runs |
| `'./lib.js'` (only `lib.ts` exists) | passes | runs | `ERR_MODULE_NOT_FOUND` |
| `'./lib'` (no extension) | error `TS2835` ("Did you mean './lib.js'?") | runs | `ERR_MODULE_NOT_FOUND` |

So only the `.ts` specifier works with all three tools, and that is why I chose it: I wanted the native run to
be a real attempt, not a guaranteed failure. The cost is that a `tsc` *build* needs help. I tried one as an
awareness check: with `.ts` specifiers and no extra option, `tsc` stops with `TS5097`. With
`"rewriteRelativeImportExtensions": true`, `"rootDir": "src"` and `"outDir": "dist"` it emitted
`import { greet } from './lib.js';` and `node dist/main-ts.js` ran. A source tree that already uses `.js`
specifiers builds with no extra option. I did not build the real project, only a two-file probe.

**Which utility types for `CreateTaskInput` and `UpdateTaskInput`, and why not by hand**

```ts
export type CreateTaskInput = Omit<Task, 'id' | 'createdAt' | 'updatedAt' | 'status'> &
  Partial<Pick<Task, 'status'>>;
export type UpdateTaskInput = Partial<CreateTaskInput>;
```

The reasoning is in "How to build it", step 3. The short version: the input types are *Task minus the fields
the server owns*, so I wrote that sentence in types (`Omit`) and let the compiler work out the rest.
`UpdateTaskInput` is `Partial<CreateTaskInput>`: any subset of what a client may set.

I tested the difference between deriving and hand-writing. I copied `task.ts` into a scratch folder, added
`dueDate?: Date` to `Task`, and tried to assign `{ title: 'a', dueDate: new Date() }` to three versions:

| Version | Before adding `dueDate` to `Task` | After adding it |
| --- | --- | --- |
| `Omit`-derived (mine) | rejected (`TS2353`) | **accepted**, with no edit to the input type |
| Hand-written `{ title; description?; status? }` | rejected | still rejected |
| `Pick`-derived (`Pick<Task, 'title' \| 'description'> & Partial<Pick<Task, 'status'>>`) | rejected | still rejected |

So deriving keeps the input type in step with `Task`, and that is the argument for it. But the same table shows
the cost of **`Omit`**: it is a deny-list. Every new `Task` field becomes client-writable by default, until I
remember to add it to the omitted list. `Pick` is an allow-list: a new field is invisible to clients until I
add it. For a create input, where the server is the authority, `Pick` may be the safer default. I used
`Omit` and I only saw this trade-off afterwards, in the experiment. The `Task` fields are my own choice
too: the M05 brief lists `id`, `title`, an optional `description` and `status`, and I added `createdAt` and
`updatedAt` so there were server-owned fields to omit.

**Why `applyTaskUpdate` names every field**

My first version was `{ ...task, ...update, updatedAt: now }`. It type-checked and passed my checks. A code review then found
two holes, and I reproduced both. First, every field of `UpdateTaskInput` is optional, and with
`exactOptionalPropertyTypes` off an optional property also accepts `undefined`, so `applyTaskUpdate(t, { title: undefined })`
compiled and returned a `Task` with no title. Second, only a *fresh object literal* is checked for extra properties. An object
held in a variable is not, so `const loose = { title: 'z', id: 'hijack' }; applyTaskUpdate(t, loose)` compiled and
overwrote the `readonly` `id`: the type says clients cannot set `id`, but the spread copies whatever the object really holds.
The function now names each field (`title: update.title ?? task.title`, `status: update.status ?? task.status`, and
`description` only when it is not `undefined`). An `undefined` field means "leave it alone", and extra keys are ignored. I did not turn on
`exactOptionalPropertyTypes`, because it would not stop the second problem. The checks now cover both cases, plus two new
type-level assertions (`keyof UpdateTaskInput` is exactly `title | description | status`, and `Task['description']` is
`string | undefined`). I confirmed each new check fails when the old code or a wrong type is put back
(`UpdateTaskInput = Partial<Task>`, `description?: string | null`, the spread, the unguarded `toError`, the `Number()` validation).

**How I modelled `Result<T, E>`, and what would change otherwise**

```ts
export type Result<T, E = Error> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };
```

`ok` is the discriminant: checking it narrows the whole object. I also compared it with the looser shape
`{ ok: boolean; value?: T; error?: E }`:

- With the loose shape, `if (r.ok) return r.value * 2;` fails: `TS18048: 'r.value' is possibly 'undefined'`, and
  the same for `r.error`. I would need a `!` or an extra check at every use.
- The loose shape also *allows impossible values*: `{ ok: true, error: 'but ok is true' }` compiled. The
  discriminated version rejected the same literal (`TS2353: ... 'error' does not exist in type '{ ok: true;
  value: number; }'`).

`ok(value)` returns `Result<T, never>` and `err(error)` returns `Result<never, E>`, so each fits any
`Result<T, E>`. `E` defaults to `Error`. I did not try other designs, such as returning `[error, value]` tuples
or throwing.

**The toolkit port: what changed from M02 and what did not**

- It is **one file**, `async-toolkit.ts`, like M02's `async-toolkit.mjs`.
- `retry<T>(fn: (attempt: number) => T | Promise<T>, options?: RetryOptions): Promise<T>`,
  `withTimeout<T>(promise: Promise<T>, ms: number): Promise<T>` and `mapLimit<T, R>(items: Iterable<T>, limit:
  number, fn: (item: T, index: number) => R | Promise<R>): Promise<R[]>`.
- `RetryError` and `TimeoutError` declare their extra property as a **field** (`readonly attempts: number;`) and
  assign it in the constructor. The M02 shortcut, `constructor(public attempts: number)`, is a parameter
  property, which `erasableSyntaxOnly` rejects.
- The `RangeError` checks stay at runtime, even though the parameters are typed `number`. `number` also
  allows `-1`, `NaN` and `1.5`, which TypeScript cannot say "no" to, so the types and the runtime checks do
  different jobs.
- **`mapLimit` changed in one place.** M02 read `list[index]`. Under `noUncheckedIndexedAccess` that is `T |
  undefined`, and the literal port failed with `TS2345: Argument of type 'T | undefined' is not assignable to
  parameter of type 'T'`. I could not write `if (item === undefined) return`, because `T` itself might include
  `undefined`. A cast, `list[index] as T`, would have silenced the compiler without fixing anything. Instead
  the workers now take `[index, item]` pairs from one shared iterator, `list.entries()`. Iterating hands me a
  real `T`, so no cast is needed. The "never more than `limit` calls at once" and "no new work after a
  failure" checks pass unchanged.

**What I did with the 23 M02 checks**

I copied `async-toolkit.check.mjs` to a `.ts` file, changed the import specifier **and the file name inside a
string** (see below), and ran `tsc --noEmit` before fixing anything. It reported 11 errors (listed in "What tripped me up", item 3). The
fixes were annotations (`check(name: string, fn: () => void | Promise<void>)`, `const attempts: number[] =
[]`), a narrowing (`assert.ok(error.cause instanceof Error)`), and `// @ts-expect-error` on the three
lines that deliberately pass bad input (`sleep('soon')`, `{ delayMs: 'soon' }`). In M02 those lines tested that
the *runtime* rejects bad input. In TypeScript the compiler now rejects them too, and the checks assert both: the
`@ts-expect-error` proves the compiler objects, and the `assert.throws` proves the runtime still does if a
plain JavaScript caller slips past the types. `@ts-expect-error` is itself checked: an unused one is an error.

The M02 file names itself twice: in the `import`, and in `new URL('./async-toolkit.mjs', import.meta.url)` inside a string that
the "does not keep the process alive" check hands to a child Node process. `tsc` cannot see inside a string, so changing only the
import looked fine to the compiler and then `tsx` failed with `ERR_MODULE_NOT_FOUND`. A code review found that my first
write-up of this step wrongly said to change only the import. I also widened one M02 timing check: the `retry` check waited 40 ms twice and
allowed only 35 ms of slack above the expected 80 ms, which a busy machine could break. It now waits 100 ms twice, with bounds of 190 ms
and 280 ms, and I confirmed it still fails (302.7 ms) when `retry` is changed to wait after the last attempt as well.

**What happened when I attempted native type stripping on Node 24**

It worked, which I did not expect from the brief's wording ("awareness-level ... full stability arrives only
with Node 26"). On Node v24.22.0, `node src/toolkit/async-toolkit.check.ts` printed all 23 `ok` lines and
exited 0, with no flag and no warning. `npm run check:native` ran all 40 checks that way, and
`node --env-file-if-exists=.env src/demo.ts` ran the demo. `process.features.typescript` is `"strip"`. I did not
look up Node's documented stability label for the feature, so I can say what I observed, not whether Node 24
calls it stable.

It worked because my code follows three rules. Where I broke one, native Node failed, with real messages:

| File | What it contains | `node file.ts` on Node 24.22.0 |
| --- | --- | --- |
| `enum Color { Red, Green }` | an enum | `SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript enum is not supported in strip-only mode` |
| `constructor(public x: number)` | a parameter property | `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX: ... parameter property is not supported in strip-only mode` |
| `import { Point, origin } from './lib.ts'` (`Point` is a type) | a type imported without `type` | `SyntaxError: The requested module './lib.ts' does not provide an export named 'Point'` |
| `import { origin } from './lib'` or `'./lib.js'` | no extension, or `.js` for a `.ts` file | `ERR_MODULE_NOT_FOUND` |
| `const n: number = 'not a number'` | a type error | **runs** and prints `not a number` |

The last row is the important one. Native stripping just deletes the types. It never checks them.
`node --experimental-transform-types` handled the `enum` (it printed `0`) but printed `ExperimentalWarning:
Transform Types is an experimental feature and might change at any time`. On Node 22.23.0 I compared only two
files: the file using `import { type Point, origin }` ran, and the file importing `Point` without `type` failed
with the same `SyntaxError`.

**`tsx` against native stripping**

I ran the same seven probe files through `tsx` 4.23.15. **All seven ran**, including the enum (`0`), the
parameter property (`1`), the type imported without `type`, the extensionless import and the `.js`
specifier. `tsx` also does not check types: the `const n: number = 'not a number'` file printed `not a
number`. So both runners are only runners. The difference is that `tsx` is forgiving and native Node is strict
about what it will strip. That is why this repo uses `tsx` as the course's dev runner and `erasableSyntaxOnly`
as the guard that keeps code portable to native stripping.

**TypeScript 6 against 7, checked rather than assumed**

I installed `typescript@6` (6.0.3) in a scratch folder and ran the same probes against 7.0.2:

- **Strict by default is not a 6 → 7 change in what I saw.** 6.0.3, given a `tsconfig` with no `strict`, reported
  the same two errors as 7.0.2. The brief lists "strict by default" among the changes from 6 to 7, so the
  EM may want to check that wording against the official release notes. I only know what these two installs did.
- **Legacy options: deprecated in 6, removed in 7.** `target: es5`, `moduleResolution: node10`, `baseUrl`,
  `downlevelIteration` and `esModuleInterop: false` each gave `TS5107` or `TS5101` in 6.0.3 ("is deprecated and
  will stop functioning in TypeScript 7.0. Specify compilerOption '"ignoreDeprecations": "6.0"' to silence this
  error"). In 7.0.2 the same options gave `TS5108` or `TS5102`: "has been removed. Please remove it from your
  configuration." For `baseUrl`, 7 adds `Use '"paths": {"*": ["./*"]}' instead`.
- **Other behaviour I hit on 7.0.2:** emitting needs an explicit `rootDir` (`TS5011`, with a link to
  `aka.ms/ts6`), and `types` must list `@types` packages (above). I ran both only on 7.0.2, so I cannot say
  whether either is new in 7.
- **The programmatic API is gone from the main entry point.** `import('typescript')` in 6.0.3 has
  `createProgram` and `transpileModule` (`typeof` both give `function`). In 7.0.2 the module's keys are
  `default`, `module.exports`, `version` and `versionMajorMinor` (the first two are module-interop names, so the real exports are the last
  two), and `typeof` both of those functions gives `undefined`. The package's
  `exports` map instead lists `./unstable/sync`, `./unstable/async`, `./unstable/ast` and others, and the
  install pulled in a native binary package, `@typescript/typescript-darwin-arm64`. Any tool that calls
  `createProgram` or `transpileModule` has to find another route. I have not checked which tools in M09 do, so
  I cannot say what M09 will need.
- **Speed:** `tsc --noEmit` on a copy of this ~700-line project took 0.23 s on 6.0.3 and 0.04 s on 7.0.2, three
  runs each. A project this small is not a benchmark, but the gap was the same on every run.

**Typing `process.env`, caught errors and async code**

- `process.env.PORT` is `string | undefined`: `const port: number = process.env.PORT` fails with `TS2322: Type
  'string | undefined' is not assignable to type 'number'`. `readEnv()` reads the variable once, validates it,
  and returns a `Result<Env, string>`, so the rest of the program never sees `undefined`. It takes the env
  object as a parameter, defaulting to `process.env`, so the checks can pass a plain object.
- `catch (error)` gives `unknown` under `strict` (`TS18046: 'e' is of type 'unknown'` when I used `e.message`).
  `toError(thrown: unknown): Error` turns anything caught into an `Error` once. Its first version called
  `String(thrown)` unguarded, which itself throws for `Object.create(null)`, so `attempt`, which is meant never to
  throw, could reject. `String()` is now inside a `try`/`catch` with a fixed fallback message, and the original value
  stays as `cause`.
- `readEnv` first used `Number(raw)`, which turns a whitespace-only value into `0` and accepts `'0x10'` and `'1e3'`. It now trims the
  value, treats an empty or whitespace-only result as unset (default 20), and accepts only whole digits (`/^\d+$/`). The error
  message is `TOOLKIT_DELAY_MS must be a whole number of milliseconds, got "..."`.
- An `async` function's declared return is `Promise<T>`: `async function f(): Promise<number> { return 'x' }`
  fails with `TS2322` on the `return`, and `const w: number = f()` fails with `Type 'Promise<number>' is not
  assignable to type 'number'`.

## How to build it (teach it to the next trainee)

My worked example is deriving the Task input types with utility types, because it shows the reasoning and not
just the syntax. The other steps follow in the order I did them.

Start each session with `cd ~/node-js-foundation-training/modules/m04-typescript-on-node`, run `nvm use` on its
own line, and check that `node --version` prints v24. Run `nvm use` in a folder that has no `.nvmrc` and it
does nothing, so do it in the repo.

**0. Branch and folder.** `git switch main`, `git pull`, `git switch -c feat/module-04/typescript-on-node`,
`mkdir -p modules/m04-typescript-on-node/src/toolkit modules/m04-typescript-on-node/src/domain`.

**1. `package.json` and the install.**
- *Do:* write a `package.json` with `"private": true`, `"type": "module"`, `"engines": { "node": ">=24" }` and
  the scripts. Then `npm install --save-dev typescript@7 tsx @types/node@24`.
- *Why:* `@types/node@24` matches the Node 24 runtime, so the compiler knows the APIs that exist.
- *Result:* `added 7 packages ... found 0 vulnerabilities`, `npx tsc --version` printed `Version 7.0.2`. npm also
  warned that `esbuild@0.28.2` has a `postinstall` script not covered by `allowScripts`. `tsx` ran fine without
  my approving it.

**2. `tsconfig.json`, and break it on purpose before porting anything.**
- *Do:* write the config from "Why it's built this way". Then, before any real code, write a scratch file with
  six deliberate mistakes and run `npx tsc --noEmit`:
  ```ts
  function double(n: number) { return n * 2; }
  double('4');                          // wrong argument type
  const user = { name: 'Ada' };
  console.log(user.age);                // unknown property
  const list: string[] = ['a'];
  const first: string = list[0];        // index access may be undefined
  function parse(x) { return x; }       // implicit any
  enum Color { Red }                    // not erasable
  try { /* ... */ } catch (e) { console.log(e.message); }   // e is unknown
  ```
- *Why:* the first time you read a compiler error should not be when you are stuck on real code. Delete the
  file afterwards.
- *Result:* six errors: `TS2345`, `TS2339`, `TS2322` (`Type 'string | undefined' is not assignable to type
  'string'`), `TS7006`, `TS1294` and `TS18046`. Each is one line: file, `(line,column)`, the code, then a
  sentence.

**3. The Task types (the worked example).** Start from the domain type and ask, for each field, *who owns it?*

```ts
export const TASK_STATUSES = ['todo', 'in_progress', 'done'] as const;
export type TaskStatus = (typeof TASK_STATUSES)[number];

export type Task = {
  readonly id: string;
  title: string;
  description?: string;
  status: TaskStatus;
  readonly createdAt: Date;
  updatedAt: Date;
};
```

| Field | Who sets it | On create | On update |
| --- | --- | --- | --- |
| `id` | the server | not allowed | not allowed |
| `title` | the client | required | optional |
| `description` | the client | optional | optional |
| `status` | the client | optional (defaults to `todo`) | optional |
| `createdAt`, `updatedAt` | the server | not allowed | not allowed |

- *Create input = Task minus what the server owns, with `status` made optional.* Take out `id`, the two
  timestamps **and** `status`, then add `status` back as optional:
  ```ts
  export type CreateTaskInput = Omit<Task, 'id' | 'createdAt' | 'updatedAt' | 'status'> &
    Partial<Pick<Task, 'status'>>;
  ```
  `Omit` keeps `title` required and `description` optional because it copies each property as it was. `Pick` and
  `Partial` are only needed for `status`, because it is the one field whose requiredness changes.
- *Update input = any subset of what a client can set:*
  ```ts
  export type UpdateTaskInput = Partial<CreateTaskInput>;
  ```
  Defining it from `CreateTaskInput`, not `Task`, is what keeps `id` out of it.
- *Why derive:* when `Task` gains a field, the input types follow, and the rule "the server owns `id` and the
  timestamps" lives in one place. Read the cost of `Omit` in "Why it's built this way": new fields are
  allowed by default.
- *Check that the types say what you meant,* with type-level assertions in `domain.check.ts`:
  ```ts
  type Equal<A, B> =
    (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;
  type Expect<T extends true> = T;

  type _CreateKeys = Expect<Equal<keyof CreateTaskInput, 'title' | 'description' | 'status'>>;
  type _UpdateIsAllOptional = Expect<Equal<{} extends UpdateTaskInput ? true : false, true>>;
  ```
  and with `// @ts-expect-error` on lines that must fail (`createTask('t4', { title: 'x', id: 'client-chose-this' })`,
  `task.id = 'other'`).
- *Prove the assertions can fail.* I changed `Partial<Pick<Task, 'status'>>` to `Pick<Task, 'status'>` and
  `tsc` reported `domain.check.ts(28,39): error TS2344: Type 'false' does not satisfy the constraint 'true'`
  plus errors at every call site that omitted `status`. Then I put it back.
- *Result:* `npm run typecheck` printed nothing and exited 0.

**4. `Result<T, E>` and its helpers (`domain/result.ts`).** Write the union above, then `ok`, `err`, `toError`,
`attempt` and the two small helpers. *Why `toError`:* a `catch` gives `unknown`, and `unknown` is not
`Error`. Convert it once, and keep the original as `cause` if it was not an `Error`. Guard `String(thrown)` with a
`try`/`catch`, because it can throw. *Result:* `tsc` was clean on the first run.

**5. The toolkit port (`toolkit/async-toolkit.ts`).** Copy the M02 file, rename it `.ts`, and add the types
function by function. Run `npx tsc --noEmit` after each. *Do not fix errors by casting.* Expect exactly one:
`mapLimit`, from `noUncheckedIndexedAccess`. *Result:* one error until the shared-iterator change, then
`tsc` printed nothing and exited 0.

**6. The ported checks (`toolkit/async-toolkit.check.ts`).** Copy the M02 check file and change every mention of
the old file name (`grep -n "async-toolkit.mjs"` finds two: the import and the one inside the child-process string), run
`tsc`, and fix the 11 errors. *Result:* `tsc` exit 0 and 23 `ok` lines.

**7. Run it three ways and read what each does.**
```bash
npx tsx src/toolkit/async-toolkit.check.ts      # 23 ok lines, exit 0
node src/toolkit/async-toolkit.check.ts         # 23 ok lines, exit 0, no flag, no warning
npm run typecheck                               # no output, exit 0
```
Then try the failures: an `enum`, a parameter property, and a type imported without `type` (the table in "Why
it's built this way"). *Why:* so you know what the failure looks like before you meet it in your own code.

**8. `env.ts`, the checks for it, and the demo.** Add `readEnv`, its three checks, and `demo.ts`.
`npm run demo` printed (the first line appears only while no `.env` exists):
```text
.env not found. Continuing without it.
task-1  [todo]  Set up tsconfig
task-2  [in_progress]  Port the toolkit
task-3  [todo]  Model Result<T, E>
task-4  [todo]  Write the guide
task-5  [todo]  Open the PR
retry: Flaky save saved
slow: TimeoutError: Timed out after 20ms
unwrapOr: fallback value
```
and `TOOLKIT_DELAY_MS=soon npm run demo` printed `TOOLKIT_DELAY_MS must be a whole number of milliseconds, got "soon"` and
exited 1. The `demo` script is `node --env-file-if-exists=.env --import tsx src/demo.ts`: one Node process with `tsx` loaded as an
import hook.

**9. Branch, commit, pull request, merge.** The same habit as M01–M03.
- *Do:* re-run the gates (`npm run typecheck`, `npm run check`, `npm run check:native`, `npm run demo`), then commit in
  three groups, staging files by name: `chore(m04): add package.json, tsconfig, lockfile and env example`,
  `feat(m04): port the async toolkit to TypeScript with generics`, and `feat(m04): add Task domain types, Result,
  typed env and demo`. Push with `git push -u origin feat/module-04/typescript-on-node`, then open the pull request with
  `gh pr create --repo kiannaalexandra-gragg/node-js-foundation-training --base main --head
  feat/module-04/typescript-on-node`, with Summary, Changes and Test plan sections. Merge it on GitHub with a merge commit.
- *Why:* the project files go first because the scripts in `package.json` point at the files in the later commits. Each
  group is one concern. The commits carry no `Co-Authored-By` trailer. The code review of both repos ran *before* the commit, so its
  fixes are in the first version of each commit.
- *Result:* PR #4, merged (see Checkpoint evidence). `node_modules`, `.env` and `scratch/` stayed out of the commits.

## Concepts worth explaining

**`unknown`, `any` and `never`**

`any` turns the type checker off for a value. `unknown` means "I do not know what this is yet, so prove it before
you use it". They look alike from the outside, since both accept anything, and they differ in what you may do
*afterwards*:

```ts
const a: any = JSON.parse('{"n":1}');
const u: unknown = JSON.parse('{"n":1}');
a.nope.deeper();        // compiles. At runtime: TypeError: Cannot read properties of undefined (reading 'deeper')
u.n;                    // TS18046: 'u' is of type 'unknown'
const m: number = u;    // TS2322: Type 'unknown' is not assignable to type 'number'
```

Nothing told me `a.nope.deeper()` was wrong until it ran. `any` also *spreads*: `takesNumber(a)` and
`const n: number = a` compiled too, so one `any` can switch off the checking of everything it touches. With
`unknown` the only way in is to narrow first:

```ts
if (typeof u === 'object' && u !== null && 'n' in u) { u.n; }   // compiles
```

So I use `unknown` for anything that comes from outside the program (`JSON.parse`, a `catch` clause,
`process.env`, a request body) and narrow it once, at the edge. The code standards say the same: prefer `unknown`,
and if `any` appears, say why. I did not need an `any` anywhere in the build.

`never` is the type with no values. I met it two ways. A function that always throws, or never returns, has
return type `never`, which is why `process.exit()` made the compiler narrow `env` after the `if`. And a
`switch` over a union can end with `const unreachable: never = s;`. When every case is handled, `s` is `never` there
and the line compiles. When I added a fourth status, `'blocked'`, the same line became `TS2322: Type '"blocked"'
is not assignable to type 'never'`, so adding a union member breaks every switch that forgot it. That is the
compiler giving me a to-do list.

**`type` and `interface`**

Both describe the shape of an object, and for plain object shapes they are close to interchangeable. I tried
the places where they are not:

- **Only `type` can name a union, a tuple or a mapped type.** `type Id = string | number` and `type Pair =
  [string, number]` compile. `interface Id2 = string | number` is a syntax error (`TS1005`), and a mapped
  type inside an `interface` fails with `TS7061: A mapped type may not declare properties or methods`. My
  `TaskStatus`, `Result<T, E>` and the utility-type results are all of those, so they must be `type`.
- **Only `interface` merges.** Two `interface Shape` declarations become one type with both properties
  (`{ name: 'sq' }` failed with `Property 'sides' is missing`). Two `type Pt` declarations are `TS2300:
  Duplicate identifier 'Pt'`. Merging is how a library lets you add to its types, and it is also a way to
  change a type by accident from another file.
- **They disagree about conflicts.** `interface B extends A { v: number }` where `A.v` is `string` is reported at
  the declaration (`TS2430: Interface 'B' incorrectly extends interface 'A'`). `type C = A & { v: number }`
  compiled without a complaint, and the problem showed up later, as `v` having type `never`
  (`TS2322: Type 'number' is not assignable to type 'never'`). Errors are earlier and clearer with `extends`.

My rule: use `type` by default, because I need unions and derived types constantly, and `interface` when I
want a named object shape that others extend or merge into. Everything in this module's domain is a `type`.
I did not test claims about performance differences between them, so I make none.

## What tripped me up

1. **`nvm use` quietly did nothing, twice.** The first time I wrote `nvm use | tail -2` to keep the output
   short. It printed `Now using node v24.22.0`, but `node -v` right after still said `v22.23.0`, because a pipe
   runs `nvm use` in a subshell. The second time I ran a probe in a scratch folder that has no `.nvmrc`: `nvm use`
   printed `No .nvmrc file found` and my "native type stripping" results were from Node 22. I only noticed
   because I printed `node -v`. I re-ran everything with `nvm use 24` and the version printed.
2. **The one real error in the port was `mapLimit`:** `TS2345: Argument of type 'T | undefined' is not
   assignable to parameter of type 'T'. 'T' could be instantiated with an arbitrary type which could be unrelated
   to 'T | undefined'.` My first instinct was `as T`. I did not use it: it makes the error go away without making
   the code safer. Reading the second line of the message helped. The compiler is saying `T` could itself be
   `undefined`, so checking for `undefined` would be wrong too.
3. **Eleven errors in the ported check file, in four groups:**
   - `TS7006` (twice): `check(name, fn)` parameters implicitly `any`. A function with untyped parameters is the
     first thing strict mode finds.
   - `TS7034` and `TS7005`: `const attempts = []` is "implicitly `any[]` in some locations where its type
     cannot be determined" (three arrays, six errors). An empty array literal has no element type until you say
     one: `const attempts: number[] = []`.
   - `TS18046: 'error.cause' is of type 'unknown'`: `cause` is `unknown` by definition. Narrow with
     `assert.ok(error.cause instanceof Error)` first. Note that `error.ms` on the next check did *not* error,
     because the callback parameter of `assert.rejects` is typed `any`, a reminder that `any` hides problems.
   - `TS2345` and `TS2322` on `sleep('soon')` and `{ delayMs: 'soon' }`: these are the *tests* passing bad input on
     purpose. The fix is `// @ts-expect-error`, not changing the test.
4. **`tsx` and native stripping are different tools with the same-looking command.** `tsx file.ts` accepted
   everything I threw at it (enum, parameter property, a type imported without `type`, `.js` and bare
   specifiers). `node file.ts` rejected five of the seven probe files. Neither one checked a single type. If
   `node file.ts` works, it proves only that the syntax is erasable, not that the types are right: only
   `tsc --noEmit` does that.
5. **A type imported without `type` is not a compile error under `tsx`, but it is a runtime crash under native
   Node.** `import { Point, origin } from './lib.ts'` ran under `tsx`, but native Node threw `SyntaxError: The
   requested module './lib.ts' does not provide an export named 'Point'`, because Node deleted the types
   but left the import of a name that no longer exists. With `verbatimModuleSyntax` on, `tsc` reports it first, as
   `TS1484: 'Point' is a type and must be imported using a type-only import`. That is the option doing its job.
6. **A `.js` specifier passes `tsc` and `tsx` and fails in native Node.** `import ... from './lib.js'` with only
   `lib.ts` on disk: `tsc` exit 0, `tsx` ran, native Node `ERR_MODULE_NOT_FOUND`.
7. **Reading compiler errors.** A message is `file(line,column): error TScode: sentence`. When there are
   indented lines under it, the last indented line is the most specific. In `TS2430: Interface 'B' incorrectly
   extends interface 'A'. Types of property 'v' are incompatible. Type 'number' is not assignable to type
   'string'`, the first line says what is wrong and the last says where. One mistake can also cause many
   errors: when I made `status` required, `tsc` printed nine errors, eight of them at call sites. Fix the
   definition, then re-run, instead of fixing each site.
8. **Smaller things.**
   - Without `"types": ["node"]`, even `console` was unknown, and `process.exit(1)` stopped narrowing.
   - `tsc` stopped with `TS5011` in my build probe until I set `rootDir`.
   - My first `demo` script was `tsx --env-file-if-exists=.env src/demo.ts`. It printed `.env not found. Continuing
     without it.` twice under `npm run demo` (three times when I ran `npx tsx` directly), because `tsx` starts further Node
     processes and each one gets the flag. It was harmless, but noisy, so the script is now `node
     --env-file-if-exists=.env --import tsx src/demo.ts`, which prints it once.
   - My first `never` probe forgot the `done` case, and the error (`Type '"done"' is not assignable to type
     'never'`) was the exhaustiveness check working on my own mistake.
   - In my first `unknown` probe I had set `"types": []`, so `console` was missing and the real errors were
     buried under `TS2584`. A tsconfig for a scratch probe still needs the types it uses.
9. **A code review found four things my checks had not.** All four passed `tsc` and every check I had written:
   - `applyTaskUpdate`'s spread let `undefined` blank a field and let extra keys overwrite `id` (see "Why `applyTaskUpdate` names
     every field").
   - `toError` threw for `Object.create(null)`, so `attempt` could reject.
   - `readEnv` accepted a whitespace-only value as `0`.
   - The ported check file still named `./async-toolkit.mjs` inside a string, which only a run reveals.

   Two of these were also holes in my *checks*: the type-level assertions covered `CreateTaskInput` but not
   `keyof UpdateTaskInput`, and no check passed an object with extra keys or an unconvertible thrown value. A passing check
   shows only what it asserts.

## Checkpoint evidence

**The build merged to `main` by pull request.**
[#4, `feat(m04): port the async toolkit to TypeScript and add the Task domain types`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/4),
from `feat/module-04/typescript-on-node` into `main`. It was merged on 2026-10-21 (00:29 UTC) with a merge commit, so the three
commits are visible in the history:

```text
$ git log --oneline --graph origin/main -6
*   aa0aac0 Merge pull request #4 from kiannaalexandra-gragg/feat/module-04/typescript-on-node
|\
| * 7aac72b feat(m04): add Task domain types, Result, typed env and demo
| * 6cb2563 feat(m04): port the async toolkit to TypeScript with generics
| * 470748c chore(m04): add package.json, tsconfig, lockfile and env example
|/
*   f26878b Merge pull request #3 from kiannaalexandra-gragg/feat/module-03/runtime-and-npm
```

`gh pr view 4` reported the state `MERGED`, with the base `main`, the head `feat/module-04/typescript-on-node`, 13 changed
files and those three commits. The test plan in the PR lists the commands whose results are shown below.

**Zero `tsc --noEmit` errors on a strict config.** Captured on the final code, Node v24.22.0, TypeScript 7.0.2:

```text
$ npm run typecheck

> m04-typescript-on-node@1.0.0 typecheck
> tsc --noEmit

$ echo $?
0
```

The same config does catch mistakes. The six deliberate errors from step 2 and the mutation test from step 3
(`TS2344`) are real runs of it.

**The toolkit running via `tsx`:** `npm run check` exited 0 with 40 `ok` lines (23 toolkit, 14 domain, 3 env),
for example:

```text
ok - sleep waits at least the requested time
ok - retry gives up with a RetryError that keeps the last error as cause
ok - withTimeout does not keep the process alive after the promise settles
ok - mapLimit rejects with the first error and starts no new work after it
ok - Result: checking `ok` narrows to value or error
ok - readEnv returns an error Result for a bad value instead of throwing
```

and `npm run demo` printed the lines shown in "How to build it", step 8.

**What happened when I attempted native type stripping on Node 24:** it ran. `node
src/toolkit/async-toolkit.check.ts` printed the same 23 `ok` lines as `tsx`, with no flag and no warning, and
`npm run check:native` ran all 40 checks and exited 0. The failure cases are in the table under "Why it's built
this way".

**My explanations of `unknown` and `any`, and of `type` and `interface`:** see "Concepts worth explaining".

**Not yet covered:**
- How long the stage took.
- Running any of this on Windows (macOS and zsh only).
- Anyone else following this path cold.
- Whether Node 24's documentation calls native type stripping stable. I observed behaviour only.
- `tsc` as a real build of this project. I built a two-file probe only.

## What I'd do differently

- Decide `Omit` against `Pick` before writing `CreateTaskInput`, by asking "should a new `Task` field be
  writable by clients by default?". I only discovered the deny-list cost afterwards, in the `dueDate`
  experiment.
- For every function that takes a partial or untrusted object, write one check that passes values the *type* allows but the
  function should not honour (`undefined` fields, extra keys, a value that cannot be turned into a string). My first checks
  only passed well-formed input.
- Print `node -v` in every command that is about a Node version, and put a `.nvmrc` (or an explicit `nvm use 24`) in
  any scratch folder. Two of my results were briefly from Node 22.
- Run the deliberate-error file (step 2) *before* writing `tsconfig.json` options one by one, and read each error
  against the option that produced it. I did it in one go and had to go back to attribute each error.
- Check each claim in the brief against the installed tools early. Two things in its wording did not match
  what I ran: native stripping being only awareness-level on Node 24 (it ran), and strict-by-default being
  listed as a 6 → 7 change (6.0.3 already did it).
