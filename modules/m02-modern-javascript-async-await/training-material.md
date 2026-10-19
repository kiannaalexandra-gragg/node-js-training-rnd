# M02 — Modern JavaScript & Async/Await: Training Material

**Week 1 · Days 2-4 · 10 hours**

A "module" in this program means a training stage. "ES module" means a JavaScript module file.

## Builds on

M01 — Dev Environment, Git & GitHub. You need its repo (the public repo you created there), Node 24
pinned with `.nvmrc`, and the branch → pull request → merge habit. This stage ends with a pull
request in that repo.

## 1. What you'll build and why

You will write idiomatic modern JavaScript, learn how the event loop orders asynchronous code, and
build four small utilities that are easy to get subtly wrong. You finish with:

- an **Async toolkit** ES module with `sleep`, `retry`, `withTimeout` and `mapLimit`, each with its own
  passing `node:assert` self-checks,
- a **timing comparison** that fetches from JSONPlaceholder one request at a time and then in
  parallel, with your own measured numbers,
- a **callback-to-async refactor** of a piece of `node:fs` code,
- the ability to **predict and explain** the output order of a snippet that mixes `setTimeout`,
  promises and `await`.

**What goes in your repo:** only the four deliverables, in `modules/m02-modern-javascript-async-await/`.
The two learning scripts stay in a scratch folder outside it.

| File | Where | Built in |
| --- | --- | --- |
| `async-toolkit.mjs` | your repo | Stages 3 to 6 |
| `async-toolkit.check.mjs` | your repo | Stages 3 to 6 |
| `timing-comparison.mjs` | your repo | Stage 7 |
| `fs-refactor.mjs` | your repo | Stage 8 |
| `research.mjs` | scratch folder, not delivered | Stage 2 |
| `event-loop-order.mjs` | scratch folder, not delivered | Stage 9 |

Each utility hides a classic async mistake: `retry` punishes a forgotten `await`, `withTimeout` a
leaked timer, and `mapLimit` unlimited concurrency. Building them lets you meet those mistakes on
purpose, while nothing depends on the result.

### How to use this guide

The **guided path** (section 3) is what to do, run and verify. **Concepts** (section 4) explains what
you did: read "Concurrency" after Stage 6 and "The event loop" after Stage 9. **Common pitfalls**
(section 5) is a lookup table for when something misbehaves. The **definition of done** (section 6) is
the final acceptance check, nothing more.

**How the build stages work.** In Stages 3 to 8 you write the core logic yourself. Each stage has the
same shape: *what it must do*, then *your turn* (a scaffold with hints you can open one at a time), then
the *self-checks*, or in Stage 7 the script itself (they fail until your code works, and their messages
say what is wrong), then a
*reference implementation* to open only after you have tried, and *compare with your version* notes on
the design decisions. Give each hint and the reference a real attempt first.

## 2. Before you start

- Your M01 repo, on `main`, with `.nvmrc` containing `24`. Run `nvm use` in the repo root and confirm
  `node --version` prints a `v24.x` version. A fresh terminal can start on an older Node, so check
  every session.
- The toolkit uses APIs built into Node 24, the pinned version, such as `Promise.withResolvers()` and
  `AbortSignal.timeout()`. Do not run it on a much older Node.
