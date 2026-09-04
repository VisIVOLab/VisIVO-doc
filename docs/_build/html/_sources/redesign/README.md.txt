# Handoff: VisIVO Visual Analytics — Analysis Workspace (direction 1d)

## Overview

VisIVO Visual Analytics is a Qt 6 / VTK desktop client for interactive visualisation of
astronomical FITS data (2-D images and 3-D spectral cubes), backed by a Python FastAPI
service that does the heavy computation locally or on HPC.

This handoff covers a redesign of the **client's information architecture**, focused on
menus, sidebar, and the visualisation + analysis surface. The chosen direction is
called **1d — the analysis workspace**.

The single idea behind it: **every analysis result becomes a first-class "product"
attached to the dataset that produced it**, instead of opening a detached window.
Today `ToolResultDialog`, `ScienceMapWindow`, `PvDiagramWidget` and `ChannelMapsWindow`
each spawn an independent top-level window; after three analyses the researcher has six
windows and no record of which parameters produced which map. The redesign replaces that
with a session tree (products nested under their parent dataset), inline task forms in a
contextual inspector, and a provenance view.

## About the Design Files

The files in this bundle are **design references authored in HTML**. They are prototypes
showing the intended layout, spacing, colour and behaviour — **not production code to
copy**. The target codebase is C++ / Qt 6 Widgets (with QSS stylesheets and VTK render
widgets). The task is to **recreate these designs inside that existing Qt environment**,
reusing its established patterns:

- `VisivoTheme` (`src/gui/theme/VisivoTheme.h/.cpp`) — the design-token source of truth.
  All colours below already exist there as accessor functions. **Do not hard-code hexes;
  call the accessors** so dark/light both work.
- `SidebarPanel` (`src/gui/SidebarPanel.h/.cpp`) — the rail + stacked-page sidebar.
- `SegmentedToggle` (`src/gui/theme/SegmentedToggle.h/.cpp`) — segmented mode controls.
- `QDockWidget` workspace assembled in `vtkWindowCube::setupWorkspaceDocks()`.
- Async pattern: `QtConcurrent::run` + heap-allocated `QFutureWatcher` (see
  `MomentMapController`, `NoiseController`, `PvController`).

## Fidelity

**High fidelity.** Colours, typography, spacing, border radii and copy in the HTML are
final and are taken directly from `VisivoTheme`. Recreate them exactly, via the theme
accessors.

One deliberate exception: the VTK viewport contents are drawn as striped placeholders
with monospace captions. Those areas are the real
`QVTKOpenGLNativeWidget` / `RemoteRenderWidget` render surfaces and are unchanged by
this design — only the chrome around them changes.

## Screens / Views

### Screen A — `1a` Cube viewer, current (reference only)

Pixel recreation of what ships today (`vtkWindowCube` + `ui/vtkWindowCube.ui` +
`CubeUiAssembler`). Included as the before-state baseline. **Do not implement.**

### Screen B — `1b` Image viewer, current (reference only)

Pixel recreation of `vtkWindowImage` + `ui/vtkWindowImage.ui` `pageLayer`, with the
Tools menu shown open at its real length (~35 flat actions). Included to document the
problem. **Do not implement.**

For accuracy, the current Layer Settings page is exactly: a "LAYER SETTINGS" header, an
unframed control group (`frameLayerControls`, `QFrame::NoFrame`, 8 px margins, 8 px
spacing) holding `comboLut` / a row of two radio buttons `radioLinear` (checked) and
`radioLog` with a spacer and a fixed-size `btnLutEdit` labelled "..." / `sliderOpacity`
(0–100, default 100), then a "Layers" sub-header and `listLayer` — a `QListView` with
`InternalMove` drag reorder. Sidebar width here is **272 px**, not the cube viewer's 220.
There is no colormap card, no min/max fields and no scaling third option in the image
viewer today; `3c` adds the per-layer LUT/scaling/opacity treatment.

### Screen C — `1c` Conservative proposal (rejected, kept for reference)

Same Qt shell, reorganised menus and a Results dock. **Not the chosen direction**, but
its menu taxonomy (Analyse / Regions / Export) is reused inside 1d's command palette and
context menus, so the grouping below is still normative.

### Screen D — `1d` Analysis workspace — **THIS IS THE ONE TO BUILD**

Window: 1440 × 880 reference size (resizable; min 1280 × 800 as today).
Vertical stack: command bar (46 px) → body (flex) → jobs rail (28 px).
Body is a horizontal split: left panel 272 px, centre flex, right panel 320 px.

#### D.1 Command bar — height 46 px

Background `kSurfaceContainerLow()`, bottom border `1px solid rgba(4,138,191,0.22)`,
horizontal padding 12 px, item gap 12 px.

Left to right:
1. **App mark** — 8×8 px square `#048ABF` (radius 2) + "VisIVO", Lato 13 px / 900.
   Followed by a 1 px vertical divider in `kOutline()`, 26 px tall, 12 px right padding.
2. **Dataset switcher** — pill: background `kSurfaceContainerHigh()`, 1 px `kOutline()`,
   radius 6, padding 5 px 11 px, gap 8. Contains: kind badge ("CUBE") in JetBrains Mono
   10 px / 700, colour `kPrimary()`; filename in Mono 12 px / 600 `kOnSurface()`;
   dimensions in Mono 11 px `kOnSurfaceVariant()`; a ▼ chevron 9 px.
   Click opens a dataset list built from `RecentDatasetsManager` + the live session list.
3. **Search field** — centred, width 420 px, background `kBackground()`, 1 px
   `kOutline()`, radius 7, padding 6 px 12 px. Placeholder "Search datasets, tools,
   products…" 12 px `kSecondary()`. Right-aligned "⌘K" key cap: Mono 10 px, 1 px
   `kOutline()`, radius 3, padding 1 px 5 px. Opens the existing `CommandPalette`.
4. **Remote chip** — pill radius 999, background `kSurfaceContainerHigh()`, 1 px
   `kOutline()`, padding 5 px 11 px. 6 px status dot + Mono 10 px text
   "hpc-node-42 · 1 running". Dot colour: `kSuccess()` idle, `kRunning()` when
   `BackendHealth.runningTasks > 0`, `kError()` when unreachable.
5. **Theme toggle** — 2-segment control, 1 px `kOutline()`, radius 6, background
   `kBackground()`, 2 px inner padding. Active segment background
   `kSurfaceContainerHighest()`. Wire to `ThemeManager`.

#### D.2 Left panel — Session data, width 272 px

Background `kSurfaceContainerLow()`, right border 1 px `kOutline()`.

- **Header** — padding 11 px 14 px, bottom border `rgba(4,138,191,0.20)`.
  "SESSION DATA" Lato 10 px / 700, letter-spacing 0.14em, `kOnSurfaceVariant()`.
  Right: a "＋" glyph 14 px in `kPrimary()` (opens `RemoteFileBrowserDialog`).
- **Tree** — padding 8 px, row gap 2 px.
  - *Dataset row*: padding 7 px 8 px, radius 5. Selected state: background
    `rgba(4,138,191,0.16)` + 2 px left border `kPrimary()`. Contains a ▼/▶ disclosure
    (9 px), a kind badge (Mono 9 px / 700, background `rgba(4,138,191,0.18)`, radius 3,
    padding 1 px 5 px — CUBE / IMAGE / CAT), and the name (Lato 12 px / 600, ellipsised).
  - *Product row* (child): padding 6 px 8 px with **left padding 26 px**, radius 5.
    A ◈ glyph 10 px coloured by state — `kPrimary()` ready, `kRunning()` running,
    `kCache()` cached/retrieved — then the label (Lato 12 px), then a right-aligned
    Mono 9 px hint ("M0", "PV", "62%", source count).
  - Hover: background `kSurfaceContainerHigh()`.
- **Layers block** — pinned to the bottom, top border `rgba(4,138,191,0.20)`, padding
  11 px 14 px, row gap 7 px. Title "LAYERS — 3D VIEW" (10 px / 700, 0.14em).
  Each row: 14×14 px checkbox (radius 3; checked = filled `#048ABF` with a white ✓ at
  9 px; unchecked = 2 px `kOutline()` border), label Lato 12 px, and a right-aligned
  affordance — a 34×9 px LUT gradient strip, an opacity percentage, or a source count.

#### D.3 Centre — pane grid

Background `kBackground()`, padding 8 px, gap 8 px.

- **Pane toolbar row** (above the grid, gap 8):
  - Layout selector: 3 segments (▣ single / ▥ split / ⊞ quad), 1 px `kOutline()`,
    radius 6, background `kSurfaceContainerLow()`, active segment background
    `kSurfaceContainerHighest()` with `kPrimary()` ink.
  - **Linked chip**: background `kSurfaceContainerLow()`, 1 px `kOutline()`, radius 6,
    padding 5 px 11 px. "⛓" in `kPrimary()`, "Linked" Lato 11 px / 600, then Mono 10 px
    "camera · channel · LUT". Click opens a popover to toggle each axis of sync
    individually. Backed by `vtkWindowCube_Link.cpp`.
  - Right-aligned live cursor readout, Mono 10 px `kOnSurfaceVariant()`:
    "α 00:42:44.3  δ +41:16:09  ·  3.87e-02 Jy/beam".
- **Grid** — CSS reference is `grid-template-columns: 1.35fr 1fr`,
  `grid-template-rows: 1.4fr 1fr`, with the 3D pane spanning both rows. In Qt this is
  a nested `QSplitter` / dock arrangement; the three layout buttons restore three saved
  `QMainWindow::saveState()` blobs.
- **Pane frame** — 1 px `kOutline()`, radius 8, background `kSurfaceContainerLow()`,
  clipped. Header: padding 7 px 10 px, bottom border 1 px `kOutline()`, gap 8.
  Contains a 5 px active dot (`kPrimary()`, only on the focused pane), title Lato 11 px
  / 700, a Mono 10 px qualifier ("composite", "full-res", "circle r=12 px"), a spacer,
  optional 34×9 px LUT strip, and a "⤢" maximise glyph 11 px `kSecondary()`.
- **Viewport** — always dark, in **both themes**. This is a deliberate rule already
  documented in `VisivoTheme.h`: scientific colormaps are designed against black.
  Background `#000004`.
- **Channel scrubber** — the existing `ChannelOverlay`, unchanged: absolutely positioned
  10 px from the pane's left/right/bottom, height 28 px, background
  `rgba(0,0,0,0.55)`, 1 px `kOutline()`, radius 6, padding 0 10 px, gap 10.
  "CH" Mono 10 px muted / channel number Mono 11 px `kOnSurface()` (min-width 38 px) /
  slider (groove 4 px, sub-page `#048ABF`, handle 12 px with a 2 px
  `kSurfaceContainer()` ring) / "v" / velocity Mono 11 px right-aligned min-width 110 px.
- **Colorbar** — 12–14 px wide vertical gradient strip inset 10–12 px from the pane's
  right edge, 1 px `rgba(255,255,255,0.18)` border. Built with the existing
  `CubeUiAssembler::buildLutPreview()` / `ColorMaps`.

#### D.4 Right panel — Contextual inspector, width 320 px

Background `kSurfaceContainerLow()`, left border 1 px `kOutline()`.

- **Tab bar** — padding 0 12 px, bottom border `rgba(4,138,191,0.20)`. Tabs
  "Properties / Analysis / Provenance": padding 11 px 12 px, Lato 12 px. Inactive
  `kOnSurfaceVariant()` / 500, active `kPrimary()` / 600 with a 2 px bottom border in
  `kPrimary()`. Matches the `QTabBar` rules already in `darkStyleSheet()`.
- **"RUN A TASK" chip cloud** — padding 13 px 14 px 11 px, bottom border
  `rgba(4,138,191,0.20)`. Label 10 px / 700, 0.14em. Chips wrap with 6 px gap: radius 5,
  padding 5 px 10 px, Lato 11 px / 600. Selected chip = filled `#048ABF` with
  `kOnPrimary()` ink; others = `kSurfaceContainerHighest()` + 1 px `kOutline()`.
  A final ghost chip "＋ 9 more" (transparent, `kOnSurfaceVariant()`) opens the full list.
  Chip set: Moment map, PV diagram, Line-width, Baseline, Stack, Source find, then
  Noise, Channel maps, Pixel math, Mosaic, Publication figure, Image quality,
  Kinematic model, Spectral index, Faraday RM.
- **Task form** — padding 13 px 14 px, row gap 11 px, bottom border
  `rgba(4,138,191,0.20)`.
  - Title row: task name Lato 13 px / 700 + right-aligned endpoint in Mono 10 px
    `kOnSurfaceVariant()` (e.g. `/v1/cube/moment`). Showing the endpoint is intentional —
    the audience is researchers who script against the same API.
  - Field label: Lato 11 px `kOnSurfaceVariant()`, 5 px above its control.
  - Segmented value picker (moment order 0/1/2/3/4/5/6/8): 1 px `kOutline()`, radius 4,
    background `kBackground()`, 1 px inner padding; segments padding 5 px 0, radius 3,
    11 px; active filled `#048ABF`.
  - Range slider: groove 4 px `kSurfaceContainerHighest()`, selected span `#048ABF`,
    two 12 px handles with a 2 px `kSurfaceContainerLow()` ring. Above it a label row
    ("Channel range" / "20 – 96"), below it the physical equivalent in Mono 10 px
    `kSecondary()` ("−212.4 … +38.9 km/s").
  - Checkbox + inline value: 14×14 px checkbox, label Lato 12 px, right-aligned value
    field (background `kBackground()`, 1 px `kOutline()`, radius 4, padding 3 px 8 px,
    Mono 11 px).
  - Action row: primary button flex-1, padding 9 px, radius 6, `#048ABF` fill,
    `kOnPrimary()` ink, Lato 12 px / 700, label "Run on backend". Secondary "Preview"
    button: padding 9 px 12 px, `kSurfaceContainerHighest()`, 1 px `kOutline()`.
  - Cost line below, Mono 10 px `kSecondary()`: "est. 6 s · 77 channels · session 8f3c…a91".
