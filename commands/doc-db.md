You generate a polished **ER model** document from `specs/dba/`, saved as `docs/db/er-model.md` with its images in a sibling folder, `docs/db/er-model/` — never flat alongside other docs' images in `docs/db/` directly. It always opens with one **全景圖 (panorama)** — the whole schema as a styled flowchart map, grouped into pastel-tinted regions by major function — then breaks down into **one full-detail `erDiagram` per major function (FK-cluster)**. The panorama gives the big picture in one glance; the per-function diagrams are where each table's actual columns and relationships get explained.

This is a different artifact from `/doc` (`.claude/commands/doc.md`). `/doc`'s DB output is diagrams-only, one small `.md` file per cluster (`docs/db/brand.md`, `docs/db/currency-pair.md`, ...) — narrow, business-audience diagram fences, no styling, no unifying overview. `/doc-db` is the richer artifact: **one consolidated `.md`** that starts wide (panorama) and then walks through every function in turn with full column/PK/FK/cardinality detail, so a reader gets both the map and the detail from one file. The two commands' outputs coexist in `docs/db/` without colliding (different filenames).

Ignore `$ARGUMENTS` — always regenerate everything. Re-running this command is cheap (one Agent dispatch, ~8 tables today).

Only the `solution-architect` agent (`.claude/agents/sa.md`) analyzes and drafts. It is read-only (`Read, Glob, Grep` — no `Write`), so it returns Mermaid text; **you** (the command runner) render that text to images and write the doc to disk — see "Rendering" below.

## Clustering

Use the same FK-connected clusters `/doc`'s Grouping Rule (`.claude/commands/doc.md`) already establishes for the DB domain — don't re-derive a different grouping. For the current spec set that resolves to:

| Cluster slug | Tables (full detail) |
|---|---|
| `currency-pair` | currency, currency_pair, currency_pair_definition |
| `brand-spread` | brand, spread_default, spread_group, spread_group_member |
| `audit` | audit_request *(deliberately FK-isolated per its own spec — no relationship lines to anything)* |

When a table's own FK reaches outside its cluster (e.g. `spread_group_member.currency_pair_id` → `currency_pair`, which lives in the `currency-pair` cluster; `currency_pair.brand_id` → `brand`, which lives in `brand-spread`), that outside table still appears in this cluster's **detail** diagram — but only as a **context entity**: just its name and PK, dashed border, no other columns — so the relationship line is visible without duplicating full column detail across two diagrams. See "Diagram Style" below. The panorama has no such distinction — every table there is name-only regardless of cluster.

When new tables are added later, fold them into an existing cluster if FK-connected to it, or add a new cluster row here if they form their own connected component — same judgment call `/doc` uses.

## Process

1. **Read** every non-`skip` spec in `specs/dba/`.
2. **Resolve clusters** per "Clustering" above.
3. **Dispatch**: one `Agent` call, `subagent_type: "solution-architect"`, covering every spec in a single prompt (see "Dispatching" below — override its default 12-section template) — one dispatch produces the panorama and every cluster's diagram together so entity naming/coloring stays consistent across all of them.
4. **Render the panorama**: `<diagram id="overview">` → `docs/db/er-model/0.png` (see "Rendering" below). Read it back.
5. **Panorama split check** (see "Panorama Split Rule" below): if it reads cleanly as one image, keep it as `er-model/0.png` and move on. If not, split it into one simplified overview per **group of clusters** (not per single table) — see that section for exactly when and how.
6. **Render each cluster's detail diagram**: for each `<diagram id="<cluster-slug>">`, render to `docs/db/er-model/<cluster-slug>.png`. Read each PNG back to sanity-check legibility (no overlapping boxes, no clipped column text) — if a cluster's own detail diagram is too dense, that's a signal the cluster itself should be split further per "Clustering" above, not a case for merging clusters back together.
7. **Write** `docs/db/er-model.md` per "File Shape" below.
8. **Index**: add/update a `## ER Model (full schema)` section in `docs/README.md`.
9. **Print** the summary table (see "Output" below).

## Dispatching

`solution-architect`'s own system prompt defaults to a 12-section per-spec report. Override that explicitly — the prompt must say so or the agent falls back to its default template.

