# Step 2 — 2D pane geometry (patch)

Verified against the build in the 15.45 screenshot. Step 1's three blockers are closed:
`MinIP` and `0.188071` render in full, the Inspector tabs read `Properties / Analysis /
Provenance`, and the 3D header carries a real Auto / Local / Remote segmented control.

Two things remain before Step 3. **2.0** is a one-line leftover from Step 1; **2.1–2.5** are
Step 2 proper.

---

## 2.0 `Mom…Map` is still elided

`SegmentedToggle` gives every segment `QSizePolicy::Expanding` and `setMinimumWidth(0)`, so
a `setMaximumWidth()` on the container silently elides the labels instead of overflowing.
That is why `Moment Map` renders as `Mom…Map` under the 128 px cap in the 2D pane header,
and it will bite again on every future segmented control.

Fix it once, in the widget — the segments become *at least* as wide as their text, so a cap
that is too small is now visibly ignored rather than quietly destructive.

`theme/SegmentedToggle.cpp`, in `addItem()`, replacing `btn->setMinimumWidth(0);`:

```cpp
    // Segments share surplus width equally (flex), but never shrink below their
    // own label — an over-tight setMaximumWidth() on the container must overflow
    // visibly, not elide ("Moment Map" → "Mom…Map").
    btn->setMinimumWidth(btn->fontMetrics().horizontalAdvance(label) + 18);  // 8 px padding × 2 + 2
```

Then drop the two caps that were sized by eye, in `vtkWindowCube::setupPaneChrome()`:

```cpp
        rmode->setMaximumWidth(168);      // delete — content width is correct
```
```cpp
        c2->setMaximumWidth(128);         // delete
```

The pane header has a stretch before the controls, so both size to content. Re-run the
`[leftdock]` instrumentation after this: `DISPLAY — 3D`'s two toggles get a real minimum
width now, and the log line must still say `OK (fits)` at 272 px. If it flips to
`OVERFLOW → clips`, shorten the *label* (`MinIP` stays, `Composite` → `Comp.` before you
touch the dock width) rather than restoring a max-width cap.

---

## 2.1 What is actually wrong in the 2D pane

The overlay code already implements the gutter. `updateSliceWcsOverlay()` draws the axis
frame at `axisX = 136`, `bottomMargin = 58`, `rightMargin = 34`, `topMargin = 28` — display
pixels, measured from the window edge.

What is missing is the other half: **the image renderer still occupies the whole viewport**
(`ren->ResetCamera()` in `setupRemoteSliceRenderer()` / `setupMomentRenderer()`), so the
pixels are drawn *under* the gutter the axes were placed in. Hence latitude ticks over the
data, `Galactic Longitude` across the bottom of the image, and the colorbar — parked at a
hard-coded normalized `(0.9, 0.1)` — covering a column of real data.

So this is not "add a gutter". It is "make the camera respect the gutter that already
exists, and put the colorbar in it".

**Do not shrink `renderer->SetViewport()` to do this.** Every overlay in the file
(`configureAxisActor`, the tick actors, the catalogue labels, the probe readout) positions
in *display* coordinates while reading `renderer->GetSize()` for the extents — those two
agree only while the viewport is full-window. Shrinking the viewport desynchronises them
and breaks the catalogue and probe overlays as collateral. Fit the **camera** instead:
single renderer, full viewport, all existing overlay maths untouched.

---

## 2.2 Shared geometry: `CubeViewportGeometry`

The margins must stop being duplicated as four `constexpr` blocks (slice overlay, moment
overlay, and now the camera fit). One source of truth, next to the other viewport helpers.

`CubeViewportGeometry.h`, add to `namespace CubeGeom`:

```cpp
class vtkScalarBarActor;   // with the other forward declarations

// Pixel gutter reserved around the 2-D data area for WCS ticks, axis titles and
// the colorbar. `left` is the x of the y-axis line (tick labels sit to its left);
// `right` holds the colorbar + its value labels.
struct ImageGutter
{
    double left{ 136. };
    double right{ 96. };
    double bottom{ 58. };
    double top{ 28. };
};

/// The gutter in force for the 2-D panes. Collapses to a thin inset when the WCS
/// overlay is off, but always keeps room for the colorbar.
ImageGutter imageGutter(bool wcsAxesVisible);

/// Aspect-preserving parallel-projection fit of `image` into the gutter's inner
/// box. Sets ParallelScale + focal point; leaves the view direction alone.
bool fitImageInGutter(vtkRenderer *renderer, vtkImageData *image, const ImageGutter &g);

/// Parks a vertical scalar bar in the right gutter, aligned to the data area.
void layoutColorbarInGutter(vtkScalarBarActor *bar, vtkRenderer *renderer,
                            const ImageGutter &g);
```

