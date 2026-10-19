# M02: backlog

Items found in VALIDATE (2026-10-19). None block the stage.

## [BACKLOG] The training material is untested on Windows

The commands in Stages 1 to 10 are for macOS or Linux shells (`~/m02-scratch`, `mkdir -p`, `node -e`
with double-quoted one-liners, `git`/`gh` from a POSIX shell). The build ran on macOS with nvm and
zsh only. The solution design says trainers dry-run Modules 1 to 9 on each OS in the cohort, so M02
needs a pass in a Windows shell, mainly for the quoting in the `node -e` and `--input-type=module -e`
one-liners.

## [BACKLOG] The guided exercises have not been tried by someone who did not build the stage

Stages 3 to 8 were restructured into exercises: requirements, a scaffold with hints, self-checks, and
a collapsed reference. The reference code and the checks were verified by script, and each scaffold was
confirmed to fail: Stages 3 to 6 and 8 at their own stage's checks, and Stage 7 with a "not implemented
yet" error from the script, since it has no self-checks. What has not been tested is whether a learner
can finish from the hints, or whether two or three hints per stage is the right amount. Run a cold read with
someone who did not build it.

## [BACKLOG] ES module convention differs from `coding-standards.md`

`coding-standards.md` says ES modules via `"type": "module"` throughout. The build repo has no
`package.json` yet, so the M02 files use the `.mjs` extension and explicit `./async-toolkit.mjs`
specifiers instead. This stage is plain JavaScript, so it does not settle the open P2 question
(`.ts` versus `.js` import extensions). The EM should decide whether a `package.json` with
`"type": "module"` should be introduced earlier, since the `.mjs` choice will not carry into M03.

## [BACKLOG] The 10-hour scope has not been timed

The brief allocates 10 hours. The guide shows no per-stage times, and the build was not timed, so
there is no evidence yet that the scope (a research stage, four utilities, a timing script, a
refactor and a PR) fits. Record how long the first cohort takes per stage before anyone relies on the
figure.

## [BACKLOG] Graph community labels for M02 are unreliable

`graphify query` for M02 returns the new training material under the "M03 — Node.js Runtime & npm"
community, and the M02 `write-up-template.md` under "M04 — TypeScript Basics...". The edges are
sensible (each file connects to its own headings and to the M01 and M03 briefs it names), but the
community labels do not match the module folders. Treat graph labels as noise for M02 until the graph
is rebuilt, and use the edges and file paths instead.