- **"RUNNING" block** — padding 13 px 14 px. Header row: "RUNNING" 10 px / 700 0.14em +
  right-aligned Mono 10 px "1 of 4". Job card: background `kBackground()`, 1 px
  `kOutline()`, radius 6, padding 10 px, gap 7. Row = ◈ in `kRunning()` + label 12 px +
  right-aligned Mono 10 px percentage. Progress bar: 3 px track
  `kSurfaceContainerHighest()`, radius 2, fill `kRunning()`. Detail line Mono 10 px
  `kSecondary()`.

#### D.5 Jobs / status rail — height 28 px

Background `kSurfaceContainerLow()`, top border `rgba(4,138,191,0.25)`, padding
0 12 px, gap 12, JetBrains Mono 10 px `kOnSurfaceVariant()`. Four fixed slots separated
by a "│" in `kSecondary()`, each prefixed by a 6 px dot:

| Slot | Dot | Content | Source |
|---|---|---|---|
| Health | `kSuccess()` | "backend 200 · 41 ms" | `BackendClient::health()` |
| Queue | `kRunning()` | "1 job running · 3 queued" | `BackendHealth.runningTasks`, `taskRegistryEntries` |
| Cache | `kCache()` | "slice cache 128 / 256" | client-side slice cache |
| Session | — | "session 8f3c…a91" | `X-Visivo-Session` |

Right-aligned: "Jobs panel ⌥J" in `kPrimary()`, opening the existing
`DiagnosticsWindow`.

## Menu Taxonomy (applies to the command palette and context menus)

The current `menuTools` carries ~30 actions in the cube viewer and ~35 in the image
viewer, flat. Regroup as:

- **Analyse › Statistics** — Estimate Noise…, Pixel Histogram…, Image Quality / Artifacts…
- **Analyse › Spectral** — Extract Spectrum (⌘E), Position–Velocity Diagram… (⌘P),
  Line-Width Map…, Baseline Subtraction…, Stack Spectral Cubes…
- **Analyse › Maps** — Compute Moment… (⌘M), Channel Maps…
- **Analyse › Sources & Kinematics** — Source Finding (SoFiA-2)…, Kinematic Lasso,
  Kinematic Model Overlay…
- **Analyse › Combine** — Pixel Math / Spectral Index…, Mosaic (noise-weighted)…
- **Regions** — Box / Circle / Polygon / Annulus, Import Region (CRTF/DS9)…,
  Export Region (CRTF/DS9)…, Mask 3-D Region…
- **Export** — Sub-Cube as FITS…, Moment Map as FITS…, Send Slice to Image Viewer…,
  Export Movie…, Publication Figure…, Save Plot as PNG…
- **View** — Rendering ▸, Volume Rendering ▸, 2D Panel ▸, Overlays ▸, WCS ▸,
  Layout ▸ (Single / Split / Quad, Reset Layout, Controls Sidebar ⌘\\)

### Screen E — `3a` Inspector, all three tabs + failed state + empty state

Three 320 px panels, one per tab, plus a whole-window empty state.

**Tab bar** (identical on all three): padding 0 12 px, bottom border
`rgba(4,138,191,0.20)`. Tabs "Properties / Analysis / Provenance", padding 11 px 12 px,
Lato 12 px. Inactive `kOnSurfaceVariant()` / 500; active `kPrimary()` / 600 with a 2 px
bottom border `kPrimary()`.

**Shared section pattern.** Every block below the tab bar is: padding 12–14 px, a
section label (Lato 10 px / 700, letter-spacing 0.14em, `kOnSurfaceVariant()`, 4 px
bottom margin), then KV rows. A KV row is a flex row with `align-items: baseline`: key
Lato 11 px `kSecondary()` flex-1, value JetBrains Mono 11–12 px `kOnSurface()`
right-aligned. Blocks are separated by a 1 px `rgba(4,138,191,0.20)` bottom border.
This is exactly the pattern already produced by
`CubeUiAssembler::buildInfoStatsPage()` — reuse its helpers.

#### E.1 Properties tab

1. **Identity header** — padding 14 px. Kind badge (Mono 9 px / 700, `kPrimary()` on
   `rgba(4,138,191,0.18)`, radius 3, padding 1 px 5 px — "M0" / "PV" / "CUBE") +
   name Lato 13 px / 700. Subtitle line Mono 10 px `kSecondary()`:
   "derived · 1024 × 1024 · float32".
2. **DISPLAY** — LUT row (44×12 px gradient + name Lato 12 px + "Edit" link 10 px / 600
   `kPrimary()` that opens `LUTCustomizerDialog`); a 3-segment scaling control
   (Linear / Log / Sqrt); a two-field min/max row (each: `kBackground()`, 1 px
   `kOutline()`, radius 4, padding 6 px 8 px, Mono 11 px); an opacity row —
   label min-width 52 px, slider, Mono 10 px percentage.
3. **STATISTICS** — KV rows Min / Max / Mean / RMS / Unit. Values are the existing
   `lineImgMin`…`lineImgRms` line edits, reparented.
4. **WCS** — KV rows Status / Frame / Pixel scale / Beam. Status value is coloured:
   `kSuccess()` for OK, `kWarning()` for Degraded, `kWarning()` darker for Repaired —
   same mapping already in `buildInfoStatsPage()`.

#### E.2 Analysis tab — failed state

Same chip cloud and form as D.4, with an error card inserted directly under the task
title:

- Card: background `kErrorContainer()`, 1 px `kError()`, radius 6, padding 11 px, gap 7.
- Header row: "▲" glyph 11 px `kError()` + "Task failed" Lato 12 px / 700 `kError()` +
  right-aligned HTTP code Mono 10 px `kError()`.
- Message: **the backend's real text**, Mono 10 px, line-height 1.6, `kOnSurface()`.
  Source: `MomentMapController::lastError()` and equivalents — they already carry the
  prefixed backend message; today it is discarded into a generic status-bar string.
- Action row: Retry / Copy log (`kSurfaceContainerHighest()` + 1 px `kOutline()`,
  radius 5, padding 5 px 10 px, Lato 11 px / 600) and a ghost Dismiss.
- Below the card, a Mono 10 px `kSecondary()` note: "Parameters below are unchanged —
  adjust and run again." **The form must not reset on failure.**
- The field that most likely caused the failure gets a `kError()` border instead of
  `kOutline()` (in the mock: the threshold value field).

#### E.3 Provenance tab

1. **DERIVED FROM** — an indented chain, each level 6 px vertical padding, levels 2+
   indented 14 px and 28 px with a 1 px `kOutline()` left border (margin-left 6 px) and
   a "↳" glyph. Levels: source FITS path (Mono 11 px, ellipsised) → dataset (kind badge
   + label) → this product (◈ in `kPrimary()` + bold label).
2. **TASK** — KV rows: Endpoint, Task id, Computed on, Duration, Finished
   ("14:07:22 · 4 min ago" — the relative part from `BackendRecentTask.ageSeconds`),
   Engine ("dask · 8 workers").
3. **PARAMETERS** — KV rows using the **wire parameter names**, not UI labels:
   `order`, `channel_start`, `channel_end`, `threshold`, `mask`, `session_id`.
   This is the request body, so it must read like the API.
4. **Actions** — "Re-run with these" (flex-1) and "Copy JSON" side by side, then a
   full-width ghost "Export recipe…". Buttons: padding 8 px, radius 6,
   `kSurfaceContainerHighest()`, 1 px `kOutline()`, Lato 12 px / 600.
   Footnote Mono 10 px `kSecondary()`: the recipe is the exact request body.

#### E.4 Empty state — no dataset open

Reference frame 1000 × 520. Same three regions, all in a resting state:

- Command bar: the dataset pill becomes "No dataset open" with a **dashed**
  `kOutline()` border and `kSecondary()` text. Remote chip reads "backend ok · idle".
- Left panel: header unchanged; body centred with 14 px gaps — a 30 px
  `rail_layers.svg` glyph stroked in `kSecondary()`, the line "Nothing open in this
  session." (Lato 12 px `kOnSurfaceVariant()`), a primary button "Browse backend files…"
  (`#048ABF`, radius 6, padding 8 px 16 px, Lato 12 px / 700 — opens
  `RemoteFileBrowserDialog`), then "or open a recent dataset below" and up to 3 recent
  rows from `RecentDatasetsManager`.
- Centre: a single dashed-border panel (1 px dashed `kOutline()`, radius 8) with
  "No pane content" Lato 14 px / 700 `kOnSurfaceVariant()` and, below it, Mono 11 px
  `kSecondary()`: "Open a dataset, or drop a .fits file here / ⌘O · ⌘K to search".
  Accept a real file drop here.
- Right panel: tab labels dimmed to `kSecondary()`, body centred, "Select a dataset or
  product to inspect it."
- Jobs rail: "no jobs", "slice cache 0 / 256", "no session" (session slot in
  `kSecondary()`).

### Screen F — `3b` The five remaining task forms

All 320 px wide, padding 14 px, row gap 11 px, background `kSurfaceContainerLow()`,
1 px `kOutline()`, radius 8. All share D.4's field vocabulary (label 11 px, segmented
picker, inline value field, primary/secondary action row, Mono 10 px cost line).
Parameter names below are lifted verbatim from the existing dialogs — **do not rename
them**, the researchers already know them.

**F.1 Line-width map** — `/v1/spectral/linewidth`, from `LinewidthDialog`.
Read-only KV "Spectral axis → VRAD · km/s". Method: 2-segment FWHM / Equivalent width,
with a Mono 10 px hint "Per-pixel Gaussian fit." Channel range: dual-handle slider,
label row "Channel range / 1 – 112". Threshold mask: checkbox + inline value field.
When method = Equivalent width, the dialog also exposes **Rest frequency (GHz)** — show
that field only in that mode (it is conditionally shown today too).
Cost line: "est. 40 s · 1.05 M px fits".

**F.2 Baseline subtraction** — `/v1/spectral/baseline`, from `BaselineDialog`.
Fit: 2-segment Polynomial / Median. Polynomial order: inline segmented 0–3 plus an "…"
overflow (range is 0–10); hide the row entirely when Fit = Median, as today.
Line-free channel ranges: a text field, Mono 11 px, value "1-18, 99-112", with the hint
"Comma-separated, 1-based. Leave empty to fit the whole axis."
**New:** a "PREVIEW — CENTRAL SPECTRUM" block — label 10 px / 700, then a 56 px tall
dark plot area (radius 4) showing raw / fitted baseline / residual. Rationale: the
current dialog gives no way to tell whether the fit is sane before committing.
Checkbox "Register result as new dataset" (the backend already registers the result as a
new dataset; make it explicit).

**F.3 Stack spectral cubes** — `/v1/spectral/stack`, from `StackDialog`.
Method: 3-segment Mean / Median / Weighted. Weight by: inline 3-segment
Uniform / RMS / Peak — visible only when Method = Weighted.
Cube list: label row "Cubes in this session" + a live count, coloured `kSuccess()` at
≥2 and `kError()` below 2 (the current dialog already does this in red).
List container: `kBackground()`, 1 px `kOutline()`, radius 5, 5 px padding. Rows:
13 px checkbox + name (Lato 11 px, ellipsised) + right-aligned depth Mono 9 px.
**New:** a row whose depth differs from the others shows its depth in `kWarning()` and
a warning line appears below: "ngc300-co21 has a different depth — regrid or exclude
it." Today the mismatch only surfaces as a backend failure.
Run button disabled until ≥2 selected; footnote "Needs at least 2 cubes."

**F.4 Source finding (SoFiA-2)** — from `SourceFindDialog`.
"S+C threshold" inline value field, "5.0 σ" (range 0.1–100.0, 1 decimal).
"Smoothing kernel": a 3-column grid — X px / Y px / Z chan, each a small centred value
field with a Mono 9 px caption above (range 0–31, default 3). Hint "0 disables that axis."
Checkboxes "Merge adjacent detections" (the current `Merge sources` true/false combo —
a boolean deserves a checkbox) and "Open mask cube when done" (wires the existing
`openMaskCubeRequested` signal).
Footnote: "Produces a catalogue and a mask cube, both registered under this dataset."

**F.5 PV diagram** — `/v1/cube/pv`, from `PvController` / `RemotePvFetchResult`.
**New:** a live path-editor state card at the top — background
`rgba(4,138,191,0.10)`, 1 px `rgba(4,138,191,0.40)`, radius 6, padding 10 px.
Row: "✛" `kPrimary()` + "Drawing path in 2D pane" Lato 12 px / 600 + right-aligned
vertex count Mono 10 px `kPrimary()`. Hint: "Click to add a vertex · ⌫ removes the
last · Enter finishes."
KV rows: Vertices (`vertexCount`), Path length ("418 px · 10.4 ′" from `totalLength` ×
`pixelScaleArcsecPerPixel`), Samples ("`numSamples` × `depth`").
"Slit width" inline value field, "3 px" (`widthPixels`).
"Position axis": 2-segment Pixels / Arcsec — Arcsec disabled when
`positionsArcsec` is empty (`spatialUnit` != "arcsec").
Actions: "Extract" primary + "Clear path" secondary.

