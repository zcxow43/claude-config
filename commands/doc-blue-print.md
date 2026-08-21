You generate **one consolidated, big-picture, diagram-led** integrated architecture document covering every backend spec, and save it under `docs/blueprint/`.

This is the big-picture layer only — it deliberately does **not** carry field-level or rule-level detail anymore. `/doc-backend` is where that detail lives, one file per spec topic (`docs/backend/<slug>.md`: field tables, constraints, cross-topic rules, full API list). `/doc-blue-print` stacks *every* spec into one system-level document built around diagrams: a full architecture diagram, per-entity excerpt/flow diagrams, and cross-spec end-to-end scenario walkthroughs — every entity chapter ends with a pointer into its `/doc-backend` file for the detail. Reference shape: an externally-provided "Exchange Rate Management — Integrated Architecture & Stackable Specification" PDF that stacked `brand`/`currency`/`currency-pair-definition`/`currency-pair`/`currency-pair-approval`/`spread`/`audit` into one file — the diagram/scenario portions of that shape are what to reproduce here (its field tables and matrices now belong in `/doc-backend`).

Backend only for now (`specs/backend/`).

**There is always exactly one blueprint, never one per feature or per connected component.** Confirmed by the user after seeing per-component fragmentation in practice: two unconnected entities (no shared `depends_on` edge, e.g. a master-data table like `brand` and a standalone CRUD entity like `currency`) still belong in the *same* single picture — the point of a "blueprint" is one panorama of the whole system, not a gallery of one-entity diagrams that happen to sit in separate files. Two entities with no relationship to each other simply render as two unconnected clusters inside that one diagram (never invent a relationship that isn't in the specs) — they do not get split into separate documents. This mirrors `/doc-db`'s own panorama-first design.

## Language & Terminology

This document is read by non-engineers (PM/QA/business stakeholders) as well as engineers — confirmed by the user. Every label, in diagrams and in prose, must be plain business Traditional Chinese, not developer jargon. This applies everywhere, not just diagrams.

Banned as a standalone label (translate the *meaning*, not the syntax):
- HTTP verbs/status codes (`GET`/`PUT`/`POST`/`DELETE`, `200`/`201`/`202`/`204`/`400`/`404`/`409`) — say what happens instead: `409` → 拒絕並說明原因; `202` → 已送出，等待審核。A status code may still appear in parentheses for an engineer cross-reference, but never as the only label.
- Raw function-call syntax (`submit(entityType, actionType, entityId, after)`, `snapshotOf(entityId)`, `handler.apply`) — describe the action in a sentence: `snapshotOf` → 記錄異動前的原始資料; `apply` → 真正執行變更; `validate` → 檢查是否合法.
- `快照`（snapshot）— say what it actually is: 異動前的資料 / 保留的原始資料 / 目前的樣子. Same for other implementation nouns like `entity`/`entityType`/`payload` — say 這個資料 / 這一筆.
- Raw enum constants as the only text (`PENDING`/`APPROVED`/`REJECTED`, `AUTO`/`MANUAL`) — pair with or replace by the Chinese state name already used elsewhere in the doc (待審核/已核准/已拒絕), for consistency.

Label brevity (this is not optional — it is what makes the previous rule renderable):
- Keep every node/edge label to one short line wherever possible. A concept that needs two clauses is two separate short labels on two edges, not one label with a line break.
- Never use a literal `\n` escape sequence inside a Mermaid label — most diagram types (`stateDiagram-v2` especially) render it as literal text, not a line break, and it collides with neighboring labels. If a label must wrap, use an actual newline in the `.mmd` file (flowchart node text) or just shorten it instead (`stateDiagram-v2` edge labels).
- After rendering, if two labels visually overlap (read the PNG back per "Rendering" step 5), the fix is shortening text or switching `direction` — never leave it as shipped.

## Argument Handling

Ignore `$ARGUMENTS` — always regenerate the single unified blueprint covering every non-`skip` spec in `specs/backend/`. Re-running this command is cheap (one `Agent` dispatch), so there is no narrower-scope mode to choose between; if you want to view just one entity's detail, that's what `docs/backend/<slug>.md` (`/doc-backend`) is for.

Output slug is always `backend`. Output file: `docs/blueprint/backend.md`. Its images live in a sibling folder, `docs/blueprint/backend/` — never flat alongside other docs' images in `docs/blueprint/` directly (see "Rendering" and "Local Excerpt Diagrams" below for exact filenames).

## Process

1. **Read** every non-`skip` spec in `specs/backend/`.
2. **Dispatch**: one `Agent` call, `subagent_type: "solution-architect"`, covering *every* spec in a single prompt (see "Dispatching" below) — the point of a blueprint is the cross-spec integrated view, so do not fragment this into one call per entity, and do not split entities across multiple output files just because some of them have no relationship to each other.
3. **Render**: the agent returns Mermaid diagram blocks tagged with placeholders (`[[DIAGRAM:n]]`). For any diagram depicting a cross-layer handoff (see "Diagram Style — Cross-Layer Handoff Flows" below), rewrite it into that colored-box flowchart shape yourself before rendering — the agent may draft it as a `sequenceDiagram`, but that is not the final shape. For each, write the (possibly rewritten) Mermaid source to a scratch `.mmd` file and render to PNG with Mermaid CLI (see "Rendering" below) into `docs/blueprint/backend/`.
4. **Derive chapter-opener excerpts**: once Diagram 1's final Mermaid source is locked in, mechanically crop one small "local view" per §3 chapter from it — see "Local Excerpt Diagrams — Chapter Openers" below. This is a render-time derivation you do yourself; it is not something you ask the agent for.
5. **Assemble**: take the agent's returned markdown body and replace each `[[DIAGRAM:n]]` placeholder with `![...](backend/n.png)`. Insert each chapter's excerpt image (from step 4) immediately after that chapter's `### 3.x` heading, before its "說明" sentence and Field table. Never leave a `[[DIAGRAM:n]]` placeholder or a raw ` ```mermaid ` fence in the final file.
6. **Index**: add/update a `## Blueprints` section in `docs/README.md`.
7. **Print** the summary table (see "Output" below).

If a previous run left per-entity fragments (`docs/blueprint/<other-slug>.md` files from before this command consolidated to a single output), delete them and their image folders as part of this run — there is only ever one blueprint file now.

## Dispatching

`solution-architect`'s own system prompt defaults to a 12-section per-spec report (Feature Summary, Actors, Context Diagram, Level 0/1 DFD, Business Flow Explanation, API Mapping, Business Rules, Error Scenarios, Specification Observations, Executive Summary). Override that explicitly for this command — the prompt must say so or the agent falls back to its default template.

Agent prompt format:
```
For THIS task only, ignore your default 12-section per-spec template. You are producing ONE integrated, big-picture document that stacks all of the specs below into a single system-level blueprint — read every one of them fully before writing anything, and treat their union (not any single file) as "the system". This document is diagram-led and big-picture only — field-level and rule-level detail belongs in the separate per-topic backend docs (`docs/backend/<slug>.md`), not here.

Produce exactly this structure, in Traditional Chinese, current-state-only (ignore superseded/historical behavior — same priority order you always use):

1. **標題 + 一段目的說明**：這份文件整合了哪些 spec、整合後系統現在的樣子（不是逐一介紹每份 spec 在做什麼）。

2. **完整架構圖**：一張 Mermaid flowchart，把所有 scope 內的 entity/data store 當節點，畫出主要關係／fan-out／cascade／audit 邊。標記為 [[DIAGRAM:1]] 取代整個 ```mermaid 區塊（保留 Mermaid 原始碼在你回覆的其他地方，用 <diagram id="1"> ... </diagram> 包起來，讓呼叫端能取出來渲染 — 其餘規則不變：不用 Controller/Service/Mapper 之類實作名詞，process 要有業務語意）。任何橫跨兩個以上 entity 的規則，都要用這張圖上的標籤邊表示出來，不要另外寫成表格。

3. **每個 entity/data store 一個小節**：3–5 句話說明這個 entity 的業務角色、以及它在整個系統大圖裡的位置（不要寫 Field 表或逐條 Constraints——那些屬於 `docs/backend/<slug>.md`）。若該 entity 的 create/update/delete 邏輯不單純（例如 fan-out、delete guard、audit 路由），仍然加一張局部 Mermaid 圖，標記為 [[DIAGRAM:2]]、[[DIAGRAM:3]]... 依序遞增，原始碼同樣用 <diagram id="n"> 包起來——圖表保留，細節表格移除。每個小節最後一行固定加上：`詳細欄位、規則與 API 定義 → docs/backend/<slug>.md`（`<slug>` 對應這個 entity 所屬的 spec 檔名）。

4. **通用／可重用模組小節**（如果 scope 內有，例如 audit）：說明它的 contract 設計（entityType/snapshot/validate/apply 之類），強調它對其他 entity 一無所知。同樣以一段說明為主，不需要表格。

5. **End-to-End 情境走查**：只挑會跨越兩個以上 entity、或揭露非顯而易見規則的情境（例如「建立 Global Definition 後 fan-out 到每個品牌」「刪除 Definition 前必須先確認沒有品牌還在使用」「送審後被駁回」）——這些才是讀者看了架構圖後還想知道「實際流程長怎樣」的地方。**不要**把一般 REST CRUD 的例行步驟寫成情境（例如「新增時驗證必填欄位」「查詢不存在的 id 回 404」「更新時忽略不可變欄位」）——這些屬於 `docs/backend/<slug>.md` 的 Acceptance Criteria，本文件寫出來只是重複、沒有提供架構層級的新資訊。若某個 entity/spec 純粹是單一資料表的標準 CRUD、沒有任何跨 entity 效應或非顯而易見規則（例如一個獨立維護的主檔，新增/查詢/修改/刪除都各自獨立生效），這節可以完全不提到它——省略比硬湊瑣碎步驟更誠實。若整個 scope 裡沒有任何情境符合這個門檻，整節可以只寫一句話說明「目前系統內都是各自獨立的標準 CRUD，沒有跨 entity 情境」，不必硬掰 3–6 個。

6. **延伸閱讀**：表格，每個 §3 章節對應回哪個 backend 詳細文件（Chapter / `docs/backend/<slug>.md`），方便讀者從大方向跳去看細節定義。

Still apply your existing rules: never invent behavior not in the specs, business terminology only, distinguish pending audit requests from committed data, identify immutable fields and delete restrictions. This document is read by non-engineers as well as engineers — do not use HTTP verbs/status codes, raw function-call syntax, or words like "snapshot"/"entity" as labels; say what actually happens in plain business Traditional Chinese (see "Language & Terminology" above for the exact banned list and replacements).

Read these spec files fully:
<file paths in scope>

Return the assembled markdown body (with [[DIAGRAM:n]] placeholders inline) followed by each diagram's Mermaid source wrapped in <diagram id="n">...</diagram> tags. Nothing else — no extra sections, no restating your default template.
```

### Rendering

For each `<diagram id="n">` block in the response:

1. Write its Mermaid source to a scratch file, e.g. `/tmp/blueprint-render/backend-<n>.mmd`.
2. Prepend a dark-theme init directive so blueprint diagrams read as one visual family:
   ```
   %%{init: {'theme':'dark'}}%%
   ```
   (only if the returned source doesn't already start with an `%%{init...}%%` line).
3. Render with Mermaid CLI via npx, into the image folder (create it first if it doesn't exist):
   ```
   mkdir -p docs/blueprint/backend
   npx --yes -p @mermaid-js/mermaid-cli mmdc -i <scratch>.mmd -o docs/blueprint/backend/<n>.png -b '#0d1117' -w 1400 --scale 2
   ```
4. If `mmdc` reports a parse error, fix the offending Mermaid line (don't drop the diagram, don't fall back to a text fence) and re-render.
5. Read the rendered PNG back to sanity-check it before referencing it (not clipped, layout not broken).

### Diagram Style — System Overview (Diagram 1)

The one full-architecture diagram (always `[[DIAGRAM:1]]`) is a map, not a spec — confirmed by the user: more color, less exhaustive labeling than the per-entity diagrams.

- `classDef` one color per entity *group* (not per individual entity) and color every node's border + a dark tinted fill in that color, e.g. `classDef pair fill:#0d2a1b,stroke:#2ecc71,color:#ffffff,stroke-width:2px;`. Reuse the group boundaries already established in "Grouping Rule"-style thinking for this spec set — master data (Brand/Currency) one color, the pair system (Definition/Pair) a second, the spread system (Default/Group) a third, the generic audit store a fourth. Pick visually distinct hues (blue/green/orange/red work well on the dark background) — do not default to Mermaid's flat gray.
- **Identify 1–3 "core feature" entities and emphasize them, don't draw every entity as equal weight.** Core = the entities the *business* actually cares about and that carry the richest rules (usually the ones the audit module gates) — for this spec set that's Currency Pair and Spread (Default + Group), not the master data feeding them (Brand/Currency/Definition) or the generic Audit Request store. Give core entities a heavier `classDef` within their group's color (`stroke-width:4px` + `font-weight:bold`) versus `stroke-width:2px` for supporting/master/infra entities in the same or other groups. The reader's eye should land on the core features first — confirmed by the user: "點差、品牌幣種對是最主要的功能，應該由他們展開述說" (spread and currency-pair are the main features — the diagram should read outward from them, not treat them as one node among equals).
- **Omit pure join/membership entities as their own node.** If an entity exists only to link two others (e.g. Spread Group Member, which just records "this pair is in this group" with no independent rules beyond the FK pair) don't draw it as a box — collapse it into a direct edge between the two real entities it joins, labeled with the relationship (`納入幣種對`, not `包含` → `對應`). This was flagged by the user as a diagram that was one node too many for what it actually represents. Still describe the join entity's own rules (e.g. "one pair belongs to at most one group") in its per-entity prose section (§3) — omitting it from *this* diagram doesn't mean dropping it from the document.
- Do not color individual edges here (that's for the cross-layer-handoff diagrams below) — plain default edge color is correct; the color budget in this diagram is spent on nodes/groups, not arrows.
- Label only the structurally important edges (fan-out/lineage, "must go through audit", the group→default fallback) with one or two words (`送審`, `核准套用`) — an edge whose meaning is obvious from its node names (e.g. `品牌 → 預設點差`) does not need a label at all. Pick verbs from the same business domain as the rest of the document — a generic-fan-out edge is not `鋪貨` (a retail/logistics term with no connection to a financial back-office system); something like `自動建立` reads correctly regardless of domain. This diagram should read at a glance, not explain every rule — the per-entity sections already carry that detail.

### Local Excerpt Diagrams — Chapter Openers

Every §3 chapter opens with a crop of Diagram 1 centered on that chapter's own entity — confirmed by the user: "每個 chapter 開始之前，借用最一開始的畫面，取自己的 part，然後才是詳細解說" (before each chapter, borrow the opening picture, take its own part, then the detailed explanation). This gives the reader a "you are here" anchor inside the already-familiar overview before they drop into field tables and rules.

This is a **crop, not a redraw** — mechanically derived by you (the command runner) from Diagram 1's finalized Mermaid source, not a fresh diagram the agent invents:

1. For each entity node that gets its own §3 chapter, collect every edge in Diagram 1 where that node is either end (source or target), plus the node itself and every neighbor those edges touch.
2. Write a new **`flowchart LR`** (horizontal, not `TB`) containing exactly those nodes and edges — copy each node's label text and each edge's label/arrow-style (`-->`, `-.->`) **character-for-character** from Diagram 1. Do not rephrase, re-color, or re-derive anything; this must visually read as "the same nodes as the big picture," not a new illustration. **Always prefer horizontal over vertical for these excerpts** — confirmed by the user as a standing habit for this command, not a one-off fix: "能橫向就不要往下" (if it can go sideways, don't let it go downward). A short chain (e.g. 3 nodes in a line) rendered `TB` stacks into a tall column that still looks oversized even at a fixed embed width, because the width shrinks but the height doesn't; the same chain rendered `LR` becomes a short wide strip that reads as an actual thumbnail once constrained to `width="320"` in step 5. This applies regardless of how the corresponding subgraph happened to be oriented in Diagram 1 itself.
3. Copy over only the `classDef` lines whose class is actually used by the included nodes (no need to carry unused color definitions), and the matching `class` assignments.
4. Render it with the exact same `mmdc` parameters as Diagram 1 and every other diagram in the document (`-w 1400 --scale 2 -b '#0d1117'`, per "Rendering" above) — same scale, not smaller and not larger. Confirmed by the user: a node in the excerpt must look identical in size to that same node inside the full architecture diagram, since this is a crop, not a rescaled thumbnail. Do not special-case the scale for excerpts — the smaller final image size (fewer KB, smaller canvas) comes naturally from having fewer nodes in the crop, not from lowering the render scale. Name it `docs/blueprint/backend/1-<entity-slug>.png` (e.g. `docs/blueprint/backend/1-currency-pair.png`) — same image folder as every other diagram in this document; the `1-` prefix marks it as derived from Diagram 1, not a new diagram number in the main sequence.
5. Embed it with an HTML `<img>` tag, not Markdown `![]()` syntax: `<img src="backend/1-<entity-slug>.png" width="320" alt="...">`. This is required, not stylistic — plain `![]()` has no size attribute, so most Markdown viewers stretch the image to fill the column width regardless of its native pixel size, which silently defeats step 4's "same scale as Diagram 1" and makes the excerpt look full-size again (confirmed by the user seeing exactly this after the first `![]()` attempt). The fixed `width="320"` is what actually keeps it thumbnail-sized on screen; the render scale from step 4 is what keeps its *contents* crisp and proportionate at that size.
6. Place it as the very first line of the chapter's body, right after the `### 3.x 標題` heading and before the `**說明**` sentence and Field table. A one-line italic caption immediately below it (on its own line, with a blank line separating it from the `<img>` tag so Markdown still parses it as italic), `*（節錄自完整架構圖）*`, tells the reader this is an excerpt, not a new diagram.

A chapter's own detailed flow diagram (fan-out logic, the cross-layer audit handoff, etc.) still appears later in that chapter, after the Field table/Constraints, exactly as before — the excerpt is an addition at the top, never a replacement for it. Do this for every §3 chapter, even supporting/master-data ones whose excerpt is small (e.g. Brand's excerpt is just itself + the two nodes it feeds) — the anchor is the point, not the size.

### Diagram Style — Cross-Layer Handoff Flows

For any diagram whose subject is a request handing off across layers (e.g. Controller → Service → Handler → Data Store, or maker-submits → audit-service-dispatches → handler-applies), draw a **colored-box flowchart**, not a `sequenceDiagram`. This reads clearer for this project — confirmed by the user against a reference document. Shape:

- `flowchart TB` with two `subgraph` rows (no visible border — `style <SubgraphName> fill:transparent,stroke:transparent`), each `direction LR`: row 1 is the submit/maker path, row 2 is the approve/apply path.
- Every node is a box with a bold title line plus 1–3 short plain-language responsibility lines via `<br/>`, e.g. `["<b>審核服務</b><br/>保留異動前的資料<br/>檢查是否已有待審申請<br/>驗證申請內容"]` — per "Language & Terminology" above, never put raw method names (`snapshotOf`, `handler.apply`) in these lines.
- `classDef` three node colors and apply consistently across every such diagram in the document:
  - `actor` (gray/white border) — human actors (Maker/Admin, Reviewer).
  - `green` (green border) — the entity's own Controller/Handler (the feature-specific code).
  - `red` (red border) — the generic Audit Service / audit_request (or equivalent generic infra data store).
- Edges carry short plain-language labels (`送出申請`, `交由處理邏輯執行`, `建立待審申請`, `核准`, `核准後才套用變更`); color edges to match their destination's class via `linkStyle <index> stroke:#e74c3c,color:#e74c3c` (red-ish path) or `#2ecc71` (green-ish path) — leave plain white/default for the maker-facing steps.
- A cross-row dotted edge for any "reads live data, bypasses pending" relationship (e.g. `查詢一律讀取目前生效資料，不受待審申請影響`), styled green.
- 1–2 free-floating dashed-border note boxes (`classDef note fill:transparent,stroke:#555555,color:#aaaaaa,stroke-width:1px,stroke-dasharray: 3 3`) below the rows for the reject/re-validate edge-case prose, connected to the row above with an invisible link (`~~~`) purely for vertical positioning.

Reuse this exact `classDef`/style block verbatim across every cross-layer-handoff diagram in a document so the palette is consistent, not just per-diagram.

### Assembly

Take the agent's returned markdown body verbatim except:
- Replace each `[[DIAGRAM:n]]` token with `![Diagram n](backend/n.png)`.
- Immediately after each `### 3.x` chapter heading, insert that entity's excerpt image + italic caption from "Local Excerpt Diagrams" above, before the heading's existing first line.

Do not include any `<diagram>` tag or Mermaid source in the final `.md`.

Write the result to `docs/blueprint/backend.md`.

## Index

Add/update in `docs/README.md`:

```markdown
## Blueprints (integrated architecture specs)
- [backend](blueprint/backend.md) — <one-line: which specs are stacked>. Big-picture/diagram-led — see the linked `docs/backend/<slug>.md` files for field/constraint/API detail.
```

## Output

After the document is written, print:

```
| Blueprint                 | Specs integrated                                            | Diagrams |
|----------------------------|--------------------------------------------------------------|----------|
| docs/blueprint/backend.md | brand, currency, currency-pair, ...                         | N        |
```

Do NOT ask for confirmation at any step. Generate immediately.
