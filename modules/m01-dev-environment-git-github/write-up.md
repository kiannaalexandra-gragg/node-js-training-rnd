# M01 Write-Up — Dev Environment, Git & GitHub

**Code repo:** [node-js-training](https://github.com/kiannaalexandra-gragg/node-js-training) · folder `m01-dev-environment-git-github/`

> Fill this in as you go, not after the fact. Write it so a trainee starting this stage next cohort
> could follow your path on their own.
>
> **Status:** Complete. Every Definition of done box in `brief.md` is satisfied. Ready for trainer review.
> Terminal output below is real, from a macOS run, and trimmed to the key lines where marked.

## What I built

- A working toolchain: VS Code, Git, Node 24 through a version manager, the GitHub CLI, and Bruno.
- A public repo, [`node-js-training`](https://github.com/kiannaalexandra-gragg/node-js-training), the single
  code repo for all modules (M01–M10), with one folder per module.
- `feat/hello-node`, a feature branch with five Conventional Commits:
  - `feat: add hello-node script`: `m01-dev-environment-git-github/hello-node.js`, which prints a
    greeting and the Node version.
  - `chore: pin Node 24 with .nvmrc`, `chore: add .gitignore for Node projects`,
    `chore: add .env.example`: the root setup files.
  - `docs: describe Node.js training repository`: the README.
- A deliberate merge conflict, created and resolved on that branch (two branches editing the same line).
- A merged pull request, [#1](https://github.com/kiannaalexandra-gragg/node-js-training/pull/1), carrying
  9 commits including the conflict resolution, merged into `main` as `0a9c616`.

## Why it's built this way (key decisions)

**One repo for every module, not one per module.** The Task API grows from M05 through M09 and goes to the
M10 capstone, so one repo with one folder per module keeps that history together, and one root `.nvmrc`
and `.gitignore` cover every module. The brief says "public training repos"; confirm this layout with your
trainer.

**HTTPS through the GitHub CLI for authentication.** The brief allows the GitHub CLI, a credential manager
or SSH. `gh auth login` is the shortest path: one browser login, no SSH key to generate and register. SSH
suits people who push from many machines, but it adds setup this module doesn't need.

**A Node version manager, not a direct install.** The cohort is pinned to Node 24 LTS and other projects
may need other versions. A version manager switches per project.

**Git practice in a throwaway repo.** Step 2 runs destructive commands (`git reset --hard`,
`git restore`). A scratch folder outside `node-js-training` means a mistake can't touch the history you're
reviewed on.

**One feature branch, five single-purpose commits, nothing straight to `main`.** `feat` marks new
behaviour (the script), `chore` marks setup files, `docs` marks the README. Small commits are easy to
review and to `revert` individually. The setup files live at the repo root, not in `m01-…`, so they apply
to every module. The `.gitignore` uses `**/` patterns so rules such as the generated Prisma client match
inside any module folder. `.env.example` documents which settings exist (`PORT`, `NODE_ENV`) with safe
placeholders, while the real `.env` is never committed.

**Public repo vs private repo with a no-direct-push rule.**

Since this was an individual activity, I created the `hello-node.js` script in my own GitHub repository
and practised using a feature branch and pull request workflow. If the repository were private, I would
need to invite collaborators and grant them access before they could view or contribute to the code.
With a no-direct-push rule on `main`, I would need to push my `feat/hello-node` branch and open a pull
request for review instead of pushing changes directly to `main`. In a team setting, this would allow
other developers to review and test changes before merging them into the main branch.

_Why the program defaults to public:_ a rule that *stops* direct pushes to `main` is GitHub
[branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches),
and whether it is available on a private repo depends on the GitHub plan (the brief says so).
A public repo avoids that dependency. On a private repo without it, "no direct pushes" would be a team
agreement, not something GitHub enforces. The day-to-day workflow is the same either way.

**How I resolved the conflict.** Both branches changed line 1 of `hello-node.js`, so Git stopped and asked.
I kept neither side word for word. I wrote one line that combined the intent of both
(`Hello, Node.js Developer!`), removed the three marker lines, ran the script, then staged and committed. I
used a **merge**, not a rebase, so the history records that two branches diverged and rejoined. What a
rebase would have changed is explained under "Merge vs rebase" in Concepts.

**Merge commit, not squash, for the PR.** The brief wants 3 or more Conventional Commits and a resolved
conflict in the history. Squashing would collapse both into one commit. A merge commit keeps them. See
"Ways to merge a pull request" in Concepts.

## How to build it (teach it to the next trainee)

Eight steps. Commands are shown for macOS/Linux (bash, zsh) and Windows (PowerShell). On Windows, **Git
Bash** (installed with Git for Windows) runs every bash command here unchanged, which is the easiest route.

### Step 1 — Set up the toolchain and the repo

Do this once. Every later module assumes it.

#### 1.1 Check what you already have

```bash
git --version
code --version
node --version
gh --version
```

Each should print a version. "command not found" (macOS/Linux) or "is not recognized" (Windows) means it
is missing. The one that must match the cohort is Node: **24.x** (see 1.3). If `code` is missing but VS
Code is installed, add its shell command to your PATH:
[Launching VS Code from the command line](https://code.visualstudio.com/docs/editor/command-line).

#### 1.2 Install what is missing

| Tool | Download |
| --- | --- |
| VS Code | [code.visualstudio.com/download](https://code.visualstudio.com/download) |
| Git | [git-scm.com/downloads](https://git-scm.com/downloads) |
| GitHub CLI (`gh`) | [cli.github.com](https://cli.github.com/) |
| Bruno | [usebruno.com/downloads](https://www.usebruno.com/downloads) |

**macOS** ([Homebrew](https://brew.sh)):

```bash
brew install git gh
brew install --cask visual-studio-code bruno
```

**Windows** ([winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/); reopen the
terminal afterwards):

```powershell
winget install --id Git.Git -e
winget install --id GitHub.cli -e
winget install --id Microsoft.VisualStudioCode -e
winget install --id Bruno.Bruno -e
```

**Linux:** use your package manager for Git (for example `sudo apt install git`) and the download pages
above for the rest, since packaging differs by distro.

Open Bruno once to confirm it launches. There is nothing to create yet; it is used from M05.

#### 1.3 Install Node 24 through a version manager

The cohort uses **Node.js 24 LTS**. Pick one manager, install it, then open a new terminal:

| Manager | Install | Notes |
| --- | --- | --- |
| [nvm](https://github.com/nvm-sh/nvm) (macOS/Linux) | its [install script](https://github.com/nvm-sh/nvm#installing-and-updating), which also adds its init lines to your shell profile | Reads `.nvmrc` when you run `nvm use`. |
| [fnm](https://github.com/Schniz/fnm) (macOS/Linux/Windows) | macOS: `brew install fnm`. Windows: `winget install Schniz.fnm` | Add its init line to your shell profile first (see below); `fnm use` reads `.nvmrc`. |
| [nvm-windows](https://github.com/coreybutler/nvm-windows) (Windows) | `winget install CoreyButler.NVMforWindows`, or its installer | A separate project from nvm. It **does not read `.nvmrc`**, so run `nvm use 24` yourself. Its README says to run it from an Administrator shell, and an existing separate Node install can conflict with it. |

**fnm shell setup** (once; reload the shell afterwards). Add this line to `~/.zshrc` (zsh), `~/.bashrc`
(bash) or your PowerShell `$PROFILE`:

```bash
eval "$(fnm env --use-on-cd --shell zsh)"                          # zsh (use --shell bash for bash)
fnm env --use-on-cd --shell powershell | Out-String | Invoke-Expression   # PowerShell
```

Then install and select Node 24:

```bash
nvm install 24          # nvm, or nvm-windows
nvm use 24
# or with fnm:
fnm install 24
fnm use 24
fnm default 24          # default for new terminals

node --version          # expect v24.x.x
```

Real output from a machine that started on Node 22 (nvm on macOS):

```
$ node --version
v22.23.0
$ nvm use v24
Now using node v24.21.0 (npm v11.19.0)
```

With nvm and fnm, `use` only affects the current terminal. If a new terminal is back on the old version, set a
default (`nvm alias default 24` or `fnm default 24`). nvm-windows differs: its README documents no `alias` command, and says `nvm use` updates every open window and persists across reboots. Downloads:
[nodejs.org/en/download](https://nodejs.org/en/download).

> **Untested on Windows and Linux.** The author ran this module on macOS with nvm. The fnm and nvm-windows
> install and shell-setup steps come from those projects' READMEs, not from a run, so check them against the
> README if something doesn't match.

#### 1.4 Terminal basics

| Goal | macOS / Linux | Windows PowerShell |
| --- | --- | --- |
| Where am I? (`pwd` = print working directory) | `pwd` | `pwd` |
| List files, including hidden | `ls -la` (`-l` long form, `-a` hidden files) | `ls -Force` |
| Make a folder | `mkdir -p ~/scratch` (`-p` creates parents, no error if it exists) | `mkdir ~/scratch` |
| Change folder (`~` is home) | `cd ~/scratch` | `cd ~/scratch` |
| Write text to a file | `echo "hi" > a.txt` (replaces the file) | `echo "hi" > a.txt` |
| Show a file | `cat a.txt` | `cat a.txt` |

`cmd1 && cmd2` runs the second command only if the first succeeds. It works in bash/zsh and PowerShell 7+,
but not Windows PowerShell 5.1, where you run them on separate lines.

#### 1.5 Git identity

```bash
git config --global user.name
git config --global user.email
```

If either prints nothing, set them (same on every OS):

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

Use the email tied to your GitHub account, or GitHub's private `noreply` address (Settings → Emails),
otherwise commits won't link to your profile. `init.defaultBranch main` makes new repos start on `main`.
See [First-time Git setup](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup).

#### 1.6 Authenticate with the GitHub CLI

```bash
gh auth login
```

Choose **GitHub.com → HTTPS → Yes (authenticate Git) → Login with a web browser**. The CLI prints a
one-time code and opens `https://github.com/login/device`; paste the code and approve. Then:

```bash
gh auth status
```

```
github.com
  ✓ Logged in to github.com account <your-username> (keyring)
  - Git operations protocol: https
  - Token scopes: 'gist', 'read:org', 'repo', 'workflow'
```

You need the `repo` scope. Treat the token like a password and never paste it into a write-up, issue or
chat. On Windows, Git Credential Manager (bundled with Git for Windows) and SSH are valid alternatives; see
[GitHub's authentication docs](https://docs.github.com/en/authentication) and
[`gh auth login`](https://cli.github.com/manual/gh_auth_login).

#### 1.7 Create the public repo and clone it

```bash
cd ~
gh repo create node-js-training --public --add-readme --clone --description "Node.js Developer Training: Code and hands-on exercises (M01-M10)"
cd node-js-training
git log --oneline
git remote -v
```

`--public` follows the program default, `--add-readme` makes the first commit so `main` exists, and
`--clone` clones it into a new `node-js-training` folder. Keep the command on one line in PowerShell, where
the `\` line continuation doesn't work.

```
✓ Created repository kiannaalexandra-gragg/node-js-training on github.com
3355772 (HEAD -> main, origin/main, origin/HEAD) Initial commit
origin  https://github.com/kiannaalexandra-gragg/node-js-training.git (fetch)
origin  https://github.com/kiannaalexandra-gragg/node-js-training.git (push)
```

Check: one commit on `main`, and `origin` points at your repo. Your username and hash will differ.
See [`gh repo create`](https://cli.github.com/manual/gh_repo_create).

### Step 2 — Practise the core Git commands in a scratch repo

Goal: see what each command does to the three places Git keeps your work (working tree, staging area,
commit history), and the difference between `revert` and `reset`. Use a throwaway folder.

**PowerShell notes** (skip if you use Git Bash): in Windows PowerShell 5.1, `>` and `>>` write UTF-16,
which makes `git diff` look broken. Use `Set-Content notes.txt "line 1"` to create and
`Add-Content notes.txt "line 2"` to append; PowerShell 7+ is fine. A `LF will be replaced by CRLF` warning
is harmless ([line-ending docs](https://docs.github.com/en/get-started/git-basics/configuring-git-to-handle-line-endings)).
If output opens in a pager (`:` at the bottom), press `q`.

#### 2.1 Create the repo and make a commit

```bash
mkdir -p ~/scratch/git-playground && cd ~/scratch/git-playground
git init
echo "line 1" > notes.txt
git status                 # run after each step below
git add notes.txt
git status
git commit -m "feat: add notes"
git log --oneline
```

Output, trimmed to the key lines:

```
$ git status        # after creating notes.txt
Untracked files:
    notes.txt
$ git status        # after git add
Changes to be committed:
    new file:   notes.txt
$ git commit -m "feat: add notes"
[main (root-commit) 1fdc517] feat: add notes
```

The file moves from **untracked** to **staged** to **committed**. Your hash will differ. See
[Recording changes](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository).

#### 2.2 `diff` and `restore`

```bash
echo "line 2" >> notes.txt
git diff                         # unstaged change
git restore notes.txt            # discard it
echo "line 2" >> notes.txt
git add notes.txt
git diff --staged                # staged change (same output as git diff above)
git restore --staged notes.txt   # unstage; the edit stays in the file
git restore notes.txt            # discard it
```

```
@@ -1 +1,2 @@
 line 1
+line 2
```

`+` lines were added, `-` lines removed. `git restore <file>` **can't be undone**, because the edit was
never committed. See [`git restore`](https://git-scm.com/docs/git-restore) and
[Undoing things](https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things).

#### 2.3 `stash`

```bash
echo "work in progress" >> notes.txt
git stash push -m "wip"          # working tree is clean again
git stash list                   # stash@{0}: On main: wip
git stash pop                    # change comes back, unstaged
git restore notes.txt
```

A stash sets uncommitted work aside, for example to switch branches in a hurry. See
[`git stash`](https://git-scm.com/docs/git-stash).

#### 2.4 `revert` vs `reset`

Make three commits so there is something to undo:

```bash
echo "a" >> notes.txt && git commit -am "chore: commit A"
echo "b" >> notes.txt && git commit -am "chore: commit B"
echo "c" >> notes.txt && git commit -am "chore: commit C"
```

**Revert commit C** adds a new commit that undoes it:

```bash
git revert HEAD --no-edit
git log --oneline
```

```
84aaf83 (HEAD -> main) Revert "chore: commit C"
458a64b chore: commit C
02641b1 chore: commit B
3895722 chore: commit A
1fdc517 feat: add notes
```

**Reset two commits back** moves the branch pointer:

```bash
git reset --hard HEAD~2
git log --oneline
git reflog
```

```
HEAD is now at 02641b1 chore: commit B

02641b1 (HEAD -> main) chore: commit B
3895722 chore: commit A
1fdc517 feat: add notes

02641b1 (HEAD -> main) HEAD@{0}: reset: moving to HEAD~2
84aaf83 HEAD@{1}: revert: Revert "chore: commit C"
458a64b HEAD@{2}: commit: chore: commit C
...
1fdc517 HEAD@{5}: reset: moving to HEAD
```

`git log` now ends at commit B, but `git reflog` still lists the revert and C, so
`git reset --hard HEAD@{2}` would bring C back (not run in this lab; try it). The `HEAD@{5}` line is
probably from the earlier `git stash push`, which resets the working tree internally. References:
[`git revert`](https://git-scm.com/docs/git-revert), [`git reset`](https://git-scm.com/docs/git-reset),
[`git reflog`](https://git-scm.com/docs/git-reflog), [Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified).
`git clone <url>` was already used in Step 1.7 through `gh repo create --clone`.

### Step 3 — Feature branch: `hello-node`, `.nvmrc`, `.gitignore`, `.env.example`, README

Goal: five Conventional Commits on a feature branch, never on `main`. Run from the root of your cloned repo.

**PowerShell equivalents** for the commands below (untested by the author; Git Bash needs none):

| Bash / zsh | PowerShell |
| --- | --- |
| `echo "24" > .nvmrc` | `Set-Content -Path .nvmrc -Value "24" -Encoding ascii` (plain `>` writes UTF-16 in 5.1, which version managers can fail to read; or create the file in VS Code) |
| `touch test.db` | `New-Item test.db -ItemType File` ([docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item)) |
| `mkdir node_modules && touch node_modules/x.js` | `mkdir node_modules`, then `New-Item node_modules/x.js -ItemType File` (`&&` isn't available in PowerShell 5.1) |
| `rm -rf .env node_modules test.db` | `Remove-Item -Recurse -Force .env, node_modules, test.db` ([docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/remove-item)) |
| `printf 'PORT=3000\nNODE_ENV=development\n' > .env.example` | create it in VS Code, or `Set-Content .env.example "PORT=3000","NODE_ENV=development"` ([docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/set-content)) |
| `nvm use` (reads `.nvmrc`) | nvm-windows: `nvm use 24`. fnm: `fnm use`. |

#### 3.1 Create the branch

```bash
git checkout -b feat/hello-node     # or: git switch -c feat/hello-node
git branch                          # * marks the current branch
```

The `feat/` prefix is a naming convention for what the branch holds. See [`git switch`](https://git-scm.com/docs/git-switch).

#### 3.2 Commit 1: `hello-node`

```bash
mkdir m01-dev-environment-git-github
code m01-dev-environment-git-github/hello-node.js
```

Plain JavaScript:

```js
const greeting = "Hello, Node.js!";

console.log(greeting);
console.log(`Running on Node ${process.version}`);
```

```bash
node m01-dev-environment-git-github/hello-node.js
git add m01-dev-environment-git-github/hello-node.js
git commit -m "feat: add hello-node script"
```

```
Hello, Node.js!
Running on Node v22.23.0
[feat/hello-node 7eda45a] feat: add hello-node script
```

The version shown is whichever Node is active in your terminal. Here it was still 22; see 3.3.

#### 3.3 Commit 2: pin Node with `.nvmrc`

```bash
echo "24" > .nvmrc
git add .nvmrc
git commit -m "chore: pin Node 24 with .nvmrc"
```

`.nvmrc` records the version the project expects. **It doesn't switch your terminal.** Running the script
again still printed `Running on Node v22.23.0`. `nvm use` with no arguments reads the file:

```
$ nvm use
Found '/Users/<you>/node-js-training/.nvmrc' with version <24>
Now using node v24.21.0 (npm v11.19.0)
$ node m01-dev-environment-git-github/hello-node.js
Running on Node v24.21.0
```

See the [nvm `.nvmrc` docs](https://github.com/nvm-sh/nvm#nvmrc). Windows: see the table above.

#### 3.4 Commit 3: `.gitignore`

```bash
code .gitignore
```

```
# dependencies
node_modules/

# secrets
.env
.env.*
!.env.example

# databases
*.db
*.db-journal

# generated Prisma client (any module)
**/src/generated/prisma/

# build and test output
dist/
coverage/

# macOS
.DS_Store
```

| Rule | Why |
| --- | --- |
| `node_modules/` | Installed dependencies; large and rebuilt with `npm install`. |
| `.env`, `.env.*` | Real settings and secrets. Never commit them. |
| `!.env.example` | The `!` re-includes the example file, which `.env.*` would otherwise hide. |
| `*.db`, `*.db-journal` | SQLite files and their journals (M06): data, not source. |
| `**/src/generated/prisma/` | The generated Prisma client (M07); `**/` matches inside any module folder. |
| `dist/`, `coverage/` | Regenerated build and coverage output. |
| `.DS_Store` | macOS Finder file. Not on the brief's list; added because it appears constantly on a Mac. |

Compare with GitHub's [`Node.gitignore`](https://github.com/github/gitignore/blob/main/Node.gitignore);
pattern syntax is in the [gitignore docs](https://git-scm.com/docs/gitignore).

**Prove it works before you commit it:** create files that should be hidden and check Git can't see them.

```bash
echo "SECRET=1" > .env
mkdir node_modules && touch node_modules/x.js
touch test.db
git status
git check-ignore -v .env node_modules/x.js test.db
rm -rf .env node_modules test.db
```

```
$ git status
Untracked files:
        .gitignore
$ git check-ignore -v .env node_modules/x.js test.db
.gitignore:5:.env       .env
.gitignore:2:node_modules/      node_modules/x.js
.gitignore:10:*.db      test.db
```

`git status` lists only `.gitignore` itself, so the three files are hidden. `git check-ignore -v` shows
which rule hid each one (`file:line:pattern`, then the path), which is how to debug a file that isn't being
ignored ([docs](https://git-scm.com/docs/git-check-ignore)). Then commit:

```bash
git add .gitignore
git commit -m "chore: add .gitignore for Node projects"
```

> `.gitignore` only affects untracked files. Adding an already-committed file to it won't remove it from
> Git; you'd need `git rm --cached <file>`. So create `.gitignore` **before** the first `git add .`, and
> treat any committed secret as leaked even after deleting it. Here it was committed third; it should have
> been first (see "What I'd do differently").

#### 3.5 Commit 4: `.env.example`

```bash
printf 'PORT=3000\nNODE_ENV=development\n' > .env.example
git add .env.example
git commit -m "chore: add .env.example"
```

The file holds `PORT=3000` and `NODE_ENV=development`. Nothing needs configuring yet. The habit is that
whoever clones the repo copies it to `.env` and fills in real values, and `.env` stays out of Git.

#### 3.6 Commit 5: README

```bash
code README.md
git add README.md
git commit -m "docs: describe Node.js training repository"
```

The README says what the repo is (the code for the training program) and lists the prerequisites (Node.js
24 LTS and Git).

```
[feat/hello-node 31b3113] docs: describe Node.js training repository
 1 file changed, 9 insertions(+), 1 deletion(-)
```

The `1 deletion` is the one-line README that GitHub generated being replaced.

#### 3.7 Check the branch

```bash
git log --oneline main..feat/hello-node
```

```
31b3113 (HEAD -> feat/hello-node) docs: describe Node.js training repository
3419d6d chore: add .env.example
90644c2 chore: add .gitignore for Node projects
c4bf719 chore: pin Node 24 with .nvmrc
7eda45a feat: add hello-node script
```

`main..feat/hello-node` means commits on the branch that aren't on `main`. All five follow
[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/): `type: description`, where `type`
is `feat` (new behaviour), `chore` (maintenance and setup) or `docs` (documentation only).

### Step 4 — Push the branch and open the pull request

Goal: a pull request (PR) from `feat/hello-node` into `main`, opened but **not merged**. The Step 6
conflict gets resolved on this branch, so it ends up in this PR. Read
[GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow) first; it is short.

```bash
git push -u origin feat/hello-node
gh pr create --base main --head feat/hello-node --title "feat: add hello-node script and Node 24 project setup" --body "Adds hello-node, .nvmrc, .gitignore and .env.example."
gh pr view --web
```

- `push -u` links your local branch to `origin/feat/hello-node`; do it once per new branch.
- `--base` is the branch the changes land on, `--head` is the branch that has them. See
  [`gh pr create`](https://cli.github.com/manual/gh_pr_create).
- **Website alternative:** after pushing, click **Compare & pull request** on the repo page, check base is
  `main` and compare is `feat/hello-node`, fill in title and description, and click **Create pull request**
  ([docs](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)).

Write a useful description: what changed, why, and how you checked it. This PR used **Summary**,
**Changes** and **Test plan** headings, with a checked item for each verification (`nvm use` picks up
Node 24, the script prints the greeting, `.env` is matched by `.gitignore`).

What PR #1 showed: state open, base `main`, head `feat/hello-node`, GitHub reporting it mergeable (no
conflict yet), and all five commits on the **Commits** tab. **Files changed** shows the combined diff a
reviewer reads. Don't merge yet. This module has no reviewer, so the Step 6 conflict-resolution commits
stand in for GitHub flow's "address review comments" stage.

### Step 5 — Read about merge vs rebase (concept only)

Don't run a rebase in this module. Read
[Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
and [Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing) (reference:
[`git merge`](https://git-scm.com/docs/git-merge), [`git rebase`](https://git-scm.com/docs/git-rebase)),
then write the difference in your own words. Mine is under "Merge vs rebase" in Concepts.

### Step 6 — Create and resolve a deliberate merge conflict

Goal: two branches edit the **same line**; merge both into the PR branch and resolve the clash. Nothing
here is OS-specific.

```bash
git switch feat/hello-node
git switch -c feat/greeting-a
```

Edit line 1 of `hello-node.js` to `const greeting = "Hello, World!";`, then:

```bash
git commit -am "feat: change greeting to Hello, World"
git switch feat/hello-node          # go back so B starts from the same commit as A
git switch -c feat/greeting-b
```

Edit line 1 to `const greeting = "Hello, Developer!";`, then:

```bash
git commit -am "feat: change greeting to Hello, Developer"
git switch feat/hello-node
git merge --no-ff feat/greeting-a -m "chore: merge greeting-a into hello-node"   # clean merge
git merge feat/greeting-b                                                        # conflict
git status
```

`--no-ff` forces a real merge commit instead of a fast-forward, keeping the merge visible. `git commit -am`
works here only because `hello-node.js` is already tracked. The second merge stops:

```
Your branch is ahead of 'origin/feat/hello-node' by 2 commits.
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)
Unmerged paths:
        both modified:   m01-dev-environment-git-github/hello-node.js
```

"Both modified" means both sides changed the file; "ahead by 2" counts A's commit and the first merge
commit, which aren't pushed yet. `git merge --abort` is the way out. Open the file:

```
<<<<<<< HEAD
const greeting = "Hello, World!";
=======
const greeting = "Hello, Developer!";
>>>>>>> feat/greeting-b

console.log(greeting);
console.log(`Running on Node ${process.version}`);
```

Between `<<<<<<< HEAD` and `=======` is your **current** branch (already containing A's change); between
`=======` and `>>>>>>>` is the **incoming** change. Everything outside the markers merged cleanly.

**Resolve it:** edit the file to the content you want and delete all three marker lines. The final file:

```js
const greeting = "Hello, Node.js Developer!";

console.log(greeting);
console.log(`Running on Node ${process.version}`);
```

VS Code offers **Accept Current / Incoming / Both Changes** links above each conflict; you can also type
something else, as here ([VS Code docs](https://code.visualstudio.com/docs/sourcecontrol/overview),
[GitHub docs](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/about-merge-conflicts)).
Run the script to check it, then finish the merge and update the PR:

```bash
node m01-dev-environment-git-github/hello-node.js    # Hello, Node.js Developer!
git add m01-dev-environment-git-github/hello-node.js
git commit -m "chore: resolve greeting merge conflict"
git log --oneline --graph --all     # output below was captured here, before the push
git push
```

```
*   aa23a3a (HEAD -> feat/hello-node) chore: resolve greeting merge conflict
|\
| * e0e2cad (feat/greeting-b) feat: change greeting to Hello, Developer
* |   94c5e0e chore: merge greeting-a into hello-node
|\ \
| |/
|/|
| * 5e85255 (feat/greeting-a) feat: change greeting to Hello, World
|/
* 31b3113 (origin/feat/hello-node) docs: describe Node.js training repository
...
* 3355772 (origin/main, origin/HEAD, main) Initial commit
```

Reading it: `31b3113` is where both branches started, `5e85255` (A) and `e0e2cad` (B) sit on top of it, and
`94c5e0e` and `aa23a3a` each have two parents, so they are merge commits. `aa23a3a` joined B and holds the
resolution. After the push, PR #1 showed 9 commits and was still mergeable. Write your own merge messages
in Conventional Commits form, because Git's default (`Merge branch ...`) isn't.

### Step 7 — Merge the pull request

Goal: land PR #1 on `main` with all commits and the conflict resolution kept in the history.

On the PR page check the **Commits** tab (9 commits here), that there is no "This branch has conflicts"
banner, and **Files changed**. Merge with **Create a merge commit** (not Squash; see "Ways to merge a pull
request" in Concepts). Reference:
[About pull request merges](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/about-pull-request-merges).
This module has no reviewer, so you approve it yourself, which a real team wouldn't do.

```bash
gh pr merge 1 --merge        # or use the merge button
git switch main
git pull
git log --oneline --graph -15
```

```
*   0a9c616 (HEAD -> main, origin/main, origin/HEAD) Merge pull request #1 from kiannaalexandra-gragg/feat/hello-node
|\
| *   aa23a3a (origin/feat/hello-node, feat/hello-node) chore: resolve greeting merge conflict
...
| * 7eda45a feat: add hello-node script
|/
* 3355772 Initial commit
```

(Trimmed; the middle is the Step 6 graph, now nested under the merge.) `0a9c616` is the new merge commit on
`main`; its parents are the old `main` (`3355772`) and the PR tip (`aa23a3a`). The PR's nine commits and the
conflict stay visible. GitHub wrote the merge message itself, so that one isn't a Conventional Commit; the
nine inside the PR are. The PR page reads "Pull request successfully merged and closed".

**Optional clean-up.** GitHub flow's last step is deleting the merged branch (the **Delete branch** button,
or locally below). `-d` refuses to delete unmerged branches, so it can't lose work. Not run in this lab.

```bash
git branch -d feat/hello-node feat/greeting-a feat/greeting-b
git fetch --prune
```

### Step 8 — Verify against the Definition of done

From the root of an up-to-date `main`:

```bash
git ls-files
git ls-files | grep -E '(^|/)\.env$' || echo "no .env tracked: good"
```

PowerShell (no `grep`; prints nothing when no `.env` is tracked):

```powershell
git ls-files | Select-String -Pattern '(^|/)\.env$'
```

```
.env.example
.gitignore
.nvmrc
README.md
m01-dev-environment-git-github/hello-node.js
no .env tracked: good
```

Expect `.env.example` and no `.env`; a match would mean a secrets file was committed. Then walk every box in
"Checkpoint evidence" below, tick each box in `tasks.md`, and say the `revert` vs `reset` difference out
loud with no notes.

## Concepts worth explaining

### The working tree → staging → commit model

Git keeps your work in three places: the **working tree** (files on disk), the **staging area** (a snapshot
of what goes into the *next* commit) and the **commit history** (permanent snapshots with hashes).
`git add` copies a change from the working tree to the staging area; `git commit` turns what is staged
into a commit. `git diff` shows working tree vs staging, and `git diff --staged` shows staging vs the last
commit.

### `git revert` vs `git reset`

Both undo changes. They differ in what happens to history.

`git revert` creates a new commit that reverses the changes introduced by an earlier commit, so the
original commit and the project history remain intact. In the scratch activity, reverting commit C
created a new commit called `Revert "chore: commit C"` instead of removing commit C from the history.

`git reset` moves the current branch pointer to an earlier commit. When I ran `git reset --hard HEAD~2`,
Git moved `main` back two commits from the revert commit to commit B, removing the revert commit and
commit C from the branch's visible history. It also reset the working directory and staging area to
match commit B, discarding tracked changes that were not saved in the resulting commit. However, the
commits were still recoverable through `git reflog` at that point.

The main difference is that `revert` preserves history by adding a new commit, while `reset` rewrites
the branch's history by moving its pointer.

**When to use each on a pushed branch.** I would use `git revert` when a commit has already been pushed
to a shared branch, such as `main` or `develop`, and I need to undo its changes without rewriting the
existing history. This is safer for collaborators, because the reversal is recorded as a new commit and
can be pushed normally.

I would use `git reset` mainly for local work, such as removing commits that I have not pushed yet. I
would avoid resetting a shared branch that others may already have pulled, because it rewrites history
and can cause synchronization issues. If the commits have already been pushed, I would generally prefer
`git revert` instead.

Two details worth adding: `git reset` has three modes (`--soft` moves the pointer only, `--mixed`, the
default, also clears staging, and `--hard` also overwrites your working tree, which makes it the dangerous
one), and a reset branch that was already pushed is rejected unless you force-push, which is why resetting
shared branches hurts everyone else.

### `.gitignore` and `.env.example`

`.gitignore` lists what Git should never track: dependencies, secrets, databases, generated code and build
output. `.env` holds real settings and is ignored; `.env.example` has the same keys with placeholder values
and *is* committed so others know what to set. Rule-by-rule reasons and the `git check-ignore -v` check are
in Step 3.4.

### Merge vs rebase

**Merging** integrates the changes from one branch into another. Branching lets developers work on
separate features or fixes without affecting the main branch, and merging brings the work back together.
Teams use this workflow to keep unfinished work separate and to review and test changes before merging.
Teams prefer merging when they want to preserve the history of how branches were integrated, especially
in collaborative work.

**Rebasing** replays commits onto a different base, creating a more linear history. Teams may use it to
incorporate the latest changes from `main` into a feature branch and keep the history easier to follow.
Teams prefer rebasing when they want a cleaner, linear commit history before merging. Unlike merging,
rebasing rewrites commit history, so teams generally avoid rebasing commits already shared with others.

**What a rebase would have changed in Step 6** (explained, not run). Rebasing B onto the PR branch after A
was merged would replay B's commit on top of the new tip. The same line would clash, and Git would stop
mid-replay so you could fix the file, `git add`, and `git rebase --continue`. The result is a straight line
with **no merge commit**, and B's commit is rewritten with a new hash that already contains the fix. The
history is tidier but no longer shows that two branches diverged.

### Merge conflicts

A conflict happens when Git can't combine two changes itself, typically edits to the same lines. Git doesn't
guess: it marks the file with `<<<<<<<`, `=======` and `>>>>>>>`, lists it under "Unmerged paths" and waits.
Resolve by editing to the final content, deleting the markers, `git add`, then `git commit`;
`git merge --abort` backs out. Edits to *different* lines of the same file merge cleanly.

### Pull requests and GitHub flow

A pull request asks to merge one branch (the **head**, `feat/hello-node`) into another (the **base**,
`main`), collecting the commits and diff for review and checks before anything reaches `main`. GitHub flow is
the loop around it: create a branch → commit and push → open a PR → address review comments → merge →
delete the branch. Opening the PR asks for feedback; it doesn't mean the work is finished.

### Ways to merge a pull request

| Option | What lands on `main` | History |
| --- | --- | --- |
| Create a merge commit | Every commit from the branch, plus one merge commit | Keeps the full story, including conflict resolutions |
| Squash and merge | One new commit with all the changes | Linear, but individual commits and the conflict resolution are gone |
| Rebase and merge | Each commit replayed on top of `main` | Linear, no merge commit, but commits get new hashes |

This module uses a merge commit so the Conventional Commits and the resolved conflict stay visible.

### Knowledge check

Try to answer without looking back, then check the answers.

1. Where is a file after `git add` but before `git commit`?
2. You edited `notes.txt` but haven't staged it. What does `git restore notes.txt` do, and can you get the
   edit back?
3. A bad commit is already pushed to `main` and teammates have pulled it. Do you `revert` or `reset`, and why?
4. You ran `git reset --hard HEAD~2` and two commits vanished from `git log`. Where can you find them?
5. You committed `.nvmrc` containing `24`, but `node --version` still prints v22. Why, and what do you run?
6. What is the difference between `.env` and `.env.example`, and which one is committed?
7. You add `.env` to `.gitignore`, but `git status` still shows it as modified. What happened?
8. `git merge` stops with a conflict. What do the two halves between the markers mean, and what are the
   steps to finish (and to back out)?
9. Why do teams avoid rebasing commits that others have already pulled?
10. Which GitHub merge option would erase a conflict resolution from `main`'s history, and why?
11. Put the GitHub flow steps in order: merge, create a branch, address review comments, open a pull
    request, delete the branch, commit and push changes.

**Answers**

1. In the staging area: part of the next commit, but not committed yet.
2. It discards the unstaged edit. No, you can't get it back; it was never committed or stashed.
3. `revert`. It records the undo as a new commit that can be pushed normally. `reset` would rewrite history
   teammates already have.
4. In `git reflog`. `git reset --hard <hash-or-HEAD@{n}>` returns to one of them.
5. `.nvmrc` only records the version; it doesn't switch the terminal. Run `nvm use` (or `fnm use`); on
   nvm-windows run `nvm use 24`.
6. `.env` holds real values and is ignored. `.env.example` holds the same keys with placeholders and is
   committed as documentation.
7. The file was already tracked, and `.gitignore` doesn't affect tracked files. Run `git rm --cached .env`,
   commit, and treat any committed secret as leaked.
8. Between `<<<<<<< HEAD` and `=======` is your current branch; between `=======` and `>>>>>>>` is the
   incoming branch. Edit to the final content and delete the markers, `git add`, `git commit`.
   `git merge --abort` backs out.
9. Rebase rewrites those commits (new hashes), so teammates' copies no longer match and their branches have
   to be repaired.
10. *Squash and merge*. It combines the branch's commits into one, so the individual commits and the
    conflict-resolution commit no longer appear.
11. Create a branch → commit and push → open a pull request → address review comments → merge → delete the
    branch.

## What tripped me up

- **The wrong Node version was active.** The machine defaulted to Node 22 but the cohort needs 24, so
  `node --version` printed `v22.23.0` until `nvm use` switched it. Committing `.nvmrc` didn't fix it, because
  `.nvmrc` is only a note and a version manager only changes the current terminal. The script runs fine on
  the wrong version, so this is easy to miss. Check before anything else; a mismatch causes confusing errors
  in later modules.
- **Re-running `gh auth login` when already logged in** prints `! You were already logged in to this
  account` and still completes. Harmless, but run `gh auth status` first.
- **`git restore` can't be undone.** An edit that was never staged or committed is gone for good once
  restored. `git reset --hard` can be rescued from the reflog only for *committed* work; uncommitted changes
  it wipes are gone too. If in doubt, `git stash` first.
- **The reflog has entries you didn't type.** `reset: moving to HEAD` appeared before the revert/reset
  exercise; it most likely came from the earlier `git stash push`.
- **Ignored files vanish from `git status`**, and only `.gitignore` itself showed up. That is correct;
  `git check-ignore -v` confirms which rule is hiding what.
- **On Windows PowerShell 5.1, `>` and `>>` write UTF-16.** This can make `git diff` output look broken.
  Use `Add-Content`, PowerShell 7+ or Git Bash. Not run on Windows by the author; it comes from
  PowerShell's documented behaviour.

## Checkpoint evidence

**Completion checklist**

- [x] Git, VS Code, Node 24.x (via a version manager), the GitHub CLI and Bruno installed and verified
- [x] Terminal basics practised; Git identity set; GitHub CLI authenticated with the `repo` scope
- [x] Public repo `node-js-training` created and cloned (first commit `3355772`)
- [x] Core Git commands practised in a scratch repo (`init`, `status`, `add`, `commit`, `log`, `diff`,
      `restore`, `stash`, `revert`, `reset`, `reflog`)
- [x] `.nvmrc`, `.gitignore` and `.env.example` committed (`git ls-files` shows all three, and no `.env`)
- [x] `hello-node` on a feature branch (`feat/hello-node`) with Conventional Commits (five)
- [x] Merge conflict created and resolved (merge commit `aa23a3a`)
- [x] Pull request merged as a merge commit, not a squash

**Definition of done (from `brief.md`)**

- [x] A merged pull request with 3 or more Conventional Commits: [PR #1](https://github.com/kiannaalexandra-gragg/node-js-training/pull/1),
      merged as `0a9c616`, 9 commits.
- [x] `.gitignore` present and covering the Node-project list (Step 3.4).
- [x] `.env.example` present (Step 3.5).
- [x] I can narrate the difference between `git revert` and `git reset` in my own words (Concepts).

## What I'd do differently

- **Check the Node version before writing any code.** My machine defaulted to Node 22, so the first
  `hello-node` run printed `v22.23.0`, and I only switched with `nvm use` after committing `.nvmrc`.
  Next time I'd run `node --version` and `nvm use 24` at the very start of Step 3. The script then
  proves the right runtime from its first run, and I'd set `nvm alias default 24` so a new terminal
  doesn't quietly go back to 22.

- **Make `.gitignore` the first commit, not the third.** I committed `hello-node.js` and `.nvmrc`
  before `.gitignore`. Nothing sensitive was at risk this time, but the order contradicts my own
  advice that `.gitignore` should exist before the first `git add`. A `chore: add .gitignore` commit
  first would have been the safer habit.

- **Write the PR description to match the final PR.** I wrote it when the PR had five commits, so
  it leaves out the README commit and says nothing about the conflict resolution that later made it
  nine commits. I'd re-read and update the description after Step 6, because a reviewer reads the
  description before they read the commits.