`CubeViewportGeometry.cpp` — add the includes `<vtkScalarBarActor.h>`,
`<vtkTextProperty.h>`, `<vtkCoordinate.h>`, then:

```cpp
ImageGutter imageGutter(bool wcsAxesVisible)
{
    ImageGutter g;
    if (!wcsAxesVisible) {
        g.left = 12.;
        g.bottom = 12.;
        g.top = 12.;
    }
    return g;   // `right` is the colorbar's — it survives the axes being hidden
}

bool fitImageInGutter(vtkRenderer *renderer, vtkImageData *image, const ImageGutter &g)
{
    if (!renderer || !image) {
        return false;
    }
    const int *size = renderer->GetSize();
    if (!size || size[0] <= 0 || size[1] <= 0) {
        return false;
    }
    double b[6];
    image->GetBounds(b);
    if (!validBounds(b)) {
        return false;
    }

    const double innerW = static_cast<double>(size[0]) - g.left - g.right;
    const double innerH = static_cast<double>(size[1]) - g.bottom - g.top;
    if (innerW < 16. || innerH < 16.) {
        return false;   // pane too small to letterbox meaningfully — leave the camera
    }

    // World units per screen pixel, aspect preserved: the BINDING axis is whichever
    // of the two is scarcer, which in a 552 × 363 pane is the height, not the width.
    const double w = std::max(b[1] - b[0], 1e-9);
    const double h = std::max(b[3] - b[2], 1e-9);
    const double perPixel = std::max(w / (innerW - 2.), h / (innerH - 2.));

    auto *cam = renderer->GetActiveCamera();
    cam->ParallelProjectionOn();
    cam->SetParallelScale(perPixel * static_cast<double>(size[1]) / 2.);

    // Offset the focal point so the image centre lands in the inner box centre
    // rather than the viewport centre — this is what puts the letterbox bands
    // around the whole group instead of between the image and its own axis.
    const double dxPx = (g.left + innerW / 2.) - static_cast<double>(size[0]) / 2.;
    const double dyPx = (g.bottom + innerH / 2.) - static_cast<double>(size[1]) / 2.;
    const double cx = (b[0] + b[1]) / 2. - dxPx * perPixel;
    const double cy = (b[2] + b[3]) / 2. - dyPx * perPixel;
    const double cz = (b[4] + b[5]) / 2.;

    double distance = cam->GetDistance();
    if (!std::isfinite(distance) || distance <= 0.) {
        distance = 1.;
    }
    cam->SetFocalPoint(cx, cy, cz);
    cam->SetPosition(cx, cy, cz + distance);
    cam->SetViewUp(0., 1., 0.);
    renderer->ResetCameraClippingRange();
    return true;
}

void layoutColorbarInGutter(vtkScalarBarActor *bar, vtkRenderer *renderer,
                            const ImageGutter &g)
{
    if (!bar || !renderer) {
        return;
    }
    const int *size = renderer->GetSize();
    if (!size || size[0] <= 0 || size[1] <= 0) {
        return;
    }
    const double W = static_cast<double>(size[0]);
    const double H = static_cast<double>(size[1]);
    const double gap = 14.;                              // data area → bar
    const double boxW = std::max(g.right - gap - 6., 24.);  // bar + value labels
    const double boxH = std::max(H - g.bottom - g.top, 1.);

    bar->SetOrientationToVertical();
    bar->GetPositionCoordinate()->SetCoordinateSystemToNormalizedViewport();
    bar->SetPosition((W - g.right + gap) / W, g.bottom / H);
    bar->SetWidth(boxW / W);
    bar->SetHeight(boxH / H);
    bar->SetMaximumWidthInPixels(static_cast<int>(boxW));
    bar->SetMaximumHeightInPixels(static_cast<int>(boxH));
    bar->SetNumberOfLabels(5);
    bar->SetTitle("");
    bar->UnconstrainedFontSizeOn();
    if (auto *tp = bar->GetLabelTextProperty()) {
        tp->SetFontSize(9);
        tp->SetColor(0.85, 0.90, 0.95);
        tp->BoldOff();
        tp->ShadowOff();
        tp->SetJustificationToLeft();
    }
}
```