### Screen G — `3c` Image viewer inside the workspace shell

Same 1440 × 880 shell as D: command bar 46 → three regions → jobs rail 28. Only the
vocabulary of each region changes. Dataset pill badge reads "IMAGE".

#### G.1 Left panel — session tree + layer stack + overlays

Width 272 px. Three stacked blocks separated by `rgba(4,138,191,0.20)` borders.

1. **Session tree** — same rows as D.2. Products here are "Spectral index map" (hint
   "α") and "Pol. intensity (P)" (hint "P").
2. **Layer stack** — header row "LAYER STACK" (10 px / 700, 0.14em) with a right-aligned
   Mono 10 px "WCS aligned" status. Then one **card per layer**, padding 9 px 10 px,
   radius 6, gap 7:
   - Active layer: background `rgba(4,138,191,0.16)`, 1 px `rgba(4,138,191,0.40)`.
   - Other layers: background `kSurfaceContainerLow()` card tint
     (`rgba(255,255,255,0.03)` dark / `rgba(0,0,0,0.03)` light), 1 px
     `rgba(4,138,191,0.14)`.
   - Row 1: 14 px visibility checkbox + name (Lato 12 px, ellipsised) + optional
     right-aligned Mono 9 px tag ("base", or "contour" in `kPrimary()`).
   - Row 2: a full-width 9 px LUT gradient strip + scaling ("log" / "sqrt") +
     opacity, both Mono 9 px `kOnSurfaceVariant()`.
   - A layer promoted to contour shows instead "5 levels · 3σ … peak" in Mono 9 px
     `kSecondary()`.
   - Last item: an "Add layer from backend…" row — 1 px **dashed** `kOutline()`,
     radius 6, "＋" in `kPrimary()`. Opens `RemoteFileBrowserDialog` through
     `loadImageLayer()` / `AstroUtils`, the existing WCS-aware path.
   - Rows are drag-reorderable (that is the z-order).
3. **Overlays** — checkbox rows with a right-aligned count in Mono 10 px:
   Catalogue sources (1 284), Polarisation vectors (every 8 px), WCS grid,
   Annotations (3). These nine-odd scattered Tools actions become four toggles.

#### G.2 Centre — mode strip + sky view

Toolbar row: the 1/2/4 layout selector, then a **mode strip** — a segmented control
(same tokens) with Pan / Region / Ruler / Annotate, active segment
`kSurfaceContainerHighest()` + `kPrimary()` ink. This is the missing affordance today:
region and measurement modes are entered from a menu with no visible state.
Right-aligned cursor readout, Mono 10 px.

Pane header: active dot, "Sky view", Mono 10 px "3 layers · zoom 1:4", spacer, a
34×9 px LUT strip, "⤢".

Viewport: `#000004` in both themes. Vertical colorbar 14 px inset 12 px right.
Bottom-left **live region readout** overlay: background `rgba(0,0,0,0.55)`, 1 px
`kOutline()`, radius 6, padding 5 px 10 px — "circle r = 24 px" `kOnSurfaceVariant()`
then "Σ 3.41e+03 · μ 1.88" `kOnSurface()`, both Mono 10 px.

#### G.3 Right inspector

- Chip cloud for the image domain: Region stats, Contours, Spectral index,
  Polarisation, Faraday RM, Cross-match, Mosaic, "＋ 6 more".
- **Region statistics** form: endpoint tag reads "local" (it is computed client-side).
  Shape: 4-segment Box / Circle / Poly / Annulus. Geometry KV rows Centre / Radius /
  Pixels. A 1 px `kOutline()` divider. Result KV rows Sum / Mean / Median / RMS / Peak,
  values Mono 12 px. Actions "Save as product" (flex-1) + "Copy".
  "Save as product" is what promotes a transient measurement into the session tree —
  the replacement for today's non-modal `ToolResultDialog`.
- **MEASUREMENTS** block: one row per active measurement — glyph ("↔" distance,
  "∠" angle), label Lato 12 px, value Mono 11 px right-aligned. Row container
  `kBackground()`, 1 px `kOutline()`, radius 6, padding 8 px 10 px.

### Screen H — `4a` Result panes

The four surfaces that today are top-level windows. All become panes inside the D/G
shell, using the pane frame from D.3 (1 px `kOutline()`, radius 8, background
`kSurfaceContainerLow()`, header padding 7 px 10 px with a 1 px `kOutline()` bottom
border).

**One rule across all four: no footer bar.** `PvDiagramWidget` and
`ChannelMapsWindow` each spend 40–44 px of footer on a single "Save as PNG…" button.
As a pane that is unaffordable, so the export action becomes a compact tag in the pane
header — Mono 10 px / 600 in `kPrimary()`, reading "PNG" or "FITS" — sitting next to the
LUT strip. Where a pane needs a status line it keeps a thin one (padding 8 px 12 px, 1 px
`kOutline()` top border) carrying real values, not a button.

#### H.1 PV diagram pane — reference 760 × 520

From `PvDiagramWidget` (`resize(960, 700)` standalone today).

- Header: active dot, "PV Diagram" Lato 11 px / 700, Mono 10 px qualifier
  "cubehi-clean-m31 · major axis", spacer, 34×9 px LUT strip, "PNG" tag, "⤢".
- **Readout strip** — replaces the two `QGroupBox` forms (`Path` and `Sampling & Axes`)
  with a two-column strip: padding 9 px 12 px, 1 px `kOutline()` bottom border, a single
  1 px `kOutline()` vertical divider between columns (14 px padding each side).
  Column caption Lato 9 px / 700, letter-spacing 0.10em, `kOnSurfaceVariant()`.
  Rows: key Lato 11 px `kSecondary()` with a fixed 56 px width, value Mono 11 px
  `kOnSurface()`, selectable.
  - **PATH**: Start, End, Vertices, Length ("418.06 px", 2 decimals as today).
  - **SAMPLING & AXES**: Width ("3 px"), Spectral ("VRAD [km/s]"), Spatial
    ("arcsec", falls back to "pixels"), Scale ("1.50002 arcsec/px", `'g', 6`; "—" when
    `pixelScaleArcsecPerPixel <= 0`).
  Same eight fields as today, roughly half the vertical cost.
- Plot: centred title Lato 11 px `kOnSurface()` (the `QCPTextElement`, 10 pt today).
  Y tick labels Mono 9 px in a 52 px right-aligned gutter; X tick labels Mono 9 px below;
  axis label Lato 10 px in `kPrimary()` — matching `applyPlotTheme()`, which already
  sets `axis->setLabelColor(kPrimary())` and `setTickLabelColor(kOnSurface())`.
  Plot canvas `kBackground()`, 1 px `kOutline()` border, `QCPColorMap` with
  `setInterpolate(false)`. A 16 px vertical LUT bar on the right, 1 px `kOutline()`.
- Keep `applyPlotTheme(bool dark)` as the single place plot colours are set; it is
  already driven by `Settings.ini` `Plots/theme`.

#### H.2 Spectrum pane — reference 620 × 520

From `ProfileWidget.ui`: a `checkLive` checkbox plus `plot1` and `plot2`
(`QCustomPlot`).

- Header: title, Mono 10 px context ("circle r = 12 px · 449 px"), spacer, the
  **Live update** checkbox (14 px box + Lato 11 px label) moved into the header —
  today it sits above the plots and costs a full row — then "PNG" and "⤢".
- Body: the two plots stacked, equal flex, separated by a 1 px `kOutline()` divider,
  10 px gap, padding 10 px 12 px 8 px.
- Each plot gets a caption row: section label Lato 10 px / 700 letter-spacing 0.10em
  ("SPECTRAL PROFILE" / "SPATIAL CUT"), spacer, and a right-aligned Mono 10 px derived
  readout ("peak 8.12e+00 @ −118.4 km/s" / "row 1993 · channel 57"). That readout is new
  and is the reason to keep both plots visible at pane size.
- Y gutter 46 px, tick labels Mono 9 px; axis label Lato 10 px `kPrimary()`
  ("VRAD [km/s]", "X [pixels]").

#### H.3 Science map pane — reference 620 × 520

From `ScienceMapWindow` (VTK image + `vtkScalarBarActor`).

- Header: kind badge ("M0"), title "Moment 0 — integrated", spacer, LUT strip,
  "FITS" export tag, "⤢".
- Viewport `#000004` in both themes. The scalar bar stays a VTK actor: 16 px wide,
  inset 14 px from the right, stopping 44 px above the bottom, with its unit
  ("Jy/beam·km/s") as a Mono 9 px label beneath — that is the existing
  `m_colorbar->SetTitle(barTitle)`.
- Bottom-left cursor overlay, same tokens as `ChannelOverlay`: background
  `rgba(0,0,0,0.55)`, 1 px `kOutline()`, radius 6, padding 5 px 10 px — coordinates in
  `kOnSurfaceVariant()`, value in `kOnSurface()`, both Mono 10 px.
- Status line: "min 0.000 · max 42.18 · ch 20–96 · 3σ mask" in Mono 10 px, then a
  right-aligned secondary button "Send to image viewer" (the existing
  `m_actSendSliceToImage` action).

#### H.4 Channel maps pane — reference 760 × 520

From `ChannelMapsWindow`. Keep the existing header string verbatim:
"Channel Maps  ·  CH %1–%2  stride %3  ·  %4" — title Lato 13 px / 700, the rest
Mono 11 px `kOnSurfaceVariant()`. `m_statusLabel` keeps its place on the right
("Loading…" → "20 of 20 rendered"), Lato 11 px `kOnSurfaceVariant()`.

- Grid: `QGridLayout`, 8 px margins, **4 px spacing** (as today), square tiles, column
  count derived from pane width so that **every requested channel is visible without
  scrolling** — at 760 px that is 7 columns × 3 rows for the 20 channels of
  "CH 20–96 stride 4". Each tile carries its absolute channel number top-left,
  Mono 9 px `rgba(241,244,246,0.8)`.
  The tile count, the header range/stride and `m_statusLabel` must always agree: if the
  grid ever has to scroll, the status label reads "showing N of M" instead of
  "M of M rendered".
- The tile matching the viewer's current channel gets a 1 px `kPrimary()` border and its
  number in `kPrimary()` / 700 — the link back to the 3D pane that is missing today.
- Clicking a tile opens it full size (the existing per-channel dialog, "Channel %1").
- Status line: Mono 10 px hint "click a tile to open it full-size" + right-aligned
  "Save mosaic as PNG…" secondary button.

### Screen I — `4b` Jobs panel (⌥J)

Reference 1100 × 480 — the size `DiagnosticsWindow` already uses.

Two tabs in a bar with the D.4 tab treatment (padding 11 px 14 px, Lato 13 px, active
`kPrimary()` / 600 + 2 px `kPrimary()` bottom border):

- **Log** — the existing `DiagnosticsWindow` verbatim: `QTableView` over
  `DiagnosticsFilterModel`, alternating rows, columns Time / Level / Category / Source /
  Operation / Context, the Level and Category combos (All / Debug / Info / Warning /
  Error and All / Scientific / Client / Backend / Task / WCS / Remote / Rendering /
  Performance), the "Auto-scroll" checkbox, "Copy Selected" and "Clear". **Do not
  redesign it** — it already follows the global QSS.
- **Jobs** — new, and the tab that opens by default.

Right of the tab bar: the same health pill as the command bar —
"hpc-node-42 · 200 · 41 ms", Mono 10 px, radius 999.

**Jobs toolbar** — padding 9 px 12 px, `kSurfaceContainerLow()`, 1 px `kOutline()`
bottom border. A 4-segment state filter (All / Running / Queued / Failed), a dataset
combo ("All datasets", min-width 200 px), spacer, a Mono 10 px registry readout
("registry 12 / TTL 900 s" from `taskRegistryEntries` and `taskTtlSeconds`), then
"Copy selected" and "Clear finished".

**Jobs table** — header row Lato 10 px / 700 letter-spacing 0.10em: a 24 px state-glyph
column, TASK, KIND, DATASET, NODE, STARTED, ELAPSED, STATUS (130 px, right-aligned).
Row padding 8 px 12 px, Lato 12 px, 1 px `kOutline()` bottom border. Monospace for
dataset, node, times and durations. State glyph "◈" coloured by state.

Status cell by state:
- **Running** — Mono 10 px / 700 "62 % · ~14 s" in `kRunning()` above a 3 px progress
  track (`kSurfaceContainerHighest()`, fill `kRunning()`, radius 2, width 110 px).
  Row background `rgba(4,138,191,0.06)`.
- **Queued** — pill "QUEUED", `kOnSurfaceVariant()` on `kSurfaceContainerHighest()`.
- **Failed** — pill "FAILED", `kError()` on `kErrorContainer()`. The row **expands in
  place**: a detail block (padding 9 px 12 px 11 px with 46 px left indent, background
  `kBackground()`) with the backend message in Mono 10 px `kError()` and three buttons —
  Retry, **Open in inspector** (loads the task into the Analysis tab with its parameters
  intact), Copy log.
- **Done** — pill "DONE", `kSuccess()` on `kTertiaryContainer()`.
- **Cached** — pill "CACHED", `kCache()` text with a 1 px `kCache()` border, no fill.
  Cache hits get rows too (node column reads "cache", elapsed "4 ms") — this is how a
  researcher finds out why a slice was instant.

### Screen J — `5a` Data Hub

Reference 1440 × 880. Same command bar and jobs rail as D; no left/right panels — a
single centred content column, `max-width: 1100px`, padding 28 px 32 px 20 px, 20 px gap.

