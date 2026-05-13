# Parallel Workspaces for Agent Isolation

`jj workspace` lets you attach additional working copies to the same repository — each with its own `@` (working-copy commit) but sharing the underlying op log and store. The primary use case in this skill: **run multiple agents (Claude/Codex/etc.) in parallel against the same repo without them stepping on each other's working copy.**

In `jj log`, additional workspaces show as `<workspace-name>@`. The default workspace is just `@`.

## When to use a workspace

- **Multi-agent parallel work**: one agent investigates while another implements; one builds the feature while another writes tests
- **Slow build/test loops**: keep editing in workspace A while a long test runs in workspace B
- **Risky experiments**: try a refactor in an isolated workspace without disturbing your main working copy
- **Reviewer + builder split**: spawn a workspace for a review/lint pass while continuing to write code

If you just want to switch what `@` points at, use `jj edit <change>` or `jj new` — workspaces are heavier and meant for *concurrent* working copies.

## Directory convention: `.workspaces/` in the project root

Always follow this priority when choosing where to put a new workspace:

1. **First**: if the project already has a `.workspaces/` or `workspaces/` directory in the repo root, use it.
2. **Second**: check `CLAUDE.md` (or equivalent project docs) for an explicit workspace location preference.
3. **Third**: if neither exists, ask the user:
   - **Option A** — Create project-local `.workspaces/` (recommended; keeps everything together)
   - **Option B** — Use global `~/workspaces/` (separation from the project tree)

**Never assume** — follow the priority above and ask when ambiguous.

## Safety check: ignore the workspace directory (CRITICAL)

For project-local workspace directories, the path **must be in `.gitignore` (or `.jjignore` if the project uses one)**. Workspaces contain working files that should never be tracked by the outer repo.

```bash
# Check the ignore file
grep -E '^\.?workspaces/?$' .gitignore .jjignore 2>/dev/null
```

If the entry is missing, add and commit it before creating the workspace:

```bash
echo ".workspaces/" >> .gitignore
jj describe -m "chore: Add .workspaces to gitignore"
```

**Never skip this check** — `jj workspace add` creates directories containing `.jj/` metadata. If those leak into the outer git history the repo will fight you.

## Naming convention

Use `${REPO_NAME}-<purpose>` so workspaces are self-describing in `jj log` and `jj workspace list`:

```bash
REPO_NAME=$(basename $(jj workspace root))
WORKSPACE_PATH=".workspaces/${REPO_NAME}-<purpose>"

# Examples:
#   .workspaces/myrepo-test-runner
#   .workspaces/myrepo-reviewer-agent
#   .workspaces/myrepo-refactor-experiment
```

Avoid generic names like `tmp`, `wip`, or `new` — they collide and obscure intent.

## Creating a workspace

```bash
jj workspace add "$WORKSPACE_PATH"
```

This will:
- Create `$WORKSPACE_PATH/` with its own `.jj/` working-copy state
- Track a new working-copy commit (visible as `<workspace-name>@` in `jj log`)
- Inherit the current change as its parent

After creation, the workspace is a real directory you can `cd` into. Make changes there — they snapshot to that workspace's `@` independently of the main workspace.

## Bootstrapping the workspace environment

After creating the workspace, detect project type and run setup so the workspace is immediately usable. Don't make the user do this manually.

| Marker file | Setup command |
|---|---|
| `package.json` | `npm install` (or `pnpm install` / `bun install` / `yarn install` per lockfile) |
| `Cargo.toml` | `cargo build` |
| `pyproject.toml` | `uv sync` (or `poetry install` / `pip install -e .`) |
| `Gemfile` | `bundle install` |
| `flake.nix` | `nix develop` |
| `go.mod` | `go mod download` |
| `mix.exs` | `mix deps.get` |

## The agent pattern: two agents, one repo

Each workspace's `@` is independent. Two agents can each have their own working copy:

```bash
# Agent A in the main workspace
cd ~/projects/myrepo
jj desc -m "feat: Add login form"
# ... edits ...

# Agent B in a parallel workspace
cd ~/projects/myrepo/.workspaces/myrepo-tests
jj desc -m "test: Cover login validation"
# ... edits ...
```

Both snapshot independently. View the whole picture with `jj log` from either workspace:

```bash
jj log
# Shows both workspace tips, marked by workspace name:
#   @       (current workspace's working copy)
#   tests@  (the parallel workspace)
```

If you need to inspect workspace tips as an agent, start with the compact `jj log` command from `references/TEMPLATES.md`.

## Combining results when work is done

When a workspace finishes its work, you usually want to bring its change back into the default workspace (or onto the main bookmark) and retire the workspace. **Always ask the user which strategy to use** — the right answer depends on context. Three strategies, with a decision table at the end.

### Strategy 1 (most common): rebase into the default workspace

For solo/agent-fleet work where you control both ends, the natural endpoint is to move the workspace's change onto your default workspace's line, then retire the workspace.

```bash
# From the default workspace, list all workspace tips
jj log

# Rebase the workspace's change onto the default workspace's @ (or any chosen base)
jj rebase -s <workspace-change-id> -d @

# Optionally advance the main bookmark
jj bookmark move main --to <workspace-change-id>

# Retire the workspace
jj workspace forget myrepo-feature
rm -rf .workspaces/myrepo-feature
```

This is the right default when one developer is orchestrating multiple agents and the result should land as ordinary commits in the default workspace.

### Strategy 2: explicit merge commit

When two workspaces produced genuinely parallel work that should remain visible as a merge in the history:

```bash
jj new <change-from-workspace-a> <change-from-workspace-b> -m "merge: Combine login feature with tests"
```

Use this when both lines of work are independently meaningful and the merge structure carries information (e.g., they were developed against the same base for a reason).

### Strategy 3: push and open a PR

When the work needs review by other humans before it lands:

```bash
# Create a bookmark for the work
jj bookmark create my-feature -r <workspace-change-id>

# Push and open a PR
jj git push -b my-feature
gh pr create     # or open in the browser
```

The workspace can stay attached during review so you can push fixups. Forget it after the PR merges.

### Decision rule: ask the user, then suggest

Don't decide silently. Surface the options and recommend a default based on context:

| Context | Recommended strategy |
|---|---|
| Solo developer running multiple agents on local work | Strategy 1 (rebase into default workspace) |
| Long-lived feature with iterative checkpoints | Strategy 1, then push a bookmark when ready |
| Code review required (team project, OSS contribution) | Strategy 3 (PR) |
| Both workspace lines are independently meaningful and the merge should be visible | Strategy 2 (explicit merge) |

If the context is ambiguous (a new repo, no prior signal), default to suggesting Strategy 1 and ask "Should this land in your default workspace, or do you want a PR?"

## Listing and cleanup

```bash
jj workspace list                  # All attached workspaces
jj workspace root                  # Show this workspace's root
jj workspace forget <name>         # Stop tracking; files stay on disk
```

If you need to run `jj workspace list`, use the compact command from `references/TEMPLATES.md`.

After `jj workspace forget`, the workspace directory is just files — you can `rm -rf` it. The change still exists in repo history if it was committed.

```bash
# Full cleanup pattern
jj workspace forget myrepo-tests
rm -rf .workspaces/myrepo-tests
```

## `update-stale`: when another agent moved things

If the change at a workspace's `@` was rewritten by work in another workspace (e.g., a squash on a shared ancestor), `jj` flags it as stale:

```bash
jj workspace update-stale
```

This re-snapshots the working copy onto the rewritten commit. Safe; preserves working-copy edits.

## Critical rules

1. **Never skip the `.gitignore` check** for project-local workspace directories.
2. **Always follow the directory priority** — `.workspaces/` → `workspaces/` → CLAUDE.md → ask.
3. **Use descriptive names** — `${REPO_NAME}-<purpose>`, not `tmp`.
4. **Auto-detect and bootstrap** — don't leave the user to `npm install` manually.
5. **Report clearly** when a workspace is ready: full path, the change ID, and what to run next.
6. **`forget` before deleting** — `jj workspace forget` first, then `rm -rf`.
7. **Ask before combining** — when a workspace's work is done, ask the user which combining strategy to use (rebase into default workspace, merge commit, or PR). Suggest a default per the decision table above; don't pick silently.

## Reference

Convention adapted from [edmundmiller/dotfiles `using-jj-workspaces`](https://lobehub.com/skills/edmundmiller-dotfiles-using-jj-workspaces).
