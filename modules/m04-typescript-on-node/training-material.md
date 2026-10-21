# M04 — TypeScript Basics & TypeScript on Node.js: Training Material

**Week 2 · Days 1-2 · 8 hours**

A "module" in this program means a training stage. "ES module" means a JavaScript module file.

## Builds on

M02 — Modern JavaScript & Async/Await, and M03 — Node.js Runtime & npm. You port your **M02 async toolkit**
(`sleep`, `retry`, `mapLimit`, `withTimeout`) to TypeScript, so have that file and its checks open. You also need
your build repo from M01–M03, Node 24 pinned with `.nvmrc`, and the branch → pull request → merge habit. Everything
from this stage onward is TypeScript.

## 1. What you'll build and why

You will learn to read and write strict TypeScript, and to set up a TypeScript project that type-checks and runs on
Node.js. You finish with:

- your **M02 toolkit ported to TypeScript with generics**: `retry<T>`, `mapLimit<T, R>`, `sleep` and `withTimeout<T>`,
- the **Task domain types**: `Task`, `CreateTaskInput` and `UpdateTaskInput` (derived from `Task` with utility types,
  not written by hand), and a discriminated-union **`Result<T, E>`**,
- **zero errors from `tsc --noEmit`** on a strict config, wired up as `npm run typecheck`,
- the toolkit **running through `tsx`**, and an **attempt to run it through Node's native type stripping** with a note of
  what actually happened on Node 24,
- your own **explanation of `unknown` against `any`, and of `type` against `interface`**.

TypeScript does not run your code. It checks it, and the runtime still runs plain JavaScript. Keeping those two jobs
apart (a *checker* and a *runner*) is the main idea of this stage, and every tool below is one or the other.

**In your repo:** the project files, the ported toolkit, the domain types, their checks and a small demo, in
`modules/m04-typescript-on-node/`. The practice files for Stage 3, 8, 9 and 10 stay in a scratch folder and are not
delivered.

| File | Where | Built in |
| --- | --- | --- |
| `package.json` | your repo, stage folder | Stage 1 |
| `tsconfig.json` | your repo, stage folder | Stage 2 |
| `src/toolkit/async-toolkit.ts`, `async-toolkit.check.ts` | your repo | Stages 4 and 5 |
| `src/domain/task.ts` | your repo | Stage 6 |
| `src/domain/result.ts`, `domain.check.ts` | your repo | Stage 7 |
| `src/env.ts`, `env.check.ts`, `demo.ts`, `.env.example` | your repo | Stage 9 |
| `.env.example` (one line added) | your repo, **repo root** | Stage 9 |
| `package-lock.json` | your repo, stage folder | Stage 1 |
| `scratch/` (practice files, one folder per practice) | stage folder, **not committed** | Stages 3, 8, 9 |
| `src/scratch-errors.ts` (deleted straight after use) and `~/ts6-probe` (outside your repo) | stage folder, and your home folder | Stages 2 and 10 |

### How to use this guide

| Stage | What it is | How to treat it |
| --- | --- | --- |
| 1 | Branch, folder, project files, install | Core |
| 2 | `tsconfig.json`, and read errors on purpose | Core |
| 3 | Core TypeScript by experiment (six short files) | Core: run each, then write your own explanations |
| 4 | Port the toolkit | Core: you write it |
| 5 | Port the toolkit's checks | Core |
| 6 | Model the Task domain | Core: you write it |
| 7 | Model `Result<T, E>`, then the domain checks | Core: you write it |
| 8 | Run it three ways: `tsx`, native stripping, `tsc` build | Core 1-3. The `tsc` build (4) is awareness only. |
| 9 | Async, errors, `process.env`, `@types` and `.d.ts` | Core |
| 10 | TypeScript 6 → 7 | In scope: go through once |
| 11 | The gate, commit, pull request, merge | Core |

- **Guided path** (section 3): what to do, run and check.
- **Concepts** (section 4): a quick reference, then short essays. Read each essay after the stage named beside it.
- **Common pitfalls** (section 5): look here when something misbehaves.
- **Definition of done** (section 6): the final check.

**Stages 4, 6 and 7 are exercises.** Each has *what it must do*, *your turn* (a scaffold that already type-checks, with
hints you open one at a time), and a *reference implementation* to open after you have tried. Stages 1-3, 8-10 give you
things to run, because there the learning is in what you observe.

## 2. Before you start

- **Your build repo** (M01-M03) on `main`, with `.nvmrc` containing `24` and a root `.gitignore` that ignores
  `node_modules/` and `.env`.
- **Your M02 toolkit and its checks** (`async-toolkit.mjs` and `async-toolkit.check.mjs` in the reference build; use
  your own file names). Stage 4 has a finished reference port in case yours is missing, but port your own.
- **Node 24 active.** Run `nvm use` **on its own line** in the repo root and check that `node --version` prints `v24.x`.
  Do this every session. A new terminal can start on an older Node, `nvm use` in a folder with no `.nvmrc` does nothing,
  and `nvm use | tail` in a pipe does not switch the version at all. Always print the version when a result depends on it.
- **Tools:** internet for npm, plus `git` and `gh` (Stage 11).
- **Pinned:** Node.js 24 LTS; TypeScript 7.0 (the reference build used 7.0.2); `@types/node@24`; `tsx` as the dev
  runner. The compiler messages quoted here are from TypeScript 7.0.2, and their wording can differ in other
  patch versions. No Express, Prisma, Zod or Jest yet.
- **Import extensions, an open decision.** Whether TypeScript projects import `./file.ts` or `./file.js` is still an open
  program decision ("P2"). **This guide uses `.ts` specifiers with `allowImportingTsExtensions`, and that is its choice for
  this stage, not the program's rule.** Stage 8 shows exactly what each choice does in `tsc`, `tsx` and native Node.
- **File headers:** start each file with a short comment: `Module 4 output`, what the file is, how to run it, and what
  success looks like.
- **No `enum`:** this course writes unions of string literals instead. Stage 2 shows the compiler enforcing that.

Commands are for macOS or Linux and have not been tried on Windows. Run commands from the stage folder,
`~/<repo-name>/modules/m04-typescript-on-node`, unless a step says otherwise.

## 3. Guided path

### Stage 1 — Create the branch, the folder and the project files

**Goal:** a folder with a `package.json`, the compiler, a runner and Node's types installed, on a branch of its own.

1. Switch to Node 24, on its own line, and check it:
   ```bash
   cd ~/<repo-name>
   nvm use
   node --version        # v24.x
   ```
2. Branch and folders (every stage is delivered from a feature branch, never from `main`):
   ```bash
   git switch main && git pull
   git switch -c feat/module-04/typescript-on-node
   mkdir -p modules/m04-typescript-on-node/src/toolkit modules/m04-typescript-on-node/src/domain
   cd modules/m04-typescript-on-node
   ```
3. Create `package.json`. The scripts point at files you create later, as in M03:

   ```json
   {
     "name": "m04-typescript-on-node",
     "version": "1.0.0",
     "private": true,
     "description": "M04: the M02 async toolkit ported to strict TypeScript, plus the Task domain types",
     "type": "module",
     "engines": {
       "node": ">=24"
     },
     "scripts": {
       "typecheck": "tsc --noEmit",
       "demo": "node --env-file-if-exists=.env --import tsx src/demo.ts",
       "check": "tsx src/toolkit/async-toolkit.check.ts && tsx src/domain/domain.check.ts && tsx src/env.check.ts",
       "check:native": "node src/toolkit/async-toolkit.check.ts && node src/domain/domain.check.ts && node src/env.check.ts"
     },
     "devDependencies": {
       "@types/node": "^24.19.2",
       "tsx": "^4.23.15",
       "typescript": "^7.0.2"
     }
   }
   ```

   - `"type": "module"` makes `.ts` files ES modules. `npm init -y` writes `"type": "commonjs"`, so set it by hand.
   - `typecheck` is the **gate**: `tsc --noEmit` checks every type and writes nothing.
   - `check` runs the three check files through `tsx`. `check:native` runs the same files with plain `node`
     (Stage 8). `demo` loads `.env` if it exists (Stage 9).
4. Install the three development dependencies. Check what is current first:
   ```bash
   npm view typescript dist-tags
   npm install --save-dev typescript@7 tsx @types/node@24
   npx tsc --version
   ```
   - `typescript` is the **checker**. `tsx` is a **runner** that executes `.ts` files directly. `@types/node@24` tells the
     checker what Node 24's built-in modules and globals look like, so its major version should match your Node.
   - In the reference build the install printed `added 7 packages ... found 0 vulnerabilities`, and `tsc --version` printed
     `Version 7.0.2`. npm also warned that `esbuild` (a dependency of `tsx`) has a `postinstall` script not covered by
     `allowScripts`. `tsx` ran without it being approved. Read an install-script warning, don't approve it blindly.
5. Check that git will not take `node_modules`: `git status -sb` should show the branch and the stage folder as untracked,
   with no `node_modules/`.

**Done when:**

- [ ] `node --version` prints `v24.x`, and `git branch --show-current` prints `feat/module-04/typescript-on-node`.
- [ ] `package.json` exists in the stage folder with `"type": "module"`, an `engines` field for Node 24, and the four scripts.
- [ ] `npx tsc --version` prints a `Version 7.x` line, and `npm ls @types/node` shows version `24.x`.

### Stage 2 — Write the `tsconfig.json`, then break things on purpose

**Goal:** a strict TypeScript config, and the experience of reading compiler errors before real code depends on it.

1. Create `tsconfig.json`:

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

   Each option, and why it is there:

   | Option | What it does here |
   | --- | --- |
   | `strict` | Turns on the whole family of strict checks (no implicit `any`, `null` and `undefined` are tracked, caught errors are `unknown`). TypeScript 7 is strict even when this is missing; it is written out so the config states the intent. |
   | `module`, `moduleResolution: nodenext` | Use Node's own rules for ES modules, so the compiler and the runtime agree about file extensions and `package.json` `"type"`. |
   | `noUncheckedIndexedAccess` | `list[0]` has type `T \| undefined`, because the index may be out of range. |
   | `verbatimModuleSyntax` | An import stays exactly as written, so a type must be imported with `import type` (or an inline `type` modifier). |
   | `erasableSyntaxOnly` | Rejects TypeScript syntax that must be *rewritten* to become JavaScript (`enum`, constructor parameter properties). What is left can simply have its types deleted. |
   | `allowImportingTsExtensions` | Lets you write `import ... from './file.ts'`. It is normally paired with `noEmit`, since a compiler that writes `.js` files would leave those imports pointing at `.ts` files *(background)*. |
   | `noEmit` | `tsc` only checks. `tsx` and `node` run the code. |
   | `skipLibCheck` | Skips type-checking the `.d.ts` files inside installed packages, so only your code is checked. *(background)* |
   | `types: ["node"]` | **Required.** Without it `console` and `process` are "Cannot find name", even with `@types/node` installed. |
   | `target`, `lib: es2024` | `Promise.withResolvers`, used by `withTimeout`, is in the ES2024 library. With `lib: ["es2022"]` it is `TS2550`. |
   | `include: ["src"]` | Only `src/` is checked. Your `scratch/` practice files stay out of the gate. |

2. **Break it on purpose.** Create `src/scratch-errors.ts` with six mistakes and run the checker:

   ```ts
   import { readFileSync } from 'node:fs';

   function double(n: number) {
     return n * 2;
   }

   double('4');
   const user = { name: 'Ada' };
   console.log(user.age);
   const list: string[] = ['a'];
   const first: string = list[0];
   function parse(x) { return x; }
   enum Color { Red }
   try { readFileSync('x'); } catch (e) { console.log(e.message); }
   ```

   ```bash
   npx tsc --noEmit
   ```

   What you should see:

   ```text
   src/scratch-errors.ts(7,8): error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
   src/scratch-errors.ts(9,18): error TS2339: Property 'age' does not exist on type '{ name: string; }'.
   src/scratch-errors.ts(11,7): error TS2322: Type 'string | undefined' is not assignable to type 'string'.
     Type 'undefined' is not assignable to type 'string'.
   src/scratch-errors.ts(12,16): error TS7006: Parameter 'x' implicitly has an 'any' type.
   src/scratch-errors.ts(13,6): error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.
   src/scratch-errors.ts(14,52): error TS18046: 'e' is of type 'unknown'.
   ```

3. **Read one.** `src/scratch-errors.ts(11,7): error TS2322: ...` is *file, `(line,column)`, error code, a sentence*. The
   indented lines under it say why. The code (`TS2322`) is searchable. For each of the six errors, say what rule you broke
   and how you would fix it. The `enum` error (`TS1294`) is the compiler enforcing "union types, not `enum`". The
   `unknown` one is a caught value, which Stage 3 and the Concepts section explain.
4. Delete the file: `rm src/scratch-errors.ts`. (`tsc` with no files under `src` reports that it found no inputs. That is
   fine until Stage 4.)

**Done when:**

- [ ] `tsconfig.json` has `strict`, `module` and `moduleResolution` set to `nodenext`, `verbatimModuleSyntax`,
      `erasableSyntaxOnly`, `noUncheckedIndexedAccess`, `noEmit` and `skipLibCheck`.
- [ ] `npx tsc --noEmit` on the scratch file printed the six errors above (`TS2345`, `TS2339`, `TS2322`, `TS7006`, `TS1294`,
      `TS18046`), and `src/scratch-errors.ts` is deleted afterwards.
- [ ] You can explain, for three of the six errors, what the compiler is objecting to and how to fix it.

### Stage 3 — Core TypeScript, by experiment

**Goal:** see the compiler's behaviour for the core ideas, in six short files, before using them in the toolkit.

Set up a scratch folder whose config *extends* yours, so it has the same strict rules but is not part of the gate. Each practice
gets its own folder with its own one-line config, so the errors of one practice never appear in another:

```bash
mkdir -p scratch/core
echo '{ "extends": "../../tsconfig.json", "include": ["*.ts"] }' > scratch/core/tsconfig.json
```

You check the files with `npx tsc -p scratch/core`, and run one with `npx tsx scratch/core/<file>.ts`. **Every error in these
files is on purpose.** Each is on a line commented `// error`, or on a `const ask...: never` line, which is how you ask the
compiler what type it worked out: nothing is assignable to `never`, so the error message names the type. Create each file, run the
checker, and match what you see against what is below. The reference build printed 30 errors across the six files.

#### Core 1 — Inference, structure, arrays, tuples, unions, literals, `as const` (`scratch/core/core1-types.ts`)

