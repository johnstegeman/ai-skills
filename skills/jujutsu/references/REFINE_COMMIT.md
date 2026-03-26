### Squashing Changes

Move changes from current commit into its parent:

```bash
# Squash all changes into parent
jj squash
```

**Note**: `jj squash -i` opens an interactive UI and will hang in agent environments. Avoid it.

### Splitting Commits

**Warning**: `jj split` is interactive and will hang in agent environments. To divide a commit, use `jj restore` to move changes out, then create separate commits manually.

### Absorbing Changes

Automatically distribute changes to the commits that last modified those lines:

```bash
# Absorb working copy changes into appropriate ancestor commits
jj absorb
```

### Abandoning Commits

Remove a commit entirely (descendants are rebased to its parent):

```bash
jj abandon <change-id>
```

### Undoing Operations

Reverse the last jj operation:

```bash
jj undo
```

This reverts the repository to its state before the previous command. Useful for recovering from mistakes like accidental `abandon`, `squash`, or `rebase`.

### Restoring Files

Discard changes to specific files or restore files from another revision:

```bash
# Discard all uncommitted changes in working copy (restore from parent)
jj restore

# Discard changes to specific files
jj restore path/to/file.txt

# Restore files from a specific revision
jj restore --from <change-id> path/to/file.txt
