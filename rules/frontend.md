# Frontend Rules

## Scope

What belongs in `specs/frontend/`: UI components, pages, forms, API calls, user interactions, validation display, routing.

## Standing Rules

- **Colors must match `docs/frontend/` exactly, and must not shift with `prefers-color-scheme`.** If a group's storyboard already exists (`docs/frontend/<group>.md` / `docs/frontend/<group>/storyboard.png`), the spec's colors are not free choices — they're read off that storyboard's actual hex values. When writing or updating a `specs/frontend/<slug>.md`, include a `## Visual Style` section listing every distinct color used (background, text, borders, interactive states) as a literal hex value, one row per element — see `specs/frontend/brand.md`'s `## Visual Style` table for the shape. Never source page/table/label text color from a theme variable or token that changes value under `prefers-color-scheme: dark` or any other OS/browser preference; a spec's page has one fixed appearance regardless of the viewer's system theme.
  - Why this rule exists: `specs/frontend/brand.md` originally shipped with no explicit color spec, so the implementation picked CSS variables that flipped under dark-mode preference, making the table text render near-invisible (white-on-white) whenever the browser reported a dark `prefers-color-scheme` — a real, user-visible bug that only surfaced after `/dev` had already marked the spec `done`.
