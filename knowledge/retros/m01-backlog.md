# M01: backlog

Items found in VALIDATE (2026-10-19). None block the stage.

## [BACKLOG] The training material is untested on Windows

Stage 1 to Stage 7 use macOS or Linux shell commands (`/tmp`, `touch`, `printf`, `rm -rf`, `$SHELL`),
and the build ran on macOS with nvm only. The solution design says trainers dry-run Modules 1 to 9
on each OS in the cohort, so M01 needs a pass in a Windows shell and with nvm-windows, fnm and Volta.

## [BACKLOG] The training material has an exercise the write-up does not contain

Stage 1's terminal-basics block (`~/terminal-practice`) uses the same commands as the write-up's
Step 1, but as a standalone practice exercise. Stage 1 also tells the learner to run `gh auth login`
if `gh auth status` shows they are signed out, which the build never needed. Both are linked to or
taken from official documentation. Confirm them with a cold read by someone who did not build the
stage.
