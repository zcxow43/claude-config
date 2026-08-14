You generate a **high-level API overview** from `specs/backend/`, saved as a single consolidated file `docs/backend/api-overview.md`.

This is a different artifact from `/doc` (`.claude/commands/doc.md`, diagrams-only DFDs, one file per feature group) and from `/doc-blue-print` (integrated field-table/constraint spec). `/doc-backend` has no diagrams and no field/constraint detail — for each API it gives a short plain-language paragraph on what it does and the general approach (not implementation detail), followed by a table of **every URL it exposes**. It shares `docs/backend/` with `/doc`'s own per-group files but never collides with them: it always writes the single file `api-overview.md`, distinct from `/doc`'s `brand.md`/`spread.md`/`currency-pair.md`/`audit.md`.

## Argument Handling

Check `$ARGUMENTS`:

- **Empty** or **`all`**: scope = every non-`skip` spec in `specs/backend/`.
- **A spec slug** (e.g. `spread`, `currency-pair`): scope = just that spec. Regenerate only that spec's section in `docs/backend/api-overview.md`, preserving every other spec's existing section untouched (read the existing file first, if it exists).
- Anything else that doesn't match a spec file: say so and stop.

## Process

1. **Read** every non-`skip` spec in scope, fully — each spec's "API Contract" (or equivalent) section is the source of truth for its URL list.
2. **Dispatch**: one `Agent` call, `subagent_type: "solution-architect"`, covering every spec in scope in a single prompt (see "Dispatching" below — override its default 12-section template).
3. **Assemble** `docs/backend/api-overview.md` per "File Shape" below. If scope was a single slug, splice that spec's returned section into the existing file in place of its old section (or append it, if new), leaving every other section byte-for-byte as it was.
4. **Index**: add/update the entry in `docs/README.md` (see "Index" below).
5. **Print** the summary table (see "Output" below).

## Dispatching

`solution-architect`'s own system prompt defaults to a 12-section per-spec report (Feature Summary, Actors, Context Diagram, Level 0/1 DFD, ...). Override that explicitly — the prompt must say so or the agent falls back to its default template.

Agent prompt format:
```
For THIS task only, ignore your default 12-section per-spec template. For each spec below, produce exactly two things, in Traditional Chinese, current-state-only (ignore superseded/historical behavior):

1. A short paragraph (3-5 sentences) explaining what this API does overall and its general approach — e.g. "查詢一律讀取目前生效資料"、"異動不會直接套用，需送審核准後才生效"。Do NOT describe implementation detail: no class/method names, no database column names, no raw function-call syntax, no HTTP status code used as a bare label (a status code may appear in parentheses next to its plain-language meaning, e.g. "拒絕並說明原因（409）", never alone). This paragraph is for a reader who wants to understand the shape of the API, not build it.

2. A table listing EVERY url this spec's API Contract section defines — do not skip or merge any, do not invent any not in the spec. Columns: Method | Path | 用途（一句話業務語言，不是函式或欄位名稱）.

Still apply your existing rules: never invent an endpoint or behavior not in the spec, business terminology only in the paragraph and the 用途 column (Method/Path columns keep their literal HTTP verb and route, that's the engineer-facing part of the table).

Read these spec files fully:
<file paths in scope>

Return, per spec, in this exact shape and nothing else (no extra sections, no restating your default template):
### <spec title>
<paragraph>

| Method | Path | 用途 |
|---|---|---|
| ... | ... | ... |
```

## File Shape

`docs/backend/api-overview.md`:

```markdown
# Backend API 總覽

高層次總覽：每個 API 在做什麼、大方向的處理方式，以及它提供的每一個 URL。實作細節（資料表欄位、程式碼結構）不在此文件範圍內 — 見 `specs/backend/` 或 `docs/backend/<slug>.md`（DFD）取得更深入的資料流。

<one `### <spec title>` section per spec in scope, each exactly as the agent returned: paragraph + URL table>
```

Spec order: same order as `specs/backend/` directory listing (stable, not dependency-derived — this document doesn't need graph reasoning).

## Index

Add/update in `docs/README.md`, right after the existing `## Backend (data flow)` bullet list:

```markdown
## Backend API 總覽
- [api-overview](backend/api-overview.md) — 每個後端 API 的大方向說明 + 完整 URL 清單，無圖表，`/doc-backend` 產出
```

## Output

After the document is written, print:

```
| Spec                          | URLs listed |
|--------------------------------|-------------|
| specs/backend/brand.md         | N           |
| specs/backend/currency.md      | N           |
...
```

## Rules

- Do NOT ask for confirmation at any step — execute immediately.
- No diagrams, no Mermaid — this document is prose + tables only.
- Never invent a URL, method, or behavior not present in the spec.
- Never modify `docs/backend/<slug>.md` (the `/doc`-owned DFD files) or anything outside `docs/backend/api-overview.md` and the `docs/README.md` index entry.
