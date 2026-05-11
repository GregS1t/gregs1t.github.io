---
layout: post
title: "Git: a cheat sheet in questions"
date: 2021-06-13
description: "A practical cheat sheet for mastering Git, from first steps to advanced usage, organised around concrete problems."
tags: [git, development, workflow, tools]
published: false
lang: en
lang_switch:
  fr: /blog/2021/git-aide-memoire/
categories: [tutorials]
giscus_comments: false
related_posts: false
toc:
  sidebar: left
---

> 🇫🇷 [Lire ce billet en français](/blog/2021/git-aide-memoire/)

A few years ago I ran some Git/GitLab/GitHub workshops for colleagues. I decided to revisit the slide content — which had aged a bit — and reframe it differently.

The post is split into two parts. First, **common problems** — phrased the way you actually encounter them, ranging from simple to trickier. Then a few **thematic references** to go deeper on each topic. It's far from exhaustive, but hopefully you'll find what you need!

It also serves — above all — as a personal cheat sheet so I don't have to look up the same commands over and over. That means the content gets regularly updated with new finds, or corrected when needed.

---

## Common problems

### Getting started

#### I have all my work on my hard drive and I want to save it

```bash
cd my-project/
git init                                        # Initialise a local repository
git add .                                       # Stage all files
git commit -m "Initial commit"                  # First snapshot
```

To push it to a remote server (GitHub, GitLab…):

```bash
git remote add origin git@github.com:user/repo.git
git push -u origin main
```

From here on, each `git commit` + `git push` is a timestamped, versioned save.

---

#### I want to get a colleague's code

When you want to grab some code you're interested in from GitHub, or a colleague's repo, it couldn't be simpler.

```bash
git clone git@github.com:user/repo.git          # Via SSH (recommended)
git clone https://github.com/user/repo.git      # Via HTTPS
```

---

#### I don't know where I stand with my changes

```bash
git status                      # Modified, staged, and untracked files
git diff                        # Details of unstaged changes
git diff --staged               # Details of staged changes
git log --oneline -10           # The last 10 commits
```

---

### Working alone

#### Git opened Vim and I don't know how to get out

Type `:q!` to quit without saving, or `:wq` to save and quit.

To avoid this altogether:

```bash
git config --global core.editor "nano"
git config --global core.editor "code --wait"   # VS Code
```

(Vim is great, though…)

---

#### I forgot a file in my last commit

```bash
git add forgotten_file.py
git commit --amend --no-edit        # Adds the file without changing the message
```

**WARNING**: Never amend a commit that has already been pushed to a shared repository.

---

#### I want to fix the message of my last commit

```bash
git commit --amend -m "Correct new message"
```

---

#### I committed on `main` instead of my branch

```bash
git branch my-feature               # Create the branch with the commit
git reset HEAD~1 --mixed            # Undo the commit on main (changes preserved)
git switch my-feature               # Switch to the right branch
```

---

#### My history is a mess (10 commits saying "fix", "wip", "test")

Rewrite the last few commits before pushing:

```bash
git rebase -i HEAD~10
```

In the editor: `squash` to merge, `reword` to rename, `drop` to delete.

---

#### I want to undo my local changes and go back to the last commit

```bash
git restore file.py                 # Undo changes to a file (irreversible)
git restore .                       # Undo all local changes (irreversible)
git clean -fd                       # Also removes untracked files
```

---

#### I did a `reset --hard` and lost my work

The reflog keeps a history of all actions on `HEAD` for 90 days.

```bash
git reflog                          # Find the hash of the lost commit
git checkout abc1234                # Go back to it
git branch recovery abc1234        # Turn it into a branch
```

---

#### I want to set my work aside without committing

```bash
git stash                           # Push current changes onto the stash stack
git stash pop                       # Reapply the last stash
git stash list                      # List all stashes
```

---

### Working as a team

#### My push was rejected

```bash
# Typical message: "rejected [...] non-fast-forward"
git pull --rebase                   # Fetch remote changes and replay yours on top
git push
```

---

#### I want to get my colleagues' changes without breaking anything

```bash
git fetch origin                    # Fetch without touching local code
git log origin/main --oneline       # Inspect what came in
git merge origin/main               # Integrate explicitly
```

Prefer `fetch` + inspection + explicit `merge` over a blind `git pull`.

---

#### Two colleagues modified the same file

```bash
git status                          # See files in conflict
# Edit the files: resolve the <<<<, ====, >>>> markers
git add resolved_file.py
git merge --continue
# To abort everything:
git merge --abort
```

With a visual tool:

```bash
git mergetool                       # Opens the configured tool (meld, VS Code…)
```

