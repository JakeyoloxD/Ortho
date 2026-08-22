# BackToTheDrawingBoard — Roadmap and Session Handoff

Written so a cold session can pick up mid-build without re-deriving anything.
Read this, then `README.md`, then the file.

**Build this document describes:** 11,539 lines / 568 KB.
**Phase 1 progress:** 1.1, 1.2, 1.3 landed. 1.4–1.7 outstanding.
**Suite:** 27 tests, 142 hints, 0 audit problems.
Line numbers drift; every anchor below is a function or constant name.

---

## 1. Read this first if you are a fresh session

Five things that are not obvious from the code and will cost you a rewrite if
you miss them.

| Rule | Why |
|---|---|
| The **Experimental Features tab is permanent.** | Sire has said so three times. Do not suggest trimming it, do not cite its line count as a problem, do not move tools out of it. Shared *maths* may be lifted into neutral blocks (this has been done once, see §2) as long as the tab's tool list is untouched. Verify with the audit in §7. |
| **Every new entity type needs five functions plus `sanEnt`.** | Miss `sanEnt` and the type renders all session then vanishes on reload with no error. `ENTITY_TYPES` + `auditEntityTypes()` catch it at boot. Register first, implement second. |
| **`snap()` before mutating, never after.** | And `snapEnts()` only for in-place edits that cannot add, remove or reorder entities. Anything structural takes a full `snap()`. |
| **Entities leave the array only via `entKeep` / `entRemove`.** | These remap `S.sel`, `S.hover`, a live `gripDrag` and (since this build) `S.op`. A raw `splice` or `filter` silently corrupts all four. |
| **Zero network, no `unsafe-eval`, no third-party code.** | Rules out OCR libraries, AI calls, and any `eval`-based scripting. Design around it, do not ask to relax it. |

House style for anything user-facing: say what happens, then the thing that is
not obvious from the name. No marketing voice.

---

## 2. What changed since the README was written

The uploaded `README.md` documents the 10,490-line build. Three passes have
landed since. `README.md` has **not** been updated and is now stale in these
areas.

### Pass A — Measure Area + region engine

- **New shared block: `REGION ENGINE`**, sits after the intersections block,
  before `ENTITY TABLE`. Owns `RGN` caps, `rgnEnts`, `rgnSegs`, `rgnBox`,
  `rgnRaster`, `rgnFlood` (pure), `rgnCellOf`, `rgnCellCentre`, `rgnStats`,
  `RGN_UNITS`, `rgnAreaText`, `rgnRegionAt`.
- The Experimental functions `xRaster`, `xFlood`, `xCellOf`, `xCellCentre`,
  `xRegionAt`, `xAreaText` are now **one-line delegations** to those. Every
  joke tool calls the same names and behaves the same. The tab's ribbon is
  byte-identical: 7 groups, 30 tool ids, Flood Fill still `big` in Back Room.
- `rgnRegionAt` seeds its raster window on the **viewport**, not the sheet, and
  grows x3 up to `RGN.GROW` (4) times only when the flood escapes. Raster cap
  raised 640 → 1024. Net effect: a 3 m room on a 40 m plan measures at ~3 mm
  per cell instead of ~62 mm. `F.escaped` is only meaningful when the returned
  `full` flag is true.
- **New tool `measurearea`**, alias `MR`, in Draw › Measure beside MB and MA.
  Temporary graphics: `S.areaResult`, painted by `paintAreaMeasure()` inside
  `paintMeasurement()`, cleared by `clearMeasurement()`, two-stage Esc like the
  rest of the family. Never an entity, never in an export.
- **New state:** `S.areaResult`, `S.areaUnit` (`auto` | `mm2` | `cm2` | `m2`).
  Unit selector is in the options bar.

### Pass B — bug fix and test harness

- `entKeep` now remaps `S.op` entity indices via `OP_ENT_KEYS =
  ["first","boundary","ent"]`. A stale index is **dropped and the tool
  re-prompts** rather than repointed at whatever slid into the slot. Add any
  new multi-step tool's index field to that list.
- **`test` at the command line** runs `SELFTEST`, currently 16 assertions, all
  passing. Pure functions only, no `S` mutation that isn't restored in a
  `finally`. It caught a real bug on its first run (`rgnSegs` was not filtering
  degenerate segments on the `curvesOf` path).

### Pass C — tooltips

- **`HINTS`**, a central table of 139 entries, one per ribbon item, keyed by id.
  Central rather than in the tool definitions because items are registered
  across six tables (`TOOLS`, `ACTIONS`, `MK_TOOLS`, `FLOW_TOOLS`, `XTOOLDEF`,
  `XACTIONS`).