- Internet access. The timing stage calls the free public API
  [JSONPlaceholder](https://jsonplaceholder.typicode.com), which needs no account or API key.
- **Pinned for this stage:** Node.js 24 LTS and plain modern JavaScript. No TypeScript (it starts in
  M04), no `npm install`, no test framework (that is M09). You use one ES module and Node's built-in
  `node:assert`.
- **File convention:** files end in `.mjs`, so Node treats them as ES modules without a
  `package.json`, and imports use the full filename (`./async-toolkit.mjs`). The program has not
  settled an import-extension convention for later TypeScript stages, so this is not its rule.
- **Exports:** the toolkit uses **named exports**, with no default export. A misspelt named import
  fails loudly, and the toolkit is a set of related utilities.

Commands are for macOS or Linux shells. Run them from the root of your build repo, `~/<repo-name>`,
where `<repo-name>` is the name you gave the repo in M01.

## 3. Guided path

### Stage 1 — Create the branch and the stage folder

**Goal:** a feature branch and a folder for this stage, like M01.

```bash
cd ~/<repo-name>
nvm use
node --version
git switch main
git switch -c feat/module-02/async-toolkit
mkdir -p modules/m02-modern-javascript-async-await
```

**Why:** every stage is delivered as a pull request, never a push to `main`. Name the folder like
the stage so the build repo and the content repo line up.

**Done when:**

- [ ] `node --version` prints a `v24.x` version.
- [ ] `git status -sb` shows you on `feat/module-02/async-toolkit`.
- [ ] The folder `modules/m02-modern-javascript-async-await/` exists.

### Stage 2 — Research by experiment

**Goal:** see how the async behaviour you are about to rely on really works, and know where to look
for the language basics.

This stage covers a lot of ground, and it does not all need the same depth. The main outcome of the
module is reliable asynchronous JavaScript, not memorising every recent language feature.

| Level | What to do |
| --- | --- |
| **Core** | Run every experiment and be able to explain each result. The utilities in Stages 3 to 6 depend on them. |
| **Reference** | Run the ones you are unsure of. Come back to the rest when you need them. |
| **Awareness** | Read the line and know it exists. No experiment. |

These experiments are for learning, not part of what you deliver, so keep them **outside your repo**,
in a scratch folder such as `~/m02-scratch/` (create it with `mkdir -p ~/m02-scratch`). Create
`~/m02-scratch/research.mjs`, start it with this
helper, and add one `await check('...', ...)` per row. Write the assertion first from what you expect,
then run the file. A passing assertion confirms your picture. A failing one shows what you had wrong,
which is the useful result.

```js
import assert from 'node:assert/strict';
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';

const run = promisify(execFile);
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms)); // your own, until Stage 3

async function check(name, fn) {
  await fn();
  console.log(`ok - ${name}`);
}
```

Run the file from your repo root with `node ~/m02-scratch/research.mjs`.

**Core experiments (do all of these)**

| Topic | Experiment | What you should be able to assert |
| --- | --- | --- |
| `const`, `let`, closures | Push `() => i` into an array inside a `for (var i...)` loop, then again with `for (let j...)` | `var` gives `[3, 3, 3]`, `let` gives `[0, 1, 2]`. `const` blocks reassignment, not `list.push` |
| `try/catch/finally`, custom errors, `cause` | A function that catches `JSON.parse` failures and throws `new ConfigError('bad config', { cause: error })` | `finally` runs on a throw and on a return. `error.cause` is the original `SyntaxError` |
| `await` and `try/catch` | `try { loose = failing(); } catch {}` versus `try { await failing(); } catch {}` | Only the awaited rejection is caught. Handle `loose` afterwards so it does not become unhandled |
| Forgotten `await` | `Boolean(isReady())` and `Boolean(await isReady())` for `async () => false` | The first is `true` (a promise is always truthy), the second `false` |
| `Promise.all`, `allSettled`, `race`, `any` | Mix a fast success, a slow success and a failing promise using `sleep` | `all` rejects at the first failure. `allSettled` gives a `status` for each. `race` follows whoever settles first, success or failure. `any` gives the first success, or `AggregateError` if all fail |
| `Promise.withResolvers` | `const { promise, resolve } = Promise.withResolvers()`, then `setTimeout(() => resolve('later'), 5)` | `await promise` gives `'later'`. You can settle a promise from outside |
| `AbortController`, `AbortSignal.timeout` | Abort a controller, then `fetch(url, { signal })`. Then `fetch(url, { signal: AbortSignal.timeout(1) })` | The first rejects with `name === 'AbortError'`, the second with `'TimeoutError'` |
| `fetch` and JSON | `fetch('https://jsonplaceholder.typicode.com/posts/99999')` | A 404 does **not** reject. `response.ok` is `false`, `response.status` is `404`, and `await response.json()` is `{}` |
| `forEach(async ...)` | Call `[1, 2, 3].forEach(async (n) => { await sleep(10); done.push(n); })` | `done` is still `[]` right after `forEach`. A `for...of` loop with `await` does wait |
| Unhandled rejection | `run(process.execPath, ['--eval', 'Promise.reject(new Error("nobody handled me"))'])` | The child exits with code `1` and the error text on `stderr` |
| Swallowed error | `await Promise.reject(new Error('lost')).catch(() => {})` | It resolves to `undefined`, so the caller sees success with no value |

**Reference experiments (run when unsure, revisit when needed)**

| Topic | Experiment | What you should be able to assert |
| --- | --- | --- |
| Arrow functions and `this` | An object whose method calls `[1, 2].forEach(() => { this.n++ })`, then the same with `function () { this.n++ }` | The arrow keeps the outer `this`. The plain function throws `TypeError` (`this` is `undefined` in a module) |
| Template literals | Build a string with `${expression}`, and one across two lines | The expression is evaluated and inserted, and the string keeps the line break |
| Destructuring, spread, rest, defaults | `const { a, b = 10, ...others } = obj`, `[first, , third, ...tail] = arr`, `{ ...x, y: 99 }`, `(...nums) => ...` | Defaults apply only for `undefined`. Spread copies one level, and later keys win |
| `?.`, `??`, `??=` | Compare `0 \|\| 5` with `0 ?? 5`, and `'' ?? 'anon'` | `\|\|` replaces any falsy value, `??` only `null` or `undefined`. `x ??= 7` assigns only when `x` is nullish |
| Truthy/falsy and equality | Check `[false, 0, -0, 0n, '', null, undefined, NaN]`, then `0 == ''`, `null == undefined`, `NaN === NaN`, `Object.is(NaN, NaN)`, `typeof null` | The eight are falsy. `[]`, `{}`, `'0'` are truthy. `==` coerces. `NaN` is not equal to itself. `typeof null` is `'object'`. Prefer `===` |
| Array methods | `map`, `filter`, `reduce`, `find`, `some`, `every`, `flatMap` on `[1, 2, 3, 4]` | Each returns what you expect. `flatMap` maps and flattens one level, so returning `[]` drops an item |
| `toSorted`, `toReversed`, `with` | `[10, 9, 1].toSorted()`, then with `(x, y) => x - y`, then `.sort()` on the original | `toSorted()` with no comparator sorts as strings, giving `[1, 10, 9]`. The new helpers leave the original alone. `sort` and `reverse` change it |
| `structuredClone` | Clone `{ when: new Date(0), tags: new Map(), nested: { n: 1 } }`, change the copy, then clone `{ fn() {} }` | A deep copy, `Date` and `Map` preserved, the original untouched. A function throws `DataCloneError` |
| `Object.groupBy`, `Map.groupBy` | Group `['apple', 'avocado', 'banana', ...]` by first letter, then `Map.groupBy` by length | `Object.groupBy` returns an object with a `null` prototype. `Map.groupBy` allows any key type |
| `Map` and `Set` | A `Map` with keys `1` and `'1'`. `new Set([1, 1, 2])` | `Map` keeps key types and order, so `1` and `'1'` are two keys. `Set` removes duplicates |
| Classes, `#private`, getters, setters | A `Temperature` class with `#celsius`, a `fahrenheit` getter and setter | `Object.keys(t)` is `[]`, and `t.celsius` is `undefined` |
| Async iteration | An `async function*` that `await`s `sleep(1)` and yields `3, 2, 1`. Consume it with `for await...of` | It gives `[3, 2, 1]` |
| Named versus default exports | The one-liner above, after Stage 6 | It lists only the names you exported (`RetryError`, `TimeoutError`, `mapLimit`, `retry`, `sleep`, `withTimeout`), and `t.default` is `undefined` |

The "named versus default exports" row needs the finished toolkit, so do it after Stage 6, as a
one-liner from your repo root:

```bash
node --input-type=module -e "import * as t from './modules/m02-modern-javascript-async-await/async-toolkit.mjs'; console.log(Object.keys(t).sort(), t.default)"
```

**Awareness only (no experiment)**

- `Promise.try(fn)` calls `fn` and turns a synchronous throw into a rejection.
- `Array.fromAsync(asyncIterable)` collects an async iterable into an array.
- The newer `Set` methods are `union`, `intersection`, `difference`, `symmetricDifference` and `isSubsetOf`.
- Temporal is the planned replacement for `Date`. You do not use it here. `typeof Temporal` is
  `undefined` on Node 24.

**Done when:**

- [ ] `node ~/m02-scratch/research.mjs` prints an `ok - ...` line for every Core check you added and
      exits without an error. It is a scratch file, so nothing from it goes into your repo.
- [ ] After Stage 6, the exports one-liner lists the four utilities and the two error classes, and
      prints `undefined` for `t.default`.
- [ ] You can explain, in your own words, why a `forEach(async ...)` loop finishes before its
      callbacks do.

### Stage 3 — Build `sleep`

**Goal:** a promise that resolves after a delay, with a self-check that proves it.

**What it must do**

- `sleep(ms)` returns a promise that resolves with `undefined` after at least `ms` milliseconds.
- If `ms` is not a finite, non-negative number, it throws a `RangeError` straight away. It does not
  return a rejected promise.

**Your turn.** Create `modules/m02-modern-javascript-async-await/async-toolkit.mjs` and fill in the
function:

```js
export function sleep(ms) {
  // TODO: reject bad input, then return a promise that resolves after ms milliseconds
}
```

<details>
<summary>Hint 1</summary>

`setTimeout` takes a callback. A promise is how you turn "call this later" into something you can `await`: `new Promise((resolve) => ...)`.

</details>

<details>
<summary>Hint 2</summary>

`Number.isFinite(ms)` is `false` for `NaN`, `Infinity` and strings.

</details>

**Self-checks.** Create `modules/m02-modern-javascript-async-await/async-toolkit.check.mjs`. It will hold
every self-check for the toolkit. Paste this first and run it: the checks fail until your function works,
and the failure messages point at what is wrong.

```js
import assert from 'node:assert/strict';
import { sleep } from './async-toolkit.mjs';

async function check(name, fn) {
  await fn();
  console.log(`ok - ${name}`);
}

await check('sleep waits at least the requested time', async () => {
  const start = performance.now();
  await sleep(50);
  const elapsed = performance.now() - start;
  assert.ok(elapsed >= 45, `expected >= 45ms, got ${elapsed.toFixed(1)}ms`);
});

await check('sleep resolves with undefined', async () => {
  assert.equal(await sleep(1), undefined);
});

await check('sleep rejects bad input synchronously', () => {
  assert.throws(() => sleep(-1), RangeError);
  assert.throws(() => sleep('soon'), RangeError);
});
```

Run it with `node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs`. The `>= 45` in the
first check is deliberate: timers are only approximately accurate, so a check that demands exactly 50
would fail at random.

<details>
<summary>Reference implementation (open it after you have tried)</summary>

```js
export function sleep(ms) {
  if (!Number.isFinite(ms) || ms < 0) {
    throw new RangeError(`sleep: ms must be a non-negative number, got ${ms}`);
  }
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

</details>

**Done when:**

- [ ] `node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs` prints `ok -` for the
      three `sleep` checks and exits without an error.
- [ ] `async-toolkit.mjs` has a named export `sleep` and no default export.

### Stage 4 — Build `retry`

**Goal:** call an async function until it succeeds or the tries run out, and report why it gave up.

**What it must do**

- `retry(fn, { retries = 3, delayMs = 100 } = {})` returns a promise for `fn`'s result. `fn` may
  return a value or a promise.
- `fn` is called with the attempt number, starting at `0`.
- `retries` is the number of extra tries after the first, so `retries: 2` means at most 3 calls.
- It waits `delayMs` between tries, but not after the last one.
- After the final failure it rejects with a `RetryError` whose `attempts` is the number of calls made
  and whose `cause` is the last error from `fn`.
- A bad `retries` (not a non-negative integer) or bad `delayMs` (not a finite, non-negative number)
  rejects with a `RangeError` before `fn` is ever called. `retry` is `async`, so it returns a rejected
  promise, it does not throw.

**Your turn.** Add to `async-toolkit.mjs`. The error class is given. Write the function:

```js
export class RetryError extends Error {
  constructor(attempts, options) {
    super(`Gave up after ${attempts} attempt${attempts === 1 ? '' : 's'}`, options);
    this.name = 'RetryError';
    this.attempts = attempts;
  }
}

export async function retry(fn, { retries = 3, delayMs = 100 } = {}) {
  // TODO 1: validate retries and delayMs before calling fn
  // TODO 2: loop over attempts 0..retries, calling fn(attempt) and returning its result on success
  // TODO 3: on failure, remember the error and wait delayMs (but not after the last attempt)
  // TODO 4: after the last failure, throw new RetryError(retries + 1, { cause: lastError })
}
```

<details>
<summary>Hint 1</summary>

Each attempt, and the delay after it, must finish before the next attempt starts. Is an `await` inside a loop a mistake here, or the point?

</details>

<details>
<summary>Hint 2</summary>

You need to catch `fn`'s rejection to retry. Run the check "retry retries an async rejection until it succeeds". If it crashes with a raw `Error: fail 0`, look at the line where you return `fn`'s result.

</details>

<details>
<summary>Hint 3</summary>

Use the `sleep` you built in Stage 3 for the delay.

</details>

**Self-checks.** Change the import at the top of `async-toolkit.check.mjs` to
`import { sleep, retry, RetryError } from './async-toolkit.mjs';`, add these, and run it. They fail
until your `retry` works.

```js
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
  const attempts = [];
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
      assert.equal(error.cause.message, 'boom 3');
      return true;
    },
  );
  assert.equal(calls, 3);
});

