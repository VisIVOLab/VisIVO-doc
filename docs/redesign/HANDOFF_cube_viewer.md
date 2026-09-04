# Cube viewer — complete implementation handoff

**This supersedes `STAGE3_REVIEW.md` and `HANDOFF_stage4_layout.md`.** It is the single
ordered list of everything still open for the cube viewer, including the parts of the
earlier review that were never done. Where anything here disagrees with an earlier
instruction of mine, this file wins.

Normative reference: **`CUBE_VIEWER_SPEC.md`** — read it fully before writing code.
Visual reference, in `VisIVO Workspace.dc.html`:

| Screen | What it shows |
|---|---|
| `#12a` | **the default state** — 1 pane, Inspector collapsed to a rail |
| `#12b` | focus mode, both rails collapsed |
| `#12c` | space budget per state |
| `#12d` | two panes, 552 × 735 each |
| `#12e` | four panes, 552 × 363 each |
| `#12f` | the switching rules |
| `#11a` | every region expanded, Properties tab — a labelling reference, **not** the default |
| `#11b` | Analysis and Provenance tabs |
| `#3b` | the inline task forms |

---

## What already exists (do not rebuild these)

Confirmed present in the tree, so the work below is mostly re-hosting and layout:

| Need | Already there |
|---|---|
| 3D volume view | `QVTKOpenGLNativeWidget` in `vtkWindowCube` |
| 2D slice view | the slice renderer |
| Moment map view | `setupMomentRenderer()` + `momentWin` |
| **Spectrum plot** | `probePlotWidget` / `profileWidget` — `setupSpectrumPlot()`, `updateSpectrumPlot()`, `setSpectrumCurrentChannel()`, `setSpectrumFrozen()`, `setSpectrumContext()`, QCustomPlot-based, currently in a **floating window** |
| Product registry | `ProductRegistry` (stage 2) |
| Command bar / status rail | `WorkspaceChrome` (stage 1) |
| Session tree | `SessionDataTree` (stage 2) |
| Inspector shell | `InspectorPanel` (stage 3) |
| Theme tokens | `theme/VisivoTheme.h` — **use the `k*()` accessors, never literal hex**, except `#000004` for viewport backgrounds, which is a documented constant |
| Tool dialog chrome | `theme/ToolDialogStyle.h` — reuse `inputCss()`, `sectionCss()`, `helperCss()`, `addSection()` for the inline forms |

So the four views in the 4-pane layout are all existing widgets. **Nothing in step 3
requires a new renderer or a new plot** — only re-parenting.

---

## Step 0 — Clean up before anything else

### 0.1 Kill the orphaned `SidebarPanel`

A ghost row still paints over the left dock: the text **"Inferno"** appears at roughly
y ≈ 1000 in an otherwise empty Session Data panel, with faint horizontal rules at
y ≈ 305 / 420 / 700 / 1105.

Diagnosis: `SidebarPanel::addPage()` wraps each page in a `QScrollArea` inside a
`QStackedWidget`, so re-parenting *the pages* does not touch `m_sidebar` itself. If it was
only removed from the layout with `removeWidget()`, it stays an **orphan child of the
`QMainWindow`** and paints at (0, 0) — rail plus stack, and the stack's current page is a
rendering page carrying a COLOR MAP combo showing "Inferno".

Confirm in one line before fixing:

```cpp
m_sidebar->setStyleSheet(u"background: magenta"_s);
```

If a magenta rectangle appears over the left dock, it is the culprit. Fix properly:
re-parent the four pages into their real containers **first**, then

```cpp
m_sidebar->deleteLater();
m_sidebar = nullptr;
```

in both the cube and the image viewer. Check nothing dereferences
`m_uiAssembler->...Button()` afterwards, and that `actionShowControlsSidebar` no longer
points at it (it should toggle the Inspector dock).

Generic guard, worth keeping as a debug helper — anything it prints that is not the menu
bar or status bar is painting where it should not:

```cpp
for (QWidget *w : findChildren<QWidget*>(Qt::FindDirectChildrenOnly))
    if (w != centralWidget() && !qobject_cast<QDockWidget*>(w)
        && !qobject_cast<QToolBar*>(w) && w->isVisible())
        qDebug() << "orphan:" << w->objectName() << w->geometry();
```

### 0.2 Move the rendering controls OUT of Properties and into the left dock

**This corrects an instruction I got wrong.** I originally said to migrate the four old
sidebar pages into the Inspector's Properties tab. That turned Properties into a scrolling
wall of bordered cards, which is most of why the build still feels confused.

