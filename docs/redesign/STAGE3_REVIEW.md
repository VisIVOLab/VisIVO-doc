> **SUPERSEDED.** Use `HANDOFF_cube_viewer.md` — it consolidates this file, the parts
> of it that were never implemented, and the turn-12 layout work into one ordered list.

# Stage 3 review — six fixes before Stage 4

Reviewed against a screenshot of the built app (cube viewer, `hi_cube.fits`, 64×64×64).
Stages 1–3 are in and the pieces exist, but the shell is not yet laid out as designed.
Reference screens: `#1d` (workspace), `#3a` (inspector tabs), `#10c` (render mode).

Fix 1 and 2 first — they are structural and everything else is easier to judge once the
space is distributed correctly. In the current build the viewports occupy roughly 15 % of
the window and the 3D volume renders in about 130 px.

---

## 1. The old sidebar is still in the central widget — BLOCKER

Between the "Session Data" dock and the "3D View" dock there is a ~70 px vertical strip
containing the rail's four icons and clipped section headers: `RE…`, `RE…`, `CU…`,
`3D…`, `3D…` — i.e. RENDERING MODE, RENDERING THRESHOLD, CUTTING PLANE, 3D REFERENCE,
3D INTERACTION, built by `CubeUiAssembler` and mounted by
`vtkWindowCube::setupSidebar()`.

The plan **moves those pages into the Inspector's tabs**; the old `SidebarPanel` must not
remain alongside it. In `setupSidebar()` the panel is currently inserted into the central
layout with `mainLayout->insertWidget(0, this->m_sidebar)`. Remove it from the layout (the
same way `ui->sidebarContainer` and `ui->viewerContainer` are already removed and hidden)
and reparent the four page widgets into the Inspector:

| Old rail page | New home |
|---|---|
| 3D View Settings | Inspector › **Properties** (a "Rendering" section) |
| 2D View Settings | Inspector › **Properties** (a "2D view" section) |
| Tools | Inspector › **Analysis** (already grouped there) |
| Inspector (info/stats) | Inspector › **Properties** (statistics + WCS) |

Keep `SidebarPanel` in the tree for the catalogue/VBT viewers until stage 6; just do not
mount it in the cube and image viewers.

Same check in `vtkWindowImage_Setup.cpp` — its four pages (Layer Settings, Catalogue,
Tools, Info / Stats) must not stay mounted next to the Inspector either.

## 2. Inspector is docked at the bottom; both docks are the wrong width — BLOCKER

Current: Inspector spans the full width below the viewports with ~950 px of empty space,
and "Session Data" is ~700 px wide and almost empty.

Required (from `#1d`):

- **Session Data** → `Qt::LeftDockWidgetArea`, width **272 px**, full height.
- **Inspector** → `Qt::RightDockWidgetArea`, width **320 px**, full height.
- Viewports take the remaining central area.

Both docks should resist stretching: set a fixed-ish width with
`setMinimumWidth`/`setMaximumWidth` on the dock's inner widget (272 / 320) rather than
letting the splitter distribute space, and call `resizeDocks({left, right}, {272, 320},
Qt::Horizontal)` after `restoreState()` so a stale saved layout cannot reintroduce the
bottom placement. If a previously saved `QSettings` geometry is being restored, bump the
state key (e.g. `dockState_v2`) — otherwise existing installs keep the old arrangement.

## 3. The 1/2/4 + Linked strip is in the wrong place

It currently sits as the first row inside the Inspector. In the design (`#1d`, `#6a`) it
belongs to the **centre pane toolbar**, above the viewport grid, because it controls the
views and not the panel. The Inspector's first row is the **"RUN A TASK" chip cloud**.

Centre toolbar, left to right: the 1/2/4 layout segmented control · the `Linked` chip
(⛓ + "Linked" + a mono qualifier "camera · channel · LUT") · spacer · the live cursor
readout in mono.

Replace the `⛓` glyph — it has no coverage in Lato or JetBrains Mono and falls back to
two vertical bars. Use `resources/icons/rail_viz.svg` through
`VisivoTheme::renderThemeIcon()`.

## 4. The backend chip is duplicated

`127.0.0.1` appears twice: once in the command bar, once inside the Inspector. Keep one,
in the command bar, and give it the designed content — a status dot plus
`host · N running`:

- dot `kSuccess()` when reachable and idle, `kRunning()` when `runningTasks > 0`,
  `kError()` when unreachable;
- text from the same health poll that already feeds it.

Remove the copy from the Inspector.

## 5. The command bar is still the old viewer toolbar

It is `VisivoTheme::makeViewerToolbar()` with a dataset dropdown bolted on. Per `#1d`
the command bar replaces it:

- **missing**: the VisIVO mark on the left — 8 × 8 px `#048ABF` square + "VisIVO" Lato
  13 px / 900, then a 1 px `kOutline()` divider;
- **missing**: the centred search field, width 420 px, placeholder
  "Search datasets, tools, products…", with a `⌘K` key cap, opening `CommandPalette`;
- **remove**: `Find` (superseded by ⌘K) and `Export` (now the Export menu);
- **keep**: the dataset pill (kind badge + filename + dimensions + ▼) and the theme
  toggle.

The path-pill styling is fine to reuse for the dataset pill; it is the surrounding
buttons that should go.

## 6. The status rail repeats itself

Currently: "Full resolution" · "Remote · Full resolution" · "Sanity: OK" ·
`RENDER: LOCAL` · then the four correct slots. Load state appears twice and render mode
does not belong here.

The rail has **exactly four slots**, each with a 6 px dot, separated by a `│` in
`kSecondary()`, all JetBrains Mono 10 px:

| Slot | Dot | Content |
|---|---|---|
| Health | `kSuccess()` | `backend ok · 1 ms` |
| Queue | `kRunning()` | `0 running · 0 queued` |
| Cache | `kCache()` | `cache 0/32` |
| Session | — | `session anonymous` |

Right-aligned: `Jobs panel ⌥J`.

- Load state ("Full resolution" / "Preview") belongs in the **dataset pill's second tag**
  in the command bar, where it already reads `remote · full-res`. Drop both status-bar
  copies.
- `Sanity: OK` moves to Inspector › Properties as a KV row.
- `RENDER: LOCAL` becomes the **Auto / Local / Remote** segmented control in the 3D pane
  header (`#10c`), which is also where the stream fps/latency overlay goes.

---

## Not a defect

- Inspector › Analysis opening the existing dialogs is correct — that is Stage 4's job.
- Panes/Linked disabled in the image viewer is per plan (single view until stage 6).
- Moving `setupInspector()` to the end of the constructor so every `QAction` exists is the
  right fix; keep it.

## After these six

Re-check with the same dataset: the 3D pane should be roughly 700 × 500 rather than
130 px, the Inspector a 320 px column on the right with three tabs and the chip cloud on
top, and nothing left of the 3D pane except the 272 px Session Data dock.
