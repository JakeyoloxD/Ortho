# BACK TO THE DRAWING BOARD

## A C++ Course in 120 Nights

**You are not learning C++. You are building a machine that happens to require it.**

The machine is a drawing program that you can ride.

You draw a track. You put a rider on it. He pumps the transitions, clears the
loop, lands the trick or eats the ground and comes apart into two projectiles.
Then you look at the corner that threw him, and you go back to the drawing
board.

Every line of it is yours. The window is yours. The pixels are yours, one
`uint32_t` at a time. The font is yours. The physics is yours. Nothing in the
final binary was written by a stranger to make your life convenient.

---

## 0. THE DOCTRINE

### 0.1 The Laws

An idiot admires complexity. A genius admires simplicity. That is not a slogan
you put on a wall, it is a build constraint.

1. **No frameworks.** Not Qt, not SDL, not SFML, not ImGui, not Boost. Windows
   already gives you a window and a way to get bytes onto the glass. Take that
   and nothing else.
2. **No package manager.** No vcpkg, no Conan, no NuGet. If a dependency cannot
   be pasted into your tree as one header you have read end to end, you do not
   have it.
3. **No network.** Ever. Not for updates, not for telemetry, not for asset
   downloads. The old HTML build enforced this in its CSP. The native build
   enforces it by never linking a socket.
4. **No build system.** One `build.bat`. If your build takes more than two
   seconds you have done something wrong, and you will find out what tonight,
   not in month four.
5. **You own every pixel.** No Direct2D, no GDI+ shapes, no GPU. You allocate a
   buffer, you write colours into it, Windows blits it. If you want an
   antialiased line you write the antialiasing.
6. **Earn complexity.** Do not build an abstraction that has not yet caused you
   pain. `std::vector<Line>` until it hurts, then and only then something else.
7. **The program explains itself.** Tests, audits, a console, a state dump.
   A machine that cannot tell you what it thinks is a machine you are afraid of.
8. **Files are hostile.** Everything on disk was written by an enemy. Your
   parser is security work whether you like it or not.
9. **Playful is allowed.** Serious software does not have to be joyless
   software. The whole point of this project is a stick figure on a skateboard.

### 0.2 What C++ is for here

C++20 underneath, modern and unashamed. `constexpr`, `std::optional`,
`std::span`, `std::variant`, structured bindings, designated initialisers,
`<format>`, ranges where they genuinely read better. Use the standard library:
it ships with your compiler, you did not add it, and rewriting `std::sort` is
not learning, it is cosplay.

The doctrine is about **dependencies and understanding**, not about pretending
it is 1994. Write good modern C++ that you can fully account for.

What you do not do is reach for a template metaprogramming cathedral to hold
three entity types. Advanced C++ is for removing complexity. Not for
demonstrating vocabulary.

### 0.3 The inversion

The first draft of this plan ended at a native CAD application, with the
Line Surfer as an afterthought in section 50, to be ported "after the serious
machinery works."

That was backwards.

**The Line Surfer is the serious machinery.** The drafting tools exist to author
tracks. The snap engine exists so spans actually join. The undo stack exists so
you can try a corner and reject it. The file format exists so a track survives
the night. The renderer exists so you can see where he is about to die.

This reorders everything. A rendering bug is now a gameplay bug. A snapping
failure is now a hole in the terrain. The editor stops being the product and
becomes the level editor, which is a much more honest thing for it to be, and a
much better teacher, because every subsystem now has a loud, moving,
unforgiving consumer.

### 0.4 What is cut, and stays cut

The HTML build has features you are not porting. Not "later". Not "in phase 7".
Cut.

| Cut | Why |
|---|---|
| PDF import and the compressed stream parser | The wrong hard problem. Months of buffer archaeology for a feature the game never calls. |
| Reference images, calibration, straighten, edge assist | You are not tracing architectural plans. You are drawing tracks. |
| Revision overlay, `R.cal`, similarity transforms | Same. |
| DXF and SVG emitters | Nothing consumes them. Your format is your format. |
| Markup status, markup CSV, the Markups List | Redline workflow. No rider cares. |
| The region engine, spaces, hatch, area takeoff | The single largest chunk of the old roadmap, and completely orthogonal to a game. |
| The Flow tab and flowchart entities | Different application wearing the same coat. |
| The ribbon | Chrome. Keyboard and a command line for months. A beautiful ribbon controlling nothing is a very polished failure. |
| Mobile mode, touch plumbing | Not this machine. |

If you miss one of these later, that is evidence, and evidence is allowed to
change the plan. But do not carry them from day one out of loyalty to the old
build.

### 0.5 What survives, and why the game needs it

| Kept | The game's reason |
|---|---|
| `Vec2` and the geometry kernel | Every physics step. |
| Viewport, world/screen, pan, zoom at cursor | Editing a track, then chasing a rider across it. |
| Entity storage, stable IDs | Terrain is entities. Terrain that renumbers itself mid-run is terrain that teleports him. |
| Layers and styles | Ride surfaces, decoration, kill zones, boost pads. Layers become **material**. |
| The full drafting toolset | You chose this. Tracks deserve a real editor. |
| The snap engine, including recursive midpoint | A 3 pixel gap between two spans is a rider-sized hole. Snapping is how the terrain stays welded. |
| Trim to corner with Revit semantics, offset, mirror, rotate | Track authoring is mostly modification, not creation. |
| Undo | The whole design loop: draw, ride, hate it, undo. |
| Self-tests and audits | Your only defence in a codebase where a sign error is a rider stuck in a wall. |
| The command console | `test`, `audit`, `ents`, `sel`, `spans`, `phys`. |
| Experimental toys | Gravity, spring, black hole, ants, flood fill. They become physics exercises and gameplay modifiers instead of curiosities. |

### 0.6 What is new, that the CAD path never needed

This is the part no C++ tutorial gives you and no CAD roadmap contains.

- A fixed timestep with an accumulator, and interpolated rendering.
- A software rasteriser fast enough to redraw the world 60 times a second.
- Span extraction: turning arbitrary hand-drawn entities into directed terrain.
- Surface normals, and the concept of which **face** of a stroke you are on.
- Continuous collision. A rider at 900 px/s tunnels through a 1 pixel line if
  you test discretely.
- A broadphase, because 14,000 spans times 240 physics ticks is not free.
- The air state, ballistic integration, and re-entry.
- Camera follow with lookahead and deadzone.
- Particles, screen shake, and a synth you wrote yourself.
- Scoring: tricks, combo, flow multiplier, bail detection.
- Determinism, so a replay replays and a ghost is a ghost.

### 0.7 The nightly template

Every night in this course is written as five parts. Use them when you invent
your own.

```
# Tonight: <one problem>
## Why          what am I trying to make happen or understand
## Start        what already works
## Explore      the C++, the maths, the API
## Win          the smallest visible thing that proves tonight happened
## Stop         what must be true before I close the laptop
```

A night that ends with three tests, a deleted experiment and a better idea of
the design still counts. A night that ends with 400 lines you do not understand
does not.

### 0.8 Rabbit holes

Follow a rabbit hole when it exposes a bad assumption in the thing you are
currently building. Snapping is wrong because screen/world conversion is wrong:
go fix screen/world conversion, that is the same problem wearing a hat.

Do not follow: snapping, then themes, then HSL colour pickers, then multiplayer.
That is not tinkering. That is getting mugged by your own imagination.

### 0.9 The journal

`docs/journal/2026-08-28-window.md`. One entry per night.

```
# Tonight
## Goal
## What I learned
## What worked
## What failed
## Weird thing I found
## Next interesting problem
```

A sentence per heading is enough. The point is to preserve your reasoning, not
to produce documentation. In three months you will want to know why you chose
snapshots over inverse commands, and your future self is a stranger with no
context.

---

## THE MAP

| Part | Nights | You end up with |
|---|---|---|
| I. The Machine Boots | 1 to 10 | A window, a backbuffer, a stable frame loop |
| II. The Pixels Are Yours | 11 to 26 | A rasteriser: lines, curves, fills, blending, your own font |
| III. The Coordinate Universe | 27 to 34 | Vec2, Viewport, pan, zoom at cursor, grid |
| IV. The Document | 35 to 48 | Entities, IDs, layers, styles, hit testing, selection, a spatial grid |
| V. Tools Are State Machines | 49 to 62 | Command registry, console, the full draw toolset |
| VI. Snapping | 63 to 72 | A snap engine with the Revit set and Tab cycling |
| VII. Undo and Modify | 73 to 86 | History, move, rotate, trim to corner, grips |
| VIII. Files Are Hostile | 87 to 94 | `.b2db` written, read, validated, fuzzed |
| IX. The Track Becomes Terrain | 95 to 112 | The Line Surfer. Spans, contact, air, loops, tricks, bails |
| X. The Engine Earns Its Name | 113 to 120 | Determinism, replays, camera, audio, HUD, the review |

Directions, not deadlines. Night 63 might take a week. Night 7 might take
twenty minutes. Nobody is timing you.

---

# PART I. THE MACHINE BOOTS
### Nights 1 to 10

No IDE ceremony. You get a compiler, a text editor, and a batch file. Visual
Studio is welcome later as a debugger, and it is an excellent one, but you
should never be in a position where you cannot explain what its Build button
did.

