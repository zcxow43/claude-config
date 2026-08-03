Discard all local changes and reset the working tree with `git reset --hard`. Optional argument: a ref/commit to reset to (defaults to `HEAD`).

## Argument Handling

- If `$ARGUMENTS` is empty or blank: reset to `HEAD` (discard uncommitted changes only, no history rewrite).
- Otherwise: reset to `$ARGUMENTS` (e.g. a commit hash, branch, or `HEAD~1`) — this can also move the branch pointer and rewrite history.

## Steps

1. Run `git status` to show what will be discarded.
2. Run `git reset --hard $ARGUMENTS` (or `git reset --hard HEAD` if no argument).
3. Run `git status` again to confirm a clean working tree.

## Rules

- Do NOT ask for confirmation — run immediately, per this project's Full Autonomy rule in `CLAUDE.md`.
- Never add `--no-verify` or other flags beyond the target ref.
- Do not touch the `.claude` submodule unless `$ARGUMENTS` explicitly targets it.
