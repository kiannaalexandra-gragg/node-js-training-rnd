# Pattern: Build and write up (the stage chain)

The repeatable chain for every module. It is this repo's version of "implement and test": the
trainee builds the stage, then captures and teaches it. Run it per stage, not once at the end.

```text
1 Read brief ─► 2 Build slice ─► 3 Verify ─► 4 Capture ─► 5 Write-up ─► 6 Training material ─► 7 Self-check ─► 8 PR
                      ▲                                                                           │
                      └───────────────────── EM feedback, revise ◄────────────────────────────────┘
```

| # | Step | What happens | Output |
| --- | --- | --- | --- |
| 1 | Read the brief | Read `brief.md` and `tasks.md` fully. Note the lab, deliverable, and definition of done. Confirm open questions with the trainer. | Clear scope |
| 2 | Build a slice | Build one lab step on a feature branch, with Conventional Commits. Small slices, not the whole stage. | A working slice |
| 3 | Verify | Check the slice against the definition of done. Run the command, check the file, say the explanation out loud. | Evidence it works |
| 4 | Capture | While it is fresh, write notes on commands, errors, decisions, and why. Real events only. | Raw notes |
| 5 | Write-up | Draft `write-up.md` sections from the notes with `knowledge/prompts/dev/write-up-draft.md`, `how-to-section.md`, and `concept-explainer.md`. Fill it in as you build. | `write-up.md` |
| 6 | Training material | When the write-up has real content, run `knowledge/prompts/dev/training-material-draft.md`. Fill every GAP from your own build. | `training-material.md` |
| 7 | Self-check | Run the quality gates in `knowledge/rules/content-standards.md`: self-contained, concepts complementary, verifiable outcomes, no invented content. | Gates pass |
| 8 | Pull request | Commit with Conventional Commits and open a PR so the EM reviews build, write-up, and training material together. | PR ready for review |

Repeat steps 2–5 for each slice. Run steps 6–8 when the stage's slices are done.

## Rules for the chain

- Do not skip step 3. A slice that is not verified against the definition of done does not get written
  up as if it worked.
- Capture in step 4, not after the whole stage. Notes written later get thin.
- Step 6 needs step 5. Don't draft training material from the brief alone: it would miss what you
  learned, and the AI would fill the gap with invention.
- The EM's feedback goes back into the build **and** the documents. Fix both before the next stage.
- Reference and blast-radius questions during the chain (for example, which modules mention a
  version you are about to change) use graphify, not grep. See `AGENTS.md`.
