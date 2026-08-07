You generate documentation from spec files and save it under `docs/`. Frontend docs are 分鏡圖 (storyboards), backend docs are 資料流程圖 (DFDs), DB docs are ER models.

**Diagrams only, as actual images — never raw Mermaid text in the doc.** Every doc is a title + a short (≤1 sentence) framing line + one or more rendered image(s). No prose walkthroughs, no per-field tables, no business-rule lists, and no `​```mermaid` code fence left in the final `.md` — a fenced code block is text, not a picture, and does not satisfy this command. Keep it to the big picture and the flow; it's fine to drop detail that doesn't serve that.

**Only the `solution-architect` agent (`.claude/agents/sa.md`) analyzes and drafts.** No other agent type may be used for this command. `solution-architect` is read-only (`Read, Glob, Grep` — no `Write`), so it returns the diagram(s) as Mermaid text; **you** (the command runner) render that text to an image and write the doc to disk — see "Rendering" below.

## Argument Handling

Check `$ARGUMENTS`:

- **Empty or blank**: regenerate every group in all three domains (`frontend`, `backend`, `db`)
- **`frontend` / `backend` / `db`**: regenerate only that domain's groups
- **A spec slug or table name** (e.g. `currency-pair`, `audit`): regenerate only the group that spec/table belongs to, across whichever domain(s) it appears in

## Process

1. **Read** every non-`skip` spec in `specs/frontend/`, `specs/backend/`, `specs/dba/` in scope for the argument above (skip files with `status: skip` — nothing to document).
2. **Group** specs within each domain — see "Grouping Rule" below.
3. **Dispatch**: for each group, spawn an Agent with `subagent_type: "solution-architect"` (see "Dispatching" below for the exact prompt shape — it must override the agent's own default 12-section template).
4. **Render**: `solution-architect` cannot write files, and returns Mermaid *source*, not an image. For each Mermaid block in its response, write the source to a temp `.mmd` file and render it to a PNG with Mermaid CLI (see "Rendering" below), saved into `docs/<domain>/`.
5. **Write**: create `docs/<domain>/<group-slug>.md` per "File Shape" below — title + framing sentence + `![...](image.png)` for each rendered image. Never paste the Mermaid source itself into the `.md`.
6. **Regenerate** `docs/README.md` (see "Index" below) once all groups are written.
7. **Print** the summary table (see "Output" below).

## Grouping Rule

Group specs by shared entity/table **and** workflow cohesion — not by blindly taking the transitive closure of `depends_on` (backend/frontend) or every FK edge (DB). A hub spec/table that's merely *referenced* by many others (e.g. `brand`, `currency`, `audit` as a generic reusable module) does not pull every referencer into one group. A linear pipeline where specs exist specifically to extend each other (e.g. define → fan-out → approve) does merge into one group with one diagram.

Concretely, using judgment on the current spec set:

| Group slug | Frontend | Backend | DB |
|---|---|---|---|
| `currency-pair` | currency, currency-pair, currency-pair-definition, currency-pair-approval | currency, currency-pair, currency-pair-definition, currency-pair-approval | currency, currency_pair, currency_pair_definition |
| `brand` | brand | brand | brand, spread_default, spread_group, spread_group_member |
| `spread` | spread | spread | *(spread tables live in the `brand` ER diagram — they're FK-clustered with `brand`/`currency_pair`, splitting them would drop the relationship lines)* |
| `audit` | audit | audit | audit_request *(deliberately FK-isolated — its own spec forbids referencing any consumer)* |

For DB specifically: draw **one ER diagram per FK-connected cluster** — that's simply the correct way to draw a connected schema. Do not split a single connected cluster across multiple files just to mirror frontend/backend slugs 1:1; where a cluster spans what frontend/backend call two slugs (e.g. `brand` cluster includes `spread_*` tables), pick the slug of the cluster's anchor table (the one others FK to) as the filename, and note in the doc which other slugs' tables it also covers.

Cross-cutting specs that don't introduce a distinct screen/flow/table of their own (e.g. a nav-reorder spec, a pure restyle spec) are **not** turned into their own doc — list them in `docs/README.md`'s index instead, with a one-line note on what they touched.

When new specs are added later, fold them into an existing group if they extend that group's workflow/entity, or create a new group if they're genuinely a new feature area — same judgment call `/spec` uses to decide "update existing spec vs. create new file".

## Dispatching

`solution-architect`'s own system prompt always produces a 12-section report (Feature Summary, Actors, Context Diagram, Level 0/1 DFD, Business Flow Explanation, API Mapping, Business Rules, Error Scenarios, Specification Observations, Executive Summary). For `/doc`, override that explicitly — the prompt must say so or the agent will default back to its full template.

Agent prompt format:
```
For THIS task only, ignore your default 12-section output structure. Produce ONLY:
1. A title line for this group.
2. At most one sentence of framing (what this group covers, why these specs are documented together) — no Feature Summary, no Actors/Data table, no Business Flow Explanation, no API Mapping, no Business Rules list, no Error Scenarios table, no Specification Observations, no Executive Summary.
3. The diagram(s) themselves, as Mermaid code block(s).