```ts
// Core 1: inference, structural typing, arrays, tuples, unions, intersections, literals, as const.

// Inference: const keeps the exact value as the type, let widens it.
const fixed = 5;
let changing = 5;
changing = 'five';                            // error: a number variable cannot hold a string

// Ask the compiler what it inferred: assign to `never`, which nothing is assignable to.
// The error message names the type.
const askFixed: never = fixed;
const askChanging: never = changing;

// Structural typing: the shape matters, not the name.
type Point = { x: number; y: number };
const point3d = { x: 1, y: 2, z: 3 };
const fine: Point = point3d;                  // fine: it has x and y, and a variable is not checked for extras
const tooMuch: Point = { x: 1, y: 2, z: 3 };  // error: a fresh object literal is checked for extra properties

// Arrays and tuples.
const names: string[] = ['ada', 'grace'];
const pair: [string, number] = ['ada', 1];
const triple: [string, number] = ['ada', 1, 2];   // error: a tuple has a fixed length
pair[2];                                          // error: no element at index 2

// Unions, intersections and literal types.
type Status = 'todo' | 'done';
const wrong: Status = 'finished';                 // error: not one of the two literals
type Named = { name: string };
type Aged = { age: number };
type Person = Named & Aged;                       // an intersection has both sets of properties
const missingAge: Person = { name: 'ada' };       // error: age is missing

// as const: freeze an array into a readonly tuple of literals, then derive a union from it.
const loose = ['todo', 'in_progress', 'done'];
const statuses = ['todo', 'in_progress', 'done'] as const;
type TaskStatus = (typeof statuses)[number];
const askLoose: never = loose;
const askStatuses: never = statuses;
const askUnion: never = null as unknown as TaskStatus;

export { fine, tooMuch, names, wrong, missingAge, askFixed, askChanging, askLoose, askStatuses, askUnion };
```

```bash
npx tsc -p scratch/core
```

What you should see for this file:

```text
scratch/core/core1-types.ts(6,1): error TS2322: Type 'string' is not assignable to type 'number'.
scratch/core/core1-types.ts(10,7): error TS2322: Type '5' is not assignable to type 'never'.
scratch/core/core1-types.ts(11,7): error TS2322: Type 'number' is not assignable to type 'never'.
scratch/core/core1-types.ts(17,38): error TS2353: Object literal may only specify known properties, and 'z' does not exist in type 'Point'.
scratch/core/core1-types.ts(22,7): error TS2322: Type '[string, number, number]' is not assignable to type '[string, number]'.
  Source has 3 element(s) but target allows only 2.
scratch/core/core1-types.ts(23,6): error TS2493: Tuple type '[string, number]' of length '2' has no element at index '2'.
scratch/core/core1-types.ts(27,7): error TS2322: Type '"finished"' is not assignable to type 'Status'.
scratch/core/core1-types.ts(31,7): error TS2322: Type '{ name: string; }' is not assignable to type 'Person'.
  Property 'age' is missing in type '{ name: string; }' but required in type 'Aged'.
scratch/core/core1-types.ts(37,7): error TS2322: Type 'string[]' is not assignable to type 'never'.
scratch/core/core1-types.ts(38,7): error TS2322: Type 'readonly ["todo", "in_progress", "done"]' is not assignable to type 'never'.
scratch/core/core1-types.ts(39,7): error TS2322: Type '"done" | "in_progress" | "todo"' is not assignable to type 'never'.
  Type '"done"' is not assignable to type 'never'.
```

Notice: `fixed` is the literal type `5` and `changing` is `number`. A fresh object literal is checked for extra properties but a
variable holding the same object is not. `as const` turns the array into a `readonly` tuple of literals, and
`(typeof statuses)[number]` turns that into a union, so the list of values and the type cannot drift apart.

#### Core 2 — Functions, generics, narrowing (`scratch/core/core2-functions.ts`)

```ts
// Core 2: functions, generics (functions, constraints, defaults), narrowing.

// A function: annotate parameters, let the return type be inferred.
function double(n: number) {
  return n * 2;
}
double('4');                                  // error: wrong argument type
const askDouble: never = double(2);           // the inferred return type shows in the error

// A generic function: T is decided by the call, so the result keeps the caller's type.
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}
const askFirst: never = first([1, 2, 3]);
const askFirstStr: never = first(['a', 'b']);

// A constraint: T must have at least an id.
function byId<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find((item) => item.id === id);
}
byId([{ id: 'a', n: 1 }], 'a');               // fine, and the result keeps the extra property n
byId([{ name: 'no id' }], 'a');               // error: does not satisfy the constraint

// A default: Box means Box<string> unless you say otherwise.
type Box<T = string> = { value: T };
const askBox: never = null as unknown as Box;

// Narrowing: typeof, in, Array.isArray, and equality.
function describe(v: string | number | string[] | { kind: string }): string {
  if (typeof v === 'string') return v.toUpperCase();      // v is string here
  if (typeof v === 'number') return v.toFixed(1);         // v is number here
  if (Array.isArray(v)) return v.join(',');               // v is string[] here
  return v.kind;                                          // only { kind } is left
}
function sizeOrKind(v: { kind: string } | { size: number }): string {
  if ('kind' in v) return v.kind;                         // the in operator narrows too
  return String(v.size);                                  // only { size } is left
}

// A discriminated union: one property ("kind") tells the variants apart.
type Shape = { kind: 'circle'; r: number } | { kind: 'rect'; w: number; h: number };
function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.r ** 2;             // s is the circle here
    case 'rect': return s.w * s.h;                        // s is the rect here
  }
}
function badArea(s: Shape): number {
  return s.r * 2;                                         // error: r exists only on one variant
}

// A user-defined type guard: `value is string[]` is a promise the compiler trusts.
function isStringArray(value: unknown): value is string[] {
  return Array.isArray(value) && value.every((item) => typeof item === 'string');
}

export { double, first, byId, describe, sizeOrKind, area, badArea, isStringArray, askDouble, askFirst, askFirstStr, askBox };
```

What you should see for this file:

```text
scratch/core/core2-functions.ts(7,8): error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
scratch/core/core2-functions.ts(8,7): error TS2322: Type 'number' is not assignable to type 'never'.
scratch/core/core2-functions.ts(14,7): error TS2322: Type 'number | undefined' is not assignable to type 'never'.
  Type 'undefined' is not assignable to type 'never'.
scratch/core/core2-functions.ts(15,7): error TS2322: Type 'string | undefined' is not assignable to type 'never'.
  Type 'undefined' is not assignable to type 'never'.
scratch/core/core2-functions.ts(22,9): error TS2353: Object literal may only specify known properties, and 'name' does not exist in type '{ id: string; }'.
scratch/core/core2-functions.ts(26,7): error TS2322: Type 'Box<string>' is not assignable to type 'never'.
scratch/core/core2-functions.ts(49,12): error TS2339: Property 'r' does not exist on type 'Shape'.
  Property 'r' does not exist on type '{ kind: "rect"; w: number; h: number; }'.
```

Notice: `first<T>` hands back the caller's type, `T extends { id: string }` accepts any object that has an `id`, and each `if`
in `describe` leaves a smaller type for the lines after it. `area` has no `default` and still type-checks, because the two
`case`s cover every variant of the union. `badArea` shows the other side: `r` exists only on the circle.

#### Core 3 — `unknown`, `any` and `never` (`scratch/core/core3-unknown-any-never.ts`)

```ts
// Core 3: unknown, any and never.

const a: any = JSON.parse('{"n":1}');
const u: unknown = JSON.parse('{"n":1}');

function takesNumber(x: number) {
  return x;
}

a.nope.deeper();              // compiles: any switches checking off (it fails when it RUNS)
const fromAny: number = a;    // compiles
takesNumber(a);               // compiles

u.n;                          // error: you cannot use an unknown value until you narrow it
const fromUnknown: number = u;   // error: unknown is not a number
takesNumber(u);                 // error: same reason

if (typeof u === 'object' && u !== null && 'n' in u) {
  u.n;                        // fine: narrowed to an object that has n
}

// never: nothing is left. A switch that handles every case ends with a `never` check.
type Status = 'todo' | 'in_progress' | 'done';
function label(s: Status): string {
  switch (s) {
    case 'todo': return 'To do';
    case 'in_progress': return 'In progress';
    case 'done': return 'Done';
    default: {
      const unreachable: never = s;
      return unreachable;
    }
  }
}

export { fromAny, fromUnknown, label };
```

What you should see for this file:

```text
scratch/core/core3-unknown-any-never.ts(14,1): error TS18046: 'u' is of type 'unknown'.
scratch/core/core3-unknown-any-never.ts(15,7): error TS2322: Type 'unknown' is not assignable to type 'number'.
scratch/core/core3-unknown-any-never.ts(16,13): error TS2345: Argument of type 'unknown' is not assignable to parameter of type 'number'.
```

Three lines using `a` (the `any`) produced **no error**. Now run the file:

```bash
npx tsx scratch/core/core3-unknown-any-never.ts
```

It stops with `TypeError: Cannot read properties of undefined (reading 'deeper')`. The compiler said nothing, and the program
crashed. The Concepts essay "`unknown`, `any` and `never`" explains why that is the whole difference.

Now the `never` part. In the file, change the union to `type Status = 'todo' | 'in_progress' | 'done' | 'blocked';` and run
`npx tsc -p scratch/core` again. There is one new error, on the `const unreachable: never = s;` line:

```text
scratch/core/core3-unknown-any-never.ts(30,13): error TS2322: Type '"blocked"' is not assignable to type 'never'.
```

The `switch` has no `case 'blocked'`, so a `'blocked'` value could reach the `default` branch, where `s` is supposed to be `never`. Add the
member back out (or add the `case`) and the error goes away.

#### Core 4 — `type` against `interface` (`scratch/core/core4-type-vs-interface.ts`)

```ts
// Core 4: type against interface.

// Object shapes: either works, and extends / & give the same result here.
interface Named { name: string }
interface Person extends Named { age: number }
type Named2 = { name: string };
type Person2 = Named2 & { age: number };
const p1: Person = { name: 'ada', age: 36 };
const p2: Person2 = { name: 'ada', age: 36 };

// Only a type alias can name a union, a tuple or a mapped type.
type Id = string | number;
type Pair = [string, number];
type Flags = { [K in 'a' | 'b']: boolean };

// Only an interface merges with another declaration of the same name.
interface Shape { name: string }
interface Shape { sides: number }
const square: Shape = { name: 'square' };         // error: after merging, sides is required
type Pt = { x: number };                       // error (reported on both declarations)
type Pt = { y: number };                          // error: a type cannot be declared twice

// Conflicting properties: extends complains at the declaration, & goes quiet and gives never.
interface A { v: string }
interface B extends A { v: number }               // error at the declaration
type C = A & { v: number };
const c: C = { v: 1 };                            // error appears later, at the use

export { p1, p2, square, c };
export type { Id, Pair, Flags };
```

What you should see for this file:

```text
scratch/core/core4-type-vs-interface.ts(19,7): error TS2741: Property 'sides' is missing in type '{ name: string; }' but required in type 'Shape'.
scratch/core/core4-type-vs-interface.ts(20,6): error TS2300: Duplicate identifier 'Pt'.
scratch/core/core4-type-vs-interface.ts(21,6): error TS2300: Duplicate identifier 'Pt'.
scratch/core/core4-type-vs-interface.ts(25,11): error TS2430: Interface 'B' incorrectly extends interface 'A'.
  Types of property 'v' are incompatible.
    Type 'number' is not assignable to type 'string'.
scratch/core/core4-type-vs-interface.ts(27,16): error TS2322: Type 'number' is not assignable to type 'never'.
```

The two ways of describing a `Person` both compiled. The errors are the cases where they differ.

#### Core 5 — Utility types (`scratch/core/core5-utility-types.ts`)

```ts
// Core 5: utility types. Equal and Expect make the compiler check a claim about a type:
// if every claim is true, tsc prints nothing. Change one claim to see an error.
type Equal<A, B> = (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;
type Expect<T extends true> = T;

type Item = { id: string; title: string; note?: string; readonly createdAt: Date };

type _partial = Expect<Equal<Partial<Item>, { id?: string; title?: string; note?: string; readonly createdAt?: Date }>>;
type _required = Expect<Equal<Required<Item>, { id: string; title: string; note: string; readonly createdAt: Date }>>;
type _pick = Expect<Equal<Pick<Item, 'id' | 'title'>, { id: string; title: string }>>;
type _omit = Expect<Equal<Omit<Item, 'createdAt' | 'note'>, { id: string; title: string }>>;
type _record = Expect<Equal<Record<'a' | 'b', number>, { a: number; b: number }>>;
type _readonly = Expect<Equal<Readonly<{ x: number }>, { readonly x: number }>>;

async function load() {
  return { n: 1 };
}
type _returnType = Expect<Equal<ReturnType<typeof load>, Promise<{ n: number }>>>;
type _awaited = Expect<Equal<Awaited<ReturnType<typeof load>>, { n: number }>>;
type _awaitedNested = Expect<Equal<Awaited<Promise<Promise<string>>>, string>>;

// A claim that is false, on purpose:
type _wrong = Expect<Equal<Pick<Item, 'id'>, { id: string; title: string }>>;   // error: the claim is false

export type {};
```

What you should see for this file (one error, from the claim marked false on purpose):

```text
scratch/core/core5-utility-types.ts(23,22): error TS2344: Type 'false' does not satisfy the constraint 'true'.
```

`Equal` and `Expect` make the compiler check a claim about a type: when a claim is true, nothing is printed. You will use the
same trick in Stage 7. Change one of the true claims and watch it fail. `Awaited<Promise<Promise<string>>>` is `string`: it
unwraps promises all the way down, which is why `Awaited<ReturnType<typeof f>>` is the type you get from `await f()`.

#### Core 6 — `satisfies` and `import type` (`scratch/core/core6-lib.ts`, `scratch/core/core6-satisfies-import-type.ts`)

**`scratch/core/core6-lib.ts`**

```ts
export type Point = { x: number };
export const origin: Point = { x: 0 };
```

**`scratch/core/core6-satisfies-import-type.ts`**

```ts
// Core 6: satisfies, and import type.
import { Point, origin } from './core6-lib.ts';   // error: Point is a type, so it needs `type`

// An annotation widens the value to the annotated type. satisfies only checks it.
const annotated: Record<'a' | 'b', string | number> = { a: 'x', b: 2 };
const checked = { a: 'x', b: 2 } satisfies Record<'a' | 'b', string | number>;
annotated.a.toUpperCase();      // error: a is string | number here
checked.a.toUpperCase();        // fine: a is still a string
const typo = { a: 'x', c: 2 } satisfies Record<'a' | 'b', string | number>;   // error: c is not allowed, b is missing

const p: Point = origin;
export { annotated, checked, typo, p };
```

What you should see for these files:

```text
scratch/core/core6-satisfies-import-type.ts(2,10): error TS1484: 'Point' is a type and must be imported using a type-only import when 'verbatimModuleSyntax' is enabled.
scratch/core/core6-satisfies-import-type.ts(7,13): error TS2339: Property 'toUpperCase' does not exist on type 'string | number'.
  Property 'toUpperCase' does not exist on type 'number'.
scratch/core/core6-satisfies-import-type.ts(9,24): error TS2353: Object literal may only specify known properties, and 'c' does not exist in type 'Record<"a" | "b", string | number>'.
```

