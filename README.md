# BackToTheDrawingBoard

Single-file, offline 2D drafting, markup and flowchart tool. One `.html` file, no build step, no dependencies, no network calls. Open it in a browser and it works.

**Build documented:** 10,490 lines / 507 KB. Line numbers below are for this build and will drift as you edit.

---

## 1. Design constraints

These are the rules the whole file is built around. Breaking one of them breaks the point of the tool.

| Constraint | Consequence |
|---|---|
| Single file | No modules, no bundler. Everything lives in one `<script>`, ordered by dependency. |
| Zero network | Strict CSP in `<head>`: `default-src 'none'`, `connect-src 'none'`, `img-src data: blob:`. `script-src` omits `unsafe-eval`, so no runtime code compilation. |
| No third-party code | Inflate uses the browser's own `DecompressionStream`. PDF parsing is hand-rolled. No PDF.js, no canvas libs. |
| Everything is an entity | Markup, flowchart nodes and CAD geometry all live in `S.ents`, so undo, selection, layers, move/copy and export work on all of them with one implementation. |
| Files are hostile | Nothing loaded from disk, clipboard or localStorage reaches state without passing through the SECURITY block. |
| Units are mm | World coordinates are millimetres. Screen-entered sizes (pen width, text height) are converted to world units at draw time so they scale with the plan. |
| One way out of the array | Entities only ever leave `S.ents` through `entKeep`/`entRemove`, which remap the selection, hover and any live grip drag. No ad-hoc `splice` or `filter`. |
| One world, many underlays | World coordinates are one shared millimetre space. Each reference image carries its own pixel→world transform; per-reference calibration never means two references disagree about what a millimetre is. |
| Types are declared, not implied | Every entity type is registered in `ENTITY_TYPES` and checked against the dispatch switches at boot. A type that renders but would not survive a reload is a console error, not a surprise. |

---

## 2. Running it

Open `BackToTheDrawingBoard.html` in Chrome, Edge or Firefox. Nothing to install.

Works from `file://`. Autosave uses IndexedDB with a localStorage fallback, both of which are origin-scoped — if you move the file, the autosave does not follow it.

---

## 3. Core concepts

### Workspaces

`S.workspaceMode` is either `whiteboard` (clean canvas, no grid, no grid snap) or `drafting` (grid and grid snap on). F7 or **View › Workspace** switches. It is a presentation switch, not a mode switch — every tool works in both.

### The state object

`S` (line 515) is the single global. There is no store, no reactivity, no framework. Mutate `S`, call `draw()`. Key fields:

```
S.ents[]        every drawable thing, CAD + markup + flow
S.layers[]      {name, aci, on, lock}
S.styles[]      {name, w, aci, pat}   line weight/colour/dash
S.sel           Set of indices into S.ents
S.view          {cx, cy, s}  world centre + scale
S.tool          active tool id
S.pending[]     points picked so far for the active tool
S.op{}          scratch space for multi-step modify tools
S.undo[]/redo[] history records, byte-budgeted (see History below)
S.refs[]        tracing underlays, back to front
S.refActive     rid of the reference the image tools point at
S.refImage      accessor (read-only) for the active reference
S.mk{}          current markup pen settings
S.flow{}        current flow node/edge styling
S.x{}           experimental feature settings
```

### Entity types

Every entity has `t` (type), `li` (layer index), `si` (style index) and `eid`.

`eid` is a session-stable identity. Nothing renders from it; it exists so
anything that has to survive a reorder — selection across an undo, a history
patch record — can hold an identity instead of an array slot. It is stamped
lazily (on the next snapshot or the next time something asks) and re-issued
from scratch on load, so it is stable within a session, not across files.

**CAD**
```
{t:"line",    a, b}
{t:"pline",   pts[], closed}
{t:"spline",  pts[]}
{t:"circle",  c, r}
{t:"arc",     c, r, a0, a1}
{t:"ellipse", c, rx, ry, rot, a0, a1}
{t:"text",    p, s, h}
{t:"dim",     a, b, p}
```