---

#### I want to sync my fork with the original repository

```bash
git remote add upstream git@github.com:original/repo.git
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

---

#### My `push --force` overwrote a colleague's work

Always use `--force-with-lease` instead — it checks that nobody has pushed in the meantime.

```bash
git push --force-with-lease
```

---

### I pushed a sensitive file (`.env`, API key)

**Step 1 — Revoke the key immediately.** The remote already has a copy.

**Step 2 — Remove the file from the entire history:**

```bash
pip install git-filter-repo
git filter-repo --path .env --invert-paths
git push --force-with-lease
```

**Step 3 — Add the file to `.gitignore`.**

---

## My repository is in a bad state

### I want to find out who changed this line

```bash
git blame file.py
git blame -L 10,25 file.py          # Lines 10 to 25 only
```

---

### I don't know which commit introduced this bug

`git bisect` performs a binary search through history.

```bash
git bisect start
git bisect bad                      # The current commit is broken
git bisect good v1.2.0              # This tag was working
# Git checks out a middle commit → test it, then:
git bisect good                     # or: git bisect bad
# Repeat until identified. Then:
git bisect reset
```

With an automated test script:

```bash
git bisect run python tests/test_regression.py
```

---

### My merge is a disaster, I want to undo everything

During the merge (before the merge commit):

```bash
git merge --abort
```

After the merge commit:

```bash
git revert -m 1 HEAD                # Creates a commit that undoes the merge
```

---

### My repository has grown huge

Identify large objects in history:

```bash
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print $3, $4}' \
  | sort -rn \
  | head -20
```

```bash
git gc --aggressive --prune=now     # Clean up and compress

# For large binary files: Git LFS
git lfs install
git lfs track "*.fits"
git add .gitattributes
```

---

### I want to go back to a clean version without losing history

```bash
git revert abc1234                  # Undo a specific commit (creates a new commit)
git revert HEAD~3..HEAD             # Undo the last 3 commits
```

To hard-reset to a past state (destructive — only use when working alone):

```bash
git reset --hard abc1234
git push --force-with-lease
```

---

## Configuration

### Setting your identity

```bash
git config --global user.name "First Last"
git config --global user.email "you@example.com"
```

---

### Understanding configuration levels

Git applies configuration in this order (highest to lowest priority):

| Level | File | Scope |
|---|---|---|
| `--local` | `.git/config` | Current repository only |
| `--global` | `~/.gitconfig` | All repositories for the user |
| `--system` | `/etc/gitconfig` | All users on the machine |

```bash
git config --local user.email "pro@lab.fr"      # Override for this repo
git config --list --show-origin                 # See all active values
```

---

### Ignoring files with `.gitignore`

```
# Compiled Python files
__pycache__/
*.pyc

# Virtual environments
.venv/

# Large data files
*.fits
*.hdf5

