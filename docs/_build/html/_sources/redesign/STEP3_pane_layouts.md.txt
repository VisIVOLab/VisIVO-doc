# Step 3 — Layout states, pane fill, scrubber overlay (patch)

Prerequisite check: §2.7 and §4.5 are in, so the 2-D geometry is derived at draw time and a
collapsed dock really is 36 px. Both matter here — this step changes pane sizes constantly.

Normative: §3 of `HANDOFF_cube_viewer.md`, screens `#12a`–`#12f`. This is the largest patch
of the series. Read all of it before starting; the data model in 3.1 decides everything after.

---

## 3.0 One blocker to settle first

The handoff says step 3 needs "no new renderer or plot — only re-parenting". That is true for
the spectrum, and **false for the moment map**: `changeImageRenderer()` swaps `sliceWin` and
`momentWin` onto the *same* `ui->vtkImage`. A 4-pane layout showing `2D Slice` **and**
`Moment 0` at once therefore cannot work — one widget cannot host two render windows.

The fix is cheap and is not a new renderer: give `momentWin` its own
`QVTKOpenGLNativeWidget`. `setupMomentRenderer()` already builds the renderer, LUT, colorbar
and overlay actors; only the host widget is missing.

```cpp
    m_vtkMoment = new QVTKOpenGLNativeWidget(/*parent set by the pane slot*/);
    m_vtkMoment->setRenderWindow(this->momentWin);
```

Consequences, all of which simplify the rest:

- `changeImageRenderer()` stops swapping render windows. Slice / Moment becomes *which widget
  is in the slot*, not *which window is in the widget*.
- the 2-D pane header's `Slice | Moment Map` toggle keeps working at `paneCount <= 2` (it
  changes the slot's view). At 4 panes, when both are on screen, it is meaningless — the
  header title dropdown (§3.4) replaces it there.
- `layout2dPane()` takes the widget it is laying out instead of asking `viewingSlice()`.

Do this first, in its own commit, and verify Slice/Moment still switches at 2 panes before
touching the layout.

---

## 3.0b Fix these before §3.1

§3.0 verified: both widgets coexist, the toggle works, and the §2.7 gutter geometry applies to
the moment widget too. Four defects the first real moment map exposed.

### (a) The moment map is a white field

`lutMoment->SetNanColor(1., 1., 1., 1.)` paints every masked-out pixel opaque white. Outside
the ≈3σ mask that is most of the frame, so the map reads as a white sheet with black speckles
— and white is the *maximum* end of half the scientific colormaps, which is exactly the wrong
reading for "no data". It also blows out the pane next to a `#000004` viewport.

Masked pixels must read as absent, not as full scale:

```cpp
    this->lutMoment->SetNanColor(0., 0., 0., 0.);   // transparent → viewport shows through
```

The same applies to `lutSlice` if it carries an opaque NaN colour. Check the moment colorbar
afterwards: with NaN transparent, the bar's range now describes only real data, which is the
point.

### (b) Colorbar labels clip again, in scientific notation

`4.5e+0…`, `3.39e+0`, `1.15e+0` are cut. Moment 0 in `Jy/beam m/s` is ~1e4, so `%.3g` yields
seven characters where the slice needed four — a fixed 116 px right gutter cannot hold both.

Stop guessing the width; measure it:

```cpp
    // In layoutColorbarInGutter(), before positioning: the gutter must fit the
    // widest label this LUT will actually print.
    const QFontMetrics fm(QFont(u"JetBrains Mono"_s, 9));
    double labelW = 0.;
    for (double v : { range[0], range[1] })
        labelW = std::max<double>(labelW, fm.horizontalAdvance(
                QString::asprintf(bar->GetLabelFormat(), v)));
```

and let `imageGutter()` take that width: `g.right = std::clamp(19. + gap + labelW + 10.,
116., 176.)`. The axes read `gutter.right` already (§2.3), so the x axis shortens to match and
nothing overlaps. A hard 176 px ceiling keeps a pathological unit from eating the pane.

### (c) Provenance is empty even though a product exists

The Inspector auto-expanded to Provenance (§4.1 works) but shows "Select a product in Session
Data." while a moment product is registered and selected. `momentProvenanceState` — summary,
details, and the `lastAcceptedConfig()` behind it — is never handed to
`InspectorPanel::showProvenance()`.