**The build, in full:**

```bat
@echo off
call "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat" >nul
cl /nologo /std:c++20 /W4 /permissive- /EHsc /Zi /Od ^
   src\main.cpp /Fe:b2db.exe /link user32.lib gdi32.lib
```

One translation unit to start. Yes, really. A unity build compiles this whole
project in well under a second for a long time, and fast feedback is worth more
to you right now than tidy module boundaries. You will split it when it hurts,
around Night 35, and splitting it is itself a lesson.

`/W4 /permissive-` from the first minute. Warnings are the compiler telling you
about tonight's bug before you have written it.

### Night 1. Make the compiler obey you
**Why** Understand the path from text to executable without a GUI in the way.
**Start** Nothing.
**Explore** `cl`, `/Fe`, object files, the linker, what `vcvars64.bat` sets up,
Debug versus Release, `/Zi` and the PDB.
**Win** `b2db.exe` prints its own name to a console.
**Stop** You can say out loud what `.cpp` to `.obj` to `.exe` means, and you have
deliberately caused one linker error and read it.

### Night 2. A window exists
**Why** Everything visible happens inside this.
**Start** A compiler you control.
**Explore** `WinMain`, `WNDCLASSEXW`, `RegisterClassExW`, `CreateWindowExW`,
`HWND`, the Windows type prefixes, `WndProc`, `DefWindowProcW`.
**Win** A window titled BackToTheDrawingBoard opens, and closes cleanly.
**Stop** You know why `WndProc` is a callback and who calls it.

### Night 3. Understand what Windows is saying to you
**Why** The message loop is the heartbeat. Do not treat it as boilerplate.
**Start** A window.
**Explore** `GetMessage` versus `PeekMessage`, `TranslateMessage`,
`DispatchMessage`, `WM_CREATE`, `WM_SIZE`, `WM_PAINT`, `WM_MOUSEMOVE`,
`WM_DESTROY`, `WM_QUIT`.
**Win** Log every message you receive for five seconds and read the list.
**Stop** You understand why a game uses `PeekMessage` and a text editor does not.

### Night 4. The backbuffer
**Why** This is the moment you take the pixels.
**Start** A message loop.
**Explore** `std::vector<std::uint32_t>`, `BITMAPINFO`, `StretchDIBits`,
top-down bitmaps and the negative `biHeight`, why 32bpp `BI_RGB` is
`0x00RRGGBB` in memory order B,G,R,X on a little-endian machine.
**Win** The window fills with one colour that you chose, from a buffer you own.
**Stop** You can change one integer in your code and one specific pixel changes
on screen.

### Night 5. Resize without breaking
**Why** Every graphics bug you will ever have is first found by dragging a
corner.
**Start** A backbuffer.
**Explore** `WM_SIZE`, reallocation strategy, `GetClientRect`, stride, why you
should not resize inside `WM_PAINT`.
**Win** Drag the window to any size, no crash, no stretching, no garbage.
**Stop** Minimise and restore works. A zero-height client area does not divide
by zero.

### Night 6. Graduate to a DIB section
**Why** `StretchDIBits` from a vector copies. A DIB section does not.
**Start** A working but naive present path.
**Explore** `CreateDIBSection`, `CreateCompatibleDC`, `SelectObject`, `BitBlt`,
GDI object lifetime, and your first real RAII wrapper.
**Win** Same picture, one less copy per frame, and a `struct Backbuffer` whose
destructor releases the GDI objects.
**Stop** You leak nothing. Check with Task Manager's GDI object column: draw
10,000 frames, the count is flat.

### Night 7. RAII, because Windows gave you something to release
**Why** This is the C++ lesson that only lands when a real resource is at stake.
**Start** A `Backbuffer` that owns an `HBITMAP` and an `HDC`.
**Explore** Constructor acquires, destructor releases. The rule of five. Deleted
copy, defaulted move. Why a raw `HBITMAP` member in a copyable struct is a
double-free waiting for you.
**Win** `Backbuffer` is move-only and correct.
**Stop** You made it copyable on purpose, watched it double-free, and then
deleted the copy constructor.

### Night 8. Time
**Why** Everything after Part IX depends on knowing exactly how long a frame was.
**Start** A loop that spins.
**Explore** `QueryPerformanceCounter`, `QueryPerformanceFrequency`,
`std::chrono::steady_clock`, why `timeGetTime` is not enough, what `Sleep(1)`
actually does to you.
**Win** Window title shows a stable frame time in milliseconds.
**Stop** You know your real frame time and whether your compositor is pacing you.

### Night 9. The fixed timestep
**Why** Variable dt physics is non-deterministic physics, and a rider who
behaves differently on a fast machine is a rider you cannot debug.
**Start** Accurate timing.
**Explore** The accumulator pattern, simulation rate versus render rate, the
spiral of death and how to cap it, render interpolation with `alpha`.
**Win** A counter increments exactly 240 times per second regardless of how
many frames you drew.
**Stop** You force a 500 ms hitch (sleep in the loop) and the sim does not
explode or fast-forward forever.

```cpp
constexpr double kSimHz = 240.0;
constexpr double kDt    = 1.0 / kSimHz;

double accumulator = 0.0;
while (running) {
    pumpMessages();
    accumulator += std::min(frameSeconds(), 0.25);   // cap: no spiral of death
    while (accumulator >= kDt) {
        previous = current;
        step(current, kDt);
        accumulator -= kDt;
    }
    const double alpha = accumulator / kDt;
    render(lerp(previous, current, alpha));
}
```

### Night 10. Input
**Why** A tool cannot be a state machine until events arrive in a defined shape.
**Start** A frame loop.
**Explore** `WM_LBUTTONDOWN`, `WM_MOUSEMOVE`, `WM_MOUSEWHEEL`, `WM_KEYDOWN`
versus `WM_CHAR`, `SetCapture` and `ReleaseCapture`, `GET_X_LPARAM`, why raw
input exists and why you do not need it yet.
**Win** A `struct InputEvent` with a kind, a screen position, and modifier
flags. `WndProc` translates and forwards. Nothing else.
**Stop** Your `WndProc` contains no application logic. It is a translator. Keep
it boring forever.

> **Law.** Windows message goes in, application event comes out. The day
> `WM_LBUTTONDOWN` contains 170 lines of geometry is the day this project
> becomes someone else's problem.

---

# PART II. THE PIXELS ARE YOURS
### Nights 11 to 26

You chose the software rasteriser. Good. This is the part of the course that
teaches the most C++ per hour, because every function is small, pure, testable,
and instantly visible when it is wrong.

Two rules before you start.

**Coordinates.** World coordinates are `double`. The rasteriser takes fixed
point, 24.8, as `std::int32_t`. Converting once at the boundary kills an entire
class of wobble where a static line shimmers as the camera moves by a
subpixel. Do not rasterise from doubles directly and do not round to integers
too early.

**Clip first.** Clip the primitive to the buffer rectangle before you touch
memory. A per-pixel bounds check inside the inner loop is both slower and a
worse design, because it hides the case where you generated 4 billion pixels of
off-screen line.

### Night 11. One pixel
**Why** The atom.
**Start** A backbuffer.
**Explore** Row stride, pointer arithmetic, `std::span`, why you write
`buf[y * stride + x]` and not a 2D array.
**Win** `putPixel(x, y, colour)`. A red dot at a coordinate you chose.
**Stop** Off-buffer coordinates are impossible, not merely unlikely.

### Night 12. Colour
**Why** Get the representation right once.
**Start** Raw `uint32_t`.
**Explore** `struct Rgba { std::uint8_t b, g, r, a; };` versus a packed
`uint32_t`, `constexpr` constructors, endianness, straight versus premultiplied
alpha, why premultiplied makes blending one multiply cheaper and compositing
correct.
**Win** A `Colour` type with `constexpr` named constants and a `blend` that is
provably correct at a = 0 and a = 255.
**Stop** A unit test asserts that blending anything over itself at full alpha is
the identity.

### Night 13. Clear, fill, and the dirty rectangle
**Why** Speed comes from not doing work.
**Start** `putPixel`.
**Explore** `std::fill`, memset semantics, row-wise fill, a `Rect` type with
intersection and union, tracking a dirty region across a frame.
**Win** `fillRect` and a `Rect` with tested `intersect`.
**Stop** You can clear only what changed and see the difference in your frame
time counter.

### Night 14. Bresenham
**Why** The first real algorithm, and the one that makes the machine feel like
yours.
**Start** `putPixel`, `Rect`.
**Explore** The integer error term, all eight octants, why the naive DDA with
floats drifts, Cohen-Sutherland or Liang-Barsky clipping.
**Win** A line from any coordinate to any other coordinate, correct in every
octant, clipped before the loop.
**Stop** A test draws 10,000 random lines with coordinates from minus a million
to plus a million and nothing writes out of bounds. Prove it with a guard page
or a canary row.

### Night 15. Wu, and why the CAD look needs it
**Why** Aliased lines look like a toy. Antialiased ones look like a drawing
program.
**Start** Bresenham.
**Explore** Xiaolin Wu's algorithm, coverage as alpha, the fractional endpoint
cases, gamma and why your AA looks thin on a dark background.
**Win** A smooth 1 px line at any angle.
**Stop** A near-horizontal line at 0.3 degrees has no visible stair steps.

