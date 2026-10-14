# Content Standards

The quality bar for the two trainee-authored documents each module produces. Both live in the
module folder, next to the EM-authored `brief.md` and `tasks.md`.

| File | Audience | Purpose |
| --- | --- | --- |
| `write-up.md` | You, the EM, and your future self | Your working record. A filled-in copy of `write-up-template.md`, written as you build. |
| `training-material.md` | The next trainee, working alone | Learner-facing teaching content that guides them through the module's required tasks. |

`brief.md` and `tasks.md` are EM-authored. They contain no lesson, no steps, no code, and no
solutions. Never add any of that to them. Teaching content belongs in `training-material.md`.

## Rules for both documents

1. **Real experience only.** Describe what was actually built and what actually happened. Do not
   invent a build, an error, a decision, a command output, or a checkpoint result. If something
   did not happen, leave a gap and mark it. AI may shape and tighten the trainee's own material; it
   does not supply experience.
2. **Teach the reasoning, not just the commands.** A step says what to do and why at that point.
3. **Write for someone with no trainer to ask.** Define terms on first use. A "module" in this repo
   means a training stage, not a JS/Node.js module. Say which you mean when it could be ambiguous.
4. **Match the pinned stack** in `knowledge/rules/coding-standards.md`, and use its code standards
   for every snippet.
5. **No secrets or personal data.** No tokens, real `.env` values, or private URLs.

## `write-up.md`

- Keep the seven template sections, in order: What I built, Why it's built this way, How to build
  it, Concepts worth explaining, What tripped me up, Checkpoint evidence, What I'd do differently.
- Fill it in **as you build**, not after the build already works.
- Checkpoint evidence is real: a merged PR link, a file that exists, an explanation in your own words.

## `training-material.md`

Standalone, learner-facing, and built from two sources: the requirements in the module's
`brief.md`, and the learner-useful parts of your `write-up.md` (How to build it, Concepts worth
explaining, What tripped me up).

Never carry over what is personal to you: What I'd do differently, your own checkpoint evidence
(your PR link, your repo), or asides about your own build.

### Sections, in order

1. **What you'll build and why:** from the brief's objective and scope.
2. **Before you start:** prerequisites, stack constraints and pinned versions.
3. **Guided path:** the stages that walk the learner through the required tasks.
4. **Concepts you need:** deeper explanations that complement the guided path.
5. **Common pitfalls:** from What tripped me up, rewritten as warnings.
6. **Check yourself against the definition of done:** the brief's definition of done as a
   self-check list.

### Rule 1: the guided path is self-contained

The learner must be able to finish every required task from `training-material.md` alone.

- `write-up.md` is a **source for authoring**, not a document the learner opens. Move the
  instructions, commands, and explanations a stage needs into the stage itself.
- Never write "see the write-up", and never link to `write-up.md`, `brief.md`, or `tasks.md` as a
  place the learner must go to continue.
- Each stage states the goal, the steps, and why they are done this way at that point. Use an
  example the learner can follow, shown with the reasoning, not just the commands.

### Rule 2: concepts complement the guided path, not repeat it

- The guided path explains **what to do and why at this point**, and stays short.
- The concepts section holds what the path cannot: deeper explanations, comparisons, and mental
  models that help the learner understand what they just did.
- Do not restate a stage's walkthrough in the concepts section. A stage may name a concept and give
  one line on why it matters here. The full explanation lives in Concepts only.
- Each concept is useful after the learner has done the work, and adds a comparison or a model that
  the path did not.

### Rule 3: every stage ends in a verifiable outcome

Each stage of the guided path closes with a **Done when** checklist the learner can verify
themselves. Each item is one of three kinds:

- [ ] **A command succeeds:** name the command and what success looks like.
- [ ] **An artifact exists:** name the file or object and the content it must have.
- [ ] **The learner can explain it:** state the question they should be able to answer in their own
  words.

The checks come from the module's `brief.md`: its lab steps, its deliverable, and its definition
of done. Do not invent requirements. A check that the brief does not require is allowed only when
it is the direct, observable result of a brief-required step (for example, `git log` showing the
commits that a required step creates). Where a brief requirement has no check yet, keep it as a gap
for the trainee to fill, rather than making one up.

### Quality gates

- [ ] Every required task in `brief.md` is covered by a stage.
- [ ] No stage depends on opening another file to continue.
- [ ] No concept explanation duplicates a stage's walkthrough.
- [ ] Every stage ends in a **Done when** checklist, and every check traces to the brief.
- [ ] Versions and code match `knowledge/rules/coding-standards.md`.
- [ ] Nothing personal to the author carried over from the write-up.
