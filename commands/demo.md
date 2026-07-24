You dispatch rapid UI prototyping work to the `demo` subagent — a UI/UX designer who only ever produces static **HTML + CSS + vanilla JS** (jQuery via CDN is fine if it speeds things up). No frameworks, no build step, no backend. Purpose: quickly mock up screens for visual review, disposable and fast.

This is intentionally exempt from the project's normal Spring Boot / Maven / test / CircleCI rules in `CLAUDE.md` — those apply to real feature work (`specs/` → `/dev` → `develop/backend`, `develop/frontend`). `/demo` is a separate, throwaway sandbox.

## Scope

- All output lives under `demo/` at the project root
- Entry point: `demo/index.html`
- Supporting files as needed: `demo/style.css`, `demo/script.js`, and additional `demo/<page>.html` files for extra screens
- No npm, no package.json, no TypeScript, no bundler, no React/Vue/Angular — plain `<script>`/`<link>` tags only (a jQuery CDN `<script src>` is allowed)
- **Never** touch `develop/backend/`, `develop/frontend/`, `specs/`, `docker/`, or `env.md` — this command is fully isolated to `demo/`, both for writes and for reference
- **Only look inside `demo/` for context.** Do not read, grep, or search `develop/`, `specs/`, `docker/`, other project docs, or any file outside `demo/` — not even for style/consistency inspiration. If the user's request references an existing screen, mockup, or pattern, treat that as a screenshot/description they've given you or as whatever already exists under `demo/`, never as a cue to go inspect the real app's source
- No real API calls — use inline mock/static data in the JS

## Preview

- `demo/` is always previewed on a fixed local port: **8099**, served from the `demo` config in `.claude/launch.json` (`python3 -m http.server --directory demo 8099`).
- To preview, use the Browser pane's `preview_start` with `{"name": "demo"}` — do not use an IDE's built-in server (e.g. JetBrains' port 63342) or a random/ephemeral port; keeping it fixed at 8099 means the same URL always works across sessions.
- URL: `http://localhost:8099/index.html` (or `http://localhost:8099/<page>.html` for other screens).

## Argument Handling

Check `$ARGUMENTS`:

- **`start`**: begin a demo session — **take no other action**. Do not create `demo/`, do not spawn the subagent, do not scaffold anything.
  - Print: `🎨 Demo session started. Editing demo/ until you say "demo end".`
  - From this point on (per `CLAUDE.md`'s DEMO MODE note), every subsequent user message — even without a `/demo` prefix — is routed here as if it were the "Anything else" case below, until the user says "demo end"
- **`end`**: close the demo session
  - Print a short summary of what's currently in `demo/` (files present, pages if more than `index.html`)
  - Print: `✅ Demo session ended.`
  - Do not delete or modify anything, and do not dispatch the subagent — closing is just an acknowledgment
- **Anything else**: spawn the `demo` subagent with `subagent_type: "demo"` to treat it as an instruction to modify the demo screen(s)
  - If `demo/` doesn't exist yet, the subagent creates the scaffold first, then applies the request
  - Agent prompt format:
    ```
    Execute the following demo request. Read the existing files under `demo/` first (if any) to keep the new/changed screen visually consistent with what's already there. Do not ask questions — proceed with your best design judgment, drawing on established UI/UX patterns.

    IMPORTANT: All output goes in `demo/` at the project root — index.html as the entry point, plus style.css/script.js/additional pages as needed. Never touch develop/backend/, develop/frontend/, specs/, docker/, or env.md. This isolation cuts both ways: only read/reference files inside `demo/` — do not read, grep, or search any file outside `demo/` for context, style, or consistency, even if the request mentions an existing screen or pattern. Rely only on what's in `demo/` plus the request itself (including any screenshot/description given in chat).

    SCOPE: Build only what is explicitly requested below — the named element/screen, styled well and consistent with existing work. Do not add extra buttons, links, toasts, or interactions beyond the request, even if a real-world reference has them. When less vs. more is ambiguous, do less.

    Request: <the user's natural-language request>
    ```

## Rules

- Do NOT ask for confirmation. Execute immediately.
- Do NOT write tests, do NOT add a CircleCI job, do NOT run linters/build tools — none of that applies here
- Do NOT use Maven, Spring Boot, or any backend framework — if the user wants a real backend, that belongs to `/spec` + `/dev`, not `/demo`
- If the user's request clearly needs a real backend, persistence, or a production frontend stack, say so and point to `/spec` instead of forcing it into `demo/`
