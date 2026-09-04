# Next session — finish Step 0 (the left-dock flatten) + carry-over fixes

Resume point for the cube-viewer handoff (`HANDOFF_cube_viewer.md`, normative
`CUBE_VIEWER_SPEC.md`). Step 0's **safe half is done and verified**; only the
flatten surgery remains. Then Step 1 (pane infrastructure), etc.

## Already done in Step 0 (do NOT redo)
- **0.1** — no orphan `SidebarPanel` (`new SidebarPanel` gone from cube+image; `m_sidebar` null). Regression check each step: empty Session Data shows no stray text/rules.
- **Spec constants** in `theme/VisivoTheme.h`: `kLeftDockWidth=272`, `kRightDockWidth=320`, `kRailWidth=36`, `sectionHeaderCss()` (Lato 10px/700, 0.14em). Dock widths use them (fixed 272/320).
- **Properties = KV rows only** (no rendering controls, no scrollbar). Info/stats page temporarily parked in the left LAYERS block — its Min/Max/Mean/RMS must be rebuilt as KV rows under **STATISTICS** in Properties (spec budget 60+110+240+120 ≈ 530 / 900).
- **Anti-scroll instrumentation**: `InspectorPanel::logPropertiesFit()` logs post-layout (`QTimer::singleShot(0)`) the Properties `contentHint` vs scroll `viewport` height + the measured widget's objectName/geometry. Add the same no-scroll check to the LEFT dock while doing the flatten.

## The flatten — HOW (user directive, do it this way)
**Build the flat rows DIRECTLY, cube-side. Do NOT de-border the existing cards with a `findChildren` sweep, and leave `CubeUiAssembler::sidebarCardStyle()` / `sidebarHdrStyle()` UNTOUCHED** — catalogue and VBT viewers still use them. More code, but the only way not to break the other viewers.
- Bind the new flat controls to the **same QActions / signals** the assembler already wires (single source of truth). The assembler stores button pointers (`m_btn*`) used by `wireToolActions()` — do not delete those.
- Flat-row spec (CUBE_VIEWER_SPEC.md “Left dock detail”): section header Lato 10px/700, 0.14em, `kOnSurfaceVariant()`, with a ▼/▶ disclosure; sections separated by a 1px `rgba(4,138,191,0.20)` rule; row padding 11px 14px, gap 7–8px; field rows = label Lato 11px `kOnSurfaceVariant()` left (flex 1), control right. LUT row = 40×11 gradient + name + **“Edit” link** in `kPrimary()` — **no full-width “Advanced…” button**.

## Two flatten-scope fixes the user flagged (or you'll redo them)
1. **Left dock must not scroll (one dataset) and has a ~250px gap today.** Cause: the session tree has the layout stretch, so it expands and pushes LAYERS down, and the tall card pages force a scrollbar. Fix: every section `flex:none`; the **stretch goes at the BOTTOM of the panel**, not on the tree. After the flatten, with a single dataset, the left dock must not scroll.
2. **Section order/hierarchy is backwards today.** A `LAYERS` header is emitted ABOVE `RENDERING MODE`, reading as if LAYERS is the parent of the rendering controls. Correct order: **DISPLAY — 3D** (mode, blend, threshold, LUT) **then LAYERS** **then INTERACTION ▶ then SLICE ANIMATION ▶** (last two collapsed). `LAYERS` is NOT the parent of the rendering controls.

### LAYERS is five fixed rows — do NOT just re-home the current checkboxes
The spec's LAYERS block is exactly **five visibility rows**, and the panel must keep this shape whether or not each thing is loaded (layout stability is half the point — the panel must not change shape when the user loads a catalogue):

| Row | Backing state today | Notes |
|---|---|---|
| **Volume** | render-mode Volume/Isosurface state | + a 32×9 LUT strip on the right |
| **Cutting plane** | `actionShowCuttingPlane` (+ its opacity) | percentage (“50%”) right-aligned |
| **Contours** | lives in the Contours *tool*, not a dock checkbox today | expose as a visibility toggle bound to the **same** contour-shown state — do **not** duplicate the state |
| **Catalogue sources** | `actionShowCatalogueOverlay` | toggle bound to that action |
| **3D WCS axes** | the “Show 3D WCS Axes” **button** (`actionShowWcsAxes`) | expose as a toggle bound to the same action |

Rules:
- Bind each row to the **existing** state/QAction — a checkbox that mirrors the tool/button, never a second source of truth.
- If a row has **no state to reflect yet** (e.g. no contours computed, no catalogue loaded), show it **disabled with the reason** (tooltip / dim label), **never omit it**. Omitting a row makes the panel change shape when the user loads data, which breaks layout stability.
- The three that are not dock checkboxes today (Contours, Catalogue sources, 3D WCS axes) get **new checkbox widgets** in LAYERS wired to the same signals; the underlying tool/menu action stays the single source of truth (`setChecked` follows it, toggling it calls the action).

## Two small independent fixes (not part of the flatten)
3. **Inspector tab labels truncate** (“Proper…”, “Analy…”, “Provena…”). Three tabs at Lato 12px + 12px padding fit in 320px; something above steals width — almost certainly the **1/2/4 + Linked strip still inside `InspectorPanel`'s top row**. That strip leaves the Inspector in Step 1 (Fix 3 — it belongs in the centre toolbar). Once it's gone the tabs get full width; verify no truncation then. Do not let tabs truncate.
4. **Command bar shows 3 separate chips** — absolute path pill · “CUBE · 64×64×64” · “remote · full-res”. Spec is **ONE dataset pill**: type badge · filename · dims · load-state · ▼ (`#12a`). The absolute path already lives in Properties › Path. Collapse the command bar to the single pill; drop the standalone path pill (keep filename + dims + load-state + ▼). This is `VisivoTheme::makeViewerToolbar()` — the path-pill + the two tag chips.

## Step 2 carry-over (don't lose)
In the 2D pane the **colorbar overlaps the image's right edge** instead of sitting in its own gutter outside the data. This is Step 2 (2D pane geometry) territory — leave it there, but it's on the list.

## Reuse / constraints (from the handoff)
- Theme accessors `k*()` only; literal hex only for `#000004` viewport backgrounds.
- No new renderers/plots — the spectrum is `probePlotWidget` (`setupSpectrumPlot()`), currently floating; the 4-pane layout re-parents existing widgets.
- `ToolDialogStyle` (`inputCss/sectionCss/helperCss/addSection`) for the Step 6 inline forms.
- Keep user-facing strings verbatim as the spec quotes them.
- New `QSettings` key prefix `layout_v2/cube/…` for persisted layout (Step 4).