Wire it in the `momentController` result callback, next to where
`updateMomentProvenancePanel()` is called today:

```cpp
    QVariantMap params;
    params.insert(u"Order"_s, cfg.order);
    params.insert(u"Channels"_s, u"%1–%2"_s.arg(cfg.channelStart).arg(cfg.channelEnd));
    params.insert(u"Mask"_s, /* the mask description already built for the status text */);
    params.insert(u"Scope"_s, MomentMapController::describeMomentScope());
    params.insert(u"Unit"_s, momentUnit);
    m_workspaceChrome->inspectorPanel()->showProvenance(u"Moment 0"_s, params);
```

Selecting the cube row instead must clear it back to the empty state — Provenance describes
the *selected* product, so `SessionDataTree`'s selection signal drives it.

### (d) The provenance string is in the status bar

`Moment map: Moment 0 (Integrated intensity) | Unit: Jy/beam m/s | Ch: 1..64 | Mask: auto
(≈ 3σ) | Scope: Remote full dataset` now occupies the middle of the status rail, next to the
two surviving "Full resolution" duplicates. That is Step 5's cleanup, but this string is new,
so retire it in the same commit as (c): it is the same data, and (c) gives it its real home.

```cpp
    this->momentProvenanceLabel->hide();   // superseded by Inspector › Provenance
```

Leave the rest of Step 5 alone for now; four slots is its own patch.

### Re-check

Compute a moment: masked pixels are viewport-black, not white; every colorbar label is fully
visible in both Slice and Moment; Provenance lists order, channels, mask, scope and unit, and
clears when the cube row is selected; the status rail carries no moment text.

---

## 3.1 The pane model

Today `applyPaneLayout()` hides `m_pane3dFrame` and calls `resetWorkspaceLayout()`. That does
not generalise. Replace it with an explicit model: **four slots, a view registry, and one
function that reconciles the two.**

```cpp
    enum class ViewId { None, View3D, Slice2D, Moment0, Spectrum };

    struct PaneSlot
    {
        QPointer<QFrame> frame;          ///< header + view, carries the active border
        QPointer<QWidget> header;
        QPointer<QLabel> dot;
        QPointer<QToolButton> titleBtn;  ///< title, a dropdown when paneCount > 1
        QPointer<QLabel> qualifier;
        QHBoxLayout *controls{ nullptr }; ///< where the per-view control is inserted
        ViewId view{ ViewId::None };
    };

    std::array<PaneSlot, 4> m_slots;
    int m_paneCount{ 1 };                ///< 1 / 2 / 4 — spec §3.1 default is 1
    int m_activePane{ 0 };
    QPointer<QWidget> m_viewParkingLot;  ///< hidden holder for off-screen views
```

The registry is a function, not a container — the widgets already exist as members:

```cpp
QWidget *vtkWindowCube::viewWidget(ViewId id) const
{
    switch (id) {
    case ViewId::View3D:   return m_renderStack;      // volume / remote-stream stack
    case ViewId::Slice2D:  return ui->vtkImage;       // now slice-only (see §3.0)
    case ViewId::Moment0:  return m_vtkMoment;
    case ViewId::Spectrum: return probePlotWidget;    // the existing ProfileWidget
    default:               return nullptr;
    }
}
```

**Invariant, enforced by construction:** nothing is created or destroyed by a layout change.
A view leaving the screen is re-parented into `m_viewParkingLot` and hidden; it keeps its
camera, zoom, loaded slice and plot data. Assert it in debug:

```cpp
    Q_ASSERT(viewWidget(id) != nullptr);   // never lazily construct here
```

---

## 3.2 The container

Keep real, draggable splitters — the user has been dragging them all along. Nested, so one
tree covers all three layouts:

```
m_paneSplitter (Horizontal)
├── m_leftColumn  (Vertical)  → slot 0 (top), slot 2 (bottom)
└── m_rightColumn (Vertical)  → slot 1 (top), slot 3 (bottom)
```

| Layout | Visible slots | Notes |
|---|---|---|
| 1 pane | 0 | right column hidden entirely |
| 2 panes, side by side | 0, 1 | today's arrangement |
| 2 panes, stacked | 0, 2 | when the window is < 1600 px (§3.4) |
| 4 panes | 0, 1, 2, 3 | 2 × 2 |