`UnconstrainedFontSizeOn()` matters: without it VTK rescales the label font to the bar's
box and the mono 9 px of the spec becomes whatever fits.

---

## 2.3 The two overlays read the shared gutter

In **both** `updateSliceWcsOverlay()` and `updateMomentWcsOverlay()`, replace the local
constants:

```cpp
    constexpr double leftMargin = 168.;
    constexpr double axisX = 136.;
    constexpr double bottomMargin = 58.;
    constexpr double rightMargin = 34.;
    constexpr double topMargin = 28.;
```

with:

```cpp
    const auto gutter = CubeGeom::imageGutter(this->showWcsAxes);
    const double axisX = gutter.left;
    const double leftMargin = gutter.left * 1.235;   // y-title column, unchanged ratio
    const double bottomMargin = gutter.bottom;
    const double rightMargin = gutter.right;
    const double topMargin = gutter.top;
```

Nothing else in either function changes — the axis, title and tick positions are already
expressed in these five names. `rightMargin` going 34 → 96 shortens the x-axis line to stop
at the colorbar, which is correct: the axis must span the data area, not the gutter.

---

## 2.4 Own the colorbars, and re-fit on resize

The two scalar bars are `vtkNew` locals in the setup functions, so nothing can re-layout
them. Promote them to members.

`vtkWindowCube.h` — add `#include <vtkScalarBarActor.h>` and, next to the other 2-D state:

```cpp
    vtkNew<vtkScalarBarActor> m_sliceColorbar;    ///< 2D slice colorbar (right gutter)
    vtkNew<vtkScalarBarActor> m_momentColorbar;   ///< moment-map colorbar (right gutter)
    std::array<int, 2> m_slice2dFitSize{ -1, -1 };
    std::array<int, 2> m_moment2dFitSize{ -1, -1 };

    /// Aspect-preserving gutter fit + colorbar placement for the active 2-D pane.
    /// Cheap and idempotent: returns immediately unless the viewport changed size,
    /// so it cannot fight a user zoom/pan. Pass force = true after new data.
    void layout2dPane(bool force = false);
```

In `setupRemoteSliceRenderer()` and `setupMomentRenderer()`, replace

```cpp
    vtkNew<vtkScalarBarActor> colorbar;
    colorbar->SetLookupTable(this->lutMoment);
    colorbar->SetMaximumWidthInPixels(100);
    colorbar->SetPosition(0.9, 0.1);
    ren->AddViewProp(colorbar);
```

with (the slice one uses `lutSlice` / `m_sliceColorbar`):

```cpp
    this->m_momentColorbar->SetLookupTable(this->lutMoment);
    ren->AddViewProp(this->m_momentColorbar);
    CubeGeom::layoutColorbarInGutter(this->m_momentColorbar, ren,
                                     CubeGeom::imageGutter(this->showWcsAxes));
```

and replace the trailing `ren->ResetCamera();` in each with

```cpp
    ren->ResetCamera();                 // keep: establishes a sane camera distance
    this->layout2dPane(/*force=*/true); // then letterbox it into the gutter
```

`layout2dPane()` itself, next to the overlay code in `vtkWindowCube_Wcs.cpp`:

```cpp
void vtkWindowCube::layout2dPane(bool force)
{
    const bool slice = this->viewingSlice();
    auto *win = slice ? this->sliceWin.GetPointer() : this->momentWin.GetPointer();
    if (!win) {
        return;
    }
    auto *renderers = win->GetRenderers();
    auto *renderer = renderers ? renderers->GetFirstRenderer() : nullptr;
    if (!renderer) {
        return;
    }
    const int *size = renderer->GetSize();
    if (!size || size[0] <= 0 || size[1] <= 0) {
        return;
    }
    const std::array<int, 2> viewport = { size[0], size[1] };
    auto &lastFit = slice ? this->m_slice2dFitSize : this->m_moment2dFitSize;
    if (!force && viewport == lastFit) {
        return;   // same geometry — do NOT touch the camera, the user may have zoomed
    }
    lastFit = viewport;

    auto *image = slice
            ? vtkImageData::SafeDownCast(this->remoteSliceDisplaySource->GetOutputDataObject(0))
            : vtkImageData::SafeDownCast(this->momentDisplaySource->GetOutputDataObject(0));
    const auto gutter = CubeGeom::imageGutter(this->showWcsAxes);
    CubeGeom::fitImageInGutter(renderer, image, gutter);
    CubeGeom::layoutColorbarInGutter(slice ? this->m_sliceColorbar : this->m_momentColorbar,
                                     renderer, gutter);
    this->invalidateWcsOverlayCache();   // margins/extent moved — force a tick recompute
    win->Render();
}
```

