# Cube viewer — normative layout spec (supersedes `1d` for the cube)

See `#12a` (**the default state**), `#12b` (focus mode), `#12c` (space budget), `#11a`
(all regions expanded, Properties tab) and `#11b` (Analysis + Provenance tabs, plus the
region table) in `VisIVO Workspace.dc.html`.

**Two corrections to my earlier instructions.** They caused the confusion in the stage-3
build, so treat them as the authoritative version:

1. **The rendering controls belong in the LEFT panel, not in Properties.** Moving the four
   old `SidebarPanel` pages into Properties turned it into a scrolling wall of bordered
   cards. Properties holds identity + DATASET + STATISTICS + WCS as key/value rows and
   **must never need a scrollbar**.
2. **The centre needs a toolbar AND one header per pane.** The 1/2/4 + Linked strip does
   not live in the Inspector.

## Default state and collapsing — read this first

`11a` shows every region expanded so the spec can name them. **It is not the default
state.** With two panes and both docks open, chrome eats 592 px of a 1440 px window — 41 %
— and each pane is a 454 × 735 portrait sliver. The default is `12a`:

| | Default (`12a`) | Focus (`12b`) | 2 panes |
|---|---|---|---|
| Pane count | **1** | 1 | on request |
| Left dock | open, 272 | rail 36 | open |
| Right dock | **rail 36** | rail 36 | open |
| Chrome | 308 | 72 | 592 |
| Viewport | **1112 × 735** | 1348 × 735 | ~596 × 735 |

Rules:

1. **`paneCount` starts at 1.** A second segmented control (**3D / 2D**) picks which view
   fills the single pane. `Linked` is disabled and dimmed to 0.4 opacity while
   `paneCount == 1` — there is nothing to link.
2. **The Inspector starts collapsed** to a 36 px rail. It expands on: a click on the rail,
   a click on one of the vertical tab names (which also selects that tab), a chip pressed
   in the Analysis tab, or a task starting. It does **not** auto-collapse.
3. **The left dock starts open.** Threshold, blend mode and LUT are touched every few
   seconds; a click to reach them costs more than the 272 px. It still gets a collapse
   button ("◧") in its header.
4. **A collapsed rail is 36 px, never 0.** It carries: the expand glyph at top
   ("◧" left / "◨" right), a 1 px rule, the panel name in `writing-mode: vertical-rl`
   (Lato 10 px / 700, 0.14em, `kOnSurfaceVariant()`), a spacer, and — right rail only —
   the three tab names vertically with the active one in `kPrimary()`. At the bottom, a
   Mono 9 px **count of what is behind the rail**: products on the left, running tasks on
   the right. A collapsed panel must still be able to tell you something happened.
5. **Focus mode** (`⌥⇧F`) collapses both rails and thins the status rail to health +
   queue. It is a toggle, not a separate mode — everything else keeps working.
6. **Persist all three states** (`paneCount`, `leftCollapsed`, `rightCollapsed`) per
   viewer type in `QSettings`, and restore them on open. Use a fresh key
   (`layout_v2/...`) so existing installs do not inherit the stage-3 arrangement.
7. **Do not force square panes.** At 1440 px no arrangement is square, and forcing a 1 : 1
   pane would letterbox away exactly the space rule 1 recovers. One large pane at
   1112 × 735 is what DS9 and CARTA give you and is nearly square already. The 2-pane
   layout only pays off from about 1800 px of window width, where each half is still
   ≥ 596 px wide — **offer it, do not default to it.** Note the vertical chrome is identical
   in every state (command bar 46 + padding 16 + toolbar 30 + gap 8 + pane header 34 +
   status rail 28), so pane **height is always 735** at a 900 px window — this change buys
   width, not height. Optionally: when the user picks 2 panes below 1600 px, split
   top/bottom instead of left/right, since a 1112 × 347 landscape pane beats a 550 × 735
   sliver for a channel map.

## Switching between 1 / 2 / 4 panes

See `#12d` (two panes), `#12e` (four panes), `#12f` (the rule table).

**Two invariants.**

- **Nothing is created or destroyed by switching layout.** Every view is a widget that is
  shown or hidden (`QStackedWidget` / `setVisible`), so ▣ → ⊞ → ▣ loses no camera, no
  zoom, no loaded slice, and costs no backend request.
