# BackToTheDrawingBoard — Roadmap and Session Handoff

Written so a cold session can pick up mid-build without re-deriving anything.
Read this, then `README.md`, then the file.

**Owner:** Jake.
**Build this document describes:** 11,539 lines / 568 KB.
**State:** 27 self-tests passing, 142 tooltip hints, 0 audit problems.
**Phase 1 progress:** 1.1, 1.2, 1.3 landed. 1.4 – 1.7 outstanding.

Line numbers drift; every anchor below is a function or constant name.

---

## 1. Read this first if you are a fresh session

Five things that are not obvious from the code and will cost you a rewrite if
you miss them.

| Rule | Why |
|---|---|
| The **Experimental Features tab is permanent.** | Jake has said so three times. Do not suggest trimming it, do not cite its line count as a problem, do not move tools out of it. Shared *maths* may be lifted into neutral blocks (done once, see §2) as long as the tab's tool list is untouched. Verify with the audit in §7. |
| **Every new entity type needs five functions plus `sanEnt`.** | Miss `sanEnt` and the type renders all session then vanishes on reload with no error. `ENTITY_TYPES` + `auditEntityTypes()` catch it at boot. Register first, implement second. |
| **`snap()` before mutating, never after.** | And `snapEnts()` / `snapEntList()` only for in-place edits that cannot add, remove or reorder entities. Anything structural takes a full `snap()`. |
| **Entities leave the array only via `entKeep` / `entRemove`.** | These remap `S.sel`, `S.hover`, a live `gripDrag` and `S.op`. A raw `splice` or `filter` silently corrupts all four. |
| **Zero network, no `unsafe-eval`, no third-party code.** | Rules out OCR libraries, AI calls, and any `eval`-based scripting. Design around it, do not ask to relax it. |

Two smaller ones that have already bitten:

- **`entPoints("mshape")` returns the two drag corners, not the outline.** That
  is correct for grips and wrong for area. Use `mkOutline()` for measurement.
- **There are two `num()` helpers.** The options bar one is
  `num(label,val,step,set,disabled,hint)`; the properties panel has its own
  `num(label,val,set)`. Do not cross them.

House style for anything user-facing: say what happens, then the thing that is
not obvious from the name. No marketing voice.

---

## 2. What changed since the README was written

`README.md` documents the 10,490-line build and is **stale** in the areas below.
Updating it is an outstanding task.

### Pass A — Measure Area + region engine

- **New shared block `REGION ENGINE`**, after intersections, before
  `ENTITY TABLE`. Owns `RGN`, `rgnEnts`, `rgnSegs`, `rgnBox`, `rgnRaster`,
  `rgnFlood` (pure), `rgnCellOf`, `rgnCellCentre`, `rgnStats`, `RGN_UNITS`,
  `rgnAreaText`, `rgnRegionAt`.
- Experimental `xRaster`, `xFlood`, `xCellOf`, `xCellCentre`, `xRegionAt`,
  `xAreaText` are now **one-line delegations** to those. Behaviour identical,
  ribbon untouched.
- `rgnRegionAt` seeds its raster on the **viewport**, not the sheet, growing x3
  up to `RGN.GROW` (4) times only when the flood escapes. Raster cap 640 → 1024.
  A 3 m room on a 40 m plan now measures at ~3 mm/cell instead of ~62 mm.
  `F.escaped` is only meaningful when the returned `full` flag is true.
- **Tool `measurearea`** (`MR`) in Draw › Measure. Temporary graphics via
  `S.areaResult` / `paintAreaMeasure()`, cleared by `clearMeasurement()`.
  Never an entity, never exported.
- **State added:** `S.areaResult`, `S.areaUnit`.

### Pass B — bug fix and test harness

- `entKeep` remaps `S.op` entity indices via
  `OP_ENT_KEYS = ["first","boundary","ent"]`. A stale index is **dropped and the
  tool re-prompts**, not repointed. Add any new multi-step tool's index field.
- **`test` at the command line** runs `SELFTEST`. Pure functions only; any `S`
  mutation must be restored in a `finally`.

### Pass C — tooltips

- **`HINTS`**, one entry per ribbon item, central because items register across
  six tables (`TOOLS`, `ACTIONS`, `MK_TOOLS`, `FLOW_TOOLS`, `XTOOLDEF`,
  `XACTIONS`).
- `itemTitle(id,T)` renders label / hint / keys, deduping alias and hotkey when
  they are the same letter.