### Night 16. Thick lines, caps and joins
**Why** Every stroke in your editor has a width.
**Start** An AA line.
**Explore** Expanding a segment into a quad by the perpendicular, round caps,
the miter join and the spike problem at sharp angles, the miter limit, why
bevel is the sane default.
**Win** A 6 px polyline with round caps that does not have a hole at each vertex.
**Stop** A polyline doubling back on itself 180 degrees does not produce a spike
to infinity.

### Night 17. The scanline polygon fill
**Why** Filled shapes, and later the rider, the board, and every particle.
**Start** Lines.
**Explore** The active edge table, sorting crossings per scanline, even-odd
versus nonzero winding, half-open intervals so adjacent polygons do not double
blend the shared edge.
**Win** Fill an arbitrary concave polygon correctly.
**Stop** A self-intersecting star fills differently under even-odd and nonzero,
and you can explain which one you want and why.

### Night 18. Circles and arcs
**Why** Half your track will be transitions.
**Start** A rasteriser.
**Explore** Midpoint circle, the eight-way symmetry, why symmetry breaks for
arcs, and the alternative: adaptive subdivision into line segments with a sag
tolerance.
**Win** `drawArc(centre, r, a0, a1)` that is smooth at any zoom.
**Stop** You picked subdivision or direct rasterisation on purpose. Write the
reason in the journal. Subdivision is probably right, because everything
downstream (hit testing, snapping, terrain spans) wants a point list anyway.

### Night 19. Flattening as a shared idea
**Why** One function, consumed by rendering, hit testing, snapping, export and
the physics span builder.
**Start** Arcs.
**Explore** Sag tolerance in **screen** space, not world space, so a circle
subdivides more when you zoom in. `std::vector<Vec2>` return versus an output
iterator, `std::back_inserter`, why the output iterator version stops allocating
in the inner loop.
**Win** `flatten(entity, tolerancePx) -> std::vector<Vec2>`.
**Stop** This is now the single source of truth for "what points is this shape
made of". Nothing downstream re-derives it.

### Night 20. Blending and the alpha compositor
**Why** Selection highlights, previews, ghosts, particle glow.
**Start** Colour and fills.
**Explore** Source-over, premultiplied maths, additive blending, saturating
arithmetic, and why `(a * b + 127) / 255` and `(a * b + 128) >> 8` differ.
**Win** A translucent preview stroke drawn over the committed drawing.
**Stop** 255 layers of 1 % alpha white on black lands where you expect, not 40
units short.

### Night 21. Your own font, part one
**Why** Every HUD, label, dimension and console line. Owning the font is the
most Terry thing in this course and it takes one evening.
**Start** A rasteriser.
**Explore** A `constexpr std::array<std::uint8_t, 96 * 16>` bitmap font, 8 wide
by 16 tall, one byte per row, bit 7 leftmost. ASCII 32 to 127.
**Win** `HELLO` on screen in a font whose bits are in your source file.
**Stop** You hand-edited one glyph and it changed on screen.

### Night 22. Your own font, part two
**Why** Fixed size text at every zoom looks wrong. Text needs to be usable.
**Start** A bitmap font.
**Explore** Integer scaling versus resampling, drawing text at 2x and 3x, a
`measureText` that agrees exactly with `drawText`, kerning (you do not have any,
and that is fine).
**Win** `drawText(p, str, scale, colour)` and `measureText(str, scale)` that are
consistent to the pixel.
**Stop** A right-aligned HUD number does not jitter as the digits change.

### Night 23. The console
**Why** Law 7. The program explains itself, starting now.
**Start** Text rendering.
**Explore** A ring buffer of lines, `std::deque<std::string>` versus a fixed
array, `std::format`, timestamps, severity colours, a slide-in animation if you
are feeling generous.
**Win** `echo("something happened")` appears on screen and scrolls.
**Stop** You use it for the rest of the course instead of `printf` into a
console window you cannot see.

### Night 24. Golden image tests
**Why** A rasteriser regression is invisible until it is catastrophic.
**Start** A rasteriser and a console.
**Explore** Rendering to an offscreen buffer, a cheap hash (FNV-1a over the
pixels), storing expected hashes in the source, writing a PNG or BMP dump on
mismatch so you can look at it.
**Win** `test` in the console renders twelve reference primitives and compares
hashes.
**Stop** You deliberately break Bresenham's octant 5 and the test catches it.

### Night 25. Dirty rectangles and the two modes
**Why** The editor is idle 99 % of the time. The game never is.
**Start** A working renderer.
**Explore** Full redraw versus dirty region, why a game loop makes dirty rects
pointless, and how to have both without two renderers.
**Win** Editing mode redraws only what changed. Play mode redraws everything at
60 Hz.
**Stop** Frame time in editor mode with a 5,000 entity drawing and no input is
under a millisecond.

### Night 26. The crosshair, the grid, and screen-space constants
**Why** The drawing surface starts feeling like a real tool.
**Start** Everything above.
**Explore** Major and minor grid lines, adaptive grid spacing by zoom level
(1, 2, 5, 10 progression), why the crosshair is drawn in screen space and the
grid in world space, and a `px(n)` helper that means "n screen pixels, whatever
the zoom".
**Win** A dark CAD canvas with a grid that stays useful from 0.01x to 100x.
**Stop** The grid never draws 40,000 lines because you zoomed out.

> **Law.** Anything measured in pixels stays in pixels. Snap radius, grip size,
> line hit tolerance, glyph size, the sag tolerance. Convert at the boundary.
> The old HTML build had `px()` and `u()` for exactly this and the discipline is
> worth carrying over verbatim.

---

# PART III. THE COORDINATE UNIVERSE
### Nights 27 to 34

Small part, enormous leverage. Everything downstream is wrong if this is wrong,
and it will be wrong in ways that look like other bugs. A snapping problem that
turns out to be a zoom problem is the most common rabbit hole in this entire
project.

### Night 27. Vec2
**Why** The atom, again, one level up.
**Start** Nothing.
**Explore** `struct` with default member initialisers, `constexpr`, free
operator overloads versus members, why you pass `Vec2` by value and not by
const reference, `[[nodiscard]]`.
**Win**
```cpp
struct Vec2 {
    double x{}, y{};
};
constexpr Vec2 operator+(Vec2 a, Vec2 b) { return {a.x + b.x, a.y + b.y}; }
constexpr Vec2 operator*(Vec2 a, double s) { return {a.x * s, a.y * s}; }
constexpr double dot(Vec2 a, Vec2 b) { return a.x * b.x + a.y * b.y; }
constexpr Vec2 perp(Vec2 a) { return {-a.y, a.x}; }
double length(Vec2 a);
Vec2 normalise(Vec2 a);           // what does this return for a zero vector
```
**Stop** You decided what `normalise({0,0})` does and it is documented in a
comment explaining why.

### Night 28. A test runner in forty lines
**Why** Law 7, and because geometry is where silent sign errors live.
**Start** Vec2.
**Explore** `assert` versus your own `CHECK` macro, floating point comparison
with an epsilon, why `==` on doubles in a test is usually a lie, `__LINE__` and
`#x` stringification.
**Win** `test` in the console runs your Vec2 suite and prints `12 passed`.
**Stop** A failing test names the file, the line and the two values. No
framework was harmed.

> Pure functions only. If a test needs the document, the viewport or the
> backbuffer, it is an audit, not a test, and it goes in the other list. The old
> build kept `SELFTEST` and `audit` separate for exactly this reason. Keep them
> separate.

### Night 29. The Viewport
**Why** The only thing allowed to know about both worlds.
**Start** Vec2.
**Explore** Classes with invariants, private state, `const` member functions,
scale as a single scalar versus a 2x2 matrix, whether y points down (screen) or
up (maths) and committing to an answer.
**Win**
```cpp
class Viewport {
public:
    Vec2 worldToScreen(Vec2 p) const;
    Vec2 screenToWorld(Vec2 p) const;
    double px(double screenPixels) const;   // screen px -> world units
private:
    Vec2   origin_{};      // world point at screen (0,0)
    double scale_{1.0};    // screen pixels per world unit
};
```
**Stop** A test asserts `screenToWorld(worldToScreen(p)) == p` within 1e-9 for a
hundred random points and scales.

### Night 30. Pan
**Win** Middle-drag moves the world. The point under the cursor at drag start is
still under the cursor at drag end, exactly.
**Stop** No drift after fifty drags. Drift means you are accumulating in screen
space instead of anchoring to a world point.

### Night 31. Zoom
**Win** Wheel changes scale, multiplicatively, not additively.
**Explore** Why `scale *= 1.1` and not `scale += 0.1`, clamping the range,
`WM_MOUSEWHEEL` delta and `WHEEL_DELTA`, accumulating fractional wheel ticks
from a precision touchpad.
**Stop** Zoom feels the same at 0.01x and at 100x.

