# Git & Version Control

> **TL;DR:** Know merge vs rebase, recover with reflog, and pick a branching model that matches your release cadence. Hooks and signed commits raise the floor.

## Mental Model

Git tracks **content addressed by SHA**, not files. A commit = pointer to a tree (snapshot) + parent SHAs + metadata. Branches and tags are just movable refs (a file under `.git/refs/`) that point to a commit SHA. Once you internalize this, advanced operations are obvious.

```bash
git cat-file -p HEAD               # see the commit object
git cat-file -p HEAD^{tree}        # the tree it points to
git rev-parse HEAD                 # current commit SHA
```

## Daily Workflow

```bash
git status -sb                     # short status with branch info
git diff                           # unstaged
git diff --staged                  # staged
git add -p                         # patch mode (chunk-by-chunk staging)
git commit -m "feat: add login"
git push origin HEAD               # safer than -u origin branch-name
git pull --rebase                  # avoid noisy merge commits on pull
```

Set `git config --global pull.rebase true` once and never look back.

## Branching Strategies

### Trunk-Based Development (recommended for CD)

- Everyone commits to `main` (or short-lived branches < 1 day).
- Feature flags hide unfinished work in prod.
- CI gates every merge.
- Releases are tags on `main`.
- Best fit: SaaS, microservices, daily/hourly deploys.

### GitFlow (legacy, for versioned releases)

```
main      ----o------------------o-------    (production tags)
              \                  /
release        \--o----o--------/
                                  
develop   -o---o---o---o---o---o-----o---    (integration)
           \           \                
feature/x   \--o---o----                
```

- `develop` for ongoing work, `feature/*` for new work, `release/*` to stabilize, `hotfix/*` straight off `main`.
- Heavy ceremony. Best for shrink-wrap software, mobile apps with store review cycles, regulated releases.

### GitHub Flow (most teams' middle ground)

- `main` always deployable.
- `feature/*` branches → PR → review → squash-merge → deploy.

## Merge vs Rebase

```bash
# Merge: preserves topology, creates a merge commit
git checkout main
git merge feature/x                # creates merge commit M
# A---B---C---M (main)
#      \     /
#       D---E (feature/x)

# Rebase: replays your commits onto a new base, linear history
git checkout feature/x
git rebase main
# A---B---C---D'---E' (feature/x)  -- D and E rewritten as D', E'
```

**Rules of thumb:**
- Rebase **local, unpushed** work freely.
- **Never** rebase a branch others have pulled. Their reflog points to old SHAs.
- Use **squash-merge** on PR merge for clean linear `main` while letting devs rebase locally.

```bash
git rebase -i HEAD~5               # interactive: reorder, squash, edit, drop
git rebase --abort                 # bail out mid-rebase
git rebase --continue              # after resolving conflicts
```

## Cherry-Pick

```bash
git checkout release/1.4
git cherry-pick abc123             # apply one commit from elsewhere
git cherry-pick abc123^..def456    # range (exclusive of abc123's parent)
git cherry-pick -x abc123          # appends "(cherry picked from commit ...)" line
```

Use case: pulling a bugfix from `main` into a frozen release branch.

## The Reflog Saves Lives

Every ref update is logged for 90 days (default). Lost a commit after a bad rebase? It's still there.

```bash
git reflog                         # local history of HEAD movements
# 8c3a1f2 HEAD@{0}: reset: moving to HEAD~3
# d5e7a91 HEAD@{1}: commit: WIP scary stuff
git reset --hard HEAD@{1}          # recover

git fsck --lost-found              # find dangling commits
```

If you ever type `git reset --hard` and regret it — reflog has you. Until garbage collection runs.

## Resolving Conflicts

```bash
# Conflict markers
<<<<<<< HEAD
your version
=======
their version
>>>>>>> feature/x

# Tooling
git mergetool                      # opens configured merge tool
git config --global merge.tool vimdiff

# After fixing
git add path/to/file
git rebase --continue              # or: git commit (during merge)

# Pick one side wholesale
git checkout --ours path/to/file   # keep current branch's version
git checkout --theirs path/to/file # keep incoming's
```

## Undoing Things

| What | Command | Destructive? |
|------|---------|--------------|
| Unstage a file | `git restore --staged FILE` | No |
| Discard working changes | `git restore FILE` | Yes |
| Amend last commit message | `git commit --amend` | Rewrites SHA |
| Add to last commit | `git add . && git commit --amend --no-edit` | Rewrites SHA |
| Undo last commit, keep changes | `git reset --soft HEAD~1` | Reflog covers it |
| Undo last commit, discard changes | `git reset --hard HEAD~1` | Reflog covers it |
| Revert a pushed commit | `git revert <sha>` | Creates new commit (safe) |