**The current `DataHubWidget` has seven blocks. Four are removed because the new shell
already provides them:**

| Removed block | Replaced by |
|---|---|
| `buildSearchBar()` — the ⌘K trigger bar | the command bar's search field, always visible |
| `buildBackendCard()` — path-pill, status badge, "Check Connection" | the command-bar health chip (which polls anyway) |
| `buildStatusBar()` — its own health/activity/session row | the jobs rail |
| `buildLast5JobsPanel()` + "System Insights" | the ⌥J Jobs tab, which shows all tasks, not five |
| credits line | the About dialog |

**Kept, restructured:**

1. **Header** — 44 px logo (`createVisivoLogoLabel(44)`) + a title column: "Data Hub"
   Lato **26 px / Black** in `kPrimary()` (exactly as today), subtitle Lato 12 px
   `kOnSurfaceVariant()` — keep the string verbatim: "Backend-authoritative workflow —
   scientific operations run on remote nodes."
2. **Quick actions** — was a vertical list; becomes a **5-column card grid**, 10 px gap.
   Card: padding 13 px 12 px, background `kSurfaceContainerLow()`, 1 px `kOutline()`,
   radius 8. First card (Open Remote Dataset) gets a `rgba(4,138,191,0.40)` border as the
   default action. Contents: a 16 px themed rail icon, right-aligned Mono 10 px shortcut,
   title Lato 13 px / 700, description Lato 11 px `kOnSurfaceVariant()` line-height 1.45.
   The five specs are verbatim from `buildQuickActionsList()`:
   Open Remote Dataset… / ⌘O / "Browse FITS on the backend filesystem";
   Open 3D Catalogue… / "CSV / VOTable / FITS / speck / IPAC in the source viewer";
   Open VBT… / ⇧⌘B / "VisIVO Binary Table — points / volumes";
   HiPS Viewer / "Hierarchical sky surveys (DSS, 2MASS, …)";
   VLKB Inventory / "Browse VLKB sources & add as image layers".
3. **Recent datasets** — promoted from the 60 % column to the **dominant block**
   (`flex: 1`). Card with a header row ("RECENT DATASETS", 10 px / 700, 0.18em — the
   `makeSectionLabel()` treatment) and a right-aligned Mono 10 px note
   "6 · reopens on the current backend". Row grid
   `60px 1.6fr 2fr 1fr 150px`, padding 9 px 14 px, 1 px `kOutline()` bottom border:
   kind badge (CUBE / IMAGE / CAT / SKAVA — SKAVA in `kCache()`), name (Lato 12 px, the
   most recent row at 600 weight), directory Mono 11 px `kOnSurfaceVariant()`, relative
   age Mono 11 px `kSecondary()` (`relativeAge()` already exists in
   `DataHubWidget_Status.cpp`), then an **Open** button (primary on the first row,
   secondary elsewhere) and a "✕" remove button. SKAVA rows show
   "SKAVA obs_id: %1" and the note "re-queries SKAVA on open", as today.
   Removing still warns "Remove from Recent Datasets — the actual dataset on the backend
   is not deleted."
4. **Workspace exports** — collapsible (▼ chevron), header "WORKSPACE EXPORTS" +
   Mono 10 px summary "3 files · 412 MB", then "Clean up older than…" and "Refresh".
   Row grid `2.4fr 0.7fr 1fr 210px`: filename Mono 11 px, size, modified
   (`yyyy-MM-dd hh:mm`, as today), then Open / Download… / Delete. Delete label in
   `kError()`.

### Screen K — `5b` Startup dialog

Two states, reference widths 520 px (running) and 560 px (failed). Padding 26 px.
Structure is unchanged from `StartupDialog`: header, step frame, 3 px indeterminate
`QProgressBar`, status label, inline auth panel, log console.

- **Header** — 44 px logo + "VisIVO Visual Analytics" Lato 18 px / 700 + version
  "v2.4.1" Mono 11 px `kOnSurfaceVariant()`. 20 px gap below.
- **Step frame** — background `kSurfaceContainerLow()`, radius 8, padding 14 px 16 px,
  row gap 10 px (matches the existing `QGridLayout`). Three rows, labels verbatim:
  **Backend**, **Authentication**, **Loading application**. Each row: a 20 px
  centre-aligned glyph, label Lato 13 px, right-aligned *italic* detail Lato 12 px.
- **Change 1 — the step glyph carries state in colour**, instead of every row staying
  `kOnSurfaceVariant()`: "●" `kSuccess()` done · "◐" `kPrimary()` running · "○"
  `kOnSurfaceVariant()` pending · "✕" `kError()` failed. The failed row's detail text
  also turns `kError()`.
- **Change 2 — the progress bar becomes determinate** once the step count is known
  (3 steps → 33 / 66 / 100 %), so "is it stuck?" is answerable. Keep
  `setRange(0, 0)` only while a step's duration is genuinely unknown.
  Track `kSurfaceContainerLow()`, chunk `#048ABF`, height 3 px, radius 2.
- **Auth panel** — background `kSurfaceContainerLow()`, 1 px `rgba(4,138,191,0.28)`,
  radius 8, padding 13 px. An explanatory line naming both sources
  (`~/.visivo_token`, `$VISIVO_TOKEN`) in `kPrimary()` mono, the token field
  (placeholder verbatim: "Paste your API token here…"), and a primary
  **Save & Continue**.
- **Log console** — background **`#060E18`** in dark (the existing `logBg`;
  `kSurfaceContainerHighest()` in light), 1 px `kOutline()`, radius 6, padding 10 px
  12 px, Mono 10 px, line-height 1.75.
- **Change 3 — log lines are colour-coded by severity**: normal `#8AB8CC`,
  warnings `kWarning()`, errors `kError()`. Free, because `BackendLauncher` already
  prefixes every line (`[launcher]`, `[uvicorn]`, `[visivo]`, `[client]`).
- **Buttons** — right-aligned secondary row. Running: Show Logs / Quit.
  Failed: Retry / Hide Logs / Quit.

### Screen L — `5c` Browse Remote FITS Files

Reference 1040 × 640. From `RemoteFileBrowserDialog`; the two-panel structure
(tree left, details right) is kept.

- **Path row** — "Path" label Lato 13 px + a field styled like the viewer path-pill
  (⌂ in `kPrimary()`, dim parent path + bright leaf, Mono 12 px). Placeholder stays
  "Enter a remote path". Then "↑ Up" and "⌂ Home" secondary buttons.
- **Options row** — "Sort by" + a **3-segment control** Name / Size / Modified
  (was a `QComboBox`; three fixed options belong in a segmented control), a
  "FITS files only" checkbox, and a right-aligned Mono 11 px count
  "12 entries · 4 FITS".
- **Tree** — `flex: 1.35`, header row Lato 10 px / 700 0.10em
  NAME / SIZE (right) / MODIFIED (right), grid `1fr 90px 130px`, rows padding
  8 px 12 px Lato 13 px. Directories get a "▸" disclosure; files get a **kind badge**
  (CUBE / IMG, Mono 9 px / 700). Selected row: text and numbers in `kPrimary()`, badge
  on `rgba(4,138,191,0.18)` — matching the QSS list-item selected rule.
- **Details pane** — `flex: 1`. "DETAILS" section label, KV rows Name / Size / Modified /
  **Dimensions** / **WCS** (status coloured: `kSuccess()` OK, `kWarning()` degraded).
  Then "FITS HEADER" and a scrollable card (background `kBackground()`, 1 px
  `kOutline()`, radius 6, Mono 10 px, line-height 1.75) with the raw cards.
- **Addition, and the point of the screen**: Dimensions and WCS status come from the
  `/v1/files/header` call the dialog already makes — so a researcher learns that a
  452 MB cube has broken WCS *before* waiting for it to open.
- **Footer** — the selected absolute path echoed in Mono 11 px, then Cancel and a
  primary **Open**.

### Screen M — `5d` Settings

Reference 900 × 600. Today `SettingsDialog.ui` stacks six `QGroupBox` widgets in one
scrolling column in the order they were added — **Remote Backend last**, although the
server URL is the one field a new user must fill.

- **Left nav** — 216 px, background `kSurfaceContainerLow()`, 1 px `kOutline()` right
  border, padding 12 px 8 px. "SETTINGS" section label, then rows padding 8 px 10 px,
  radius 5, Lato 13 px. Selected: background `rgba(4,138,191,0.16)` + 2 px left border
  `kPrimary()`, label at 600. Order, reprioritised:
  **Remote Backend** (default) · Appearance · Visualization · VLKB · Panoramic View ·
  Python. At the bottom, a Mono 10 px note naming the `Settings.ini` location.
- **Right pane** — padding 22 px 26 px. Page title Lato 18 px / 700, a 12 px
  `kOnSurfaceVariant()` explanatory paragraph, then fields: label Lato 12 px
  `kOnSurfaceVariant()` above the control, help text Lato 11 px `kSecondary()` below.
  Section rules are 1 px `kOutline()`.
- **Field mapping — keep every real name and placeholder from the `.ui` file:**
  - *Remote Backend*: "Server URL" (`lineBackendUrl`, placeholder
    `http://127.0.0.1:8000`); "Access token" (`lineBackendToken`, password echo,
    placeholder "Leave empty to auto-load from ~/.visivo_token") with the
    **👁** show/hide toggle and **⎘** copy buttons.
  - *Appearance*: "Theme:" (`comboTheme` — System / Light / Dark) and
    "Plot background:" (`comboPlotTheme` — Dark / Light).
  - *Visualization*: "Max Glyphs:" (`lineMaxGlyphs`).
  - *VLKB*: "VLKB URL:" (`lineVLKBUrl`), "User status:" with the
    `labelVLKBAuth` state ("Not authenticated") and a "Sign in" button,
    "Search when loading local data" (`checkVLKBSearch`).
  - *Panoramic View*: "URL:" (`linePanoramicView`) + browse tool button.
  - *Python*: "Interpreter path:" (`linePythonPath`) + browse tool button. Document the
    six-level resolution chain here as help text — it is currently only in the README.
- **Addition: "Test connection"** next to the server URL, with an inline result line
  (6 px status dot + Mono 10 px "200 · 38 ms · 4 workers · 0 running"). Today you have to
  close Settings and press "Check Connection" in Data Hub to learn whether the URL works.
  Two checkboxes carry existing behaviour that had no UI: "Start the bundled backend
  automatically" (noting the `--no-backend-autostart` / `VISIVO_NO_BACKEND_AUTOSTART`
  overrides) and "Prefer remote GPU rendering when available".
- **Footer** — a Mono 10 px caveat "Changes apply on Save · theme changes rebuild open
  windows" (true: inline-styled widgets do not restyle live), then Cancel and Save.

### Screen N — `6a` 3D catalogue viewer

Reference 1440 × 880, the same D shell. From `vtkWindowCatalogue3D` +
`vtkWindowCatalogue3D_Setup.cpp` + `Catalogue3DTableModel`.

- **Command bar** — dataset pill badge "CAT", filename, and the source count
  ("184 217 sources").
- **Left panel** — session tree (a catalogue's derived products are *selections*:
  "Selection — 3 filters", count as the right-aligned hint), then two new blocks:
  - **SCHEMA** — KV rows RA / Dec units and frame, Distance unit, raw column count,
    morphology class count. Read off the `Catalogue3DSchema` the model already holds.
  - **LEGEND — MORPHOLOGY** — one row per class: a 9 px colour dot, the class name, and
    the source count in Mono 10 px. **New.** "Morphology colors" is a checkbox today with
    nothing anywhere telling the user which colour means which class.
- **Centre** — layout selector, a mode strip (Orbit / Select / Pick), a right-aligned
  Mono 10 px budget notice in `kWarning()`, then two panes:
  - **3D Scene** (`flex: 1.6`) — pane header qualifier "ellipsoid · FK5 / J2000",
    LUT strip, viewport `#000004`, vertical colorbar, and a bottom-left overlay showing
    the live rubber-band selection count (same overlay tokens as `ChannelOverlay`).
  - **Sources** (`flex: 1`) — the table as a pane. Header: "12 401 after filters ·
    318 selected", raw-column count, a "CSV" export tag. Columns are
    `Catalogue3DTableModel`'s fixed set **in its exact order and with its exact header
    strings** — Name, Type, RA, Dec, Distance, X, Y, Z — followed by the schema's raw
    columns (only "FLUX" shown in the mock). Numeric columns right-aligned, Mono 11 px.
    Selected row in `kPrimary()` on `rgba(4,138,191,0.10)`. Scene selection and table
    selection are one selection model.
- **Right panel — tabs become `Display / Filters / Source`**, i.e. the existing rail
  pages (Visualization / Filters / Source Info) moved into the inspector so a catalogue
  and a cube share one layout. Content is unchanged; sections are collapsible
  (▼/▶ + 10 px / 700 0.12em caption), fields use a 78 px label column:
  - **SOURCE REPRESENTATION** — "Shape" combo (Ellipsoid / Sphere / Point / Cross),
    "Show labels" (`chkLabels`), "Morphology colors" (`chkMorphColors`).
  - **AXIS MAPPING** — X / Y / Z combos, default "Computed Cartesian X/Y/Z"
    (`computed:x|y|z`).
  - **VISUAL ENCODING** — "Size" (Fixed / Major axis / LLS / Flux), "Color"
    (Morphology color / Speed |v| / raw columns), "Color scale" (`LutScale::modeLabels()`),
    "Power γ" (0.1–10.0, step 0.1, 2 decimals, default 2.00 — **enabled only in Power
    mode**), "Scale" slider (`sliderScale`), and "Render budget"
    (500k / 1M / 2M / 5M / Unlimited, default **1M**).
  - **SCENE** — "Show axes" (checked), "Show bounding box", "Show redshift shells",
    "Frame" combo (FK5 / J2000 · Galactic (l, b)).