### Night 32. Zoom around the cursor
**Why** The single interaction that separates a real tool from a student
project.
**Explore** Anchor the world point under the cursor, apply the scale, solve for
the new origin.
**Win** The pixel under the mouse does not move while you scroll the wheel.
**Stop** Hold a mouse position over a specific grid intersection and wheel in
and out forty times. It has not crept.

### Night 33. Fit and zoom to extents
**Why** You will use this every time you load a track.
**Explore** Bounding box union, aspect ratio, margin in screen pixels, the
degenerate cases: empty document, a single point, a horizontal line with zero
height.
**Win** `zoomExtents()` frames everything with a 5 % margin.
**Stop** An empty document does not divide by zero and does not scale to
infinity.

### Night 34. Interpolated view for the chase camera
**Why** Foreshadowing Part X. Do it now while the viewport is simple.
**Explore** Smoothing with an exponential decay that is framerate independent:
`a = 1 - exp(-rate * dt)`, and why the naive `a = 0.1` per frame is a different
camera at 30 fps and 144 fps.
**Win** `Viewport::followTo(target, dt)` that eases and never overshoots.
**Stop** The same follow behaviour at 30, 60 and 240 fps.

---

# PART IV. THE DOCUMENT
### Nights 35 to 48

Also the night you split `main.cpp`. Around 3,000 lines a unity build stops
being a pleasure. Split by responsibility, not by type: `geom`, `raster`,
`doc`, `tools`, `app`.

### Night 35. LineEntity, and the discipline of not abstracting
**Why** You do not need architecture because architecture sounds impressive.
**Start** Vec2.
**Explore** `class` versus `struct`, aggregate initialisation, designated
initialisers, `std::vector`, range-based for, `const auto&`.
**Win** `std::vector<LineEntity>` renders on screen.
**Stop** You resisted writing a base class. There is nothing to be polymorphic
over yet.

### Night 36. The Document
**Explore** Moving entity storage out of the window code, why the renderer takes
`const Document&`, what the document is not allowed to know about (the
viewport, the backbuffer, the current tool).
**Win** `class Document` owns entities. `render(const Document&, Viewport, Buf&)`
is a free function.
**Stop** The document header does not include Windows.h. It never will.

### Night 37. Stable identity
**Why** Array indices are a lie that survives exactly until the first delete.
**Explore** `using EntityId = std::uint64_t;`, a monotonic counter, 0 as the
null id, `std::unordered_map<EntityId, size_t>` for lookup, or a sorted vector
and binary search.
**Win** Every entity has an id that never changes and is never reused.
**Stop** You can delete entity 3 of 10 and entity 7 is still entity 7.

> **Law.** The old build deferred this and paid for it. `S.sel` held indices, and
> every removal had to remap the selection, the hover, a live grip drag and the
> in-progress operation. Four things, one `splice` away from silent corruption.
> Do it on Night 37 where it costs nothing.

### Night 38. Many entity types: the experiment
**Why** Do not choose inheritance because a tutorial did.
**Explore** Build the same thing twice.
```cpp
// A
struct Entity { virtual ~Entity() = default; virtual Rect bounds() const = 0; };
std::vector<std::unique_ptr<Entity>> ents;

// B
using Entity = std::variant<Line, Polyline, Arc, Circle, Text, Ink>;
std::vector<Entity> ents;
```
**Win** Both compile and render. You measured the cache behaviour of each with
50,000 entities.
**Stop** You chose one and can defend it in two sentences. For this project,
`variant` plus `std::visit` is probably right: the type set is closed, you own
all of it, and the physics wants tight contiguous data. But you decided, not me.

### Night 39. Ownership
**Explore** `std::unique_ptr`, `std::make_unique`, move semantics, why
`std::shared_ptr` is almost always the wrong answer in a program with one
document, and what happens to references into a `vector` when it reallocates.
**Win** A clean answer to "who owns entities, who may reference them, when do
they die".
**Stop** You have `push_back`ed into a vector while holding a reference into it,
watched it become garbage, and understood iterator invalidation forever.

### Night 40. Bounds
**Win** `bounds()` for every entity type, `const` correct.
**Explore** `const` as an interface promise, arc bounding boxes and the quadrant
extremes (a 270 degree arc's box is not its endpoints' box), a cached bounds with
a dirty flag and why caching now is premature.
**Stop** An arc crossing the top quadrant has the right bounding box. Draw the
box on screen to check. This is the most commonly wrong function in any CAD
program.

### Night 41. Hit testing
**Explore** Point to segment distance, tolerance in screen pixels, point to arc,
point inside polygon, the ray casting method.
**Win** Click near a line, its id prints to the console.
**Stop** Tolerance is constant in screen pixels at every zoom.

### Night 42. Selection
**Explore** `std::unordered_set<EntityId>`, `std::vector<EntityId>` and which
one you want, why the answer depends on whether selection order matters.
**Win** Click to select. Selected entities render highlighted.
**Stop** Selecting 50,000 entities does not take a visible amount of time.

### Night 43. Modifier semantics
**Win** Shift removes, Ctrl adds, click on empty space clears.
**Stop** The behaviour matches the HTML build. Open it and check rather than
guessing. That is what the old build is for.

### Night 44. Box and crossing selection
**Explore** The left-to-right versus right-to-left convention from every CAD
program, rectangle contains versus rectangle intersects, segment versus
rectangle intersection.
**Win** Both directions work and are visually distinguished (solid box versus
dashed).
**Stop** A crossing selection catches an entity whose bounding box overlaps but
whose geometry does not. Then fix it, because that is wrong.

### Night 45. Tab cycling overlapping entities
**Win** Repeated clicks in the same place cycle through everything under the
cursor.
**Stop** Moving the mouse resets the cycle.

### Night 46. Layers as material
**Why** This is the design decision that makes the game work.
**Explore** A layer table, id, name, colour, visibility, lock. Then the twist:
add a `Material` enum. `Ride`, `Decoration`, `Kill`, `Boost`, `Ice`.
**Win** The physics later asks a span for its layer and gets a material.
Terrain authoring becomes layer management.
**Stop** An entity on a Decoration layer is drawn but is not terrain. You have
just made the level editor real.

### Night 47. Line styles
**Explore** A style table, width, dash pattern, colour. Dash pattern
rasterisation with a phase accumulated along the polyline so dashes do not
restart at every vertex.
**Win** Dashed and dash-dot strokes that look right around corners.
**Stop** A dashed circle has evenly spaced dashes.

### Night 48. The spatial grid
**Why** Everything from here (picking, snapping, the broadphase) needs it.
**Explore** A uniform grid of cells, cell size chosen from the median entity
size, `std::unordered_map<CellKey, std::vector<EntityId>>` versus a flat array
with a hash, insertion and removal, rebuild versus incremental update, why a
quadtree is not obviously better and is definitely more code.
**Win** Query "everything near this point" in constant time.
**Stop** Picking in a 50,000 entity drawing is instant, and there is an audit
command that verifies the grid agrees with a brute force scan.

---

# PART V. TOOLS ARE STATE MACHINES
### Nights 49 to 62

The central insight of the whole editor. A drawing tool is a tiny state machine,
and the moment you see it that way, twenty tools become one pattern with twenty
bodies.

```
idle --click--> haveFirstPoint --move--> preview --click--> commit --> idle
                     |                                        ^
                     +----------------- Esc ------------------+
```

### Night 49. The Tool interface
**Explore** A virtual interface versus a variant of tool structs versus a
function-pointer table. Pick one. `onDown`, `onMove`, `onUp`, `onKey`,
`onCancel`, `drawPreview`.
**Win** A `ToolManager` holding the active tool and routing input events.
**Stop** The app has one place where a tool receives a click.

### Night 50. LineTool
**Explore** `std::optional<Vec2>` for "the first point, if we have one", state
as data rather than as a bool pair.
**Win** Click, click, a line exists in the document.
**Stop** The tool has no idea a document is a vector, and the document has no
idea a tool exists.

### Night 51. The preview lives in the tool
**Why** The single most important separation in the program.
**Explore** Document holds things that exist. Tool holds things that might
exist. This principle later governs move previews, rotation previews, selection
boxes, snap glyphs, measurement previews and the rider himself.
**Win** A rubber-band line follows the cursor and is not in the document.
**Stop** Undo has nothing to undo while a preview is on screen.

### Night 52. Esc, and cancellation as a first-class idea
**Win** Esc from the second point returns to the first-point state. Esc again
exits the tool. Right-click does something you decided on.
**Stop** Behaviour matches the HTML build. Go and test it there.

### Night 53. The command registry
**Why** Five implementations of Rotate is how a program dies.
**Explore** `std::unordered_map<std::string, Command>` where a `Command` is a
name, an alias, a hint and a `std::function<void(App&)>`, or a plain function
pointer if you would rather see what is happening.
**Win** Keyboard, console and (later) any button all dispatch through one path.
**Stop** There is no second way to start the Line tool.

### Night 54. The console command line
**Explore** Tokenising, an alias table, command history with up-arrow,
autocomplete, error reporting.
**Win** Type `line`, `L`, `zoom extents`, `test`, `audit`. All work.
**Stop** An unknown command says so and suggests the nearest match by edit
distance, because you now want to write Levenshtein and it is thirty lines.