Call it from exactly four places — **not** from the `vtkCommand::EndEvent` observers, which
would recurse through `win->Render()`:

| Where | Call |
|---|---|
| `eventFilter()`, `QEvent::Resize` on `ui->vtkImage` | `QTimer::singleShot(0, this, [this]{ layout2dPane(); });` |
| `changeImageRenderer()`, after `setRenderWindow()` | `layout2dPane(true);` |
| `applyRemoteSliceResult()`, after the new image is set | `layout2dPane(true);` |
| `applyMomentMapResult()`, after `momentDisplaySource->SetOutput()` | `layout2dPane(true);` |

`setupPaneChrome()` already installs `this` as an event filter on the pane headers, so add
`ui->vtkImage->installEventFilter(this)` beside it and extend the existing filter:

```cpp
    if (event->type() == QEvent::Resize && watched == ui->vtkImage) {
        QTimer::singleShot(0, this, [this]() { this->layout2dPane(); });
    }
```

The deferral is deliberate — during `QEvent::Resize` the VTK render window has not yet
adopted the new size, so an immediate fit uses the old one.

---

## 2.5 Acceptance

Open `hi_cube.fits`, two panes, 1440 × 900.

1. No tick label, axis title or colorbar pixel overlaps the data area. The colorbar sits
   in a column to the right of the image with its values to *its* right.
2. The image keeps `NAXIS1 : NAXIS2` and is centred in the inner box. Letterbox bands are
   `#000004` and fall outside the whole group — never between the image and its own axis.
3. Drag the splitter narrow, then tall: the fit follows on both, and the binding axis
   changes with the pane (height-bound when short and wide).
4. Zoom with the scroll wheel: the fit does **not** snap back. Only a resize re-fits.
5. Switch Slice → Moment → Slice: both panes are laid out, neither camera is lost.
6. Turn the WCS overlay off: the image grows into the freed left/bottom gutter, the
   colorbar stays put.
7. `Moment Map` reads in full in the 2D pane header, and the `[leftdock]` line still says
   `OK (fits)`.

Once this passes, Step 3 (scrubber overlay, inset 10 px) is unblocked — it lands inside the
same inner box this patch just defined, so do them in this order.

---

## 2.6 Corrections after the 16.24 build

2.1–2.5 are in and correct: the image is square, centred, and nothing overlaps the data.
Three things are still off, and two of them are the same mistake — **the gutter's inner box
is not the data area**. The image is letterboxed *inside* the inner box, so anything aligned
to the box instead of to the image annotates empty space. That is exactly what item 2 of
Step 2's own rules warned about; the patch only applied it to the image.

### (a) The colorbar spans the viewport, not the data

It runs the full inner height — roughly 320 → 1360 px — while the data ends at 590 and 1090.
So its top and bottom fifths map values onto letterbox. The bar must be exactly as tall as
the image and share its top and bottom edge.

Make the fit report where the image actually landed, and align to that.
`CubeViewportGeometry.h`:

```cpp
/// Where the fitted image landed, in display pixels — the DATA AREA, which is
/// the gutter's inner box minus the letterbox. Everything that annotates the
/// data (colorbar, WCS axes, ticks) aligns to this, never to the inner box.
struct ImageRect
{
    bool valid{ false };
    double x{ 0. };
    double y{ 0. };
    double width{ 0. };
    double height{ 0. };
};

bool fitImageInGutter(vtkRenderer *renderer, vtkImageData *image, const ImageGutter &g,
                      ImageRect *dataArea = nullptr);
void layoutColorbarInGutter(vtkScalarBarActor *bar, vtkRenderer *renderer,
                            const ImageGutter &g, const ImageRect &dataArea);
```

In `fitImageInGutter()`, after `perPixel` is known and before returning true:

```cpp
    if (dataArea) {
        const double drawnW = w / perPixel;
        const double drawnH = h / perPixel;
        dataArea->valid = true;
        dataArea->x = g.left + (innerW - drawnW) / 2.;
        dataArea->y = g.bottom + (innerH - drawnH) / 2.;
        dataArea->width = drawnW;
        dataArea->height = drawnH;
    }
```

In `layoutColorbarInGutter()`, replace the `boxH` line and the two position/height calls:

```cpp
    const double barY = dataArea.valid ? dataArea.y : g.bottom;
    const double boxH = std::max(dataArea.valid ? dataArea.height : H - g.bottom - g.top, 1.);
    // …
    bar->SetPosition((W - g.right + gap) / W, barY / H);
    bar->SetHeight(boxH / H);
```

