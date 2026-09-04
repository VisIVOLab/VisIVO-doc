# Next session — Step 2 (2D pane geometry) + carry the 3 done fixes

Resume for the cube-viewer handoff. Steps 0 and 1 are done. This is **Step 2**
(CUBE_VIEWER_SPEC.md "2D pane geometry"): axes + colorbar OUTSIDE the data, image
keeps its pixel aspect ratio with letterbox, colorbar in its own right gutter.

## Already done and Codex-clean (verify visually with Step 2, no separate screenshot)
- **Left-dock clipping** fix: SegmentedToggle segments Expanding + `minimumWidth(0)`; LUT `QComboBox` → `Ignored` size policy + `AdjustToMinimumContentsLengthWithIcon` + `minimumContentsLength(3)` so it shrinks; section margins 10; threshold value maxWidth 140. **Instrumentation**: on run the console prints `[leftdock] content minWidth=… viewport=… OK (fits)|OVERFLOW → clips`. If it still says OVERFLOW, that number names the remaining offending width — reduce that widget's min.
- **Inspector tabs**: `tabBar()->setElideMode(Qt::ElideNone)` + `setExpanding(false)` + `QTabBar::tab{padding:8px 12px}` (InspectorPanel.cpp).
- **3D pane header**: Auto/Local/Remote `SegmentedToggle` bound to `m_actionRenderAuto/Local/Remote` (replaced the RENDER:LOCAL badge; badge removed from status bar).

## Step 2 — findings (where the 2D geometry lives)
- The 2D view is a **single `vtkRenderer`** created in `setupSliceRenderer()` (vtkWindowCube_Setup.cpp ~1178). On it: the image `vtkImageSlice` (fills the viewport → stretched + drawn under the axes), the WCS axes `vtkAxisActor2D` (`sliceOverlayXAxis/YAxis`, added to the SAME `ren` at ~1254), the colorbar `vtkScalarBarActor` (`SetPosition(0.9,0.1)`, MaxWidth 100 → line 1237), contours, and the PV/region/probe interactor observers.
- Axis positions are recomputed in `updateSliceWcsOverlay()` (vtkWindowCube_Wcs.cpp:563) using FIXED pixel margins: `leftMargin=168, axisX=136, bottomMargin=58, rightMargin=34, topMargin=28`. The axes sit in those margins; the image is NOT constrained to the inner region, so it extends under them.

## Step 2 — the plan (do it this way)
1. **Inset the image into the data region.** Cleanest: give the image its OWN `vtkRenderer` with a sub-viewport = the data rectangle `[axisX .. size-rightMargin-colorbarGutter] × [bottomMargin .. size-topMargin]` (normalised, recomputed on resize), layered under a full-viewport overlay renderer that carries the axes + colorbar. Keep the interactor observers on the image renderer. Alternative (less clean): keep one renderer but position/scale the `vtkImageSlice` actor into the data rect — harder with the camera.
2. **Aspect ratio / letterbox.** The data area must keep `NAXIS1:NAXIS2` (not 1:1 unless square) and be sized from the SHORTER of the two available axes (in the tall 2D pane the binding axis is width; leftover height → equal `#000004` letterbox bands above/below the WHOLE group, never inside the gutters). In VTK: set the image renderer's sub-viewport to the aspect-correct rectangle (compute from image dims + pane size), not a stretched cell.
3. **Colorbar → its own right gutter** (spec: colorbar w12 + value gutter w34), OUTSIDE the data rect, not `SetPosition(0.9)` over the data. Its gradient must map exactly the visible image height.
4. **Axis gutters shrink-wrap to the image height** — a latitude tick can never fall outside the pixels it labels; the longitude row stays adjacent to the image (letterbox above/below the group, never a gap between image and its axis).

Risk: this renderer also hosts PV extraction, region stats, and the probe (display↔world `this->coordinate` uses `ren`). If the image moves to a sub-viewport renderer, re-point `this->coordinate->SetViewport(imageRenderer)` and verify PV/region/probe still map correctly. Do it incrementally, build + run after each of the four points.

## Reuse / constraints
- Theme accessors `k*()`; `#000004` for viewport background only.
- No new renderers/plots for the 4-pane layout later (spectrum = probePlotWidget); but Step 2 DOES legitimately add a sub-viewport renderer for the 2D image.
- Positive side effect already present: the colorbar now sits outside the image's right edge — keep it.
