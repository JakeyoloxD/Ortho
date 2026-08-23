# BackToTheDrawingBoard

BackToTheDrawingBoard is a single-file, offline 2D drafting, markup, diagramming, reference-image and visual communication tool.

The entire application is one HTML file. There is no build step, no server requirement, no third-party runtime dependency and no network access. Open the HTML file in a modern browser and work locally.

This README documents the current feature set in `BackToTheDrawingBoard.html`.

## Contents

1. [What the application is](#what-the-application-is)
2. [Running the application](#running-the-application)
3. [Interface overview](#interface-overview)
4. [Core interaction model](#core-interaction-model)
5. [Keyboard modifiers](#keyboard-modifiers)
6. [Snapping and drafting aids](#snapping-and-drafting-aids)
7. [Markup tab](#markup-tab)
8. [Draw tab](#draw-tab)
9. [Flow tab](#flow-tab)
10. [Insert tab](#insert-tab)
11. [Reference images](#reference-images)
12. [Manage tab](#manage-tab)
13. [Theme Forge](#theme-forge)
14. [View tab](#view-tab)
15. [Experimental Features](#experimental-features)
16. [Selection, grips and editing](#selection-grips-and-editing)
17. [Layers and line styles](#layers-and-line-styles)
18. [Properties](#properties)
19. [Markups List and review workflow](#markups-list-and-review-workflow)
20. [Command line](#command-line)
21. [Keyboard shortcuts](#keyboard-shortcuts)
22. [File formats and export](#file-formats-and-export)
23. [Autosave, undo and recovery](#autosave-undo-and-recovery)
24. [Mobile and touch](#mobile-and-touch)
25. [Security model](#security-model)
26. [Current limitations](#current-limitations)
27. [Developer architecture](#developer-architecture)
28. [Verification and self-test](#verification-and-self-test)

---

## What the application is

BackToTheDrawingBoard combines several related workflows in one canvas:

- CAD-style 2D drafting
- Revit-inspired modify and measurement interactions
- freehand and structured markup
- flowchart and process-diagram creation
- raster reference-image tracing and calibration
- vector PDF and DXF import
- PNG, SVG, DXF and markup CSV export
- layers, line styles and object properties
- application theme generation through Theme Forge
- an opt-in Experimental Features workspace for cleanup, diagnostics, layout and deliberately ridiculous tools

World coordinates are stored in millimetres. CAD geometry, markup and flow entities all live in the same drawing model, which gives them shared selection, undo, layers, movement and export infrastructure where appropriate.

The application is deliberately local-first. It does not call cloud services, analytics, AI endpoints or remote storage.

---

## Running the application

Open the HTML file directly in Chrome, Edge or Firefox.

```text
BackToTheDrawingBoard.html
```

No installation is required.

The application works from `file://` and keeps autosave data and preferences in browser-local storage mechanisms. Moving the HTML file can change the browser origin and therefore make previous autosave data unavailable from the new location.

### Recommended browser

Chromium-based browsers currently receive the most testing.

### Units

Drawing world units are millimetres.

Examples:

```text
1000 = 1000 mm = 1 m
250  = 250 mm
```

Area measurement can display `mm²`, `cm²`, `m²` or choose automatically.

---

## Interface overview

The application shell contains:

- application/document bar
- ribbon tabs
- contextual Options Bar
- drawing canvas
- status strip
- command line

### Ribbon tabs

The normal ribbon contains:

- **Markup**
- **Draw**
- **Flow**
- **Insert**
- **Manage**
- **View**

The **Experimental Features** tab is hidden by default and can be revealed with the `experimental` or `labs` command.

### Options Bar

The strip directly under the ribbon changes according to the active tool, similar to Revit's Options Bar.

Examples include:

- Line: Chain, Radius, Offset
- Copy: Multiple
- Rotate: copy state where applicable
- Measure Between: Chain and Total Length
- Measure Area: Area, Perimeter and unit
- Offset/Fillet/Chamfer: operation distance or radius
- markup tools: contextual pen controls
- flow tools: node and connector options

### Status strip

The status strip exposes frequently used drafting toggles and current information without changing tabs.

---

## Core interaction model

### Pick, preview, commit

Most drawing and modify tools follow this pattern:

1. activate a tool
2. pick the required reference point or object
3. move the cursor to preview the result
4. click or type a value to commit
5. continue if the tool supports repeated operations

### Esc

`Esc` backs out progressively.

When a tool has already collected points, the first Esc removes the latest stage where possible. Another Esc exits the operation and returns toward selection.

### Enter

`Enter` accepts or finishes tools that support an open-ended point sequence, including polyline-style operations.

### Right-click

Right-click is also used by several chain and multi-point tools to finish or cancel the active sequence.

### Numeric entry

When a tool is asking for a point, the command line accepts coordinate and distance forms.

```text
100,200       absolute point
@50,-20       relative point from last point
500<45        500 mm at 45 degrees from last point
250           250 mm along the current cursor direction
```

Rotate and Rotate Image also accept a typed angle after choosing the rotation base point.

---

## Keyboard modifiers

Modifiers are treated as application-level interaction rules rather than unrelated per-tool hacks.

### Shift

**Shift means temporary geometric constraint while a tool is actively asking for geometry.**

Current uses:

| Tool | Shift behaviour |
|---|---|
| Calibrate Image | Locks the second calibration point horizontal or vertical |
| Move Image | Constrains movement horizontally or vertically |
| Line | Constrains the next point horizontally or vertically |
| Polyline | Constrains the next segment horizontally or vertically |
| Spline | Constrains the next point horizontally or vertically |
| Move | Constrains destination horizontally or vertically |
| Copy | Constrains destination horizontally or vertically |
| Rotate | Snaps rotation to 15 degree increments |
| Rotate Image | Snaps rotation to 15 degree increments |
| Rectangle | Makes the rectangle square |
| Ellipse / Partial Ellipse | Makes the secondary radius match the first radius |
| Markup line | 45 degree constraint |
| Markup rectangle | Square |
| Markup ellipse | Circle |

Outside an active geometry operation, Shift retains selection semantics.

### Ctrl or Command

**Ctrl means additive selection or duplicate transform.**

- Ctrl + click: add an entity to the selection
- Ctrl during Move: move a duplicate and leave the original
- Ctrl during Rotate: rotate a duplicate
- Ctrl during Scale: scale a duplicate

On macOS-style keyboards, Command is accepted for the normal command shortcuts.

### Shift + click selection

Removes the clicked entity from the current selection.

### Ctrl + Shift + click

Toggles the clicked entity in or out of the current selection.

### Alt

Alt is reserved for viewport panning.

```text
Alt + drag = pan
```

It is intentionally not overloaded with transform-anchor behaviour.

### Tab

Tab is contextual:

- Select: cycle overlapping selectable entities under the cursor
- object snapping: cycle snap candidates under the cursor
- Measure Along: toggle connected-chain preview

### Space

During Move or Copy after choosing a base point:

```text
Space = rotate placement preview by 90 degrees
```

Repeated presses cycle 0, 90, 180 and 270 degrees.

### Modifier policy summary

```text
Shift  temporary geometric constraint
Ctrl   add selection / duplicate transform
Alt    viewport pan
Tab    cycle snap or selection candidates
Space  rotate move/copy placement preview by 90 degrees
Enter  finish / accept
Esc    back one stage / exit
```

---

## Snapping and drafting aids

BackToTheDrawingBoard has a Revit-inspired object snap system plus grid, image-edge and weld behaviour.

### Object snaps

The current object snap set is:

| Snap | Override |
|---|---|
| Endpoint | `SE` |
| Intersection | `SI` |
| Midpoint | `SM` |
| Center | `SC` |
| Quadrant | `SQ` |
| Perpendicular | `SP` |
| Tangent | `ST` |
| Point | `SX` |
| Nearest | `SN` |

The **Nearest** snap is important for circles and curves because it allows a point to land on the actual circumference rather than only the centre or quadrants.

### Recursive midpoints

Midpoint snapping can recursively subdivide a segment near the cursor. This makes quarter, eighth and deeper subdivision points available when the zoom level makes them useful.

### Tab cycling

When multiple snap candidates are close enough to the cursor, press `Tab` to cycle between them.

### One-shot snap overrides

Snap override commands apply to the next point only.

```text
SE endpoint
SI intersection
SM midpoint
SC center
SQ quadrant
SP perpendicular
ST tangent
SX point
SN nearest
```

### Snap control commands

```text
SS   suspend/resume snapping temporarily
SO   clear a one-shot override
SR   toggle object snap
```

### F3: Object Snap

Toggles the object snap engine.

### F4: Weld

Coincident connected endpoints move together.

This is useful when moving one end of connected wall linework and you want joined geometry to remain joined.

### F6: Image Edge Assist

Enables snapping to strong edges detected in the active reference image.

The image is locally raster-processed using an edge detector. No OCR or remote processing is used.

### F7: Grid / Drafting workspace

If the app is not already in Drafting mode, F7 switches to Drafting. Once there, F7 toggles grid visibility.

### F8: Ortho

Persistent horizontal/vertical drafting constraint.

Object snaps take priority over Ortho.

Markup interprets Ortho differently where useful, such as 45 degree line constraints and square/circle shape constraints.

### F9: Grid Snap

Rounds picked points to the nearest grid intersection.

Grid visibility and Grid Snap are independent.

### Grid spacing

The base grid step is entered in millimetres. The displayed grid pitch adapts to zoom to prevent the view becoming visually saturated.

---

## Markup tab

Markup is for review, annotation and visual communication rather than CAD construction.

Markup generally ignores CAD object snapping and grid snapping.

### Freehand group

#### Pencil

Freehand thin, opaque, smoothed annotation.

#### Marker

Thicker freehand marker with transparency.

#### Highlighter

Uses multiply blending so underlying drawing linework remains readable.

#### Lasso

Creates a freehand closed annotation loop around a region.

#### Eraser

Erases markup only. CAD and Flow entities are left untouched.

### Shapes group

#### Arrow

Single-headed annotation arrow.

#### Line

Straight markup line.

Shift or Ortho constrains it to 45 degree increments.

#### Double Arrow

Double-headed markup arrow for spans or bidirectional relationships.

#### Rectangle

Markup rectangle. Shift or Ortho makes it square.

#### Ellipse

Markup ellipse. Shift or Ortho makes it circular.

#### Revision Cloud

Cloud-shaped review boundary for changed or questioned regions.

### Text group

#### Markup Text

Free annotation text placed at a point.

#### Callout

Text with a leader back to the referenced location.

#### Sticky Note

Text in a filled box for comment-style annotation.

### Pen group

Controls current markup styling:

- colour
- width
- opacity
- fill where relevant
- fill opacity where relevant

Markup widths are entered in screen-oriented units and converted to world size at creation so they scale naturally with the drawing.

### Review group

#### Status

Assigns a review status to selected markup.

The current status vocabulary includes states such as:

- Open
- Resolved
- Rejected
- For Info
- For Action

Status is stored on the markup entity and survives save, undo and copy operations.

#### Status Display

Shows or hides fixed-screen-size coloured status dots.

Status dots are review UI and are not exported as drawing content.

### Markup group

#### Select Markup

Selection mode restricted to markup entities.

#### Pick Style

Eyedropper. Click existing markup to load its colour, width and opacity into the current pen.

#### Apply Style

Applies the current markup pen settings to clicked markup.

#### Show / Hide Markup

Globally hides or reveals markup entities without changing layers.

#### Clear Markup

Deletes all markup entities. Undoable.

---

## Draw tab

The Draw tab contains CAD-style drawing, modification, annotation, measurement, line-style and layer controls.

### Draw group

#### Line

Two-point line.

With **Chain** enabled in the Options Bar, each segment commits as an independent line entity. This is intentional so each segment can later be trimmed, filleted, moved or styled independently.

#### Polyline

Connected multi-segment polyline stored as one entity.

Finish with Enter or right-click.

#### Rectangle

Creates a closed rectangular polyline from opposite corners.

Hold Shift for a square.

#### Circle

Pick center then radius point, or type the radius.

#### Arc, 3 Point

Pick start, end and a point on the arc.

#### Arc Center

Pick centre, start point and sweep.

#### Ellipse

Pick centre and axis information.

Hold Shift during the secondary-axis stage to force a circle.

#### Partial Ellipse

Ellipse/elliptical arc defined from a centre, axis, secondary radius and angular limits.

#### Polygon

Creates a regular polygon from a centre and vertex after asking for the side count.

#### Spline

Smooth curve through picked control points.

Finish with Enter or right-click.

#### Pick Lines

Traces a connected line run and applies an offset to the chain. Pick a source line and then choose the offset side.

### Modify group

#### Select

Click objects or drag a selection window.

Window direction matters:

- left-to-right: selects only objects fully inside
- right-to-left: crossing selection, selects anything touched by the window

Selection modifiers are documented in [Keyboard modifiers](#keyboard-modifiers).

#### Move

Pick a base point and destination.

Modifiers:

- Shift: horizontal/vertical movement
- Ctrl: duplicate instead of moving original
- Space: rotate placement preview by 90 degrees

With Weld on, coincident connected endpoints can follow the move.

#### Copy

Copies selected entities using a base point and destination.

The **Multiple** option keeps the same base point for repeated placements.

Shift constrains placement. Space rotates the placement preview.

#### Rotate

Rotate selection around a chosen base point.

- Shift: 15 degree increments
- Ctrl: rotate a copy
- typed number: exact angle in degrees after base-point selection

#### Mirror

Mirror selection across a two-point axis.

#### Scale

Uniformly scales selection about a base point.

Supports typed factor/reference workflows.

Ctrl scales a duplicate.

#### Trim to Corner

Uses Revit-style keep-side semantics.

Click the part of the first element you want to keep, then the part of the second element you want to keep. The unwanted overhangs are removed and elements can extend to the intersection if necessary.

#### Trim/Extend Single

Pick the boundary first, then the piece to trim.

The tool stays active after an operation so repeated cuts can be made against new boundaries without falling back to Select after every cut.

#### Trim/Extend Multiple

Pick one boundary and then trim multiple pieces against it.

#### Offset

Pick an element and then the side to place the offset copy.

Distance is controlled through the Options Bar or typed input.

#### Fillet

Rounds the corner between two lines.

Radius is controlled in the Options Bar.

A radius of zero effectively extends the lines to meet.

#### Chamfer

Creates a straight bevel between two lines at the configured distance.

#### Array

Creates a rectangular array from the current selection using columns, rows and spacing.

#### Explode

Breaks compound geometry into simpler pieces where supported.

Examples include polylines, splines, polygons and certain Flow structures.

#### Erase

Deletes selected entities.

Equivalent to Delete/Backspace while not typing.

#### Group

Binds selected entities so selecting a member can select the group.

#### Ungroup

Returns grouped entities to independent selection behaviour.

#### Cut / Copy / Paste

The application has its own internal entity clipboard.

This is separate from the system image clipboard and preserves application-specific entity data such as styling and Flow bindings.

### Annotate group

#### Linear Dimension

Pick two measured points and a third point for dimension-line placement.

#### Text

Single-line CAD text.

Text height is converted into world units so it scales with drawing geometry.

### Measure group

Measurements are temporary and do not become drawing entities.

#### Measure Between Two References

Pick two points to display distance and angle.

With **Chain** enabled, each next pick adds another segment and updates **Total Length**.

Double-click or right-click finishes a chain.

#### Measure Along An Element

Click an existing element to measure its length.

Press `Tab` before picking to preview and measure a connected chain of linework.

#### Measure Area

Click inside an enclosed region to estimate:

- area
- perimeter

The current engine rasterises the enclosure and reports the raster resolution in `mm/cell` so the approximation is visible rather than hidden.

Area units can be:

- Auto
- mm²
- cm²
- m²

The result is temporary and is never added as an entity.

### Line Style group

Selects the active CAD line style for newly created geometry.

### Layers group

Selects the current layer, creates layers, shows all layers and can delete a layer with its contents.

---

## Flow tab

Flow provides process and diagramming entities that remain integrated with the same canvas and history system.

### Nodes

#### Process

Standard rectangular process node.

Click for the default size or drag to size.

#### Decision

Diamond decision node.

#### Terminator

Pill-shaped start/end node.

#### Data

Parallelogram input/output node.

#### Note

Diagram note node.

#### Swimlane

Creates a swimlane container. Lanes can expand as nodes are placed into them.

### Connect

#### Connector

Connects nodes by stable node ID rather than raw coordinates.

Moving a node therefore re-routes its attached connectors instead of leaving them behind.

Routing can be orthogonal or straight depending on current Flow settings.

#### Label

Adds text to a connector, useful for branch labels such as Yes/No.

#### Clear Waypoints

Removes manually added routing waypoints from selected connectors and returns them to automatic routing.

### Style

Controls Flow visual styling such as:

- node fill
- line colour
- text colour
- label/text size

### Arrange

#### Tidy

Snaps selected Flow nodes into a cleaner grid-aligned arrangement.

#### Align Left

Align to the leftmost selected edge.

#### Align Centre

Align to a common vertical centreline.

#### Align Top

Align to the topmost selected edge.

#### Align Middle

Align to a common horizontal centreline.

#### Distribute Horizontal

Evenly spaces three or more selected nodes horizontally.

#### Distribute Vertical

Evenly spaces three or more selected nodes vertically.

#### Auto Size

Changes node size to fit its text.

#### Fit Text

Shrinks text to fit the node rather than resizing the node.

---

## Insert tab

Insert handles files, references, imports, exports and raster export settings.

### Drawing group

#### Save

Saves the native `.b2db` drawing format.

#### Open

Opens a `.b2db` file and replaces the current drawing.

#### New

Starts an empty drawing.

#### Undo / Redo

Steps through drawing history.

#### Wipe Local

Clears autosave and stored preferences for the current app origin.

It does not delete files saved to disk.

### Reference group

See [Reference images](#reference-images).

### Import group

#### DXF

Imports supported DXF R12 geometry.

#### PDF

Imports vector geometry directly from PDF content streams.

This is not OCR and not raster tracing.

Multi-page PDFs are supported and prompt for a page.

### Export group

#### PNG

Raster image export using the settings in the Image group.

#### Copy Image

Copies the rendered drawing to the system clipboard as an image.

#### SVG

Exports scalable vector output.

#### DXF

Exports DXF R12 for broad CAD compatibility.

#### Markup CSV

Exports markup review data as a spreadsheet-compatible CSV.

### Image group

Controls raster export options such as:

- output width
- background
- margin
- grid inclusion

---

## Reference images

Reference images are tracing underlays rather than drawing entities.

Supported raster input includes PNG, JPEG and WebP.

A drawing can hold multiple references at once.

Each reference has its own:

- stable reference ID
- source image
- position
- width and height
- rotation
- crop
- visibility
- lock state
- layer
- calibration record

The active reference is the one affected by reference-image tools.

### Load Reference Image

Loads a raster image as an underlay and creates/manages its reference state.

### Next Reference

Cycles the active reference when several are loaded.

Command-line alternatives:

```text
ref
nextref
prevref
ref 2
```

### Move Image

Pick a base point and destination.

Hold Shift to constrain the move horizontally or vertically.

### Rotate Image

Uses the same general workflow as Draw Rotate:

1. choose rotation base point
2. choose a second point or type an exact angle
3. preview and commit

Important behaviour:

- Grid Snap works
- normal drafting snap pipeline is available
- Shift snaps the angle to 15 degree increments
- locked references cannot be rotated
- undo records the transform

### Calibrate Image

Calibrates image scale from a known real-world distance.

Workflow:

1. click first calibration point
2. click second calibration point
3. enter the real distance in millimetres

Hold Shift while placing the second point to force the calibration line exactly horizontal or vertical.

Calibration intentionally stays isolated from CAD object snaps so unrelated geometry cannot hijack the image measurement.

Calibration evidence is stored in image-pixel space:

```text
point A in image pixels
point B in image pixels
known distance in mm
```

Because the registration lives in intrinsic image coordinates, it can survive later movement, rotation, scaling and cropping of the reference.

### Straighten Image

Pick two points on the active image that should be level.

The reference rotates so the picked line becomes horizontal or vertical as appropriate.

Useful for slightly crooked scans or photographed drawings.

### Crop Image

Creates a non-destructive crop rectangle.

The original image pixels remain available.

### Reset Crop

Restores the full reference image.

### Lock Image

Prevents accidental manipulation by reference-image tools.

### Show / Hide Image

Toggles visibility without unloading the reference.

### Remove Image

Removes the active reference.

Undo can restore removed reference pixels because image elements are retained outside the JSON history blob.

### Image Edge Assist

F6 enables snapping to strong raster edges in the active reference, useful for tracing scanned plans.

---

## Manage tab

Manage is a native ribbon workspace containing the live management controls.

The current layout contains:

- Theme Forge
- Layers
- Line Styles
- Markups
- Properties

The management dock is mounted directly inside the ribbon rather than shown as a fixed-position floating overlay.

This matters because its controls use the same live DOM and event handlers as the rest of the application.

---

## Theme Forge

Theme Forge is the application-level palette generator descended from the Chroma Forge concept.

It changes the UI theme, not drawing content.

### Semantic colour tokens

Theme Forge exposes six primary tokens:

- Background
- Surface
- Primary
- Accent
- Text
- Muted

Other shell colours are derived from these tokens.

### Harmony modes

Available generation modes:

- Analogous
- Complementary
- Triadic
- Split complement
- Monochrome

### Locks

Each colour can be locked before generating a new palette.

Locked colours remain unchanged while unlocked values are regenerated.

### Preview

Theme Forge shows a compact live preview before application.

### Generate

Produces a coordinated candidate palette using the selected harmony.

### Apply

Applies the draft theme to the application shell.

### Reset

Returns to the default BackToTheDrawingBoard theme and clears Theme Forge locks.

### Persistence

Applied themes are stored as a local UI preference.

They are not written into `.b2db` drawings. Opening another person's drawing cannot restyle your application interface.

---

## View tab

### Workspace

Two presentation modes are available:

#### Whiteboard

Clean canvas presentation with no drafting grid.

#### Drafting

Grid-oriented drafting presentation with grid/grid-snap behaviour enabled as configured.

This is a presentation/workflow switch, not a different document type. Every tool remains available.

### Navigate

#### Zoom Extents

Fits all drawing content into the viewport.

Shortcut: `Z`.

#### Mobile UI

Enables touch-oriented interaction controls.

### Drafting Aids

Contains the same six drafting toggles exposed in the status strip:

- Ortho
- Show Grid
- Grid Snap
- Object Snap
- Image Edge
- Weld

### Snaps

Configures:

- enabled object snap kinds
- recursive midpoint behaviour
- dimension snap increments

### Grid

Configures base grid spacing in millimetres and display behaviour.

---

## Experimental Features

Experimental Features is permanent in the codebase but hidden by default.

Reveal or hide it from the command line:

```text
experimental
```

or:

```text
labs
```

Experimental effects are designed to be undoable and excluded from exports where they are presentation-only.

### Arcade

#### Line Surfer

Sends an animated rider along linework so you can visually follow a route.

#### Line Muncher

Arcade toy that eats along linework.

#### Gremlins

Releases small chaos agents into the drawing.

#### Termites

Chews geometry. Undo restores it.

### Mayhem

#### Black Hole

Pulls geometry toward a picked point.

#### Wrecking Ball

Hits geometry around an impact point and scatters affected objects.

#### Lightning

Strikes from a point and forks through nearby linework.

#### Bulldozer

Pick two points to define a corridor. Geometry in the corridor is shoved aside.

#### Blender

Stirs selected geometry.

#### Confetti

Presentation effect with no serious engineering justification whatsoever.

#### Earthquake

Visually shakes the drawing without permanently moving geometry.

### Physics Toys

#### Gravity

Lets selected objects fall and settle against geometry below.

#### Repel

Pushes overlapping objects apart until configured padding is achieved.

#### Spring Layout

Treats connected structure as a spring system and relaxes the layout.

#### Centrifuge

Pushes selection outward from its centre to separate dense piles.

#### Magnet

Pulls nearby endpoints together within a tolerance.

Useful for closing small gaps that prevent region detection.

#### Shrink Wrap

Creates a surrounding boundary around selected content at a configured offset.

### Weird Stuff

#### Vacuum

Removes small stray fragments left behind by editing operations.

#### Steam Iron

Steam Iron is now a **windowed tool**.

Workflow:

1. activate Steam Iron
2. click first corner of the cleanup window
3. click opposite corner
4. only entities touching that window are processed

Current behaviour:

- straightens slightly crooked CAD line-like geometry
- works on line-like markup
- rough markup strokes can become clean orthogonal markup lines
- nearby horizontal/vertical markup endpoints are reconciled to exact intersections to close tiny gaps
- entities outside the Iron window are untouched
- the tool remains active for repeated windows

Important current limitation: Iron does **not yet promote straightened markup into CAD Draw lines**. That conversion has been discussed as a future interaction because independent CAD wall segments would then be easier to move, trim and measure individually.

#### Rain Test

Pours a diagnostic flood into a region and highlights escape points, useful for finding enclosure gaps.

#### Sonar

Casts rays from a picked point to show nearby clearance and hits.

#### Marching Ants

Animates along connected linework to reveal connectivity.

#### Stamp

Drops large drawing stamps such as:

- VOID
- REVISED
- APPROVED
- AS BUILT

Stamp wording is configured in Experimental Settings.

#### Current

Persistent animated flow along linework for presentation or direction indication.

### Back Room

#### Flood Fill

Diagnostic enclosure fill. Use Measure Area when you need the actual area/perimeter readout.

#### X-Ray

Reveals linework through fills and overlaps.

#### Spirit Level

Straightens a picked line to exactly horizontal or vertical, whichever is closer.

#### Time Lapse

Replays drawing history as an animation.

#### Spotlight

Dims the drawing except around the cursor for focused presentation.

### Combat

#### Test Your Might

F10 enables Punch mode.

Clicking an object damages it. Four hits destroys it. Undo restores it.

This is intentionally stupid and intentionally permanent.

### Settings

Controls tolerances and parameters used by Experimental tools, including:

- weld/magnet radius
- Iron tolerance
- repel padding
- shrink-wrap offset
- bulldozer corridor width
- stamp wording

---

## Selection, grips and editing

### Basic selection

Click an entity to select it.

Selection-window direction follows CAD convention:

- left-to-right: full enclosure required
- right-to-left: crossing selection

### Selection modifiers

```text
Ctrl + click          add
Shift + click         remove
Ctrl + Shift + click  toggle
```

### Overlapping entities

With Select active, press Tab to cycle overlapping selectable entities under the cursor.

### Select all

```text
Ctrl+A
```

or command:

```text
all
```

### Grips

Selected geometry exposes editable grips where supported.

The grip system is polymorphic: each entity type provides its own handle set and setter behaviour.

### Arrow-key nudge

Selected entities can be nudged with arrow keys while no multi-stage tool is active.

Shift accelerates nudge distance.

### F2 editing

F2 edits supported selected text-like entities, including:

- CAD text
- markup text
- Flow labels/nodes where supported

Flow nodes can also enter inline label editing by typing while one node is selected.

---

## Layers and line styles

## Layers

Layers carry:

- name
- colour
- visibility
- lock state

New CAD geometry lands on the current layer.

Reference images also carry a layer assignment, so underlay visibility can be managed through the same layer system rather than a second parallel visibility model.

### Layer operations

- choose current layer
- add layer
- rename layer
- change colour
- show/hide
- lock/unlock
- show all
- delete layer and its contents

Deleting a layer is undoable.

## Line Styles

CAD line styles carry:

- name
- line weight
- colour
- dash pattern

Line styles apply to newly created CAD geometry such as lines, polylines, circles, arcs, ellipses, polygons and splines.

Markup styling is separate and does not use CAD line styles.

---

## Properties

Properties displays and edits state for the current selection.

Depending on entity type it can expose values such as:

- position
- dimensions
- rotation
- layer
- line style
- text
- markup styling
- review status
- Flow styling and identity-related controls
- reference-image placement and calibration information
- drawing-level defaults when nothing is selected

The exact panel is entity-type aware rather than a single fixed property list.

---

## Markups List and review workflow

Manage contains a dedicated Markups List.

The list supports filtering by:

- markup type
- status
- free-text search

Selecting a list item selects the corresponding markup entity.

Filtered markups can be zoomed to in the drawing.

A typical review workflow is:

1. annotate using Markup tools
2. assign statuses such as Open or For Action
3. use the Markups List to filter review items
4. update status as work progresses
5. export Markup CSV for external reporting

---

## Command line

The bottom command line accepts:

- tool aliases
- action aliases
- coordinate entry
- distances and angles
- snap overrides
- diagnostic/admin commands

### Point entry

```text
100,200
@50,0
500<30
250
```

### Common commands

```text
undo
redo
zoom
z
ze
ortho
grid
grid 50
snap
osnap
weld
all
restore
clearautosave
refs
ref
nextref
prevref
ref 2
layer WALLS
img
imgshow
imgdel
dxf
svg
mobile
desktop
experimental
labs
hist
history
audit
test
selftest
```

### Snap commands

```text
SE endpoint
SI intersection
SM midpoint
SC center
SQ quadrant
SP perpendicular
ST tangent
SX point
SN nearest
SS suspend/resume snaps
SO clear override
SR toggle object snap
```

Snap overrides are only interpreted as snap commands while a point is being requested. This prevents collisions such as `SC` also being used for Scale outside snap-pick context.

### Tool aliases

Most ribbon tools expose an alias in their tooltip.

Examples:

```text
L     Line
PL    Polyline
REC   Rectangle
C     Circle
A     Arc
DLI   Linear Dimension
TXT   Text
MB    Measure Between
MA    Measure Along
S     Select
RO    Rotate
MI    Mirror
SC    Scale
AR    Array
X     Explode
E     Erase
```

The tooltip is the authoritative discoverable source for each tool's current alias.

---

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| F3 | Object Snap |
| F4 | Weld |
| F6 | Image Edge Assist |
| F7 | Enter Drafting mode / toggle grid |
| F8 | Ortho |
| F9 | Grid Snap |
| F10 | Punch mode |
| Z | Zoom Extents |
| Delete / Backspace | Erase selection |
| Ctrl+A | Select all |
| Ctrl+C | Internal entity copy |
| Ctrl+X | Internal entity cut |
| Ctrl+V | Internal entity paste |
| Ctrl+S | Save `.b2db` |
| Ctrl+O | Open `.b2db` |
| Ctrl+Z | Undo |
| Ctrl+Shift+Z | Redo |
| Ctrl+Y | Redo |
| F2 | Edit supported selected text/label |
| Tab | Cycle selection/snap candidates or Measure Along chain mode |
| Space | Rotate Move/Copy placement preview 90 degrees |
| Esc | Back one stage / cancel / exit |
| Alt+drag | Pan |
| Middle-drag | Pan |

Single-letter hotkeys also activate several common tools where defined.

---

## File formats and export

## Native `.b2db`

`.b2db` is the only format intended to round-trip the complete application model.

It preserves information that generic CAD/vector formats cannot, including application-specific state such as:

- CAD entities
- layers
- line styles
- markup entities and styling
- markup status
- Flow node identity
- Flow connector binding
- reference-image placement/calibration metadata
- current drawing state required for reopening

### Backward compatibility

The loader sanitises and normalises older supported document shapes where compatibility code exists, including older singular reference-image storage.

## DXF import/export

Target format is DXF R12 / AC1009 for broad compatibility.

Trade-offs:

- splines and ellipses may be flattened to polylines
- markup styling does not map cleanly
- application-specific Flow bindings do not round-trip

## SVG export

Produces scalable vector output intended for visual fidelity rather than reopening as a native editable BackToTheDrawingBoard document.

## PNG export

Raster export with configurable:

- width
- background
- margin
- grid inclusion

## Copy Image

Copies a raster rendering to the system clipboard.

## PDF import

The application reads vector drawing operations directly from PDF content streams using an internal parser.

It supports multi-page selection.

It does not use OCR and does not depend on PDF.js.

Raster-only scanned PDF content therefore does not magically become CAD vectors. A raster reference-image workflow is more appropriate for that case.

## Markup CSV

Exports review information for markup entities, including relevant values such as:

- type
- layer
- status
- colour
- position
- length where applicable
- enclosed area where applicable

Spreadsheet formula injection is mitigated by neutralising text cells that begin with spreadsheet formula trigger characters such as `=`, `+`, `-` or `@`.

---

## Autosave, undo and recovery

### Autosave

Autosave uses IndexedDB as the primary recovery store with localStorage fallback where required.

### Restore

Use:

```text
restore
```

or the recovery prompt when available.

### Clear autosave

```text
clearautosave
```

### Undo model

History supports full-document snapshots and smaller per-entity patch records.

The safe default for structural changes is a full snapshot.

Patch records are only appropriate when an operation cannot add, remove or reorder entities and cannot alter document-wide structures such as layers, styles or reference-image state.

### Byte-budgeted history

History depth is controlled by memory usage rather than a fixed number of steps. Small drawings can therefore retain more undo records than very large drawings.

### Image recovery

Reference image elements are retained separately from the JSON snapshot representation so undoing reference removal can restore the original pixels.

---

## Mobile and touch

Mobile UI adds a touch interaction layer rather than merely enlarging desktop buttons.

Features include:

- two-finger pinch zoom
- two-finger pan
- one-finger Pan mode
- cursor/aim offset so the picked point appears above the fingertip
- floating Finish
- floating Cancel
- floating Undo
- touch-oriented modifier controls
- touch-sized hit tolerances
- mobile access to management controls

Toggle with:

```text
mobile
```

Return to desktop layout with:

```text
desktop
```

---

## Security model

The application is designed around hostile local files rather than trusted input.

### Zero network

The page uses a strict Content Security Policy and makes no network requests.

Key policy characteristics include:

- `default-src 'none'`
- no `connect-src` access
- images restricted to local data/blob sources
- no remote fonts
- no frames
- no plugins/object content
- no dynamic `eval`-style runtime code compilation

### No third-party runtime code

PDF parsing, geometry, drawing, snapping and document handling are implemented in the application itself using browser-native APIs.

### Input sanitisation

Loaded `.b2db` content passes through sanitisation before becoming application state.

Defences include:

- entity type allow-lists
- numeric/string bounds
- colour validation
- image source validation
- prototype-pollution-safe object merging
- bounded array/entity counts
- XML/SVG attribute escaping
- decompression ceilings for PDF streams

### Spreadsheet safety

Markup CSV text beginning with formula trigger characters is neutralised so opening the file in spreadsheet software does not execute user-supplied text as a formula.

---

## Current limitations

The documentation intentionally states current behaviour, including rough edges.

### Measure Area is approximate

The current area engine uses bounded raster flood fill. It reports its cell resolution so accuracy can be judged. A future analytic region entity may replace or supplement this.

### Iron does not yet promote markup to CAD

Steam Iron can straighten and close rough markup linework, but the current build leaves the result as markup. Converting rough markup-room sketches into independent CAD line entities is a logical future extension but is not currently implemented.

### PDF import is vector-oriented

Raster scans inside a PDF are not OCR'd into CAD geometry.

### DXF R12 loses richer application semantics

It cannot preserve all native styling, Flow identity, markup behaviour or modern curve types.

### Theme Forge themes the application shell

Theme Forge does not currently define a general design-system entity style engine for future web/UI mockup components. It is application UI theming today.

### Experimental tab reveal is command-line driven in this build

Type `experimental` or `labs`. A ribbon toggle function exists internally but is not currently mounted into the visible View tab.

### No cloud collaboration

There is no:

- cloud storage
- SSO
- real-time multi-user editing
- DMS integration
- AI/MCP integration
- OCR service
- external analytics

This is deliberate in the current architecture.

---

## Developer architecture

This section documents the implementation model for anyone extending the single-file app.

## Single global state

`S` is the central application state object.

Important fields include concepts such as:

```text
S.ents          drawable entities
S.layers        layer definitions
S.styles        CAD line styles
S.sel           selected entity indices
S.view          camera centre and scale
S.tool          active tool id
S.pending       points collected by the active tool
S.op            scratch state for multi-stage operations
S.undo / redo   history
S.refs          reference images
S.refActive     active reference id
S.mk            current markup settings
S.flow          current Flow settings
S.x             Experimental settings
```

## Entity model

CAD, markup and Flow content share the entity array.

Common CAD entity families include:

```text
line
pline
spline
circle
arc
ellipse
text
dim
```

Markup families include:

```text
ink
mshape
mtext
```

Flow families include:

```text
node
edge
```

Every persistent entity type must be registered and supported by the relevant dispatch paths.

## Entity identity

Entities can carry a session-stable `eid` so selection/history operations that survive reorder can hold identity instead of relying only on array slots.

Flow nodes also use node IDs because connectors bind relationally to nodes.

## One removal path

Entities should leave `S.ents` through the central removal helpers rather than ad-hoc array splicing/filtering.

This is necessary because selection, hover state, grip dragging and active operations can hold entity indices.

## Tool registries

The application uses central registries for interactive tools and one-shot actions.

Ribbon buttons are generated from metadata rather than handwritten HTML.

The major feature families register into the shared tool/action system so they receive command aliases, activation and ribbon wiring consistently.

## Ribbon dispatcher

Ribbon activation flows through one dispatcher.

This prevents tool pressed-state and one-shot action state from drifting between tabs.

## Hints

Every ribbon item has a central tooltip hint entry.

The `audit` command reports missing hints.

## Modifier layer

The current modifier policy is centralised rather than duplicated across tools.

The shared point constraint helper supports modes including:

- ortho
- square
- 15 degree angle snap
- 45 degree angle snap

Tools choose which temporary Shift constraint applies.

## Geometry helpers

The engine contains reusable vector, intersection, flattening, bounding, hit-testing and transformation helpers.

Curved entities can be flattened to point sequences where an algorithm requires a polyline approximation.

## Snaps v2

Object snap candidate generation is bounded to prevent a crafted or extremely dense drawing from making snap resolution explode quadratically.

Candidate counts, nearby entities and intersection pair counts are capped.

## References

References live in `S.refs[]` and have an active `rid`.

Each reference owns its image-to-world transform and calibration evidence.

The legacy `S.refImage` name survives as a read-only accessor for the active reference.

## History rule

Use a full `snap()` before structural mutation.

Use entity-only history patches only for operations that provably cannot change entity count/order or document-level structures.

## Security rule

All file-derived data is hostile until sanitised.

Do not bypass the security block when adding a new import path or entity property.

## Theme Forge rule

Theme Forge preference data stays outside the native drawing file. UI preference and document content are intentionally separate concerns.

## Manage ribbon rule

Manage uses the live management palette mounted inside the ribbon.

The palette must be attached to `document` before functions such as layer/style/property painters run because those painters resolve hosts through `document.getElementById()`.

Painting the palette while detached causes the blank Manage-tab failure that was previously fixed.

---

## Verification and self-test

### JavaScript syntax check

Extract the final script block and run Node syntax validation:

```bash
python3 -c "
import re
s=open('BackToTheDrawingBoard.html', encoding='utf-8').read()
open('/tmp/app.js','w', encoding='utf-8').write(re.findall(r'<script[^>]*>(.*?)</script>', s, re.S)[-1])
" && node --check /tmp/app.js && echo SYNTAX_OK
```

### In-app self-tests

Type:

```text
test
```

or:

```text
selftest
```

The current build contains pure-function self-tests for shared geometry and constraint behaviour.

### Audits

Type:

```text
audit
```

The audit checks entity registration, references and ribbon tooltip coverage.

### History statistics

```text
hist
```

or:

```text
history
```

### Reference diagnostics

```text
refs
```

### Recommended change discipline

For feature development:

1. make one coherent behavioural change
2. run syntax validation
3. run `test`
4. run `audit`
5. exercise the feature in a real browser
6. verify undo/redo
7. save and reopen if persistent data changed
8. verify unrelated ribbon tabs still mount correctly

Do not bundle architectural refactors with unrelated features when avoidable. It makes regressions much harder to isolate.

---

## Project direction

BackToTheDrawingBoard is increasingly a general visual communication platform rather than only a small CAD clone.

The current foundations already cover:

- precision drafting
- review markup
- diagrams
- references
- measurements
- visual styling
- local-first files

That makes future UI/wireframe mockup tooling a natural extension, but it is not documented above as a current feature because it has not been implemented yet.