### Night 55. Aliases and hotkeys, plus the audit that keeps them honest
**Why** The old build learned this the hard way and encoded it as a rule: every
ribbon item needs a hint entry or an audit fires at boot.
**Win** `auditCommands()` verifies every command has a name, a unique alias, a
hint, and that no hotkey is bound twice.
**Stop** It runs at startup and the console is clean.

### Night 56. Polyline
**Explore** A growing `std::vector<Vec2>` in the tool, Enter to commit,
Backspace to undo one vertex, closing the loop.
**Stop** Backspace at the first vertex does something sane.

### Night 57. Rectangle
**Explore** Constraint as a pure function: `constrainSquare(cursor, origin)`.
Shift for square. Two-point versus centre-out.
**Stop** The constraint is a free function with a test and no knowledge of input.

### Night 58. Circle and arc
**Explore** Centre-radius, two-point, three-point circle (the circumcentre, and
the collinear degenerate case). Arc by three points, arc by centre-start-angle,
arc by start-end-bulge.
**Win** At least two arc construction methods, because tracks are arcs.
**Stop** Three collinear points do not produce a circle of radius 1e18.

### Night 59. Ink, the freehand tool
**Why** This is how you will actually draw tracks. It matters more here than it
ever did in the CAD build.
**Explore** Capturing every mouse move, the Ramer-Douglas-Peucker simplification,
tolerance in screen pixels, why raw mouse points make terrible terrain (a
thousand 2 px spans, each one a potential corner event for the physics).
**Win** A freehand stroke that stores 40 points instead of 900 and looks
identical.
**Stop** You can see the point count in the console and it is sane. Your future
rider thanks you tonight and you will not know it for sixty nights.

### Night 60. Smoothing, and the corner problem
**Why** Foreshadowing the physics. A hand-drawn line has sampling noise, and the
physics must not read noise as geometry.
**Explore** Chaikin subdivision, moving average, curvature-preserving smoothing,
and the alternative of leaving the points alone and making the physics
length-qualify its corner test instead.
**Win** A smoothing pass you can toggle, and an opinion about which layer should
own the problem.
**Stop** Journal entry recording the opinion. You will revisit it on Night 103.

### Night 61. Text
**Explore** UTF-8 in your `std::string`, decoding to codepoints, why your bitmap
font only has ASCII and what you do about that (nothing, for now, but you know
it).
**Win** A text entity that draws, bounds and hit tests.
**Stop** Editing uses a native `EDIT` control or an in-place caret you wrote.
Either is fine, but do not build a text editor tonight.

### Night 62. The tool chest
**Explore** Named presets over the current style, layer and tool settings.
Serialised later with the document.
**Win** Save a preset, apply it, delete it.
**Stop** Something you will genuinely use while authoring tracks. Boost pads are
always the same width.

---

# PART VI. SNAPPING
### Nights 63 to 72

In the CAD build, snapping was a convenience. Here it is structural. Two spans
that are three pixels apart are a hole in the terrain. The rider will find it,
because he is a search algorithm with a skateboard.

Do not teach every tool how to snap. Build one machine.

```cpp
enum class SnapKind { Endpoint, Midpoint, Centre, Quadrant, Intersection,
                      Perpendicular, Tangent, Nearest, Grid };

struct SnapCandidate {
    Vec2      p;
    SnapKind  kind;
    EntityId  a{}, b{};        // b only for intersections
    double    screenDist{};
};

class SnapEngine {
public:
    std::vector<SnapCandidate> find(Vec2 cursorWorld, const Document&,
                                    const Viewport&, const SpatialGrid&) const;
};
```

This subsystem will teach you more STL than any tutorial: `sort`, `find_if`,
`min_element`, `remove_if`, `transform`, lambdas, `optional`, ranges, custom
comparators. Use them here, where the elements mean something, and not on
vectors of integers.

### Night 63. The engine and endpoint snap
**Explore** Radius in screen pixels, the spatial grid query, why you collect
candidates rather than returning the first hit.
**Win** The cursor recognises a line endpoint.
**Stop** The search touches only nearby entities, not all of them.

### Night 64. The glyph
**Why** Invisible snapping is indistinguishable from broken snapping.
**Win** A square for endpoint, a triangle for midpoint, a circle for centre, an
X for intersection. Drawn in screen space at a fixed size.
**Stop** You can see what it decided before you click.

### Night 65. Midpoint, and the recursive midpoint
**Why** The Revit behaviour you specifically wanted: snap a midpoint, and it
then offers the midpoint of the resulting half, repeatedly.
**Explore** State that persists across cursor moves, depth limiting, how to
reset it, why this is a small state machine living inside a stateless-looking
engine.
**Win** Midpoint, then quarter, then eighth.
**Stop** It resets when you move away, and the depth is capped.

### Night 66. Centre, quadrant, nearest
**Stop** Nearest on a flattened arc uses `flatten` from Night 19 and does not
re-derive the point list.

### Night 67. Intersection
**Explore** Segment-segment, segment-arc, arc-arc. The parametric solution, the
determinant, the parallel case, the tolerance question, and why intersections
are the most expensive candidate type (it is a pairwise problem, so use the
grid).
**Win** X marks the crossing of two strokes.
**Stop** Two nearly parallel lines do not produce an intersection candidate at
the edge of the universe.

### Night 68. Perpendicular and tangent
**Explore** Projection onto a segment, the tangent from an external point to a
circle (two solutions, or none), deferred snaps that need the first point of the
current tool as context.
**Win** Draw a line that lands exactly perpendicular to another.
**Stop** The engine can take an optional "from point" for context-dependent
candidates.

### Night 69. Priority and Tab cycling
**Explore** Sorting by kind priority then screen distance, `std::stable_sort`,
Tab stepping through candidates at the current position, and the priority order
that feels right (endpoint beats intersection beats midpoint beats nearest).
**Win** Tab cycles. The glyph updates. The console names the active kind.
**Stop** Priority is data, in one table, not scattered through comparisons.

### Night 70. Constraints as pure functions
**Explore**
```cpp
Vec2 constrainOrtho(Vec2 cursor, Vec2 origin);
Vec2 constrainAngle(Vec2 cursor, Vec2 origin, double incrementRad);
```
Ortho as a special case of angle with a 90 degree increment. Polar tracking.
The interaction between a constraint and a snap, and which one wins.
**Win** Shift constrains. It composes correctly with snapping.
**Stop** Both are pure, both are tested, neither knows what a mouse is.

### Night 71. Snap toggles and the settings block
**Explore** One table as the single source of truth for the toggle set, consumed
by the keyboard handler, the status bar and the settings serialiser. The old
build called it `TOGGLE_DEFS` and having three copies of that list is how they
drift.
**Win** F3 toggles snapping, individual kinds toggle individually, state is
visible.
**Stop** Adding a new snap kind requires touching exactly one table.

### Night 72. Weld: the audit that saves the game
**Why** This is the bridge to Part IX and it belongs here.
**Explore** Walk every entity endpoint, find neighbours within tolerance,
report gaps. This is the same query the terrain builder will do, so write it
once and share it.
**Win** `audit gaps` lists every pair of endpoints closer than 4 px but not
identical, and can optionally weld them.
**Stop** You draw a deliberately sloppy track, run it, and the console tells you
exactly where the rider is going to fall through.

---

# PART VII. UNDO AND MODIFY
### Nights 73 to 86

### Night 73. Feel the problem before you read about the pattern
**Why** Do not begin by reading ten articles on the command pattern.
**Start** Move a line. Now put it back.
**Explore** Nothing. Just try it badly for an hour.
**Win** A single-level undo that works for exactly one operation.
**Stop** You can articulate why it does not generalise.

### Night 74. Snapshots versus commands
**Explore** Two designs.

*Snapshot:* serialise the document before each operation, push it on a stack.
Simple, impossible to get wrong, memory proportional to document size times
history depth.

*Command:* every operation is an object with `execute` and `undo`. Cheap in
memory, and every single operation is now a place where you can make a mistake
that undo will faithfully reproduce.

The old HTML build used snapshots (`snap()`), with a narrower variant
(`snapEnts()`) for in-place edits that cannot add, remove or reorder anything.
That is a good design and the constraint on the narrow variant is the load
bearing part.
**Win** You implemented snapshots, measured the cost with a 20,000 entity
document, and know exactly when it stops being viable.
**Stop** A decision, written down, with the number that justified it.

### Night 75. Transaction boundaries
**Why** The bug that ships in every undo system: half an operation on the stack.
**Explore** Snapshot before mutating, never after. RAII again: a `Transaction`
object that snapshots in its constructor and marks complete in its destructor,
so an early return cannot leave the stack inconsistent.
**Win** `Transaction t{doc, "Move"};` and you cannot get it wrong.
**Stop** An operation that throws or returns early does not corrupt history.

### Night 76. Removal is not `erase`
**Why** The single most valuable scar in the old build's roadmap.
**Explore** When an entity leaves the document, four things reference it: the
selection, the hover, any live grip drag, and the in-progress operation. A raw
`erase` silently corrupts all four. Route every removal through one function.
With stable ids from Night 37 this is much less painful, but it is not free: a
tool holding an id whose entity is gone must be told, and the right answer is
usually to drop the reference and re-prompt, not to guess a replacement.
**Win** `Document::remove(EntityId)` notifies. A tool mid-operation re-prompts
rather than acting on a dead id.
**Stop** An audit asserts that every id held by the selection and the active tool
exists. It runs after every history apply.