Diagram type for the "<domain>" domain: <frontend: a flowchart of screen-to-screen navigation (nodes = screens/modals, edges = the user action that triggers the transition) | backend: a DFD per your usual rules (External Entity / Process / Data Store, business-oriented process names, no Controller/Service/Mapper) | db: a Mermaid erDiagram with entities, key columns (PK/FK), and relationship cardinality>

Still apply your existing rules: current-state-only (ignore historical/superseded behavior), never invent anything not in the specs, business terminology only, Traditional Chinese.

Read these spec files fully:
<file paths in the group>

Return only the title + framing sentence + diagram(s) — nothing else.
```

### Rendering

`solution-architect` returns Mermaid *source* inside ` ```mermaid ` fences — this is a draft, not the deliverable. For each fenced block in the group's response:

1. Write the Mermaid source (the fence contents only) to a scratch file, e.g. `/tmp/doc-render/<domain>-<group-slug>[-<n>].mmd` (append `-context`/`-level0`/`-1`/`-2`… if a group has multiple diagrams).
2. Render it with Mermaid CLI via npx (no local install required):
   ```
   npx --yes -p @mermaid-js/mermaid-cli mmdc -i <scratch>.mmd -o docs/<domain>/<group-slug>[-<n>].png -b white -w 1400 --scale 2
   ```
3. If `mmdc` reports a parse error, the Mermaid source itself is invalid (e.g. unescaped parentheses/special characters inside a piped edge label) — fix the offending line in the `.mmd` (simplify the label, don't drop the diagram) and re-render. Never ship a fenced-code fallback because rendering failed.
4. Read the rendered PNG back (e.g. with the Read tool) to sanity-check it actually looks like the intended diagram — text not clipped, no obviously broken layout — before referencing it in the doc.

### File Shape

Write `docs/<domain>/<group-slug>.md` as: the agent's title line, its one-sentence framing, then one `![...](<group-slug>[-<n>].png)` per rendered image (use a `## Context Diagram` / `## Level 0 DFD` sub-header per image only when a group has more than one diagram). Do not include the Mermaid source anywhere in this file.

### Parallel Rules

- Different groups, same or different domain → MAY run in parallel (no shared state between doc files)
- All dispatch happens after grouping is finalized for the current run — don't dispatch a group before you know its final spec list

## Index

Regenerate `docs/README.md`:

```markdown
# Docs

Generated documentation derived from `specs/`. Regenerate with `/doc` after specs change — these files are derived output, not hand-maintained. Diagrams are rendered as actual images (PNG), drafted by the `solution-architect` agent (`.claude/agents/sa.md`) and rendered via Mermaid CLI — big picture and flow, not exhaustive detail.

## Frontend (storyboards)
- [<group-slug>](frontend/<group-slug>.md) — <one-line: screens covered>

## Backend (data flow)
- [<group-slug>](backend/<group-slug>.md) — <one-line: flow covered>

## DB (ER models)
- [<group-slug>](db/<group-slug>.md) — <one-line: tables covered>

## Not documented separately
- `<spec-slug>` (<domain>) — <why: cross-cutting, no distinct screen/flow/table of its own>
```

## Output

After all groups are generated, print:

```
| Domain   | Doc                          | Specs covered                                              |
|----------|-------------------------------|-------------------------------------------------------------|
| Frontend | docs/frontend/currency-pair.md | currency, currency-pair, currency-pair-definition, ...      |
| Backend  | docs/backend/currency-pair.md  | currency, currency-pair, currency-pair-definition, ...      |
| DB       | docs/db/currency-pair.md       | currency, currency_pair, currency_pair_definition            |
...
```

Do NOT ask for confirmation at any step. Generate everything immediately.