- **Second addition, and the important one: the render budget is surfaced in the status
  rail** in `kWarning()` — "drawing 12 401 · budget 1M" — and the budget combo itself
  gets a `kWarning()` border while a subsample is being shown. A researcher must never
  mistake a subsampled scene for the full catalogue. The existing tooltip already says
  picking and stats use every source; repeat that as help text under the combo.
- **Status rail** — health · "184 217 sources · 12 401 after filters" · the budget
  warning · session.

### Screen O — `6b` VLKB Inventory

Reference 900 × 640. From `VLKBInventoryTree`.

- **Header** — title "VLKB Inventory" Lato 15 px / 700 and the region caption
  ("Region: " + `posCutout`) in Mono 10 px `kOnSurfaceVariant()`. Note the current code
  hard-codes `#607080` / `#8090a0` for this caption and the legend — **replace both with
  `kOnSurfaceVariant()`** so they follow the theme.
- **Filter row** — the filter field keeps its placeholder verbatim: "Filter datasets
  (survey, band, line…)". Next to it a 3-segment type filter All / Continuum /
  Spectroscopy.
- **Tree, real hierarchy preserved** — top level `Continuum` and `Spectroscopy`
  (`QStandardItem`s named exactly that), then the five `WavelengthGroup`s in their
  source order with their real boundaries shown as a Mono 10 px caption:
  Visible (0 – 1 µm) · Near-IR / Mid-IR (1 – 25 µm) · Far-IR (25 – 500 µm) ·
  Submm (500 µm – 3 mm) · Radio (> 3 mm).
  Group rows: background `kSurfaceContainerHigh()` for the two top-level rows, plain for
  wavelength groups; children indented 50 px.
- **The change: the VOTable fields the parser already reads become columns instead of
  tooltip HTML.** Header row Lato 10 px / 700 0.10em, grid
  `1.9fr 1.5fr 1fr 0.9fr 90px`:
  | Column | VOTable field |
  |---|---|
  | SURVEY / DATASET | `obs_collection` |
  | TITLE | `obs_title` |
  | WAVELENGTH | derived from `em_min` / `em_max` |
  | OVERLAP | `overlapSky` |
  | TYPE | `dataproduct_type` → IMAGE / CUBE badge |
  Overlap below 100 % renders in `kWarning()` — partial coverage becomes visible
  *before* a 400 MB download. CUBE badges use `kCache()` to distinguish them from images.
- **Details strip** (above the buttons) — KV rows Publisher DID
  (`obs_publisher_did`), Reference (`bib_reference`) and Sky overlap. The first two are
  the citation a paper needs and are currently only in a tooltip.
- **Footer** — the hint "Double-click a dataset, or:" verbatim, then **Add as layer**
  (secondary — routes through `addLayerImage()` → `loadImageLayer()`) and
  **Download & Open** (primary, `m_openButton`).

### Screen P — `6c` HiPS viewer

Reference 1200 × 720. From `HiPSWindow`.

Today **two full toolbar strips** stack above the sky — strip 1: "Survey:" URL field,
− / + / ⌂, "Order: —", Grid, Cat; strip 2: "Target:", "Overlay:" URL field, the ⧸⧸
side-by-side toggle — roughly 80 px of chrome above the data.

- **One 40 px strip**: "Survey" + a **named survey picker** (a combo whose free-text
  entry keeps the current placeholder `https://alasky.unistra.fr/DSS/DSSColor`);
  "Target" + the target field (placeholder verbatim: "M31, Cas A, NGC 1952, …");
  spacer; the − / + / ⌂ zoom group; the Grid / Cat toggle group; and an **Overlay chip**
  that absorbs the side-by-side toggle — "⧸⧸ Overlay · 2MASS/J", border
  `rgba(4,138,191,0.40)` when active, its picker keeping the
  `https://alasky.unistra.fr/2MASS/J` placeholder.
- **Viewports** — one or two `HiPSViewportWidget`s separated by a 1 px `kOutline()`
  gap, cameras synced. Each carries its survey name top-left in the same
  translucent overlay tokens (`rgba(0,0,0,0.55)`, 1 px `kOutline()`, radius 5,
  Mono 10 px). Always `#000004`.
- **Status rail** — the four labels move here from the toolbar, which is where every
  other viewer already keeps its readouts: `m_coordsLabel` ("RA … Dec …"),
  `m_fovLabel` ("FOV 1.82°"), `m_orderLabel` ("order 9"), and `m_surveyLabel` with a
  health dot plus the tile-cache count. `m_statusMsgLabel` ("Open a HiPS survey to
  begin." / "Loading…") occupies the same row when there is a message.
  Right-aligned: **"Add as image layer"** in `kPrimary()` — closes the loop into the
  image viewer.

### Screen Q — `7a` Command palette (⌘K)

Reference width 640 px. From `CommandPalette`. The widget is already built and
keyboard-complete; **only its grouping and two affordances change.**

- **Shell, unchanged** — outer radius 10, 1 px `kOutline()`. Input row: padding
  12 px 14 px 12 px 16 px, background `kSurfaceContainerLow()`, 1 px `kOutline()` bottom
  border, "⌕" 16 px `kSecondary()`, the field at 14 px (placeholder verbatim:
  "Search actions, datasets, layers…"), and the "esc" key cap (Mono 10 px on
  `kSurfaceContainerHigh()`, 1 px `kOutline()`, radius 3, padding 2 px 6 px).
- **Group captions** — Lato 9 px / 700, letter-spacing 0.16em, `kSecondary()`,
  padding 8 px 16 px 5 px, with a 1 px `rgba(4,138,191,0.20)` rule between groups.
- **Row** — padding 8 px 16 px, an 18 px centred glyph column, then a two-line block:
  title Lato 13 px and subtitle Lato 11 px `kOnSurfaceVariant()` (ellipsised), then a
  right-aligned Mono 10 px shortcut. Selected row: background
  `rgba(4,138,191,0.14)` + a 2 px `kPrimary()` left border; reserve those 2 px as a
  transparent border on every row so the glyph column does not shift.
- **Change 1 — regroup to mirror the menubar.** Today the groups are
  File · Application · View · Science · SAMP · Recent datasets, which matches no menu.
  New order: **Analyse · Regions · Export · View · Data · Recent datasets ·
  Products in this session · Application**. The subtitle echoes the menu path
  ("Analyse › Spectral · FWHM / equivalent-width per pixel") so the palette teaches the
  menu instead of competing with it. SAMP actions move into **Application** — they are
  plumbing, and they currently sit at the same level as Line-Width Map.
- **Change 2 — "Products in this session"**, fed by the `ProductRegistry`: one row per
  product, glyph "◈" coloured by state, subtitle "from <dataset> · <params>". Typing a
  product's name is the fastest way back to it.
- **Change 3 — disabled actions stay visible** at `opacity: 0.5` with the reason in the
  subtitle ("open a dataset first") and "unavailable" where the shortcut would be. Today
  a disabled action is listed and silently no-ops when chosen (`QAction::trigger()` on a
  disabled action), which reads as a broken palette.
- **Footer** — padding 9 px 16 px, `kSurfaceContainerLow()`, 1 px `kOutline()` top
  border, Mono 10 px `kSecondary()`: "↑↓ navigate  ↵ run  ⇥ filter by group" and a
  right-aligned match count ("18 of 46 actions").

### Screen R — `7b` Tool dialogs

Three examples of one family: **Channel Maps** (400 px), **Noise region** (400 px),
**LUT customizer** (560 px). `ToolDialogStyle` already defines this chrome and needs
no change — reuse `inputCss()`, `sectionCss()`, `helperCss()`, `makeBreadcrumb()`
and `addSection()` rather than restyling.

- **Shell** — root padding 14 px 16 px, 10 px row spacing (matches the existing
  `QVBoxLayout`).
- **Breadcrumb** — `makeBreadcrumb()`: 10 px / 700, 0.10em, uppercase,
  `kOnSurfaceVariant()`, 6 px bottom padding. **One change: it reads "ANALYSE › …"
  instead of "TOOLS › …"**, following the new menu taxonomy — update the format string
  in `makeBreadcrumb()` (or pass the menu path in).
- **Intro paragraph** — `helperCss()` (11 px `kOnSurfaceVariant()`, 2 px left padding),
  word-wrapped. Keep the existing copy verbatim; it is written in scientific terms and is
  the best thing about these dialogs.
- **Section captions** — `sectionCss()`: 10 px / 700, 0.10em, uppercase, **`kPrimary()`**.
  Channel Maps: "CHANNEL RANGE", "LAYOUT". Noise: "SPATIAL REGION (PIXELS)",
  "SPECTRAL RANGE (CHANNELS)". LUT: "LOOKUP TABLE".
- **Fields** — label column Lato 13 px on the left (96 px in Channel Maps, 82 px in
  Noise, 76 px in the LUT dialog), control on the right: background
  `kSurfaceContainerLow()`, 1 px `kOutline()`, radius 4, padding 5 px 10 px, min-height
  18 px. Spin buttons are a 20 px column split top/bottom with a 1 px `kOutline()` left
  border and the `chevron_up.svg` / `chevron_down.svg` arrows; combo drop-downs a 24 px
  column with `chevron_down.svg`. Numeric values Mono 12 px.
- **Range rows** (Noise) — two right-aligned spin boxes with a "→" in
  `kOnSurfaceVariant()` / 600 between them, one line per axis. Keep this; it halves the
  form height versus separate min/max rows.
- **Restored hint** (Noise) — "↺ Restored from your last session for this cube.",
  italic, `kOnSurfaceVariant()`, shown only when `setRegion()` restored a saved region.
- **Validation** — the existing inline "⚠ start must be ≤ end." in `kError()` via
  `helperCss()`'s geometry, the offending fields get a `kError()` border, **and the
  primary button goes to its disabled style** (`rgba(4,138,191,0.25)` fill,
  `rgba(255,255,255,0.45)` ink — the `makePrimaryButton()` disabled rule). Do not let a
  known-invalid form submit.
- **Change: show what the parameters will produce.** Channel Maps gets a summary line
  ("20 panels · 7 × 3 grid · one shared scale from 0.00 to 8.12 Jy/beam") in the existing
  `m_summaryLabel`; the noise dialog gets a **region preview** — a 74 px dark strip with
  the selected extent drawn as a `kPrimary()` rectangle over
  `rgba(4,138,191,0.12)`, plus the voxel count in Mono 10 px.
- **LUT customizer specifics** — the histogram (`QCustomPlot`) is the primary element:
  a card with caption "PIXEL HISTOGRAM" and the hint "drag the guides to set min / max",
  the plot at 130 px, the two reference lines as 1 px `kPrimary()` verticals with
  "min" / "max" tags, and the active colormap as a 12 px gradient strip directly beneath
  so the mapping is visible. Fields keep the `.ui` labels verbatim — **"Min:", "Max:",
  "Colormap:", "Scale:"** — with the `RESET.png` tool buttons as 28 px squares
  ("Reset min to data minimum" / "…maximum"). "Scale:" is the existing
  `SegmentedToggle` over `LutScale::modeLabels()` (Linear / Log / Sqrt / Square /
  Power); **"Power γ" and its spin box appear only in Power mode** (0.1–10.0, step 0.1,
  2 decimals, default 2.00) with the tooltip as help text: "Exponent γ for the Power
  stretch (t^γ)." Footer note states the scope ("applies to the 2D slice only") —
  the three LUT editors are independent and users conflate them.
- **Footer** — a 1 px `kOutline()` divider, then `makeSecondaryButton()` +
  `makePrimaryButton()` right-aligned. Labels verbatim: Close / **Generate**,
  Cancel / **Estimate**, Close / **Apply**.

### Screen S — `7c` About

Reference width 460 px, padding 26 px 24 px 20 px. From `AboutDialog`.

Today: the 512 px logo and one "<name> v<version>" line. Given that the product is a
Qt/VTK client talking to an independently versioned Python backend, the one thing this
dialog owes a researcher is **what is actually running**.

- Logo centred at 64 px via `createVisivoLogoLabel(64)` (not the raw
  `VisIVO_512.png` pixmap — it does not follow the theme).
- Title "VisIVO Visual Analytics v2.4.1" Lato 15 px / 700, centred — the existing
  `applicationName()` + `applicationVersion()` string.
- One-sentence description, Lato 12 px `kOnSurfaceVariant()`, centred, line-height 1.6.
- A 1 px `kOutline()` rule, then KV rows (11 px key `kSecondary()` / Mono 11 px value):
  **Client** (Qt + VTK versions), **Backend** (FastAPI / astropy / numpy, from
  `/v1/health`), **Build** (branch + date, from `Version.h`), **License**.
- A second rule, then the credits line moved here from the Data Hub footer:
  "VisIVO Visual Analytics, crafted with ❤ by VisIVOLab team — INAF", with the team as
  a link.
- Footer: "Copy build info" and "Documentation" secondary buttons, then a primary
  **Close**. "Copy build info" copies the KV block as text — it makes a bug report one
  click instead of a scavenger hunt.

### Screen T — `7d` VBT and velocity-field inspectors

Both 320 px, the D.4 inspector panel. Both viewers already use `SidebarPanel`, so
moving them into the shell is the same operation as the cube viewer.

**T.1 VBT** (`vtkWindowVbt`) — keeps its three rail pages as tabs:
**Display / Filters / Info**. Sections, with the existing labels:
- **RENDERING** — "Mode" combo (`comboRenderMode`), "Point size" slider
  (`sliderPointSize`), "Ray intensity" slider.
- **FIELD MAPPING** — "X" / "Y" / "Z" / "Scalar" combos (`comboXField`, `comboYField`,
  `comboZField`, `comboColorField` — whose first entry is "Solid color" with an empty
  userData).
- **COLOR** — "Colormap", "Range min" / "Range max" (shown as one two-field row),
  and the **Autoscale** / **Reset View** buttons side by side.
- **SCENE** — "Show color bar", "Show bounding box", "Show orientation axes"
  checkboxes and the "Background" combo.
- The Info tab keeps its four existing cards verbatim: *Dataset Summary*
  (File / Rows / Fields / Kind / Mode / Color field), *Mapping* (X / Y / Z / Scalar),
  *Data Range / Statistics* (Data min / Data max / Mean / Display min / Display max),
  *Available Fields*.
- The Filters tab keeps *Active filters* + the Field / Operator / Value editor and the
  Add… / Remove / Clear All / Apply buttons, plus "Load more…".

**T.2 Velocity field** (`vtkWindowVectorField`) — **has no rail today**: one
`QFormLayout` with 20-plus rows in a single side panel. It gains three tabs matching how
that form is already grouped internally: **Field / Structure / Overlay**.
- **REPRESENTATION** — "Arrows" (`chkGlyphs`), "Streamlines" (`chkStreamlines`),
  "Max arrows", "Arrow scale", "Seeds", "Box size (Mpc)", "Colormap".
- **COSMIC STRUCTURE** — "Show basins", "Basin type"
  (**Attraction (superclusters)** / **Repulsion (voids)** — verbatim),
  "Overdensity (−∇·v)", "Void (+∇·v)", "∇·v level %", "Define ROI",
  then the **Load full-res ROI** and **Field bulk flow vs radius…** buttons.
- **GALAXY OVERLAY** — "Show galaxies" (`chkShowCatalogue`), "Galaxy size",
  "Galaxy color" (Uniform), "Galaxy shape" (Points), "Galaxy density",
  "Density level %", then **Bulk flow vs radius…** and **Remove overlay**.
- **Every physics label is verbatim** — `Overdensity (−∇·v)`, `Void (+∇·v)`,
  `∇·v level %`, `Attraction (superclusters)`, `Repulsion (voids)`. Do not paraphrase.
- Two additions: **units move into the field** ("500 Mpc", "68%") instead of the label,
  which frees the 84 px label column; and the galaxy overlay shows its **source count**
  ("31 402") — the one number that confirms the overlay actually loaded.

### Screen U — `8a` Cube tool forms (the remaining eight)

All 320 px (Kinematic model 340 px) inspector forms, same field vocabulary as F
(`3b`). **Design rule that applies to all of them:** the tools that today ask a
question through `QInputDialog::getItem` / `getText` *after* the user has already
committed — mask mode, region format, output basename — become **fields inside the form**.
A modal question posed after Run is the worst pattern left in the app: the parameters you
chose are hidden behind it.

**U.1 Mask 3-D region** (`vtkWindowCube.cpp` ~3790–3840, `/v1/cube/subregion`).
Mask mode as two radio rows, labels from the existing `QStringList modes`:
"Isolate box — blank everything OUTSIDE" (`blank_outside`, default) and
"Remove box — blank everything INSIDE" (`blank_inside`). Box as three X/Y/Z
start→end rows (22 px axis label, two right-aligned fields, "→" between). A
"Use current region" link plus the resulting extent in Mono 10 px. "Save as" field
with the generated default name and the note "Goes to Workspace Exports. Existing names
get a _1, _2 suffix." A **box preview** strip (66 px, `kPrimary()` rectangle over
`rgba(4,138,191,0.12)`) — the code already draws this in the 3D volume via
`setNoiseRegionPreview()`, but only *after* Run; show it while editing.
Footnote: "One mask at a time — the action is disabled while a mask is computing."
(`m_actMaskRegion->setEnabled(false)`.)