- `itemTitle(id,T)` renders label / hint / keys as three blocks. Alias and
  hotkey are deduped when they are the same letter.
- **`auditRibbonHints()`** at boot and in the `audit` command. Currently zero
  missing. **Any new ribbon item must add a `HINTS` entry or the audit fires.**
- `TOGGLE_DEFS` is now the single source for the six drafting aids, consumed by
  both `toggleWidget()` (View tab) and `applyToggleTitles()` (status bar).
- Options bar factories `num`, `chk`, `radio`, `readField` all take an optional
  trailing `hint`. **Note:** the properties panel has its own separate
  three-argument `num(label,val,set)`. Do not confuse them.

**Action item:** `README.md` needs a pass to cover all of the above. Not urgent,
but it is the document a future session reads first.

---

## 3. Decisions

### Locked

- Rename to BackToTheDrawingBoard; direction is a tool that communicates ideas
  visually.
- Experimental Features tab and Flow tab both permanent.
- Trim to Corner keeps Revit semantics (click what you keep).
- Prior build order: references plural (**done**) → revision overlay → markup
  summary export.
- Architecturally blocked and **not being built** (agreed): Studio and any
  real-time collaboration, SSO, DMS integrations, cloud storage, AI/MCP
  features, OCR, 3D PDF / IFC / U3D, full PDF editing (form fields, redaction,
  page manipulation, digital signatures), and batch multi-document operations.

### Open — blocks Phase 2, confirm before writing code

1. **Binding.** Does a filled region bind to its boundary geometry (move a wall,
   the area updates) or freeze a snapshot with a staleness flag like
   `refDrift`?
   *Recommendation: bound.* Bluebeam calls the equivalent object a **Space**,
   not a measurement, and has "Convert Revit Rooms to Revu Spaces" — clearly a
   first-class object with identity. A region that goes stale when a wall moves
   is a worse object.
2. **Boundary engine.** Analytic trace with raster fallback, or raster only?
   *Recommendation: analytic with fallback.* Raster alone gives a number whose
   error is a function of window size rather than geometry. Analytic gives an
   exact polygon you can store as the entity's point list.
3. **DXF fidelity for regions.** R12 has no real HATCH. A region exports as a
   boundary polyline plus a text label, losing the fill. Acceptable?

---

## 4. Phase 1 — trivial wins, no new entity type

Greenlit. Each item is independently shippable; do them in any order, verify
after each. None of these needs the open decisions resolved.

### 1.1 Markup summary export (CSV) — DONE
Honours the previously chosen build order. `S.ents` → rows: type, layer, colour,
text, position, area/length where applicable, status. Reuse the existing
download helper. Emit CSV only for now; XML if asked.
*Touches:* new action in `ACTIONS`, `HINTS` entry, `RIBBON` Insert › Export.

### 1.2 Markup status field — DONE
A `status` string on markup entities plus a colour map. Bluebeam ships custom
statuses at Core tier.
*Touches:* `sanEnt` (accept and clamp the string), `mkPropRows`, paint routine
for the status tint, `.b2db` round-trip is automatic once `sanEnt` allows it.

### 1.3 Markups List panel — DONE
Filterable table over `S.ents`. Cheap here and expensive for Bluebeam because
your annotations are already a typed array. Filter by type, layer, colour,
status; click a row to select and zoom.
*Touches:* new right-dock section beside Layers / Line Styles / Properties, or
a new panel; reuse `selectByIds` for selection so it survives reorder.

### 1.4 Magic Markups equivalents
Bluebeam's `$590` tier. Structurally cheap here because every entity is a point
list with a type tag.
- **Convert to:** type swap plus `mapEnt`. e.g. pline → mshape rect, ink →
  pline.
- **Duplicate as:** clone then convert.
- **Offset:** already exists, just expose it for markup.
*Touches:* new actions, `HINTS`, a conversion table keyed by source type.
*Gotcha:* converting across families (CAD ↔ markup) changes which paint router
claims the entity. Test with `auditEntityTypes()`.

### 1.5 Count measurement
A stamp-style entity with a running tally, temporary like the rest of Measure or
persistent if you want takeoff counts. Decide which when you start.
*Touches:* Draw › Measure group, `HINTS`.

### 1.6 Command macros
Bluebeam's "scripting commands" is `$440` tier. You already have a command line,
an `ALIAS` table and `handleCommand`. A macro is a recorded list of command
strings replayed through `handleCommand`. **No `eval`, so no CSP conflict.**
*Touches:* record/stop/play actions, storage in `.b2db` and localStorage,
`HINTS`. Guard against a macro that calls itself.

