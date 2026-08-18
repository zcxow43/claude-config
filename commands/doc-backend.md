You generate **per-topic backend API documentation** from `specs/backend/`, one file per spec: `docs/backend/<slug>.md`.

This is the detail layer that `/doc-blue-print` deliberately no longer carries: `/doc-blue-print` is big-picture and diagram-led (system-wide architecture diagram, cross-spec end-to-end scenarios) — it does not carry field-level or rule-level detail anymore. `/doc-backend` is where that detail lives: for each topic, what its API does, its field/constraint definitions, any rule that reaches into another topic, and every URL it exposes. No diagrams here — diagrams are `/doc-blue-print`'s job, this command is prose + tables only. `/doc-backend` owns every `docs/backend/<slug>.md` file directly.

## Argument Handling

Check `$ARGUMENTS`:

- **Empty** or **`all`**: scope = every non-`skip` spec in `specs/backend/`.
- **A spec slug** (e.g. `spread`, `currency-pair`): scope = just that spec.
- Anything else that doesn't match a spec file: say so and stop.

Each spec in scope maps 1:1 to its own output file. Regenerating one slug only overwrites that slug's file — every other `docs/backend/<slug>.md` is untouched.

## Process

1. **Read** every non-`skip` spec in scope, fully — each spec's field/constraint definitions and "API Contract" (or equivalent) section are the source of truth.
2. **Dispatch**: one `Agent` call, `subagent_type: "solution-architect"`, covering every spec in scope in a single prompt (see "Dispatching" below — override its default 12-section template).
3. **Assemble**: write each spec's returned section to its own `docs/backend/<slug>.md` per "File Shape" below — no merging, no splicing into an existing file.
4. **Index**: add/update the entries in `docs/README.md` (see "Index" below).
5. **Print** the summary table (see "Output" below).

## Dispatching

`solution-architect`'s own system prompt defaults to a 12-section per-spec report (Feature Summary, Actors, Context Diagram, Level 0/1 DFD, ...). Override that explicitly — the prompt must say so or the agent falls back to its default template.

Agent prompt format:
```
For THIS task only, ignore your default 12-section per-spec template. For each spec below, produce exactly the sections listed, in Traditional Chinese, current-state-only (ignore superseded/historical behavior):

1. A short paragraph (3-5 sentences) explaining what this API does overall and its general approach — e.g. "查詢一律讀取目前生效資料"、"異動不會直接套用，需送審核准後才生效"。Do NOT describe implementation detail: no class/method names, no database column names, no raw function-call syntax, no HTTP status code used as a bare label (a status code may appear in parentheses next to its plain-language meaning, e.g. "拒絕並說明原因（409）", never alone). This paragraph is for a reader who wants to understand the shape of the API, not build it.

2. **欄位定義**：this spec's entity/entities, one Field table each (columns: | Field | Type/Role | Rule |). Business terminology for the Rule column, not raw validation code.

3. **限制條件**：a bullet list of this spec's own constraints (immutability, delete guards, uniqueness, etc.), one list per entity if it owns more than one.

4. **跨主題規則**：any rule in this spec that reaches into another spec/entity (e.g. "品牌停用時連動停用其下所有幣種對"). Write it from THIS topic's own angle, and end the bullet with a pointer to the other topic's file, e.g. "（見 currency-pair.md）". If this spec has no such rule, omit this section entirely (say so by simply not producing the heading).

5. **API 清單**：a table listing EVERY url this spec's API Contract section defines — do not skip or merge any, do not invent any not in the spec. Columns: Method | Path | 用途（一句話業務語言，不是函式或欄位名稱） | 送審分類（Direct / Audited / No API / Live direct，依這個 API 是否需要送審核准才生效判斷）.

Still apply your existing rules: never invent an endpoint, field, or behavior not in the spec, business terminology only in prose and business-facing columns (Method/Path keep their literal HTTP verb and route, that's the engineer-facing part of the table).

Read these spec files fully:
<file paths in scope>

Return, per spec, in this exact shape and nothing else (no extra sections, no restating your default template):
### <spec title>
<paragraph>

#### 欄位定義
<Field table(s)>

#### 限制條件
<bullets>

#### 跨主題規則
<bullets, or omit this heading entirely if none>

#### API 清單
| Method | Path | 用途 | 送審分類 |
|---|---|---|---|
| ... | ... | ... | ... |
```

## File Shape

`docs/backend/<slug>.md`:

```markdown
# <spec title> API

<paragraph>

## 欄位定義
<Field table(s), one per entity>

## 限制條件
<Constraints bullets>

## 跨主題規則
<cross-topic notes — omit this section if the agent produced none>

## API 清單
| Method | Path | 用途 | 送審分類 |
|---|---|---|---|
```

Strip the agent's `### <spec title>` heading and its `####` sub-headings down to the `#`/`##` levels shown above when writing the file (each file stands alone, it doesn't need the nesting used inside the agent's combined response).

## Index

Add/update in `docs/README.md`, right after the existing `## Backend (data flow)` bullet list — one bullet per generated file:

```markdown
## Backend API 詳細定義
- [<slug>](backend/<slug>.md) — <spec title>：欄位定義、限制條件、跨主題規則、完整 API 清單，`/doc-backend` 產出
```

## Output

After the document is written, print:

```
| File                       | URLs listed |
|-----------------------------|-------------|
| docs/backend/brand.md      | N           |
| docs/backend/currency.md   | N           |
...
```

## Rules

- Do NOT ask for confirmation at any step — execute immediately.
- No diagrams, no Mermaid — this command is prose + tables only.
- Never invent a URL, method, field, or behavior not present in the spec.
- Never modify `docs/blueprint/*` or anything outside `docs/backend/<slug>.md` (one file per spec in scope) and the `docs/README.md` index entries.