**U.2 Export sub-cube as FITS** (`vtkWindowCube.cpp` ~3616–3748).
Intro verbatim: "Crop the current cube to a spatial + spectral ROI and save it as a
standalone FITS file. WCS is preserved (CRPIX shifted) so coordinates of every pixel
match the source." Three range rows with the **exact labels** "X (columns)",
"Y (rows)", "Z (channels)". "Save in Workspace Exports as" field, default
`<base>_<x0>-<x1>_<y0>-<y1>_<z0>-<z1>`, help text ".fits is appended if missing.
Existing names get a _1, _2 suffix." **New:** an "Output size" readout row
("1024 × 1024 × 77 · 311 MB") — a sub-cube export can be hundreds of MB and nothing
says so today.

**U.3 Export movie** (`vtkWindowCube_Movie.cpp` ~81–130).
"Mode" as a 2-segment control, labels shortened from the combo entries
"Channel scan (2D channel maps)" / "Camera orbit (3D view)" with the qualifier as help
text. Then "First channel" / "Last channel" (1–nz), "Orbit frames (360°)" (8–1440,
default 120) and "Frames per second" (1–60). **Keep the existing enable logic**
(`syncRows()`): channel fields disabled in orbit mode, orbit frames disabled in scan
mode — render the disabled group at `opacity: 0.45`. **New:** a "Result" row
("112 frames · 7.5 s"). The ffmpeg note keeps both verbatim strings and becomes a
status block — `kSuccess()` for "Output: MP4 (H.264) via ffmpeg." and `kWarning()` for
"ffmpeg not found — frames will be saved as a PNG sequence." Exactly one shows at
runtime. Primary button label "Export…".

**U.4 Tilted-ring kinematic model** (`vtkWindowCube_Kinematic.cpp` ~225–300, ~620–700).
Three `sectionCss()` groups. **GEOMETRY**: "Centre X (x0)", "Centre Y (y0)",
"Position angle (PA)", "Inclination", "Disk radius (rmax)". **KINEMATICS**:
"Systemic velocity (Vsys)", "Rotation velocity (Vrot)", "Velocity dispersion (σ)".
All labels verbatim; units live in the field. Then the checkbox
"Per-ring profile (warp / rotation curve)" and, when checked, a 4-column ring table
(R px / PA ° / INCL ° / VROT, Mono 10 px, right-aligned) with **Add ring** /
**Remove selected**. **APPEARANCE**: "Contour level (% peak)" (5–95),
"Opacity (%)" (5–100), "Representation" combo, "Colour" (a 26 × 22 swatch opening the
colour dialog), "Contour on 2D channel maps", "Show overlay". Footer:
**Apply model** primary + **Remove** secondary (the existing "Remove overlay").

**U.5 Kinematic lasso** (`vtkWindowCube_Lasso.cpp` ~450–545).
A `kPrimary()`-tinted state card at the top ("✛ Click to select in 3D view") carrying
the live count from `m_lassoCountLabel` as a KV row. "Reach" slider (1–40) and
"Velocity coupling" slider (10–500), each with the value above and the range as Mono
10 px help ("higher follows steeper velocity gradients"). A **Refine** group:
"Refine on 2-D map…" then an **Add / Subtract** 2-segment control. An **Apply** group,
stacked so the destructive pair reads clearly: **Isolate → new cube** (primary),
**Remove → new cube** (secondary), **Clear selection** (ghost). Labels verbatim.

**U.6 Region file** (`vtkWindowCube_RegionIO.cpp` ~99–170).
An Export / Import 2-segment control at the top. Format as four radio rows with the
**exact `QStringList choices`** — "DS9 — sky (FK5)", "DS9 — pixel",
"CASA CRTF — sky (FK5)", "CASA CRTF — pixel" — each with its extension
(`.reg` / `.crtf`) right-aligned, so the mapping from choice to file type is visible.
A "Current region" readout row. Primary **Save file…**; footnote
"Import accepts *.crtf and *.reg."

**U.7 Send slice to image viewer** (`vtkWindowCube.cpp` ~4146).
Explanatory line, a "Channel" readout ("57 · −124.6 km/s"), the
"Save in Workspace Exports as" field with the generated default, a checkbox
"Open in image viewer when ready", and a primary **Send**.

**U.8 Catalogue overlay** — collapses **four** Tools actions (Load Catalogue Overlay,
Show Catalogue Overlay, Show Catalogue Labels, Clear Catalogue Overlay) into one block:
a dashed "Load catalogue…" row, then a loaded-file row (kind badge, name, source count,
"✕" = Clear) and two checkboxes, "Show overlay" and "Show labels".

### Screen V — `8b` Pixel histogram pane

Reference 620 × 420. From `showSliceHistogramPanel()`; **one pane serves both viewers**
(the cube's "Pixel Histogram (current slice)…" and the image viewer's
"Pixel Histogram…").

- Header: title, Mono 10 px context ("CH 57 · −124.6 km/s"), a Linear / Log
  2-segment toggle, the "PNG" export tag, "⤢".
- Plot: `QCustomPlot`, Y gutter 48 px with decade labels, X tick labels and an
  axis label in `kPrimary()` ("Pixel value [Jy/beam]").
- **Axis geometry rule for every plot pane** (PV, spectrum, histogram): the Y tick
  gutter and the plot form **their own row**, and the X tick row plus axis label sit in a
  second row below it, offset by the gutter width. If the gutter is a flex sibling of a
  column that also holds the X axis, it stretches to that column's height and the bottom
  Y tick drifts onto the X-axis baseline — a mislabelled axis. In Qt this is what
  `QCustomPlot` already does natively, so simply do not add hand-built gutters around
  it; the rule matters only when reproducing the layout in HTML/CSS.
- **New — the LUT guides**: the active LUT min and max drawn as 1 px `kPrimary()`
  verticals with "LUT min" / "LUT max" tags, so the histogram and the colour scale are
  visibly the same axis. And a `kWarning()` vertical at **3σ** when a noise estimate
  exists for the dataset.
- Status line: min / max / mean / rms in Mono 10 px, and a right-aligned
  **"Set LUT from selection"** in `kPrimary()` — dragging a range on the plot writes it
  into the LUT. That round trip is the reason a researcher opens a histogram at all, and
  it does not exist today.

### Screen W — `9a` Image-viewer tool forms (the remaining eight)

Inspector forms, 320 px (Polarisation 340, Faraday RM 400). Same rule as `8a`: the
`QInputDialog` prompts that fire *after* a file is picked — contour levels, annotation
text — become fields in the form.

**W.1 Contours** (`vtkWindowImage_Contours.cpp` ~205–230). Replaces three Tools actions
(Show Contours, Load Contour from FITS…, Clear All Contours).
"Source" as a 2-segment control (A layer / External FITS) then the layer combo.
"Levels" gets a **new 2-segment Absolute / × σ** mode: in σ mode the values multiply the
existing noise estimate, which is what radio astronomers actually want and currently
have to compute by hand. The field keeps the verbatim default
`0.1, 0.5, 1.0, 2.0, 5.0` and the "Comma-separated" help. Then per-set "Colour" swatch
and "Line width". Below, a list of **active contour sets** — checkbox, name + level
count, a 12 × 3 colour swatch, "✕" to remove — so several overlaid sets are
distinguishable. Footer: **Add contour set** primary + **Clear all**.

**W.2 Cross-match with catalogue** (`vtkWindowImage_Catalogue.cpp` ~478–490).
"Catalogue" combo, then "Search radius" as a slider with the value above and the real
range as help: **1.0 – 3600.0 arcsec**, annotated with the beam size so the choice has a
physical reference. **New:** an "EXPECTED" block — overlay source count and candidates
within the radius — before you commit.

**W.3 Measure** (`vtkWindowImage_Measurement.cpp`). Distance / Angle (3 pts) as a
2-segment mode control (bound to the centre mode strip from `3c`). Active measurements
as rows: glyph, primary value, secondary in Mono 10 px, "✕". "Copy values" and
"Clear all" replace the standalone Clear Measurement action.

**W.4 Annotations** (`vtkWindowImage_Annotations.cpp` ~158–184). The text field comes
**before** placement, not as a modal after: type the text, press **Place text** or
**Place arrow**, then click on the image — the existing two-step flow with the prompt
moved into the panel. Help text: "Then click on the image to place it. Arrow label is
optional." Save / Load / Clear as one row.

**W.5 Polarisation** (`vtkWindowImage_Polarisation.cpp`). **The biggest consolidation in
the redesign — nine Tools actions become one panel.**
- **STOKES LAYERS** — four rows (I / Q / U / V) with the layer name and a state dot;
  a missing companion shows a dashed row with "not found" and a "Locate…" link. Help
  text names the detection rule verbatim: auto-detected by replacing `StokesI` in the
  basename (`findStokesCompanionPath`).
- **DERIVED PRODUCTS** — chips using the exact `stokesRoleLabel()` strings:
  "Pol. intensity (P)", "Pol. angle (PA)", "P (debiased)", "Frac. pol. (P/I)".
  A chip is **enabled only when its inputs exist** — P/PA/debiased need Q+U, Frac. pol.
  needs I+Q+U. This replaces four `QMessageBox::warning` dead ends
  ("Need both Stokes Q and Stokes U layers loaded.").
- **VECTORS** — "Show polarisation vectors" plus the three parameters that are
  currently only member variables with no UI: "Grid step" (`m_polVecGridStep`),
  "Length scale" (`m_polVecScale`), "Min SNR" (`m_polVecSnrMin`), and a "Drawn"
  readout carrying the count and σ that the code already puts in a status message.