Move them to the **left dock**, flattened (see `#12a` and the "Left dock detail" section
of the spec):

| Old sidebar content | New home |
|---|---|
| RENDERING MODE, VOLUME RENDERING, RENDERING THRESHOLD, COLOR MAP | left dock → **DISPLAY — 3D** |
| volume / plane / markers / axes visibility | left dock → **LAYERS** |
| 3D INTERACTION | left dock → **INTERACTION** (collapsed by default) |
| slice animation controls | left dock → **SLICE ANIMATION** (collapsed by default) |
| Tools page | already superseded by the Analysis tab |
| Info / stats | Inspector → Properties (STATISTICS + WCS, as KV rows) |

Flattening rules: **no card borders, no per-card padding, no full-width "Advanced…"
buttons.** Section header = Lato 10 px / 700, letter-spacing 0.14em, `kOnSurfaceVariant()`,
with a ▼/▶ disclosure; sections separated by a 1 px `kOutlineVariant()` rule; row padding
11 px 14 px; field rows = 11 px label left (stretch), control right.

After this, **Properties must fit without a scrollbar** at a 900 px window:
identity ≈ 60 + DATASET ≈ 110 + STATISTICS ≈ 240 + WCS ≈ 120. If it scrolls, something
that belongs on the left is still in it.

### 0.3 The placement rule, for every future doubt

| Question the control answers | Home |
|---|---|
| What am I looking at? | left dock |
| Compute something new | right dock (Inspector) |
| The data, and how panes are arranged | centre (toolbar + pane headers) |
| Is the system healthy? | status rail |

---

## Step 1 — Pane infrastructure (was "Fix 3", never done)

Everything in steps 3 and 5 depends on this. Build it first and rebuild before continuing.

### 1.1 Centre toolbar, h 30, above the splitter

Not inside the Inspector. Left to right:

1. the **1 / 2 / 4** segmented control (`▣ ▥ ⊞`);
2. the **3D / 2D** segmented control — visible only while `paneCount == 1`;
3. the **`Linked`** chip: icon + "Linked" + a mono 10 px qualifier stating exactly what is
   synced. Replace the `⛓` glyph — it has no coverage in Lato or JetBrains Mono and falls
   back to two vertical bars — with `resources/icons/rail_viz.svg` via
   `VisivoTheme::renderThemeIcon()`;
4. stretch;
5. the live cursor readout, JetBrains Mono 10 px: `l 104.87  b +68.52  ·  2.14e-02 Jy/beam`.

### 1.2 One header per pane, h 34

When the views moved into the splitter they lost their dock titles. Every pane gets:

- an **active dot**, 5 px: filled `kPrimary()` when active, else a 1 px `kSecondary()` ring;
- the **title**, Lato 11 px / 700 — a **dropdown** when `paneCount > 1`;
- a **qualifier**, mono 10 px `kOnSurfaceVariant()`: "volume · composite", "28 / 64",
  "ch 1–64 · 3σ", "cursor · 1 px";
- stretch;
- pane-specific controls — the **Auto / Local / Remote** segmented control on the 3D pane
  (this is where `RENDER: LOCAL` goes, off the status bar), **Slice / Moment** on the 2D
  pane;
- the active **LUT strip**, 30 × 9 (24 × 8 in the 4-pane compact header);
- **⤢** maximise.

Active pane: 1 px `kPrimary()` border on the pane frame, 1 px `kOutline()` on the others.
Clicking anywhere in a pane makes it active. Exactly one active at a time. The active pane
is what the channel scrubber drives and what "Send slice to image viewer" takes.

### 1.3 Channel scrubber as an overlay

Inset 10 px from the left, right and bottom edges of the 3D and 2D viewports:
`rgba(0,0,0,0.55)`, 1 px `kOutline()`, radius 6, h 28. Contents: "CH", the channel number,
the slider, and the velocity in mono 11 px. It floats over the data, it does not steal a
row from it.

---

## Step 2 — 2D pane geometry (was item #3, never done)

Today the WCS labels and the colorbar are drawn **inside** the data area: latitude ticks
sit over the pixels, "Galactic Longitude" runs across the bottom of the image, and the
colorbar covers a column of data.

The viewport is a **centring container** (`#000004`, padding 8, `box-sizing: border-box`).
Inside it, **one group** holds everything and is shrink-wrapped to the image height:

- row 1: latitude gutter **w 44** (right-aligned mono 9) · **data area** · colorbar
  **w 12** · colorbar value gutter **w 34** (left-aligned mono 9);
- row 2: a 44 px spacer, the longitude tick row, the axis label ("Galactic Longitude",
  Lato 10 px `kPrimary()`), then a 46 px spacer.