`layout2dPane()` keeps the rect and hands it on:

```cpp
    CubeGeom::ImageRect dataArea;
    CubeGeom::fitImageInGutter(renderer, image, gutter, &dataArea);
    CubeGeom::layoutColorbarInGutter(slice ? this->m_sliceColorbar : this->m_momentColorbar,
                                     renderer, gutter, dataArea);
    this->m_slice2dDataArea = dataArea;   // or m_moment2dDataArea — see (b)
```

### (b) The WCS axes have the same bug, latent

In this build the binding axis happens to be the width, so the drawn image fills the inner
box horizontally and the longitude ticks line up by luck. Make the pane tall and narrow —
or go to four panes, where the binding axis is the height — and the latitude ticks will
annotate letterbox. Fix it now, in the same pass, or Step 3's 4-pane layout will look broken
for a reason that has nothing to do with Step 3.

Store the rect per view (`ImageRect m_slice2dDataArea, m_moment2dDataArea;` in the header),
then in **both** overlay functions derive the frame from it instead of from the margins:

```cpp
    const auto gutter = CubeGeom::imageGutter(this->showWcsAxes);
    const auto &data = this->m_slice2dDataArea;          // moment: m_moment2dDataArea
    const double axisX = data.valid ? data.x : gutter.left;
    const double axisRight = data.valid ? data.x + data.width : size[0] - gutter.right;
    const double axisTop = data.valid ? data.y + data.height : size[1] - gutter.top;
    const double bottomMargin = data.valid ? data.y : gutter.bottom;
    const double leftMargin = gutter.left * 1.235;
```

and swap the two `configureAxisActor` calls plus the tick loop to use `axisRight` / `axisTop`
where they currently compute `size[0] - rightMargin` / `size[1] - topMargin`. The x tick
labels stay at `bottomMargin - 18` and the y labels at `axisX - 10`, so the label offsets
are unchanged — only the endpoints move.

Because the axes now depend on the fit, `layout2dPane()` must run **before** the overlay
update. It already calls `invalidateWcsOverlayCache()`, which is what makes the next
`EndEvent` recompute — keep that call last.

### (c) Colorbar labels are clipped, and too long

`0.038…`, `-0.017` and `-0.036` are cut by the pane edge: the 96 px right gutter leaves ~35 px
for labels after VTK's own bar width, and `%g`'s default precision wants more. Two changes
in `imageGutter()` / `layoutColorbarInGutter()`:

```cpp
    g.right = 116.;                  // bar ≈ 19 + values ≈ 70 + padding
```
```cpp
    bar->SetLabelFormat("%.3g");     // 0.0386 → "0.0386", -0.0361 → "-0.0361"
```

Three significant figures is the right precision for a colorbar — the exact extrema are in
Inspector › Properties › STATISTICS, which is where a number you intend to quote comes from.

### (d) The cursor readout is a raw dump

Unrelated to the geometry but now the most visibly broken thing in the toolbar: it reads
`X=54 Y=25 Z=0 Value=0.013683276 Galactic Longitude=104.89 Galactic Latitude=68.53 Radio
Velo…` and clips. Spec §1.1 asks for JetBrains Mono 10 px in the compact form:

```
l 104.89  b +68.53  ·  1.37e-02 Jy/beam  ·  ch 1
```

Rules: WCS first (it is what the user reads out loud), value in scientific notation with the
unit from the header, voxel indices **only** when there is no WCS solution, and no `KEY=`
labels — the axis names live on the axes. Cap the label with `setMinimumWidth()` on the
longest expected string so the toolbar stops reflowing as the cursor moves, and elide the
tail rather than the head if it still overflows.

### Re-check

On top of §2.5: the colorbar's top and bottom edges are level with the image's, its labels
are fully visible, and in a **tall narrow** pane (drag the splitter to ~380 px wide) the
latitude ticks still sit against the image edge with letterbox left and right of the whole
group.

---

## 2.7 The data area must be derived, not cached

The rails (Step 4.1) exposed a real bug in §2.6(b). Same build, three dock states:

| Dock state | Where the x tick row lands |
|---|---|
| both rails collapsed | **on top of the pixels**, ~70 px inside the image |
| left collapsed, right open | ~70 px **below** the image's bottom edge |
| left open, right collapsed | roughly correct |

