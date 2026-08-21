You are a technical architect. Analyze the following requirement and generate **three** spec files — one each for frontend, backend, and DBA — then, for whichever domains actually changed, chain into the matching doc command(s) (`/doc-fronend`, `/doc-backend`, `/doc-blue-print`, `/doc-db`) so a single `/spec` run can produce all five artifacts: spec, blueprint, frontend, backend, db (see "Doc Generation" below).

## Requirement

$ARGUMENTS

## Process

1. **Read the codebase** first to understand existing conventions, patterns, and structure
2. **Read `env.md`** to understand the tech stack — do NOT hardcode languages, frameworks, or connection info into specs
3. **Analyze** the requirement and split into three domain concerns
4. **Check for drift** (see "Keeping Specs Concise" below): if this requirement changes a feature that already has a spec, decide now whether to fold the change in directly or append a delta — before writing anything
5. **Generate/update** spec file(s) per domain and save to the `specs/` subfolder
6. **Verify cross-domain consistency** (see "Cross-Domain Consistency" below) before treating the run as finished
7. **Chain into doc generation** for whichever domains actually changed in this run — see "Doc Generation" below

## File Naming

Format: `specs/<domain>/<slug>.md`

Example: `specs/backend/audit-update.md`

## Spec File Format

```markdown
---
status: pending
title: "<feature title>"
requirement: "<original requirement summary>"
depends_on: [<slug>, <slug>]
---

# <Feature Title> — <Domain> Spec

## Overview
<What this spec covers and why>

## Requirements
<Detailed requirements for this domain>

## Implementation Details
<Specific implementation instructions>

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>
```