Fix the first error by changing the import to `import { type Point, origin } from './core6-lib.ts';`. Before you fix it, run
`node scratch/core/core6-satisfies-import-type.ts` and see that Node fails too, which Stage 8 explains.

#### Say it in your own words

Close the files and write three or four sentences each, in your own words, for these two questions. You will need them for
the definition of done. Use a short example of your own.

1. What is the difference between `unknown` and `any`? What can you do with each, and what happens when you are wrong?
2. What is the difference between `type` and `interface`? When would you pick one over the other?

Then read the first two essays in section 4 and compare.

**Done when:**

- [ ] `npx tsc -p scratch/core` printed errors only on lines commented `// error` or on the `never` lines that ask for a type (30 in the
      reference build), and you can say what each one means. In `core6` the `import` line is the one you fix, after which two remain.
- [ ] `npx tsx scratch/core/core3-unknown-any-never.ts` fails with the `TypeError` shown above, with no compiler error behind it.
- [ ] You can explain, in your own words and with an example of your own, `unknown` against `any`, and `type` against
      `interface`.
- [ ] You can say what `as const`, a generic constraint, a discriminated union and `satisfies` each do, and what each of the
      eight utility types you ran (`Partial`, `Required`, `Pick`, `Omit`, `Record`, `Readonly`, `ReturnType`, `Awaited`) does.

### Stage 4 — Port the toolkit to TypeScript (`async-toolkit.ts`)

**Goal:** the M02 toolkit in TypeScript, with generics, behaving exactly as before, and no `any`.

**What it must do**

- Export `sleep`, `retry<T>`, `withTimeout<T>` and `mapLimit<T, R>`, and the classes `RetryError` and `TimeoutError`. The
  runtime behaviour is your M02 behaviour. Only the types are new.
- `retry<T>` accepts a function that returns `T` or `Promise<T>` and returns `Promise<T>`. `mapLimit<T, R>` accepts any
  `Iterable<T>` and returns `Promise<R[]>`.
- Keep the `RangeError` checks. `number` also allows `-1`, `NaN` and `1.5`, which the type system cannot rule out, so the types
  and the runtime checks do different jobs.
- No `any`, and no casts (`as`) to silence the compiler.

**Your turn.** Copy your M02 toolkit into `src/toolkit/async-toolkit.ts`, or start from this scaffold. It already type-checks, so
you can fill it in one function at a time and run `npx tsc --noEmit` after each:

```ts
// Module 4 output: the M02 async toolkit ported to strict TypeScript (named exports).
// Nothing to run here. Test it with async-toolkit.check.ts.

export function sleep(ms: number): Promise<void> {
  // TODO: your M02 body, unchanged: reject a bad ms with a RangeError, then resolve after ms
  throw new Error('TODO');
}

export class RetryError extends Error {
  // TODO: declare `attempts` as a readonly field (not a constructor parameter property)

  constructor(attempts: number, options?: ErrorOptions) {
    // TODO: your M02 constructor: call super with the message, set name, set attempts
    super('TODO', options);
  }
}

export type RetryOptions = {
  retries?: number;
  delayMs?: number;
};

export async function retry<T>(
  fn: (attempt: number) => T | Promise<T>,
  { retries = 3, delayMs = 100 }: RetryOptions = {},
): Promise<T> {
  // TODO: your M02 body. What type should the variable holding the last error have?
  throw new Error('TODO');
}

export class TimeoutError extends Error {
  // TODO: declare `ms` as a readonly field

  constructor(ms: number, options?: ErrorOptions) {
    // TODO: your M02 constructor
    super('TODO', options);
  }
}

export function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  // TODO: your M02 body
  throw new Error('TODO');
}

export async function mapLimit<T, R>(
  items: Iterable<T>,
  limit: number,
  fn: (item: T, index: number) => R | Promise<R>,
): Promise<R[]> {
  // TODO: your M02 body. Port it as it is first and read the one error the compiler gives you.
  throw new Error('TODO');
}
```

<details>
<summary>Hint 1: how to work</summary>

Change only the types, not the bodies. Port one function, run `npx tsc --noEmit`, then the next. A port that compiles after
every step is easier to fix than one that fails in 11 places at the end.

</details>

<details>
<summary>Hint 2: the error classes</summary>

Your M02 class probably said `this.attempts = attempts` with nothing declared. TypeScript needs the field declared. The
shortcut `constructor(public attempts: number)` is a *parameter property*, and `erasableSyntaxOnly` rejects it. Declare
`readonly attempts: number;` above the constructor and assign it inside.

</details>

<details>
<summary>Hint 3: <code>retry</code> and <code>withTimeout</code></summary>

What is the type of the last error? Anything can be thrown in JavaScript, so a caught value is `unknown`, and a variable that
holds it should say so. In `withTimeout`, `Promise.withResolvers<never>()`: the timeout promise never resolves with a value,
it only rejects, and `never` says that.

</details>

<details>
<summary>Hint 4: <code>mapLimit</code> will give you one error</summary>

Port it exactly as it is first. With `noUncheckedIndexedAccess` on, `list[index]` is `T | undefined`, and the compiler stops
you at the call `fn(list[index], index)`. Do not add `as T`: it silences the error and checks nothing. Do not check for
`undefined`: `T` could itself be `undefined`. Think about a way to hand each worker a real `T` without indexing. A shared
iterator from `list.entries()` does it.

</details>

<details>
<summary>The error you should hit in <code>mapLimit</code></summary>

```text
src/toolkit/async-toolkit.ts(113,35): error TS2345: Argument of type 'T | undefined' is not assignable to parameter of type 'T'.
  'T' could be instantiated with an arbitrary type which could be unrelated to 'T | undefined'.
```

The second line is the clue: `T` is any type, including one that contains `undefined`.

</details>

<details>
<summary>Reference implementation, <code>async-toolkit.ts</code> (open it after you have tried)</summary>

```ts
// Module 4 output: the M02 async toolkit ported to strict TypeScript (named exports).
// Nothing to run here. Test it with async-toolkit.check.ts.

export function sleep(ms: number): Promise<void> {
  if (!Number.isFinite(ms) || ms < 0) {
    throw new RangeError(`sleep: ms must be a non-negative number, got ${ms}`);
  }
  return new Promise((resolve) => setTimeout(resolve, ms));
}

export class RetryError extends Error {
  // Declared as a field, not a constructor parameter property (`public attempts`): parameter
  // properties are not erasable syntax, so `erasableSyntaxOnly` rejects them.
  readonly attempts: number;

  constructor(attempts: number, options?: ErrorOptions) {
    super(`Gave up after ${attempts} attempt${attempts === 1 ? '' : 's'}`, options);
    this.name = 'RetryError';
    this.attempts = attempts;
  }
}

export type RetryOptions = {
  retries?: number;
  delayMs?: number;
};

/**
 * Call `fn` until it succeeds. `retries` is the number of extra tries after the first,
 * so `retries: 2` means at most 3 calls. Waits `delayMs` between tries.
 * Rejects with a RetryError whose `cause` is the last error from `fn`.
 * `fn` receives the 0-based attempt number.
 */
export async function retry<T>(
  fn: (attempt: number) => T | Promise<T>,
  { retries = 3, delayMs = 100 }: RetryOptions = {},
): Promise<T> {
  if (!Number.isInteger(retries) || retries < 0) {
    throw new RangeError(`retry: retries must be a non-negative integer, got ${retries}`);
  }
  if (!Number.isFinite(delayMs) || delayMs < 0) {
    throw new RangeError(`retry: delayMs must be a non-negative number, got ${delayMs}`);
  }
  let lastError: unknown;
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      // `return await`, not `return`: without the await a rejection skips this catch.
      return await fn(attempt);
    } catch (error) {
      lastError = error;
      // The await inside this loop is deliberate: each try must finish before the next starts.
      if (attempt < retries) await sleep(delayMs);
    }
  }
  throw new RetryError(retries + 1, { cause: lastError });
}

export class TimeoutError extends Error {
  readonly ms: number;

  constructor(ms: number, options?: ErrorOptions) {
    super(`Timed out after ${ms}ms`, options);
    this.name = 'TimeoutError';
    this.ms = ms;
  }
}

/**
 * Settle like `promise`, unless it takes longer than `ms`: then reject with a TimeoutError.
 * This does not cancel the underlying work. A promise cannot be cancelled from outside, so
 * pass an AbortSignal to whatever created it if the work itself must stop.
 */
export function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  if (!Number.isFinite(ms) || ms < 0) {
    throw new RangeError(`withTimeout: ms must be a non-negative number, got ${ms}`);
  }
  // AbortSignal.timeout's timer is unref'd, so it never keeps the process alive after the
  // promise has settled (a plain setTimeout would, until it fired).
  const signal = AbortSignal.timeout(ms);
  const { promise: timeout, reject } = Promise.withResolvers<never>();
  const onAbort = () => reject(new TimeoutError(ms, { cause: signal.reason }));
  signal.addEventListener('abort', onAbort, { once: true });
  return Promise.race([promise, timeout]).finally(() => {
    signal.removeEventListener('abort', onAbort);
  });
}

/**
 * Like `items.map(fn)` but runs at most `limit` calls of `fn` at the same time.
 * Results keep the order of `items`. If any call rejects, mapLimit rejects with that error
 * and starts no new calls (calls already running are left to finish on their own).
 */
export async function mapLimit<T, R>(
  items: Iterable<T>,
  limit: number,
  fn: (item: T, index: number) => R | Promise<R>,
): Promise<R[]> {
  if (!Number.isInteger(limit) || limit < 1) {
    throw new RangeError(`mapLimit: limit must be an integer >= 1, got ${limit}`);
  }
  const list = [...items];
  const results = new Array<R>(list.length);
  // One shared iterator of [index, item] pairs. Each worker takes the next pair when it
  // finishes its current one, so there are never more than `limit` calls in flight, and two
  // workers can never take the same pair. Iterating, rather than reading list[index], means
  // `item` is a T and not a `T | undefined` under noUncheckedIndexedAccess.
  const pairs = list.entries();
  let failed = false;

  async function worker(): Promise<void> {
    while (!failed) {
      const next = pairs.next();
      if (next.done) return;
      const [index, item] = next.value;
      try {
        results[index] = await fn(item, index);
      } catch (error) {
        failed = true;
        throw error;
      }
    }
  }

  const workers = Array.from({ length: Math.min(limit, list.length) }, worker);
  await Promise.all(workers);
  return results;
}
```

</details>

**Done when:**

- [ ] `npx tsc --noEmit` prints nothing and exits 0 (run `echo $?` straight after it to see `0`).
- [ ] The toolkit exports `retry<T>`, `mapLimit<T, R>`, `sleep` and `withTimeout`, and
      `grep -nE ": any\b|<any>|\bas (any|unknown|const|[A-Z])" src/toolkit/async-toolkit.ts` prints nothing (it exits 1, meaning no
      match): no `any` and no cast.
- [ ] You can explain why `mapLimit` produced an error and how your fix avoids the problem without a cast.

### Stage 5 — Port the toolkit's checks

**Goal:** the toolkit's checks running as TypeScript, with the compiler and the runtime both checking the bad-input cases.

1. Copy your M02 check file to `src/toolkit/async-toolkit.check.ts` and change **every** mention of the old file name to
   `./async-toolkit.ts`. Search for it first: `grep -n "async-toolkit.mjs" src/toolkit/async-toolkit.check.ts`. In the reference M02
   checks there are two mentions: the `import` on line 7, and a `new URL('./async-toolkit.mjs', import.meta.url)` on line 126, **inside a
   string** that the "does not keep the process alive" check passes to a child Node process. The compiler cannot see inside a string, so
   if you change only the import, `tsc` looks fine and then `tsx` fails with `ERR_MODULE_NOT_FOUND ... async-toolkit.mjs`. That child
   process is plain `node`, so this one check already imports your `.ts` file through native type stripping (Stage 8). If you do not
   have an M02 check file, use the finished one below.
2. Run `npx tsc --noEmit` **before** fixing anything, and read what it says.
3. Fix each error properly. If you ported the reference M02 checks you will see about 11 errors, in four kinds:

   | You will see | Why | Fix |
   | --- | --- | --- |
   | `TS7006` Parameter implicitly has an `any` type | The `check(name, fn)` helper has untyped parameters | Annotate it: `(name: string, fn: () => void \| Promise<void>)` |
   | `TS7034` / `TS7005` Variable implicitly has type `any[]` | `const attempts = []` has no element type | `const attempts: number[] = []` |
   | `TS18046` `'error.cause'` is of type `unknown` | `cause` is `unknown` by definition | `assert.ok(error.cause instanceof Error)` before `.message` |
   | `TS2345` / `TS2322` on `sleep('soon')`, `{ delayMs: 'soon' }` | The checks pass bad input **on purpose** | `// @ts-expect-error` above the line, not a changed test |

   `@ts-expect-error` is itself checked: if the line below it stops being an error, `tsc` reports the unused directive. The
   runtime `assert.throws` stays, because a plain JavaScript caller can still send bad input past the types.

   The M02 timing check for `retry` (two 40 ms waits, with an upper bound only 35 ms above the expected time) can fail on a busy
   machine even when the code is right. The finished file below uses 100 ms waits with bounds of 190 ms and 280 ms: still far enough
   apart to catch a wait after the last attempt (about 300 ms), but with room to spare.
4. Run it, through `tsx`:
   ```bash
   npx tsx src/toolkit/async-toolkit.check.ts
   ```

<details>
<summary>The finished check file, <code>async-toolkit.check.ts</code></summary>