Two rules that are easy to get wrong:

1. **The data area keeps the image's pixel aspect ratio and never stretches.** Derive the
   side from the **shorter of the two available axes** — do not assume it is the width. In a
   552 × 363 pane the binding axis is *height*: the square lands at about **308 px**, not
   the ~478 px the width would allow. In the 1-pane layout the binding axis is the width.
   Use `NAXIS1 : NAXIS2`, not 1 : 1, for a non-square image. In Qt: a fixed-ratio container
   or `heightForWidth`, not a stretched grid cell.
2. **The letterbox applies to the whole group, not just the image.** Put the render widget,
   both gutters and the colorbar in one fixed-ratio container and centre *that* in the pane.
   Otherwise a declination tick annotates empty space and the colorbar maps pixels that are
   not there.

---

## Step 3 — Layout states and switching

### 3.1 Default: one pane

`paneCount` defaults to **1**; the 3D / 2D segmented control picks which view fills it. The
other widgets are hidden, never destroyed (`QStackedWidget` or `setVisible(false)`).
`Linked` is disabled and dimmed to 0.4 opacity while `paneCount == 1`.

At 1440 × 900 with the default chrome this gives a **1112 × 735** viewport, against
~454 × 735 for each of today's two panes. Note the height is identical in every state —
vertical chrome does not change — so **this buys width, not height**. Do not go hunting for
vertical space that was never there.

### 3.2 Two invariants

- **Nothing is created or destroyed by switching layout.** ▣ → ⊞ → ▣ must lose no camera,
  no zoom, no loaded slice, and must not issue a backend request.
- **The active pane is the unit of everything** — which pane survives a downgrade, which one
  the scrubber drives, which one exports.

### 3.3 Fill order

A newly appearing pane takes the first of **2D Slice → Moment 0 → Spectrum** not already on
screen. Products in the `ProductRegistry` extend that list after Moment 0.

The Spectrum entry re-hosts the existing `probePlotWidget` inside a pane instead of a
floating window: same `setupSpectrumPlot()` / `updateSpectrumPlot()` calls, and
`setSpectrumCurrentChannel()` already keeps its marker in sync with the channel — so a
spectrum pane follows the scrubber for free.

### 3.4 The switching table

| Switch | Filled with | Side effects |
|---|---|---|
| ▣ → ▥ | left keeps the current view; right takes the first unused entry | 3D/2D picker **hides**, per-pane title dropdowns appear; `Linked` **enables**, on by default, qualifier "camera · channel · LUT" |
| ▥ → ⊞ | existing two keep top-left / top-right; the new two continue the list | `Linked` qualifier narrows to **"channel · LUT"** — camera sync between a volume and three 2-D products is meaningless, so state only what is real; headers go compact (padding 6/9, LUT strip 24 × 8) |
| ⊞ → ▣ | the **active** pane survives, not necessarily the first | picker returns, set to the survivor; `Linked` disables and dims but remembers its state |
| a slot has nothing to show | an **empty pane**: dark background, a centred "Pick a view" dropdown | never a bare black rectangle; **never silently fall back to fewer panes** — the user asked for 4 |
| window < 1600 px | ▥ splits **top/bottom** (1112 × 363 beats 552 × 735 for a channel map); ⊞ stays 2 × 2 | direction derived from window width on every resize, not persisted |

With `paneCount > 1` each header title is a dropdown; picking a view already shown
elsewhere **swaps** the two panes rather than duplicating it.

---

## Step 4 — Collapsible rails, focus mode, persistence

### 4.1 Rails

Both docks collapse to a **36 px rail — never 0**. Rail contents, top to bottom: the expand
glyph ("◧" left, "◨" right) in `kPrimary()`; a 20 × 1 px `kOutlineVariant()` rule; the panel
name rotated (in Qt: a custom `paintEvent` with `painter.rotate(90)`), Lato 10 px / 700,
0.14em, `kOnSurfaceVariant()`; stretch; **right rail only**, the three tab names rotated the
same way with the active one in `kPrimary()` / 600 — clicking one expands the dock *and*
selects that tab; and at the bottom a mono 9 px **count of what is behind the rail** —
products on the left, running tasks on the right, so a collapsed panel can still tell you
something happened.

Defaults: **left open at 272, right collapsed at 36.** Threshold, blend mode and LUT are
touched every few seconds, so hiding them costs more than the 272 px; the Inspector is only
needed when starting a task. Give the left dock a "◧" button in its SESSION DATA header.

The Inspector auto-expands when a chip is pressed, a task starts, or a vertical tab name is
clicked. It never auto-collapses.