### Night 77. Move
**Win** Select, click a base point, click a destination.
**Stop** Undo restores the original coordinates exactly, bit for bit.

### Night 78. Move preview and Ctrl-copy
**Explore** Nothing commits until the final click. Ctrl during the operation
switches to copy. Where does that flag live (the tool, obviously).
**Stop** Ctrl pressed and released mid-drag updates the preview immediately.

### Night 79. Rotate
**Explore** `std::atan2`, radians everywhere internally and degrees only at the
UI boundary, rotation about an arbitrary origin, the order of translate-rotate-
translate.
**Win** Rotate about a picked base point, with preview.
**Stop** Rotating by 360 degrees returns coordinates within 1e-9 of the original.

### Night 80. Scale and mirror
**Explore** Uniform versus non-uniform, reference-length scaling (pick two
points meaning "make this that long"), mirror about an axis, and what a negative
determinant does to text and to arc directions.
**Stop** Mirrored text is not backwards. Mirrored arcs sweep the correct way.
This is where you learn that a transform is not only about points.

### Night 81. Offset
**Why** You will use it constantly. A track needs a parallel surface.
**Explore** Offsetting a polyline, the outer corner problem (a gap opens, fill
it with an arc or a miter), the inner corner problem (self-intersection, and the
loop you must remove), which side the cursor is on.
**Win** Offset a polyline by a distance, both sides.
**Stop** A tight inner corner does not produce a bow tie.

### Night 82. Trim to corner, with Revit semantics
**Why** Your favourite tool, and the roadmap locked its semantics: you click the
part of each element you want to **keep**, and the overhang past the intersection
goes away.
**Explore** Intersection, then decide which side of it each pick was on, then
shorten. Two-element and single-element modes. The case where they do not
intersect and you must extend instead.
**Win** Two crossing lines become a clean corner.
**Stop** The behaviour matches the HTML build under every pick combination.
Test all four. The old build also had `trimsingle` stay loaded instead of
dropping back to Select, so match that too.

### Night 83. Extend
**Stop** Extending to an entity that would only be hit by the projection behind
the cursor does the sane thing.

### Night 84. Grips
**Explore** Grip generation per entity type, hit testing in screen space,
hover feedback, the drag state machine, and why a live grip drag is one of the
four things that break when an entity disappears.
**Win** Drag the endpoint of a line. Drag the radius of a circle.
**Stop** Grip size is constant in screen pixels. Undo restores the pre-drag
geometry.

### Night 85. Grips on everything
**Win** Every entity type has grips: polyline vertices, arc endpoints and
midpoint, text insertion point, ink control points (decimated, or you will have
900 grips).
**Stop** An audit verifies every registered entity type provides grips.

### Night 86. The entity type checklist
**Why** The old build's hardest-won rule. Miss one function and the type renders
all session, then vanishes on reload with no error.
**Explore** Register the type first, implement second. One table listing every
type and every function it must provide: sanitise, points, bounds, hit, paint,
grips, serialise, deserialise, and (new for this project) span extraction.
**Win** `auditEntityTypes()` fails loudly at boot if a type is missing anything.
**Stop** You added a fake type with a missing function and the audit caught it
before you saw a window.

> **Law.** Register first, implement second. The audit is not a nicety, it is
> the thing that turns a silent data-loss bug into a startup assertion.

---

# PART VIII. FILES ARE HOSTILE
### Nights 87 to 94

Native C++ gives a malformed file far more opportunities to hurt you than
JavaScript ever did. This is not ancillary knowledge. This is the part of the
project where you are doing real engineering, and you already do this for a
living.

### Night 87. Streams
**Explore** `std::ofstream`, `std::ifstream`, binary versus text mode on
Windows and the `\r\n` translation that will eat your file, `std::filesystem`,
writing to a temp file and renaming so a crash mid-save does not destroy the
previous version.
**Win** Write five lines. Read them back.
**Stop** A save interrupted halfway leaves the old file intact.

### Night 88. The format
**Explore** Text or binary. Text wins here: diffable, greppable, recoverable by
hand, and the parser is the lesson. Version it in the first line. Do not write a
generic industrial JSON implementation, write exactly the parser your format
needs and grow it when the format grows.
**Win** `.b2db` v1 holding entities, layers, styles and the view.
**Stop** You can open a saved track in Notepad and understand it.

### Night 89. The parser
**Explore** Tokenising, `std::string_view` and the lifetime trap that comes with
it, `std::from_chars` (fast, no locale, no exceptions) versus `std::stod`
(locale-dependent, which means a machine with a comma decimal separator writes
files your machine cannot read).
**Win** Parse a number, a string, an object, an array.
**Stop** You tested with a comma-decimal locale.

### Night 90. Defensive parsing
**Why** Treat every file as though someone wrote it to hurt you, because
eventually someone will.
**Explore** Bounds checking, integer overflow on a length prefix, allocation
limits (a file claiming 4 billion entities must not attempt the allocation),
`std::isfinite` on every double you read, nesting depth limits, total size
limits, and never trusting a count field.
**Win** A parser that rejects rather than crashes.
**Stop** A file claiming two billion points fails in microseconds with a clear
message.

### Night 91. Validate before you apply
**Explore** Parse into a candidate document, validate it fully, and only then
swap it in. A half-loaded document is worse than a failed load. Clear all
transient state on load: selection, hover, active tool, grip drag, history.
**Win** A corrupt file leaves your current drawing untouched.
**Stop** Loading while a tool is mid-operation does not leave a dangling id.

### Night 92. Fuzz your own parser
**Why** This is the night that separates you from most C++ programmers.
**Explore** Write a loop that loads a valid file, flips random bits, and feeds it
back. Ten thousand iterations. Then run it under Address Sanitizer:
`cl /fsanitize=address`.
**Win** A crash you found yourself, before a user did.
**Stop** Ten thousand mutations, zero crashes, zero ASan reports.

### Night 93. Autosave and recovery
**Explore** A rolling autosave on a timer, crash detection via a lock file,
recovery prompt on next launch. The HTML build did this with IndexedDB and a
localStorage fallback. Yours is a directory.
**Win** Kill the process with Task Manager. Reopen. Get your work back.
**Stop** Autosave never blocks the frame. If it takes 40 ms, it goes on a
thread, and now you have learned `std::jthread` for a real reason.

### Night 94. Import from the old build
**Why** You have a body of work in `.b2db` from the HTML version, and this is a
compatibility target.
**Explore** Reading the old JSON-ish format, mapping its entity types onto
yours, deciding what to do with the types you cut (markup status, flow nodes,
references): drop them, and say so in the console rather than silently.
**Win** A drawing made in the HTML build opens in the native one.
**Stop** The import reports what it discarded. It never lies about what it kept.

---

# PART IX. THE TRACK BECOMES TERRAIN
### Nights 95 to 112

Everything so far was the level editor. This is the game.

The HTML build already solved most of these problems and you should treat that
code as the behavioural specification, exactly as the rest of the course does.
But do not transliterate it. Ask what responsibility each piece has, then build
that responsibility natively. C++ will hand you better answers than JavaScript
could: a span becomes a value type in a contiguous array instead of an object in
a hash map, and the difference is not stylistic, it is 240 physics ticks a
second across 14,000 spans.

**The architecture:**

```
Document (entities)
    |  flatten()
    v
SpanBuilder  ->  Terrain { std::vector<Span>, SpatialGrid, joins }
                     ^
                     |  read-only, rebuilt when the document changes
                     |
                  Rider (state machine: Ride | Air | Crashed)
                     |
                     v
                 RunState (score, combo, checkpoints, particles)
```

The terrain is derived data. It is never edited. When an entity changes, the
affected spans are rebuilt. That one rule keeps the editor and the game from
growing into each other.

### Night 95. Spans
**Why** A span is the atom of terrain: a directed straight piece of surface.
**Explore**
```cpp
struct Span {
    Vec2      a, b;            // in the order the stroke was drawn
    Vec2      dir;             // normalised, a -> b
    double    length{};
    double    angle{};
    EntityId  owner{};
    Material  material{};      // from the layer, Night 46
    std::int32_t linkF{-1};    // span index reached by leaving via b
    std::int32_t linkB{-1};    // span index reached by leaving via a
};
```
Build them from `flatten()` (Night 19) so curves are already point lists.
**Win** A drawing becomes a `std::vector<Span>`, drawn as a debug overlay with
an arrow on each showing its direction.
**Stop** You can see the direction of every span on screen. You will need this
overlay all through Part IX.

> **Law.** Never normalise a span left to right. The old build did, and it
> deleted the single fact that makes a loop possible: the top half of a loop is
> travelled right to left. Direction is data, not noise.