Agent prompt format:
```
For THIS task only, ignore your default 12-section per-spec template. You are producing an ER model covering every table below, organized into these FK-connected clusters (= major functions) — read every spec fully before writing anything:

<cluster table from "Clustering", e.g.:>
- currency-pair: currency, currency_pair, currency_pair_definition
- brand-spread: brand, spread_default, spread_group, spread_group_member
- audit: audit_request (FK-isolated — no relationships to any other table)

Produce exactly this structure, in Traditional Chinese prose (table/column names stay as their real physical names — snake_case, not translated; an ER diagram is the physical schema, unlike a DFD):

1. A title line and one paragraph (2-3 sentences) of overview: what the database as a whole covers, and the major functions (clusters) it splits into — this is the reader's first orientation, written for someone who hasn't seen any of the tables yet.
2. One Mermaid **`flowchart LR`** (not `erDiagram` — a plain ER box grid reads as a spec, not a map; a flowchart lets the panorama actually look designed) tagged [[DIAGRAM:overview]] (source wrapped separately in <diagram id="overview">...</diagram>), built exactly like this:
   - One `subgraph <ClusterId>["<cluster 中文名>"]` per cluster (`direction TB` inside each), containing every table in that cluster as a stadium-shaped node: `table_name(["table_name<br/>一句話中文角色"])`.
   - Every WITHIN-cluster FK as a plain `-->` edge between the two specific table nodes, no label (the detail diagram carries the label).
   - Every CROSS-cluster FK as a single labeled edge between the two **subgraph container IDs themselves** (e.g. `BS -->|品牌擁有幣種對| CP`), never between two individual nodes in different subgraphs — Mermaid's layout engine routes node-to-node edges across subgraph boundaries ambiguously (they visually land on the boundary, not the actual node), so anchoring at the subgraph level is the only way this stays legible. One short Traditional Chinese verb phrase per cross-cluster edge (e.g. `幣種對可納入點差群組`).
   - `classDef` one saturated fill+stroke color per cluster applied to that cluster's table nodes (reuse identically in that cluster's own detail diagram later — this is what lets a reader visually jump from panorama to detail), plus a `style <ClusterId> fill:...,stroke:...,stroke-width:1px` soft pastel background (a lighter tint of the same hue) on the subgraph itself so each function reads as a distinct region.
   - A `%%{init: {'theme': 'base', 'themeVariables': {'fontSize': '18px', 'fontFamily': 'trebuchet ms, verdana, arial', 'edgeLabelBackground': '#ffffff'}, 'flowchart': {'curve': 'basis'}}}%%` directive as the very first line.
   - This diagram is a map, not a spec: no table columns, no PK/FK markers — resist the urge to add them back in.
3. For EACH cluster, in order: a short (1 sentence) framing of what the cluster/function covers, then one Mermaid `erDiagram` tagged [[DIAGRAM:<cluster-slug>]] (source wrapped in <diagram id="<cluster-slug>">...</diagram>). Requirements for each cluster's diagram:
   - Every table IN this cluster as a full entity: every column listed with its type and `PK`/`FK` marker (Mermaid erDiagram syntax: `column_name type PK` / `type FK`).
   - Any table OUTSIDE this cluster that has a direct FK relationship to/from one of this cluster's tables appears too, but only as a context entity: just `id PK`, nothing else — mark it with a comment noting it's detailed in its own cluster's diagram.
   - Every relevant foreign key (within-cluster or to a context entity) as a relationship line with correct crow's-foot cardinality — derive from each spec's own cardinality language (e.g. "one brand has many pairs", "at most one membership row per pair").
   - A short verb/noun relationship label in Traditional Chinese on each line where it adds clarity (擁有 / 屬於 / 納入) — skip the label when the table names already make it obvious.
   - `classDef`/`class`: this cluster's own tables use the SAME color assigned to it in the overview diagram; any outside context entity uses a neutral gray dashed style regardless of which cluster it belongs to.
4. After all clusters' diagrams, one bullet per table (grouped under its cluster, in cluster order): table name, one-sentence purpose, and which FK guards apply (e.g. `ON DELETE RESTRICT`/`CASCADE`) — mention delete-guards in prose here, don't diagram them separately.

Still apply your existing rules: current-state-only (ignore historical/superseded columns), never invent a column/relationship not in the specs.

Read these spec files fully:
<file paths for every non-skip specs/dba/*.md>

Return the assembled markdown (title + overview paragraph + [[DIAGRAM:overview]] placeholder, then per-cluster framing + [[DIAGRAM:<slug>]] placeholders, then the grouped bullet list) followed by each diagram's Mermaid source wrapped in <diagram id="...">...</diagram> tags (one for "overview", one per cluster slug). Nothing else.
```

### Rendering

For each `<diagram id="...">` block in the response (`overview` and every cluster slug):

1. Write its Mermaid source to a scratch file, e.g. `/tmp/er-model-render/er-model-<id>.mmd`.
2. Render with Mermaid CLI via npx, into the doc's own image folder (create it first if it doesn't exist):
   ```
   mkdir -p docs/db/er-model
   npx --yes -p @mermaid-js/mermaid-cli mmdc -i <scratch>.mmd -o docs/db/er-model/<n>.png -b white -w 1600 --scale 2
   ```
   where `<n>` is `0` for `overview`, otherwise the cluster slug.