```ts
// Module 4 output: self-checks for the TypeScript async toolkit (a port of the M02 checks).
// Run from this folder: npm run check   (or: npx tsx src/toolkit/async-toolkit.check.ts)
// Success: 23 "ok - ..." lines and no error.
import assert from 'node:assert/strict';
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
import { sleep, retry, RetryError, withTimeout, TimeoutError, mapLimit } from './async-toolkit.ts';

const run = promisify(execFile);

async function check(name: string, fn: () => void | Promise<void>): Promise<void> {
  await fn();
  console.log(`ok - ${name}`);
}

await check('sleep waits at least the requested time', async () => {
  const start = performance.now();
  await sleep(50);
  const elapsed = performance.now() - start;
  // setTimeout can fire a hair early against performance.now(), so allow 5ms of slack.
  assert.ok(elapsed >= 45, `expected >= 45ms, got ${elapsed.toFixed(1)}ms`);
});

await check('sleep resolves with undefined', async () => {
  assert.equal(await sleep(1), undefined);
});

await check('sleep rejects bad input synchronously', () => {
  assert.throws(() => sleep(-1), RangeError);
  // @ts-expect-error a string is not a number: the type error is the point, the RangeError is the runtime backstop
  assert.throws(() => sleep('soon'), RangeError);
});

await check('retry returns the first success without retrying', async () => {
  let calls = 0;
  const result = await retry(async () => {
    calls++;
    return 'done';
  }, { retries: 3, delayMs: 1 });
  assert.equal(result, 'done');
  assert.equal(calls, 1);
});

await check('retry retries an async rejection until it succeeds', async () => {
  const attempts: number[] = [];
  const result = await retry(async (attempt) => {
    attempts.push(attempt);
    if (attempt < 2) throw new Error(`fail ${attempt}`);
    return 'third time lucky';
  }, { retries: 3, delayMs: 1 });
  assert.equal(result, 'third time lucky');
  assert.deepEqual(attempts, [0, 1, 2]);
});

await check('retry gives up with a RetryError that keeps the last error as cause', async () => {
  let calls = 0;
  await assert.rejects(
    retry(async () => {
      calls++;
      throw new Error(`boom ${calls}`);
    }, { retries: 2, delayMs: 1 }),
    (error) => {
      assert.ok(error instanceof RetryError);
      assert.equal(error.attempts, 3);
      assert.ok(error.cause instanceof Error);
      assert.equal(error.cause.message, 'boom 3');
      return true;
    },
  );
  assert.equal(calls, 3);
});

await check('retry waits delayMs between tries but not after the last one', async () => {
  const start = performance.now();
  await assert.rejects(retry(async () => { throw new Error('x'); }, { retries: 2, delayMs: 100 }));
  const elapsed = performance.now() - start;
  // 3 attempts -> 2 waits of 100ms = 200ms. A wait after the last attempt would make it ~300ms.
  // The bounds sit 10ms below and 80ms above 200ms, so a busy machine does not fail the check,
  // while the extra 100ms wait it guards against still does.
  assert.ok(elapsed >= 190, `expected >= 190ms, got ${elapsed.toFixed(1)}ms`);
  assert.ok(elapsed < 280, `expected < 280ms, got ${elapsed.toFixed(1)}ms`);
});

await check('retry with retries: 0 calls fn exactly once', async () => {
  let calls = 0;
  await assert.rejects(retry(async () => { calls++; throw new Error('x'); }, { retries: 0 }), RetryError);
  assert.equal(calls, 1);
});

await check('retry rejects a bad retries option', async () => {
  await assert.rejects(retry(async () => {}, { retries: -1 }), RangeError);
});

await check('retry rejects a bad delayMs even when fn would succeed first time', async () => {
  let calls = 0;
  await assert.rejects(retry(async () => { calls++; }, { delayMs: -1 }), RangeError);
  // @ts-expect-error delayMs must be a number
  await assert.rejects(retry(async () => { calls++; }, { delayMs: 'soon' }), RangeError);
  assert.equal(calls, 0); // fn is never called with invalid options
});

await check('withTimeout passes through a value that arrives in time', async () => {
  const value = await withTimeout(sleep(10).then(() => 'fast'), 200);
  assert.equal(value, 'fast');
});

await check('withTimeout rejects with a TimeoutError when the promise is too slow', async () => {
  const start = performance.now();
  await assert.rejects(withTimeout(sleep(500), 40), (error) => {
    assert.ok(error instanceof TimeoutError);
    assert.equal(error.ms, 40);
    return true;
  });
  assert.ok(performance.now() - start < 300, 'should reject near 40ms, not wait for the 500ms promise');
});

await check('withTimeout passes through the original rejection unchanged', async () => {
  const original = new Error('original failure');
  await assert.rejects(withTimeout(Promise.reject(original), 200), (error) => error === original);
});

await check('withTimeout survives the slow promise rejecting after the timeout', async () => {
  // Promise.race has already attached handlers, so this late rejection must not become unhandled.
  const late = sleep(80).then(() => { throw new Error('too late'); });
  await assert.rejects(withTimeout(late, 20), TimeoutError);
  await sleep(120);
});

await check('withTimeout does not keep the process alive after the promise settles', async () => {
  const code = `
    import { withTimeout, sleep } from ${JSON.stringify(new URL('./async-toolkit.ts', import.meta.url).href)};
    await withTimeout(sleep(5), 10_000);
  `;
  const start = performance.now();
  await run(process.execPath, ['--input-type=module', '--eval', code]);
  const elapsed = performance.now() - start;
  // A lingering 10s timer would make the child take ~10000ms, so 8000ms leaves room for a slow start.
  assert.ok(elapsed < 8000, `child took ${elapsed.toFixed(0)}ms, a lingering 10s timer would be ~10000ms`);
});

await check('withTimeout rejects a bad ms synchronously', () => {
  assert.throws(() => withTimeout(sleep(1), -5), RangeError);
});

await check('mapLimit keeps results in input order even when calls finish out of order', async () => {
  // Earlier items sleep longer, so they finish last.
  const results = await mapLimit([30, 20, 10, 0], 4, async (ms, index) => {
    await sleep(ms);
    return `${index}:${ms}`;
  });
  assert.deepEqual(results, ['0:30', '1:20', '2:10', '3:0']);
});

await check('mapLimit never runs more than `limit` calls at once, and does reach the limit', async () => {
  let running = 0;
  let peak = 0;
  const results = await mapLimit([...Array(10).keys()], 3, async (n) => {
    running++;
    peak = Math.max(peak, running);
    await sleep(15);
    running--;
    return n * 2;
  });
  assert.equal(peak, 3);
  assert.deepEqual(results, [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]);
});

await check('mapLimit with limit 1 is sequential', async () => {
  const order: string[] = [];
  await mapLimit([1, 2, 3], 1, async (n) => {
    order.push(`start ${n}`);
    await sleep(5);
    order.push(`end ${n}`);
  });
  assert.deepEqual(order, ['start 1', 'end 1', 'start 2', 'end 2', 'start 3', 'end 3']);
});

await check('mapLimit passes the index and accepts a sync fn and any iterable', async () => {
  const results = await mapLimit(new Set(['a', 'b']), 2, (item, index) => `${item}${index}`);
  assert.deepEqual(results, ['a0', 'b1']);
});

await check('mapLimit handles an empty input and a limit larger than the input', async () => {
  assert.deepEqual(await mapLimit([], 5, async (x) => x), []);
  assert.deepEqual(await mapLimit([1, 2], 50, async (x) => x + 1), [2, 3]);
});

await check('mapLimit rejects with the first error and starts no new work after it', async () => {
  const started: number[] = [];
  await assert.rejects(
    mapLimit([0, 1, 2, 3, 4, 5], 2, async (n) => {
      started.push(n);
      await sleep(10);
      if (n === 1) throw new Error('item 1 failed');
      return n;
    }),
    /item 1 failed/,
  );
  await sleep(60); // give a runaway worker time to start more items, if it were going to
  assert.deepEqual(started, [0, 1, 2]);
});

await check('mapLimit rejects a bad limit', async () => {
  await assert.rejects(mapLimit([1], 0, async (x) => x), RangeError);
  await assert.rejects(mapLimit([1], 1.5, async (x) => x), RangeError);
});
```

</details>

**Done when:**

- [ ] `npx tsc --noEmit` prints nothing and exits 0.
- [ ] `npx tsx src/toolkit/async-toolkit.check.ts` exits 0 and prints 23 `ok - ...` lines (if you ported the reference checks).
- [ ] You can say why the bad-input lines carry `// @ts-expect-error` **and** still have a runtime `assert.throws`.

### Stage 6 — Model the Task domain (`task.ts`)

**Goal:** a `Task` type, with the create and update input types **derived** from it, not written by hand.

Start from the domain type and ask, for each field, *who owns it?*

| Field | Who sets it | On create | On update |
| --- | --- | --- | --- |
| `id` | the server | not allowed | not allowed |
| `title` | the client | required | optional |
| `description` | the client | optional | optional |
| `status` | the client | optional (a new task is `todo`) | optional |
| `createdAt`, `updatedAt` | the server | not allowed | not allowed |

**What it must do**

- `TASK_STATUSES` is the list of the three statuses, frozen with `as const`, and `TaskStatus` is derived from it. The type is a
  **union, not an `enum`**.
- `Task` has `id` and `createdAt` as `readonly`, `description` optional, and the fields in the table.
- `CreateTaskInput` is *`Task` minus what the server owns, with `status` optional*. `UpdateTaskInput` is *any subset of what a
  client may set*. Both use utility types.
- `isTaskStatus(value: unknown): value is TaskStatus` is a user-defined type guard. `STATUS_LABELS` has a label per status, checked
  with `satisfies`.
- `createTask(id, input, now?)` builds a `Task` (status defaults to `todo`, `description` is left out when absent).
  `applyTaskUpdate(task, update, now?)` returns a **new** task with only the given fields changed and `updatedAt` bumped. A field that is
  missing or `undefined` is left alone, and a key that is not part of `UpdateTaskInput` (such as `id`) is ignored even if it sneaks in.

**Your turn.** Create `src/domain/task.ts` from this scaffold. It type-checks as it stands. Replace each `TODO`:

```ts
// Module 4 output: the Task domain types.
// Nothing to run here. Type-check with `npm run typecheck`, exercise with domain.check.ts.

// TODO: TASK_STATUSES, a list of the three statuses, frozen with `as const`
export const TASK_STATUSES = ['todo'] as const;

// TODO: TaskStatus, a union derived from TASK_STATUSES (not typed out a second time)
export type TaskStatus = string;

// TODO: Task. id and createdAt must be readonly, description optional, status a TaskStatus.
export type Task = {
  title: string;
};

// TODO: CreateTaskInput, derived from Task with utility types. Not written by hand.
export type CreateTaskInput = Task;

// TODO: UpdateTaskInput, derived from CreateTaskInput
export type UpdateTaskInput = Task;

// TODO: a type guard, so an unknown value can be narrowed to a TaskStatus
export function isTaskStatus(value: unknown): value is TaskStatus {
  throw new Error('TODO');
}

// TODO: a label for every status, checked with `satisfies Record<TaskStatus, string>`
export const STATUS_LABELS = {} satisfies Record<string, string>;

// TODO: build a Task from an id and a CreateTaskInput (the server owns id and the timestamps)
export function createTask(id: string, input: CreateTaskInput, now: Date = new Date()): Task {
  throw new Error('TODO');
}

// TODO: return a new Task with the update applied and updatedAt bumped. Do not mutate the input.
// A missing or undefined field is left alone, and keys outside UpdateTaskInput are ignored.
export function applyTaskUpdate(task: Task, update: UpdateTaskInput, now: Date = new Date()): Task {
  throw new Error('TODO');
}
```

<details>
<summary>Hint 1: the status list and the union</summary>

`['todo', 'in_progress', 'done'] as const` is a `readonly` tuple of three literals. `(typeof TASK_STATUSES)[number]` is the
union of what is in it. Write the list once and derive the type, so they cannot disagree.

</details>

<details>
<summary>Hint 2: <code>CreateTaskInput</code></summary>

Read the table as a sentence: *everything except `id`, the timestamps and `status`*, then *status, but optional*. Which utility
type removes properties? Which one keeps only some? Which one makes properties optional? `Omit` keeps each remaining property
exactly as it was, so `title` stays required and `description` stays optional. Join the two parts with `&`.

</details>

<details>
<summary>Hint 3: <code>UpdateTaskInput</code></summary>

It is the create input with every property optional. Define it from `CreateTaskInput`, **not** from `Task`. That is what keeps
`id` out of it.

</details>

<details>
<summary>Hint 4: the type guard and <code>satisfies</code></summary>

`TASK_STATUSES` is `readonly ['todo', ...]`, and `includes` on it only accepts a `TaskStatus`. Widen it for the call:
`(TASK_STATUSES as readonly string[]).includes(value)`, after checking `typeof value === 'string'`. For the labels, put
`satisfies Record<TaskStatus, string>` after the object literal, so a missing or misspelled status is an error but the object
keeps its precise type.

</details>

<details>
<summary>Hint 5: <code>applyTaskUpdate</code> is easy to get subtly wrong</summary>

`{ ...task, ...update, updatedAt: now }` looks right and compiles. Try two calls against it: `applyTaskUpdate(t, { title: undefined })`,
which blanks the title (every field is optional, so `undefined` is allowed), and an object held in a variable,
`const loose = { title: 'x', id: 'hijack' }; applyTaskUpdate(t, loose)`, which overwrites `id`, because only a *fresh object literal* is
checked for extra properties. Name the fields instead: `title: update.title ?? task.title`, and so on, and add `description` only when it
is not `undefined`.

</details>

<details>
<summary>Reference implementation, <code>task.ts</code> (open it after you have tried)</summary>

```ts
// Module 4 output: the Task domain types.
// Nothing to run here. Type-check with `npm run typecheck`, exercise with domain.check.ts.

// One list of allowed values is the source of truth: the type is derived from it, so the
// runtime array and the compile-time union cannot drift apart. `as const` keeps each entry a
// literal ('todo') instead of widening to string. A union, not an enum: enums are not
// erasable syntax, so `erasableSyntaxOnly` rejects them.
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

// What a client sends to create a task. The server owns id and the timestamps, so they are
// omitted. status is optional on create (a new task starts as 'todo'), so it is taken out of
// the required part and put back as optional.
export type CreateTaskInput = Omit<Task, 'id' | 'createdAt' | 'updatedAt' | 'status'> &
  Partial<Pick<Task, 'status'>>;

// What a client sends to change a task: any subset of the creatable fields.
export type UpdateTaskInput = Partial<CreateTaskInput>;

// A user-defined type guard: `value is TaskStatus` tells the compiler that when this returns
// true, `value` can be treated as a TaskStatus.
export function isTaskStatus(value: unknown): value is TaskStatus {
  return typeof value === 'string' && (TASK_STATUSES as readonly string[]).includes(value);
}

// `satisfies` checks the object against Record<TaskStatus, string> (a missing or misspelled
// key is an error) without widening its type to that Record, so STATUS_LABELS.todo is
// accepted and STATUS_LABELS.nope is an error.
export const STATUS_LABELS = {
  todo: 'To do',
  in_progress: 'In progress',
  done: 'Done',
} satisfies Record<TaskStatus, string>;

// Build a Task from a create input: the server-owned fields are filled in here.
export function createTask(id: string, input: CreateTaskInput, now: Date = new Date()): Task {
  return {
    id,
    title: input.title,
    ...(input.description !== undefined && { description: input.description }),
    status: input.status ?? 'todo',
    createdAt: now,
    updatedAt: now,
  };
}

// Apply an update input to a task, returning a new task. A field that is missing or undefined is
// left alone. Each field is named here on purpose, instead of `{ ...task, ...update }`: a spread
// would copy whatever the object holds, including `undefined` values and keys that are not part
// of UpdateTaskInput (an object held in a variable is not checked for extra properties), so a
// client could blank a required field or overwrite the server-owned id and createdAt.
export function applyTaskUpdate(task: Task, update: UpdateTaskInput, now: Date = new Date()): Task {
  return {
    ...task,
    title: update.title ?? task.title,
    ...(update.description !== undefined && { description: update.description }),
    status: update.status ?? task.status,
    updatedAt: now,
  };
}
```

</details>

Check your work with the compiler: `npx tsc --noEmit` prints nothing. The runtime and type-level checks for these types are in
`domain.check.ts`, which you create in Stage 7 once `Result` exists.

**Done when:**

