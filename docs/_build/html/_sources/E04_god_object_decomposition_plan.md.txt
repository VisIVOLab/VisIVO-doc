# E-04 — Decomposing the `vtkWindowCube` / `vtkWindowImage` god-objects

## Progress
- **Step 1 — RemoteSliceCache — DONE (2026-08-27), Codex-reviewed, app+ctest green.**
  Scope reality: the plan's `RemoteSliceSession` (fetch + prefetch + cache + display)
  turned out too coupled to extract safely without a running GUI — `applyRemoteSliceResult`
  touches ~20 window members (VTK actors, `ui->*`, the linked-view state machine), so it is
  fundamentally a window-display method. The genuinely **disjoint** slice — the LRU slice
  cache (`remoteSliceCache` + `remoteSliceCacheLru` + capacity) — was extracted instead into
  `src/gui/RemoteSliceCache.h`, a `template<class Slice>` LRU (so it carries **no** VTK/backend
  dependency and is unit-tested headless with a fake slice). `remoteSliceFetchesInFlight` stays
  on the window (fetch orchestration). 9 headless tests (`tests/test_remote_slice_cache.cpp`),
  behaviour parity verified. Codex found one Low bug (`touch()` on an absent key polluted the
  LRU → over-eviction) — fixed with a `contains` guard + a regression test. This is the safe,
  incremental first extraction the risk-controls below mandate.

- **Step 2 — LinkedWindowRegistry — DONE (2026-08-27), Codex-reviewed, app+ctest green.**
  Scope reality (again narrower than the plan's optimistic full `LinkController`): the linked-view
  broadcast/apply methods are deeply coupled to the window's VTK state (camera vectors, channel
  index, LUT), and the echo-suppression state machine (`m_applyingLinked*` RAII guards,
  `m_lastLinkedChannel` dedup, the async `m_linkedRemoteRequestIds` token set) is entangled with
  the apply paths — too risky to move compile-only. The **genuinely-disjoint** slice — the global
  `static QList<QPointer<vtkWindowCube>> s_cubeWindows` registry — was extracted into
  `src/gui/LinkedWindowRegistry.h` (`template<class W>`, VTK-free, headless-testable). `s_cubeWindows`
  → `static LinkedWindowRegistry<vtkWindowCube> s_registry`; the 8 broadcast snapshots, the ctor
  `add`, and the dtor `remove` now go through it. `snapshot()` returns the QPointer list **by value**,
  preserving the original mid-iteration auto-null safety verbatim. 6 headless tests
  (`tests/test_linked_window_registry.cpp`), including the safety-critical destroyed-peer-auto-nulls
  case. Codex confirmed behaviour parity (copy semantics, static-duration, no missed uses).
  The echo-guard state machine remains on the window — a later, higher-risk step (needs the
  broadcast/apply dispatch restructured behind a callback the test can stub).

- **Shared-helper lift: spectral-unit conversion — DONE (2026-08-27), Codex-reviewed, app+ctest green.**
  The pure functions `parseSpectralUnit` / `convertSpectralValue` (+ `SpectralUnit`) that drive
  linked-channel velocity matching (km/s↔m/s, the mhz-vs-velocity disambiguation, astropy `m.s**-1`
  spelling) lived untested in the anonymous namespace of `vtkWindowCube_Link.cpp`. Lifted verbatim
  into `src/gui/SpectralUnitConvert.h` (namespace `spectral`, `inline`), `u"..."_s`→`QStringLiteral`,
  and locked under 14 headless tests (`tests/test_spectral_unit_convert.cpp`). Zero behaviour change
  (Codex-confirmed). This is the "safe, independent shared-helper dedup" the section below anticipated.

- **Shared-helper lift + de-dup: region hit-testers — DONE (2026-08-28), Codex-reviewed, app+ctest green.**
  `CubeRegionGeom`'s four pure ROI hit-testers (`pointInBox` / `pointInCircle` / `pointInPolygon` /
  `pointInAnnulus`) were made `inline` in the header (moved verbatim from the .cpp, which keeps only
  the VTK-dependent `buildAnnulusFill`) so they can be unit-tested without linking VTK — 15 headless
  tests (`tests/test_cube_region_geom.cpp`). This also surfaced and **removed a real duplicate**:
  `vtkWindowCube_Lasso.cpp` had its own divergent `pointInPolygon` (used for the kinematic-lasso
  per-channel 2-D mask refinement); it now calls the shared, tested one. Codex confirmed the two were
  behaviour-equivalent (the `+0.5` pixel-centre shift is a uniform translation of point+vertices; the
  ≥3-vertex guard is already enforced by the caller; the horizontal-edge divide is short-circuited),
  so the de-dup is zero-behaviour-change, and the arg-order swap `(x,y,verts)`→`(verts,x,y)` is correct.