**Markup** — carries its own colour/width/opacity in `e.mk`, ignores osnap, grid snap and CAD line styles.
```
{t:"ink",    pts[], closed, cloud, bump, mk}
{t:"mshape", k:line|arrow|darrow|rect|ellipse|cloud, a, b, hl, bump, mk}
{t:"mtext",  p, s, h, lead, box, hl, mk}
mk = {col, w, alpha, blend, fill, fillAlpha}
```

**Flow** — connectors bind to nodes by `id`, not coordinate. Move a node and every edge re-routes. That binding is the entire reason the Flow tab exists.
```
{t:"node", k:proc|dec|term|data|note, id, p, w, h, s, fs, mk}
{t:"edge", from:{id}|{pt}, to:{id}|{pt}, route:"ortho"|"straight", wp[], a0, a1, s, fs, hl, mk}
```

### The three tool families

`TOOLS` (line 994) and `ACTIONS` (line 1076) are the registries. A tool waits for input; an action fires once.

Tools are flagged `markup:true`, `flow:true` or neither (CAD). Pointer events fan out on those flags:

```
pointerdown → touchDown()?          mobile gesture layer intercepts
            → S.punch?              punchAt()
            → mkIsTool(S.tool)?     mkPointerDown()
            → flowIsTool(S.tool)?   flowPointerDown()
            → gripBegin()?          grip drag
            → else                  CAD pick / band select
```

`MK_TOOLS` and `FLOW_TOOLS` register themselves into `TOOLS` at load, so markup and flow tools get command-line aliases, hotkeys and ribbon wiring for free.

### References

A drawing carries a list of underlays, not one. `S.refs[]` holds them back to
front; `S.refActive` is the `rid` of the one the image tools point at. Most of
the file wants the active reference, so `S.refImage` survives as a read-only
accessor that returns it — every existing read keeps working. Writes go through
`refAdd` / `refRemove` / `refSetActive`, and the setter logs an error rather
than failing silently.

Each reference owns its own image→world transform (`x`, `y`, `w`, `h`, `rot`,
`crop`) and its own layer (`li`), which means overlay visibility is the layer
panel you already have rather than a second show/hide mechanism. World units
stay millimetres throughout: per-reference calibration means each underlay maps
its own pixels into the one shared world, never that two of them disagree about
scale. Measuring across references has to keep working.

**Calibration is stored, in image pixel space.**

```
R.cal = { a:{x,y}, b:{x,y}, mm, at }      a and b in image pixels
```

Calibrating used to compute a scale factor, bake it into `w`/`h` and discard
the factor, so a calibrated drawing carried no record of how it got that way.
Pixel coordinates are intrinsic to the image and survive every move, rotate,
scale and crop that follows. `refCalWorld(R)` maps them back to wherever the
reference is now; `refDrift(R)` reports the ratio between the current placement
scale and the calibrated one, and the reference audit flags anything past 1%.

Two registered points per reference is exactly what a similarity transform
between two references needs, which is what revision overlay will be built on.
Nothing consumes it yet — it is stored now so drawings calibrated from here on
never need re-registering.

Files saved before this change carry a singular `ref`; `sanRefs` reads both
shapes, so every existing `.b2db` still opens.

### Removing entities

`S.sel`, `S.hover` and a live grip drag all hold indices into `S.ents`, so any
splice or filter silently repoints them at whatever slid into the gap. There is
exactly one way out of the array:

```
entKeep(pred)        keep the entities pred likes, remap sel/hover/gripDrag
entRemove(indices)   the same thing by index
entAdd(list)         append and stamp eids
```

`entKeep` also invalidates the flow node index, so connectors resolve against
the new array. Selection can be captured and restored across a reorder with
`selIds()` / `selectByIds()`, or by object with `selectEntities()`.

### One dispatcher

`activateRibbonItem(id)` (line 8716) is the **only** place a ribbon button activates anything. That is deliberate — it means active state cannot drift between tabs, one-shot commands always clear the previously armed tool, and Experimental gating happens in one place.

---

## 4. Module map

### Document shell

