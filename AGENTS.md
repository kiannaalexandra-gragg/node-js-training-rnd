# AGENTS.md

Project rules for AI coding agents working in this repository. These apply to any tool.

## 1. What this repo is

The Build-to-Teach version of the **Node.js Developer Training Program**: nine training stages
(M01–M09) and a capstone (M10), 10 modules in about 80 hours over four weeks. A trainee builds
toward each stage's deliverable from a brief, and writes up how they did it so the next cohort can
learn from it.

- A **module** here is a training stage, **not** a JavaScript/Node.js module. Say which you mean
  when it could be ambiguous.
- This is a **content repo**. There is no application, no `package.json`, no CI, and no Task API
  here. Trainees build their own projects in their own repos.
- Source of truth: `docs/arch-docs/` (solution design), `docs/prd/` (PRD), and
  `docs/build-to-teach-framework.md`.

## 2. Layout

```text
.
├─ AGENTS.md                      # this file
├─ README.md                      # root overview (do not edit unless asked)
├─ docs/
│  ├─ arch-docs/                  # solution design
│  ├─ prd/                        # product requirements
│  └─ build-to-teach-framework.md
├─ modules/
│  └─ mNN-<name>/                 # one folder per stage, M01–M10
│     ├─ brief.md                 # EM-authored: what to build, definition of done
│     ├─ tasks.md                 # EM-authored: checklist
│     ├─ write-up-template.md     # EM-authored: the template
│     ├─ write-up.md              # trainee-authored: filled-in copy of the template
│     └─ training-material.md     # trainee-authored: learner-facing teaching content
├─ knowledge/
│  ├─ rules/
│  │  ├─ arch-summary.md          # generated from the arch doc
│  │  ├─ coding-standards.md      # pinned stack and code standards
│  │  └─ content-standards.md     # quality bar for write-up and training material
│  ├─ prompts/dev/                # prompts for authoring write-ups and training material
│  └─ patterns/
│     └─ build-and-write-up-chain.md
└─ graphify-out/                  # the ONLY knowledge graph (repo root, git-ignored)
```

`write-up.md` and `training-material.md` appear in a module folder when the trainee builds that
stage. Do not create empty placeholder copies.

## 3. Content rules

- **`brief.md`, `tasks.md`, `write-up-template.md` are EM-authored.** They contain no lesson, no
  steps, no code, and no solutions. Do not edit them unless asked, and never add teaching content
  to them.
- **`write-up.md` is the trainee's working record.** It uses the seven sections of the template,
  filled in as the stage is built.
- **`training-material.md` is learner-facing.** It is built from the module's `brief.md` and the
  learner-useful parts of `write-up.md`, and it must **stand alone**: a learner can finish the
  stage's required tasks from it without opening another file. It has a guided path, a concepts
  section that complements the path without repeating it, and a **Done when** checklist ending
  every stage.
- **Never invent experience.** Do not make up a build, an error, a command output, a decision, or a
  checkpoint result. When a source does not cover something, mark it `GAP:` and ask the trainee.
  AI may shape and tighten the trainee's own material, but it does not supply it.
- Full rules and quality gates: `knowledge/rules/content-standards.md`.
- Authoring prompts: `knowledge/prompts/dev/` (`write-up-draft.md`, `how-to-section.md`,
  `concept-explainer.md`, `training-material-draft.md`). There are no prompts for briefs or tasks.
- Workflow: `knowledge/patterns/build-and-write-up-chain.md`.

## 4. Pinned stack

Any content that names a version must match this table. Details and code standards are in
`knowledge/rules/coding-standards.md`.

| Layer | Version |
| --- | --- |
| Node.js | 24 LTS (decided), pinned in `.nvmrc` |
| TypeScript | 7.0 from M04. Plain JavaScript in Week 1. |
| Express | 5.x |
| Prisma ORM | 7.x, always as `@7` (`npx prisma@7`). Never an unversioned `prisma`. |
| Database | SQLite only |
| Zod | 4.x |
| Jest / Supertest | 30.x / 7.x |

Free tools only. No Docker and no paid licences. Do not teach removed Express 4 patterns. Open
decision: the import-extension convention (P2) is not settled, so state which one a stage uses.

## 5. Graphify: reference and call-site questions

A knowledge graph is built at `graphify-out/` in the **repo root**. It is the only one.

**Use graphify, not grep, for reference, call-site, and blast-radius questions.** For example: what
references this file, term, or version? What breaks if I rename or remove this? Does anything
depend on this?

- Do not fall back to grep or manual file reading for these questions just because grep is more
  familiar. `graphify explain` gives structural confirmation from real graph edges, not just textual
  absence, and it is faster to trust for "does anything reference this" than scanning grep output
  across dozens of files.
- Commands: `graphify explain "<node>"`, `graphify path "<A>" "<B>"`, `graphify query "<question>"`.
- **grep is still the right tool** for things the graph does not model well: exact string content,
  comments, and config file values.
- **Run graphify from the repo root only.** It reads and writes `graphify-out/` relative to the
  current directory, so running it from a subfolder creates a nested `graphify-out/`. There must be
  exactly one, at the root. If another appears, delete it.

## 6. Security

- Never commit secrets: no tokens, real `.env` values, or private URLs. Examples use fake values.
- Show an `.env.example` in examples, never a real `.env`.
- `.gitignore` conventions for a Node project: `node_modules`, `.env`, `*.db`, the generated Prisma
  client, `dist`, `coverage`.

## 7. Git conventions

- **Conventional Commits** for every commit (`feat:`, `fix:`, `docs:`, `chore:`, …).
- Work on a feature branch, named like the existing `feat/module-N/<topic>`. **No direct pushes to
  `main`**.
- Open a pull request for review. The EM reviews the build, the write-up, and the training material
  together.
- Public training repos by program convention.
- Never rewrite published history, and do not commit generated or local files (`graphify-out/`,
  `.claude/`).

## 8. Quality gates

Before a stage is ready for review:

- [ ] Every versioned claim matches the pinned stack table above.
- [ ] `brief.md` and `tasks.md` contain no lesson, steps, code, or solutions.
- [ ] `write-up.md` has all seven template sections.
- [ ] `training-material.md` traces to `brief.md` and `write-up.md`, covers every required task,
      stands alone, and ends every stage in a **Done when** checklist with checks from the brief.
- [ ] No concept explanation repeats a stage's walkthrough.
- [ ] No invented experience and no unresolved `GAP:` markers.
- [ ] Relative links resolve. No secrets.
- [ ] Exactly one `graphify-out/`, at the repo root.