### 1.7 Tool Chest / preset library
Named presets over `S.mk` (and later `S.flow`). Save, apply, delete. Serialise
into `.b2db`.

**Verification after each item:** §7.

---

## 5. Phase 2 — the region entity

Gated on the open decisions in §3. This is the big one and it unlocks three
other features.

- **New entity type,** registered in `ENTITY_TYPES` first. Working name
  `space`. Needs: `sanEnt`, `entPoints`, `entBox`, `entHit`, `paintEnt` or a
  family router, `gripSet`, DXF and SVG emitters, properties rows.
- **Boundary trace:** from the pick point, ray-cast to the nearest edge, then
  walk the segment graph always taking the tightest turn by angular sort until
  you return to the start. Shoelace for area. Build on `curveInt`, the
  intersections block, `flatten` and `traceChainPoints`.
- **Gap tolerance and defined failure.** Real rooms are not closed. If the walk
  fails to close, fall back to `rgnRegionAt`, label the result approximate, and
  report which engine produced the number.
- **Binding**, if confirmed: the region holds the eids of its boundary
  entities and re-derives when they move. Model it on how flow edges bind to
  nodes by id.
- Then: **Measure Area becomes the throwaway** and the region entity is the
  thing you keep.

Unlocks, in order of cheapness once `space` exists:
- **Custom hatch patterns** (Core tier) — a fill style on the region.
- **Region-scoped markup reporting** (Complete tier) — point-in-polygon query
  over `S.ents` inside a named region, feeding the Phase 1.1 CSV.
- **Named spaces** with schedules, if you want takeoff.

---

## 6. Phase 3+ — larger work, ordered

| Phase | Item | Notes |
|---|---|---|
| 3 | **Revision overlay** | Previously chosen next direction. `R.cal` was built for exactly this and nothing consumes it yet. Two registered points per reference is a similarity transform. Manual registration first; auto-align is a separate, harder job. |
| 4 | **PDF export** | Your biggest real gap and not on Bluebeam's feature list because it is their substrate. A minimal vector + text PDF writer is a few hundred lines, no compression needed if you accept larger files. PDF is what people in Sire's world actually send. |
| 5 | **VisualSearch** | Template-match a rasterised region against the drawing. `rgnRaster` and the Sobel edge pass already exist. Bounded correlation search, moderately fiddly. |
| 6 | **`S.sel` → eids** | Deliberately deferred. Touches picking, grips, properties and every modify op. Must be its own pass with nothing else in the diff, or a break in one is indistinguishable from a break in the other. `entKeep` and the `S.op` fix keep it honest in the meantime. |
| 7 | **Multi-document** | Only if batch operations ever become worth it. `S` becomes one of many. Large architectural change; currently no reason to. |

---

## 7. Verification

Run after every change. Do not skip the syntax check; it catches most damage
from scripted edits.

```bash
# syntax
python3 -c "
import re
s=open('BackToTheDrawingBoard.html').read()
open('/tmp/app.js','w').write(re.findall(r'<script[^>]*>(.*?)</script>', s, re.S)[-1])
" && node --check /tmp/app.js && echo SYNTAX_OK
```

Headless test + hint audit: rebuild `/tmp/run.js` with browser stubs
(`document`, `window`, `getComputedStyle`, `navigator`, `localStorage`,
`ResizeObserver`, a canvas 2D context stub returning `createImageData` /
`getImageData` / `measureText`), append the script, then run `SELFTEST` and
`auditRibbonHints()`. Expect **16 passed, 0 missing hints**.

Experimental tab must stay at **7 groups, 30 tool ids**:

```bash
python3 -c "
import re
s=open('BackToTheDrawingBoard.html').read()
b=s[s.index('\"Experimental Features\":['):]; b=b[:b.index(']\n};')]
print('groups',b.count('{name:'),'ids',len(re.findall(r'\"(x[a-z]+)\"',b)))
"
```

In-browser: `test`, `audit`, `hist`, `refs` at the command line. Console should
be clean at boot — `auditRibbonButtons`, `auditRibbonHints`, `auditEntityTypes`,
`auditReferences` all run there.

---

## 8. Working agreement

- Step by step on technical work, stop at first failure.
- One coherent chunk per pass, verified, then the next. Do not bundle a
  refactor with a feature.
- Add a `HINTS` entry for every new ribbon item.
- Add a `SELFTEST` case for every new pure function.
- Flag stale `README.md` sections rather than silently letting them rot.
