# STEP 6 — Linked "ampio" (broader Linked) — SPEC

Status: **IMPLEMENTED — COMPLETE** (2026‑08‑25). P5·P2·P3·P1·P4‑camera·P4‑slice‑LUT·
P4‑moment‑LUT all done, each build + Codex clean. **Nothing deferred.** The per‑window
scoped‑id issue for cross‑window moment/image LUT was solved by matching on the
window‑independent product STEM ("moment0", "linewidth_fwhm"). Cross‑window colour syncs
colormap+scale+gamma+invert (range excluded, like the 3‑D volume path).

Original spec follows (author: pairing session, 2026‑08‑25).

This spec extends the existing **Linked views** feature. It is deliberately written
*after* reading the current implementation so the scope is only the genuine gaps, not
a rewrite. **Nothing here is implemented until approved.**

---

## 1. What Linked already does today (baseline — do NOT rebuild)

Master switch: `m_linkViews` (the toolbar **Linked** chip). Two per‑facet chips,
enabled only while `master = m_linkViews && m_paneCount > 1`:
`m_linkChannel`, `m_linkLut`. **Camera has no facet chip — it is always‑on when
Linked.** Echo loops are prevented by `m_applyingLinkedUpdate` (cross‑window) and
`m_applyingPaneDisplay` (intra‑window).

### 1a. Intra‑window (panes inside ONE window)
- **2D camera** — `propagate2DCamera(srcSlot)`: a pan/zoom on one **image product**
  pane (`m_products2d`) copies world focal x/y + parallel scale to every other visible
  image‑product pane. Same cube grid ⇒ direct copy. **Spectra and the 2D Slice pane are
  excluded.** Gated on `m_linkViews` only (no camera facet).
- **LUT** — `propagatePaneLutWithinWindow` / `propagatePaneRangeWithinWindow`: colormap
  + scale + gamma + invert + range follow panes showing the **same productId**. Gated on
  `m_linkViews && m_linkLut`. **Same‑product only** — never moment→slice, never across kinds.

### 1b. Cross‑window (the `s_cubeWindows` registry)
- **3D camera** — `broadcastLinkedCamera`: dir/up/offset/dist/scale/angle relative to
  each cube's data bounds (so different‑sized cubes still line up).
- **Channel** — `broadcastLinkedChannel`: matched by spectral value (velocity/frequency
  with unit conversion) else proportional. Gated on `m_linkChannel`.
- **Volume colour** — `broadcastLinkedColour`: 3D volume colormap + scale mode + gamma.
  Gated on `m_linkLut`.
- **Blend mode** — `broadcastLinkedBlendMode`: Composite / MIP / MinIP.

### 1c. Known deferred limitations (recorded during the Pane2DView work)
1. Linked‑2D uses the **raw focal**; it does not compensate for the per‑pane header/gutter,
   so identical focal+scale can frame slightly differently when panes differ in size.
2. **`fit()` resets the camera** — an auto‑fit (on mount/recompute) silently breaks the
   shared 2D view.
3. The **2D Slice pane is not part** of the 2D camera link (only image products are).

---

## 2. Scope of "Linked ampio" (what this step ADDS)

Five candidate pieces. Each is independently shippable; the spec proposes an order in §5.

- **P1 — Include the 2D Slice pane in the intra‑window 2D camera link.**
  Today only `m_products2d` panes sync. Add `ViewId::Slice2D` so a slice and a moment
  (same grid) pan/zoom together. (Fixes limitation §1c‑3.)
- **P2 — Make the 2D camera link survive `fit()`.**
  When Linked is on, a programmatic `fit()` on a receiver must re‑assert the shared
  camera state instead of resetting to its own bounds. (Fixes §1c‑2.)
- **P3 — Gutter‑correct the 2D camera copy.**
  Account for each pane's usable viewport (exclude the header strip) so identical world
  focal + scale frames identically regardless of pane size. (Fixes §1c‑1.)
- **P4 — Cross‑window 2D sync (camera + LUT).**
  Extend the `s_cubeWindows` broadcast so two windows each showing a 2D slice/moment
  share the 2D pan/zoom **and** the 2D LUT — today cross‑window only covers the 3D
  volume. Reuses the intra‑window primitives (`propagate2DCamera`, `propagatePaneLut…`).
- **P5 — A dedicated Camera facet chip.**
  Camera is currently always‑on when Linked. Add `m_linkCamera` + a chip so users can
  link LUT/channel WITHOUT camera (symmetric with the other facets). Off ⇒ P1–P4 camera
  paths no‑op; LUT/channel still sync.

---

## 3. Design decisions — RESOLVED (approved 2026‑08‑25)

- **D1 → Same‑grid only.** Cross‑window 2D camera syncs only when the two cubes share
  dimensions/WCS; otherwise no‑op. (WCS‑relative is a possible later follow‑up.)