- **Type + helper lift: spectral-axis inference — DONE (2026-08-28), Codex-reviewed, app+ctest green.**
  The nested `SpectralAxisKind` enum + `SpectralAxisDescriptor` struct were lifted verbatim out of the
  `vtkWindowCube` class body into `src/gui/SpectralAxisTypes.h`; the class keeps
  `vtkWindowCube::SpectralAxisKind` / `::SpectralAxisDescriptor` working via `using` aliases (zero call-site
  churn across the 25 references). The FITS-CTYPE classifier `inferSpectralAxisDescriptor` (FREQ/VRAD/VOPT/
  FELO/VELO/CHAN → kind, with its order-sensitive branches) + `spectralAxisKindLabel` moved to
  `src/gui/SpectralAxisInfer.h` (namespace `spectralaxis`, inline); the trivial `upperCtype`/`cleanAxisUnit`
  were inlined so the header is VTK/libwcs-free. 14 headless tests (`tests/test_spectral_axis_infer.cpp`)
  lock the conventions (incl. FELO-before-VELO ordering, CHAN→non-physical, case-insensitivity, unit
  trimming). Codex confirmed the using-alias substitution, the inlining, and the branch logic are all
  behaviour-identical, with no ODR concern. First step to lift a nested god-object TYPE, not just a helper.

**Status:** steps 1–2 + spectral, region-geom & spectral-axis lifts done; steps 3–4 (+ the link echo-guard) pending. **Why incremental, not a big-bang commit:**
`vtkWindowCube` is ~22k LOC with 205 member variables and 625 methods, all shared
mutable state across the partial `.cpp` files. A decomposition that *compiles*
but is not runtime-verified would very likely regress subtle behavior
(initialisation order, signal/slot wiring, guard-flag interplay across the
linked-view state machine). During an autonomous run without a running GUI to
test against, shipping that is higher-risk than valuable. This document is the
concrete, low-risk path to do it incrementally *with* tests.

## The problem (measured)
- `vtkWindowCube.h`: 205 members, 625 method decls; `.cpp` family split into
  `_Setup / _Wcs / _Region / _Lasso / _Link / _RemoteSlice / _RegionIO`.
- The split reduced file size but NOT coupling: every partial can read/write all
  205 members. `vtkWindowImage` is the same shape (~11k LOC).
- No unit test exercises any of this (the C++ suite is headless Qt-only).

## Strategy: extract collaborators behind narrow interfaces, one at a time
Each step: (1) define a small class owning a *disjoint* slice of state + the
methods that touch only it; (2) give `vtkWindowCube` a member of that class and
forward to it; (3) add a headless unit test for the extracted class over a fake
window registry; (4) build + ctest green; (5) Codex review; (6) commit. Never
extract two collaborators in one commit.

### Extraction order (least-coupled first)
1. **`RemoteSliceSession`** — owns the remote-slice fetch/prefetch/cache state:
   `currentRemoteSliceRequestId`, `activeRemoteSliceRequests`,
   `remoteSliceFetchesInFlight`, `idlePrefetchTimer`, `prefetchAllowedForNextResult`,
   `m_linkedRemoteRequestIds` (added in E-07/M6). Interface:
   `request(index)`, `applyResult(result)`, signals `sliceReady`, `linkedBroadcast`.
   The window keeps only the display wiring. This slice is already isolated in
   `vtkWindowCube_RemoteSlice.cpp` and has the clearest boundary — do it FIRST.
2. **`LinkController`** — owns the cross-window linked-view state:
   `m_linkViews/m_linkCamera/m_linkChannel/m_linkLut`, the echo guards
   `m_applyingLinkedUpdate/m_applyingLinkedResult`, `m_lastLinkedChannel`, the
   `s_cubeWindows` registry, and the `broadcast*`/`applyLinked*` family
   (currently `vtkWindowCube_Link.cpp`). Interface: `broadcastCamera/Channel/Lut`,
   `applyCamera/Channel/Lut`, `registerWindow/unregister`. This is the
   correctness-hazardous state machine E-07 hardened — extracting it behind a
   testable interface is the biggest long-term win, and it's where a headless
   fake-registry test (see below) pays off most.
3. **`RegionController`** — owns region-stats/region-IO/overlay state
   (`vtkWindowCube_Region.cpp` + `_RegionIO.cpp`).
4. **`LassoController`** — owns the kinematic-lasso interaction state
   (`vtkWindowCube_Lasso.cpp`).

After the four collaborators exist, `vtkWindowCube` becomes a thin composition
root wiring them to the VTK render widgets — a few hundred lines, not 22k.

### The test that unlocks it (do before step 2)
A headless test of the linked-view state machine over a **fake window registry**:
construct N `LinkController`s sharing a registry, drive `broadcastChannel` from
one, assert the others receive exactly one `applyChannel` (no echo, no
ping-pong), including the async-clamped-index case E-07/M6 fixed. This locks the
most hazardous behavior BEFORE the refactor moves it, so the refactor is
verifiable without a GUI.

### Shared-helper dedup (safe, independent, can land anytime)
The `wcscon` extern declaration + WCS constants are copy-pasted across 5
catalogue files (`vtkWindowCatalogue3D*.cpp`). Extracting them into a shared
`CatalogueWcsUtil.h` (with `galacticToSupergalactic`, added in R-13) is a
zero-behavior-change coupling reduction that can be done independently of the
god-object work.

## Risk controls
- One collaborator per commit; build + ctest + Codex between each.
- Keep the extracted class's members `private` with the window as a `friend`
  only where a first cut needs it, then tighten.
- Do NOT change guard-flag semantics during extraction — move them verbatim
  (E-07 already made them RAII/token-based); behavior parity is the acceptance
  bar, measured by the new state-machine test.