> **Never `--amend` or `reset --hard` on a branch others have pulled.** Use `revert` for shared history.

## Git Hooks

Local hooks live in `.git/hooks/`. They're not versioned by default — use [Husky](https://typicode.github.io/husky) (Node) or [pre-commit](https://pre-commit.com) (language-agnostic) to share them.

```bash
# .git/hooks/pre-commit  (executable)
#!/usr/bin/env bash
set -e
echo "Running lint..."
npm run lint
npm test
```

Common hooks:
- `pre-commit` — lint, format, test
- `commit-msg` — enforce conventional commits
- `pre-push` — run integration tests / block force-push to main
- `prepare-commit-msg` — auto-add Jira ticket from branch name

### Conventional Commits

```
<type>(<scope>): <subject>

<body>

<footer>
```

```
feat(auth): add OIDC login
fix(api): handle null user in /me endpoint
chore(deps): bump axios to 1.7.4
docs(readme): add deploy instructions

BREAKING CHANGE: /v1/users response shape changed
```

Tools like `semantic-release` and `release-please` use this to compute the next version automatically.

## Signed Commits

```bash
git config --global user.signingkey ABC123DEF
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# SSH signing (newer, simpler)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
```

GitHub shows a "Verified" badge. Enforce with branch protection rules.

## Tagging Releases

```bash
git tag -a v1.4.0 -m "release 1.4.0"   # annotated tag (recommended)
git push origin v1.4.0
git push --tags                         # all tags

git tag -d v1.4.0                       # local delete
git push origin :refs/tags/v1.4.0       # remote delete

git describe --tags                     # closest tag (v1.4.0-3-gabc123)
```

## `.gitignore` & `.gitattributes`

```gitignore
# .gitignore
node_modules/
*.log
.env
.env.*
!.env.example
dist/
.idea/
.DS_Store
```

```gitattributes
# .gitattributes
* text=auto                # normalize line endings
*.sh text eol=lf           # always LF (esp. on Windows checkouts)
*.bat text eol=crlf
*.png binary
package-lock.json -diff    # don't show diff for lockfiles
```

## Submodules vs Subtrees vs Monorepo

- **Submodule:** pinned ref to another repo. Pain to onboard, fast for read-only deps.
- **Subtree:** copies content in. Easier for contributors, history bloats.
- **Monorepo with workspaces:** simplest if you can — tooling is mature (Nx, Turborepo, Bazel).

## Interview Questions

**Q: When do you `git rebase` vs `git merge`?**
A: Rebase to clean up *your own* local feature branch before pushing — gives linear history. Merge when integrating into shared branches, or when topology matters. Never rebase shared branches; you'll force-push and break everyone else.

**Q: How do you recover a commit you accidentally `git reset --hard`-ed away?**
A: `git reflog` to find the prior HEAD position, then `git reset --hard HEAD@{1}` (or whichever entry). Works until GC runs (~30–90 days).

**Q: What's the difference between `git revert` and `git reset`?**
A: `revert` creates a new commit that undoes a previous one — safe for shared history. `reset` moves the branch pointer — rewrites history, dangerous if pushed.

**Q: How do you push a hotfix to production while a feature branch is in progress?**
A: Branch from `main`, fix, PR, merge to `main`, tag, deploy. Then cherry-pick or merge `main` into the feature branch to stay current.

**Q: What's a fast-forward merge?**
A: When the target branch hasn't diverged from the source, git just moves the pointer forward — no merge commit. Disable with `--no-ff` if you want a merge commit for traceability.

**Q: How do you find which commit introduced a bug?**
A: `git bisect start` → `git bisect bad HEAD` → `git bisect good v1.3.0` → git checks out midpoints; you mark good/bad until it identifies the culprit. Can be scripted: `git bisect run ./test.sh`.

## Common Pitfalls

- Force-pushing to `main` / shared branches — destroys colleagues' work.
- Committing secrets — even after deletion, history retains them. Use `git filter-repo` + rotate the secret immediately.
- Merging without pulling first — gives confusing merge commits.
- Letting branches diverge for weeks — merge conflicts compound.
- `git pull` overwriting local commits when `pull.rebase=false` and you have local commits (results in merge commit on every pull).

## Related

- [10-ci-cd-fundamentals.md](10-ci-cd-fundamentals.md)
- [11-github-actions.md](11-github-actions.md)
- [21-security-devsecops.md](21-security-devsecops.md)
