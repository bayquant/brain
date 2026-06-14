---
tags: [git, github, workflow]
---
# Git and GitHub

## Git State Mechanism

```
[Working Directory] → [Staging Area (Index)] → [Local Repository] → Remote Repository
```

## File Types

- **Tracked**: files in the last snapshot — can be Unmodified, Modified, or Staged
- **Untracked**: files not in the last snapshot and not in the staging area

---

## Core Commands

| Command | Purpose |
|---|---|
| `git init` | Turn current directory into a Git repo |
| `git init <dir>` | Create empty Git repo in specified directory |
| `rm -rf .git` | Undo `git init` |
| `git clone <HTTPS/SSH>` | Clone a remote repo locally |
| `git status` | Show working directory and staging area state |
| `git add <file>` | Stage a file |
| `git add .` | Stage everything in current path |
| `git commit -m 'msg'` | Record staged changes to the repo |
| `git push --set-upstream origin <branch>` | Push branch to remote for the first time |
| `git pull <remote> <branch>` | Fetch and merge from remote |
| `git log` | Show commit history with authors and dates |
| `git show` | Show diff between last two commits |

---

## Configuration

```bash
git config --global user.email 'email'   # Set global email
git config --global user.name 'name'     # Set global username
git config --list                         # Show all config values
```

---

## Branches

```bash
git branch                    # List local branches
git branch -r                 # List remote branches
git branch -a                 # List all branches (local + remote)
git checkout <branch>         # Switch to a branch
git checkout -b <branch>      # Create and switch to a new branch
git branch -d <branch>        # Delete a local branch
```

---

## Pull Request Workflow

**1. Stage, commit, and push your branch:**
```bash
git add .
git commit -m 'message'
git push origin <branch>
```

**2. Create the PR on GitHub.**

**3. After merge, update local main:**
```bash
git checkout main
git pull origin main
```

**4. Update other local branches:**
```bash
git checkout <branch>
git merge main
git push origin <branch>
```

---

## Overwrite Local Files with Remote

```bash
git fetch --all                             # Download latest without merging
git reset --hard <remote>/<branch>          # Reset to match remote exactly (local changes lost)
```

---

## diff

```bash
git diff                    # Working directory vs staging area
git diff --staged           # Staging area vs last commit (also: --cached)
```

---

## reset — Revert Commits

**Mixed (default):** unstage files, keep changes in working directory
```bash
git reset
# or
git restore --staged
```

**Soft:** move commits back to staging area (commits removed)
```bash
git reset --soft HEAD~<n>          # Remove last n commits
git reset --soft <commit-hash>     # Remove all commits after this hash
```

**Hard:** permanently discard commits and changes
```bash
git reset --hard HEAD~<n>
git reset --hard <commit-hash>
```

> After a hard reset, force-push with: `git push --force origin <branch>`
> Prefer `git push --force-with-lease` — safer, fails if someone else pushed in the meantime.

---

## clean — Remove Untracked Files

```bash
git clean -f      # Remove untracked files (non-recursive)
git clean -df     # Remove untracked files and directories (recursive)
```

---

## merge

```bash
git merge <branch>   # Merge branch into current branch
```

**Manual merge (good practice even when auto-merge is possible):**
```bash
git checkout master
git pull
git checkout <dev-branch>
git merge master -m 'message'
```

**Merge conflict:** when the same lines are changed in both branches. Git marks conflicts:
```
<<<<<<< HEAD
your changes
=======
incoming changes
>>>>>>> master
```

Fix the file, then:
```bash
git add .
git commit -m 'merge conflict addressed'
```

---

## rebase

Moves commits to a new base to maintain a **linear history**.

```bash
git checkout master
git pull
git checkout <dev-branch>
git rebase master
```

### Interactive rebase with `-i` and pick options (`-p`)

`git rebase -i` opens an editor listing commits, each prefixed with a command. Change the command to control what happens to each commit:

| Command | Shorthand | Effect |
|---|---|---|
| `pick` | `p` | Keep the commit as-is |
| `reword` | `r` | Keep commit but edit its message |
| `edit` | `e` | Pause rebase to amend the commit (files + message) |
| `squash` | `s` | Meld into the previous commit, combining messages |
| `fixup` | `f` | Meld into the previous commit, discarding this message |
| `drop` | `d` | Remove the commit entirely |
| `exec` | `x` | Run a shell command after the preceding commit |

```bash
git rebase -i HEAD~3        # Rewrite last 3 commits
git rebase -i <commit-hash> # Rewrite commits after this hash
```

Workflow: change `pick` to the desired command, save and quit (`:wq`). Git replays commits top-to-bottom, pausing when `edit` or a conflict is encountered.

```bash
git rebase --continue   # Proceed after resolving a conflict or edit
git rebase --abort      # Cancel and return to the original state
git rebase --skip       # Skip the current conflicting commit
```

---

## squash — Combine Commits

Squash after rebasing or when branch is ahead. Run squash only when your branch is already up to date.

```bash
git rebase -i HEAD~<n>      # Open interactive rebase for last n commits
```

In the editor: change `pick` to `squash` on commits you want to combine.
Press `i` to insert, `ESC` when done, then `:wq` to save.

---

## commit --amend — Edit Last Commit

```bash
git commit --amend -m 'new message'     # Amend message directly
git commit --amend --no-edit            # Add staged changes without changing message
```

If already pushed:
```bash
git push --force-with-lease
```

---

## cherry-pick — Move a Commit to Another Branch

```bash
git checkout <target-branch>
git cherry-pick <commit-hash>
# If conflict:
git cherry-pick --continue
```

---

## checkout + stash

```bash
git checkout <branch>           # Switch branch
git checkout <commit-hash>      # Go to a specific commit (creates detached HEAD)
git checkout -b <new-branch>    # Create a branch from detached HEAD state
```

> **Warning:** uncommitted changes carry over when switching branches. Use stash to prevent this:

```bash
git stash          # Save dirty working directory
git stash pop      # Restore stashed changes on current branch
```

---

## blame — Who Changed What

```bash
git blame <file>    # Show author, date, and commit for each line
```

---

## bisect — Find the Commit That Introduced a Bug

Binary search through commit history.

```bash
git bisect start
git bisect bad <commit-hash>     # Known bad commit
git bisect good <commit-hash>    # Known good commit
# Git checks out a midpoint — test it, then:
git bisect good   # or: git bisect bad
# Repeat until the culprit commit is found
```

---

## tags — Mark Specific Points in History

Tags are refs that point to a specific commit — typically used to mark releases.

**Two types:**

- **Lightweight:** a simple pointer to a commit, no extra metadata.
- **Annotated:** a full Git object with tagger name, date, message, and optional GPG signature. Preferred for releases.

```bash
# Annotated tag (recommended)
git tag -a v1.0.0 -m "Release v1.0.0"

# Lightweight tag
git tag v1.0.0

# Tag a specific commit
git tag -a v1.0.0 <commit-hash> -m "message"
```

**Listing and inspecting:**
```bash
git tag                    # List all tags
git tag -l "v1.*"          # Filter tags by pattern
git show v1.0.0            # Show tag details and the tagged commit
```

**Pushing tags to remote:**
```bash
git push origin v1.0.0     # Push a single tag
git push origin --tags     # Push all tags
```

**Deleting tags:**
```bash
git tag -d v1.0.0                    # Delete local tag
git push origin --delete v1.0.0     # Delete remote tag
```

**Checking out a tag** puts you in a detached HEAD state:
```bash
git checkout v1.0.0
git checkout -b <branch> v1.0.0    # Create a branch from a tag
```

---

## GitHub

- [Rule sets for branches and tags](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)

---

## Terminal

See [[Terminal]].