`QSplitter` ignores hidden children, so `applyPaneLayout()` is show/hide plus a reconcile —
no re-parenting of frames, no splitter rebuild, no lost drag positions:

```cpp
void vtkWindowCube::applyPaneLayout(int panes)
{
    m_paneCount = panes;
    const bool stacked = panes == 2 && this->width() < 1600;
    assignViews(panes);                       // §3.3 — fills empty slots

    const std::array<bool, 4> visible = panes == 1 ? std::array{ true, false, false, false }
                                      : panes == 2 ? (stacked ? std::array{ true, false, true, false }
                                                              : std::array{ true, true, false, false })
                                                   : std::array{ true, true, true, true };
    for (int i = 0; i < 4; ++i) {
        if (m_slots[i].frame)
            m_slots[i].frame->setVisible(visible[i]);
        if (!visible[i] && m_slots[i].view != ViewId::None)
            parkView(i);                      // re-parent to m_viewParkingLot, hide
    }
    m_rightColumn->setVisible(visible[1] || visible[3]);

    if (!visible[m_activePane])
        setActivePane(firstVisibleSlot());     // the active pane must always be on screen
    refreshPaneChrome();                       // dots, titles, qualifiers, controls
    layout2dPane(/*force=*/true);              // every 2-D pane changed size
    saveLayoutSettings();                      // §4.3
}
```

`resetWorkspaceLayout()` is no longer part of this path — layout switching must not touch
dock geometry. Leave it as the View-menu "Reset layout" action only.

---

## 3.3 Fill order

Spec §3.3: a newly appearing pane takes the first of **2D Slice → Moment 0 → Spectrum** not
already on screen; `ProductRegistry` products extend the list after Moment 0.

```cpp
void vtkWindowCube::assignViews(int panes)
{
    m_slots[0].view = m_slots[0].view == ViewId::None ? ViewId::View3D : m_slots[0].view;
    const QList<ViewId> order{ ViewId::Slice2D, ViewId::Moment0, ViewId::Spectrum };
    for (int i = 1; i < panes; ++i) {
        if (m_slots[i].view != ViewId::None)
            continue;                          // keep what the user put there
        for (ViewId candidate : order) {
            if (!isViewOnScreen(candidate, panes)) {
                m_slots[i].view = candidate;
                break;
            }
        }
        // Nothing left to show → an EMPTY pane, never a bare black rectangle and
        // never a silent downgrade to fewer panes: a centred "Pick a view" dropdown.
        if (m_slots[i].view == ViewId::None)
            showEmptyPanePlaceholder(i);
    }
    for (int i = 0; i < panes; ++i)
        mountView(i);                          // re-parent from the parking lot
}
```

`Moment0` is offered even before a moment has been computed — the pane then shows the
existing "no moment yet" state, which is information, not an error. Do **not** trigger a
computation from a layout change; that would break the no-backend-request invariant.

`Spectrum` mounts `probePlotWidget` — the existing `ProfileWidget`, currently in a floating
window. `setupSpectrumPlot()` / `updateSpectrumPlot()` / `setSpectrumCurrentChannel()` are
unchanged, so a spectrum pane follows the scrubber for free. When it is mounted in a pane,
suppress the floating window (`show()` / `raise()` at the two call sites near the end of
`vtkWindowCube.cpp` must check whether it is currently in a slot).

---

## 3.4 Headers and the switching table

Per-slot header changes, on top of §1.2:

- **title becomes a dropdown when `paneCount > 1`.** Picking a view already shown in another
  slot **swaps** the two slots rather than duplicating it — one `std::swap(m_slots[a].view,
  m_slots[b].view)` then `mountView()` both.
- at `paneCount == 1` the title is plain text and the toolbar's **3D / 2D** segmented control
  is visible (it is already built and hidden in `setupPaneChrome()`); it drives
  `m_slots[0].view`.
- **compact headers at 4 panes**: padding 6/9, LUT strip 24 × 8.
- `Linked` chip: disabled at 0.4 opacity when `paneCount == 1`; enabled and on by default at
  2; at 4 its qualifier narrows to **"channel · LUT"** — camera sync between a volume and
  three 2-D products is not real, so do not claim it.

