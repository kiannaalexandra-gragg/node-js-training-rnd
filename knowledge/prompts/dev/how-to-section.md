# Prompt: Shape the "How to build it" section

Use to turn your build into the "How to build it" section of `write-up.md`: a guide that teaches
the reasoning, not only the commands. This section is the main source for the guided path in
`training-material.md`.

## Inputs (provide all of these)

- **Module:** the stage folder.
- **My build notes:** the steps you actually took, in order, with the commands and the reasoning
  behind each.
- **My worked example:** the real example from your build that you want to teach with.

If any input is missing, ask for it before writing anything.

## Prompt

```text
You are helping me write the "How to build it" section of my write-up for <MODULE>. I am the
trainee. This section teaches the next trainee how to take the stage from start to finish.

Read these first:
- <MODULE>/write-up-template.md, for what this section must contain
- <MODULE>/brief.md, for the lab and definition of done
- knowledge/rules/content-standards.md and knowledge/rules/coding-standards.md

Structure my notes into an ordered guide, using ONLY my build notes and worked example.

For each step, include:
1. The goal of the step, in one line.
2. What to do, with the exact commands or code I used.
3. Why it is done this way at this point.
4. The result that shows it worked.

Rules:
- Use my worked example to show the reasoning, not just the commands.
- Do not invent steps, commands, outputs, or results. If a lab step in the brief has no matching
  notes, write "GAP: <the brief step I haven't documented>".
- Keep each step's "why" to what matters at that moment. Deeper explanations belong in a separate
  concepts section. Point to the concept by name only.
- Cover every lab step the brief requires, in a sensible order.
- Match the pinned stack and code standards. Flag conflicts.

My build notes:
<NOTES>

My worked example:
<EXAMPLE>

Return the drafted section, then a list of GAPs and conflicts.
```