- **D2 → Same‑product only.** LUT link stays keyed on productId, as today.
- **D3 → Camera facet gates BOTH 2D and 3D.** One "Camera" facet for all camera sync.
- **D4 → Scope + order confirmed.** Out of scope: pane‑composition mirroring, spectrum/PV
  camera links, a new grid window. Order: **P5 → P1 → P2 → P3 → P4**.

<details><summary>Original decision menu (for the record)</summary>

> These change the implementation materially; picking them wrong wastes work.

**D1 — Cross‑window 2D camera alignment across DIFFERENT cubes.**
Intra‑window panes share one grid, so world focal copies directly. Two *different* cubes
(different WCS/pixel scale) do not. Options:
  - (a) **World/WCS‑relative** — match by RA/Dec (or the shared celestial frame). Correct
    for co‑spatial surveys; needs a WCS transform and fails gracefully when frames differ.
  - (b) **Fraction‑of‑extent** — like the existing 3D camera (relative to data bounds).
    Always works, but "same feature" only lines up when the cubes are co‑registered.
  - (c) **Same‑grid only** — cross‑window 2D camera syncs only when the two cubes share
    dimensions/WCS; otherwise it's a no‑op (safe, minimal).
  *Recommendation: (c) for the first cut, (a) as a follow‑up.*

**D2 — LUT "cross‑view": same‑product only, or across products?**
Intra‑window LUT link is **same‑productId only** today. "LUT cross‑view" could mean:
  - (a) keep **same‑product only** (two panes on Moment 0 share; Moment 0 vs Moment 1 do not);
  - (b) link by **kind** (all moment panes share a colormap; ranges stay independent);
  - (c) **all 2D panes** share one colormap (range independent).
  *Recommendation: (a) now; expose (b) later only if requested. Sharing a RANGE across
  different products is almost always wrong (different physical units).*

**D3 — Does the Camera facet (P5) also gate the 3D camera?**
Today 3D camera is always‑on when Linked. If we add `m_linkCamera`, should it gate BOTH
2D and 3D camera, or only the new 2D paths?
  *Recommendation: gate both — one "Camera" facet for all camera sync, consistent mental model.*

**D4 — Scope ceiling.** Confirm we are NOT doing:
  - synchronising the pane *composition* itself (window A's layout mirrored in B);
  - linking spectra/PV cameras (only image/slice 2D cameras);
  - a new comparative‑grid window (the whole point is to link existing viewers).

</details>

---

## 4. Architecture notes (how it lands, once decisions are fixed)

- **Reuse, don't fork.** P1–P3 are edits to `propagate2DCamera` + the `fit()` path, not
  new machinery. P4 adds `broadcastLinked2DCamera` / `broadcastLinked2DLut` mirroring the
  existing 3D broadcasts, delegating to the same intra‑window appliers on the receiver.
- **Echo safety.** Cross‑window uses `m_applyingLinkedUpdate`; intra‑window uses
  `m_applyingPaneDisplay`. P4 must set BOTH on a cross‑window→pane apply so the receiver's
  pane change doesn't re‑broadcast. (This is the main new correctness risk — call out in review.)
- **Facet gating.** P5's `m_linkCamera` mirrors `m_linkChannel`/`m_linkLut`: a member +
  a chip in `refreshLinkedFacetChips()`, `enabled = master`, and every camera path early‑returns
  when off. Persist it next to the other facet defaults.
- **No‑receiver hint.** Reuse `showLinkNoReceiverHint` so "correct = no visible effect"
  (e.g. cross‑window 2D link with no matching pane open elsewhere) is explained once.

---

## 5. Proposed implementation order (each: build → Codex → deliver)

1. **P5 (Camera facet chip)** — smallest, unlocks clean gating for everything else.
2. **P1 (Slice pane in 2D camera link)** — one predicate widening + the fit guard's target set.
3. **P2 (fit survives link)** — re‑assert shared camera after a linked `fit()`.
4. **P3 (gutter‑correct focal)** — viewport‑aware copy; verify with unequal pane sizes.
5. **P4 (cross‑window 2D camera + LUT)** — the largest; do last, on top of the fixed intra‑window base.

Each piece is independently revertible. P4 is the only one touching the cross‑window
broadcast protocol.

---

## 6. Risks

- **Double‑echo on P4** (cross‑window apply re‑broadcasting) — mitigated by setting both
  guards; explicitly adversarially verified in review.
- **WCS alignment (D1a)** — if pursued, projection edge cases (CAR vs TAN, see the VLKB
  catalogue‑overlay note) can misalign; the same‑grid fallback (D1c) is the safe default.
- **`fit()` interactions (P2)** — fit is called from several paths (mount, recompute,
  maximise); the re‑assert must not fight a legitimate user fit when Linked is OFF.

---

## 7. Ask

Approve the scope (§2), answer **D1–D4** (§3), and confirm the order (§5). On approval I
implement piece by piece with a build + Codex review per piece, exactly as the previous steps.