await check('retry waits delayMs between tries but not after the last one', async () => {
  const start = performance.now();
  await assert.rejects(retry(async () => { throw new Error('x'); }, { retries: 2, delayMs: 40 }));
  const elapsed = performance.now() - start;
  // 3 attempts -> 2 waits of 40ms = 80ms. A wait after the last attempt would make it ~120ms.
  assert.ok(elapsed >= 75, `expected >= 75ms, got ${elapsed.toFixed(1)}ms`);
  assert.ok(elapsed < 115, `expected < 115ms, got ${elapsed.toFixed(1)}ms`);
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
  await assert.rejects(retry(async () => { calls++; }, { delayMs: 'soon' }), RangeError);
  assert.equal(calls, 0); // fn is never called with invalid options
});
```

<details>
<summary>Reference implementation (open it after you have tried)</summary>

```js
export class RetryError extends Error {
  constructor(attempts, options) {
    super(`Gave up after ${attempts} attempt${attempts === 1 ? '' : 's'}`, options);
    this.name = 'RetryError';
    this.attempts = attempts;
  }
}

export async function retry(fn, { retries = 3, delayMs = 100 } = {}) {
  if (!Number.isInteger(retries) || retries < 0) {
    throw new RangeError(`retry: retries must be a non-negative integer, got ${retries}`);
  }
  if (!Number.isFinite(delayMs) || delayMs < 0) {
    throw new RangeError(`retry: delayMs must be a non-negative number, got ${delayMs}`);
  }
  let lastError;
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await fn(attempt);
    } catch (error) {
      lastError = error;
      if (attempt < retries) await sleep(delayMs);
    }
  }
  throw new RetryError(retries + 1, { cause: lastError });
}
```

</details>

**Compare with your version**

- `retries` counts extra tries after the first, so `retries: 2` means at most 3 calls. Both options
  are validated before the first call, so a bad `delayMs` is caught even when `fn` succeeds first time.
- The `await` inside the loop is deliberate: each attempt, and the delay after it, must finish before
  the next one starts.
- `return await fn(attempt)` is needed **here** because the rejection has to be caught by this `catch`,
  so the loop can retry. Inside a `try`, `return promise` hands the promise to the caller and its
  rejection skips the `catch`. Outside a `try`, `return promise` and `return await promise` behave the
  same for the caller, so do not add `await` to every `return` by reflex. To see the difference, copy
  the folder to a temporary directory (copy `.nvmrc` too), change it to `return fn(attempt)`, and run
  the checks: the run crashes with the raw `Error: fail 0` and nothing is retried.
- `RetryError` keeps the last error as `cause`, so the caller learns both how many tries were made and
  why the last one failed.

**Done when:**

- [ ] `node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs` prints `ok -` for 10
      checks (3 for `sleep`, 7 for `retry`) and exits without an error.
- [ ] You can explain why `retry` needs `return await` here, and why it does not sleep after the last
      attempt.

### Stage 5 — Build `withTimeout`

**Goal:** stop waiting for a promise after `ms` milliseconds, without leaking a timer or causing an
unhandled rejection.

**What it must do**

- `withTimeout(promise, ms)` settles like `promise` if it settles within `ms` milliseconds. Otherwise it
  rejects with a `TimeoutError` (`name` `'TimeoutError'`, an `ms` property and a `cause`).
- If `promise` rejects first, that exact error comes through unchanged.
- It must not keep the Node process alive after the promise settles, and a late rejection from the slow
  promise must not crash the process.
- A bad `ms` (not a finite, non-negative number) throws a `RangeError` straight away.
- **Limit:** it stops the *caller* waiting. It does not stop the underlying operation, because a promise
  cannot be cancelled from outside. It relies on APIs built into Node 24, such as `Promise.withResolvers()`
  and `AbortSignal.timeout()`.

**Your turn.** Add to `async-toolkit.mjs`. The error class is given. Write the function:

```js
export class TimeoutError extends Error {
  constructor(ms, options) {
    super(`Timed out after ${ms}ms`, options);
    this.name = 'TimeoutError';
    this.ms = ms;
  }
}