# Secrets
.env
```

```bash
git check-ignore -v file.txt       # Check which rule applies
```

---

### Setting up aliases

```bash
git config --global alias.st "status"
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.undo "reset HEAD~1 --mixed"
```

---

### Excluding files locally without touching `.gitignore`

Edit `.git/info/exclude` — same syntax as `.gitignore`, not versioned, not shared.

---

### Normalising line endings with `.gitattributes`

See the section *Common problems → My line endings are causing conflicts*.

---

## Daily workflow

### The basic cycle: modify → stage → commit

```bash
git add file.py                 # Stage a file
git add -p                      # Stage interactively by chunk
git commit -m "Short, precise description"
```

Practical rule for messages: complete the sentence *"This commit…"*.

---

### Create a branch and switch to it

```bash
git switch -c my-feature        # Create + switch
git switch main                 # Switch to an existing branch
git branch                      # List local branches
git branch -d my-feature        # Delete a merged branch
```

---

### Merge a branch

```bash
git switch main
git merge my-feature            # Fast-forward if possible
git merge --no-ff my-feature    # Force a merge commit (keeps the trace)
```

---

### `merge` vs `rebase`: when to use which?

| | `merge` | `rebase` |
|---|---|---|
| History | Preserved, non-linear | Rewritten, linear |
| Use case | Branch integration | Cleanup before merge |
| Rule | Always fine | **Never on a shared public branch** |

```bash
git switch my-feature
git rebase main                 # Replay my-feature on top of main
```

---

### Viewing history efficiently

```bash
git log --oneline --graph --all --decorate
git log --author="Greg" --since="2 weeks ago"
git log --grep="fix" --oneline
git log -- src/model.py                         # History of a file
git log --follow -- old_name.py                 # Follow renames
```

---

### Comparing branches or commits

```bash
git diff main..my-feature
git log main..my-feature --oneline              # Commits in my-feature not yet in main
git diff HEAD~3 HEAD -- file.py
```

---

### Rewriting several commits (interactive `rebase`)

```bash
git rebase -i HEAD~5
```

In the editor: `pick`, `squash`, `reword`, `drop`, `edit`.

---

## Collaboration / Remotes

### Adding and managing a remote

```bash
git remote add origin git@github.com:user/repo.git
git remote -v
git remote set-url origin git@github.com:user/new-repo.git
```

---

### Pushing a branch

```bash
git push -u origin my-feature       # First time (sets up tracking)
git push                            # Subsequent pushes
git push origin --delete my-feature # Delete the remote branch
```

---

### `fetch` vs `pull`: what's the difference?

- `git fetch`: fetches remote changes without integrating them. Safe at any time.
- `git pull`: `fetch` + `merge` (or `rebase`). Modifies the current branch.

```bash
git fetch origin
git log origin/main --oneline       # Inspect before integrating
git merge origin/main
```

---

### Tagging a version

```bash
git tag -a v1.0.0 -m "Version 1.0.0"
git push origin v1.0.0
git push origin --tags
git tag -d v1.0.0                   # Delete locally
git push origin --delete v1.0.0     # Delete on the remote
```

---

#### Is there a convention for writing good commit messages?

Yes. The most widely adopted one is called **Conventional Commits** ([conventionalcommits.org](https://www.conventionalcommits.org)).

**Format:**

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

**Standard types:**

| Type | Use |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Neither feat nor fix — restructuring |
| `test` | Adding or fixing tests |
| `chore` | Maintenance, dependencies, CI |
| `perf` | Performance improvement |

**Examples:**

```
feat(model): add KL regularization to VAE encoder
fix(nms): move reshape inside conditional block
docs(api): update OTFClient usage examples
chore(deps): bump torch from 2.1 to 2.3
```

**Core rules** (Tim Pope, 2008 — still the reference):

- Subject line: **50 characters max**, no trailing period
- If body: **blank line** between subject and body
- Body: **72 characters per line**, explains the *why*, not the *what*
- Use the **imperative**: *Add feature*, not *Added feature*

**Why it's useful:** Conventional Commits allows you to automatically generate a changelog and drive semantic versioning (SemVer):

```bash
# With semantic-release or standard-version
feat  → minor version bump  (1.0.0 → 1.1.0)
fix   → patch version bump  (1.0.0 → 1.0.1)
feat! → major version bump  (1.0.0 → 2.0.0)  # breaking change
```

For a solo project or small team, the full convention is often overkill. The useful minimum is just the type prefix:

```
feat: add support for 512x512 tiles
fix: correct reshape outside conditional block
refactor: extract OTFClient into a separate module
docs: add LaTeX spec for the protocol
```

Even without tooling, it makes `git log --oneline` immediately readable.

---

## Debugging & Inspection

### Display the contents of a commit

```bash
git show abc1234
git show HEAD
git show HEAD~2
```

---

### List tracked files

```bash
git ls-files
git ls-files --others --exclude-standard    # Untracked files
```

---

### Search in code and history

```bash
git grep "function_name"                        # In current code
git log -S "function_name" --oneline            # Commits that changed this string
git log -G "regex" --oneline                    # Same with a regex
```

---

### Contribution statistics

```bash
git shortlog -sn                        # Number of commits per author
git shortlog -sn --since="1 year"
```

---

### Recovering a lost commit (`reflog`)

```bash
git reflog                              # History of all HEAD actions
git checkout abc1234
git branch recovery abc1234
```

---

### Finding the commit that introduced a bug (`bisect`)

See the section *Common problems → I don't know which commit introduced this bug*.

---

### Checking and optimising the repository

```bash
git fsck --full                         # Check integrity
git count-objects -vH                   # Repository size
git gc --aggressive                     # Clean up and compress
```

---

### Inspecting Git's internal objects

```bash
git cat-file -t abc1234     # Type: blob, tree, commit, tag
git cat-file -p abc1234     # Object contents
git ls-tree HEAD            # Tree of the current commit
```

---

## References

- Official Git documentation: [git-scm.com/doc](https://git-scm.com/doc)
- **Pro Git** (Chacon & Straub, 2nd ed.) — free online: [git-scm.com/book/en/v2](https://git-scm.com/book/en/v2)
- **Version Control with Git**: Powerful Tools and Techniques for Collaborative Software Development, Prem Kumar Ponuthorai, O'Reilly, 2022
- `man git-<command>` — built-in manual pages