### Night 96. Joins
**Why** O(1) traversal instead of a scan, which matters enormously once nothing
is filtered out by length.
**Explore** Precompute, for each span end, which other span end is within
tolerance. Only weld where spans genuinely meet: one entity's flattened output
can contain disjoint pieces, and welding those invents terrain that the author
never drew.
**Win** `linkF` and `linkB` filled in, verified by an audit that walks a closed
loop and returns to its start.
**Stop** A track drawn as one continuous stroke traverses end to end with no
scanning.

### Night 97. Accept every slope
**Why** The second reason the old builder could not represent a loop.
**Explore** The temptation to filter out near-vertical and overhanging spans as
"not floor". Resist it. Vertical spans are the sides of a loop and the walls of
a halfpipe. Overhanging spans are the top of the loop.
**Win** A hand-drawn loop survives the builder intact, all 360 degrees of it.
**Stop** Your debug overlay shows a complete closed ring of spans.

### Night 98. The terrain broadphase
**Explore** Reuse the spatial grid from Night 48, or build a dedicated one keyed
by span index. Cell size against typical span length. A query returns candidate
span indices for a segment or a point.
**Win** "Which spans are near this swept segment" in constant time.
**Stop** 14,000 spans, 240 ticks a second, and the physics is not the frame
budget. Measure it. Put the number in your journal.

### Night 99. The rider exists
**Explore** A struct holding position, the span he is on, arc position along it,
speed as a signed scalar, and `side` (which face of the stroke he is on, +1 or
-1). A state enum: `Ride`, `Air`, `Crashed`.
**Win** A stick figure sitting motionless on a line you drew.
**Stop** He is at the right place, on the correct side, and the debug overlay
prints his span index and side.

### Night 100. Gravity along a surface
**Why** The first moment it feels like a game.
**Explore** The rider on a span is a one-dimensional problem. Project gravity
onto the tangent:

```
a = g . t          where t is the unit travel direction
v += a * dt
s += v * dt
```

The normal component is absorbed by the surface. No normal force calculation, no
constraint solver, no penetration. This is why parametrising along the span is
worth the bookkeeping.
**Win** He slides down a hill and slows going up one.
**Stop** He oscillates in a valley forever with no friction, losing no energy.
That is the correctness check: if he gains energy your integration is wrong, and
if he loses it you have accidental damping.

### Night 101. Friction, drag and pump
**Explore** Coulomb friction proportional to the normal component, quadratic air
drag, and pumping: a speed gain proportional to turn rate through a transition.
Pumping is what makes a halfpipe work and what lets a rider build up to a loop he
could not clear on the first pass.
**Win** He settles at the bottom of the valley. Held input through a transition
gains speed.
**Stop** Pumping cannot produce infinite speed. There is a ceiling and you chose
it deliberately.

### Night 102. The transfer across a joint
**Why** The hardest bug in the original, and worth understanding completely
before you write a line of it.
**Explore** When he runs off the end of a span, move him onto the joined one,
preserving both his world travel direction and which face he is on.

The trap: resolving the entry direction from the sign of a dot product. At a
right-angled corner that dot is exactly zero, the naive resolution is `+1`, and a
rider arriving at the far end of a span gets sent straight back out of the end he
came in by. He then ping-pongs across the joint inside the hop loop about 120
times a second and sticks to the corner permanently.

**The fix:** anchor the entry direction to the **joint**, not to a dot product.
Ask which end of the next span the joint is nearest, and enter from that end.
```cpp
const bool atA  = dist(jointP, next.a) <= dist(jointP, next.b);
const double e  = atA ? 1.0 : -1.0;
const Vec2 into = next.dir * e;
```
**Win** He crosses a corner once, in the right direction.
**Stop** A square drawn as four lines can be circumnavigated indefinitely
without a single ping-pong. Put a counter on joint transfers and watch it
increment sanely.

### Night 103. The corner test, length-qualified
**Why** The Night 60 question comes due.
**Explore** A large angle between two spans is a corner. A large angle between
two *short* spans is sampling noise on a hand-drawn line. Treating noise as
geometry is what throws a rider off a line that looks perfectly straight.
```cpp
const bool sharp = turn > 0.35 && std::min(q.length, next.length) >= u(30);
```
The same length qualifier the builder uses for its kink test. Both places, one
constant.
**Win** A freehand line is rideable at speed.
**Stop** A genuine 90 degree corner still behaves like a corner.

### Night 104. Leaving the ground
**Explore** Every exit to the air state goes through one function. A launch off a
lip, a launch off a sharp corner, a deliberate jump, and a boost pad all share
it, so the trick setup, the no-land window and the goal marker can never be
forgotten by one of them.
```cpp
void goAir(Vec2 travel, double speed, double pop);
```
The no-land window (about 70 ms) stops him instantly re-colliding with the lip
he just left.
**Win** He launches off a ramp.
**Stop** There is exactly one function that sets `state = Air`. Grep and prove
it.

### Night 105. Ballistic flight
**Explore** Integrate velocity, integrate position, track air time and launch
height. Rotation in the air. The board and the rider as one object for now.
**Win** A clean parabola.
**Stop** Air time from a known launch matches the analytic answer to three
decimal places. This is a test, not a vibe.

### Night 106. The swept landing
**Why** A rider at 900 px/s covers 3.75 px per physics tick and 15 px per frame.
A 1 px line is not something you test for by asking "am I inside it".
**Explore** Two passes.

1. **Exact crossing.** Segment-segment intersection of the swept path against
   each candidate span.
2. **Closest approach.** March the sweep in a few steps, take the nearest point
   on the span, land if it is within tolerance.

The second pass is what makes vert skating work, and it is not an optimisation,
it is a correctness requirement. A rider popping straight up off vertical coping
has zero dx. His path is **collinear** with the wall. There is no crossing to
find, so a crossing-only test can never re-attach him to the wall he just left,
and every air out of a halfpipe becomes a one-way trip. Distance handles it with
no special case: while he is above the coping the nearest point on the wall is
its top end and he is far from it, and as he drops back in that distance closes
to nothing and he lands.
**Win** He lands on ramps, on flats, and back into a halfpipe.
**Stop** At 2,000 px/s he cannot tunnel through anything. Test with a
deliberately absurd speed.

### Night 107. The landing judgement
**Explore** Compare his air rotation against the surface tangent at the landing
point. Small difference: stomped, keep the speed, multiply the combo. Medium:
sketchy, lose speed and the combo. Large: bail. Also decide which side of the
span he arrives on, because that determines `side` for the ride state that
follows.
**Win** Three distinguishable outcomes with different sounds and different
console lines.
**Stop** Landing backwards on a transition is possible and is scored as such.

### Night 108. The bail
**Why** The best twenty minutes in the old build, and the thing that makes
failure fun instead of punishing.
**Explore** On a bail, the rider and the board **separate into two independent
projectiles**, each with its own velocity, spin and rest state. They tumble,
they bounce, they come to rest. It is not a death, it is a ragdoll with two
bodies and no skeleton, which is the cheapest ragdoll that has ever looked good.
**Win** He comes apart, the board slides off, both settle.
**Stop** Bailing in the same place twice ends the run instead of looping
forever. The old build tracked the last bail position and landing count for
exactly this.

### Night 109. Checkpoints and respawn
**Explore** Record the last good grounded state: span, side, position, direction.
Respawn restores all of it and resets the transient loop and trick accumulators.
**Win** Bail, respawn, keep riding.
**Stop** Respawning inside a loop does not leave `inLoop` set. Every transient
field is reset in one place.

### Night 110. Loops
**Why** The signature trick, and a real physics lesson.
**Explore** Track accumulated turn and signed turn to know he is in a loop.
Estimate the local radius of curvature from the change in tangent angle over
arc length. The classical condition to complete a circular loop is

```
v_bottom >= sqrt(5 * g * r)
```

which for any hand-drawn loop bigger than about 60 units is far above anything a
rider will realistically carry. Every drawn loop is therefore unmakeable if you
implement the physics honestly.

So do not implement it honestly. **Commit.** Once he is clearly carving into a
loop, judge him on the speed he carried *into* the carve, not his speed right
now, because by the time the turn accumulator trips he has already traded speed
for height. If he arrived with a credible fraction of what the loop demands
(the old build used 38 %), grant him what it demands and a little more. If he
arrived far too slow, refuse, let him roll back out, and let that failure mean
something.

Re-evaluate when the radius tightens sharply, or a gentle run-in curve consumes
the commit before he reaches the actual loop.
**Win** A hand-drawn loop-de-loop, cleared, with the exit speed reported.
**Stop** A loop entered far too slowly is refused and he rolls back down. Both
outcomes are reachable and both feel correct.

> This is the night you learn the most important thing about game physics: the
> goal is not simulation, it is a system that behaves the way the player
> believes it should. Every deviation from the real equations is a deliberate,
> documented decision. Write down each one. An undocumented fudge is a bug
> waiting to be "fixed" by a future you who no longer remembers.

### Night 111. Tricks, combo and flow
**Explore** A trick begins on launch and resolves on landing. Rotation counted in
half-turns, board spin, grabs as a held input. A flow multiplier that builds
across clean landings and resets on a bail. Speed tiers with names.
**Win** A scoring line in the console on every landing.
**Stop** The scoring rules live in one table, not scattered through the physics.