- [ ] `npx tsc --noEmit` prints nothing and exits 0 with your `task.ts` in place.
- [ ] `CreateTaskInput` and `UpdateTaskInput` are defined with utility types from `Task`, and `grep -n "title: string" src/domain/task.ts`
      finds `title` declared only once, in `Task`.
- [ ] `TaskStatus` is a union derived from `TASK_STATUSES`, and no `enum` appears in the file.
- [ ] You can explain why `UpdateTaskInput` is defined from `CreateTaskInput` and not from `Task`.

### Stage 7 — Model `Result<T, E>`, then check the domain (`result.ts`, `domain.check.ts`)

**Goal:** a discriminated union that makes a success and a failure two different shapes, so the compiler makes you handle both.

**What it must do**

- `Result<T, E = Error>` is a union of `{ ok: true; value: T }` and `{ ok: false; error: E }`. `ok` is the **discriminant**: checking
  it narrows the whole object, so `value` is reachable only when `ok` is `true` and `error` only when it is `false`.
- `ok(value)` and `err(error)` build the two variants. `ok` returns `Result<T, never>` and `err` returns `Result<never, E>`, so
  each fits wherever a `Result<T, E>` is expected.
- `toError(thrown: unknown): Error`, `attempt(fn)`, `mapResult` and `unwrapOr` are small helpers. `attempt` runs an async
  function and returns a failure `Result` instead of throwing. `toError` turns anything thrown into an `Error`, keeping the original
  as `cause` when it was not an `Error`, and it must not throw itself, even for `Object.create(null)`, which `String()` cannot convert.

**Your turn.** Create `src/domain/result.ts` from this scaffold. It type-checks, but only because it is deliberately wrong: the
`Result` type is the *loose* shape, where `value` and `error` are both optional. Replace it:

```ts
// Module 4 output: a discriminated-union Result<T, E>.
// Nothing to run here. Type-check with `npm run typecheck`, exercise with domain.check.ts.

// TODO: Result<T, E = Error>: a union of a success variant and a failure variant,
// told apart by a property that is a literal type in each.
export type Result<T, E = Error> = { ok: boolean; value?: T; error?: E };

// TODO: ok(value) and err(error). Their return types should let either one be used
// wherever a Result<T, E> is expected.
export function ok<T>(value: T): Result<T, never> {
  throw new Error('TODO');
}
export function err<E>(error: E): Result<never, E> {
  throw new Error('TODO');
}

// TODO: turn anything that was thrown into an Error, keeping the original as `cause`.
// This function must never throw itself, whatever it is given.
export function toError(thrown: unknown): Error {
  throw new Error('TODO');
}

// TODO: run an async function and return a failure Result instead of throwing
export async function attempt<T>(fn: () => Promise<T>): Promise<Result<T, Error>> {
  throw new Error('TODO');
}

export function mapResult<T, U, E>(result: Result<T, E>, fn: (value: T) => U): Result<U, E> {
  throw new Error('TODO');
}

export function unwrapOr<T, E>(result: Result<T, E>, fallback: T): T {
  throw new Error('TODO');
}
```

<details>
<summary>Hint 1: the discriminant</summary>

Each variant needs a property whose type is a **literal** (`true` in one, `false` in the other). A plain `boolean` would not tell
the variants apart. Mark the properties `readonly` if you want results to be immutable.

</details>

<details>
<summary>Hint 2: what the loose shape costs you (try it)</summary>

With `{ ok: boolean; value?: T; error?: E }`, `if (r.ok) return r.value * 2;` fails with `'r.value' is possibly 'undefined'`, and
`{ ok: true, error: 'but ok is true' }` is accepted. With the union, the first compiles and the second is `TS2353`. That is the
reason for the discriminant.

</details>

<details>
<summary>Hint 3: <code>toError</code> and <code>attempt</code></summary>

Return `thrown` itself if it is an `Error`. Otherwise build `new Error(String(thrown), { cause: thrown })`, but note that `String()` can
throw (try `String(Object.create(null))`), so put it in a `try`/`catch` with a fixed fallback message. In `attempt`, a `try` around
`await fn()`, `return ok(...)`, and in `catch (thrown)` return `err(toError(thrown))`. Inside `catch`, `thrown` is `unknown`, which is why
`toError` takes `unknown`.

</details>

<details>
<summary>Reference implementation, <code>result.ts</code> (open it after you have tried)</summary>

```ts
// Module 4 output: a discriminated-union Result<T, E>.
// Nothing to run here. Type-check with `npm run typecheck`, exercise with domain.check.ts.

// `ok` is the discriminant: checking it narrows the whole object, so `value` is only
// reachable when ok is true and `error` only when it is false.
export type Result<T, E = Error> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

// Result<T, never> and Result<never, E> are assignable to any Result<T, E>, so ok(1) fits a
// function declared to return Result<number, string>.
export function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

export function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// Anything can be thrown in JavaScript, so a caught value is `unknown`. Turn it into an Error
// once, here, and keep the original as `cause` when it was not one already. String() can itself
// throw (for Object.create(null), or an object whose toString throws), so it is guarded: this
// function is what lets `attempt` promise that it never throws.
export function toError(thrown: unknown): Error {
  if (thrown instanceof Error) return thrown;
  let message: string;
  try {
    message = String(thrown);
  } catch {
    message = 'A value that cannot be converted to a string was thrown';
  }
  return new Error(message, { cause: thrown });
}

// Run an async function and capture a rejection as a Result instead of throwing.
export async function attempt<T>(fn: () => Promise<T>): Promise<Result<T, Error>> {
  try {
    return ok(await fn());
  } catch (thrown) {
    return err(toError(thrown));
  }
}

export function mapResult<T, U, E>(result: Result<T, E>, fn: (value: T) => U): Result<U, E> {
  return result.ok ? ok(fn(result.value)) : result;
}

export function unwrapOr<T, E>(result: Result<T, E>, fallback: T): T {
  return result.ok ? result.value : fallback;
}
```

</details>

**Self-checks.** Create `src/domain/domain.check.ts`. It has two kinds of check. The `Equal` / `Expect` lines and the
`@ts-expect-error` lines are checked by `npx tsc --noEmit`. The `check(...)` calls are checked by running the file. It fails until
both of your files are right:

<details>
<summary>The check file, <code>domain.check.ts</code> (open it, copy it, then run it)</summary>

```ts
// Module 4 output: self-checks for the Task domain types and Result<T, E>.
// Run from this folder: npm run check   (or: npx tsx src/domain/domain.check.ts)
// Two kinds of check: runtime asserts (run the file) and type-level asserts (`npm run typecheck`).
// Success: "ok - ..." lines and no error from either command.
import assert from 'node:assert/strict';
import { sleep } from '../toolkit/async-toolkit.ts';
import {
  applyTaskUpdate,
  createTask,
  isTaskStatus,
  STATUS_LABELS,
  TASK_STATUSES,
  type CreateTaskInput,
  type Task,
  type TaskStatus,
  type UpdateTaskInput,
} from './task.ts';
import { attempt, err, mapResult, ok, toError, unwrapOr, type Result } from './result.ts';

// Type-level assertions: these cost nothing at runtime and fail `tsc --noEmit` if a type drifts.
type Equal<A, B> =
  (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;
type Expect<T extends true> = T;

type _StatusIsTheUnion = Expect<Equal<TaskStatus, 'todo' | 'in_progress' | 'done'>>;
type _CreateKeys = Expect<Equal<keyof CreateTaskInput, 'title' | 'description' | 'status'>>;
type _TitleRequiredOnCreate = Expect<Equal<CreateTaskInput['title'], string>>;
type _StatusOptionalOnCreate = Expect<Equal<{} extends Pick<CreateTaskInput, 'status'> ? true : false, true>>;
type _UpdateIsAllOptional = Expect<Equal<{} extends UpdateTaskInput ? true : false, true>>;
type _UpdateKeepsTypes = Expect<Equal<UpdateTaskInput['status'], TaskStatus | undefined>>;
// The update input must not offer the server-owned fields, and must keep Task's own field types.
type _UpdateKeys = Expect<Equal<keyof UpdateTaskInput, 'title' | 'description' | 'status'>>;
type _DescriptionType = Expect<Equal<Task['description'], string | undefined>>;

async function check(name: string, fn: () => void | Promise<void>): Promise<void> {
  await fn();
  console.log(`ok - ${name}`);
}

const now = new Date('2026-10-21T09:00:00.000Z');

await check('TASK_STATUSES lists the three statuses in order', () => {
  assert.deepEqual([...TASK_STATUSES], ['todo', 'in_progress', 'done']);
});

await check('isTaskStatus accepts the three statuses and rejects everything else', () => {
  for (const status of TASK_STATUSES) assert.equal(isTaskStatus(status), true);
  for (const bad of ['TODO', 'finished', '', 0, null, undefined, {}]) {
    assert.equal(isTaskStatus(bad), false);
  }
});

await check('isTaskStatus narrows an unknown value to TaskStatus', () => {
  const raw: unknown = 'done';
  if (isTaskStatus(raw)) {
    // Inside this branch `raw` is a TaskStatus, so it can index STATUS_LABELS.
    assert.equal(STATUS_LABELS[raw], 'Done');
  } else {
    assert.fail('expected done to be a TaskStatus');
  }
});

await check('STATUS_LABELS has a label for every status', () => {
  assert.deepEqual(Object.keys(STATUS_LABELS), [...TASK_STATUSES]);
});

await check('createTask fills server-owned fields and defaults status to todo', () => {
  const task = createTask('t1', { title: 'Write the M04 guide' }, now);
  assert.deepEqual(task, {
    id: 't1',
    title: 'Write the M04 guide',
    status: 'todo',
    createdAt: now,
    updatedAt: now,
  });
  assert.equal('description' in task, false);
});

await check('createTask keeps an explicit status and description', () => {
  const task = createTask('t2', { title: 'Ship', description: 'to main', status: 'in_progress' }, now);
  assert.equal(task.status, 'in_progress');
  assert.equal(task.description, 'to main');
});

await check('applyTaskUpdate changes only the given fields and bumps updatedAt', () => {
  const before: Task = createTask('t3', { title: 'Old', description: 'keep me' }, now);
  const later = new Date('2026-10-22T09:00:00.000Z');
  const after = applyTaskUpdate(before, { title: 'New', status: 'done' }, later);
  assert.equal(after.title, 'New');
  assert.equal(after.status, 'done');
  assert.equal(after.description, 'keep me');
  assert.equal(after.id, 't3');
  assert.equal(after.createdAt, now);
  assert.equal(after.updatedAt, later);
  assert.equal(before.title, 'Old'); // the input task is not mutated

  // An undefined field means "leave it alone", it must not blank the task. This compiles
  // because every field of UpdateTaskInput is optional.
  const untouched = applyTaskUpdate(before, { title: undefined, status: undefined }, later);
  assert.equal(untouched.title, 'Old');
  assert.equal(untouched.status, 'todo');

  // An object held in a variable is not checked for extra properties, so this compiles. The
  // server-owned fields must still survive it.
  const loose = { title: 'Newer', id: 'hijack', createdAt: new Date(0) };
  const guarded = applyTaskUpdate(before, loose, later);
  assert.equal(guarded.title, 'Newer');
  assert.equal(guarded.id, 't3');
  assert.equal(guarded.createdAt, now);
});

await check('the compiler rejects wrong task and input shapes', () => {
  // @ts-expect-error id is server-owned: it is not part of CreateTaskInput
  createTask('t4', { title: 'x', id: 'client-chose-this' }, now);
  // @ts-expect-error 'finished' is not one of the three statuses
  createTask('t5', { title: 'x', status: 'finished' }, now);
  // @ts-expect-error title is required on create
  createTask('t6', {}, now);
  const task = createTask('t7', { title: 'x' }, now);
  // @ts-expect-error id is readonly
  task.id = 'other';
  // @ts-expect-error createdAt is readonly
  task.createdAt = now;
  // These compile: title, description, status and updatedAt are writable.
  task.title = 'y';
  task.status = 'done';
});

await check('Result: ok and err build the two variants', () => {
  assert.deepEqual(ok(1), { ok: true, value: 1 });
  assert.deepEqual(err('nope'), { ok: false, error: 'nope' });
});

await check('Result: checking `ok` narrows to value or error', () => {
  function describe(result: Result<number, string>): string {
    if (result.ok) {
      // @ts-expect-error `error` does not exist on the ok variant
      result.error;
      return `got ${result.value}`;
    }
    // @ts-expect-error `value` does not exist on the failed variant
    result.value;
    return `failed: ${result.error}`;
  }
  assert.equal(describe(ok(42)), 'got 42');
  assert.equal(describe(err('boom')), 'failed: boom');
});

await check('Result: mapResult maps ok values and passes failures through untouched', () => {
  assert.deepEqual(mapResult(ok(2), (n) => n * 10), ok(20));
  const failure: Result<number, string> = err('bad');
  assert.equal(mapResult(failure, (n) => n * 10), failure);
});

await check('Result: unwrapOr returns the value or the fallback', () => {
  assert.equal(unwrapOr(ok('v'), 'fallback'), 'v');
  assert.equal(unwrapOr(err('x'), 'fallback'), 'fallback');
});

await check('toError keeps an Error and wraps anything else with the original as cause', () => {
  const original = new RangeError('already an error');
  assert.equal(toError(original), original);
  const wrapped = toError('just a string');
  assert.ok(wrapped instanceof Error);
  assert.equal(wrapped.message, 'just a string');
  assert.equal(wrapped.cause, 'just a string');

  // String() throws for an object with no prototype, so toError has to guard it.
  const bare = Object.create(null);
  const fromBare = toError(bare);
  assert.ok(fromBare instanceof Error);
  assert.equal(fromBare.cause, bare);
});

await check('attempt turns a resolved promise into ok and a rejection into err', async () => {
  const good = await attempt(async () => {
    await sleep(1);
    return 'fine';
  });
  assert.deepEqual(good, { ok: true, value: 'fine' });

  const bad = await attempt(async () => {
    throw 'a thrown string, not an Error';
  });
  assert.equal(bad.ok, false);
  if (!bad.ok) {
    assert.ok(bad.error instanceof Error);
    assert.equal(bad.error.message, 'a thrown string, not an Error');
  }

  // attempt never throws, even when what was thrown cannot be turned into a string.
  const worst = await attempt(async () => {
    throw Object.create(null);
  });
  assert.equal(worst.ok, false);
});
```

</details>

```bash
npx tsc --noEmit
npx tsx src/domain/domain.check.ts      # 14 "ok - ..." lines
```

**Prove the type-level checks can fail.** A check that cannot fail proves nothing. Change `Partial<Pick<Task, 'status'>>` to
`Pick<Task, 'status'>` in `task.ts` and run `npx tsc --noEmit`. You should see `TS2344: Type 'false' does not satisfy the constraint
'true'` on the `_StatusOptionalOnCreate` line, and `Property 'status' is missing ... but required` at every call that omits it. Put it
back.

**Done when:**