**W.6 Compute spectral index** (`vtkWindowImage_Polarisation.cpp` ~486–510).
Four fields with the **verbatim labels including their colons** — "Layer A:",
"Frequency A (GHz):", "Layer B:", "Frequency B (GHz):". Note in the intro that the result
registers as "Spectral index (α)" on the **Spectrum** colour map scaled −2.5 … 1.5
(the values in `addDerivedLayer`). **New:** a "Frequency ratio" readout.

**W.7 Compute Faraday rotation measure** (`vtkWindowImage_Polarisation.cpp` ~596–640).
Intro **verbatim** — it carries the method and its limit. Table with the exact header
labels "Stokes Q layer", "Stokes U layer", "ν (GHz)" (default `1.284`, validator
1e-9…1e9 with 6 decimals), plus **Add row** / **Remove row**. **New:** a row count with
the **λ² span** in `kSuccess()`, and the "no unwrapping" caveat promoted to a
`kWarning()` block — a silently ambiguous fit is worse than a refused one.

**W.8 Export to workspace as FITS**. "Layer" combo including derived layers (kind badge
per role), the basename field with the same ".fits is appended…" help as the cube
exports, and a size + WCS provenance readout ("inherited from Layer A") — a derived
layer's WCS origin is otherwise invisible.

### Screen X — `10a` Science-menu tools

Four 340 px inspector forms. All four are currently bare inline `QFormLayout` dialogs in
`MainWindow_Science.cpp` with no `ToolDialogStyle` and no explanation of any option.
**Keep every option string verbatim** — they are the backend API's own vocabulary.