### Night 112. The goal marker
**Explore** Look ahead along the terrain graph from the current span and find
the next landing surface. Draw it. This is the difference between a physics demo
and a game: the player can see where he is supposed to go.
**Win** A marker showing the intended landing.
**Stop** It updates on every launch and never points at the span he just left.

---

# PART X. THE ENGINE EARNS ITS NAME
### Nights 113 to 120

### Night 113. Determinism
**Why** Replays, ghosts, regression tests and honest bug reports all depend on
it.
**Explore** Fixed timestep (you have had it since Night 9). A seeded xorshift
PRNG for the visual randomness, never consulted by the simulation. No
dependence on frame rate, wall clock, or the order of an `unordered_map`
iteration. Floating point: `/fp:precise`, avoid `/fp:fast`, and understand that
you are not promising cross-platform determinism, only same-binary determinism.
**Win** The same input sequence produces the identical final position twice.
**Stop** A test replays 10,000 ticks and asserts the final state matches a
recorded hash.

### Night 114. Replays and ghosts
**Explore** Record the input stream, not the state. Play it back through the
same simulation. Then render a second, translucent rider from a previous
recording.
**Win** Race your own ghost.
**Stop** A replay recorded before a physics change diverges, and you can detect
and report that rather than showing nonsense.

### Night 115. The camera
**Explore** Follow with lookahead proportional to velocity, a deadzone so small
movements do not swim, framerate-independent smoothing (Night 34), zoom out
with speed, and a hard clamp so he never leaves the screen. Screen shake as a
decaying offset applied at render, never to the simulation.
**Win** A chase camera that does not make you seasick.
**Stop** At 2,000 px/s he is never off screen and the camera never overshoots
into empty space.

### Night 116. Sound
**Why** Half of game feel, and a genuinely excellent C++ exercise.
**Explore** Start with `waveOutOpen` and a small ring of buffers: it is thirty
lines and it works. Synthesise everything: a square wave for the pop, filtered
noise for the grind, a downward sweep for the bail. An ADSR envelope. Mix into a
float buffer, clamp, convert to 16 bit. Then, if you want the real lesson,
`IAudioClient` and WASAPI shared mode, and learn what a lock-free ring buffer is
for, because your audio thread must never take a mutex the render thread holds.
**Win** Sounds you wrote, from maths, with no asset files.
**Stop** No clicks, no dropouts, no allocation on the audio thread.

### Night 117. Particles
**Explore** A fixed-capacity pool, no allocation at runtime, structure of arrays
versus array of structures, integrate and cull, additive blending from Night 20.
**Win** Sparks on a grind, dust on a landing, a burst on a bail.
**Stop** 10,000 particles cost less than a millisecond. You measured.

### Night 118. Mode switching
**Explore** Edit mode and play mode as one application with different input
routing and different render paths. Not two programs. The document is live in
both, so you can pause, fix a corner, and resume.
**Win** Tab to play, Tab to edit, and the terrain rebuilds only for entities you
changed.
**Stop** Editing a span the rider is standing on does something defined. You
chose what.

### Night 119. Track validation
**Why** Close the design loop.
**Explore** Reuse the gap audit from Night 72, plus: unreachable sections, gaps
too wide for any achievable speed, loops whose radius no run-up can satisfy,
kill-zone-only routes.
**Win** `audit track` tells you what is wrong with your level before you ride it.
**Stop** You deliberately build an impossible track and it is diagnosed
correctly.

### Night 120. Review night
**No feature requirement.** Sit down and read your own program.

Answer all of these out loud. If you cannot answer one, that is your next
excellent session.

- Where is the document stored, and who owns entities?
- How does a mouse click reach `LineTool`?
- Who performs snapping, and who decides which candidate wins?
- Who converts world coordinates, and how many places know about the viewport?
- How does Undo know what happened?
- Who owns the backbuffer, and what releases it?
- How does saving reach the filesystem, and what happens if it fails halfway?
- How does an entity become a span?
- Who decides the rider is in the air?
- Why does the loop commit exist, and what does it break if I remove it?
- Where does the frame time go? Name the top three costs, from a measurement.

Then refactor only what you can justify. That review night matters more than any
three feature nights.

---

# APPENDICES

## A. Verification, carried over

Run after every change. The old build's working agreement holds here verbatim.

- **`test`** runs the pure suite: geometry, rasteriser hashes, constraints,
  parsing, span building, ballistic solutions. Pure functions only. Any global
  state a test touches is restored in a scope guard, not by hand.
- **`audit`** runs the impure checks: entity type completeness, command table
  integrity, spatial grid agreement with brute force, selection ids all live,
  terrain joins consistent, hint coverage.
- Both run at startup. **The console is clean at boot or you do not have a
  build.**
- One coherent chunk per session, verified, then the next. **Never bundle a
  refactor with a feature.** When a refactor and a feature are in the same diff
  and something breaks, you cannot tell which one did it, and you will spend the
  night bisecting your own evening.
- Step by step on technical work. Stop at the first failure. Do not stack three
  unverified changes and hope.

## B. The maths, in one place

| Need | Method |
|---|---|
| Point to segment distance | Project onto the segment, clamp the parameter to [0,1], measure |
| Segment to segment intersection | Cross-product denominator, both parameters in [0,1], guard the near-zero denominator |
| Line to circle | Substitute and solve the quadratic; the discriminant sign is your case analysis |
| Circle from three points | Circumcentre by perpendicular bisectors; collinear means an infinite radius, handle it |
| Polygon area | Shoelace. Sign tells you the winding |
| Polyline simplification | Ramer-Douglas-Peucker, tolerance in screen pixels |
| Curve flattening | Adaptive subdivision to a sag tolerance in screen pixels |
| Rotation about a point | Translate to origin, rotate, translate back |
| Ballistic flight | `p += v*dt; v.y += g*dt`. Symplectic Euler, not RK4. It conserves energy in a way that looks right and it is one line |
| Gravity along a surface | `a = dot(g, tangent)`. The normal component is absorbed |
| Loop completion | `v >= sqrt(5*g*r)` at the bottom, then decide how much you are prepared to cheat |
| Radius of curvature | `r = ds / dtheta` over a short arc |
| Framerate-independent smoothing | `alpha = 1 - exp(-rate * dt)` |
| Antialiased line | Xiaolin Wu, coverage as alpha |
| Polygon fill | Active edge table, half-open scanline intervals |

## C. Comments are for why

Not this:
```cpp
++i;   // increment i
```

This:
```cpp
// Snap radius is in screen pixels so snapping feels the same at every zoom.
// Converting here rather than at each call site is what stops the radius from
// silently becoming a world-space distance in one of the eleven callers.
```

The HTML build is full of comments explaining the reasoning behind decisions,
including several that read like a message from a man who had just spent four
hours on one bug. Those comments are the most valuable text in the file. Keep
that culture. When you fix something subtle, write down what the wrong version
did, because the wrong version is what a future you will independently invent.

## D. Do not hide ignorance behind frameworks

If you do not understand window resizing, you do not solve it by introducing a
GUI toolkit. If you do not understand ownership, you do not solve it with a
service container. If you do not understand serialisation, you do not bury it
under a library.

But there is a difference between learning and pointless suffering. Use Win32
for windows, the standard library for containers and algorithms, `std::filesystem`
for paths, and the OS file dialogs. Do not write your own allocator, your own
`std::sort`, or your own PNG encoder unless one of those has genuinely become
the thing you want to learn that night.

Platform capability: take it. Third-party abstraction that hides the platform
from you: refuse it.

## E. The toys become the curriculum

The Experimental Features stay. Ported after the machine works, and each one is
now a real exercise with a gameplay payoff.

| Toy | The lesson | What it becomes |
|---|---|---|
| Gravity | Vector fields, integration | A gravity well hazard |
| Spring | Simulation, stiffness, stability | Bungee sections and trampolines |
| Black hole | Spatial transforms, singularities | A track hazard that bends the path |
| Flood fill | Graph search, queues, visited sets | Closed-area detection for the editor |
| Sonar | Distance geometry, propagation | Track proximity visualisation |
| Current | Time, animation, phase | Conveyor and wind zones |
| Ants | Autonomous agents, steering | Opponents. Rivals on your track |
| Termites | Emergent behaviour, state machines | Terrain that degrades as you ride it |

The joke features become some of the best learning in the project, and unlike in
the CAD build they now have somewhere to live.

## F. The real goal

Success is not `b2db.exe` existing.

Success is opening the source in two years and thinking: I understand how the
window exists. I understand how a pixel gets there. I understand where these
entities live and who owns them. I understand how a click becomes a tool state
transition. I understand how a stroke becomes terrain. I understand why he
leaves the ground and how he knows he has landed. I understand why the loop
commit is a lie and exactly which lie it is.

I understand enough of this machine to change it without being afraid of it.

That is the reason to build it this way.

---

## WHEN YOU SIT DOWN TONIGHT

Do not ask what chapter of C++ you should learn.

Ask what you want it to do tonight, then learn whatever C++ is standing between
you and that result.

Compile often. Run constantly. Break things on purpose. Inspect what happened.
Rewrite code you no longer believe in. Keep the machine small enough to hold in
your head.

Finish the night with one more part of the system belonging to you.

Then come back tomorrow.
