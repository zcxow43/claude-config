You generate **real-looking screen storyboards** for the frontend and save them under `docs/frontend/` — this command owns that directory; `/doc` (backend/DB docs) never writes there.

`/doc-fronend` never draws a diagram and never uses text/shapes to represent a screen — every frame is a real, populated-looking screen mockup, screenshotted. But it is **inferred purely from the spec**, not produced by running the real app: no docker, no backend, no frontend dev server, no Playwright clicking through a live page. That keeps it independent of whether `develop/` has even been built yet — same as `/doc`, which only ever reads `specs/`.

Unrelated to the `solution-architect` agent (`.claude/agents/sa.md`) that `/doc` uses for drafting backend/DB diagrams — `/doc-fronend` does not read/write Mermaid and does not dispatch that agent.

## How it works

For each flow, you (not a live app) build one static, self-contained HTML file per meaningful screen state — realistic layout, realistic Chinese content, inferred straight from the spec's field/table/button definitions (and, if `develop/frontend/src/` already has the real page for that route, grounded further in its actual copy/labels/DOM roles — nice-to-have for accuracy, never a hard dependency). A small headless-browser script then screenshots each static HTML file and composes the sequence into one filmstrip image: real screen → arrow captioned with the real action → next real screen. No live interaction ever happens; "the click" is depicted by authoring the *next* screen's mockup to already reflect its effect (e.g. a toggle mid-flight, a form filled in, a list with the new row).

## One-time Setup

