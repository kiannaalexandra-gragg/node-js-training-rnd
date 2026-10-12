# M01 write-up: backlog from /validate (2026-10-12)

Minor items deferred. None blocks the checkpoint review.

- **Windows/Linux commands are untested.** The winget IDs resolve in the winget-pkgs catalogue, but no
  Windows or Linux run happened. The nvm-windows "does not read `.nvmrc`" point comes from its README, and
  the PowerShell 5.1 UTF-16 point comes from documented behaviour. Have a Windows trainee dry-run Steps 1-3.
- **Length.** `write-up.md` is 960 lines. Consider splitting each Step into its own LMS lesson.
- **Confirm with the trainer:** one `node-js-training` repo with a folder per module (the brief says
  "public training repos"), and how the Task API (M05-M09) is carried between folders (suggestion: tag each
  finished stage, for example `m05-v1`).
- **Not run:** optional branch clean-up (Step 7), `git reset --hard HEAD@{2}` recovery (Step 2.4) and the
  rebase scenario (Concepts). All three are labelled "not run" in the write-up.
- **Trainee-only check:** saying `revert` vs `reset` out loud, unaided, can't be verified from the files.
