# Prompt: Write a concept explanation

Use for the "Concepts worth explaining" section of `write-up.md`, and for the "Concepts you need"
section of `training-material.md`. Pick 1–2 ideas per module that deserve more than a line.

## Inputs (provide all of these)

- **Module:** the stage folder.
- **Concept:** the idea to explain, e.g. `git revert` vs `git reset`.
- **My understanding:** your own explanation, even if rough.
- **Where the guided path mentions it:** the stage that names this concept (training material only).

If any input is missing, ask for it before writing anything.

## Prompt

```text
You are helping me explain the concept "<CONCEPT>" for <MODULE>. I am the trainee. The reader is
the next trainee, working alone, who has just done the work and wants to understand it.

Read these first:
- <MODULE>/brief.md, for the stage's scope
- knowledge/rules/content-standards.md and knowledge/rules/coding-standards.md

Write the explanation from MY understanding below. Tighten it. Don't replace it with a generic
textbook answer, and don't add claims I didn't make without flagging them.

The explanation must:
- Go deeper than the guided path does, adding a comparison, a mental model, or a contrast the
  learner can reuse.
- Avoid retelling the step-by-step walkthrough. The reader has already done the steps. The
  explanation says what the steps mean.
- Be correct for the pinned stack. Flag anything in my understanding that looks wrong, and don't
  silently fix it.
- Define terms on first use and stay short enough to read in a few minutes.

My understanding:
<UNDERSTANDING>

Where the guided path mentions it (training material only):
<STAGE_MENTION>

Return the explanation, then a list of anything you flagged as uncertain or incorrect.
```

## Notes

- If the explanation mostly repeats a guided-path stage, cut it down to the part the path can't
  carry, or drop it.