### 4.2 Focus mode

`⌥⇧F` (`Ctrl+Shift+F` elsewhere) collapses both rails and thins the status rail to health +
queue. A toggle, not a mode. Add it to the View menu as "Focus mode".

### 4.3 Persistence

Store `paneCount`, the per-pane view assignment, `leftCollapsed`, `rightCollapsed` and the
active pane per viewer type in `QSettings`.

**Use a new key prefix** — `layout_v2/cube/...`, `layout_v2/image/...`. If anything is still
restored from an older key, existing installs come back with the stage-3 arrangement and
this work will look like it never landed.

---

## Step 5 — Status rail cleanup (was "Fix 6", never done)

Depends on step 1, because two of these labels move into a pane header that does not exist
yet. Currently the rail reads "Full resolution · Remote · Full resolution · Sanity: OK ·
RENDER: LOCAL" and then the four correct slots — load state three times, and a render mode
that is not system health.

**Exactly four slots**, each a 6 px dot, separated by `│` in `kSecondary()`, all JetBrains
Mono 10 px:

| Slot | Dot | Content |
|---|---|---|
| Health | `kSuccess()` | `backend ok · 1 ms` |
| Queue | `kRunning()` when > 0 | `0 running · 0 queued` |
| Cache | `kCache()` | `cache 2/32` |
| Session | — | `session anonymous` |

Right-aligned: `Jobs panel ⌥J`.

Moves: load state → the **dataset pill's second tag** in the command bar ("full-res"); both
status-bar copies deleted. `Sanity: OK` → Inspector › Properties, as a KV row.
`RENDER: LOCAL` → the **Auto / Local / Remote** control in the 3D pane header, which is also
where the stream fps/latency overlay belongs.

---

## Step 6 — Inline task forms (the original stage 4)

Reference `#3b` and `#11b`. Migrate the Analysis tab from "each entry opens the existing
modal" to inline forms in the 320 px column:

- **RUN A TASK** chip cloud at the top, one chip per task, `＋ N more` for the overflow;
- the selected task's form below it, using `ToolDialogStyle`'s field geometry;
- a **RUNNING** block with inline progress, and completed tasks registered in the
  `ProductRegistry` so they appear in the session tree;
- Line-width and Stack currently create their result window from inside the dialog — they
  need their product registered instead;
- Provenance shows the selected product's parameters plus "Re-run with these", "Copy JSON",
  "Export recipe…".

**One design rule that applies to every form** (see `#8a`, `#9a`): the tools that today ask
a follow-up question through `QInputDialog::getItem` / `getText` *after* Run — mask mode,
region format, output basename — take those as **fields inside the form**. A modal question
posed after the user committed hides the parameters they just chose.

---

## Acceptance check

Open `hi_cube.fits` in a 1440 × 900 window.

1. **No ghost text anywhere over the left dock** — no "Inferno", no stray rules.
2. One pane, viewport ≈ **1112 × 735**. Height unchanged from before by design.
3. Left dock open at 272 px: SESSION DATA · DISPLAY — 3D (Isosurface/Volume,
   Composite/MIP/MinIP, Threshold + value, LUT + Edit) · LAYERS · INTERACTION ▶ ·
   SLICE ANIMATION ▶. **No bordered cards.**
4. Right side is a 36 px rail reading "INSPECTOR" with three tab names and a "0".
   Clicking "Analysis" expands it to 320 px with that tab selected.
5. Inspector › Properties **does not scroll**, and contains no rendering controls.
6. Centre toolbar above the panes carries 1/2/4, 3D/2D, a dimmed `Linked`, and the cursor
   readout. The pane has a header with an active dot, a qualifier, Auto/Local/Remote, a LUT
   strip and ⤢.
7. Status rail is exactly four slots; "full-res" appears once, in the dataset pill.
8. 2D pane: axes and colorbar **outside** the image, image square and centred, letterbox
   bands above and below the whole group — never a gap between the image and its own axis.
9. ▥ → two 552 × 735 panes (3D + 2D Slice), `Linked` enabled with "camera · channel · LUT".
   ⊞ → four 552 × 363 panes (3D, 2D Slice, Moment 0, Spectrum), `Linked` reading
   "channel · LUT", each square image ≈ 308 px.
10. ⊞ → ▣ keeps whichever pane was **active**; ▣ → ⊞ → ▣ leaves camera and zoom untouched
    and fires no backend request.
11. `⌥⇧F` collapses both sides to a ≈ 1348 px viewport, and restores.
12. Quit and reopen: the layout comes back as you left it.