The rest of the table from the handoff, as code paths:

| Switch | What runs |
|---|---|
| 1 → 2 | `assignViews(2)` fills slot 1 with Slice2D; picker hides; `setLinkViews(true)` |
| 2 → 4 | slots 2, 3 continue the list; headers go compact; Linked qualifier narrows |
| 4 → 1 | the **active** slot's view moves to slot 0 (not slot 0's view — the survivor is whichever pane was active); picker returns set to it; Linked disables but keeps its state |
| window resize crossing 1600 px at `paneCount == 2` | re-run `applyPaneLayout(2)`; direction is derived from the width every time, **never persisted** |

The 4 → 1 survivor rule is the one most likely to be got wrong: it is
`m_slots[m_activePane].view`, moved into slot 0.

---

## 3.5 The channel scrubber as an overlay

`ChannelOverlay` already exists, is parented to a viewport, and already re-parents itself in
the `m_renderStack` path around line 3224. Extend that to the pane model:

- it lives inside the **active** pane's viewport when that pane shows 3D or a 2-D image, and
  moves on `setActivePane()`;
- geometry: inset **10 px** from the viewport's left, right and bottom edges — so it floats
  over the data instead of taking a row from it. `rgba(0,0,0,0.55)`, 1 px `kOutline()`,
  radius 6, height 28;
- it is hidden when the active pane is the Spectrum (there is no channel axis to scrub in a
  plot — the spectrum's own marker already shows the channel);
- reposition in the host's `resizeEvent` via the existing event filter, not on a timer.

The inset means the scrubber sits **inside** the 2-D pane's letterbox, not inside the data
area — it must not cover pixels. If the letterbox is thinner than 38 px, place it over the
bottom of the data area at 0.55 alpha; that is the documented compromise, and it is why the
background is semi-transparent.

---

## 4.3 Persistence (same patch — it needs the pane assignment)

`QSettings`, prefix **`layout_v2/cube/`** (and `layout_v2/image/`) — a new prefix, so no
existing install can restore the stage-3 arrangement and make this work look like it never
landed. Nothing is read from an older key.

| Key | Value |
|---|---|
| `paneCount` | 1 / 2 / 4 |
| `views` | the four `ViewId`s as an int list, slot order |
| `activePane` | 0–3 |
| `leftCollapsed`, `rightCollapsed` | bool |
| `splitterState`, `leftColumnState`, `rightColumnState` | `QSplitter::saveState()` |

Restore order matters: docks first (they change the pane sizes), then `paneCount` + views,
then the splitter states, then `setActivePane()`, then one `layout2dPane(true)`. Restoring a
`ViewId` whose widget is null (an old setting, a product that no longer exists) falls back to
the fill order rather than leaving an empty pane.

The stacked-vs-side-by-side direction is **not** persisted — it is re-derived from the window
width, as above.

---

## Acceptance

The handoff's list, plus what this patch adds:

1. Opens at **1 pane**, 3D, viewport ≈ 1112 × 735 at 1440 × 900. Toolbar shows the 3D / 2D
   picker; `Linked` is dimmed to 0.4.
2. `2` → 3D + 2D Slice, both 552 × 735, `Linked` on, qualifier "camera · channel · LUT".
3. `4` → 3D, 2D Slice, Moment 0, Spectrum, each ≈ 552 × 363, each 2-D image square at
   ≈ 308 px, headers compact, qualifier "channel · LUT".
4. **1 → 4 → 1 loses nothing**: same camera, same zoom, same slice, and the network log shows
   **no** backend request. Check the log, do not assume.
5. From 4 panes, make the Moment pane active, then press `1`: the Moment map survives, not
   the 3D view.
6. Two panes at a window < 1600 px split **top/bottom**; widen past 1600 and they become
   side-by-side without a reload.
7. A slot with nothing left to show is an empty pane with a centred "Pick a view" dropdown —
   never a black rectangle, never a silent drop to 3 panes.
8. Picking a view already shown elsewhere swaps the two panes.
9. The scrubber floats inset 10 px in the active viewport, moves when the active pane
   changes, and hides on the Spectrum pane.
10. Quit and reopen: pane count, per-slot views, active pane, both dock states and the
    splitter positions all come back.
