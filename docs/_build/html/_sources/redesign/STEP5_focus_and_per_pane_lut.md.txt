# Step 5 — Pane activation and per-pane LUT (patch)

Written against the B3 build (Pane2DView landed, Moment 0 ∥ Moment 1 side by side).
Design reference: `#13a` (focus) and `#13b` (DISPLAY follows the active pane) in
`VisIVO Workspace.dc.html`. Two defects, one shared cause: the pane is not a
first-class focus target, and DISPLAY is still hard-wired to the cube.

---

## 5.1 One activation rule for every pane

**Defect.** `Pane2DView`'s interactor is live on hover: moving the pointer across a
moment pane re-windows it. The 2D Slice requires a click, which is the correct
behaviour; the moments must match it, and so must 3D, PV and Spectrum.

Rules, identical in every pane type:

| Event | Behaviour |
|---|---|
| `enterEvent` | nothing — no interactor, no cursor change, no render. Do not implement the handler. |
| first `mousePressEvent` | activates the pane, and is **consumed** — it must not also begin a window/level drag, a pick or a camera rotate. |
| while active | full interactor: right-drag window/level, wheel zoom, middle-drag pan. |
| `leaveEvent` | stays active. Focus moves only on a press inside another pane. |
| dock/toolbar clicks | do not change the active pane (so you can click Moment 1, then edit its LUT in the left dock). |

Implementation:

- `QVTKOpenGLNativeWidget::setFocusPolicy(Qt::ClickFocus)` in every pane, including
  the 3D one.
- The pane container installs an event filter on the render widget. On
  `MouseButtonPress`, if `pane != activePane`: set the active pane, `return true`
  (swallow). Otherwise fall through.
- `vtkRenderWindowInteractor::SetEnabled(pane == activePane)` — flip it in the
  activation slot, not per-event, so no VTK observer sees a stray move.
- Delete any `setMouseTracking(true)` / enter-leave handling on the render widgets.
- Exactly one active pane at a time, in a single `m_activePane` on the pane
  container; `activePaneChanged(PaneId)` is the only signal anything else listens to.

Active-pane chrome (already half-present): 1 px `kPrimary()` (`#048ABF`) border on
the pane frame, filled header dot, header row tinted 8 % primary. Inactive: hairline
`kLine()`, hollow dot.

`viewingSlice()` and the probe / region / PV tools already route to the active pane —
keep that, and when the active pane is a moment, **disable** those actions with the
reason in the tooltip ("a moment map is collapsed in velocity — pick the 2D Slice"),
never silently no-op.

### 5.1b Carried into §5.2 (deferred from the first §5.1 pass)

- **Remote 3D pane.** The remote render widget is a plain `QWidget` with its own mouse
  handlers, so an inactive remote 3D pane still interacts. It needs the same gate as the
  VTK interactor: a single `setInteractive(bool)` on the remote widget, driven from
  `refreshPaneInteractors()`, and the same press-swallow in the event filter. The rule is
  "not active ⇒ no input", regardless of who renders.
- **Tool gating.** Probe / region / PV keep their own backend enable logic; add
  `activePaneChanged` as one more input to it, and when the active pane is a moment set
  the disabled tooltip ("a moment map is collapsed in velocity — pick the 2D Slice").
  Disabled, never silently no-op.

### 5.1c Resolved — Qt trackpad, not a shared LUT

The hover mutation was a known Qt defect with the macOS trackpad: a pointer move with no
button held was delivered as a drag, so VTK's image style did a genuine window/level.
Nothing in our colour-state ownership was at fault. The Qt-level swallow in §5.1 is the
right and sufficient fix — keep it, and keep swallowing `MouseMove` on an inactive pane
(it is what makes the workaround hold on that hardware).

Still worth one check while building §5.3, cheap because you are in that code anyway: if
`ColorMaps` hands out a cached mutable `vtkLookupTable` / `vtkColorTransferFunction`
*instance*, per-pane LUT state cannot work at all — whichever pane calls
`SetTableRange` last wins for everyone. Every `Pane2DView` must own a deep copy.

## 5.2 DISPLAY binds to the active pane

**Defect.** The left dock's DISPLAY section always edits the cube's LUT, so a Slice or
a Moment cannot be recoloured at all.

- Rename the section header to `DISPLAY — <active pane title>`
  (`DISPLAY — 3D VIEW`, `DISPLAY — 2D SLICE`, `DISPLAY — MOMENT 1`). The pane name in
  `kPrimary()`; the word DISPLAY keeps the existing section style.
- Rebuild the section body on `activePaneChanged`:
  - **3D View** — Isosurface/Volume, blend, threshold, Color map + Edit, Range.
    Exactly what exists today.
  - **2D Slice** — Color map + Edit, Scale (linear / log / asinh), Range
    (min–max / 99.5 % / manual) and the two window numbers.
  - **Moment n** — Color map + Edit, "Symmetric about 0" (default on for mom1),
    Range. No Scale row.
- The two Range numbers and the pane's right-drag window/level are **one state**:
  dragging writes the fields, typing in a field windows the pane. Today they are two:
  the drag windows the `vtkImageProperty` (post-scale, on the RGBA) while the colorbar
  reads the `TableRange` — which is why the bar and the image disagree, and the likely
  root of the Slice washout. Redirect the drag to `SetTableRange` and neutralise the
  property window/level, so there is exactly one authoritative range per pane.
