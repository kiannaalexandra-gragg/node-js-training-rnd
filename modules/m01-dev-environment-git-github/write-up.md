# M01 Write-Up — Dev Environment, Git & GitHub

> Written from what I captured as I built it. Written so a trainee starting this stage next cohort
> could follow my path on their own.

## What I built

- **A public repo**, [`node-js-foundation-training`](https://github.com/kiannaalexandra-gragg/node-js-foundation-training), for all my stage builds. Each stage gets its own folder under `modules/`, named like its folder in `node-js-training-rnd`. The root `README.md` says how the two repos relate: that one holds the briefs and my write-ups, this one holds the code I build.
- **A `hello-node` script** at `modules/m01-dev-environment-git-github/hello-node.js`. It prints a greeting and `Running on Node <version>`, and it opens with a comment labelling it as Module 1 output.
  - **To run it:** from the repo root, switch to the pinned Node and run the file. On the merged `main` it prints `Welcome to Node.js!` and then `Running on Node v24.22.0`.
    ```bash
    nvm use
    node modules/m01-dev-environment-git-github/hello-node.js
    ```
- **Repo hygiene files:** `.nvmrc` pinning Node 24, a `.gitignore` for a Node project, and an `.env.example` with comments only (there is nothing to configure yet).
- **A merged pull request:** https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/1. It carries five non-merge Conventional Commits and two merge commits, and one of the merges resolved a real conflict.

Toolchain I confirmed on macOS: Git 2.54.0, nvm 0.40.5 with Node 24.22.0 LTS installed, VS Code 1.142.0, GitHub CLI 2.96.0 (logged in over HTTPS), and Bruno, which was already in `/Applications`. I did not open Bruno in this stage. For terminal basics I used zsh on macOS, and the whole stage needed only `cd`, `ls`, `mkdir -p`, `cat`, `touch`, `rm`, and `printf` or `echo` with `>` and `>>` to write to files.

## Why it's built this way (key decisions)

**Branches and commits.**
- `main` got one commit (README and `.nvmrc`) so the pull request would have a base to merge into. A brand-new GitHub repo has no branch at all, so this was the only commit I pushed directly to `main`.
- Everything else went on `feat/module-01/hello-node`, named for the stage and the topic.
- I made one commit per concern: `chore: add .gitignore for Node projects`, `chore: add .env.example`, `feat: add hello-node script`. A reviewer can read each commit on its own, and the type says what kind of change it is.
- For the conflict I branched twice from the same commit: `feat/module-01/hello-node-greeting` and `feat/module-01/hello-node-welcome`. Both were named after the feature branch so they sort next to it.
- I opened the PR after resolving the conflict, not before. The brief lists the PR first, but the PR only needs the branch, and pushing more commits to that branch adds them to the PR.

**Public repo versus private with a no-direct-push rule.**
- The brief says branch protection on private repos depends on the GitHub plan. A public repo avoids that, so branch protection stays available on a free account.
- I did not switch branch protection on. My branch-then-PR flow was my own discipline, and my one push to `main` was the initial commit.
- On a private repo without a plan that offers protection, nothing would technically block a push to `main`. The rule would exist only as an agreement, and the PR review would be the only gate. I have not tried a private repo, so this is reasoning from the brief, not something I observed.

**How I resolved the conflict.**
- Both branches changed line 2 of `hello-node.js`. One made it `Hello from Node.js!`, the other `Welcome to Node.js!`.
- I merged the greeting branch into the feature branch first. That merged cleanly.
- Merging the welcome branch stopped with `CONFLICT (content)`, and `git status` listed the file as `UU`.
- The file held conflict markers, with my feature branch's line (`HEAD`) above the `=======` and the welcome branch's line below it.
- I kept the welcome wording, deleted the three marker lines, ran the script, checked no markers were left, then committed with `chore: resolve greeting conflict between greeting and welcome branches`.
- A rebase would have replayed my commit on top of the other branch and stopped at the same line. The conflict would be the same, but the result would be a straight line of commits with no merge commit recording the resolution. I did not run a rebase, as the brief treats it as conceptual at this stage.

## How to build it (teach it to the next trainee)

Everything below is the order I did it in. `<user>` stands for your GitHub username.

**1. Check the toolchain.**
- *Goal:* know what is installed and which Node you are on.
- *Do:*
  ```bash
  git --version
  node --version
  nvm --version
  code --version
  gh --version
  gh auth status
  ```
  Also check that Bruno is installed (on macOS, look in `/Applications`).

  Then confirm the terminal basics you will lean on for the rest of the stage:
  ```bash
  echo $SHELL                 # which shell you are in
  ls -a ~/node-js-foundation-training   # list files, including hidden ones like .git and .nvmrc
  cd ~/node-js-foundation-training      # move into a folder
  mkdir -p modules/m01-dev-environment-git-github   # make a folder, and any parents
  cat .nvmrc                  # print a file
  printf '24\n' > .nvmrc      # write a file (> replaces, >> appends)
  touch scratch.txt && rm scratch.txt   # create an empty file, then delete it
  ```
- *Why:* every later step assumes these work. Checking first means a failure later is not a setup problem. The terminal commands matter because everything else in this stage, from Git to running the script, is typed there.
- *Result:* `node --version` printed v22.23.0, not 24. nvm had Node 24.22.0 installed, but its default alias pointed at 22. `gh auth status` showed me logged in. `echo $SHELL` printed `/bin/zsh`.

**2. Pin and use Node 24.**
- *Goal:* the repo says which Node it needs, and your shell uses it.
- *Do:* `nvm ls` showed v24.22.0 already installed, so I did not need `nvm install 24`. In the repo I wrote the version file and switched:
  ```bash
  printf '24\n' > .nvmrc
  nvm use
  node --version
  ```
- *Why:* `.nvmrc` is the single place that records the version, so everyone on the cohort runs the same Node. `nvm use` with no argument reads that file.
- *Result:* `node --version` printed v24.22.0 inside the repo.

**3. Create the repo and connect the remote.**
- *Goal:* an empty local repo wired to an empty public GitHub repo.
- *Do:*
  ```bash
  mkdir -p ~/node-js-foundation-training/modules/m01-dev-environment-git-github
  cd ~/node-js-foundation-training
  git init -b main
  gh repo create node-js-foundation-training --public --description "Trainee builds for the Node.js Developer Training Program (M01-M10)"
  git remote add origin https://github.com/<user>/node-js-foundation-training.git
  ```
- *Why:* `git init -b main` names the first branch `main` so it matches GitHub. `gh repo create` without a source only creates the remote, so nothing is pushed before I choose what goes in.
- *Result:* the repo URL printed, and `git remote -v` listed `origin` for fetch and push.

**4. Make the first commit on `main`.**
- *Goal:* give the repo a base branch to open PRs against.
- *Do:* write a `README.md`, then:
  ```bash
  git add .nvmrc README.md
  git commit -m "docs: add README and pin Node 24 in .nvmrc"
  git push -u origin main
  ```
- *Why:* a pull request merges one branch into another, so `main` has to exist on GitHub first. I staged the two files by name, not with `git add .`, so I committed exactly what I meant to.
- *Result:* `main` appeared on GitHub as a new branch.

**5. Practise the core commands in a throwaway repo.**
- *Goal:* see each command change the working tree, the staging area or the history.
- *Do:* in a scratch folder outside the project:
  ```bash
  git init -b main
  echo "line one" > notes.txt
  git status --short          # ?? notes.txt
  git add notes.txt
  git status --short          # A  notes.txt
  git commit -m "chore: add notes"
  echo "line two" >> notes.txt
  git diff                    # unstaged change
  git add notes.txt
  git diff --staged           # same change, now staged
  git restore --staged notes.txt   # unstage, edit stays
  git restore notes.txt            # discard the edit
  echo "work in progress" >> notes.txt
  git stash                   # edit set aside, file is back to line one
  git stash pop               # edit comes back
  git commit -am "feat: add wip line"
  git revert --no-edit HEAD   # new commit that undoes the last one
  echo "oops" >> notes.txt
  git commit -am "fix: oops"
  git reset --hard HEAD~1     # branch moves back, "fix: oops" leaves the log
  git reflog                  # still lists the dropped commit
  ```
- *Why:* a scratch repo means a mistake costs nothing and the project history stays clean. Doing `revert` and `reset` next to each other shows the difference directly.
- *Result:* after `revert`, `git log --oneline` had three commits including `Revert "feat: add wip line"`. After `reset --hard HEAD~1`, `fix: oops` was gone from the log but still in `git reflog`.

**6. Start the feature branch and add `.gitignore`.**
- *Goal:* a branch for the stage's work, with ignore rules proved against real paths.
- *Do:*
  ```bash
  git switch -c feat/module-01/hello-node
  ```
  Write `.gitignore` with `node_modules/`, `.env`, `.env.test`, `*.db`, `**/generated/prisma`, `dist/`, `coverage/`, `.DS_Store` and `.vscode/`. Then create throwaway files at real paths and ask Git:
  ```bash
  mkdir -p modules/m01-dev-environment-git-github/src/generated/prisma node_modules/x dist coverage
  touch modules/m01-dev-environment-git-github/src/generated/prisma/client.js node_modules/x/i.js dist/a.js coverage/c.json .env .env.test app.db .DS_Store
  git check-ignore -v modules/m01-dev-environment-git-github/src/generated/prisma/client.js .env app.db dist/a.js
  git check-ignore -v .env.example || echo "not ignored"
  ```
  Delete the throwaway files afterwards.
- *Why:* the Prisma client is generated inside `src/generated/prisma/`, deep in the tree. A pattern with a slash in the middle only matches from the repo root, so `**/generated/prisma` is used to match at any depth. `git check-ignore -v` shows which rule matched, so the ignore is proved and not assumed.
- *Result:* every path matched a rule, and `.env.example` printed `not ignored`.

**7. Add `.env.example` and commit.**
- *Goal:* the habit of committing an example and never the real file.
- *Do:* write `.env.example` with two comment lines saying to copy it to `.env`, that nothing is configured yet, and that `.env` is never committed. Then:
  ```bash
  git add .gitignore
  git commit -m "chore: add .gitignore for Node projects"
  git add .env.example
  git commit -m "chore: add .env.example"
  ```
- *Why:* it holds no value, so there is nothing secret to leak, and the first real variable will have a place to be documented. Two commits keep the two concerns apart.
- *Result:* `git log --oneline` showed both commits above the README commit.

**8. Write `hello-node` and commit it.**
- *Goal:* the deliverable script on the feature branch.
- *Do:* create `modules/m01-dev-environment-git-github/hello-node.js`:
  ```js
  // Module 1 output: hello-node script (M01 — Dev Environment, Git & GitHub)
  const greeting = "Hello, Node.js!";

  console.log(greeting);
  console.log(`Running on Node ${process.version}`);
  ```
  Run it with `node modules/m01-dev-environment-git-github/hello-node.js`, then:
  ```bash
  git add modules/m01-dev-environment-git-github/hello-node.js
  git commit -m "feat: add hello-node script"
  git push -u origin feat/module-01/hello-node
  ```
- *Why:* it is plain JavaScript, as Week 1 requires, and the version line shows `.nvmrc` took effect. `feat` is the type because it adds something new.
- *Result:* the script printed `Hello, Node.js!` and `Running on Node v24.22.0`.

**9. Create the conflict and resolve it.**
- *Goal:* a real conflict between two of your own branches, with the resolution in the history.
- *Do:* branch twice from the same commit and change line 2 differently on each:
  ```bash
  git switch -c feat/module-01/hello-node-greeting feat/module-01/hello-node
  sed -i '' 's/Hello, Node.js!/Hello from Node.js!/' modules/m01-dev-environment-git-github/hello-node.js
  git commit -am "feat: reword greeting"
  git switch -c feat/module-01/hello-node-welcome feat/module-01/hello-node
  sed -i '' 's/Hello, Node.js!/Welcome to Node.js!/' modules/m01-dev-environment-git-github/hello-node.js
  git commit -am "feat: change greeting to a welcome message"
  git switch feat/module-01/hello-node
  git merge --no-ff -m "chore: merge greeting branch" feat/module-01/hello-node-greeting
  git merge --no-ff feat/module-01/hello-node-welcome
  ```
  The second merge stops with a conflict. Open the file, keep one line, delete the `<<<<<<<`, `=======` and `>>>>>>>` lines, run the script, then:
  ```bash
  git add modules/m01-dev-environment-git-github/hello-node.js
  git commit -m "chore: resolve greeting conflict between greeting and welcome branches"
  git push origin feat/module-01/hello-node
  ```
  (`sed -i ''` is the macOS form. You can make the same edit in the editor.)
- *Why:* the first merge succeeds because only one side has changed the line so far. The second cannot, because both sides changed the same line and Git will not guess. `--no-ff` forces a merge commit even when a fast-forward is possible, so the history shows that two lines of work joined. I passed `-m` so the merge commit follows Conventional Commits, because Git's default `Merge branch ...` message does not.
- *Result:* `git log --graph --oneline` showed both branches rejoining at the resolution commit.

**10. Open the PR and merge it.**
- *Goal:* the merged PR the brief requires.
- *Do:*
  ```bash
  gh pr create --base main --head feat/module-01/hello-node --title "Add hello-node script, .gitignore and .env.example" --body "<Summary, Changes and Test plan>"
  ```
  The Test plan listed only checks I had actually run. The PR showed 7 commits. I then merged it on GitHub with the **Create a merge commit** option.
- *Why:* merge-commit style, not squash, because squash would collapse the seven commits into one and erase the resolved conflict and the 3-plus Conventional Commits the brief checks. GitHub names its own merge commit `Merge pull request #1 from ...`, which is not a Conventional Commit. The PR's own commits are.
- *Result:* `gh pr view 1` reported `MERGED`, and `git log --graph` on `main` showed the branch joining at the PR's merge commit.

## Concepts worth explaining

**The working tree, the staging area and commits.**
- Git keeps my work in three places. The **working tree** is the files on disk, where I edit. The **staging area** is the list of changes the next commit will contain. The **commit history** is the saved snapshots. A change moves in one direction: edit, then `git add`, then `git commit`.
- In my scratch repo, `git status --short` showed `?? notes.txt` for the new file (untracked), `A  notes.txt` after `git add` (staged), and nothing after the commit.
- The staging step is what lets me choose what goes into a commit. When I made `.gitignore` and `.env.example`, both files existed in my working tree, and I staged and committed them one at a time to get two separate commits.
- There are two different diffs. `git diff` compares the working tree with the staging area, so it shows edits I have not staged. `git diff --staged` compares the staging area with the last commit, so it shows what the next commit would hold. After `git add`, my edit moved from the first to the second.
- Going back follows the same three places. `git restore --staged <file>` takes a change out of staging and leaves my edit in the file. `git restore <file>` throws the edit away from the working tree. I treat that as permanent, because the edit was never in a commit, though I did not test recovering it. `git stash` sets edits aside and `git stash pop` brings them back.
- This is also why `reset --hard`, explained next, is risky: it changes all three places at once.

**`git revert` versus `git reset`.**
- `git revert <commit>` adds a new commit that does the opposite of that commit. The old commit stays in history and the log grows. In my scratch repo the log went from two commits to three, with `Revert "feat: add wip line"` on top.
- `git reset` moves the current branch back to another commit. With `--hard` it also resets the staging area and the files to match. The commits after that point drop out of `git log`, although `git reflog` still lists them for a while. In my scratch repo `git reset --hard HEAD~1` removed `fix: oops` from the log.
- Revert is safe on commits that are already pushed, because nobody's history changes. Reset rewrites history, so I use it only on commits that exist on my machine alone. I only ran `--hard`. I did not run `--soft` or `--mixed`.

## What tripped me up

- **The shell's default Node was 22.** Node 24.22.0 was installed through nvm, but the default alias pointed at 22, so `node --version` printed v22.23.0 until I ran `nvm use` in the repo, where `.nvmrc` told it to use 24.
- **A new GitHub repo has no branch.** `gh repo create` leaves it empty, so there was nothing for a PR to target until I pushed a first commit to `main`. That is why my first push was directly to `main`.
- **Git's default merge message isn't a Conventional Commit.** `git merge` writes `Merge branch '...'`. I had to pass `-m` on the merges I made myself. The merge commit GitHub makes when you merge the PR (`Merge pull request #1 from ...`) is also not conventional, and I cannot change it from the command line.
- **I added a merge commit message without being asked.** I wrote `chore: merge greeting branch` for the first, clean merge before I had agreed a message for it. It was already pushed by the time I raised it, so it stays in the history.

## Checkpoint evidence

**Merged PR:** https://github.com/kiannaalexandra-gragg/node-js-foundation-training/pull/1, merged on 2026-10-19 as merge commit `ef59bd6`.

**History on `main` after the merge:**

```text
*   ef59bd6 Merge pull request #1 from kiannaalexandra-gragg/feat/module-01/hello-node
|\
| *   5c60daa chore: resolve greeting conflict between greeting and welcome branches
| |\
| | * 211192c feat: change greeting to a welcome message
| * |   ca93232 chore: merge greeting branch
| |\ \
| | |/
| |/|
| | * 5325a74 feat: reword greeting
| |/
| * 98332cb feat: add hello-node script
| * c4f5c1b chore: add .env.example
| * 0a94dc1 chore: add .gitignore for Node projects
|/
* 86b9f6d docs: add README and pin Node 24 in .nvmrc
```

**Files on `main`:** `.env.example`, `.gitignore`, `.nvmrc`, `README.md`, `modules/m01-dev-environment-git-github/hello-node.js`.

**`.gitignore`:**

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

**`.env.example`:**

```text
# Copy this file to .env and fill in real values. Never commit .env.
# Nothing to configure yet. This file is here so the habit is in place before the first variable.
```

**`revert` versus `reset`, in my own words:** revert records a new commit that undoes an old one and leaves history alone, so it is the one to use on anything already pushed. Reset moves the branch back and the later commits fall out of the log, so I only use it on local commits nobody else has.

## What I'd do differently

- Agree every merge commit message up front, including the clean merge, instead of choosing one on the spot.
- Run `nvm alias default 24` as part of setup, so a new terminal does not start on Node 22.
- Keep the capture notes open from the first command, so the "what tripped me up" list is written as things happen.
