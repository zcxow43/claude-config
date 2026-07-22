Commit all changes and push to the current branch.

## Process

1. Run `git status` to see all changes
2. Run `git diff` to understand what changed
3. Run `git log --oneline -3` to match commit message style
4. Stage ALL changed and untracked files (except .env, credentials, secrets)
5. Write a concise commit message summarizing the changes
6. Commit and push to the current branch

## Commit Message Format

- One-line summary of what changed and why
- If multiple concerns, use bullet points in the body
- End with: `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`

## Rules

- Do NOT ask for confirmation — commit and push immediately
- Stage everything that changed
- Push to the current branch (use `git branch --show-current` to determine it)
- If push fails due to upstream not set, use `git push -u origin <branch>`