The cause: `m_slice2dDataArea` is written by `layout2dPaneFor()` and read by the overlay, and
the two do not run at the same viewport size. The fit is deferred a tick
(`QTimer::singleShot(0)`, correctly — VTK has not adopted the new size during
`QEvent::Resize`), while the overlay runs off `vtkCommand::EndEvent` with whatever size the
window has *now*. Collapsing a dock resizes the pane twice in quick succession, so the
overlay draws its frame from a rect measured for the previous width. Where the pane got
taller the stale `data.y` falls inside the new image; where it got shorter it falls below it.

Caching a derived geometric value in a member is the bug. The rect is a pure function of
three things the overlay already has — viewport size, image bounds, gutter — so derive it at
the point of use and delete the member.

Split the calculation out of `fitImageInGutter()`; the fit then calls it too, so there is one
formula and nothing to keep in sync:

```cpp
// CubeViewportGeometry.h
/// Where an aspect-preserved image lands inside the gutter, for a GIVEN viewport
/// size. Pure — no renderer state, no caching. Both the camera fit and the WCS
/// overlays call this, so they cannot disagree about where the data is.
ImageRect dataAreaFor(const int size[2], vtkImageData *image, const ImageGutter &g);
```

```cpp
// CubeViewportGeometry.cpp
ImageRect dataAreaFor(const int size[2], vtkImageData *image, const ImageGutter &g)
{
    ImageRect r;
    if (!size || size[0] <= 0 || size[1] <= 0 || !image) {
        return r;
    }
    double b[6];
    image->GetBounds(b);
    if (!validBounds(b)) {
        return r;
    }
    const double innerW = static_cast<double>(size[0]) - g.left - g.right;
    const double innerH = static_cast<double>(size[1]) - g.bottom - g.top;
    if (innerW < 16. || innerH < 16.) {
        return r;
    }
    const double w = std::max(b[1] - b[0], 1e-9);
    const double h = std::max(b[3] - b[2], 1e-9);
    const double perPixel = std::max(w / (innerW - 2.), h / (innerH - 2.));
    r.valid = true;
    r.width = w / perPixel;
    r.height = h / perPixel;
    r.x = g.left + (innerW - r.width) / 2.;
    r.y = g.bottom + (innerH - r.height) / 2.;
    return r;
}
```

`fitImageInGutter()` keeps the camera work and takes its numbers from the same call —
`perPixel` is recoverable as `w / rect.width`, or return it in the struct; either way it is
computed in one place.

In **both** overlay functions, replace the cached read with a local derivation. `size` is
already in scope from `renderer->GetSize()`:

```cpp
    const auto gutter = CubeGeom::imageGutter(this->showWcsAxes);
    const auto data = CubeGeom::dataAreaFor(size, imageData, gutter);   // never cached
```

Then **delete** `m_slice2dDataArea` / `m_moment2dDataArea` and every write to them. Keep
`m_slice2dFitSize` / `m_moment2dFitSize` — those guard the *camera*, which must not be
re-fitted on every render, and that is a different concern.

The overlay is then correct for whatever size it is drawing at, regardless of how many
resizes a dock toggle produced or when the deferred fit ran.

### The axis titles travel with their axis

Second, smaller mistake, visible in all three shots: `Galactic Longitude` sits ~250 px below
the image, at the pane's bottom edge. §2.6 anchored it to `gutter.bottom / 2 - 2` — fixed to
the *window*, which is right when the letterbox is thin and absurd when it is 250 px tall.
An axis label belongs to its axis:

```cpp
    // x title: centred under the tick row, which sits at data.y - 18
    xTitle->SetDisplayPosition(static_cast<int>(data.x + data.width / 2.),
                               static_cast<int>(std::max(data.y - 38., 10.)));
    // y title: left of the tick column, which sits at data.x - 10
    yTitle->SetDisplayPosition(static_cast<int>(std::max(data.x - 74., 12.)),
                               static_cast<int>(data.y + data.height / 2.));
```

The `std::max` floors keep both titles on screen when the gutter is tight; otherwise they
follow the letterbox and stay ~20 px from their own tick row. The y title is already rotated
90° and centre-justified, so the position is its midpoint — no other change.

### Re-check

Cycle all four dock states (both open → left collapsed → right collapsed → both collapsed)
and `⌥⇧F` twice. In every state the tick rows sit **just outside** the image — never on the
pixels, never adrift in the letterbox — and both titles stay with their tick row. Then drag
the splitter through the aspect flip (wide → tall), which changes the binding axis rather
than the size, and confirm the same.