- **The active pane is the unit of everything** — which pane survives a downgrade, which
  one the channel scrubber drives, which one "Send slice to image viewer" takes. Exactly
  one filled dot in one header at a time; clicking anywhere in a pane makes it active
  (1 px `kPrimary()` border on the active pane, 1 px `kOutline()` on the others).

**Fill order.** When a new pane appears it takes the first entry of
**2D Slice → Moment 0 → Spectrum** that is not already on screen. Products registered in
the session extend that list after Moment 0.

| Switch | Panes filled with | Side effects |
|---|---|---|
| ▣ → ▥ | left keeps the current view; right takes the first unused entry | toolbar 3D/2D picker **hides**, per-pane title dropdowns appear; `Linked` **enables**, on by default, scope "camera · channel · LUT" |
| ▥ → ⊞ | the two existing panes keep top-left / top-right; the new two continue down the list | `Linked` scope narrows to "channel · LUT"; pane headers go compact (padding 6/9, LUT strip 24 × 8, qualifier only if it fits) |
| ⊞ → ▣ | the **active** pane survives, not necessarily the first | toolbar 3D/2D picker returns, set to whatever survived; `Linked` disables and dims but remembers its state |
| ⊞ with nothing to show | the empty slot renders an **empty pane**: dark background, a centred "Pick a view" dropdown | never a bare black rectangle, and never a silent fallback to 2 panes — the user asked for 4 |
| any, window < 1600 | ▥ splits **top/bottom** (1112 × 363 beats 552 × 735 for a channel map); ⊞ stays 2 × 2 | split direction is derived from window width each time, not persisted |

**Per-pane view picker.** With `paneCount > 1` each pane header's title becomes a
dropdown (3D View / 2D Slice / Moment 0 / … / Spectrum). Choosing a view already shown in
another pane **swaps** the two panes rather than duplicating it.

**`Linked` scope is stated, not implied.** The chip's mono qualifier lists exactly what is
synced — "camera · channel · LUT" with two panes, "channel · LUT" with four, because camera
sync between a volume and three 2-D products means nothing.

**Pane sizes for reference** at a 1440 × 900 window with the default chrome (left dock
open, right rail collapsed): ▣ 1112 × 735 · ▥ 552 × 735 each · ⊞ 552 × 363 each. In a
552 × 363 pane a square image resolves to about **308 px**, limited by **pane height**
(a 322 px content box minus the 14 px longitude row) — the gutters constrain width, and
width is not the binding axis there. Tick labels drop to 8 px and every second label is
skipped. Always derive the square from the shorter of the two available axes; do not
assume it is the width.

## The placement rule

One rule decides where any control goes:

| Question it answers | Home |
|---|---|
| What am I looking at? | **left dock** |
| Compute something new | **right dock (Inspector)** |
| The data, and how panes are arranged | **centre** (toolbar + pane headers) |
| Is the system healthy? | **status rail** |

Worked examples: threshold and LUT change what I am looking at → left. Moment map computes
→ right. Auto/Local/Remote changes how this pane draws → that pane's header. Cache
occupancy → rail.

## Regions

| Region | Size | Contains | Must NOT contain |
|---|---|---|---|
| Command bar | h 46 | mark · dataset pill (badge · name · dims · load-state) · search 420 · health chip · theme | Find, Export |
| Left dock | w 272 (rail 36) | SESSION DATA · DISPLAY — 3D · LAYERS · INTERACTION ▶ · SLICE ANIMATION ▶ | bordered cards |
| Centre toolbar | h 30 | 1/2/4 segmented · **3D/2D segmented** · Linked chip · spacer · cursor readout | — |
| 3D pane | flex 1.25 | header h 34 + viewport `#000004` + channel scrubber inset 10 | — |
| 2D pane | flex 1 | header h 34 + viewport with axes outside the data | axes/colorbar over data |
| Right dock | w 320, **collapsed to 36 by default** | Properties / Analysis / Provenance | rendering controls, pane strip, backend chip |
| Status rail | h 28 | health │ queue │ cache │ session · right: Jobs panel ⌥J | load state, Sanity, RENDER |

## Left dock detail (w 272)

