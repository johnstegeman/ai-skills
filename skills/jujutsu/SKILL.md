---
name: jujutsu
description: Use this skill for any git/vcs operations (commit, fetch, clone, push, diff, log, etc) - ESPECIALLY if git HEAD is detached. If a .jj directory exists, this is a jujutsu repository, and git commands could corrupt the repository. This skill includes essential safety instructions for working with git. **DO NOT IGNORE**
allowed-tools: Bash(jj *)
license: Apache-2.0
metadata:
  author: johnstegeman
  version: "1.0"
---

# Jujutsu (jj) Version Control System

This skill helps you work with Jujutsu, a Git-compatible VCS with mutable commits and automatic rebasing.

**Tested with jj v0.39.0** - Commands may differ in other versions.

## Important - discovering when to use jujutsu vs git

If the "jj" program is installed, you can run this in order to determine whether a directory is part of a jujutsu repository:

```bash
jj root
```

If the command returns a path, the repo is managed by jujutsu
If the command returns an error ("Error: There is no jj repo in <directory>") then it is not

Another tell-tale sign of a jujutsu repository is the presence of a .jj folder.

If the repository is a jj one, do NOT use git commands, as they are unsafe in jj repos. If the repo is not a jj one, do not further use this skill.

## Important: Automated/Agent Environment

When running as an agent or running on behalf of the user:

1. **Always use `-m` flags** to provide commit messages inline rather than relying on editor prompts:

```bash
# Always use -m to avoid editor prompts
jj desc -m "message"      # NOT: jj desc
jj squash -m "message"    # NOT: jj squash (which opens editor)
```

Editor-based commands will fail in non-interactive environments.

2. **Verify operations with `jj st`** after mutations (`squash`, `abandon`, `rebase`, `restore`) to confirm the operation succeeded.

## Important Principles

1. Each commit should represent one logical change. See [commit messages](references/COMMIT_MSG.md) for details on good commit messages. Before starting work on a new change, make sure the working copy is clean (no changes)

```bash
jj st
```

## Core Concepts

### The Working Copy is a Commit

In jj, your working directory is always a commit (referenced as `@`). Changes are automatically snapshotted when you run any jj command. There is no staging area.

There is no need to run `jj commit`.

### Commits Are Mutable

**CRITICAL**: Unlike git, jj commits can be freely modified. This enables a high-quality commit workflow:

1. Before starting work, run `jj st`. If `@` already has changes, run `jj new` first. If `@` is empty, use it as-is.
2. Describe your intended changes with `jj desc -m "Message"`
3. Make your changes.
4. Do NOT run `jj new` when finished — leave that to the next task's step 1.

You may refine the commit using `jj squash` or `jj absorb` as needed

### Change IDs vs Commit IDs

- **Change ID**: A stable identifier (like `tqpwlqmp`) that persists when a commit is rewritten
- **Commit ID**: A content hash (like `3ccf7581`) that changes when commit content changes

Prefer using Change IDs when referencing commits in commands.

## Essential Workflow

### Starting Work: Describe First, Then Code

**Always create your commit message before writing code:**

```bash
# First, describe what you intend to do
jj desc -m "feat: Add user authentication to login endpoint"

# Then make your changes - they automatically become part of this commit
# ... edit files ...

# Check status
jj st
```

### Creating Atomic Commits

Each commit should represent ONE logical change.

### Viewing History

```bash
# View recent commits
jj log

# View with patches
jj log -p

# View specific commit
jj show <change-id>

# View diff of working copy
jj diff
```

### Moving Between Commits

```bash
# Create a new empty commit on top of current
jj new

# Create new commit with message
jj new && jj desc -m "Commit message"

# Edit an existing commit (working copy becomes that commit)
jj edit <change-id>

# Edit the previous commit
jj prev -e

# Edit the next commit
jj next -e
```

## Refining Commits

If you need to refine a commit (squash, abandon, undo, split, etc), see (Refining commits)[references/REFINE_COMMIT.md]

## Working with Bookmarks (Branches)

Bookmarks are jj's equivalent to git branches:

```bash
# Create a bookmark at current commit
jj bookmark create my-feature -r@

# Move bookmark to a different commit
jj bookmark move my-feature --to <change-id>

# List bookmarks
jj bookmark list

# Delete a bookmark
jj bookmark delete my-feature
```

## Working with tags

jujutsu does not yet support tags and pushing them to a remote. If you need to tag (such as for a release), you will need to use git to create and push the tags.

## Git Integration

### Working with Existing Git Repos
Always use colocated repositories
```bash
# Clone a git repository
jj git clone <url> --colocate

# Initialize jj in an existing git repo
jj git init --colocate
```

### Switching Between jj and git (Colocated Repos)

In a colocated repository (where both `.jj/` and `.git/` exist), you can use both jj and git commands. However, there are important considerations:

**Switching to git mode** (e.g., for merge workflows):
```bash
# First, ensure your jj working copy is clean
jj st

# Then checkout a branch with git
git checkout <branch-name>
```

**Switching back to jj mode**:
```bash
# Use jj edit to resume working with jj
jj edit <change-id>
```

**Important notes:**
- Git may complain about uncommitted changes if jj's working copy differs from the git HEAD
- ALWAYS ensure your work is committed in jj before switching to git
- After git operations, jj will detect and incorporate the changes on next command


### Pushing Changes

If you need to push changes to a remote, see (Pushing changes)[references/PUSH.md]

## Handling Conflicts

jj allows committing conflicts — you can resolve them later:

```bash
# View conflicts
jj st
```

**Agent conflict resolution**: Do not use `jj resolve` (interactive). Instead, edit the conflicted files directly to remove conflict markers, then run `jj st` to verify resolution.

## Preserving Commit Quality

**IMPORTANT**: Because commits are mutable, always refine them:

1. **Review your commit**: `jj show @` or `jj diff`
2. **Is it atomic?** One logical change per commit
3. **Is the message clear?** Use imperative verb phrase in sentence case format with no full stop: "Verb object"
4. **Are there unrelated changes?** Use `jj restore` to move changes out, then create separate commits
5. **Should changes be elsewhere?** Use `jj squash` or `jj absorb`

## Quick Reference

| Action | Command |
|--------|---------|
| Describe commit | `jj desc -m "message"` |
| View status | `jj st` |
| View log | `jj log` |
| View diff | `jj diff` |
| New commit | `jj st` then `jj new` only if `@` has changes, then `jj desc -m "message"` |
| Edit commit | `jj edit <id>` |
| Squash to parent | `jj squash` |
| Auto-distribute | `jj absorb` |
| Abandon commit | `jj abandon <id>` |
| Undo last operation | `jj undo` |
| Restore files | `jj restore [paths]` |
| Create bookmark | `jj bookmark create <name>` |
| Push bookmark | `jj git push -b <name>` |

## Best Practices Summary

1. **Describe first**: Set the commit message before coding
2. **One change per commit**: Keep commits atomic and focused
3. **Use change IDs**: They're stable across rewrites
4. **Refine commits**: Leverage mutability for clean history
5. **Embrace the workflow**: No staging area, no stashing - just commits
