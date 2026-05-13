# Token-Reducing Templates

Use these inline `-T` templates when you want compact, agent-friendly jj output. Prefer inline `-T` use over editing repo or global jj config so the skill stays explicit and portable.

Assumptions:
- Assume `jj 0.41.0` or later.
- Each template below was tested with `jj 0.41.0`.
- If a template fails to parse, treat that as jj version drift and verify with `jj <command> --help` plus `jj help -k templates`.

## `jj log`

Tested with: `jj 0.41.0`

```bash
jj log --no-graph -n 20 -T 'change_id.shortest(8) ++ if(bookmarks, " " ++ bookmarks, "") ++ if(current_working_copy, " @", "") ++ if(empty, " (empty)", "") ++ if(conflict, " (conflict)", "") ++ " " ++ if(description, description.first_line(), "(no description)") ++ "\n"'
```

This keeps the important bits: change ID, bookmark or `@`, empty/conflict markers, and the first description line. Use `shortest(8)` instead of bare `shortest()` so IDs do not collapse to one character on `jj 0.41.0`.

## `jj show`

Tested with: `jj 0.41.0`

```bash
jj show <change-id> --stat -T 'change_id.shortest(8) ++ if(bookmarks, " " ++ bookmarks, "") ++ if(current_working_copy, " @", "") ++ if(empty, " (empty)", "") ++ if(conflict, " (conflict)", "") ++ " " ++ if(description, description.first_line(), "(no description)") ++ "\n"'
```

Prefer `--stat` for a compact file summary. If you only need the header, swap `--stat` for `--no-patch`.

## `jj diff`

Tested with: `jj 0.41.0`

```bash
jj diff -T 'self.status_char() ++ " " ++ self.display_diff_path() ++ "\n"'
```

Add `-r <rev>` if you want a diff for a specific revision instead of the working copy.

## `jj bookmark list`

Tested with: `jj 0.41.0`

```bash
jj bookmark list -T 'if(self.normal_target(), self.name() ++ ": " ++ self.normal_target().change_id().shortest(8) ++ " " ++ if(self.normal_target().description(), self.normal_target().description().first_line(), "(no description)") ++ "\n", self.name() ++ ": (conflicted)\n")'
```

Use this for ordinary bookmark inspection. If you are explicitly debugging conflicted bookmarks and need every old/new target, fall back to jj's default output.

## `jj workspace list`

Tested with: `jj 0.41.0`

```bash
jj workspace list -T 'self.name() ++ ": " ++ self.target().change_id().shortest(8) ++ " " ++ if(self.target().description(), self.target().description().first_line(), "(no description)") ++ if(self.target().empty(), " (empty)", "") ++ "\n"'
```

This shows the workspace name, working-copy change ID, description, and whether that working copy is empty.

## `jj op log`

Tested with: `jj 0.41.0`

```bash
jj op log --no-graph -n 20 -T 'id.short(8) ++ " " ++ time.start().ago() ++ " " ++ description.first_line() ++ "\n"'
```

Use this first when you need recovery context but do not yet need full diffs. Add `-p` only after you have narrowed down the interesting op IDs.

## `jj evolog`

Tested with: `jj 0.41.0`

```bash
jj evolog -r <change-id> --no-graph -n 20 -T 'commit.change_id().shortest(8) ++ " " ++ if(commit.description(), commit.description().first_line(), "(no description)") ++ " | " ++ operation.id().short(8) ++ " " ++ operation.description().first_line() ++ "\n"'
```

This keeps the evolving change summary on the left and the rewriting operation on the right.

## Not Templated

`jj st` and `jj status` do not expose a template flag in `jj 0.41.0`, so leave status output untemplated. The same applies to commands like `jj undo`, `jj op restore`, and `jj resolve` where the value is in the side effect, not in custom rendering.
