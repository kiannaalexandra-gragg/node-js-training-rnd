# M01 — Dev Environment, Git & GitHub: Training Material

**Week 1 · Day 1 · 4 hours**

A "module" in this program means a training stage, not a JavaScript module.

## Builds on

None — this is the first stage.

## 1. What you'll build and why

You will set up the toolchain used for the whole program, learn the everyday Git workflow, and take
one change through the cycle that every later stage uses: **branch → pull request → merge**. You
will also create a real merge conflict and resolve it yourself.

By the end you will have a public GitHub repo containing:

- a `hello-node` script,
- a `.nvmrc`, a `.gitignore` and an `.env.example`,
- a **merged pull request** with 3 or more Conventional Commits, with a resolved merge conflict in
  its history.

Why this stage comes first: every other stage is delivered as a pull request, so you need the
workflow before you need any Node.js skill. The conflict is deliberate. Meeting one on purpose, when
nothing is at stake, makes the first real one much less stressful.

## 2. Before you start

You need a free GitHub account, a terminal, and the tools below. Stage 1 checks each one and links
the official installation guide for any that are missing.

| Tool | What it is for | Installation guide |
| --- | --- | --- |
| GitHub account | Hosts your repo and pull requests | [Creating an account on GitHub](https://docs.github.com/en/get-started/start-your-journey/creating-an-account-on-github) |
| VS Code | Code editor | [VS Code setup](https://code.visualstudio.com/docs/setup/setup-overview) |
| Git | Version control | [Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) |
| A Node version manager | Installs and switches Node versions | [nvm](https://github.com/nvm-sh/nvm#installing-and-updating) (macOS, Linux), [nvm-windows](https://github.com/coreybutler/nvm-windows), [fnm](https://github.com/Schniz/fnm) or [Volta](https://docs.volta.sh/guide/getting-started) |
| Node.js 24 LTS | The runtime, installed through the version manager | [Node.js downloads](https://nodejs.org/en/download) |
| Bruno | Free API client, used from a later stage | [Bruno downloads](https://www.usebruno.com/downloads) |
| GitHub CLI (`gh`) | Creates repos and PRs, and signs you in | [GitHub CLI installation](https://github.com/cli/cli#installation) |

**GitHub authentication** can be the GitHub CLI, a credential manager, or SSH. The commands in this
guide use `gh`. The alternatives are
[Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager) and
[SSH keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh).

The commands use `nvm`. If you use another version manager, use its equivalent of "install Node 24"
and "use the version in `.nvmrc`".

**Pinned for the program:** Node.js 24 LTS, plain JavaScript only in this stage (TypeScript starts
in a later stage), free tools only, and a **public** repo. Public is the program's convention for
every stage, because branch protection on private repos depends on your GitHub plan and a public
repo avoids that dependency.

A **Conventional Commit** is a commit message that starts with a type and a colon: `feat:` for new
functionality, `fix:` for a bug fix, `docs:` for documentation, `chore:` for upkeep that changes no
behaviour. Example: `feat: add hello-node script`. See
[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/). Use one for every commit in
this guide.

## 3. Guided path

Commands are for macOS or Linux shells. Replace `<user>` with your GitHub username and `<repo-name>`
with the name you give your repo, for example `node-js-foundation-training`. The commands keep the
repo in your home folder, `~/<repo-name>`, so you can return to it from anywhere.

### Stage 1 — Check the toolchain and terminal basics

**Goal:** know every tool works, and that you are comfortable with the terminal commands the rest of
the stage relies on.

#### Step 1 — Check each tool

```bash
git --version
node --version
nvm --version
code --version
gh --version
gh auth status
```

Also confirm Bruno is installed. For any tool that fails, use its guide in section 2, then run the
check again.

**Important:** on macOS, `code --version` only works after you install the `code` command in your
PATH. The [VS Code setup guide](https://code.visualstudio.com/docs/setup/setup-overview) covers it.

#### Step 2 — Install Node 24 if it is missing

```bash
nvm ls
```

If no `v24.x` version is listed:

```bash
nvm install 24
```

**Important:** having Node 24 installed does not mean every new terminal starts on it. Your default
version can still point at an older one. Stage 2 fixes that for your repo with `.nvmrc`.

#### Step 3 — Check you are signed in to GitHub

```bash
gh auth status
```

It should say you are logged in to github.com. If it does not, run `gh auth login` and follow the
prompts, using the [`gh auth login` guide](https://cli.github.com/manual/gh_auth_login), then run
`gh auth status` again.

#### Step 4 — Practise the terminal basics

```bash
echo $SHELL                              # which shell you are in
mkdir -p ~/terminal-practice/inner       # make a folder, and any parent folders
cd ~/terminal-practice                   # move into it
printf 'hello\n' > note.txt              # write a file (> replaces, >> appends)
printf 'again\n' >> note.txt             # append to it
cat note.txt                             # print a file
ls -a                                    # list files, including hidden ones
touch scratch.txt                        # create an empty file
rm scratch.txt                           # delete a file
cd ~ && rm -rf ~/terminal-practice       # leave, then delete the folder
```

**Important:** `rm` deletes for good, with no recycle bin. Check the path before you press Enter,
especially with `rm -rf`.

**Why it matters:** every later step, from Git to running a script, is typed in the terminal. A
failed check now is a setup problem. The same failure later looks like a Git or Node problem.

**Done when:**

- [ ] `git --version`, `code --version` and `gh --version` each print a version.
- [ ] Bruno is installed.
- [ ] `nvm ls` lists a `v24.x` version.
- [ ] `gh auth status` says you are logged in to github.com.
- [ ] You can create a folder, write a file, print it and delete both from the terminal.

### Stage 2 — Create the public repo and make the first commit

**Goal:** an empty local repo wired to an empty public GitHub repo, with a `main` branch to open pull
requests against, and Node 24 pinned.

#### Step 1 — Create the folders and the local repo

Each stage gets a folder under `modules/`.

```bash
mkdir -p ~/<repo-name>/modules/m01-dev-environment-git-github
cd ~/<repo-name>
git init -b main
```

#### Step 2 — Pin Node 24

```bash
printf '24\n' > .nvmrc
nvm use
node --version
```

Expected output:

```text
v24.x.x
```

The exact patch version varies.

#### Step 3 — Create the public repo and add it as the remote

```bash
gh repo create <repo-name> --public --description "Trainee builds for the Node.js Developer Training Program"
git remote add origin https://github.com/<user>/<repo-name>.git
git remote -v
```

`gh repo create` without a source only creates the GitHub side, so nothing is pushed yet. `git
remote -v` should list `origin` for both fetch and push.

#### Step 4 — Write a README and push the first commit

Create `README.md`. A short one is enough:

```markdown
# <repo-name>

My hands-on builds for the Node.js Developer Training Program.

## Conventions

- Node.js 24 LTS, pinned in `.nvmrc`. Run `nvm use` in the repo root.
- Work happens on feature branches named `feat/module-NN/<topic>`. Nothing is pushed straight to `main`.
- Commits follow Conventional Commits.
- Real secrets are never committed. Stages that need configuration ship an `.env.example`.
```

Then commit and push:

```bash
git add .nvmrc README.md
git commit -m "docs: add README and pin Node 24 in .nvmrc"
git push -u origin main
```

**Important:** stage files by name, not with `git add .`, so you commit exactly what you meant to.
This first push is the only one you make straight to `main`.

**Why it matters:** `.nvmrc` is the single place that records the Node version, so everyone on the
cohort runs the same one, and `nvm use` with no argument reads it. Your shell's default Node can be
older than the one you installed, so always check `node --version` inside the repo. A pull request
merges one branch into another, and a brand-new GitHub repo has no branch at all, so `main` has to
exist before you can open a PR.

**Done when:**

- [ ] `cat .nvmrc` prints `24`, and `node --version` prints a `v24.` version inside the repo.
- [ ] `git remote -v` lists `origin` for fetch and push.
- [ ] The repo is visible on GitHub without logging in, with `README.md` and `.nvmrc` on `main`.
- [ ] `git log --oneline` shows one commit with a Conventional Commit message.

### Stage 3 — Practise the core Git commands

**Goal:** see each command move a change between the working tree, the staging area and the history,
clone a repo, and see `revert` and `reset` side by side.

**Important:** do this in a throwaway repo **outside** your project, so mistakes cost nothing and your
project history stays clean.

#### Step 1 — Make the practice repo, stage a file and commit it

```bash
cd "$(mktemp -d)"                # a fresh empty folder, safe to re-run
git init -b main
echo "line one" > notes.txt
git status --short               # ?? notes.txt  (new, untracked)
git add notes.txt
git status --short               # A  notes.txt  (staged)
git commit -m "chore: add notes"
```

#### Step 2 — Inspect, unstage, discard and stash an edit

```bash
echo "line two" >> notes.txt
git diff                         # the edit, not yet staged
git add notes.txt
git diff --staged                # the same edit, now staged
git restore --staged notes.txt   # unstage; the edit stays in the file
git restore notes.txt            # throw the edit away
echo "work in progress" >> notes.txt
git stash                        # set the edit aside; the file is back to "line one"
git stash pop                    # bring it back
git commit -am "feat: add wip line"
```

#### Step 3 — Undo a commit with `revert`

```bash
git revert --no-edit HEAD        # undo it with a NEW commit
git log --oneline
```

Expected output (your hashes differ):

```text
<hash> Revert "feat: add wip line"
<hash> feat: add wip line
<hash> chore: add notes
```

#### Step 4 — Undo a commit with `reset`

```bash
echo "oops" >> notes.txt
git commit -am "fix: oops"
git reset --hard HEAD~1          # move the branch back one commit
git log --oneline                # "fix: oops" is gone from the log
git reflog                       # ...but is still listed here
```

#### Step 5 — Clone a repo

`git clone` copies a repo, with its whole history, into a new folder. Clone the repo you pushed in
Stage 2 into a fresh folder:

```bash
cd "$(mktemp -d)"
git clone https://github.com/<user>/<repo-name>.git
cd <repo-name>
git log --oneline
```

Expected output (your hash differs):

```text
<hash> docs: add README and pin Node 24 in .nvmrc
```

#### Step 6 — Look up the difference

Read the Git documentation for [`git revert`](https://git-scm.com/docs/git-revert) and
[`git reset`](https://git-scm.com/docs/git-reset), using what you just saw, rather than guessing. Say
the difference out loud before you move on. Section 4 has a full comparison to check yourself
against.

**Why it matters:** these commands move changes between three places: the working tree, the staging
area and the commit history. Doing each once on purpose makes that model concrete. See *The working
tree, the staging area and commits* in section 4.

**Done when:**

- [ ] `git log --oneline` in your practice repo showed a `Revert "feat: add wip line"` commit.
- [ ] After the reset, `git log --oneline` no longer shows `fix: oops`, and `git reflog` still does.
- [ ] `git clone` created a `<repo-name>` folder, and `git log --oneline` inside it shows your first
      commit.
- [ ] You can explain, in your own words and without notes, the difference between `git revert` and
      `git reset`.

### Stage 4 — Start the feature branch, add `.gitignore` and `.env.example`

**Goal:** a branch for the stage's work, ignore rules proved against real paths, and an example
environment file.

#### Step 1 — Create the feature branch

Name it for the stage and the topic.

```bash
cd ~/<repo-name>
git switch -c feat/module-01/hello-node
```

#### Step 2 — Create `.gitignore`

Create `.gitignore` with the Node-project list. It also ignores `.env.test`, which the program does
not commit, and your editor and OS files.

```gitignore
# Dependencies
node_modules/

# Environment files: never commit real values, commit .env.example instead
.env
.env.test

# SQLite database files
*.db

# Generated Prisma client (generated in src/generated/prisma/, so match at any depth)
**/generated/prisma

# Build and test output
dist/
coverage/

# Editor and OS files
.DS_Store
.vscode/
```

**Important:** use `**/generated/prisma`, not `generated/prisma`. A pattern with a slash in the middle
only matches from the repo root, so it would miss the client generated in `src/generated/prisma/`.

#### Step 3 — Prove the rules work

Create throwaway files at real paths and ask Git which rule matches each:

```bash
mkdir -p modules/m01-dev-environment-git-github/src/generated/prisma node_modules/x dist coverage
touch modules/m01-dev-environment-git-github/src/generated/prisma/client.js node_modules/x/i.js dist/a.js coverage/c.json .env .env.test app.db
git check-ignore -v modules/m01-dev-environment-git-github/src/generated/prisma/client.js node_modules/x/i.js .env app.db dist/a.js coverage/c.json
```

Expected output, one line for each path, naming the rule that matched:

```text
.gitignore:12:**/generated/prisma	modules/m01-dev-environment-git-github/src/generated/prisma/client.js
.gitignore:2:node_modules/	node_modules/x/i.js
.gitignore:5:.env	.env
.gitignore:9:*.db	app.db
.gitignore:15:dist/	dist/a.js
.gitignore:16:coverage/	coverage/c.json
```

Then delete the throwaway files:

```bash
rm -rf modules/m01-dev-environment-git-github/src node_modules dist coverage .env .env.test app.db
```

#### Step 4 — Create `.env.example`

Create `.env.example` with comments only, because there is nothing to configure yet:

```text
# Copy this file to .env and fill in real values. Never commit .env.
# Nothing to configure yet. This file is here so the habit is in place before the first variable.
```

Check that it is not ignored. This command should print nothing:

```bash
git check-ignore -v .env.example
```

#### Step 5 — Commit the two files separately

```bash
git add .gitignore
git commit -m "chore: add .gitignore for Node projects"
git add .env.example
git commit -m "chore: add .env.example"
```

**Why it matters:** `git check-ignore -v` names the rule that matched, which proves an ignore instead
of assuming it. `.env.example` holds no values, so there is nothing secret to leak, and it builds the
habit before the first real variable arrives. Never commit a real `.env`. One commit per concern
lets a reviewer read each change on its own.

**Done when:**

- [ ] `git check-ignore -v` named a rule for a file under `node_modules/`, `.env`, a `.db` file, a
      file under `src/generated/prisma/`, a file under `dist/`, and a file under `coverage/`.
- [ ] `git check-ignore -v .env.example` printed nothing.
- [ ] `.env.example` exists and contains no real values.
- [ ] `git log --oneline` shows both commits on `feat/module-01/hello-node`, each a Conventional
      Commit.

### Stage 5 — Write the `hello-node` script

**Goal:** create and run the stage's deliverable script on your feature branch, not on `main`.

#### Step 1 — Create the file

Create `modules/m01-dev-environment-git-github/hello-node.js`. It is plain JavaScript:

```js
// Module 1 output: hello-node script (M01 — Dev Environment, Git & GitHub)
const greeting = "Hello, Node.js!";

console.log(greeting);
console.log(`Running on Node ${process.version}`);
```

#### Step 2 — Run it

```bash
node modules/m01-dev-environment-git-github/hello-node.js
```

Expected output:

```text
Hello, Node.js!
Running on Node v24.x.x
```

The exact patch version varies.

#### Step 3 — Commit and push the branch

```bash
git add modules/m01-dev-environment-git-github/hello-node.js
git commit -m "feat: add hello-node script"
git push -u origin feat/module-01/hello-node
```

**Important:** keep the greeting on its own line. Stage 6 edits that line on two branches.

**Why it matters:** `process.version` prints the Node version running the script, so it shows that
your `.nvmrc` took effect. `feat` is the commit type because the commit adds something new.

**Done when:**

- [ ] The script prints the greeting, then `Running on Node v24.…`.
- [ ] `git branch --show-current` prints `feat/module-01/hello-node`.
- [ ] The branch is on GitHub, and `git log --oneline` shows `feat: add hello-node script`.

### Stage 6 — Create a merge conflict and resolve it

**Goal:** a real conflict between two of your own branches, with its resolution in the branch history.

#### Step 1 — Branch twice from the same commit

Change the greeting line differently on each branch. Make the edits in your editor.

```bash
git switch -c feat/module-01/hello-node-greeting feat/module-01/hello-node
# edit the greeting line to: const greeting = "Hello from Node.js!";
git commit -am "feat: reword greeting"

git switch -c feat/module-01/hello-node-welcome feat/module-01/hello-node
# edit the same line to: const greeting = "Welcome to Node.js!";
git commit -am "feat: change greeting to a welcome message"
```

#### Step 2 — Merge both into the feature branch

Pass `-m` so the merge commit follows Conventional Commits:

```bash
git switch feat/module-01/hello-node
git merge --no-ff -m "chore: merge greeting branch" feat/module-01/hello-node-greeting
git merge --no-ff feat/module-01/hello-node-welcome
```

The first merge succeeds. The second stops with:

```text
Auto-merging modules/m01-dev-environment-git-github/hello-node.js
CONFLICT (content): Merge conflict in modules/m01-dev-environment-git-github/hello-node.js
Automatic merge failed; fix conflicts and then commit the result.
```

`git status --short` lists the file as `UU`, meaning both sides modified it.

**Important:** decide your merge commit messages before you merge, including for the clean merge.
Nothing is pushed yet, but once you push, those commits stay in the history.

#### Step 3 — Resolve the conflict

Open the file. Your current branch is above `=======` and the incoming branch is below it:

```text
<<<<<<< HEAD
const greeting = "Hello from Node.js!";
=======
const greeting = "Welcome to Node.js!";
>>>>>>> feat/module-01/hello-node-welcome
```

Keep one version, or write a new line, and delete the three marker lines. Run the script to confirm
it works.

#### Step 4 — Finish the merge and push

```bash
git add modules/m01-dev-environment-git-github/hello-node.js
git commit -m "chore: resolve greeting conflict between greeting and welcome branches"
git push origin feat/module-01/hello-node
```

**Why it matters:** the first merge succeeds because only one side has changed that line so far. The
second cannot, because both sides changed it and Git will not guess which you meant. Resolving it is
your decision, and the commit records it. `--no-ff` forces a merge commit even when Git could skip
it, so the history shows that two lines of work joined. Git's default merge message
(`Merge branch ...`) is not a Conventional Commit, which is why you pass your own. See *Merge
versus rebase* in section 4.

**Done when:**

- [ ] `hello-node.js` contains none of the lines `<<<<<<<`, `=======` or `>>>>>>>`, and the script
      runs.
- [ ] `git log --graph --oneline` shows the two branches rejoining at your resolution commit.
- [ ] `git status` reports a clean working tree.

### Stage 7 — Open the pull request and merge it

**Goal:** the merged pull request this stage requires.

#### Step 1 — Open the pull request

```bash
gh pr create --base main --head feat/module-01/hello-node \
  --title "Add hello-node script, .gitignore and .env.example" \
  --body "<a Summary, a list of Changes, and a Test plan>"
```

Write the Test plan from checks you actually ran, such as running the script or
`git check-ignore`. The PR flow follows [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow).

**Important:** you can open the PR before or after you create the conflict. A PR follows its branch,
so pushing more commits to the branch adds them to the PR either way.

#### Step 2 — Check the commits

On the PR's page, confirm it shows **3 or more Conventional Commits**.

#### Step 3 — Merge it on GitHub

Use the **Create a merge commit** option.

**Important:** do not squash and do not rebase. A squash merge collapses your commits into one,
which erases the resolved conflict and the commit messages this stage checks.

#### Step 4 — Update your local `main` and look at the history

```bash
git switch main
git pull --ff-only origin main
git log --graph --oneline
```

**Why it matters:** GitHub Flow is: branch, commit, open a PR, review, merge, and never push to `main`
directly. GitHub writes its own merge commit message (`Merge pull request #… from …`), which is not
a Conventional Commit. Your own commits inside the PR are.

**Done when:**

- [ ] `gh pr view feat/module-01/hello-node` reports the PR as `MERGED`. Name the branch, because a
      bare `gh pr view` on `main` finds no PR.
- [ ] The PR carries 3 or more Conventional Commits.
- [ ] `git log --graph --oneline` on `main` shows the resolved conflict in its history.
- [ ] `.gitignore`, `.env.example` and `modules/m01-dev-environment-git-github/hello-node.js` are on
      `main`.

## 4. Concepts you need

Dip into these when a stage points to one, and read them all after Stage 7. They explain what the
steps mean. They do not repeat them.

### The working tree, the staging area and commits

Git keeps your work in three places, and every command in Stage 3 moves a change between them.

| Place | What it holds |
| --- | --- |
| Working tree | The files on disk, where you edit |
| Staging area | The changes the next commit will contain |
| Commit history | The saved snapshots |

A change moves in one direction: edit, then `git add`, then `git commit`. The middle step is the
useful one, because it lets you choose what goes into a commit, so two files edited together can
become two separate commits.

Each command looks at a different pair of places:

| Command | What it compares or moves |
| --- | --- |
| `git diff` | Working tree against staging area: edits you have not staged |
| `git diff --staged` | Staging area against the last commit: what the next commit would hold |
| `git restore --staged <file>` | Staging area back to the last commit; your edit stays in the file |
| `git restore <file>` | Working tree back to the staging area; the edit is gone |

Treat `git restore <file>` as permanent. The edit was never in a commit, so Git has no copy of it to
give back. This three-places model is also why `reset --hard`, covered next, is risky: it changes all
three at once.

### `git revert` versus `git reset`

Both "undo", but they undo different things. Revert works on the history by adding to it. Reset works
on where a branch points, and with `--hard`, on the other two places as well.

| | `git revert <commit>` | `git reset --hard <commit>` |
| --- | --- | --- |
| What it does | Adds a new commit that applies the opposite change | Moves the branch back and makes the files match |
| History afterwards | Longer. The original commit stays | Shorter. Later commits leave the log |
| Safe on pushed commits | Yes, nobody's history changes | No, it rewrites history others may have |
| Recoverable | Trivially, it is just another commit | Through `git reflog` for a while, not for ever |

A mental model: revert is a correction written in the ledger, and reset is tearing out the page. Use
revert on anything shared. Use reset only on commits that exist on your machine alone. This guide
only uses `--hard`. `reset` also has softer forms that keep your changes, which are worth reading
about.

### Merge versus rebase

Both bring one branch's work into another, and they trade history shape against honesty.

- **Merge** joins the branches and records the join in one extra commit. The history shows what
  really happened, including that two lines of work ran in parallel. Nothing existing is changed.
- **Rebase** takes your commits and replays them on top of another branch. The history becomes a
  straight line, but the replayed commits are new commits with new hashes, so it rewrites history. A
  conflict shows up while each commit is replayed, and you resolve it commit by commit.

Rule of thumb: rebase only commits nobody else has. Merge when the branch is shared, or when you want
the record of the join. At this stage rebase is conceptual only. You do not need to run one.

## 5. Common pitfalls

- **Your terminal starts on the wrong Node.** Having Node 24 installed does not mean a new terminal
  uses it, because your default version can point at an older one. Run `nvm use` inside the repo and
  confirm with `node --version`. Do not assume.
- **You can't open a PR on an empty repo.** A brand-new GitHub repo has no branch, so there is nothing
  to merge into. Push a first commit to `main` before you branch.
- **Git's default merge message is not a Conventional Commit.** `git merge` writes
  `Merge branch '...'` unless you pass `-m`. Decide your merge commit messages before you merge,
  including for the clean merge.
- **GitHub's merge commit is not conventional either.** `Merge pull request #… from …` is written by
  GitHub, and you cannot change it from the command line. That is expected. What matters is that the
  commits inside the PR are conventional.
- **A wrong ignore pattern fails silently.** `generated/prisma` matches from the repo root only. Use
  `**/generated/prisma`, and prove it with `git check-ignore -v` on a real nested path.
- **Don't commit a real `.env`.** Commit `.env.example` instead, and keep real values out of
  examples.

## 6. Check yourself against the definition of done

- [ ] A **merged pull request** on your public repo, with **3 or more Conventional Commits**.
- [ ] The merged history contains a **resolved merge conflict**.
- [ ] A `hello-node` script is in the repo and runs on Node 24.
- [ ] `.gitignore` is present and covers `node_modules`, `.env`, `*.db`, the generated Prisma client,
      `dist` and `coverage`.
- [ ] `.env.example` is present, even though there is nothing to configure yet. This is about the
      habit.
- [ ] You can narrate, in your own words and without notes, the difference between `git revert` and
      `git reset`.

## 7. Next step

If every box in section 6 is ticked, this stage is done.

1. Keep this repo. Every later stage adds its own folder under `modules/`, for example
   `modules/m02-modern-javascript-async-await/`.
2. Carry the workflow forward. For each stage, create a branch named `feat/module-NN/<topic>`,
   commit with Conventional Commits, and open a pull request. Never push to `main` directly.
3. Move on to **M02 — Modern JavaScript & Async/Await** (Week 1 · Days 2-4). It builds on this stage:
   you will use the same branch → PR → merge habit for its checkpoint.
