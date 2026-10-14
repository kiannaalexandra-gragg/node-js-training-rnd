# Prompt: Draft a write-up section

Use while building a stage, to turn your raw notes into a section of `write-up.md`.

## Inputs (provide all of these)

- **Module:** the stage folder, e.g. `modules/m01-dev-environment-git-github/`.
- **Section:** which `write-up.md` section to draft. Use the headings in that module's
  `write-up-template.md`.
- **My notes:** what you actually did and saw. Commands you ran, errors you hit, decisions you
  made, and why.

If any input is missing, ask for it before writing anything.

## Prompt

```text
You are helping me write one section of my write-up for <MODULE>. I am the trainee, and the
write-up is my own working record of what I built.

Read these first:
- <MODULE>/write-up-template.md, for the heading and its guiding questions
- <MODULE>/brief.md, for what the stage requires
- knowledge/rules/content-standards.md and knowledge/rules/coding-standards.md

Draft the section "<SECTION>" using ONLY the notes below.

Rules:
- Use my notes as the only source of what happened. Do not invent a command, an error, an output,
  a decision, or a result. If the guiding question needs something my notes don't contain, write
  "GAP: <what I still need to capture>" instead of filling it in.
- Answer the template's guiding questions for this section directly.
- Keep my voice. Tighten and clarify; don't add experience I didn't have.
- Explain the reasoning behind decisions, not just what I did.
- Match the pinned stack and code standards. Flag anything in my notes that conflicts, and don't
  silently correct it.

My notes:
<NOTES>

Return the drafted section, then a short list of GAPs and any conflicts you flagged.
```

## Notes

- Never use this for "What tripped me up" or "Checkpoint evidence" without real notes. They stay
  empty until you have captured them.
- For the "How to build it" and "Concepts worth explaining" sections, prefer `how-to-section.md`
  and `concept-explainer.md`.