export function withTimeout(promise, ms) {
  // TODO 1: validate ms
  // TODO 2: create a timer that rejects with new TimeoutError(ms, { cause: ... }) after ms
  // TODO 3: race the promise against that timer, and remove the timer's listener when the race is over
}
```

<details>
<summary>Hint 1</summary>

`Promise.race([a, b])` settles with whichever of `a` and `b` settles first.

</details>

<details>
<summary>Hint 2</summary>

`AbortSignal.timeout(ms)` gives you a signal that aborts after `ms`. Its timer does not keep the process alive. Verify that: `node -e "const s = AbortSignal.timeout(10000); s.addEventListener('abort', () => {})"` exits at once, but `node -e "setTimeout(() => {}, 3000)"` waits 3 seconds. The child-process check below fails if you use a plain `setTimeout` and never clear it.

</details>

<details>
<summary>Hint 3</summary>

`Promise.withResolvers()` returns `{ promise, resolve, reject }`, a promise you can reject from outside, such as from an `abort` listener. Remove the listener in `.finally`.

</details>

**Self-checks.** Update the top of `async-toolkit.check.mjs`, add these, and run it:

```js
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
import { sleep, retry, RetryError, withTimeout, TimeoutError } from './async-toolkit.mjs';

const run = promisify(execFile);
```

```js
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
  const late = sleep(80).then(() => { throw new Error('too late'); });
  await assert.rejects(withTimeout(late, 20), TimeoutError);
  await sleep(120);
});

await check('withTimeout does not keep the process alive after the promise settles', async () => {
  const code = `
    import { withTimeout, sleep } from ${JSON.stringify(new URL('./async-toolkit.mjs', import.meta.url).href)};
    await withTimeout(sleep(5), 10_000);
  `;
  const start = performance.now();
  await run(process.execPath, ['--input-type=module', '--eval', code]);
  const elapsed = performance.now() - start;
  assert.ok(elapsed < 3000, `child took ${elapsed.toFixed(0)}ms, a lingering 10s timer would be ~10000ms`);
});

await check('withTimeout rejects a bad ms synchronously', () => {
  assert.throws(() => withTimeout(sleep(1), -5), RangeError);
});
```

<details>
<summary>Reference implementation (open it after you have tried)</summary>

```js
export class TimeoutError extends Error {
  constructor(ms, options) {
    super(`Timed out after ${ms}ms`, options);
    this.name = 'TimeoutError';
    this.ms = ms;
  }
}