- **`auditRibbonHints()`** at boot and in `audit`. **Every new ribbon item needs
  a `HINTS` entry or the audit fires.**
- `TOGGLE_DEFS` is the single source for the six drafting aids, used by both
  `toggleWidget()` and `applyToggleTitles()`.

### Pass D — Phase 1.1 – 1.3

- **Markup status.** `MK_STATUS` closed vocabulary of six, stored as `e.st`,
  absent when unset. `sanStatus()` in the `sanEnt` chain drops unknown values
  rather than clamping them. `mkPaintStatus()` draws a fixed-screen-size dot,
  gated on `S.exporting` and `S.mkStatusShow`. Actions `mkstatus` (`MST`) and
  `mkstatusshow` (`MSTD`) in a new Markup › Review group.
- **Markup summary CSV.** `exportmarkupcsv` (`MCSV`) in Insert › Export.
  `csvCell()` neutralises leading `=`, `+`, `-`, `@` so a redline comment cannot
  execute as a spreadsheet formula. `mkOutline()` / `polyArea()` /
  `polyLength()` / `mkMeasureOf()` do the geometry.
- **Markups List.** Dock section between Line Styles and Properties.
  `mkListMatches()` filters by type, status and free text; `paintMarkupList()`
  builds the filter bar once and `paintMarkupRows()` rebuilds only the rows, so
  the search field keeps focus mid-word. Hooked off `updateProps()` and the
  history apply path. `zoomToEntities()` / `mkZoomFiltered()` for the ⌖ button.

---

## 3. Decisions

### Locked

- Name is BackToTheDrawingBoard; direction is a tool that communicates ideas
  visually.
- Experimental Features tab and Flow tab both permanent.
- Trim to Corner keeps Revit semantics (click what you keep).
- Prior build order: references plural (**done**) → revision overlay → markup
  summary export (**done**, arrived early as Phase 1.1).
- **Not being built**, agreed: Studio and real-time collaboration, SSO, DMS
  integrations, cloud storage, AI/MCP features, OCR, 3D PDF / IFC / U3D, full
  PDF editing (form fields, redaction, page manipulation, digital signatures),
  and batch multi-document operations.

### Open — blocks Phase 2

1. **Binding.** Does a filled region bind to its boundary geometry (move a wall,
   the area updates) or freeze a snapshot with a staleness flag like
   `refDrift`?
   *Recommendation: bound.* Bluebeam calls the equivalent object a **Space**,
   not a measurement, and ships "Convert Revit Rooms to Revu Spaces" — a
   first-class object with identity. A region that goes stale when a wall moves
   is a worse object.
2. **Boundary engine.** Analytic trace with raster fallback, or raster only?
   *Recommendation: analytic with fallback.* Raster alone gives a number whose
   error is a function of window size rather than geometry.
3. **DXF fidelity.** R12 has no real HATCH, so a region exports as a boundary
   polyline plus a text label, losing the fill. Acceptable?

---

## 4. Phase 1 — trivial wins

### Done

- **1.1 Markup summary export (CSV)** — see Pass D.
- **1.2 Markup status field** — see Pass D.
- **1.3 Markups List panel** — see Pass D.

### 1.4 Magic Markups equivalents — NEXT
Bluebeam's $590 tier. Structurally cheap here because every entity is a point
list with a type tag.
- **Convert to:** type swap plus `mapEnt`. e.g. pline → mshape rect, ink →
  pline, mshape → ink.
- **Duplicate as:** clone then convert, original untouched.
- **Offset:** already exists, expose it for markup.
*Touches:* new actions, `HINTS`, a conversion table keyed by source type.
*Gotcha:* converting across families (CAD ↔ markup) changes which paint router
claims the entity. `mkOutline()` already gives you the point list to convert
from. Run `auditEntityTypes()` after.

### 1.5 Count measurement
A stamp-style entity with a running tally. Decide temporary (like the rest of
Measure) or persistent (for takeoff counts) when you start. Persistent means a
new entity type and the full five-function checklist.

### 1.6 Command macros
Bluebeam's "scripting commands" is $440 tier. You already have a command line,
an `ALIAS` table and `handleCommand`. A macro is a recorded list of command
strings replayed through `handleCommand`. **No `eval`, so no CSP conflict.**
*Touches:* record/stop/play actions, storage in `.b2db` and localStorage,
`HINTS`. Guard against a macro that calls itself.