Flat rows, **no bordered cards**. Section header: Lato 10 px / 700, letter-spacing 0.14em,
`kOnSurfaceVariant()`, with a ▼/▶ disclosure. Sections separated by a 1 px
`rgba(4,138,191,0.20)` rule. Row padding 11 px 14 px, gap 7–8 px. Field rows: label
Lato 11 px `kOnSurfaceVariant()` on the left (flex 1), control right.

1. **SESSION DATA** — header with a "＋"; dataset row (badge + name, selected state
   `rgba(4,138,191,0.16)` + 2 px left border); product rows indented 26 px. When there are
   none: a dim "no products yet" row.
2. **DISPLAY — 3D** — the 2-segment Isosurface/Volume control, the 3-segment
   Composite/MIP/MinIP control, a "Threshold" label + mono value row above its slider, and
   a LUT row (40 × 11 gradient + name + "Edit" link in `kPrimary()`). These are the old
   RENDERING MODE / VOLUME RENDERING / RENDERING THRESHOLD / COLOR MAP cards, **flattened**:
   no card borders, no per-card padding, no "Advanced…" full-width button.
3. **LAYERS** — checkbox rows: Volume (+ 32 × 9 LUT strip), Cutting plane (+ "50%"),
   Contours, Catalogue sources, 3D WCS axes.
4. **INTERACTION** and **SLICE ANIMATION** — collapsed by default (▶).

## Right dock detail (w 320)

Tabs: padding 11 px 12 px, Lato 12 px; active `kPrimary()` / 600 + 2 px bottom border.

- **Properties** — identity block (kind badge + filename + a mono subtitle
  "64 × 64 × 64 · float32 · remote · full-res"), then **DATASET** (Path, Session, Backend,
  **Sanity**), **STATISTICS** (2D image and 3D cube sub-blocks, Min/Max/Mean/RMS), **WCS**
  (Status, Frame, Spectral axis, Pixel scale). Budget at 900 px height: 60 + 110 + 240 +
  120 ≈ 530 px. **If it scrolls, something that belongs on the left is in here.**
- **Analysis** — RUN A TASK chip cloud, one task form, RUNNING block. Before any task:
  "Tasks you start appear here with progress, and land in Session Data as products."
- **Provenance** — DERIVED FROM chain, then params of the selected product. With nothing
  selected: the chain plus "Select a product in Session Data to see the exact request that
  produced it.", and the three action buttons disabled.

## 2D pane geometry (the fix for axes over data)

Viewport is a **column**, padding 8, `box-sizing: border-box`, background `#000004`:

- row 1 (flex 1): latitude gutter **w 44** (right-aligned mono 9) · data area · colorbar
  **w 12** · colorbar value gutter **w 34** (left-aligned mono 9);
- the **data area keeps the image's pixel aspect ratio and never stretches**. It sits in a
  centring flex cell and is sized from the **shorter axis of that cell**: when the cell is
  taller than wide — which is the normal case in this layout, ~366 px wide × ~735 px tall —
  the side is the cell's *width*, and the leftover height becomes equal letterbox bands
  above and below, inside the `#000004` viewport. Never inside the gutters.
  In Qt: give the render widget a fixed-ratio container (or override
  `resizeEvent`/`heightForWidth`) rather than letting the layout stretch it. For a
  non-square image use `NAXIS1 : NAXIS2`, not 1 : 1;
- row 2 (flex none): a 44 px spacer, then the longitude tick row and the axis label
  ("Galactic Longitude", Lato 10 px `kPrimary()`), then a 46 px spacer.

**The letterbox applies to the whole group, not just the image.** The viewport is a
centring container (`align-items: center`); inside it a single group holds row 1 and
row 2 and is shrink-wrapped to the image's height. So:

- the latitude gutter, the colorbar and the value gutter all resolve to **exactly the
  image's height** — a declination tick can never fall outside the pixels it labels, and
  the colorbar gradient always maps the visible image and nothing more;
- the longitude row stays **adjacent to the image**: it is inside the group, so the
  leftover vertical space becomes equal bands *above the latitude gutter and below the
  longitude label*, never a gap between the image and its own axis.

In Qt: put the render widget, the two axis gutters and the colorbar in one
fixed-ratio container and centre that container in the pane — do not letterbox the render
widget alone inside a stretched grid, or the axes will annotate empty space.

Same rule as the plot panes (`4a`): a tick gutter and the data it annotates share a row,
so the gutter can never be taller than the data.
