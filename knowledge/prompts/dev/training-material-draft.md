# Prompt: Draft training-material.md

Use once your `write-up.md` has real content, to produce the learner-facing
`modules/mNN-*/training-material.md` for the next trainee.

## Inputs (provide all of these)

- **Module:** the stage folder, e.g. `modules/m01-dev-environment-git-github/`.
- **Filled-in `write-up.md`:** in the same folder. It must have real content in at least "How to
  build it". Concepts and "What tripped me up" are used when present.
- **`brief.md`:** in the same folder.

**Stop conditions.** If `write-up.md` is missing or its "How to build it" section is empty or
mostly GAPs, do not draft. List which write-up sections are needed and stop.

## Prompt

```text
You are drafting training-material.md for <MODULE>. I am the trainee, and the reader is the next
trainee, working alone with no trainer. Read, in this order:

1. knowledge/rules/content-standards.md, which defines the sections and the three rules below
2. knowledge/rules/coding-standards.md
3. <MODULE>/brief.md, the source of the requirements
4. <MODULE>/write-up.md, the source of the how-to, concepts, and pitfalls

Write <MODULE>/training-material.md with these sections, in this order:
1. What you'll build and why
2. Before you start
3. Guided path
4. Concepts you need
5. Common pitfalls
6. Check yourself against the definition of done

Three rules apply to the whole document.

SELF-CONTAINED GUIDED PATH
- The learner must finish every required task from this file alone. Use write-up.md as a source
  and move every instruction, command, and explanation a stage needs into the stage itself.
- Never say "see the write-up" and never link to write-up.md, brief.md, or tasks.md as somewhere
  the learner must go to continue.
- Each stage gives: the goal, the steps, and why they are done that way at that point.

COMPLEMENTARY CONCEPTS
- The guided path says what to do and why at this point, briefly.
- "Concepts you need" holds deeper explanations, comparisons, and mental models that help the
  learner understand what they did. It must not repeat a stage's walkthrough. A stage names a
  concept and gives one line on why it matters there.

VERIFIABLE OUTCOMES
- Every stage ends with a "Done when" checklist of items the learner can verify themselves, each
  one of:
  - [ ] a command succeeds (name the command and what success looks like)
  - [ ] an artifact exists (name the file or object and the content it must have)
  - [ ] the learner can explain it (state the question to be answered in their own words)
- Take the checks from brief.md: its lab steps, deliverable, and definition of done. Do not
  invent requirements. A check the brief doesn't require is allowed only when it is the direct
  observable result of a brief-required step.

Other rules:
- Use ONLY the brief and the write-up as sources. Do not invent steps, commands, outputs,
  pitfalls, or versions. If the brief requires something the write-up does not cover, add a
  "GAP: <requirement>" marker in the stage and list it for me. Don't fill it in.
- Leave out anything personal to me: What I'd do differently, my checkpoint evidence, my PR link
  or repo, and asides about my own build.
- Rewrite "What tripped me up" as warnings addressed to the learner.
- Match the pinned stack and code standards. Flag conflicts.
- In "Check yourself against the definition of done", turn the brief's definition of done into a
  checklist the learner can run through at the end.

After the file, return a separate block (not part of the file) with:
- TRACE: for each stage and each "Done when" item, the brief line it comes from
- GAPS: brief requirements the write-up doesn't cover
- CONFLICTS: anything that disagrees with the pinned stack or code standards
- STANDALONE CHECK: where a learner would need another file to continue, if anywhere

Do not claim the guide is complete if any GAP or STANDALONE CHECK item remains.
```

## After drafting

1. Fill every GAP from your own build, and don't let the AI invent it.
2. Run the quality gates at the bottom of `knowledge/rules/content-standards.md`.
3. Have someone who didn't build the stage follow the guide cold, if possible.