`depends_on` is **required on every backend and frontend spec** (omit it on DBA specs — see below) — a list of same-domain slugs that must reach `status: done` before this one can be built. `/dev` uses it to determine execution order within a domain, since filenames don't encode build order. Get this right now: `/dev` running "starting from a completely empty project" must reproduce the same result as running specs incrementally over time, and that only works if every spec honestly declares what it needs to already exist.
- Set it to `[]` (or omit entirely) only if the spec genuinely has no same-domain prerequisite (e.g. a foundational table/API like `brand`/`currency` with no FK/import dependency on anything else you're adding).
- List every same-domain spec whose code this one's implementation reads, imports, calls, or extends — not just the ones its own prose happens to mention. If in doubt, check what the target spec's own `Overview`/`Implementation Details` reuses (shared components, existing services, existing tables via FK) and depend on the specs that built those.
- DBA specs don't use `depends_on` — order there comes from the migration version number embedded in the spec's own content (`V0xx__description.sql`), which every DBA spec must already state. State clearly which existing `Vxxx` your new migration comes after.

## Domain Split Rules

Each domain has its own rules file — read the one relevant to each spec you touch, fully, before step 3 ("Analyze") and again before step 5 ("Generate/update"):

- `.claude/rules/frontend.md` — scope of `specs/frontend/`, plus standing rules (e.g. colors must match `docs/frontend/`'s storyboard, fixed regardless of theme preference)
- `.claude/rules/backend.md` — scope of `specs/backend/`, plus standing rules
- `.claude/rules/dba.md` — scope of `specs/dba/`, plus standing rules (one table per file, migration SQL lives only in the spec)

These are classification + standing-rule files, not implementation conventions (those live in `.claude/agents/{frontend,backend,dba}.md` instead) — so other commands can consult the same definitions without duplicating them, and each domain's rules can grow independently.

## Rules

- **Check for an existing spec first**: if `specs/<domain>/` already has a file covering the same feature/entity, update that file instead of creating a new one — see "Keeping Specs Concise" below for exactly how. Only create a new file for a genuinely new feature area.
- Set `status: pending` on all new specs
- If a domain has NO work needed, create the file with `status: skip` and note "No changes required"
- Be specific and actionable — specs go directly to developer agents
- **Do NOT include database connection info** (host, port, username, password, JDBC URL) in any spec
- **Do NOT assume or specify programming languages or frameworks** — the dev agents will read `env.md` for that
- Focus specs on **what** to build (data model, API contract, UI layout), not **how** (tech stack details)
- Include all field names, types, validation rules, API contracts, SQL statements
- Cross-reference between specs where there are dependencies
- For backend: include endpoint path, HTTP method, request/response JSON, validation, service logic flow
- For frontend: include page layout, components, API integration, form fields, error states
- For DBA: include full SQL statements, indexes, constraints, migration order

## Keeping Specs Concise (Consolidation)

Specs get read as "what does this feature do right now" — by you on the next `/spec` call, by `/dev`'s executor agents, by a human. Every time a change is layered on top as a separate "Delta:"/"Increment" note instead of being merged into the text it overrides, that question gets harder to answer: the reader has to hold both the old and new text in their head and reconcile them. The goal here is that a spec never requires reading its own history to know its current contract.

**Before writing anything, check whether the target spec's `status` is `pending` or `done` — they're handled differently:**

### Target spec is `status: pending` (not yet built)
Nothing has shipped against it yet, so there's no execution history to preserve mid-document. **Edit the file in place**: rewrite the affected `## Overview`/`## Requirements`/`## Implementation Details`/API-contract text to state the final, current behavior directly — do not add a new "## Delta"/"## Addendum" section layered on top of text it overrides. Replace superseded sentences; don't append qualifiers next to them. Update the frontmatter `requirement` field to summarize the merged requirement. Keep `status: pending`.

### Target spec is `status: done` (already built and shipped)
Its `## Execution Result` (and any `### Increment <n>`) is a factual record of what was actually built and why — **never rewrite or delete that history**. But the parts a reader consults to learn "what does this do today" (`## Overview`, `## Requirements`, the API contract / page layout / table definitions) should still say the current truth plainly, not "see the other file for the real answer":
1. **Prefer editing the same file's contract sections in place**, exactly like the `pending` case, immediately above its `## Execution Result`. Superseded response codes, fields, or behavior get replaced in the text, not annotated with a "this changed, see elsewhere" note.
2. **Append new, unchecked `## Acceptance Criteria` items** for whatever the new requirement adds, and set `status` back to `pending` so `/dev` executes only those (per `dev.md`'s "Incremental Specs & Consolidation") — this is what actually drives the new build work.
3. **Only create a separate companion spec** (e.g. `<slug>-approval.md`) when the new work is substantial enough to need its own Acceptance Criteria/Execution Result trail distinct from the original (a new plug-in module, a new audit handler, etc.) — and even then, still update the original file's contract sections per (1) so it doesn't go stale. A companion file should hold implementation-level "why" (design rationale, judgment calls) that doesn't belong in the base contract, not a second copy of "what" the base file already states.
4. If you find a feature already fragmented this way from past increments (a base spec whose contract text still describes superseded behavior, with the real current behavior only stated in a later companion file), fix it now as part of this pass: fold the companion file's contract-level facts back into the base file's text, and trim the companion file to keep only its non-redundant implementation detail and history.

### Signal to watch for
If you're about to write a sentence like "this supersedes X" or "note: Y no longer applies, see Z" — stop, and instead go edit X/Y directly to say the new truth. Qualifier-on-top-of-old-text is exactly the pattern that makes specs harder to read with each revision; the fix is always to edit the original statement, not to add a correction next to it.

## Cross-Domain Consistency (Doc Is Source of Truth)

The three domain specs describe one feature from three angles — they must agree on a single contract, not three independent guesses. Before treating any `/spec` run as done, cross-check every spec touched this run (and every existing spec it references via `depends_on` or shared table/entity) for:

- **Frontend ↔ Backend**: every API call the frontend spec makes — path, method, request body fields, response fields, status/error codes — must match the backend spec's endpoint contract exactly. Same field names and types on both sides; no field the frontend reads/sends that the backend spec doesn't define, and vice versa.
- **Backend ↔ DBA**: every field the backend spec's service/mapper/DTO reads or writes must match the DBA spec's column names, types, nullability, and constraints exactly. No backend field without a backing column; no silent type mismatch (e.g. backend treats a column as `String` but DBA spec defines it `ENUM`/`INT`).
- **Frontend ↔ DBA**: every screen field that ultimately displays or edits a stored value must trace to a real column in the DBA spec — no phantom fields invented on the frontend that neither backend nor DBA account for.

**If a mismatch is found, the fix always goes into the spec doc(s) first — doc is authoritative, not code:**
1. Edit the spec doc(s) so all three domains state the same single, current contract (apply the "Keeping Specs Concise" rules above — edit the contract text in place, don't layer a correction note on top).
2. If code has already been built (`status: done`) and disagrees with the corrected doc, the code is what's wrong. Append the fix as new unchecked `## Acceptance Criteria` items on the affected domain spec(s) and set `status: pending` so `/dev` brings the code in line with the doc — never resolve a mismatch by rewriting the spec to match whatever the code currently does.
3. This check runs on **every** `/spec` invocation, not only when the current requirement obviously spans multiple domains — a change scoped to one domain (e.g. renaming a backend response field) can silently break its contract with an already-existing spec in another domain (e.g. the frontend spec that still reads the old field name). Check specs the current change touches indirectly, not just the ones it creates/edits directly.

## Doc Generation

`/spec` is the single entry point for all five artifacts a requirement can produce: the spec itself, plus the four downstream docs — blueprint, frontend storyboard, backend API doc, DB ER model. The four doc commands (`.claude/commands/doc-fronend.md`, `doc-backend.md`, `doc-blue-print.md`, `doc-db.md`) still exist and still work standalone (a user can run `/doc-backend spread` directly at any time) — this step just makes `/spec` chain into them automatically, scoped to **only the domains this run actually touched**. Never regenerate a domain's docs because another domain changed, and never regenerate a domain whose spec file wasn't actually created or edited in step 5 (a spec that stayed `status: skip`, or that you checked and found didn't need changes, does not count as touched).

For each domain with at least one spec file created/edited this run, follow the referenced command's own file **exactly as written there** (its own Argument Handling, Process, Dispatching, Rendering, Index, Output) — this section only decides *whether* to trigger it and *what scope to pass*, never how it works internally, so behavior stays identical to running that command by hand:

- **Frontend spec(s) changed** → for each affected group under `/doc-fronend`'s Grouping Rule (`.claude/commands/doc-fronend.md`), run its Process with that group slug as scope. Two changed specs landing in the same group only trigger that group once.
- **Backend spec(s) changed** → two things:
  1. Run `/doc-backend`'s Process (`.claude/commands/doc-backend.md`) with scope = the changed slug(s) — one output file per changed spec.
  2. Run `/doc-blue-print`'s Process (`.claude/commands/doc-blue-print.md`) once — it always regenerates the single unified `docs/blueprint/backend.md` covering every non-`skip` backend spec (not just the changed ones), so there is nothing to scope here; just trigger it once per `/spec` run where at least one backend spec changed.
- **DBA spec(s) changed** → run `/doc-db`'s Process (`.claude/commands/doc-db.md`). It always regenerates the whole ER model regardless of scope — that's fine, only the *trigger* (at least one `specs/dba/*.md` changed this run) is scoped, not its own internal behavior.

If a run touches zero domains for a given category, skip that category's docs entirely — don't touch its existing files.

## Output

After generating all files, print this summary table:

| Domain   | File                              | Status  |
|----------|-----------------------------------|---------|
| Frontend | specs/frontend/slug.md | pending |
| Backend  | specs/backend/slug.md  | pending |
| DBA      | specs/dba/slug.md      | pending |

Then, if any doc generation ran (per "Doc Generation" above), append a second table listing what was produced:

| Artifact  | Command       | File(s)                              |
|-----------|---------------|---------------------------------------|
| Blueprint | /doc-blue-print | docs/blueprint/backend.md            |
| Frontend  | /doc-fronend    | docs/frontend/<group>.md             |
| Backend   | /doc-backend    | docs/backend/<slug>.md               |
| DB        | /doc-db         | docs/db/er-model.md                  |

Omit any row for a category that didn't run this time.

Do NOT ask for confirmation. Generate all three specs, then chain into doc generation, immediately.