3. If `mmdc` reports a parse error, fix the offending Mermaid line yourself (a stray unescaped character in a label is the usual cause) and re-render — never fall back to a text fence.
4. If the installed Mermaid CLI version doesn't support `classDef`/`class` on `erDiagram` (added ~v10.5 — check with `npx --yes -p @mermaid-js/mermaid-cli mmdc --version` if a render fails specifically on those lines), drop the color styling and keep going; cluster identity is still conveyed by the doc's section headings/order. Don't fail the whole command over cosmetic color. (The panorama is a `flowchart`, not `erDiagram`, so this caveat doesn't apply to it.)
5. Read each rendered PNG back to sanity-check before referencing it: for the panorama, no overlapping boxes, subgraph background tints visually distinct per cluster, and every cross-cluster edge visibly anchored at a subgraph boundary (not floating mid-air or appearing to touch the wrong node); for detail diagrams, columns not clipped and context entities visibly muted/dashed vs. this cluster's own tables.

## Panorama Split Rule

The panorama (`er-model/0.png`) is illegible when, after reading it back, any of: entity boxes visually overlap, more than ~12 entities are crammed in, or relationship lines cross so densely that cardinality markers are unreadable. Table count alone across 3 small clusters is not a reason to split — only split when it actually renders badly.

If it *is* illegible, split it along cluster boundaries into **groups of clusters** (not down to per-table pieces — a per-cluster panorama fragment would just duplicate the detail diagrams one level up): pick a small number of panorama fragments (2-3, not one per cluster) each covering multiple related clusters, e.g. `er-model/0-data.png` (master/reference clusters) + `er-model/0-workflow.png` (process/audit clusters) — the exact split depends on which clusters are actually causing the crowding. Name them `er-model/0-<fragment-slug>.png`. Each fragment keeps the name-only, no-attribute style — this is still a map, just a smaller one.

## Diagram Style

- Panorama: a `flowchart`, one pastel-tinted subgraph region per cluster (soft `style` fill, thin matching-hue stroke), stadium-shaped table nodes colored with that cluster's saturated `classDef`, no columns/PK/FK. Within-cluster edges are plain unlabeled arrows between the two specific tables; cross-cluster edges are labeled arrows between the two subgraph IDs, never between individual nodes in different subgraphs (see Dispatching above for why). A table with zero relationships (e.g. `audit_request`) sits alone in its own subgraph with no edges touching it, anywhere.
- Detail diagrams: `erDiagram`, full attributes with PK/FK markers; this cluster's own tables use the identical color assigned to it in the panorama; any table from another cluster shown here is always neutral gray with a dashed border, regardless of which cluster it actually belongs to — the point is "not detailed here," not "which other cluster."
- Keep every relationship label (detail diagrams, and cross-cluster panorama edges) to one short phrase; omit it entirely on a within-cluster panorama edge or a detail-diagram edge where the two table names already make the relationship obvious.
- `audit_request`'s detail diagram has no context entities and no relationship lines by design — a single isolated box is the correct, accurate rendering of "this table deliberately has no FK to anything," not a diagram to pad out.

## File Shape

`docs/db/er-model.md`:
- The agent's title line and overview paragraph.
- `## 全景` — `![全景](er-model/0.png)` (or, if split, one `![...](er-model/0-<fragment-slug>.png)` per fragment, each captioned with which clusters it covers).
- One `## <cluster slug>` section per cluster, in the order from "Clustering": that cluster's one-sentence framing, then `![...](er-model/<cluster-slug>.png)`, then that cluster's grouped bullet list (table name / purpose / delete-guards) from the agent's response.

Never include Mermaid source or a `[[DIAGRAM:...]]` placeholder in the final file.

## Index

Add/update in `docs/README.md`:
```markdown
## ER Model (full schema)
- [er-model](db/er-model.md) — panorama of the whole schema plus one detail diagram per major function (currency-pair / brand-spread / audit), cross-cluster FKs shown via context entities
```

## Output

After the document is written, print:
```
| Doc                  | Section        | Tables (full detail)                                          | Context entities |
|-----------------------|-----------------|----------------------------------------------------------------|-------------------|
| docs/db/er-model.md   | 全景 (panorama) | all 8, name-only                                                | n/a               |
| docs/db/er-model.md   | currency-pair   | currency, currency_pair, currency_pair_definition               | brand             |
| docs/db/er-model.md   | brand-spread    | brand, spread_default, spread_group, spread_group_member         | currency_pair     |
| docs/db/er-model.md   | audit           | audit_request                                                    | (none)            |
```

Do NOT ask for confirmation at any step. Generate immediately.