The render pipeline lives in `docs/frontend/_scripts/` (Playwright, plain Node — already scaffolded, kept separate from the Vite app's own `package.json`/`node_modules`). If `_scripts/node_modules/` doesn't exist yet, install once:

```bash
cd docs/frontend/_scripts && npm install && npx playwright install chromium
```

Skip this if `node_modules/.bin` and the Chromium cache already exist — don't reinstall every run.

## Argument Handling

Check `$ARGUMENTS`:

- **Empty or blank**: regenerate every flow group (see Grouping Rule below)
- **A group/spec slug** (e.g. `brand`, `currency-pair`, `audit`): regenerate only that flow
- Anything else: treat it as a group slug and proceed the same way; if no matching spec exists, say so and stop

## Grouping Rule

Same grouping as `/doc`'s frontend groups — one flow per group, not per spec file, so a flow that spans several screens (e.g. define → approve) stays one storyboard:

| Group slug | Frontend specs | Entry route |
|---|---|---|
| `currency-pair` | currency, currency-pair, currency-pair-definition, currency-pair-approval | `/currency-pairs` |
| `brand` | brand | `/brands` |
| `spread` | spread | `/spread-groups` |
| `audit` | audit | `/audit-requests` |

If specs have changed since this table was written (new spec files, new routes in `develop/frontend/src/App.tsx`), re-derive groups using the same judgment `/doc` uses: group by shared entity/workflow, not blind dependency-closure.

## Process

1. **Read the spec(s)** for the group in scope, fully (`specs/frontend/<slug>.md`) — page layout/table columns/fields, button labels, API integration table, error/empty/loading states, acceptance criteria. This is the primary source of truth for content.
2. **Optionally ground further in real code** — if `develop/frontend/src/pages/<Page>.tsx` (and its subcomponents/tests) already exists for that route, skim it for exact copy, real seeded values, and real interaction-state behavior (e.g. "disable the control while its request is in flight") so the mockup matches what was actually built. If the code doesn't exist yet, infer everything from the spec alone — that's fine, it's the expected case for a not-yet-built feature.
3. **Decide the states**: the flow's happy path end-to-end as a short sequence of screens — initial load, then one screen per real user action that visibly changes something (open a form, fill a field, submit, see the updated list/row), landing on the resulting state. 3–6 steps is typical; don't enumerate every edge case.
4. **Author one HTML file per state** at `docs/frontend/<group>/step-N.html` — a complete, self-contained page (inline `<style>`, inline `<script>` if it's simplest to compute rows from a small data array, no external assets). Design at 1440×900. Use real Traditional-Chinese labels/values from the spec (table columns, button text, status labels, toast copy) — never "Lorem ipsum" or "test test". Depict the *effect* of an action in the next step's screen (a toggle now shows its new/pending state, a table has the new row, a toast is visible) rather than trying to animate the click itself.
   Where a step is caused by clicking a specific real element, mark it — invisibly, no visual/behavioral change to the mockup itself:
   - Add a bare `data-trigger` attribute to the element in step N that was clicked to produce step N+1 (e.g. the `編輯` button, a `+新增群組` button, `核准`).
   - Add a bare `data-target` attribute to the element in step N+1 that resulted from that click (e.g. the modal that opened, a toast, the row that changed).
   The render step below draws that transition's arrow starting exactly at `data-trigger`'s position and ending exactly at `data-target`'s — a real diagonal line from the real button to its real effect, not a generic centered glyph. Leave both off a transition that isn't caused by one specific click (e.g. "time passes and the request resolves") — it falls back to a sensible edge-to-edge default automatically.
5. **Write** `docs/frontend/<group>/manifest.json`:
   ```json
   [
     { "file": "step-1.html", "caption": "初始畫面：..." },
     { "file": "step-2.html", "caption": "點擊「...」按鈕：..." }
   ]
   ```
   Each caption is a short, real, Traditional-Chinese description of the action that produced that state (the first step's caption just describes the initial state).
6. **Render**: from the project root,
   ```bash
   node docs/frontend/_scripts/render.mjs docs/frontend/<group>
   ```
   This screenshots each `step-N.html` to `step-N.png` at 2x device pixel ratio (sharp, not blurry, when embedded or zoomed). The board wraps into rows of 3 instead of one ever-widening strip — row 1 reads left→right, then turns down and row 2 reads right→left, zigzagging (a short drop at whichever edge the previous row ended on, never a long diagonal jump across the page). Every connector arrow is a straight line measured from the real `data-trigger`/`data-target` positions (falling back to an edge-to-edge default when a transition has neither) — so arrows are never forced to be purely horizontal; a diagonal from a button to the modal it opened, or a vertical drop across a row wrap, are both just the measured line between two real points. All of this composes into three equivalent views of the same board — `storyboard.png` (flat image, for embedding inline), `storyboard.html` (open directly in a browser for native pan/zoom), and `storyboard.pdf` (captions render vector-crisp at any zoom; good for sharing as one file).
7. **Sanity-check** the rendered `storyboard.png` (read it back) — confirm every frame renders as intended (no blank/broken layout, Chinese text isn't mojibake, captions match what the frame actually shows). Fix the HTML and re-run if not.
8. **Write** `docs/frontend/<group>.md` (File Shape below).
9. **Regenerate** `docs/frontend/README.md` (Index below) once all groups in scope are done.
10. **Print** the summary table (Output below).

`docs/frontend/brand/` is a working example already in the repo — a 3-step flow inferred from `specs/frontend/brand.md`'s own ASCII mockup and its real `BrandTable.tsx` (toggle-switch markup, "更新中" mid-flight state). `docs/frontend/currency-pair/` is a richer example showing `data-trigger`/`data-target` in active use across all 4 of its steps (each button → the exact modal/row it produces). Use either as the template for new flows.

### File Shape

`docs/frontend/<group>.md`:

```markdown
# <Title, e.g. 品牌管理>

<one-sentence framing: what real flow this storyboard walks through>

![storyboard](<group>/storyboard.png)

放大瀏覽：[瀏覽器開啟 (storyboard.html)](<group>/storyboard.html) ・ [PDF](<group>/storyboard.pdf)
```

No Mermaid, no ASCII wireframes, no prose walkthrough beyond the one framing sentence — the screenshots carry the content. The `.html`/`.pdf` links exist because the flat PNG is the only one that degrades at zoom for a wide multi-step flow — always include both links, not just for flows long enough to feel cramped.

## Index

Regenerate `docs/frontend/README.md`:

```markdown
# User Flow Storyboards

Real-looking screen storyboards inferred from `specs/frontend/` — every frame is a rendered, realistically-populated HTML mockup (not a drawn diagram), captured with `_scripts/render.mjs`. No live app required. Regenerate a flow with `/doc-fronend <group>` after its spec changes.

- [<group>](<group>.md) — <one-line: what flow it walks through>
...
```

## Output

After all groups in scope are generated, print:

```
| Group        | Storyboard                                  | Steps | Specs covered                                    |
|---------------|----------------------------------------------|-------|---------------------------------------------------|
| brand         | docs/frontend/brand.md          | 3     | brand                                              |
| currency-pair | docs/frontend/currency-pair.md  | 6     | currency, currency-pair, currency-pair-definition  |
...
```

## Parallel Rules

- Different groups are fully independent (separate spec reads, separate HTML files, separate render invocations) — dispatch them in parallel.
- Within one group, author all of that group's `step-N.html` files before rendering — the render step needs the whole manifest at once.

## Rules

- Do NOT ask for confirmation at any step — execute immediately.
- Never invent content the spec doesn't support (a field, a button, a status) — when the spec is silent on a concrete value (e.g. which brands start active), pick something plausible and consistent with any example the spec itself gives, and keep it consistent across a group's steps.
- `storyboard.html` and `storyboard.pdf` are first-class deliverables (the comfortable ways to browse a wide flow), not debug scratch — always keep them, and always link both from the group's `.md` per File Shape above.
- Write only under `docs/frontend/` — never modify `develop/frontend/src/` (the real app source), `docs/backend/`, `docs/db/`, `docs/blueprint/`, `demo/`, `specs/`, or `develop/backend/`. `/doc-fronend` reads what exists; it doesn't change it.
