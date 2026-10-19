# Day 0 scaffold: backlog

Items found in VALIDATE (2026-10-14). None block the scaffold.

## [BACKLOG] The graph may not capture references between markdown files

`graphify explain "AGENTS.md"` showed one edge only (`AGENTS.md --contains--> AGENTS.md`), with no
edges to the `knowledge/` files it mentions. The `PostToolUse` hook runs `graphify update`, which
re-extracts code files and does not use an LLM, so doc-to-doc references may not be modelled. This
repo is almost all markdown.

The AGENTS.md rule (use graphify for reference and blast-radius questions) is kept as written. Until
the docs are extracted:

- An empty `graphify explain` result is **not** proof that nothing references a file.
- Run `/graphify --update` after structural changes to docs, so semantic extraction covers them.
- Re-test with a known reference, e.g. `explain "Content Standards"`, and confirm the edges to the
  prompts that cite it.

## [BACKLOG] The prompts have not been run against a real stage

No `write-up.md` or `training-material.md` exists yet, so the four prompts in `knowledge/prompts/dev/`
are untested. Dry-run them on M01 once its write-up has real content, then tighten wording where the
output invents content, repeats the walkthrough in Concepts, or leaves a stage without a **Done when**
checklist.

## [BACKLOG] Native AGENTS.md loading is unconfirmed in a live session

`CLAUDE.md` was removed so Claude Code reads `AGENTS.md` natively (documented for v2.1.277+; this
machine runs 2.1.299). Confirm in a fresh session: the "no CLAUDE.md found; AGENTS.md loaded" line, or
`/context` under "Memory files". If it does not load, restore `CLAUDE.md` containing `@AGENTS.md`.
