Commit and push changes. Optional argument: `wdd` or `claude`. With no argument, commit **both**.

## Argument Validation

Check the value of `$ARGUMENTS`:

- If `$ARGUMENTS` is empty or blank: commit **both** repos — run the `claude` flow first, then the `wdd` flow (see **When no argument** below).
- If `$ARGUMENTS` is exactly `wdd` or `claude`: run that flow only.
- Otherwise: **print error** `Error: invalid argument "$ARGUMENTS". Usage: /commit [wdd|claude]` and **STOP**.

## When no argument

Commit **both** repos, in this order:

1. Run the **`claude`** flow (commits the submodule and updates its reference in the main repo).
2. Then run the **`wdd`** flow (commits all remaining main-repo changes).

## When argument is `wdd`

Commit and push the **main project repo** (the project root).

1. Run `git status` to see all changes
2. Run `git diff` to understand what changed
3. Run `git log --oneline -3` to match commit message style
4. Stage ALL changed and untracked files (except .env, credentials, secrets)
5. Write a concise commit message summarizing the changes
6. Commit and push to the current branch

## When argument is `claude`

Commit and push the **`.claude` submodule** only.

1. `cd .claude`
2. Run `git status` to see all changes
3. Run `git diff` to understand what changed
4. Stage ALL changed and untracked files
5. Write a concise commit message summarizing the changes
6. Commit and push to the current branch
7. `cd ..` back to project root
8. Stage the updated submodule reference: `git add .claude`
9. Commit with message `Update .claude submodule` and push

## Commit Message Format

- One-line summary of what changed and why
- If multiple concerns, use bullet points in the body
- End with: `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`

## Rules

- Do NOT ask for confirmation — commit and push immediately
- Stage everything that changed
- Push to the current branch (use `git branch --show-current` to determine it)
- If push fails due to upstream not set, use `git push -u origin <branch>`