- No active pane: the section stays, showing the last pane's controls disabled, header
  reads `DISPLAY — no pane selected`.
- `Edit` opens the existing LUT customizer, targeting the active pane's lookup table —
  today it is bound to the volume's; take the target from `activePane()->lookupTable()`.

## 5.3 Per-pane LUT state and defaults

LUT is **per pane slot**, not per product — two panes showing the same Moment 0 can
carry different colour maps (the shared `imageData` is untouched by this).

Store on `Pane2DView`: `colorMapName`, `invert`, `scale`, `rangeMode`, `rangeMin`,
`rangeMax`. Defaults by product kind:

| Kind | Default map |
|---|---|
| 2D Slice, Moment 0 | `inferno` |
| Moment 1 | `coolwarm`, symmetric about 0 |
| Moment 2, PV | `viridis` |

A recompute of a product refreshes the pixels in every visible copy and **keeps** each
pane's LUT.

## 5.4 The header LUT strip becomes the shortcut

The 30 × 9 strip in the pane header is currently decorative. Make it a button: click
opens a small popover — the colour-map list with the current one ticked, then
`Edit LUT… · Invert · Reset`. It writes the same per-pane state as §5.2; the left dock
is the full control, the strip is the one-click path.

### 5.4b The popover list must be curated, not the registry

The first build dumps the entire VTK colour-map registry into the popover — `AllBlack`,
`AllCyan`, `PureRed`, `Run1`, `Run2`, `Sar`, `VolRenRGB`, `TenStep`, `DefaultStep`,
~35 entries, most of them volume-rendering transfer functions or single-colour ramps
that are meaningless on a 2D map. A list that long is also taller than the pane it
belongs to.

The popover shows **six** maps, in this order, each with its gradient swatch and the
current one ticked:

`inferno · magma · plasma · viridis · coolwarm · gray`

Then a separator, `More maps…` (opens the full registry in a proper scrollable dialog,
for the rare case), then `Edit LUT… · Invert · Reset`. Same six in the dock's Color map
combo, with `More maps…` as its last item.

`MinMax` is not a colour map and must not appear in the list — it is a range mode, and
its presence in that combo is why the dock read `Color map: MinMax`. Range lives in the
Range row only.

**The current map must be visible in the popover.** Today no row is marked, so the
popover cannot tell you what the pane is on. The active row gets: a tick at the right
edge, the label in `kFg()` and semibold while the others are `kFg2()`, and a
`rgba(4,138,191,0.14)` fill with a 1 px `kBorderLine()` — the same treatment as a
selected list row elsewhere in the app. Exactly one row is ever marked.

**A map chosen from `More maps…` joins the list.** Picking e.g. `Temperature` leaves
the six-item list unable to represent the state — the popover would show nothing
selected while the dock reads `Temperature`. The chosen map is inserted as a **seventh
row at the top**, above `inferno`, marked selected, with its swatch. It persists in that
pane's popover for the session, so it can be re-picked after switching away. If a second
non-standard map is chosen it replaces the first — never a growing list. Same rule in
the dock combo: `Temperature` sits at the top as a real item, not a transient label.

## 5.5 Linked · LUT

With **Linked** on, a LUT change propagates only to panes showing the **same product**
— the two copies of Moment 0 stay in step, Moment 1 keeps coolwarm. Never across
kinds, never from a moment to the slice. With Linked off, every pane is independent.

**Make the scope legible — the silence is the bug.** In a 3D · Slice · Moment 0 ·
Moment 1 layout no two panes show the same product, so a correct implementation does
nothing when Linked is on, and looks broken. Two additions:

- The `channel · LUT` text next to the Linked chip becomes real per-facet state: two
  small toggles, lit when on, greyed when Linked itself is off. LUT is a facet you can
  switch off independently of channel.
- When a LUT change propagates, the receiving panes flash their header LUT strip for
  ~400 ms — the only way to see that linking did something. When Linked is on and a LUT
  change propagates to **nothing**, the chip shows a one-shot hint:
  `only pane showing Moment 1 — nothing to link`.

---

## Acceptance

Open `hi_cube.fits`, four panes, 3D · 2D Slice · Moment 0 · Moment 1.

1. Sweep the pointer across all four panes without pressing: nothing changes — no
   contrast shift, no cursor change, no re-render.
2. Click Moment 1: blue border and filled dot appear there and nowhere else. That
   click alone does not alter the window. Right-drag now windows it; Moment 0 does not
   move.
3. The left dock header reads `DISPLAY — MOMENT 1`, colour map `coolwarm`. Change it to
   `viridis`: only Moment 1 changes.
4. Click the 2D Slice, set Scale = log: only the slice changes, and Moment 1 has kept
   viridis when you go back to it.
5. Click the LUT strip in the Moment 0 header, pick `gray`: the popover selection and
   the left dock agree.
6. Two panes on Moment 0, Linked off: different LUTs stick. Linked on: they follow each
   other, Moment 1 unaffected.
7. Recompute Moment 0: both copies redraw with their own LUTs intact.