| Lines | Block | What it does |
|---|---|---|
| 1–12 | `<head>` | CSP, viewport, no-referrer. The policy is strict because the app makes zero network calls. |
| 13–435 | `<style>` | 2026 dark CAD shell. All colour goes through CSS custom properties on `:root` — the canvas reads them back via `getCSS()`, so the theme is defined once. Subsections: app/document bar, ribbon, contextual tool strip, canvas, viewport splitters, status strip, right palette, command line, inline text editor, touch/mobile, markup palette, experimental feedback, punch mode. |
| 437–496 | `<body>` | CSS-grid shell: `top / ribbon / optbar / canvas / status / cmd`. The ribbon and right palette are empty containers — everything in them is generated. Four hidden `<input type=file>` elements handle DXF, .b2db, PDF and image import. |

### Engine

| Lines | Block | Notes |
|---|---|---|
| 499–513 | **constants** | AutoCAD Color Index table. `aciColor(i)` maps index → hex. |
| 514–575 | **state** | The `S` object. Everything else reads and writes this. |
| 576–749 | **geometry** | Vector primitives (`V/add/sub/mul/norm/perp/dist/ang`), then the polymorphic entity helpers that the rest of the file leans on: `entPoints`, `entMids`, `entCenters`, `entQuads`, `entHit`, `entBox`, `moveEnt`. Curve flatteners `arc3`, `ellPts`, `splPts`, `flatten` turn anything into a point list. |
| 750–832 | **view** | World↔screen transform, canvas resize, and the on-canvas inline text editor (a positioned `<textarea>` overlaid on the canvas rather than a modal). |
| 834–1017 | **history** | `stateBlob()` JSON-snapshots entities, layers, styles, current indices, reference-image placement and the flow/group/entity id sequences. The `<img>` element itself stays out of JSON and is held in `refImgEl`, so removing an image and undoing brings the pixels back. Two record kinds, a byte budget, coalescing and selection restore — see [History](#7-history) below. |
| 1018–1123 | **snapping** | Grid pitch that adapts to zoom, ortho constraint, the main `resolveSnap()` entry point, and reference-image edge snapping (`prepareReferenceEdges` runs a Sobel-style pass so you can snap to strong edges in a scanned plan). |
| 1124–1239 | **REFERENCES** | `S.refs[]` and the active-reference accessor, per-reference calibration helpers (`refScale`, `refCalScale`, `refCalWorld`, `refDrift`), layer-aware visibility (`refShown`), and add/remove/cycle. |
| 1240–1744 | **tools** | Tool and action registries, the Revit-style Measure implementation (MB between references, MA along an element with Tab chain preview), the command/prompt plumbing (`say`, `askArray`, `requestString`, `setPrompt`, `echo`), and the point-feeding state machine `feed()` → `commit()`. Chain line drawing commits each segment as its own entity rather than one polyline, so weld and per-segment fillet work. |
| 1745–1833 | **intersections** | Analytic line/line, line/circle, circle/circle, plus generic `curveInt` for anything flattened. Feeds trim, extend, fillet, chamfer and the Intersection snap. |
| 1834–1907 | **ENTITY TABLE** | Stable `eid` stamping, and the single removal path (`entKeep`/`entRemove`/`entAdd`) that keeps `S.sel`, `S.hover` and a live grip drag pointing at what they were pointing at. Also `selIds`/`selectByIds`/`selectEntities`/`entIndicesOf` for holding a selection across a reorder. |
| 1908–1955 | **ENTITY TYPES** | `ENTITY_TYPES` declares every type once. `auditEntityTypes()` runs at boot and checks each one against `sanEnt`, `entPoints`, `gripSet` and the paint router, plus reports any type present in the drawing that is not registered. Findings go to the console; `audit` at the command line reports the count. |
| 1956–1991 | **welding** | Coincident-point propagation. `nodeRefs` finds every entity endpoint sharing a location; `applyDelta` moves them together when Weld (F4) is on. |
| 1992–2084 | **transforms** | `mapEnt` is the generic point-mapper; rotate/scale/mirror are thin wrappers over it. Also group identity: `freshGroupId`, `cloneEntitiesFreshGroups`, `cloneFlowFreshIds` — copying a flowchart mints new node ids so edges bind to the copy, not the original. |
| 2085–2270 | **modify ops** | Trim to Corner (Revit semantics: click the part you keep), single/multiple trim, offset, fillet, chamfer, explode, and arrow-key nudge with acceleration. |
| 2271–2746 | **render** | The whole paint pipeline. `draw()` orchestrates: reference image → grid → entities → previews → grips → snap markers → overlays. `getCSS()` pulls theme colours from the stylesheet. `S.exporting` is checked here so effects and UI chrome never appear in a PNG or SVG. |
| 2747–2908 | **picking** | Point pick, window/crossing band, hit-index resolution, and chain tracing (`traceChainPoints`, `offsetChain`) used by Pick Lines and Measure Along. |
| 2909–3052 | **pointer** | Mouse and pen event handling. Middle-drag or Alt-drag pans, right-click finishes or cancels, left-click routes into whichever tool family is active. |
| 3053–3073 | **zoom helpers** | `zoomExtents()`. |
| 3074–3199 | **command line** | `parsePoint` handles `x,y`, `@dx,dy`, `len<ang` and bare distance-along-cursor-direction. `handleCommand` resolves snap overrides first (only while a point is being asked for, which is what keeps `SC` free for Scale), then points, then the alias table, then actions, then a switch of bare commands. |
| 3200–3218 | **toggles** | Shared toggle handler for the status-bar buttons. |

### Feature blocks

| Lines | Block | Notes |
|---|---|---|
| 3219–4038 | **MARKUP** | Paint/GIMP-style annotation over the drawing and the reference image. Pencil, marker, highlighter (multiply blend), cloud region, eraser, shapes, arrows, revision clouds, free text, callouts, sticky notes, plus style pick/apply. `mkRDP` and `mkSmooth` decimate and smooth freehand strokes. Own SVG and DXF emitters so markup survives export. Widths are entered in screen px and stored in world units at the moment of drawing. |
| 4039–4835 | **FLOW** | Relational diagramming. Node shapes (process, decision, terminator, data, note), id-bound connectors with ortho or straight routing, ray-exit point calculation so edges leave a node's true perimeter, auto text fitting, align/distribute/tidy. Deliberately platform agnostic — no vendor shapes, no compliance vocabulary. |
| 4836–5229 | **FILES & IMAGE** | Native `.b2db` save/open (round-trips what DXF and SVG throw away: markup styling, node identity, connector bindings), PNG export and clipboard copy, IndexedDB-backed autosave with localStorage fallback and `offerRestore()`, internal clipboard cut/copy/paste, layer deletion, `wipeLocalData()`. |
| 5230–5351 | **GRIPS** | One table (`gripSet`) describes every draggable handle in the app. It returns handle positions plus a setter that consumes a new world point, so the drag loop never needs to know what it is holding. |
| 5352–5412 | **SWIMLANES** | Lane headers, lane geometry, lane auto-expansion as nodes are dropped in. |
| 5413–5475 | **WAYPOINTS** | Edge routing chain: `anchor → waypoints → anchor`. Elbows are generated on top of that chain, so a waypoint index always matches a segment index and the ghost handles line up with what is drawn. |
| 5476–5593 | **UI wiring** | Snap settings widget and PNG export options widget for the ribbon. |
| 5594–5859 | **MOBILE** | Adds the missing touch input model rather than restyling the mouse one: two-finger pinch/pan, one-finger Pan mode, an aim offset (`AIM_PX`) so the point lands above your fingertip, a floating action cluster carrying Finish/Cancel/Undo and the keyboard modifiers, a bottom sheet for Layers/Styles/Properties, and touch-sized hit tolerances via `tolPx()`. The drawing engine is untouched. |
| 5860–6291 | **SECURITY** | Rejected entities are now counted by type and reported to the console and the command line rather than vanishing quietly. Threat model: everything dangerous arrives as a file or as localStorage the user did not write. Caps (`SEC.MAX_*`), `okColor` and `xmlAttr` to stop attribute breakout in exported SVG, `okImageSrc`/`sniffImage` to block remote reference-image URLs (network beacon + canvas taint), `mergePlain`/`safeParse` to block `__proto__` injection, and the `sanEnt`/`sanitizeEnts`/`sanitizeDoc` chain that every imported document goes through. Nothing from a file is trusted until it has been through here. |
| 6292–6707 | **SNAPS v2** | The Revit 2016 object snap set: Endpoint, Intersection, Midpoint, Center, Quadrant, Perpendicular, Tangent, Point, Nearest, plus recursive midpoints — halve a segment, then halve the half under the cursor, as deep as the zoom usefully allows. Tab cycles candidates. Every generator is bounded (`SNAP_MAX_ENTS` 40, `SNAP_MAX_INT_ENTS` 10, `SNAP_MAX_CAND` 200), so no drawing can make snapping quadratic. |
| 6708–8611 | **EXPERIMENTAL FEATURES** | Hidden tab, opt-in from View. Physics- and animation-driven tools grouped as Cleanup (magnet, vacuum, iron, centrifuge, level), Diagnostics (flood fill to prove an enclosure is closed, x-ray, sonar, ants, bulldozer), Layout (gravity, repel, spring, shrink wrap), Mayhem, Arcade Toys, and Presentation (time-lapse of the undo stack, spotlight, stamp). House rules enforced by `XCAP`: bounded particle/iteration/raster counts, everything undoable via `snap()`, nothing ever appears in an export. |

### UI generation

| Lines | Block | Notes |
|---|---|---|
| 8612–8653 | **keyboard** | Global keydown. Function keys map to the drafting toggles (F3 osnap, F4 weld, F6 edge assist, F7 grid, F8 ortho, F9 grid snap, F10 punch), Ctrl combos to file/clipboard/undo, Tab to snap cycling, F2 to rename, single letters to tool hotkeys. Typing into a node with it selected starts inline editing. |
| 8654–8897 | **ribbon UI** | `RIBBON` declares the tab structure. `paintRibbon()` generates every button from the `TOOLS`/`ACTIONS` metadata (label, alias, inline SVG icon), so a new tool needs no HTML. `activateRibbonItem()` is the single dispatcher. `auditRibbonButtons()` at boot catches any button referencing a tool id that does not exist. |
| 8898–9053 | **punch mode** | Test Your Might. Four hits destroys an object, destruction goes through `snap()` so Ctrl+Z brings it back. Also contains `setWorkspaceMode()` and the workspace/grid widgets. |
| 9054–9167 | **options bar** | Context strip under the ribbon that changes per tool, the way Revit's Options Bar does. Chain, Radius and Offset live here. |
| 9168–9205 | **layers UI** | Layer list, visibility, lock, colour. |
| 9206–9248 | **line styles UI** | Style list and dash pattern editing. |
| 9249–9431 | **properties** | Right-hand palette. Dispatches by entity type — CAD, markup and flow each contribute their own property rows (`mkPropRows`, `flowPropRows`). |

### Import / export

| Lines | Block | Notes |
|---|---|---|
| 9432–9530 | **DXF export** | R12 / AC1009. Widest compatibility, at the cost of no true splines or ellipses — those are written as polylines. |
| 9531–9624 | **DXF import** | R12 entity reader. |
| 9625–9667 | **SVG export** | All attribute values go through `okColor`/`xmlAttr` first. |
| 9668–10170 | **PDF vector import** | Reads content streams directly — no tracing, no OCR — because planning drawings are already vector. Hand-rolled lexer, object parser, and Flate/ASCII85/LZW decoders. Inflate reads in chunks and aborts past a ceiling: buffering the whole stream first means a decompression bomb has already won by the time you could check its size. Multi-page. |
| 10171–10308 | **file io** | Download helper, reference-image load/toggle/lock/crop/remove, new drawing. |
| 10309–10458 | **shell preferences** | Compact ribbon, draggable ribbon and command-area splitters, persisted to localStorage. |
| 10459–10490 | **boot** | ResizeObserver on the stage, `auditRibbonButtons()`, `auditEntityTypes()`, `auditReferences()`, `reseedEntSeq()`, initial paint of every UI region, workspace set to whiteboard, then two IIFEs: `initExperimental` (always starts disabled, deliberately unadvertised) and `initMobile` (respects a stored preference, otherwise falls back to `coarsePointer()` detection). |

---

## 5. History

Undo is a snapshot model, kept because it is the only thing that is reliably
correct in a codebase where entities are mutated in place all over. Three of
its costs are handled explicitly.

### Two record kinds

```
{k:"full",  blob}                      whole-document JSON
{k:"ents",  ids[], idx[], before[]}    per-entity patch
```

`snap(tag)` records a full snapshot and is the safe default — every call site
in the file can use it and be correct. `snapEnts(indices, tag)` and
`snapEntList(entities, tag)` record only the listed entities.

**A patch is only legal when the operation cannot add, remove or reorder
entities, and cannot touch layers, styles or the reference image.** If there is
any doubt, use `snap()`. The operations currently on the patch path are grip
drags, markup and flow restyle, flow align/distribute/tidy/autosize/fit-text,
and nudge with weld off. Nudge with weld on takes a full snapshot, because weld
pulls in entities outside the selection that were not known when the record was
made.

At 20,000 entities a full snapshot costs about 15 ms and a patch about 0.2 ms,
so a grip drag went from a visible hitch per drag to free.

### Byte budget

`HIST.BYTES` (64 MB) and `HIST.RECORDS` (200) bound the undo stack, trimming
oldest-first. History is therefore deep on small drawings and shallow on large
ones instead of a fixed 120 records regardless of size. A 20k-entity drawing
settles at around 42 records; a small one keeps all 200.

### Coalescing and dedupe

`snap("tag")` with a repeated tag inside `HIST.COALESCE_MS` (700 ms) folds into
the record already on the stack, so a burst of arrow-key nudges is one undo. A
blob byte-identical to the one below it is dropped, which kills the records left
behind by operations that snapshot and then bail out. `snapCancel()` removes a
record an operation decided not to need.

### Selection and the blob cache

Selection is captured as `eid`s before an undo and reapplied after, so nudging
something and pressing Ctrl+Z leaves it still selected. After an undo or redo
the applied blob is cached as the current state, so a run of repeated undos
serialises once rather than once per step.

`hist` at the command line prints record count, bytes, and the full/patch split.

---

## 6. Reference

### Snap overrides

Type the code at the command line while a point is being asked for. Applies to one pick.

| Code | Snap | Code | Snap |
|---|---|---|---|
| `SE` | Endpoint | `SP` | Perpendicular |
| `SI` | Intersection | `ST` | Tangent |
| `SM` | Midpoint | `SX` | Point |
| `SC` | Center | `SN` | Nearest |
| `SQ` | Quadrant | | |

`SS` suspends snapping, `SO` clears an override, `SR` toggles object snap entirely. Tab cycles candidates under the cursor.

### Diagnostics

| Command | Output |
|---|---|
| `hist` | Undo record count, bytes against the budget, full/patch split, redo depth |
| `audit` | Entity type and reference audit result; details go to the console |
| `refs` | List loaded references, active one marked |
| `ref N` | Make reference N active |
| `ref` / `prevref` | Cycle the active reference forward or back |

### Point entry

| Input | Meaning |
|---|---|
| `100,50` | Absolute world coordinate |
| `@100,50` | Relative to the last point |
| `@100<45` | Polar: distance and angle from the last point |
| `100` | Distance along the current cursor direction |

### Keyboard

| Key | Action | Key | Action |
|---|---|---|---|
| F3 | Object snap | Ctrl+Z / Ctrl+Shift+Z | Undo / Redo |
| F4 | Weld | Ctrl+A | Select all |
| F6 | Reference edge assist | Ctrl+C / X / V | Copy / Cut / Paste |
| F7 | Grid (or switch to drafting) | Ctrl+S | Save `.b2db` |
| F8 | Ortho | Ctrl+O | Open `.b2db` |
| F9 | Grid snap | F2 | Rename / edit text |
| F10 | Punch mode | Tab | Cycle snap candidate |
| Esc | Cancel | Del | Erase selection |
| Z | Zoom extents | Arrows | Nudge selection |

Right-click finishes a polyline or chain, otherwise cancels. Middle-drag or Alt-drag pans.

### File formats

| Format | Direction | Fidelity |
|---|---|---|
| `.b2db` | in / out | Full. The only format that round-trips markup styling, node ids and connector bindings. |
| DXF R12 | in / out | Geometry only. Splines and ellipses flattened to polylines. |
| SVG | out | Visual fidelity, no editability. |
| PNG | out | Raster, configurable width/background/margin/grid. Also copies to clipboard. |
| PDF | in | Vector linework from content streams. Multi-page. |
| PNG/JPEG/WebP | in | Reference underlay, up to `SEC.MAX_REFS` (24) per drawing, each with its own layer, calibration, straighten and crop. |

---

## 7. Extending it

**Add a CAD tool:** add an entry to `TOOLS` with `label`, `pts`, `alias`, inline `svg`, and `prompts`. Add a `case` to `commit()`. Add the id to the relevant `RIBBON` group. The ribbon button, command alias and prompt chain generate themselves.

**Add an action:** add to `ACTIONS` with a `run()`. Same ribbon wiring.

**Add a markup or flow tool:** add to `MK_TOOLS` or `FLOW_TOOLS` — the registration loops copy it into `TOOLS` and `ALIAS` automatically. Then handle it in the family's pointer functions and paint functions.

**Add an entity type:** register it in `ENTITY_TYPES` first, then touch
`entPoints`/`entBox`/`entHit` (geometry), `paintEnt` or the relevant family
router (render), `gripSet` (grips), `sanEnt` (security), and the DXF/SVG
emitters. Reload and check the console: `auditEntityTypes()` names anything you
missed, including the `sanEnt` case that would otherwise let the type render all
session and disappear on the next open.

**Rules that will bite you if you ignore them:**

- Call `snap()` **before** mutating, not after.
- Remove entities with `entKeep`/`entRemove`. A raw `splice` or `filter` on `S.ents` leaves the selection pointing at the wrong things.
- Use `snapEnts()` only for in-place edits to existing entities. Anything that adds, removes or reorders, or touches layers, styles or the reference image, needs `snap()`.
- Anything that reaches an SVG attribute goes through `okColor()` or `xmlAttr()`.
- Anything read from a file goes through the sanitizer chain.
- Sizes the user enters in screen pixels get divided by `S.view.s` before storage.
- Check `S.exporting` before painting anything that is UI rather than drawing.
- Never bypass `activateRibbonItem()`.
- Never assign to `S.refImage`. Use `refAdd` / `refRemove` / `refSetActive`.
- Anything that changes a reference's `w`/`h` after calibration invalidates the registration. Either rescale `R.cal.mm` with it or leave `cal` alone and let `refDrift` report the discrepancy — do not silently clear it.

---

## 8. Known limits

- `S.sel` is still a set of indices. `entKeep` keeps it honest across removals, but code that reorders `S.ents` by other means (there is none today) would have to remap it by hand or capture `selIds()` first.
- `auditEntityTypes()` reads function source to find `case "type":` labels. Exact for this file, wrong the moment anything minifies it.
- `eid` is session-scoped. The sanitiser strips it on load and `reseedEntSeq()` reissues, so ids in a saved `.b2db` are not meaningful across files.
- History is still snapshot-based at heart. The patch path covers the frequent in-place edits, but drawing, erasing and any structural change still serialise the whole document once per operation.
- `S.op`, the scratch state for multi-step modify tools, can hold entity indices. Nothing removes entities mid-operation today, so this is latent rather than broken.
- Reference edge snapping considers the active reference only. Snapping to whichever underlay happens to be nearest would multiply the candidate count by reference count and break the bounded-generator discipline in Snaps v2.
- `R.cal` is recorded but nothing consumes it yet. Revision overlay is the consumer it was built for.
- Saved files now carry `refs[]` and no longer write the singular `ref`, so a drawing saved by this build will not show its underlays in an older copy of the app. Reading is backward compatible in the other direction.
- Reference images are embedded as base64 PNG in `.b2db`. Two or three underlays make for a large file.