### 1.7 Tool Chest / preset library
Named presets over `S.mk` (and later `S.flow`). Save, apply, delete. Serialise
into `.b2db`.

---

## 5. Phase 2 — the region entity

Gated on §3. This is the big one and it unlocks three other features.

- **New entity type,** registered in `ENTITY_TYPES` first. Working name
  `space`. Needs `sanEnt`, `entPoints`, `entBox`, `entHit`, paint routing,
  `gripSet`, DXF and SVG emitters, properties rows.
- **Boundary trace:** ray-cast from the pick point to the nearest edge, then
  walk the segment graph always taking the tightest turn by angular sort until
  you return to the start. Shoelace for area — `polyArea()` already exists.
  Build on `curveInt`, the intersections block, `flatten`, `traceChainPoints`.
- **Gap tolerance and defined failure.** Real rooms are not closed. If the walk
  fails to close, fall back to `rgnRegionAt`, label the result approximate, and
  report which engine produced the number.
- **Binding**, if confirmed: the region holds the eids of its boundary
  entities and re-derives when they move. Model on how flow edges bind to nodes
  by id.
- Then **Measure Area becomes the throwaway** and `space` is the thing you keep.

Unlocks, cheapest first once `space` exists:
- **Custom hatch patterns** (Core tier) — a fill style on the region.
- **Region-scoped markup reporting** (Complete tier) — point-in-polygon over
  `S.ents` inside a named region, feeding the Phase 1.1 CSV.
- **Named spaces** with schedules, if you want takeoff.

---

## 6. Phase 3+ — larger work, ordered

| Phase | Item | Notes |
|---|---|---|
| 3 | **Revision overlay** | `R.cal` was built for this and nothing consumes it yet. Two registered points per reference is a similarity transform. Manual registration first; auto-align is a separate, harder job. |
| 4 | **PDF export** | The biggest real gap, and absent from Bluebeam's feature list only because PDF is their substrate. A minimal vector + text PDF writer is a few hundred lines, no compression needed if larger files are acceptable. PDF is what people in this line of work actually send. |
| 5 | **VisualSearch** | Template-match a rasterised region against the drawing. `rgnRaster` and the Sobel edge pass already exist. Bounded correlation search, moderately fiddly. |
| 6 | **`S.sel` → eids** | Deliberately deferred. Touches picking, grips, properties and every modify op. Must be its own pass with nothing else in the diff, or a break in one is indistinguishable from a break in the other. `entKeep` and the `S.op` fix hold the line meanwhile. |
| 7 | **Multi-document** | Only if batch operations ever become worth it. `S` becomes one of many. Large architectural change; currently no reason to. |
| — | **`README.md` refresh** | Stale since Pass A. Not urgent, but it is the first document a fresh session reads. |

---

## 7. Verification

Run after every change. Do not skip the syntax check; it catches most damage
from scripted edits.

```bash
python3 -c "
import re
s=open('BackToTheDrawingBoard.html').read()
open('/tmp/app.js','w').write(re.findall(r'<script[^>]*>(.*?)</script>', s, re.S)[-1])
" && node --check /tmp/app.js && echo SYNTAX_OK
```

Headless suite: build `/tmp/run.js` as browser stubs (`document`, `window`,
`getComputedStyle`, `navigator`, `localStorage`, `ResizeObserver`, and a canvas
2D context stub returning `createImageData` / `getImageData` / `measureText`),
then the script, then run `SELFTEST`, `auditRibbonHints()` and
`auditEntityTypes()`. Expect **27 passed, 0 missing hints, 0 entity problems**.

Experimental tab must stay at **7 groups, 30 tool ids**:

```bash
python3 -c "
import re
s=open('BackToTheDrawingBoard.html').read()
b=s[s.index('\"Experimental Features\":['):]; b=b[:b.index(']\n};')]
print('groups',b.count('{name:'),'ids',len(re.findall(r'\"(x[a-z]+)\"',b)))
"
```

In-browser: `test`, `audit`, `hist`, `refs`. Console clean at boot —
`auditRibbonButtons`, `auditRibbonHints`, `auditEntityTypes` and
`auditReferences` all run there.

---

## 8. Working agreement

- Step by step on technical work; stop at first failure.
- One coherent chunk per pass, verified, then the next. Never bundle a refactor
  with a feature.
- `HINTS` entry for every new ribbon item.
- `SELFTEST` case for every new pure function.
- Flag stale `README.md` sections rather than letting them rot silently.