Real endpoints, from `BackendClient_Products.cpp`: `/v1/products/publication_figure`,
`/v1/products/image_quality`, `/v1/products/map_math`, `/v1/products/mosaic`. (The same
file also holds `/v1/photometry/aperture/{id}`, `/v1/photometry/gaussfit/{id}`,
`/v1/astrometry/reproject/{id}` and `/v1/astrometry/crossmatch/{id}` — the last is what
`9a`'s cross-match form calls.)

**X.1 Publication figure** (~81–140). "Stretch" as a 5-segment control with the exact
values `linear / log / sqrt / asinh / power`; "Colormap" combo with the exact list
`inferno / viridis / magma / gray / hot / cubehelix` and a gradient swatch; "Title" text;
the two checkboxes with their verbatim labels **"WCS sky grid"** and
**"Colorbar (BUNIT)"**, both checked by default. **New:** "Resolution" exposed as a field
— the call passes a hard-coded `200` dpi today. A preview strip. Primary button
**Render** (the existing OK text). Footnote carries the real caveat verbatim: the figure
is written on the backend and opening it locally only works when the backend shares this
filesystem.

**X.2 Image quality / artifacts** (~142–195). No input parameters. The twelve metrics
land in a read-only `QPlainTextEdit` as space-aligned text today; group them, keeping the
**wire key names** as the row labels so they match the JSON:
- *COVERAGE* — `total_pixels`, `blanked_fraction`, `zero_fraction`, `negative_fraction`
- *STATISTICS* — `min`, `max`, `mean`, `median`, `robust_std`
- *ARTIFACTS* — `dynamic_range`, `striping_metric`, `negative_bowl`
Out-of-range values render in `kWarning()`. The `flags` array is promoted to a
`kWarning()` block; when empty it keeps the existing string "(none — looks clean)".
Footer: Re-run / Copy report.

**X.3 Pixel math / spectral index** (~196–295). "Operation" as a wrapping chip set with
the **nine exact values** `ratio, divide, subtract, add, multiply, spectral_index, log10,
abs, square`. Then "Map A" / "Map B" combos over the session datasets, and "Freq A" /
"Freq B" in **Hz** (the placeholders are "freq A (Hz)"). **Keep the existing enable
logic**: `log10 / abs / square` are unary → Map B disabled; only `spectral_index`
enables the frequency fields. State both rules as help text. **New:** an "Expression"
readout showing what will be evaluated (e.g. `α = ln(A/B) / ln(νA/νB)`).

**X.4 Mosaic (noise-weighted)** (~296–340). Intro verbatim: "Select two or more images to
mosaic." Multi-select list of session datasets with each image's noise σ. **New:** images
that do not overlap the selection on the sky are flagged in `kWarning()` with a
"no overlap" tag and a warning line, and an "OUTPUT GRID" block reports extent, pixel
dimensions and size — a mosaic is the most expensive product in the app and today you
learn its size after it exists.

### Screen Y — `10b` FITS header viewer

Reference 620 × 480 — the size the dialog already uses. From
`vtkWindowCube::showFitsHeaderDialog()` (and the identical one in
`vtkWindowImage_Setup.cpp` ~459). Opened by clicking the primary tag chip in the viewer
toolbar. Stays **non-modal** (`Qt::NonModal`, as today) so it can sit open beside the data.

- Header card: background `kSurfaceContainerLow()`, 1 px `kOutline()` bottom border,
  padding 12 px 16 px, 2 px row gap — "FITS Header" Lato 14 px / 700 over the full path
  in Mono 11 px `kOnSurfaceVariant()`, selectable. Exactly as built today.
- Body: read-only, **no wrapping**, Mono 11 px, line-height 1.85, padding 12 px 16 px,
  and it **scrolls vertically** — a real header is 100–200 cards, so the body is a
  scrolling `QPlainTextEdit` and the mock shows a visible scrollbar plus a
  "… N more cards" marker at the end of the flow. Do not clip it silently.
- **Two additions:** a **keyword filter** field (148 cards is a normal header; finding
  `BUNIT` by scrolling is absurd) with a card count, and **Copy all**.

### Screen Z — `10c` Remote GPU rendering

**Z.1 The pane** (660 × 440). `RemoteRenderWidget` inside the standard pane frame.
- Header: a `kRunning()` state dot, title, "remote GPU" qualifier in `kRunning()`, then
  the **Auto / Local / Remote** segmented control — the actions
  `m_actionRenderAuto/Local/Remote`, currently buried in View › Rendering, moved to where
  the mode applies. Then the LUT strip and "⤢".
- Viewport `#000004`. **Telemetry overlay** top-left (same translucent tokens as
  `ChannelOverlay`): a `kRunning()` dot, the frame rate from the widget's existing
  `framesPerSecondChanged` EMA, plus latency and bitrate. The widget computes fps today
  and nothing displays it.
- The channel scrubber is unchanged — the same `ChannelOverlay`.
- Status line: stream state, stream resolution, "interaction LOD on drag" (the widget
  already drops resolution during drags), and a right-aligned "Switch to Local".

**Z.2 The offer** (440 px). The banner already exists in
`vtkWindowCube::setupWorkspaceDocks()` with the right amber gradient
(`rgba(245,158,11,0.22)` → `rgba(217,119,6,0.16)`, bottom border
`rgba(217,119,6,0.40)`) and the "Switch to Remote" button. Keep the title verbatim:
**"Large cube — remote rendering recommended"**.
- **What it lacks is a reason.** Add the cube's dimensions and size as a subtitle, and a
  three-row comparison: what the user has now ("preview 256³ · 11 fps" in `kWarning()`),
  what remote gives ("full-res · ~58 fps" in `kRunning()`), and the node. That turns a
  nag into a decision.
- Dismissible ("✕"), and the footnote points at the pane-header control as the permanent
  way to change mode.

## Interactions & Behavior

- **Selecting a product** in the session tree loads it into the focused pane and switches
  the inspector to its Properties tab. Double-click opens it in a new pane.
- **Selecting a task chip** swaps the form below it. Form state persists per
  (task, dataset) via the existing `AnalysisParamStore`.
- **Run on backend** creates the task (`createMomentTask` / `createPvTask` / the spectral
  endpoints), immediately inserts a product row in the tree with state *running*, and
  starts a status poll. Never block the UI thread — keep the
  `QtConcurrent::run` + `QFutureWatcher` pattern.
- **Poll cadence**: `requestTaskStatus(taskId)` every 750 ms while running; on completion
  the row flips to *ready* and the result loads into the pane it was launched from.
- **Errors**: the row turns `kError()` and shows the backend's real message on hover —
  `MomentMapController::lastError()` already carries it.
- **Linked panes**: camera, channel and LUT sync are three independent toggles; changing a
  linked property in one pane propagates to all panes bound to the same dataset.
- **Sidebar collapse**: keep `SidebarPanel`'s 240 ms `InOutCubic` width animation and the
  ⌘\\ shortcut.
- **Theme switch**: already handled globally by `ThemeManager`; note that widgets built
  with inline stylesheets do not restyle live — they are rebuilt on theme change.
- **Task failure**: the form keeps its values, the error card appears above it, and the
  offending field gets a `kError()` border. Retry re-submits the same body.
- **"Save as product"** (region stats, measurements) inserts a `Product` with
  `kind = "region_stats"` and the geometry in `params` — no backend call.
- **Mode strip** (image viewer): exactly one of Pan / Region / Ruler / Annotate is
  active at all times; the active mode determines what a click in the viewport does and
  must be visible without opening a menu.
- **Layer drag-reorder** in the layer stack changes z-order; promoting a layer to a
  contour overlay swaps its second row for the contour-level summary.
- **Empty state**: accept a real file drop on the centre panel.
- **Result panes**: opening a product from the session tree loads it into the focused
  pane. The header export tag ("PNG" / "FITS") triggers the existing save path
  (`QCustomPlot::savePng()`, `saveMosaicPng()`, the FITS export actions).
- **Channel-map tile ↔ viewer channel** stay in sync both ways: the highlighted tile
  follows `ChannelOverlay`, and clicking a tile moves the viewer's channel.
- **Failed job row** expands inline on click; "Open in inspector" restores the task form
  with the failed parameters (from `AnalysisParamStore`) and switches the right panel to
  the Analysis tab.
- **Plot theming** stays in `applyPlotTheme(bool dark)`, already bound to `Settings.ini`
  `Plots/theme`. Note the plot canvas *does* follow the theme, unlike the VTK viewports.

## State Management

New model — **`ProductRegistry`** (one per session, owned by `MainWindow`):

```
struct Product {
  QString  productId;
  QString  parentDatasetId;   // dataset it was derived from
  QString  kind;              // moment | pv | linewidth | baseline | stack | sources | …
  QString  label;             // "Moment 0 — integrated"
  QVariantMap params;         // serialisable recipe (feeds Provenance + JSON export)
  QString  taskId;            // backend task, empty when computed locally
  enum State { Queued, Running, Ready, Failed } state;
  double   progress;          // 0..1
  QString  error;
};
```

Signals: `productAdded`, `productUpdated`, `productRemoved`. The session tree is a
`QAbstractItemModel` over datasets (from `listSessionDatasets()`) with products as
children. Every existing controller gains one `registry->add(...)` call — no change to
its computation path.

Existing state that stays as-is: `AnalysisParamStore`, `RecentDatasetsManager`,
`ThumbnailCache`, `Settings`, session id in `BackendClient`.

## Design Tokens

All already defined in `src/gui/theme/VisivoTheme.h` — use the accessors, not literals.

| Token | Dark | Light |
|---|---|---|
| `kBackground()` / `kSurface()` | #090F14 | #FFFFFF |
| `kSurfaceContainerLow()` | #11181F | #FAFBFD |
| `kSurfaceContainerHigh()` | #192229 | #F1F4F9 |
| `kSurfaceContainerHighest()` | #242E37 | #F4F6FA |
| `kOnSurface()` | #F1F4F6 | #1C2734 |
| `kOnSurfaceVariant()` | #9FA6AC | #5B6A7E |
| `kSecondary()` | #6B737A | #8A99AE |
| `kOutline()` | #2B343C | #D8DEE7 |
| `kPrimary()` | #048ABF | #048ABF |
| `kPrimaryDim()` | #005574 | #007CA5 |
| `kPrimaryFixed()` | #5BB8D8 | #3AAECF |
| `kSuccess()` | #57BC80 | #14874E |
| `kWarning()` | #E8AA4E | #B77610 |
| `kError()` | #D67069 | #BD413F |
| `kRunning()` | #17D0D8 | #0081A5 |
| `kCache()` | #A39FC8 | #716B98 |

Brand-blue alpha overlays used throughout (theme-invariant):
`rgba(4,138,191,0.14)` card border · `rgba(4,138,191,0.18)` panel divider ·
`rgba(4,138,191,0.20)` section rule / checked fill · `rgba(4,138,191,0.22)` chrome border ·
`rgba(4,138,191,0.32)` input border · `rgba(4,138,191,0.40)` checked border.

**Typography**
- Brand / UI: **Lato** — bundled at `resources/fonts`, registered by
  `VisivoTheme::loadBundledFonts()`. Fallback `"Helvetica Neue", Arial, sans-serif`.
- Data / numeric: **JetBrains Mono**, fallback `"SF Mono", Menlo, Consolas, monospace`.
- Scale: 9 px badge · 10 px section label (700, 0.14em tracking) · 11 px control /
  Mono readout · 12 px body · 13 px menu & primary control · 15 px prose.

**Spacing** — 2 / 4 / 6 / 8 / 10 / 12 / 14 / 18 px.
**Radii** — 3 px inner segment · 4 px input & chip · 5 px tree row · 6 px button, card,
overlay · 8 px pane & toolbar · 999 px status pill.
**Fixed dimensions** — command bar 46 · jobs rail 28 · left panel 272 · right panel 320 ·
channel overlay 28 (10 px inset) · sidebar rail 48 (76 with labels) · rail button 36.

## Assets

- Data Hub quick-action glyphs: generated in code by
  `actionCardSvg(kind, stroke)` in `DataHubWidget.cpp` — not files. Four glyphs
  (`dataset`, `catalogue`, `vbt`, fallback) cover the five cards.
- Rail icons: `resources/icons/rail_*.svg` — 24×24 viewBox, `stroke="currentColor"`,
  stroke-width 1.6, round caps/joins. Rendered through
  `VisivoTheme::renderThemeIcon(path, size, colorOverride)`, which substitutes
  `currentColor` for the theme hex. Two of them (`rail_contours.svg`,
  `rail_threshold.svg`) still hard-code `#8AB8CC` / `#048ABF` — **fix them to
  `currentColor`** so they follow the theme like the rest.
- Logo: **always go through `VisivoTheme::createVisivoLogoLabel(height, parent)`** —
  never a fixed file, and never a rail icon. It picks
  `:/icons/visivo_logo_white.svg` in dark mode and `:/icons/VisIVO.svg` (coloured) in
  light, scales to the requested height preserving the 837 × 505 aspect ratio
  (44 px tall → ~73 px wide), and falls back to `:/icons/VisIVO_128.png`.
  Both variants are in this bundle under `icons/`. Used at 44 px in the Data Hub header
  (`DataHubWidget.cpp` ~236) and in both startup states (`StartupDialog.cpp` ~130–138 —
  the `QLabel("⬡")` there is the *fallback inside an if*, not the design).
- `visivoBrandIconPixmap(stroke, w, h)` in `DataHubWidget.cpp` ~98 paints a separate
  code-drawn orbit mark (a rotated ellipse plus an arc). It is **not** the header logo —
  do not swap one for the other.
- Colormap gradients: generated at runtime by
  `CubeUiAssembler::buildLutPreview(name, w, h)` over `ColorMaps`. No static assets.

## Files

- `VisIVO Workspace.dc.html` — all four screens. Open it in a browser.
  - `#1a` cube viewer, current state (reference)
  - `#1b` image viewer, current state with Tools menu open (reference)
  - `#1c` conservative proposal (rejected, taxonomy still normative)
  - `#1d` **the analysis workspace — build this**
  - `#2a` region → source implementation map with effort estimates
  - `#3a` inspector Properties / Analysis-failed / Provenance tabs + empty state
  - `#3b` the five remaining task forms
  - `#3c` image viewer inside the same shell
  - `#4a` the four result panes (PV, spectrum, science map, channel maps)
  - `#4b` the ⌥J jobs panel
  - `#5a` Data Hub, reduced
  - `#5b` startup dialog, running and failed
  - `#5c` remote FITS file browser
  - `#5d` Settings
  - `#6a` 3D catalogue viewer with the source table as a pane
  - `#6b` VLKB inventory
  - `#6c` HiPS viewer
  - `#7a` command palette
  - `#7b` tool dialogs (Channel Maps, noise region, LUT customizer)
  - `#7c` About
  - `#7d` VBT and velocity-field inspectors
  - `#8a` the eight remaining cube tool forms
  - `#8b` pixel histogram pane (shared by both viewers)
  - `#9a` the eight remaining image-viewer tool forms
  - `#10a` the four Science-menu tools
  - `#10b` FITS header viewer
  - `#10c` remote GPU render pane and the offer banner
- `icons/` — the SVGs copied out of the repo and used in the mock.

### Source files this design touches

| Area | Files |
|---|---|
| Theme tokens | `src/gui/theme/VisivoTheme.h/.cpp`, `SegmentedToggle.*` |
| Shell & menus | `src/gui/MainWindow*.cpp`, `ui/MainWindow.ui` |
| Cube viewer | `src/gui/vtkWindowCube*.cpp`, `CubeUiAssembler.*`, `ui/vtkWindowCube.ui` |
| Image viewer | `src/gui/vtkWindowImage*.cpp`, `ui/vtkWindowImage.ui` |
| Sidebar | `src/gui/SidebarPanel.h/.cpp` |
| Overlay | `src/gui/ChannelOverlay.h/.cpp` |
| Dialogs → inspector | `LinewidthDialog`, `BaselineDialog`, `StackDialog`, `SourceFindDialog`, `ChannelMapsDialog`, `NoiseRegionDialog`, `ToolResultDialog` |
| Controllers (unchanged) | `MomentMapController`, `NoiseController`, `PvController`, `ImageLayerController` |
| Backend | `src/app/BackendClient.h`, `BackendClient_Tasks.cpp`, `BackendClient_Spectral.cpp` |
| Status / jobs | `src/gui/DiagnosticsWindow.*`, `src/app/DiagnosticsManager.*` |
| Palette | `src/gui/CommandPalette.h/.cpp` |
| Tool dialogs | `src/gui/theme/ToolDialogStyle.h`, `ChannelMapsDialog.*`, `NoiseRegionDialog.*`, `LUTCustomizerDialog.*` + `ui/LUTCustomizerDialog.ui`, `AboutDialog.*` + `ui/AboutDialog.ui` |
| Science tools | `MainWindow_Science.cpp`, `BackendClient_Products.cpp` |
| Remote rendering | `RemoteRenderWidget.*`, `RemoteRenderWebRTCClient.*`, `vtkWindowCube_RemoteRender.cpp` |
| Other viewers | `vtkWindowVbt.*`, `vtkWindowVbtVolume.*`, `vtkWindowVectorField.*` |
| Result panes | `PvDiagramWidget.*`, `ProfileWidget.*` + `ui/ProfileWidget.ui`, `ScienceMapWindow.*`, `ChannelMapsWindow.*`, `qcustomplot` |
| Catalogues & sky | `vtkWindowCatalogue3D*.cpp` + `ui/vtkWindowCatalogue3D.ui`, `Catalogue3DTableModel.*`, `CatalogueTableModel.*`, `Catalogue3DParser.h`, `VLKBInventoryTree.*`, `HiPSWindow.*`, `HiPSViewportWidget.*` |
| Shell screens | `DataHubWidget.*` + `DataHubWidget_{Recent,Status,Exports}.cpp`, `StartupDialog.*`, `BackendLauncher.*`, `RemoteFileBrowserDialog.*`, `SettingsDialog.*` + `ui/SettingsDialog.ui` |

## Suggested Implementation Order

1. **Jobs / status rail + command bar** — visible, cheap, no scientific code touched.
2. **`ProductRegistry` + session tree** — the structural core. Wire existing controllers
   to register their outputs; results still open in their current windows at this stage.
3. **Pane grid + Linked chip** — three saved `saveState()` layouts, compact dock titles.
4. **Dialogs → inspector Analysis tab** — last, and one task at a time. Start with
   Moment (fully async already), then Line-width, Baseline, Stack, Source find.
5. **Provenance tab + recipe JSON export** — trivial once the registry stores params.
6. **Result panes (`4a`)** — do this *with* stage 4, not after: the moment the task
   forms move into the inspector, their results need somewhere to land. Mechanically it
   is dropping each widget's footer, moving its export action into the pane header, and
   letting it be reparented into a dock.
7. **Jobs panel (`4b`)** — the Log tab is the existing window; only the Jobs tab is new.
8. **Image viewer shell (`3c`)** — after the cube viewer works. The layer-stack cards
   and the mode strip are the only genuinely new widgets; overlays and measurements are
   existing actions re-presented as toggles and rows.
9. **Shell screens (`5a`–`5d`)** — independent of stages 1–8 and safe to do in any gap.
   `5a` is mostly *deletion*, so it is the cheapest visible win in the whole plan.
   `5d` is a container change with no new fields except "Test connection".
10. **Catalogue viewer (`6a`)** — last of the three viewers to move into the shell. Its
    three rail pages become the inspector's three tabs with no content change.
11. **VLKB inventory (`6b`) and HiPS (`6c`)** — self-contained; both are mainly a matter
    of promoting data already parsed (VOTable fields, order/FOV/coords) out of tooltips
    and toolbars into columns and the status rail.
12. **Command palette regroup (`7a`)** — do it right after the menu split in stage 1,
    while the taxonomy is fresh. Pure data change plus two small affordances.
13. **Tool dialogs (`7b`) and About (`7c`)** — `7b` is one string change in
    `makeBreadcrumb()` plus two summary widgets; the rest of the chrome already exists.
14. **VBT and velocity field (`7d`)** — last; the velocity field is the only one that
    needs a new `SidebarPanel` with three pages.
15b. **Image-viewer tool forms (`9a`)** — with stage 6 (the image viewer shell). The
    polarisation panel is the only one that needs real logic beyond relayout: input-driven
    enablement of the four derived-product chips.
15. **Cube tool forms (`8a`) and the histogram pane (`8b`)** — do these with stage 4,
    alongside the five forms in `3b`. Each is mechanically the same job: lift the
    existing `QDialog` body into an inspector page and fold its follow-up
    `QInputDialog` prompts in as fields.

## Deliberate Additions (not present today)

These four are new behaviour, not just relayout. They are cheap and they close real gaps,
but flag them if they are out of scope:

1. **Baseline inline preview** of the central spectrum (raw / fit / residual) — needed to
   judge a fit before committing.
2. **Stack depth-mismatch warning** before the run — today a mismatch surfaces only as a
   backend error.
3. **PV live path editor** with vertex count and path length in the inspector — today the
   path-drawing mode has no visible state or readout.
4. **Image viewer mode strip** (Pan / Region / Ruler / Annotate) — today the active
   interaction mode is invisible.
5. **Derived readouts on the spectrum plots** (peak value + velocity, row + channel) —
   currently the user reads them off the axes.
6. **Current-channel highlight** in the channel-map mosaic, linked both ways to the
   viewer's channel.
7. **Cache rows in the Jobs tab** — makes client-side cache hits visible instead of
   mysterious.
8. **Determinate startup progress** and severity-coloured startup log lines.
9. **Dimensions + WCS status in the file browser's Details pane** — data the
   `/v1/files/header` call already returns.
10. **"Test connection" in Settings** with an inline health result.
11. Two Settings checkboxes for behaviour that exists but has no UI: backend auto-start
    and prefer-remote-rendering.
12. **Morphology legend with per-class counts** in the catalogue viewer.
13. **Render-budget warning in the status rail** when the 3D scene is subsampled.
14. **VLKB columns + citation details** promoted out of tooltip HTML.
15. **HiPS: named survey pickers** instead of raw URL fields, and readouts moved to the
    status rail.
16. **Palette: products group, visible-but-disabled actions with a reason, footer hints.**
17. **Tool dialogs: outcome summaries** (panel count / grid, region preview + voxel count)
    and a disabled primary button on invalid input.
18. **About: client + backend + build versions** and "Copy build info".
19. **Velocity field: units inside fields, galaxy source count.**
20. **No post-Run modal questions.** Mask mode, region format and output basename move
    into their forms; `QInputDialog::getItem`/`getText` disappears from these paths.
21. **Output-size readout** on sub-cube export; **frame/duration readout** on movie export.
22. **Mask box preview while editing**, not only after Run.
23. **Catalogue overlay: four actions collapsed** into one row with two toggles.
24. **Histogram: LUT min/max + 3σ guides and "Set LUT from selection".**
25. **Contours: × σ level mode** and a list of active sets with per-set colour.
26. **Cross-match: expected-match counts** before running; radius annotated with the beam.
27. **Polarisation: one panel** with visible Stokes state and input-driven enablement,
    replacing four `QMessageBox::warning` dead ends; the three vector parameters
    (grid step / scale / min SNR) get UI for the first time.
28. **Faraday RM: λ² span readout** and the no-unwrapping caveat as a warning block.
29. **Export: WCS provenance** on derived layers.
30. **Publication figure: dpi exposed** (hard-coded 200 today).
31. **Image quality: metrics grouped** into Coverage / Statistics / Artifacts with amber
    out-of-range values and `flags` as a warning block.
32. **Pixel math: expression readout**; **Mosaic: sky-overlap check + output grid size.**
33. **FITS header: keyword filter and Copy all.**
34. **Remote rendering: fps/latency/bitrate overlay** (already computed, never shown),
    the Auto/Local/Remote control moved into the pane header, and a *reason* on the
    large-cube banner.

## Still Not Designed

Every screen in the app now has a comp. What remains is small and follows the documented
token set — build it by analogy, but flag it if you are unsure:

- **The filter-editor dialog** behind "Add…" in the catalogue and VBT Filters tabs
  (Field / Operator / Value). Use the `7b` tool-dialog chrome.

**Every tool in both viewers now has a form.** `3b` + `8a` + `9a` + `7b` cover the
complete Tools/Analyse surface of the cube viewer and the image viewer.
- **SAMP status surfaces** beyond the palette entries and the health chip.
- **`vtkWindowVbtVolume`** — assume the VBT inspector (`7d`) plus the cube viewer's
  volume-rendering section from `1d`.
- **Print / publication-figure output itself** (the `Publication Figure…` product), as
  opposed to the dialog that launches it.

## Known Gaps to Confirm With the Team

- Only Moment and PV have `create*Task` async wrappers; the spectral endpoints
  (`/v1/spectral/*`) are called synchronously inside `QtConcurrent`. Inline progress for
  those needs matching task wrappers, or the row shows an indeterminate state.
- Client-side slice-cache occupancy is not currently exposed as a number; the rail's
  cache slot needs a small accessor.
- The design assumes one session per window. If multiple sessions are ever concurrent,
  the dataset switcher needs a session grouping level.
