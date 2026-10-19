# M02 Write-Up — Modern JavaScript & Async/Await

> Written from what I captured as I built it. Written so a trainee starting this stage next cohort
> could follow my path on their own.

A "module" in this program means a training stage. In this write-up "ES module" means a JavaScript
module file.

## What I built

All code is in my build repo, [`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training),
in `modules/m02-modern-javascript-async-await/`, on the branch `feat/module-02/async-toolkit`.

- **The Async toolkit**, `async-toolkit.mjs`: one ES module with named exports `sleep`, `retry`, `withTimeout` and `mapLimit`, plus two small error classes, `RetryError` and `TimeoutError`.
- **Its self-checks**, `async-toolkit.check.mjs`: 23 `node:assert` checks, a few per utility. All 23 pass.
- **The timing comparison**, `timing-comparison.mjs`: fetches 10 posts from JSONPlaceholder one at a time and then all at once, and prints both timings. Parallel won in every round of both runs (numbers under Checkpoint evidence).
- **The callback-to-async refactor**, `fs-refactor.mjs`: one file in three parts. Part 1 is a callback-style `node:fs` function, Part 2 is its `async`/`await` version, and Part 3 is the self-checks that both give the same result and the same errors (`ENOENT` for a missing file, `SyntaxError` for malformed JSON).

These four files are the whole stage folder. The event-loop snippet and the research experiments were throwaway scripts that I ran from a scratch folder outside the repo (written `~/m02-scratch/` in this write-up). Their code and results are in this write-up, under steps 7 and 8 of "How to build it".

Run any of the four from the repo root, on Node 24:

```bash
nvm use
node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs
node modules/m02-modern-javascript-async-await/timing-comparison.mjs
node modules/m02-modern-javascript-async-await/fs-refactor.mjs
```

I built the toolkit one function at a time and ran the checks after each one, so each function was
passing before I started the next. The order was `sleep`, `retry`, `withTimeout`, `mapLimit`.

Environment: Node v24.22.0 (from `.nvmrc`), macOS, zsh.

## Why it's built this way (key decisions)

**`retry(fn, { retries, delayMs })`**

- `retries` means extra tries after the first, so `retries: 2` runs `fn` at most 3 times. I chose that because "retry twice" reads naturally as "try, then try again twice". The failing case `retries: 0` still calls `fn` once.
- It loops with `await` inside on purpose. Each attempt has to finish, and the delay has to pass, before the next attempt starts. This is the one place `await` in a loop is the correct design.
- It uses `return await fn(attempt)`, not `return fn(attempt)`. I tested the difference on a scratch copy. Without the `await`, the first rejection (`Error: fail 0`) skipped `retry`'s `catch` block and crashed the check run, so nothing was retried.
- After the last failure it throws a `RetryError` with the last error as `cause` and the number of `attempts`, so the caller learns both how hard it tried and why it failed.
- It does not sleep after the last attempt. A check proves it: two waits of 40 ms finish in about 80 ms, not 120 ms.

**`withTimeout(promise, ms)`**

- It races the promise against a timer. I made the timer with `AbortSignal.timeout(ms)` and `Promise.withResolvers()`, so I have a promise that I can reject from the abort listener.
- I used `AbortSignal.timeout` and not a plain `setTimeout` because its timer does not keep the process alive, while a plain `setTimeout` does until it fires or is cleared. I measured it: a `node -e` script with a pending `AbortSignal.timeout(10000)` and a listener exited after 0.02 s, and one with a pending `setTimeout(3000)` exited after 3.02 s. A check runs `withTimeout(sleep(5), 10_000)` in a child process and asserts it exits in under 3 s.
- It cannot cancel the work behind the promise. A promise cannot be cancelled from outside, so the timeout only stops the caller waiting. I wrote that limit into the doc comment. Only the code that started the work (for example `fetch` with a signal) can stop it.
- It removes its abort listener in `.finally`, and a late rejection of the slow promise does not become an unhandled rejection, because `Promise.race` has already attached a handler to it. A check covers that.

**`mapLimit(items, limit, fn)`**

- It is a worker pool. It starts `Math.min(limit, items.length)` workers, and each worker keeps claiming the next unclaimed index until none are left. That keeps at most `limit` calls in flight, and it writes each result into `results[index]`, so the output order matches the input order even when calls finish in a different order.
- There is no race on `nextIndex++`. Claiming an index has no `await` between reading and bumping it, and JavaScript runs one piece of synchronous code at a time, so two workers cannot claim the same item.
- A shared `failed` flag stops workers from claiming new items after one call rejects. I tested it on a scratch copy by removing `!failed &&` from the loop. The check then failed with `[0, 1, 2, 3, 4, 5]` started instead of `[0, 1, 2]`: `mapLimit` had already rejected, but the worker that had not failed kept running items 3, 4 and 5 in the background.

**Sequential versus parallel fetching**

- Sequential is a `for...of` loop with `await fetchPost(id)`. Its total time is the sum of ten waits. Parallel is `Promise.all(IDS.map((id) => fetchPost(id)))`. Its total is about the slowest single wait plus overhead.
- Parallel also changes behaviour: all 10 requests are in flight at once, and `Promise.all` rejects as soon as one fails. For 10 requests to a free API that is fine. For hundreds, I would use `mapLimit` so I do not open hundreds of connections at once.
- Sequential is still the right choice when a request needs the previous one's result, or when order of side effects matters, as in `retry`.

**Pitfalls I designed around**

- *Forgotten `await`*: `return await` inside `retry`'s `try`, shown above.
- *`await` in a loop*: used only where order matters (`retry`, and the timing rounds, which must not overlap or they would skew each other). Everything that can overlap uses `Promise.all` or `mapLimit`.
- *Unhandled rejections*: `withTimeout` is safe against a late rejection, and `mapLimit` never leaves a rejected worker unobserved, because `Promise.all` attaches a handler to every worker.
- *Swallowed errors*: `retry` keeps the last error as `cause`, and the timing script throws on a non-OK response, because `fetch` does not reject on a 404 (my probe showed `response.ok === false` and a `{}` body).

**Small decisions**

- *Named exports only.* The toolkit has no default export, because it is a bag of related utilities and a check asserts the export names. Named exports also fail loudly at import if a name is misspelt.
- *`.mjs` files and no `package.json`.* The repo has none, so I used the `.mjs` extension to make Node treat the files as ES modules, and imports use the explicit `./async-toolkit.mjs` specifier. `knowledge/rules/coding-standards.md` says ES modules via `"type": "module"` throughout, so this is a deliberate difference that the EM may want to decide on. This stage is plain JavaScript, so it does not settle the open P2 `.ts`-versus-`.js` question.
- *`fs-refactor.mjs` is one file in three parts, not two files.* The point of a refactor is to see the same behaviour before and after, so the callback version, the `async`/`await` version and the checks that prove they match sit together in the order Before, After, Self-checks. With two files, the checks would have to import both, and the "before" code would be a file nobody runs on its own.
- *Only deliverables live in the repo.* The brief asks for the toolkit with its checks, the timing comparison and the refactor. The event-loop snippet and the research experiments answer "be ready to explain" and "research before you start", so they are not files in the repo.
- *Checks in a sibling file.* The module stays free of test code. Timer checks allow a few milliseconds of slack (`sleep(50)` must take at least 45 ms), because timers are only approximately accurate.

## How to build it (teach it to the next trainee)

I chose `mapLimit` as my worked example, because it is the one where the easy version is subtly wrong.
The other steps follow in the order I did them, with their code.

Start each session with `cd ~/node-js-foundation-training` and `nvm use`. Check `node --version` prints v24.

**0. Create the branch and the stage folder.**
- *Goal:* a branch for this stage, like M01.
- *Do:*
  ```bash
  git switch main
  git switch -c feat/module-02/async-toolkit
  mkdir -p modules/m02-modern-javascript-async-await
  ```
- *Why:* every stage is delivered by a branch and a pull request, and the folder is named like the stage folder in the content repo.
- *Result:* `git status -sb` shows the new branch.

**1. Build `sleep` and its check.**
- *Goal:* a promise that resolves after a delay, with a check that proves it.
- *Do:* create `async-toolkit.mjs`:
  ```js
  export function sleep(ms) {
    if (!Number.isFinite(ms) || ms < 0) {
      throw new RangeError(`sleep: ms must be a non-negative number, got ${ms}`);
    }
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
  ```
  And `async-toolkit.check.mjs`, with a tiny `check(name, fn)` helper that awaits `fn` and prints `ok - name`:
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
  ```
- *Why:* `setTimeout` takes a callback, and `sleep` turns it into a promise so the other utilities can `await` it. The 5 ms of slack is there because timers are only approximately accurate.
- *Result:* `node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs` printed `ok - ...` for every check (3 for `sleep`).

**2. Build `retry`.**
- *Goal:* call an async function until it succeeds or the tries run out.
- *Do:* add to `async-toolkit.mjs`:
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
  Add checks for: first success calls `fn` once; a function that fails twice then succeeds is called 3 times with attempts `[0, 1, 2]`; giving up throws a `RetryError` with `attempts` 3 and `cause.message` of the last error; the delay applies between tries but not after the last; `retries: 0` calls `fn` once; a bad `retries` rejects; a bad `delayMs` rejects even when `fn` would succeed first time, and `fn` is never called.
- *Why:* `return await` is what lets the `catch` see a rejection. The `await` in the loop is deliberate, as each attempt must finish first. The error class carries the attempts and the cause so the caller gets the full story. Both options are validated before the first call, because `delayMs` is otherwise only used after a failure, so a bad value would go unnoticed whenever `fn` succeeds first time.
- *Result:* 10 checks passing in total. To see why `return await` matters, copy the folder somewhere temporary, remove the `await` and run the checks: the run crashes with `Error: fail 0`.

**3. Build `withTimeout`.**
- *Goal:* give up waiting for a promise after `ms` milliseconds.
- *Do:* add to `async-toolkit.mjs`:
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
  Add checks for: a fast promise passes its value through; a slow one rejects with a `TimeoutError` near the limit (not at the slow promise's own 500 ms); the original rejection comes through unchanged (same object); a slow promise that rejects after the timeout causes no unhandled rejection; and a child process running `await withTimeout(sleep(5), 10_000)` exits in under 3 s.
- *Why:* `Promise.race` settles with whichever promise settles first. `AbortSignal.timeout` makes the timer, and it does not keep the process alive. `Promise.withResolvers` gives me a promise I can reject from the listener.
- *Result:* 16 checks passing in total. The child-process check is the one that fails if the timer is a plain `setTimeout` that is never cleared: the process then waits for the timer.

**4. Build `mapLimit` (the worked example).**
- *Goal:* run `fn` over a list with at most `limit` calls at once, and give back the results in input order.
- *Do:* the naive idea is `Promise.all(items.map(fn))`, but that starts every call at once, so there is no limit. The next idea is to split the list into chunks of `limit` and `await Promise.all` each chunk, but then each chunk waits for its slowest call and the other slots sit idle. A worker pool avoids both:
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
  Walk through `mapLimit([30, 20, 10, 0], 2, sleepAndReturn)`: two workers start and claim indexes 0 and 1. The worker on index 1 (20 ms) finishes first, claims index 2, and so on. Index 0 (30 ms) finishes last of the first pair, but its result still goes into `results[0]`, so the order is kept.
- *Checks to write:* results stay in input order when calls finish in reverse order; the peak number of running calls equals `limit` exactly (track `running` and `peak` counters); `limit: 1` runs strictly start/end/start/end; it accepts a `Set` and a synchronous `fn`, and passes the index; an empty list and a limit larger than the list work; a failure rejects with the first error and starts no new items (`started` equals `[0, 1, 2]` for six items with `limit: 2` when item 1 fails); a bad `limit` rejects.
- *Why:* each worker pulls work when it is free, so no slot waits for a slower neighbour. The `failed` flag stops the surviving workers from carrying on after `Promise.all` has already rejected.
- *Result:* 23 checks passing in total. Remove `!failed &&` on a scratch copy to see the last check fail.

**5. Compare sequential and parallel fetching.**
- *Goal:* measure the difference on a real API.
- *Do:* create `timing-comparison.mjs`. The core is two functions and a timer:
  ```js
  async function fetchPost(id) {
    const response = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
      signal: AbortSignal.timeout(10_000),
    });
    if (!response.ok) throw new Error(`GET /posts/${id} failed with ${response.status}`);
    return response.json();
  }

  async function fetchSequential() {
    const posts = [];
    for (const id of IDS) posts.push(await fetchPost(id));
    return posts;
  }

  async function fetchParallel() {
    return Promise.all(IDS.map((id) => fetchPost(id)));
  }
  ```
  with `IDS = [1, ..., 10]`. Time each with `performance.now()`, assert that the returned ids equal `IDS`, repeat for several rounds (rounds are awaited one after another so they do not overlap), then print which strategy was faster. The script validates the rounds argument (a positive integer) and reports the result instead of failing when parallel loses, because network conditions vary. Run it with `node modules/m02-modern-javascript-async-await/timing-comparison.mjs 5` (the number is the rounds).
- *Why:* `fetch` does not reject on a 404, so the code checks `response.ok`. `AbortSignal.timeout` makes the request cancel if the server hangs. Asserting the ids stops a wrong or partial result from being timed as a win.
- *Result:* parallel beat sequential in every round. Numbers are under Checkpoint evidence.

**6. Refactor callback-style `node:fs` code to `async`/`await`.**
- *Goal:* the same behaviour with a flat structure.
- *Do:* create `fs-refactor.mjs` with three labelled parts. Start the file with the imports `assert`, `fs`, `fsPromises` (from `node:fs/promises`), `os`, `path` and `promisify`.
  - **Part 1, Before (callbacks).** It reads a JSON file, adds `stamped: true`, writes a new file and reads it back, which nests three callbacks:
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
  - **Part 2, After (promises and `async`/`await`).** The same steps with `node:fs/promises`:
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
  - **Part 3, Self-checks.** Wrap the callback version with `promisify` so the checks can `await` it. Run both versions in a temp directory made with `mkdtemp`, delete the directory in a `finally`, assert both give `{ name: 'toolkit', stage: 2, stamped: true }`, then run both on a missing file and assert the error `code` is `'ENOENT'`, then on a file with malformed JSON and assert the error `name` is `'SyntaxError'`:
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
- *Why:* in the callback version a `JSON.parse` throw inside a callback would not reach the caller, so every `JSON.parse` needs its own `try/catch`, including the final parse of the file I just wrote. I parse inside the `try` and call the callback outside it, so that if the caller's own callback throws, the `catch` does not call it a second time. In the `async` version a parse error and a file error travel the same rejection path. `promisify` works on the callback version because its callback is the last argument and takes `(error, value)`. It is the same job `node:fs/promises` already does for the built-in functions.
- *Result:* all three `ok -` lines printed, and the temp directory was gone afterwards (I counted zero `m02-fs-refactor-` folders left in the temp directory).

**7. Predict the output order.**
- *Goal:* explain the order of `setTimeout`, promise and `await` output.
- *Do:* this is a throwaway, not a deliverable, so I kept it in the scratch folder outside the repo (`~/m02-scratch/event-loop-order.mjs` here). I wrote the prediction in the comment before running anything, then ran it from the repo root with `node ~/m02-scratch/event-loop-order.mjs`:
  ```js
  // Scratch file (not part of the repo): event-loop ordering of setTimeout vs promise vs await.
  // Run from the repo root: node ~/m02-scratch/event-loop-order.mjs
  //
  // My prediction, written BEFORE running it:
  //   1 sync start
  //   4 async fn: sync part
  //   7 sync end
  //   3 promise then
  //   5 after first await
  //   6 after second await
  //   2 setTimeout 0
  //   8 setTimeout 0 scheduled from a microtask
  // Reasoning: all synchronous code runs first, including the part of the async function before its
  // first `await`. Then the call stack is empty, so the whole microtask queue drains (promise
  // callbacks and the continuations after `await`) before the event loop takes the next task, and
  // timers are tasks. The timer scheduled from inside a microtask is a new task behind the first one.

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
- *Why:* write the prediction first, so you find out whether your model of the event loop is right instead of fitting it to the output.
- *Result:* the output was `1, 4, 7, 3, 5, 6, 2, 8`, exactly my prediction.

**8. Run the research experiments.**
- *Goal:* run each topic on the research checklist once, in a tiny experiment with an assertion.
- *Do:* a throwaway script again, `~/m02-scratch/research.mjs` in the scratch folder, with one `check(...)` per topic: language essentials, then async concepts and pitfalls, then a printed `typeof Temporal`. Run it from the repo root on Node 24 with `node ~/m02-scratch/research.mjs`. The one topic that needs the finished toolkit, named versus default exports, is a one-liner from the repo root after step 4:
  ```bash
  node --input-type=module -e "import * as t from './modules/m02-modern-javascript-async-await/async-toolkit.mjs'; console.log(Object.keys(t).sort(), t.default)"
  ```
  The full script:

  <details>
  <summary><code>research.mjs</code> (23 checks)</summary>

  ```js
  // Scratch file (not part of the repo): one tiny experiment per research topic.
  // Run from the repo root: node ~/m02-scratch/research.mjs
  import assert from 'node:assert/strict';
  import { execFile } from 'node:child_process';
  import { promisify } from 'node:util';

  const run = promisify(execFile);
  const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

  async function check(name, fn) {
    await fn();
    console.log(`ok - ${name}`);
  }

  // ---- Language essentials -------------------------------------------------------------------

  await check('let gives each loop pass its own variable; var shares one (closures)', () => {
    const withVar = [];
    for (var i = 0; i < 3; i++) withVar.push(() => i);
    const withLet = [];
    for (let j = 0; j < 3; j++) withLet.push(() => j);
    assert.deepEqual(withVar.map((fn) => fn()), [3, 3, 3]);
    assert.deepEqual(withLet.map((fn) => fn()), [0, 1, 2]);
  });

  await check('const stops reassignment, not mutation', () => {
    const list = [1];
    list.push(2);
    assert.deepEqual(list, [1, 2]);
    assert.throws(() => eval('const x = 1; x = 2;'), TypeError);
  });

  await check('an arrow function keeps the outer `this`; a plain function in a callback does not', () => {
    const counter = {
      n: 0,
      incArrow() { [1, 2].forEach(() => { this.n++; }); },
      incPlain() { [1, 2].forEach(function () { this.n++; }); },
    };
    counter.incArrow();
    assert.equal(counter.n, 2);
    // Modules are strict, so `this` inside the plain callback is undefined.
    assert.throws(() => counter.incPlain(), TypeError);
  });

  await check('destructuring, defaults, rest and spread', () => {
    const { a, b = 10, ...others } = { a: 1, c: 3, d: 4 };
    assert.deepEqual([a, b, others], [1, 10, { c: 3, d: 4 }]);
    const [first, , third, ...tail] = [1, 2, 3, 4, 5];
    assert.deepEqual([first, third, tail], [1, 3, [4, 5]]);
    const merged = { ...{ x: 1, y: 2 }, y: 99 };
    assert.deepEqual(merged, { x: 1, y: 99 });
    const sum = (...nums) => nums.reduce((total, n) => total + n, 0);
    assert.equal(sum(...[1, 2, 3]), 6);
  });

  await check('?. , ?? and ??= only react to null/undefined, unlike || ', () => {
    const user = { profile: null, count: 0, name: '' };
    assert.equal(user.profile?.email, undefined);
    assert.equal(user.count || 5, 5);
    assert.equal(user.count ?? 5, 0);
    assert.equal(user.name ?? 'anon', '');
    user.count ??= 7;
    user.missing ??= 7;
    assert.deepEqual([user.count, user.missing], [0, 7]);
  });

  await check('truthy/falsy and equality pitfalls', () => {
    const falsy = [false, 0, -0, 0n, '', null, undefined, NaN];
    assert.ok(falsy.every((value) => !value));
    assert.ok([] && {} && '0' && 'false'); // empty array, empty object and these strings are truthy
    assert.ok(0 == '' && null == undefined && '1' == 1); // eslint-disable-line eqeqeq
    assert.ok(!(null == 0)); // eslint-disable-line eqeqeq
    assert.ok(NaN !== NaN && Object.is(NaN, NaN));
    assert.equal(typeof null, 'object');
  });

  await check('array methods: map/filter/reduce/find/some/every/flatMap', () => {
    const nums = [1, 2, 3, 4];
    assert.deepEqual(nums.map((n) => n * 2), [2, 4, 6, 8]);
    assert.deepEqual(nums.filter((n) => n % 2 === 0), [2, 4]);
    assert.equal(nums.reduce((total, n) => total + n, 0), 10);
    assert.equal(nums.find((n) => n > 2), 3);
    assert.ok(nums.some((n) => n > 3) && nums.every((n) => n > 0));
    assert.deepEqual(nums.flatMap((n) => (n % 2 ? [n, n] : [])), [1, 1, 3, 3]);
  });

  await check('toSorted/toReversed/with copy; sort/reverse mutate; default sort is by string', () => {
    const original = [10, 9, 1];
    assert.deepEqual(original.toSorted(), [1, 10, 9]); // string order, not numeric
    assert.deepEqual(original.toSorted((x, y) => x - y), [1, 9, 10]);
    assert.deepEqual(original.toReversed(), [1, 9, 10]);
    assert.deepEqual(original.with(0, 99), [99, 9, 1]);
    assert.deepEqual(original, [10, 9, 1]);
    original.sort();
    assert.deepEqual(original, [1, 10, 9]);
  });

  await check('structuredClone makes a deep copy, including Date and Map, but not functions', () => {
    const source = { when: new Date(0), tags: new Map([['a', 1]]), nested: { n: 1 } };
    const copy = structuredClone(source);
    copy.nested.n = 2;
    assert.equal(source.nested.n, 1);
    assert.ok(copy.when instanceof Date && copy.tags instanceof Map);
    assert.throws(() => structuredClone({ fn() {} }), { name: 'DataCloneError' });
  });

  await check('Object.groupBy and Map.groupBy', () => {
    const words = ['apple', 'avocado', 'banana', 'blueberry', 'cherry'];
    const byLetter = Object.groupBy(words, (word) => word[0]);
    assert.deepEqual({ ...byLetter }, {
      a: ['apple', 'avocado'], b: ['banana', 'blueberry'], c: ['cherry'],
    });
    assert.equal(Object.getPrototypeOf(byLetter), null);
    const byLength = Map.groupBy(words, (word) => word.length);
    assert.deepEqual(byLength.get(6), ['banana', 'cherry']);
  });

  await check('Map keeps key types and order; Set de-duplicates and has set methods', () => {
    const map = new Map([[1, 'number'], ['1', 'string']]);
    assert.equal(map.size, 2);
    const a = new Set([1, 2, 3]);
    const b = new Set([3, 4]);
    assert.deepEqual([...new Set([1, 1, 2])], [1, 2]);
    assert.deepEqual([...a.union(b)], [1, 2, 3, 4]);
    assert.deepEqual([...a.intersection(b)], [3]);
    assert.deepEqual([...a.difference(b)], [1, 2]);
    assert.deepEqual([...a.symmetricDifference(b)], [1, 2, 4]);
    assert.ok(new Set([1]).isSubsetOf(a) && !a.isSubsetOf(b));
  });

  await check('class with #private field, getter and setter', () => {
    class Temperature {
      #celsius = 0;
      get fahrenheit() { return this.#celsius * 9 / 5 + 32; }
      set fahrenheit(value) { this.#celsius = (value - 32) * 5 / 9; }
    }
    const t = new Temperature();
    t.fahrenheit = 212;
    assert.equal(t.fahrenheit, 212);
    assert.deepEqual(Object.keys(t), []);
    assert.equal(t.celsius, undefined);
  });

  await check('try/catch/finally, custom error and Error cause', () => {
    class ConfigError extends Error {
      constructor(message, options) {
        super(message, options);
        this.name = 'ConfigError';
      }
    }
    const log = [];
    const parse = (text) => {
      try {
        return JSON.parse(text);
      } catch (error) {
        throw new ConfigError('bad config', { cause: error });
      } finally {
        log.push('finally ran');
      }
    };
    assert.throws(() => parse('{oops'), (error) => {
      assert.ok(error instanceof ConfigError && error instanceof Error);
      assert.ok(error.cause instanceof SyntaxError);
      return true;
    });
    assert.deepEqual(parse('{"ok":true}'), { ok: true });
    assert.deepEqual(log, ['finally ran', 'finally ran']); // finally runs on throw and on return
  });

  // ---- Async concepts ------------------------------------------------------------------------

  await check('try/catch only catches a rejection if you await it', async () => {
    const fail = () => Promise.reject(new Error('nope'));
    let caughtWithoutAwait = false;
    let caughtWithAwait = false;
    let loose;
    try {
      loose = fail(); // forgotten await
    } catch {
      caughtWithoutAwait = true;
    }
    await assert.rejects(loose, /nope/); // handle it now so it never becomes unhandled
    try {
      await fail();
    } catch {
      caughtWithAwait = true;
    }
    assert.deepEqual([caughtWithoutAwait, caughtWithAwait], [false, true]);
  });

  await check('a forgotten await leaves a Promise, which is always truthy', async () => {
    const isReady = async () => false;
    assert.equal(Boolean(isReady()), true);
    assert.equal(Boolean(await isReady()), false);
  });

  await check('Promise.all fails fast; allSettled reports every outcome; race and any pick a winner', async () => {
    const ok = (value, ms) => sleep(ms).then(() => value);
    const bad = (message, ms) => sleep(ms).then(() => { throw new Error(message); });

    await assert.rejects(Promise.all([ok('a', 10), bad('b failed', 5)]), /b failed/);

    const settled = await Promise.allSettled([ok('a', 5), bad('b failed', 5)]);
    assert.deepEqual(settled.map((result) => result.status), ['fulfilled', 'rejected']);

    assert.equal(await Promise.race([ok('slow', 30), ok('fast', 5)]), 'fast');
    await assert.rejects(Promise.race([bad('fast failure', 5), ok('slow', 30)]), /fast failure/);

    assert.equal(await Promise.any([bad('x', 5), ok('first success', 15)]), 'first success');
    await assert.rejects(Promise.any([bad('x', 5), bad('y', 5)]), AggregateError);
  });

  await check('Promise.try turns a sync throw into a rejection and a sync return into a promise', async () => {
    const thrower = () => { throw new Error('sync boom'); };
    await assert.rejects(Promise.try(thrower), /sync boom/);
    assert.equal(await Promise.try(() => 42), 42);
    assert.throws(thrower, /sync boom/); // called directly, it throws instead
  });

  await check('Promise.withResolvers exposes resolve/reject outside the executor', async () => {
    const { promise, resolve } = Promise.withResolvers();
    setTimeout(() => resolve('resolved later'), 5);
    assert.equal(await promise, 'resolved later');
  });

  await check('for await...of and Array.fromAsync consume async iterables', async () => {
    async function* countdown() {
      for (let n = 3; n > 0; n--) {
        await sleep(1);
        yield n;
      }
    }
    const seen = [];
    for await (const n of countdown()) seen.push(n);
    assert.deepEqual(seen, [3, 2, 1]);
    assert.deepEqual(await Array.fromAsync(countdown()), [3, 2, 1]);
  });

  await check('AbortController and AbortSignal.timeout cancel a real fetch', async () => {
    const controller = new AbortController();
    controller.abort();
    assert.equal(controller.signal.aborted, true);
    assert.equal(controller.signal.reason.name, 'AbortError');
    await assert.rejects(
      fetch('https://jsonplaceholder.typicode.com/posts/1', { signal: controller.signal }),
      { name: 'AbortError' },
    );
    await assert.rejects(
      fetch('https://jsonplaceholder.typicode.com/posts/1', { signal: AbortSignal.timeout(1) }),
      { name: 'TimeoutError' },
    );
  });

  await check('fetch needs response.ok checking: a 404 does not reject', async () => {
    const response = await fetch('https://jsonplaceholder.typicode.com/posts/99999');
    assert.equal(response.ok, false);
    assert.equal(response.status, 404);
    assert.deepEqual(await response.json(), {});
  });

  await check('pitfall: forEach(async ...) does not wait; for...of with await does', async () => {
    const doneForEach = [];
    [1, 2, 3].forEach(async (n) => {
      await sleep(10);
      doneForEach.push(n);
    });
    assert.deepEqual(doneForEach, []); // forEach returned before any callback finished
    await sleep(30);
    assert.deepEqual(doneForEach, [1, 2, 3]);

    const doneLoop = [];
    for (const n of [1, 2, 3]) {
      await sleep(1);
      doneLoop.push(n);
    }
    assert.deepEqual(doneLoop, [1, 2, 3]);
  });

  await check('pitfall: an unhandled rejection crashes a Node 24 process; a swallowed one hides the error', async () => {
    await assert.rejects(
      run(process.execPath, ['--eval', 'Promise.reject(new Error("nobody handled me"))']),
      (error) => error.code === 1 && error.stderr.includes('nobody handled me'),
    );
    const swallowed = await Promise.reject(new Error('lost')).catch(() => {});
    assert.equal(swallowed, undefined); // the caller now sees "success" with no value
  });

  // ---- Awareness only ------------------------------------------------------------------------

  console.log(`info - typeof Temporal on Node ${process.version}: ${typeof Temporal}`);
  ```

  </details>
- *Why:* an assertion that passes means my picture of the feature is right, and one that fails shows what I had wrong.
- *Result:* all 23 checks passed on the first run, and the one-liner printed the six export names (`RetryError`, `TimeoutError`, `mapLimit`, `retry`, `sleep`, `withTimeout`) and `undefined` for `t.default`. I wrote each assertion from what I already expected, so this confirmed behaviour; it was not a discovery.

**9. Commit, push, open the pull request and merge it.**
- *Goal:* deliver the stage the way M01 taught, with nothing pushed straight to `main`.
- *Do:* from the repo root, on the stage branch, I re-ran the checks, then committed in three groups, staging files by name and not with `git add .`:
  ```bash
  git add modules/m02-modern-javascript-async-await/async-toolkit.mjs modules/m02-modern-javascript-async-await/async-toolkit.check.mjs
  git commit -m "feat(m02): add async toolkit with sleep, retry, withTimeout and mapLimit"
  git add modules/m02-modern-javascript-async-await/timing-comparison.mjs
  git commit -m "feat(m02): add sequential vs parallel timing comparison"
  git add modules/m02-modern-javascript-async-await/fs-refactor.mjs
  git commit -m "feat(m02): refactor callback node:fs code to async/await"
  git push -u origin feat/module-02/async-toolkit
  ```
  Then I opened the pull request with the GitHub CLI, giving the repo, base and head branch explicitly, and a body with Summary, Changes and Test plan sections:
  ```bash
  gh pr create --repo <user>/node-js-foundation-training --base main --head feat/module-02/async-toolkit \
    --title "feat(m02): add async toolkit, timing comparison and fs refactor" --body "<Summary, Changes, Test plan>"
  ```
  I merged it on GitHub with a merge commit, and then ran `git fetch origin` and `git log --oneline --graph origin/main` to see the result.
- *Why:* each commit is one deliverable that works on its own. The toolkit and its self-checks go together, because the checks are the proof that the toolkit works. Staging by name means I commit exactly what I meant to. The test plan in the PR lists only commands I had just run, with their results. A merge commit, not a squash, keeps the three commits visible in the history.
- *Result:* PR #2 was merged. `origin/main` shows the merge commit with the three commits under it (full history under Checkpoint evidence).

## Concepts worth explaining

**The event loop and the two queues**

JavaScript runs one piece of code at a time on a call stack. When something asynchronous finishes
(a timer fires, a file is read, a promise settles), its callback does not run straight away. It waits
in a queue until the stack is empty. There are two queues that matter here. The **task queue**
(timers, I/O callbacks) and the **microtask queue** (promise callbacks, and the code after an
`await`). The rule that explained my snippet: after each task, and whenever the stack empties, the
event loop drains the **whole** microtask queue before it takes the next task. That is why
`setTimeout(..., 0)` ran after everything promise-related, even though its delay is 0: a timer
callback is a task, and promise callbacks are microtasks that cut in line. A timer scheduled from
inside a microtask (my `8`) is a new task, so it goes behind the one already waiting (`2`). It also
explains why the part of an `async` function before its first `await` runs synchronously, and only
the rest becomes a microtask.

**`Promise.all` versus `allSettled`, `race` and `any`**

They all take several promises and give back one, and differ in what the single promise means.
`all` means "every one succeeded, or fail fast": it rejects as soon as any one rejects, with that
error. `allSettled` means "tell me what happened to each": it never rejects, and gives back a
`status` for every promise. `race` means "whichever settles first", success or failure. `any` means
"the first success": it ignores rejections unless all of them reject, and then it rejects with an
`AggregateError`. I ran each of them in a probe. The one that matters for `mapLimit` is `all`'s
fail-fast: it only reports the first failure and does not stop the other promises. They keep
running, which is why `mapLimit` needs its own `failed` flag.

## What tripped me up

- **A new terminal started on Node 22, again.** A plain `node` in a fresh shell printed v22.23.0. It was v24.22.0 only after `nvm use` in the repo, where `.nvmrc` says 24. A scratch folder without an `.nvmrc` printed `No .nvmrc file found` and ran my experiment on Node 22. That experiment behaves the same on 22 and 24, but I should have copied `.nvmrc` into the scratch folder, which I did for the second experiment.
- **`time ... | grep real` printed nothing in zsh.** I was timing two `node -e` scripts, and zsh's `time` has no line starting with `real`. I redid the measurement with a small Python timer, which gave 0.02 s and 3.02 s.
- **Round 1 of the timing was the slowest every time.** I expected parallel to be fast in every round. Parallel round 1 took 246 ms and 254 ms in the two runs, against 80 to 102 ms in later rounds. I did not investigate it, so I cannot say why.
- **The first crash from a missing `await` was a raw error, not a hint.** When I removed `return await`, the run died with `Error: fail 0` thrown from the check file, and nothing in the message said the cause was a missing `await` in `retry`. I only knew because I had made the change myself.

## Checkpoint evidence

**All `node:assert` self-checks pass.** Output of `node modules/m02-modern-javascript-async-await/async-toolkit.check.mjs` on Node v24.22.0, exit code 0:

```text
ok - sleep waits at least the requested time
ok - sleep resolves with undefined
ok - sleep rejects bad input synchronously
ok - retry returns the first success without retrying
ok - retry retries an async rejection until it succeeds
ok - retry gives up with a RetryError that keeps the last error as cause
ok - retry waits delayMs between tries but not after the last one
ok - retry with retries: 0 calls fn exactly once
ok - retry rejects a bad retries option
ok - retry rejects a bad delayMs even when fn would succeed first time
ok - withTimeout passes through a value that arrives in time
ok - withTimeout rejects with a TimeoutError when the promise is too slow
ok - withTimeout passes through the original rejection unchanged
ok - withTimeout survives the slow promise rejecting after the timeout
ok - withTimeout does not keep the process alive after the promise settles
ok - withTimeout rejects a bad ms synchronously
ok - mapLimit keeps results in input order even when calls finish out of order
ok - mapLimit never runs more than `limit` calls at once, and does reach the limit
ok - mapLimit with limit 1 is sequential
ok - mapLimit passes the index and accepts a sync fn and any iterable
ok - mapLimit handles an empty input and a limit larger than the input
ok - mapLimit rejects with the first error and starts no new work after it
ok - mapLimit rejects a bad limit
```

`fs-refactor.mjs` printed `ok - callback and async/await versions produce the same result` and
`ok - both versions fail with ENOENT for a missing file`, and `ok - both versions fail with SyntaxError for malformed JSON`. The scratch `research.mjs` printed 23 `ok`
lines and `info - typeof Temporal on Node v24.22.0: undefined`, and the exports one-liner listed the six
named exports with no default.

**Timing comparison: parallel beat sequential.** Fetching posts 1 to 10 from JSONPlaceholder, Node
v24.22.0, times in milliseconds.

Run 1 (3 rounds):

| Round | Sequential | Parallel |
| --- | --- | --- |
| 1 | 781 | 246 |
| 2 | 630 | 88 |
| 3 | 628 | 80 |
| Total | 2039 | 413 |

Parallel was 4.9x faster overall.

Run 2 (5 rounds):

| Round | Sequential | Parallel |
| --- | --- | --- |
| 1 | 900 | 254 |
| 2 | 777 | 83 |
| 3 | 861 | 87 |
| 4 | 714 | 88 |
| 5 | 729 | 102 |
| Total | 3982 | 614 |

Parallel was 6.5x faster overall.

*Why parallel wins:* the time in a fetch is almost all waiting for the network, not computing. The
sequential loop starts each wait after the previous one ends, so the total is the sum of ten waits
(about 63 to 90 ms each here). The parallel version starts all ten waits together, so the total is
about the longest single wait plus overhead (about 80 to 100 ms in the settled rounds).

**Event-loop order.** My prediction, written before running, and the actual output were the same:

```text
1 sync start
4 async fn: sync part
7 sync end
3 promise then
5 after first await
6 after second await
2 setTimeout 0
8 setTimeout 0 scheduled from a microtask
```

*My explanation:* all synchronous code runs first, including the
part of the `async` function before its first `await` (1, 4, 7). Then the stack is empty, so the
event loop drains the microtask queue: the promise callback (3), then the continuations after each
`await` (5, 6). Only then does it take the next task, which is the timer (2), and the timer that the
microtask scheduled is a later task behind it (8).

I wrote the explanation above and then said it out loud, as the definition of done asks.

**Merged PR:** https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/2, merged on
2026-10-19 as merge commit `30b6a9e`. It has 3 Conventional Commits and changes 4 files.

**History on `main` after the merge:**

```text
*   30b6a9e Merge pull request #2 from kiannaalexandra-gragg/feat/module-02/async-toolkit
|\
| * 2ce847b feat(m02): refactor callback node:fs code to async/await
| * 4f2cfd3 feat(m02): add sequential vs parallel timing comparison
| * 936f308 feat(m02): add async toolkit with sleep, retry, withTimeout and mapLimit
|/
*   ef59bd6 Merge pull request #1 from kiannaalexandra-gragg/feat/module-01/hello-node
```

The PR's test plan lists the runs I did just before committing: 23 toolkit checks, 3 `fs-refactor`
checks, a 3-round timing run (sequential 2786 ms, parallel 419 ms, 6.7x faster) and the rejected
`0` rounds argument.

## What I'd do differently

- **Add a warm-up round to the timing script and report it separately.** Round 1 was the slowest parallel round in both runs. I do not know the cause, so I would add a warm-up round, then see whether the gap closes. Until I test it, any explanation is a guess.
- **Copy `.nvmrc` into any scratch folder** before running an experiment, so the experiment runs on the pinned Node.