- [ ] `npx tsc --noEmit` prints nothing and exits 0, with `task.ts`, `result.ts` and `domain.check.ts` in place.
- [ ] `npx tsx src/domain/domain.check.ts` exits 0 and prints 14 `ok - ...` lines.
- [ ] `Result<T, E>` is a union with a `true` / `false` literal discriminant, and inside `if (result.ok) { ... }` only `value` is
      accessible (the check file's `@ts-expect-error` lines prove it).
- [ ] You can explain what would be different if `Result` had been modelled as `{ ok: boolean; value?: T; error?: E }`.

### Stage 8 — Run it three ways: `tsx`, native type stripping, and a `tsc` build

**Goal:** run the ported toolkit with `tsx`, attempt it with Node's native type stripping and record what happens on Node 24, and
know what a `tsc` build is for.

Run these from the stage folder, with Node 24 active. Print the version in the same command.

1. **`tsx`, this course's dev runner:**
   ```bash
   node --version && npx tsx src/toolkit/async-toolkit.check.ts
   ```
   It should print 23 `ok` lines and exit 0.
2. **Native type stripping**: Node itself running a `.ts` file, with no tool in between:
   ```bash
   node --version && node src/toolkit/async-toolkit.check.ts
   ```
   Write down what happens. In the reference build, on Node v24.22.0, this printed the same 23 `ok` lines and exited 0, with no flag
   and no warning. Then do the same for the domain checks, which exist since Stage 7: `npx tsx src/domain/domain.check.ts` and
   `node src/domain/domain.check.ts` (14 `ok` lines each). Your result is the evidence the brief asks for. (`npm run check` and
   `npm run check:native` also run `env.check.ts`, which you create in Stage 9, so run them there.) `node -p process.features.typescript`
   prints `strip` on that version.
3. **Make native Node fail, and read why.** In a new folder, `scratch/strip/`, create these eight small files, each named in bold
   above its code, and run each with `node` and with `tsx`:

   **`scratch/strip/a-enum.ts`**

   ```ts
   enum Color { Red, Green }
   console.log(Color.Red);
   ```

   **`scratch/strip/b-paramprop.ts`**

   ```ts
   class P { constructor(public x: number) {} }
   console.log(new P(1).x);
   ```

   **`scratch/strip/lib.ts`** (imported by `c`, `d`, `e` and `f`)

   ```ts
   export type Point = { x: number; y: number };
   export const origin: Point = { x: 0, y: 0 };
   ```

   **`scratch/strip/c-notype.ts`**

   ```ts
   import { Point, origin } from './lib.ts';
   const p: Point = origin;
   console.log(p);
   ```

   **`scratch/strip/d-type-ok.ts`**

   ```ts
   import { type Point, origin } from './lib.ts';
   const p: Point = origin;
   console.log(p);
   ```

   **`scratch/strip/e-noext.ts`**

   ```ts
   import { origin } from './lib';
   console.log(origin);
   ```

   **`scratch/strip/f-js.ts`**

   ```ts
   import { origin } from './lib.js';
   console.log(origin);
   ```

   **`scratch/strip/g-badtype.ts`**

   ```ts
   const n: number = 'not a number';
   console.log(n);
   ```

   ```bash
   for f in a-enum b-paramprop c-notype d-type-ok e-noext f-js g-badtype; do
     out=$(node scratch/strip/$f.ts 2>&1); echo "=== node $f.ts"; echo "$out" | grep -m1 "Error" || echo "$out"
     echo "=== tsx $f.ts"; npx tsx scratch/strip/$f.ts 2>&1 | head -2
   done
   ```
   (A failing Node run prints a long stack of internals. `grep -m1 "Error"` keeps the one line that names the problem, and when there
   is no error line the whole output is shown instead.)
   These files are ES modules because your stage folder's `package.json` has `"type": "module"`. In the reference build, on Node
   v24.22.0:

   | File | `node file.ts` | `tsx file.ts` |
   | --- | --- | --- |
   | `a-enum.ts` | `SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript enum is not supported in strip-only mode` | runs, prints `0` |
   | `b-paramprop.ts` | `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`: `parameter property is not supported in strip-only mode` | runs, prints `1` |
   | `c-notype.ts` | `SyntaxError: The requested module './lib.ts' does not provide an export named 'Point'` | runs |
   | `d-type-ok.ts` | prints `{ x: 0, y: 0 }` | runs |
   | `e-noext.ts` | `ERR_MODULE_NOT_FOUND` (cannot find `.../lib`) | runs |
   | `f-js.ts` | `ERR_MODULE_NOT_FOUND` (cannot find `.../lib.js`) | runs |
   | `g-badtype.ts` | prints `not a number` | prints `not a number` |

   Last row first: neither runner checked the type. The type error in `g-badtype.ts` is found only by a checker: add a
   `scratch/strip/tsconfig.json` like Stage 3's and run `npx tsc -p scratch/strip`. For the other rows, the checker is your safety net
   too: with your `tsconfig` it rejects `a-enum` and `b-paramprop` (`erasableSyntaxOnly`), `c-notype` (`verbatimModuleSyntax`) and
   `e-noext` (a relative import needs an extension in `nodenext`). **`f-js.ts` is the exception**: `tsc` accepts a `.js` specifier for a
   `.ts` file, `tsx` runs it, and only native Node fails, so no compiler error warns you about it.
4. **A `tsc` build (awareness only).** Your project never emits, but you should have seen what emitting looks like. In `scratch/build/`,
   make a tiny two-file project and build it:
   ```bash
   mkdir -p scratch/build/src && cd scratch/build
   echo '{"type":"module"}' > package.json
   echo "export const greet = (n: string): string => \`hi \${n}\`;" > src/lib.ts
   printf "import { greet } from './lib.ts';\nconsole.log(greet('built'));\n" > src/main.ts
   echo '{"compilerOptions":{"strict":true,"module":"nodenext","target":"es2024","types":["node"],"verbatimModuleSyntax":true,"erasableSyntaxOnly":true,"rewriteRelativeImportExtensions":true,"rootDir":"src","outDir":"dist"},"include":["src"]}' > tsconfig.json
   ln -s ../../node_modules node_modules
   npx tsc -p . && cat dist/main.js && node dist/main.js
   cd ../..
   ```
   (`ln -s` lets the scratch project find your installed `@types/node`.) `dist/main.js` should contain `import { greet } from './lib.js';`, and `node dist/main.js` should print `hi built`. Now delete
   `rewriteRelativeImportExtensions` from that `tsconfig.json` and rebuild: you should get `TS5097`, because a `.ts` specifier cannot
   be emitted as it is. Delete `rootDir` too and you should get `TS5011`, because TypeScript 7 wants the output layout stated.

**Done when:**

- [ ] `npx tsx src/toolkit/async-toolkit.check.ts` exits 0 and prints 23 `ok - ...` lines (and `npm run check` prints 40 after Stage 9).
- [ ] You ran `node src/toolkit/async-toolkit.check.ts` on Node 24 and wrote down exactly what happened (the result, the Node version
      printed by `node --version`, and any warning). This is awareness-level on Node 24, not a pass-or-fail requirement.
- [ ] You can name the three rules your code follows so that native stripping works (erasable syntax, `type` imports, real file
      extensions), say which of the probe files broke which rule, and say which one `tsc` does not warn you about (`f-js.ts`).
- [ ] You can say what `tsx`, native stripping and `tsc --noEmit` each do and which of them checks types.

### Stage 9 — Async code, errors, `process.env`, `@types` and `.d.ts`

**Goal:** type the three places where untyped data enters a Node program, and understand what `@types` packages and `.d.ts` files are.

1. **Typing `process.env`, and the demo.** Create these four files, each named in bold above its code. `readEnv` reads the environment
   once, validates it, and returns a `Result`, so the rest of the program never sees `string | undefined`. It takes the env object as a
   parameter, so the checks can pass a plain object. It treats an unset, empty or whitespace-only value as "use the default", and
   accepts only whole digits otherwise: `Number()` alone would turn `' '` into `0` and accept `'0x10'` and `'1e3'`.

   **`src/env.ts`**

   ```ts
   // Module 4 output: typed access to process.env.
   // Nothing to run here. Used by demo.ts and checked by env.check.ts.
   import { err, ok, type Result } from './domain/result.ts';

   export type Env = {
     /** How long the demo's simulated save takes, in ms. */
     toolkitDelayMs: number;
   };

   // process.env is typed `NodeJS.ProcessEnv`: every value is `string | undefined`, whatever the
   // variable really holds. Read it once, here, check it, and hand the rest of the program a
   // proper Env. Taking `env` as a parameter (defaulting to process.env) makes it testable.
   export function readEnv(env: NodeJS.ProcessEnv = process.env): Result<Env, string> {
     const raw = env.TOOLKIT_DELAY_MS?.trim();
     if (raw === undefined || raw === '') return ok({ toolkitDelayMs: 20 });
     // Digits only: Number() would also accept '0x10' and '1e3', and turn whitespace into 0.
     if (!/^\d+$/.test(raw)) {
       return err(`TOOLKIT_DELAY_MS must be a whole number of milliseconds, got "${raw}"`);
     }
     return ok({ toolkitDelayMs: Number(raw) });
   }
   ```

   **`src/env.check.ts`**

   ```ts
   // Module 4 output: self-checks for the typed environment reader.
   // Run from this folder: npm run check   (or: npx tsx src/env.check.ts)
   import assert from 'node:assert/strict';
   import { readEnv } from './env.ts';

   function check(name: string, fn: () => void): void {
     fn();
     console.log(`ok - ${name}`);
   }

   check('readEnv falls back to the default when the variable is unset or empty', () => {
     assert.deepEqual(readEnv({}), { ok: true, value: { toolkitDelayMs: 20 } });
     assert.deepEqual(readEnv({ TOOLKIT_DELAY_MS: '' }), { ok: true, value: { toolkitDelayMs: 20 } });
     // A whitespace-only value counts as unset too: Number('  ') is 0, which would silently mean "no delay".
     assert.deepEqual(readEnv({ TOOLKIT_DELAY_MS: '   ' }), { ok: true, value: { toolkitDelayMs: 20 } });
   });

   check('readEnv parses a valid number', () => {
     assert.deepEqual(readEnv({ TOOLKIT_DELAY_MS: '5' }), { ok: true, value: { toolkitDelayMs: 5 } });
     assert.deepEqual(readEnv({ TOOLKIT_DELAY_MS: ' 7 ' }), { ok: true, value: { toolkitDelayMs: 7 } });
     assert.deepEqual(readEnv({ TOOLKIT_DELAY_MS: '0' }), { ok: true, value: { toolkitDelayMs: 0 } });
   });

   check('readEnv returns an error Result for a bad value instead of throwing', () => {
     // '0x10' and '1e3' are numbers to Number(), but not a plain count of milliseconds.
     for (const bad of ['soon', '-1', 'NaN', '0x10', '1e3', '1.5']) {
       const result = readEnv({ TOOLKIT_DELAY_MS: bad });
       assert.equal(result.ok, false);
       if (!result.ok) assert.match(result.error, /TOOLKIT_DELAY_MS/);
     }
   });
   ```

   **`src/demo.ts`**

   ```ts
   // Module 4 output: a small run of the whole toolkit with the Task types and Result.
   // Run from this folder: npm run demo
   import { attempt, unwrapOr } from './domain/result.ts';
   import { createTask, type CreateTaskInput, type Task } from './domain/task.ts';
   import { readEnv } from './env.ts';
   import { mapLimit, retry, sleep, withTimeout } from './toolkit/async-toolkit.ts';

   const env = readEnv();
   if (!env.ok) {
     console.error(env.error);
     process.exit(1);
   }
   const { toolkitDelayMs } = env.value;

   const inputs: CreateTaskInput[] = [
     { title: 'Set up tsconfig' },
     { title: 'Port the toolkit', status: 'in_progress' },
     { title: 'Model Result<T, E>', description: 'discriminated union' },
     { title: 'Write the guide' },
     { title: 'Open the PR' },
   ];

   // A pretend save: takes `toolkitDelayMs`, then builds the Task.
   async function save(input: CreateTaskInput, index: number): Promise<Task> {
     await sleep(toolkitDelayMs);
     return createTask(`task-${index + 1}`, input);
   }

   // mapLimit<CreateTaskInput, Task>: at most 2 saves at a time, results in input order.
   const tasks = await mapLimit(inputs, 2, save);
   for (const task of tasks) console.log(`${task.id}  [${task.status}]  ${task.title}`);

   // retry<Task>: the first two tries fail, the third succeeds.
   const flaky = await retry(
     async (tryNumber) => {
       if (tryNumber < 2) throw new Error(`save failed on attempt ${tryNumber}`);
       return save({ title: 'Flaky save' }, 99);
     },
     { retries: 3, delayMs: 5 },
   );
   console.log(`retry: ${flaky.title} saved`);

   // withTimeout + attempt: a save slower than the timeout becomes an error Result, not a throw.
   const slow = await attempt(() => withTimeout(sleep(toolkitDelayMs + 200).then(() => 'late'), toolkitDelayMs));
   console.log(slow.ok ? `slow: ${slow.value}` : `slow: ${slow.error.name}: ${slow.error.message}`);
   console.log(`unwrapOr: ${unwrapOr(slow, 'fallback value')}`);
   ```

   **`.env.example`** (in the stage folder)

   ```text
   # Module 4 setting for the demo: copy this file to .env in this folder.
   # npm run demo loads .env if it exists. Never commit .env.

   # How long the demo's simulated save takes, in milliseconds (optional, default 20).
   TOOLKIT_DELAY_MS=20
   ```

   Add one line to the **repo root's** `.env.example`, which is an index of each module's settings:
   ```bash
   echo "# M04, modules/m04-typescript-on-node/.env.example: TOOLKIT_DELAY_MS, how long the demo's simulated save takes in ms (optional, default 20)" >> ../../.env.example
   ```
   Then run:
   ```bash
   npx tsc --noEmit
   npm run check          # 23 + 14 + 3 = 40 "ok - ..." lines
   npm run demo
   TOOLKIT_DELAY_MS=soon npm run demo
   ```
   `npm run demo` should print (the first line appears only while no `.env` file exists):
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
   and the last command should print `TOOLKIT_DELAY_MS must be a whole number of milliseconds, got "soon"` and exit 1. The `demo` script
   is `node --env-file-if-exists=.env --import tsx src/demo.ts`: one Node process, with `tsx` loaded as an import hook. Launching it
   as `tsx --env-file-if-exists=.env src/demo.ts` also works, but `tsx` starts further Node processes and each one prints the
   "`.env` not found" line, so you see it two or three times.
2. **Typing practice.** Create `scratch/typing/` with a `tsconfig.json` like Stage 3's (`echo '{ "extends": "../../tsconfig.json", "include": ["*.ts"] }' > scratch/typing/tsconfig.json`), then create `scratch/typing/errors.ts`:
   ```ts
   async function f(): Promise<number> { return 'x'; }
   const w: number = f();
   try { JSON.parse('{'); } catch (e) { console.log(e.message); }
   try { JSON.parse('{'); } catch (e) { if (e instanceof Error) console.log(e.message); }
   export { w };
   ```
   and `scratch/typing/env.ts`:
   ```ts
   const port: number = process.env.PORT;
   const checked = Number(process.env.PORT ?? '3000');
   export { port, checked };
   ```
   Run `npx tsc -p scratch/typing`. You should see, for `errors.ts`, `TS2322: Type 'string' is not assignable to type 'number'` (an `async`
   function declared `Promise<number>` must return a number), `TS2322: Type 'Promise<number>' is not assignable to type 'number'`
   (you forgot an `await`), and `TS18046: 'e' is of type 'unknown'` (the first `catch`, and not the second). For `env.ts`:
   `TS2322: Type 'string | undefined' is not assignable to type 'number'`.
3. **`@types` packages.** Prove what `types: ["node"]` and `@types/node` do. Make a copy of
   `tsconfig.json` without the `types` line (`sed '/"types"/d' tsconfig.json > tsconfig.notypes.json`) and run
   `npx tsc -p tsconfig.notypes.json`: the whole project falls over (33 errors in the reference build). `console`, `process`, `setTimeout`,
   `performance`, `URL` and `AbortSignal` become "Cannot find name" (`TS2584`, `TS2591`, `TS2304`), and `node:assert` and `node:util` cannot be
   found, even though `@types/node` is installed. One error is not a missing name: in `demo.ts`, after
   `if (!env.ok) { ...; process.exit(1); }` the compiler no longer narrows `env`, because without `@types/node` it does not know
   that `process.exit` returns `never` (it never returns), so it cannot tell the code after the `if` only runs when `env.ok` is
   `true`. Delete the copy (`rm tsconfig.notypes.json`).
4. **A `.d.ts` file.** Plain JavaScript has no types. Copy your M02 toolkit into a new `scratch/dts/` folder, and import it from a `.ts` file:
   ```bash
   mkdir -p scratch/dts
   echo '{ "extends": "../../tsconfig.json", "include": ["*.ts", "*.mts"] }' > scratch/dts/tsconfig.json
   cp ../m02-modern-javascript-async-await/async-toolkit.mjs scratch/dts/
   printf "import { sleep } from './async-toolkit.mjs';\nawait sleep('1');\n" > scratch/dts/use-m02.ts
   ```
   (Use your own M02 file path if it differs.) Run `npx tsc -p scratch/dts`. You should see `TS7016: Could not find a declaration file for module './async-toolkit.mjs'. '...' implicitly
   has an 'any' type.` Now add a **declaration file** next to it, `scratch/dts/async-toolkit.d.mts`, containing one line:
   ```ts
   export declare function sleep(ms: number): Promise<void>;
   ```
   Run `npx tsc -p scratch/dts` again. The `TS7016` disappears and `await sleep('1')` is now `TS2345: Argument of type 'string' is not
   assignable to parameter of type 'number'`. A `.d.ts` describes the types of code without containing the code. `@types/node` is a
   package of them, and `@types/*` packages are how the types for untyped libraries are shared. Finally run
   `npx tsx scratch/dts/use-m02.ts`: the compiler complained, and `tsx` runs the file anyway. M02's own runtime check then throws a
   `RangeError`. Types did not stop it. The `RangeError` did.

**Done when:**

- [ ] `npm run check` exits 0 and prints 40 `ok - ...` lines, and `npm run demo` prints the eight lines shown above.
- [ ] `TOOLKIT_DELAY_MS=soon npm run demo` prints the error message and exits 1 (`echo $?`).
- [ ] You can explain why a `catch` variable is `unknown`, what type `process.env.X` has and what `readEnv` does about it, and what an
      `async` function's declared return type has to be.
- [ ] You can explain what `@types/node` and a `.d.ts` file are for, from the two experiments.

### Stage 10 — TypeScript 6 to 7

**Goal:** know what changed between TypeScript 6 and 7 by running both, and what the missing programmatic API means for tooling you will
meet again in M09 (the testing module).

Do this in a scratch folder **outside** your repo, so version 6 never touches your project:

```bash
mkdir -p ~/ts6-probe && cd ~/ts6-probe
nvm use 24            # on its own line; there is no .nvmrc here, so check the version
node --version
echo '{"name":"ts6probe","private":true,"type":"module"}' > package.json
npm install typescript@6 @types/node@24
node_modules/.bin/tsc --version
```

Create `a.ts` and a `tsconfig.json` with **no** `strict` option, then run the TypeScript 6 you just installed and your project's 7:

```bash
cat > a.ts <<'EOF'
function f(x) { return x; }
let s: string = null;
export {};
EOF
echo '{"compilerOptions":{"noEmit":true,"module":"nodenext"},"files":["a.ts"]}' > tsconfig.json
node_modules/.bin/tsc -p .
~/<repo-name>/modules/m04-typescript-on-node/node_modules/.bin/tsc -p .
```

1. **Strict.** In the reference build both versions printed `TS7006` and `TS2322`: 6.0.3 was already strict by default, so in what
   was run, strict-by-default is **not** something that changed between 6.0.3 and 7.0.2. Trust your own run over any summary, and
   check the official release notes.
2. **Legacy options.** Replace the config with one legacy option at a time (`"target":"es5"`, `"moduleResolution":"node10"`,
   `"baseUrl":"."`, `"downlevelIteration":true`, `"esModuleInterop":false`) and run both. TypeScript 6.0.3 said
   `TS5107` / `TS5101`: *"Option '...' is deprecated and will stop functioning in TypeScript 7.0. Specify compilerOption
   '"ignoreDeprecations": "6.0"' to silence this error."* TypeScript 7.0.2 said `TS5108` / `TS5102`: *"Option '...' has been removed.
   Please remove it from your configuration."* (For `baseUrl` it added: *Use `"paths": {"*": ["./*"]}` instead.*) So legacy options were
   deprecated in 6 and are gone in 7.
3. **Speed.** Time both compilers on **your project**, calling each binary directly (your project's `tsconfig.json` and installed
   `@types/node` work for either version):
   ```bash
   cd ~/<repo-name>/modules/m04-typescript-on-node
   for i in 1 2 3; do time ~/ts6-probe/node_modules/.bin/tsc --noEmit; done     # TypeScript 6
   for i in 1 2 3; do time node_modules/.bin/tsc --noEmit; done                 # TypeScript 7
   ```
   Do not time `npx tsc` or `npm run typecheck`: they add start-up time of their own (in the reference build `npx tsc --noEmit` took
   about 0.24 s and `npm run typecheck` about 0.14 s on TypeScript 7, which hides the difference). Called directly, the reference project
   (about 700 lines) took about 0.23 s on 6.0.3 and 0.04 s on 7.0.2, three runs each. That is a small project, so it is not a
   benchmark. The point is that the gap was the same on every run.
4. **The programmatic API.** The compiler used to be a JavaScript library you could `import`:
   ```bash
   cd ~/ts6-probe
   node -e "import('typescript').then(m=>{const t=m.default??m; console.log(typeof t.createProgram, typeof t.transpileModule, t.version)})"
   cd ~/<repo-name>/modules/m04-typescript-on-node
   node -e "import('typescript').then(m=>{const t=m.default??m; console.log(Object.keys(m), typeof t.createProgram, typeof t.transpileModule, t.version)})"
   ```
   TypeScript 6.0.3 printed `function function 6.0.3`. TypeScript 7.0.2 printed the keys `default`, `module.exports`, `version` and
   `versionMajorMinor` (the first two are module-interop names, so the real exports are the last two), and `typeof` both `createProgram` and
   `transpileModule` was `undefined`. Look at `node_modules/typescript/package.json`: its `exports` lists `./unstable/...` entry points, and
   `ls node_modules/@typescript` shows a platform-specific native binary package. The compiler is now a native program, and what
   it offers to other programs is marked `unstable`.
5. **What it means.** Any tool that used `createProgram` or `transpileModule` to compile or check TypeScript from inside Node, such
   as a test runner's TypeScript transform, has to take another route: use `tsc` as a separate step, use a different transpiler, or
   use the unstable API. You will meet this again in M09. This is why the course keeps `tsc --noEmit` as its own gate (`npm run typecheck`)
   and does not rely on the test runner to check types.

**Done when:**

- [ ] You ran the same `tsconfig` with TypeScript 6 and 7 and have the results written down.
- [ ] You can state three things you saw differ or not differ: what happens with a legacy option, what `import('typescript')` exports,
      and whether strict-by-default differed in your run.
- [ ] You can explain in your own words what the missing programmatic API means for a tool that wants to compile TypeScript.

### Stage 11 — The gate, then commit, push, pull request and merge

**Goal:** prove the whole deliverable once more from a clean state, then deliver the stage as M01-M03 did: a branch, Conventional
Commits and a pull request, with nothing pushed straight to `main`.

Run every command from your stage folder, with Node 24 active (`nvm use` on its own line).

#### Step 1 — Re-run the gate and the checks

```bash
node --version
rm -rf node_modules && npm ci
npm run typecheck && echo "typecheck exit 0"
npm run check
npm run check:native
npm run demo
git status -sb
```

**Why:** `npm ci` proves the lockfile alone rebuilds the project (in the reference build it printed `added 8 packages ... found 0 vulnerabilities`,
plus install-script warnings like Stage 1's), and the PR's test plan should list results you have just seen.
`git status -sb` should show the stage branch, the root `.env.example` modified, and the stage folder untracked, with **no**
`node_modules/` and no `.env`. `scratch/` will appear: do **not** commit it, and stage by name below.

#### Step 2 — Commit in one group per concern

Stage files **by name or by folder**, not with `git add .`:

```bash
git add package.json package-lock.json tsconfig.json .env.example ../../.env.example
git commit -m "chore(m04): add package.json, tsconfig, lockfile and env example"
git add src/toolkit
git commit -m "feat(m04): port the async toolkit to TypeScript with generics"
git add src/domain src/env.ts src/env.check.ts src/demo.ts
git commit -m "feat(m04): add Task domain types, Result, typed env and demo"
```

#### Step 3 — Push the branch and open the pull request

```bash
git push -u origin feat/module-04/typescript-on-node
gh pr create --repo <user>/<repo-name> --base main --head feat/module-04/typescript-on-node \
  --title "feat(m04): port the async toolkit to TypeScript and add the Task domain types" \
  --body "<your description>"
```

Replace `<user>` and `<repo-name>` with your GitHub username and repo name. The description has three sections:

- **Summary:** why the stage exists and what it adds, in one short paragraph.
- **Changes:** one bullet per notable change (project setup, toolkit, domain types, `Result`, typed env, checks).
- **Test plan:** one checked item per command you ran in Step 1, with what it printed (for example "`tsc --noEmit` exit 0", "40 `ok` lines
  under `tsx` and 40 under native Node", the demo output). Never tick a box for something you did not run.

#### Step 4 — Merge it and look at the history

Merge the PR on GitHub with a **merge commit**, not a squash, so your three commits stay visible. Then:

```bash
git fetch origin
git log --oneline --graph origin/main -8
```

The graph should show a `Merge pull request #<number> from ...` commit on `main` with your three `...(m04): ...` commits under it. GitHub
writes that message, so it is not a Conventional Commit. That is expected.

**Done when:**

- [ ] `npm run typecheck` exits 0 with no output, after `npm ci` from a clean `node_modules`.
- [ ] `git log --oneline main..HEAD` on the stage branch shows three commits (`chore(m04)`, `feat(m04)`, `feat(m04)`), and nothing from
      `scratch/` is in them (`git show --stat HEAD~2 HEAD~1 HEAD` lists no `scratch/` path).
- [ ] A pull request from `feat/module-04/typescript-on-node` into `main` exists on your build repo, with Summary, Changes and Test plan
      sections.
- [ ] The PR is merged, and `git log --oneline --graph origin/main -8` shows its merge commit with your three commits under it.

## 4. Concepts you need

Start with the **quick reference** if a term is new. The **essays** below it explain the bigger ideas. Read each essay after the stage named
beside it.

How far to trust each statement:

- **Unmarked:** you ran or measured it in a guided step.
- **Marked *(background)*:** standard TypeScript and Node knowledge that the steps do not prove.
- **Marked *(reference build)*:** seen while building this guide, but not a guided step. You can try it yourself in a scratch file.

Check the marked points against the TypeScript and Node docs before relying on them.

### Quick reference

| Concept | In one line | Used in |
| --- | --- | --- |
| **Structural typing** | A value fits a type if it has the right shape. The name does not matter. | Stage 3 |
| **Inference vs annotation** | The compiler works out types from values. Annotate parameters and where inference would be too wide. | Stage 3 |
| **Literal type, `as const`** | `'todo'` is a type. `as const` freezes an array into a `readonly` tuple of literals. | Stages 3, 6 |
| **Union `A \| B`, intersection `A & B`** | Either one, or both at once. | Stages 3, 6, 7 |
| **Generic `<T>`** | A type parameter the caller fills in, so one function keeps the caller's types. | Stages 3, 4 |
| **Narrowing** | An `if`, `switch` or guard leaves a smaller type for the code after it. | Stages 3, 6, 7 |
| **Discriminated union** | A union whose variants share a literal property, which tells them apart. | Stage 7 |
| **Type guard `x is T`** | A function whose `true` result narrows its argument. | Stage 6 |
| **`unknown`, `any`, `never`** | Unproven value, checking off, no value. See the essay. | Stages 3, 6, 9 |
| **Utility types** | Functions from types to types: `Partial`, `Required`, `Pick`, `Omit`, `Record`, `Readonly`, `ReturnType`, `Awaited`. | Stages 3, 6 |
| **`satisfies`** | Check a value against a type without widening it to that type. | Stages 3, 6 |
| **`import type`** | Marks an import as type-only, so it is erased entirely. | Stages 3, 8 |
| **`tsc --noEmit`** | The checker, with no output. The type-check gate. | Stages 2, 5, 11 |
| **`tsx`** | A runner: runs `.ts` files, does not check types. | Stage 8 |
| **Native type stripping** | Node deleting the types itself. Erasable syntax only. Does not check types. | Stage 8 |
| **`.d.ts`, `@types/*`** | Files that describe the types of JavaScript without containing it. | Stage 9 |

### `unknown`, `any` and `never` (after Stage 3)

**In short:** these three are the top, the escape hatch and the bottom of TypeScript's types. Knowing where each sits tells you when to
reach for it.

- **A type is a set of values** *(background)*. `string` is every string, `'todo'` is one value, `string | number` is the union of two
  sets. Assigning `A` to `B` is allowed when every value of `A` is also in `B`.
- **`unknown` is the top: the set of everything.** Every value can be assigned *to* it, and nothing comes back *out* of it until you prove
  more. That is why you must narrow it. A test such as `typeof`, `instanceof`, `in` or a type guard shrinks the set to something you may use.
  A guard `value is T` is a promise you make to the compiler, which trusts you, so its body has to be right.
- **`never` is the bottom: the empty set.** It can be assigned to everything (there is no value to get wrong), and nothing can be assigned
  to it. You met it three ways: as the type that makes the exhaustiveness check in Core 3 work, as the quiet result of a clashing
  intersection in Core 4, and as the return type of a function that never returns, which is why the compiler narrows after
  `process.exit()` *(background for the return-type rule)*. Adding a member to a union gives you a to-do list of every `switch` that forgot it.
- **`any` is neither: it opts out.** It is assignable to everything and accepts everything, so the compiler stops checking anything it
  touches, and the error shows up far from its cause, at run time. Treat it like a cast you have to justify, and keep it as small as you can.
- **Where `unknown` belongs** *(background)*: at every edge where data enters from outside the program, such as a parsed request body, a
  file, a caught error or an environment variable. `JSON.parse` is typed to return `any` *(background)*, so give its result the type
  `unknown` and narrow it once, at the edge, so the rest of the code works with real types. The course's coding standards say the same:
  prefer `unknown`, and if `any` appears, the text must say why.

**Mental model:** `unknown` is a sealed parcel: anyone can post one, and you must open it and look before you use what is inside. `any` is
a parcel you have agreed never to open and to trust. `never` is the empty parcel that cannot exist.

### `type` and `interface` (after Stage 3)

**In short:** an `interface` is open and a `type` alias is closed. Most of the differences you saw in Core 4 follow from that one idea.

- **Closed means complete and computable.** A `type` alias names exactly one thing, once, and that thing can be anything the type system
  can compute: a union, a tuple, a mapped or conditional type, a utility-type result. That is why `TaskStatus`, `Result<T, E>`, `CreateTaskInput`
  and `UpdateTaskInput` are all `type`: each is *derived*, not declared.
- **Open means extensible.** An `interface` can be added to, from another declaration or another file, and merged into one type. That is
  what a library wants, so that you can add a property to its type. It is also how a type can change under you by accident.
- **The same idea explains the error timing** *(background, as an explanation of what Core 4 showed)*. `extends` checks a clash when the
  interface is declared, because the interface is a finished contract at that point. An intersection is only computed when it is used, so
  the clash surfaces there.
- **Classes** *(background)* can `implements` either, as long as the shape is an object. Nothing in this module needs a class, and
  `erasableSyntaxOnly` rules out the class features (parameter properties) that people often reach for first.
- **A rule to start from:** use `type` by default, because this course constantly derives one type from another. Use `interface` for a
  named object contract that other code is meant to extend or merge into. Pick one style for a codebase and stay with it. Claims that one
  is faster than the other are *(background)* and not something these steps measure.

### Derive, don't copy: `Omit` and `Pick` (after Stage 6)

**In short:** a derived type stays in step with its source, but each way of deriving has a default, and the default decides what happens
to fields you add later.

A utility type is a function from a type to a type. Writing `CreateTaskInput` by hand repeats facts already in `Task` (that `title` is
a string, that `description` is optional), and the two copies can drift. Deriving keeps one source of truth.

The two ways to say "Task, minus the server's fields" differ in what they do to a **new** field. Try it: in a scratch copy of `task.ts`, add
`dueDate?: Date` to `Task`, then assign `{ title: 'a', dueDate: new Date() }` to three versions: your `Omit`-derived `CreateTaskInput`, a
hand-written `{ title: string; description?: string; status?: TaskStatus }`, and a `Pick`-derived
`Pick<Task, 'title' | 'description'> & Partial<Pick<Task, 'status'>>`. In the reference build:

| Version | Before `dueDate` exists | After `dueDate` is added to `Task` |
| --- | --- | --- |
| `Omit`-derived | rejected (`TS2353`) | **accepted**, with no edit |
| Hand-written | rejected | still rejected |
| `Pick`-derived | rejected | still rejected |

- **`Omit` is a deny-list:** everything is allowed except what you name. A new `Task` field is writable by clients **by default**, until you
  remember to add it to the omitted list.
- **`Pick` is an allow-list:** nothing is allowed except what you name. A new field is invisible to clients until you opt it in.
- **Which to use** is a question about who is the authority. For an input the server validates, an allow-list (`Pick`) fails safe: forgetting to
  update it hides a field. For a type that mirrors a source and should follow it (a view, a read model), `Omit` follows the source. Decide
  this *before* you write the type, by asking whether a new field should be client-writable by default.
- **Hand-written** fails safe for a new field, as the table shows, but it is a second copy of what `Task` already says. It does not follow
  the source, so a change to `Task` (the type of `title`, say) has to be repeated by hand, and a stale copy stays stale until something
  notices.
- **A derived type does not guard the code that uses it.** A type only constrains what the compiler can see. A function that spreads
  an object into a `Task` copies whatever is in the object at run time, including keys the type never mentioned. Stage 6's `applyTaskUpdate`
  names its fields for that reason.

### Runner or checker: `tsx`, native stripping and `tsc` (after Stage 8)

**In short:** none of these three does both jobs. `tsc` checks and does not run (here). `tsx` and `node` run and do not check.

- **A checker** reads your types and says whether the program is consistent. **A runner** executes the program. TypeScript's types are
  *erased* before the program runs, so at run time there is only JavaScript. A green `node file.ts` or `tsx file.ts` therefore tells you
  nothing about types, and the only thing that does is `tsc --noEmit`, which is why it is a script of its own and a step of every pull
  request.
- **Strip or transform.** Native type stripping *deletes* type syntax and runs what is left, so it can only run code where deleting the
  types leaves valid JavaScript. `enum` and parameter properties are not like that: they need code *generated*, so they are not
  *erasable*. `tsx` transforms the file, so it can run them. Node's `--experimental-transform-types` flag is Node's own way of generating
  code for those, and printed an `ExperimentalWarning` *(reference build)*.
- **Why this course keeps `erasableSyntaxOnly` on, while using `tsx`.** `tsx` would run `enum`s, but then the code would stop being
  runnable by plain Node. The option makes the compiler stop you at the moment you write something native stripping could not run, while
  you are still at your desk. The cost is small, because a union of string literals does the job of an `enum`.
- **`verbatimModuleSyntax` is the same kind of guard.** A stripper only sees syntax, so it cannot know that an imported name is only a type
  and leaves the import in place *(background)*. The option forces you to say which imports are types, so every tool gets it right.
- **File extensions follow the runtime.** Node resolves files by their real names, so native Node needs `./lib.ts`, and `tsx` is lenient. A
  project that will be *built* wants the opposite: emitted files are `.js`, so the import should say `.js`, or the build rewrites it
  (`rewriteRelativeImportExtensions`). The program's choice between these is the open decision P2. Whichever one a stage uses, say which.
- **Which tool for what** *(background, as the course uses them)*: `tsx` for daily development and running scripts, `tsc --noEmit` as the
  gate, native stripping as something to understand because Node 24 already runs `.ts` files, and a `tsc` build when you need `.js` files to ship.

**Mental model:** a spell-checker, a printer and a photocopier. `tsc` is the spell-checker, and `tsx` and `node` are printers that print
whatever you give them, typos included.

### TypeScript 6 to 7, and the missing API (after Stage 10)

**In short:** version 7 is a new native implementation of the compiler. It is much faster, it is a clean break from legacy settings, and
the library API that other tools relied on is no longer in the usual place.

- **Why a deprecation release exists.** A version that warns before it removes gives you a window to fix a config while each message
  still names what to change. The sensible route is to move a project to 6 first, clear the warnings, and only then move to 7.
- **Why speed is the reason for the rewrite** *(background)*: a compiler that is a native program does not need a JavaScript runtime to
  start, and it can check in parallel. The gap you measured is small in absolute terms because the project is small. It grows with the
  size of the codebase.
- **A library API is a promise to other tools.** Editors, linters and test-runner transforms asked the compiler their questions from
  inside Node. When the main entry point stops offering that, each of those tools has to move to a different route, and the replacement is
  labelled `unstable`. That is a compatibility risk to plan for, not a bug: pin versions, keep type-checking as its own command, and read each
  tool's notes about which TypeScript versions it supports *(background)*.
- **What stays the same for you:** the language. A type error in 6 is a type error in 7. Most of what changed is *around* the language:
  configuration, the build, and the tools that embed the compiler.

## 5. Common pitfalls

| Mistake | What happens | Do this |
| --- | --- | --- |
| `nvm use \| tail -2` | Prints `Now using node v24`, but `node -v` still says 22: a pipe runs `nvm use` in a subshell | `nvm use` on its own line, then `node --version` |
| Running a probe in a folder with no `.nvmrc` | `nvm use` prints `No .nvmrc file found`, and your results come from the wrong Node | `nvm use 24`, and print `node --version` in the same command |
| `npm init -y` | Writes `"type": "commonjs"` | Set `"type": "module"` by hand |
| No `"types": ["node"]` | `console` and `process` are "Cannot find name", and `process.exit()` stops narrowing | List `node` in `types` |
| `lib` older than `es2024` | `Promise.withResolvers` is `TS2550` | `lib` and `target` `es2024` |
| `list[index]` under `noUncheckedIndexedAccess` | `T \| undefined` is not assignable to `T` | Iterate (`list.entries()`), don't cast with `as T` |
| `constructor(public attempts: number)` | `erasableSyntaxOnly` error, and native Node refuses it | Declare the field, assign in the constructor |
| `enum` | `TS1294` and native Node refuses it | A union of string literals, derived from an `as const` array |
| `import { SomeType } from './x.ts'` | `TS1484` under `verbatimModuleSyntax`. Under native Node a runtime `SyntaxError` | `import { type SomeType }` or `import type` |
| `import ... from './x.js'` for a `.ts` file | `tsc` and `tsx` accept it. Native Node: `ERR_MODULE_NOT_FOUND` | `./x.ts`, with `allowImportingTsExtensions`, or build first |
| `import ... from './x'` | `TS2835` in `nodenext`. Native Node: `ERR_MODULE_NOT_FOUND` | Write the extension |
| `const attempts = []` | `TS7034` / `TS7005`: implicit `any[]` | `const attempts: number[] = []` |
| Using `error.cause` or `e.message` in a `catch` | `TS18046`: `unknown` | Narrow first: `instanceof Error` |
| Changing a test to fix `TS2345` on deliberate bad input | The runtime check is no longer tested | `// @ts-expect-error` above the line, keep the `assert.throws` |
| Trusting `node file.ts` or `tsx file.ts` | Both print `not a number` for `const n: number = 'not a number'` | `npm run typecheck` is the only check |
| A type-level check that cannot fail | It proves nothing | Break the type on purpose once and see it fail |
| Fixing errors one by one after changing a definition | Most are call sites of the one you changed. Making `status` required on create gave 4 errors at Stage 7 (one assertion and three calls) and 9 once `demo.ts` existed | Fix the definition, re-run, then see what is left |
| `tsc` build with no `rootDir` | `TS5011` | Set `"rootDir": "src"` |
| `.ts` specifiers in a `tsc` build | `TS5097` | `rewriteRelativeImportExtensions`, or `.js` specifiers |
| Importing untyped JavaScript | `TS7016`: implicit `any` | Write a `.d.ts` next to it, or use JSDoc and `allowJs` |
| `tsx --env-file-if-exists=.env file.ts` | `.env not found. Continuing without it.` printed two or three times, because `tsx` starts more Node processes and each gets the flag | Harmless. The `demo` script avoids it with `node --env-file-if-exists=.env --import tsx file.ts` |
| `new URL('./x.mjs', ...)` or any file name inside a string, after renaming a file | `tsc` cannot see inside a string, so only a run shows the stale name (`ERR_MODULE_NOT_FOUND`) | Search for the old file name, not just the imports |
| `{ ...task, ...update }` to apply a partial update | `undefined` fields blank the task, and keys outside the input type (an object held in a variable is not checked for extras) overwrite `id` | Name each field: `update.title ?? task.title` |
| `String(thrown)` on an unknown thrown value | It throws for `Object.create(null)`, so a function meant never to throw does | `try`/`catch` it, with a fixed fallback message |
| `Number(process.env.X)` as validation | `' '` becomes `0`, and `'0x10'` and `'1e3'` are accepted | Trim, then test with `/^\d+$/` |
| A timing check with a tight upper bound | A busy machine fails a correct program | Make the waits long enough that the bounds have 80 ms or more of room |
| An unused `scratch/` folder | Shows up in `git status` | Stage files by name, and never `git add .` |

## 6. Check yourself against the definition of done

Tick each one only when you have just verified it.

- [ ] **Zero errors from `tsc --noEmit` on a strict config.** `npm run typecheck` prints nothing and exits 0, after a clean `npm ci`, and
      `tsconfig.json` has `strict`, `nodenext` module resolution, `verbatimModuleSyntax`, `erasableSyntaxOnly`,
      `noUncheckedIndexedAccess`, `noEmit` and `skipLibCheck`.
- [ ] **You can explain `unknown` against `any`, and `type` against `interface`, in your own words.** Without looking, you can say what each
      allows and forbids, give an example of your own for each pair, and say which you would pick and why.
- [ ] **The toolkit runs successfully via `tsx`.** `npx tsx src/toolkit/async-toolkit.check.ts` prints 23 `ok - ...` lines and exits 0, and
      `npm run check` prints 40.
- [ ] **You attempted native type stripping on Node 24 and documented what happened.** `node --version` printed `v24.x`, you ran
      `node src/toolkit/async-toolkit.check.ts` (or `npm run check:native`), and you wrote down the real result and at least one case where
      native Node refuses code that `tsx` runs. This is awareness-level on Node 24, so there is no pass-or-fail on the result itself.
- [ ] Supporting evidence for the deliverable: the ported toolkit exports `retry<T>`, `mapLimit<T, R>`, `sleep` and `withTimeout`; the Task
      domain has `Task`, `CreateTaskInput`, `UpdateTaskInput` (derived with utility types) and a discriminated-union `Result<T, E>`; and the
      stage is delivered by a merged pull request.

## 7. Next step

If every box in section 6 is ticked, this stage is done.

1. Keep the repo and carry the same workflow forward: a branch named `feat/module-NN/<topic>`,
   Conventional Commits and a pull request. Never push to `main` directly.
2. Move on to **M05 — Express: Routing, Middleware & Error Handling** (Week 2 · Days 3-5). It builds on this stage: the **Task domain
   types** you modelled here (`Task`, `CreateTaskInput`, `UpdateTaskInput`, `Result<T, E>`) become an actual API, so keep them and the
   project setup (`tsconfig.json`, `npm run typecheck`, `tsx`) on hand. This is also where the **Task API** starts: it carries
   forward through M07, M08 and M09 as v1, v2, v3 and v4.
3. Expect Express 5 in TypeScript, with the app and the server split into separate files. The M05 stage fixes the API's fields as `id`,
   `title`, an optional `description` and `status`, and an error response shape that every endpoint must use. This stage's `Task` also
   has `createdAt` and `updatedAt`, so that there were server-owned fields to omit from the input types: decide whether they stay on
   your `Task` and whether the API exposes them.