export function withTimeout(promise, ms) {
  if (!Number.isFinite(ms) || ms < 0) {
    throw new RangeError(`withTimeout: ms must be a non-negative number, got ${ms}`);
  }
  const signal = AbortSignal.timeout(ms);
  const { promise: timeout, reject } = Promise.withResolvers();
  const onAbort = () => reject(new TimeoutError(ms, { cause: signal.reason }));
  signal.addEventListener('abort', onAbort, { once: true });
  return Promise.race([promise, timeout]).finally(() => {
    signal.removeEventListener('abort', onAbort);
  });
}
```

</details>

**Compare with your version**

- `AbortSignal.timeout`'s timer does not keep the process alive. A plain `setTimeout` does, until it fires
  or you clear it. The child-process check catches a timer that is never cleared.
- `.finally` removes the listener. The slow promise may still reject after the timeout wins, and
  `Promise.race` has already attached a handler to it, so that late rejection is not unhandled.

**Done when:**

- [ ] `node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs` prints `ok -` for 16
      checks and exits without an error.
- [ ] The "does not keep the process alive" check passes, and you can say what would make it fail.
- [ ] You can explain what `withTimeout` stops and what it does not.

### Stage 6 — Build `mapLimit`

**Goal:** run an async function over a list with at most `limit` calls at once, and return the results
in input order.

Two easy answers are wrong. `Promise.all(items.map(fn))` starts every call at once, so there is no
limit. Splitting the list into chunks of `limit` and awaiting each chunk with `Promise.all` makes each
chunk wait for its slowest call while the other slots sit idle. A worker pool avoids both.

The worker-pool idea: start up to `limit` workers. Each worker repeatedly claims the next unclaimed index,
awaits `fn` for it, and stores the result at that index. When a worker finishes one item it claims
another, so no slot waits on a slower neighbour. Because results are stored by index, the output order
matches the input order however the calls finish.

Trace `mapLimit([30, 20, 10, 0], 2, ...)` where each call sleeps for its number of milliseconds. Two
workers start and claim indexes 0 and 1. The worker on index 1 (20 ms) finishes first and claims
index 2. Index 0 (30 ms) finishes later, but its result goes into `results[0]`, so the output order
still matches the input.

**What it must do**

- `mapLimit(items, limit, fn)` returns a promise for an array of `fn`'s results, in the order of `items`.
- At most `limit` calls of `fn` are in flight at once. `items` can be any iterable. `fn` receives
  `(item, index)` and may be sync or async.
- An empty input gives `[]`. A `limit` that is not an integer of at least 1 rejects with a `RangeError`.
- If one call rejects, `mapLimit` rejects with that error and **starts no new items**. Calls that are
  already running are **not cancelled**: they run to completion and their results are discarded.

**Your turn.** Add to `async-toolkit.mjs`. The setup is given. Write the worker loop, including the
`failed` flag, and start the workers:

```js
export async function mapLimit(items, limit, fn) {
  if (!Number.isInteger(limit) || limit < 1) {
    throw new RangeError(`mapLimit: limit must be an integer >= 1, got ${limit}`);
  }
  const list = [...items];
  const results = new Array(list.length);
  let nextIndex = 0;
  let failed = false;

  async function worker() {
    // TODO: while there is work left and nothing has failed:
    //   claim the next index, await fn(list[index], index), store the result at that index.
    //   If fn rejects, make sure the other workers stop claiming new items, then rethrow.
  }

  // TODO: start Math.min(limit, list.length) workers and wait for all of them
  return results;
}
```

<details>
<summary>Hint 1</summary>

Why `results[index] = value` and not `results.push(value)`? Think about which call finishes first.

</details>

<details>
<summary>Hint 2</summary>

Claim an index with `const index = nextIndex++`, with no `await` between reading and bumping `nextIndex`. JavaScript runs one piece of synchronous code at a time, so two workers can never claim the same item.

</details>

<details>
<summary>Hint 3</summary>

`Promise.all(workers)` rejects as soon as one worker rejects, but it does not stop the other workers. What must the surviving workers check before claiming their next item?

</details>

**Self-checks.** Update the import in `async-toolkit.check.mjs` to include `mapLimit`, add these, and run it.
The last-but-one check is the one that exposes a missing `failed` flag.

```js
await check('mapLimit keeps results in input order even when calls finish out of order', async () => {
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
  const order = [];
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
  const started = [];
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

<details>
<summary>Reference implementation (open it after you have tried)</summary>

```js
export async function mapLimit(items, limit, fn) {
  if (!Number.isInteger(limit) || limit < 1) {
    throw new RangeError(`mapLimit: limit must be an integer >= 1, got ${limit}`);
  }
  const list = [...items];
  const results = new Array(list.length);
  let nextIndex = 0;
  let failed = false;

  async function worker() {
    while (!failed && nextIndex < list.length) {
      const index = nextIndex++;
      try {
        results[index] = await fn(list[index], index);
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

**Compare with your version**

- Without the `failed` flag, the surviving worker keeps starting items after the caller has already seen
  the error. Calls already running are not cancelled either way.
- To see why the flag matters, remove `!failed &&` from the loop in a temporary copy (with `.nvmrc`).
  The last-but-one check then fails because `started` contains `[0, 1, 2, 3, 4, 5]`.

**Done when:**

- [ ] `node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs` prints `ok -` for 23
      checks and exits without an error.
- [ ] The peak-concurrency check proves `peak` equals `limit`, not only that it stays below it.
- [ ] You can explain what happens to in-flight calls when one call fails, and why `mapLimit` needs
      the `failed` flag even though `Promise.all` already rejects.

### Stage 7 — Compare sequential and parallel fetching

**Goal:** measure the difference between awaiting each request in turn and starting them all at once.

**What it must do**

- Fetch posts `1` to `10` from JSONPlaceholder two ways, and time each way over several rounds.
- `fetchSequential` awaits each request before starting the next. `fetchParallel` starts all ten at
  once. Both return the posts in order.
- The script prints a table with one sequential and one parallel time per round, and a one-line summary
  of which was faster.

**Your turn.** Create `modules/m02-modern-javascript-async-await/timing-comparison.mjs`. Everything is
given except the two strategies. Write `fetchSequential` and `fetchParallel`:

```js
import assert from 'node:assert/strict';

const BASE_URL = 'https://jsonplaceholder.typicode.com';
const IDS = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
const ROUNDS = Number(process.argv[2] ?? 3);

if (!Number.isInteger(ROUNDS) || ROUNDS < 1) {
  throw new Error('Rounds must be a positive integer.');
}

async function fetchPost(id) {
  const response = await fetch(`${BASE_URL}/posts/${id}`, { signal: AbortSignal.timeout(10_000) });
  if (!response.ok) throw new Error(`GET /posts/${id} failed with ${response.status}`);
  return response.json();
}

async function fetchSequential() {
  // TODO: fetch posts 1..10 one at a time, each fetch finishing before the next starts.
  // Return the posts in order.
  throw new Error('fetchSequential is not implemented yet');
}

async function fetchParallel() {
  // TODO: start all ten fetches at once and wait for all of them. Return the posts in order.
  throw new Error('fetchParallel is not implemented yet');
}

async function timed(label, fn) {
  const start = performance.now();
  const posts = await fn();
  const ms = performance.now() - start;
  assert.deepEqual(posts.map((post) => post.id), IDS, `${label} returned the wrong posts`);
  return ms;
}

console.log(`Fetching ${IDS.length} posts from ${BASE_URL}, ${ROUNDS} rounds`);
console.log('round  sequential(ms)  parallel(ms)');
const sequentialTimes = [];
const parallelTimes = [];
for (let round = 1; round <= ROUNDS; round++) {
  const sequential = await timed('sequential', fetchSequential);
  const parallel = await timed('parallel', fetchParallel);
  sequentialTimes.push(sequential);
  parallelTimes.push(parallel);
  console.log(`${String(round).padStart(5)}  ${sequential.toFixed(0).padStart(14)}  ${parallel.toFixed(0).padStart(12)}`);
}

const total = (times) => times.reduce((sum, ms) => sum + ms, 0);
const sequentialTotal = total(sequentialTimes);
const parallelTotal = total(parallelTimes);
console.log(`total  ${sequentialTotal.toFixed(0).padStart(14)}  ${parallelTotal.toFixed(0).padStart(12)}`);
if (parallelTotal < sequentialTotal) {
  console.log(`Parallel was ${(sequentialTotal / parallelTotal).toFixed(1)}x faster overall.`);
} else {
  console.log(
    'Parallel was not faster in these measurements. Network variability may affect the result; try again.',
  );
}
```

<details>
<summary>Hint 1</summary>

Sequential is a loop with `await` inside it. Here the `await` in the loop is the point: it is exactly what makes this version slow.

</details>

<details>
<summary>Hint 2</summary>

`IDS.map(...)` starts every fetch straight away and gives you an array of promises. What turns that into one promise for all the results?

</details>

Run it with `node modules/m02-modern-javascript-async-await/timing-comparison.mjs 5` (the number is how
many rounds).

<details>
<summary>Reference implementation of the two functions (open it after you have tried)</summary>

```js
async function fetchSequential() {
  const posts = [];
  for (const id of IDS) {
    posts.push(await fetchPost(id));
  }
  return posts;
}

async function fetchParallel() {
  return Promise.all(IDS.map((id) => fetchPost(id)));
}
```

</details>

**Compare with your version**

- `fetch` does not reject on a 404 or 500, so the code checks `response.ok` and throws. The ids are
  asserted, so a wrong or partial result cannot be timed as a win.
- `ROUNDS` is validated, so `0`, `-1` or `abc` fail with a clear message.
- The script reports which strategy was faster and does not fail when parallel loses, because network
  conditions vary from run to run. The stage still needs a run where parallel wins (see Done when).
- The rounds are awaited one after another so they do not overlap and skew each other.

**Reading your results:** sequential time is the sum of ten waits, and parallel time is about the
slowest single wait plus overhead. Your numbers will differ from anyone else's, and the first round
may be slower than the rest.

**Done when:**

- [ ] The script exits without an error and prints a table with one sequential and one parallel time
      per round.
- [ ] You have saved the table from a run whose summary says parallel was faster. More rounds can help
      when network variability hides the difference, but they cannot guarantee a win. If parallel never
      wins, look for a bug (for example an `await` inside the `map` callback) before blaming the network.
- [ ] You can explain the measurements: why sequential time is a sum of waits and parallel time is
      about the longest one.

### Stage 8 — Refactor callback-style `node:fs` code to `async`/`await`

**Goal:** the same behaviour as a callback-style function, written as a flat `async` function, with
checks that prove the two match.

**What it must do**

- `stampFile(inputPath, outputPath)` does what the callback version does: read a JSON file, add
  `stamped: true`, write it to a new file, read that back, and return the parsed result.
- File errors and parse errors reach the caller as a rejection of the returned promise.

**The code to refactor.** Create `modules/m02-modern-javascript-async-await/fs-refactor.mjs` as one file in
three parts, so the before, the after and the proof sit together. Begin with the imports:

```js
import assert from 'node:assert/strict';
import fs from 'node:fs';
import fsPromises from 'node:fs/promises';
import os from 'node:os';
import path from 'node:path';
import { promisify } from 'node:util';
```

**Part 1 — Before (callbacks).** This is given. It reads a JSON file, adds `stamped: true`, writes a new
file and reads it back:

```js
// ---- Part 1: Before (callbacks) -------------------------------------------------------------
// Read a JSON file, add `stamped: true`, write it to a new file, read it back.
// Each step nests one level deeper and repeats its own `if (error)` check.
function stampFileCallback(inputPath, outputPath, callback) {
  fs.readFile(inputPath, 'utf8', (readError, text) => {
    if (readError) return callback(readError);
    let data;
    try {
      data = JSON.parse(text);
    } catch (parseError) {
      return callback(parseError);
    }
    data.stamped = true;
    fs.writeFile(outputPath, JSON.stringify(data, null, 2), (writeError) => {
      if (writeError) return callback(writeError);
      fs.readFile(outputPath, 'utf8', (verifyError, written) => {
        if (verifyError) return callback(verifyError);
        // Parse inside the try, but call the callback outside it, so a throw from the
        // caller's own callback is never mistaken for a parse error and called back twice.
        let result;
        try {
          result = JSON.parse(written);
        } catch (parseError) {
          return callback(parseError);
        }
        callback(null, result);
      });
    });
  });
}
```

**Part 2 — After (promises and `async`/`await`).** This is your turn. Write the same steps with
`node:fs/promises`:

```js
// ---- Part 2: After (promises and async/await) -----------------------------------------------
// Same steps in a straight line. File errors and JSON.parse errors share one rejection path.
async function stampFile(inputPath, outputPath) {
  // TODO: do the same steps as Part 1 with node:fs/promises and await:
  //   read the input, parse it, set stamped = true, write the output, read it back and parse it.
  // Return the parsed result. Do not use callbacks or new Promise.
}
```

<details>
<summary>Hint 1</summary>

Each `fsPromises.readFile` and `fsPromises.writeFile` returns a promise. `await` each one in turn, in a straight line.

</details>

<details>
<summary>Hint 2</summary>

You do not need a `try/catch` for the caller's sake: a throw inside an `async` function becomes a rejection.

</details>

**Part 3 — Self-checks.** This is given. Both versions must give the same result and fail the same way.
It wraps the callback version with `promisify` so the checks can `await` it, and runs everything in a
temporary directory that is always removed:

```js
// ---- Part 3: Self-checks --------------------------------------------------------------------
// Both versions must give the same result and fail the same way. The callback version is wrapped
// with promisify so the checks can await it.
const stampFileViaCallback = promisify(stampFileCallback);

const dir = await fsPromises.mkdtemp(path.join(os.tmpdir(), 'm02-fs-refactor-'));
try {
  const inputPath = path.join(dir, 'input.json');
  await fsPromises.writeFile(inputPath, JSON.stringify({ name: 'toolkit', stage: 2 }));

  const expected = { name: 'toolkit', stage: 2, stamped: true };
  assert.deepEqual(await stampFileViaCallback(inputPath, path.join(dir, 'out-callback.json')), expected);
  assert.deepEqual(await stampFile(inputPath, path.join(dir, 'out-await.json')), expected);
  console.log('ok - callback and async/await versions produce the same result');

  const missing = path.join(dir, 'nope.json');
  await assert.rejects(stampFileViaCallback(missing, path.join(dir, 'x.json')), { code: 'ENOENT' });
  await assert.rejects(stampFile(missing, path.join(dir, 'y.json')), { code: 'ENOENT' });
  console.log('ok - both versions fail with ENOENT for a missing file');

  const malformed = path.join(dir, 'malformed.json');
  await fsPromises.writeFile(malformed, '{ not json');
  await assert.rejects(stampFileViaCallback(malformed, path.join(dir, 'x.json')), { name: 'SyntaxError' });
  await assert.rejects(stampFile(malformed, path.join(dir, 'y.json')), { name: 'SyntaxError' });
  console.log('ok - both versions fail with SyntaxError for malformed JSON');
} finally {
  await fsPromises.rm(dir, { recursive: true, force: true });
}
```

Run `node modules/m02-modern-javascript-async-await/fs-refactor.mjs`. It fails until Part 2 works.

<details>
<summary>Reference implementation of Part 2 (open it after you have tried)</summary>

```js
// ---- Part 2: After (promises and async/await) -----------------------------------------------
// Same steps in a straight line. File errors and JSON.parse errors share one rejection path.
async function stampFile(inputPath, outputPath) {
  const text = await fsPromises.readFile(inputPath, 'utf8');
  const data = JSON.parse(text);
  data.stamped = true;
  await fsPromises.writeFile(outputPath, JSON.stringify(data, null, 2));
  return JSON.parse(await fsPromises.readFile(outputPath, 'utf8'));
}
```

</details>

**Compare with your version**

- A `JSON.parse` throw inside a callback never reaches the caller, so every `JSON.parse` needs its own
  `try/catch`, including the final parse of the file you just wrote. Parse inside the `try` but call
  the callback outside it, so a throw from the caller's own callback is not mistaken for a parse error
  and called back a second time.
- In the `async` version, file errors and parse errors arrive as one rejection.
- `promisify` works on the callback version because its callback is last and takes `(error, value)`.
- The `finally` removes the temporary folder even if an assertion fails.

**Done when:**

- [ ] `node modules/m02-modern-javascript-async-await/fs-refactor.mjs` prints all three `ok -` lines (same
      result, `ENOENT`, `SyntaxError`) and exits without an error.
- [ ] The file has three labelled parts: the callback version, the `async` version and the self-checks.
- [ ] You can explain what the refactor removed (the nesting and the repeated `if (error)` checks) and
      what stays the same (the order of the steps and the error codes).

### Stage 9 — Predict the output order

**Goal:** be able to say, before running it, what a mix of `setTimeout`, promises and `await` prints.

This is for learning, not part of what you deliver, so create it in your scratch folder, outside your
repo, as `~/m02-scratch/event-loop-order.mjs`, with this snippet:

```js
console.log('1 sync start');

setTimeout(() => console.log('2 setTimeout 0'), 0);

Promise.resolve().then(() => {
  console.log('3 promise then');
  setTimeout(() => console.log('8 setTimeout 0 scheduled from a microtask'), 0);
});

(async () => {
  console.log('4 async fn: sync part');
  await null;
  console.log('5 after first await');
  await null;
  console.log('6 after second await');
})();

console.log('7 sync end');
```

**Before you run it**, write the eight labels in the order you expect, as a comment at the top of the
file, with one sentence of reasoning. Then run `node ~/m02-scratch/event-loop-order.mjs` from your repo
root and compare. The order should be: `1, 4, 7, 3, 5, 6, 2, 8`.

**Why:** writing the prediction first shows whether your model of the event loop is right, instead of
fitting an explanation to the output. If your prediction differs, read "The event loop and the two
queues" in Concepts, then move the `await` or add another timer and predict again.

**Done when:**

- [ ] The file has your written prediction above the snippet, and the actual output order is
      `1, 4, 7, 3, 5, 6, 2, 8`.
- [ ] You can explain, out loud and without notes, why `2 setTimeout 0` prints after `6 after second
      await` even though its delay is 0.
- [ ] You can explain why `8` prints after `2`.

### Stage 10 — Commit, push, open the pull request and merge it

**Goal:** deliver the stage the way M01 taught: a branch, Conventional Commits and a pull request, with
nothing pushed straight to `main`.

Run every command from your repo root, with Node 24 active (`nvm use`).

#### Step 1 — Re-run the deliverables

```bash
node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs
node modules/m02-modern-javascript-async-await/fs-refactor.mjs
node modules/m02-modern-javascript-async-await/timing-comparison.mjs 3
git status -sb
```

**Why:** the pull request's test plan should list results you have just seen. `git status -sb` should
show the stage branch and the stage folder as untracked, with nothing you did not mean to add.

#### Step 2 — Commit in one group per deliverable

Stage files **by name**, not with `git add .`, so you commit exactly what you intend:

```bash
git add modules/m02-modern-javascript-async-await/async-toolkit.mjs modules/m02-modern-javascript-async-await/async-toolkit.check.mjs
git commit -m "feat(m02): add async toolkit with sleep, retry, withTimeout and mapLimit"
git add modules/m02-modern-javascript-async-await/timing-comparison.mjs
git commit -m "feat(m02): add sequential vs parallel timing comparison"
git add modules/m02-modern-javascript-async-await/fs-refactor.mjs
git commit -m "feat(m02): refactor callback node:fs code to async/await"
```

**Why this grouping:** each commit is one deliverable that works on its own. The toolkit and its
self-checks share a commit, because the checks are the proof that it works.

#### Step 3 — Push the branch

```bash
git push -u origin feat/module-02/async-toolkit
```

`-u` links your local branch to the new remote branch, so later pushes and pulls need no extra
arguments.

#### Step 4 — Open the pull request

```bash
gh pr create --repo <user>/<repo-name> --base main --head feat/module-02/async-toolkit \
  --title "feat(m02): add async toolkit, timing comparison and fs refactor" \
  --body "<your description>"
```

Replace `<user>` and `<repo-name>` with your GitHub username and repo name. Passing `--repo`, `--base`
and `--head` explicitly means the PR cannot land somewhere you did not intend. The description has
three sections:

- **Summary:** why the stage exists and what it adds, in one short paragraph.
- **Changes:** one bullet per notable change (each utility, the timing script, the refactor).
- **Test plan:** one checked item per command you ran in Step 1, with what it printed (for example
  "23 `ok -` lines and no error"), and an unchecked item for anything the reviewer should run. Never
  tick a box for something you did not run.

#### Step 5 — Merge it and look at the history

Merge the PR on GitHub with a **merge commit**, not a squash, so that your three commits stay visible
in the history. Then update your view of `main`:

```bash
git fetch origin
git log --oneline --graph origin/main -8
```

The graph should show a `Merge pull request #<number> from ...` commit on `main`, with your three
`feat(m02): ...` commits under it. GitHub writes the merge commit's message, so it is not a
Conventional Commit. That is expected. The commits inside the PR are.

**Done when:**

- [ ] `git log --oneline main..HEAD` on the stage branch shows three `feat(m02): ...` commits, one per
      deliverable, and `git status -sb` shows nothing left to commit.
- [ ] A pull request from `feat/module-02/async-toolkit` into `main` exists on your build repo, and its
      description has Summary, Changes and Test plan sections.
- [ ] The PR is merged, and `git log --oneline --graph origin/main -8` shows its merge commit with your
      three commits under it.

## 4. Concepts you need

### The event loop and the two queues

JavaScript runs one piece of code at a time on a **call stack**. When something asynchronous finishes
(a timer fires, a file is read, a promise settles), its callback does not run straight away. It waits
in a queue until the stack is empty. Two queues matter:

| Queue | What goes in it | When it is processed |
| --- | --- | --- |
| **Task queue** | timers (`setTimeout`), I/O callbacks | one task at a time, each after the microtask queue is empty |
| **Microtask queue** | promise callbacks (`then`, `catch`, `finally`) and the code after an `await` | the whole queue is drained after the current code finishes, before the next task |

The rule: after each task, the event loop drains the **entire** microtask queue before it takes the
next task. Microtasks cut in line, so `setTimeout(fn, 0)` ("as soon as possible") still waits behind
every promise callback that is already queued, and behind any new ones those callbacks add.

Two consequences are worth keeping as mental shortcuts:

- An `async` function runs **synchronously up to its first `await`**. Only the code after that
  `await` becomes a microtask.
- A timer scheduled from inside a microtask is a **new task**. It goes to the back of the task queue,
  behind timers that were already waiting.

A comparison that helps: a task is a customer walking up to the counter, and the microtask queue is
the same customer's follow-up questions. The clerk (the event loop) answers every follow-up before
calling the next customer. A long chain of microtasks can therefore starve timers, which is why
recursive promise chains need an exit.

### Concurrency: what `Promise.all` does, and does not, control

`Promise.all`, `allSettled`, `race` and `any` all take several promises and give back one. They differ
in what that one promise means:

| Combinator | Fulfils when | Rejects when | Use it when |
| --- | --- | --- | --- |
| `all` | every promise fulfils | the first one rejects (fail fast) | you need every result, and one failure makes the rest useless |
| `allSettled` | every promise has settled | never | you need to know what happened to each one |
| `race` | the first promise to settle fulfils | the first promise to settle rejects | you want whichever finishes first, success or failure (a timeout is a race) |
| `any` | the first promise to fulfil | every promise rejects (`AggregateError`) | you want the first success and can ignore failures |

None of them starts, limits or cancels work. They only decide what you are told about work that is
already running, which is why Stages 5 and 6 had to stop waiting and stop claiming work themselves.

Sequential, parallel and limited are three points on one scale. Sequential is a limit of 1. Parallel
is a limit of "all of them". `mapLimit(items, limit, fn)` lets you pick any point. Choose sequential
when a step needs the previous result or the order of side effects matters. Choose parallel for a
small number of independent requests. Choose a limit when the number of requests is large or the
server pushes back.

## 5. Common pitfalls

| Symptom | Cause | Fix |
| --- | --- | --- |
| `node --version` is not the version in `.nvmrc` | A fresh terminal uses your default Node, which can be older | Run `nvm use` in the repo root, and run the scripts from there. |
| A failure shows up as a raw error and nothing is retried | `return promise` inside a `try` skips the `catch` | Use `return await` where this `catch` must handle the rejection. It is not needed outside a `try`. |
| `if (isReady())` is always true | An `async` function returns a promise, and a promise is always truthy | `await` it first |
| A `forEach(async ...)` loop finishes before its callbacks | `forEach` ignores the promises its callback returns | `for...of` with `await` for in-order work, or `map` with `Promise.all` or `mapLimit` |
| Independent requests are slow | `await` inside a loop turns parallel waits into a sum | Start them together with `Promise.all` or `mapLimit` |
| A 404 page is parsed as data | `fetch` only rejects when the request itself fails | Check `response.ok` |
| A script lingers after it finishes | A pending plain `setTimeout` keeps Node alive | Use `AbortSignal.timeout`, or `clearTimeout` |
| The process exits with code 1 after losing a race | A late rejection with no handler is unhandled | `Promise.race` attaches handlers, as `withTimeout` relies on. Otherwise add a `.catch`. |
| A call "succeeds" with no value | `.catch(() => {})` swallowed the error | Handle the error or rethrow it |
| Work continues after `Promise.all` rejects, or after a timeout | They report a failure and do not cancel running work | Stop starting new work yourself (the `failed` flag). Cancel running work only through an `AbortSignal`. |
| A timing check fails at random | Timers are approximate | Allow a few milliseconds of slack |
| Timings vary between runs | Network conditions | Run several rounds. |
| Round 1 is slower than the rest | Not established. A guess such as connection setup is untested. | Treat round 1 with caution, and test a cause before you claim it. |

## 6. Check yourself against the definition of done

- [ ] All `node:assert` self-checks pass: `async-toolkit.check.mjs` prints 23 `ok -` lines and
      `fs-refactor.mjs` prints 3, each exiting without an error.
- [ ] The toolkit exports `sleep`, `retry(fn, { retries, delayMs })`, `withTimeout(promise, ms)` and
      `mapLimit(items, limit, fn)`, each with its own self-checks.
- [ ] You can predict and explain, out loud, the output order of a snippet mixing `setTimeout`,
      promises and `await`.
- [ ] You have saved timing results from a run where parallel beat sequential, and you can explain why.
- [ ] A callback-style `node:fs` function is refactored to promises/`async`-`await`, with both versions
      in your build repo.
- [ ] A pull request from your stage branch is merged into `main` on your build repo.

## 7. Next step

If every box in section 6 is ticked, this stage is done.

1. Keep the repo and carry the same workflow forward: a branch named `feat/module-NN/<topic>`,
   Conventional Commits and a pull request. Never push to `main` directly.
2. Move on to **M03 — Node.js Runtime & npm** (Week 1 · Days 4-5). Its CLI lab reuses the async
   patterns you built here.
