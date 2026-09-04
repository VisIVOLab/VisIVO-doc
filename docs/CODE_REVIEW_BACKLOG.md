# VisIVO-next — Multi-expert code review & investment backlog

**Date:** 2026-08-26 · **Branch:** `skava`
**Method:** parallel read-only review by four specialist lenses — observational astronomer, cosmologist / LSS, radio astronomer (HI/SKA), software engineer.
**Status:** astronomer ✅ · cosmologist ✅ · radio astronomer ✅ · software engineer ✅ — **pool complete**.

This file consolidates the reviews into a **prioritized backlog**. Each item has an ID, severity, rough effort, the domain(s) that raised it, evidence (`file:line`), the fix, and why it matters. Items flagged by **more than one expert independently** are marked ⭐ and ranked first.

---

## ✅ Implementation status (autonomous run, 2026-08-26)

Worked through in priority order (correctness → robustness → cosmo → validation → strategic). Every item below was tested and Codex-reviewed with fixes applied; backend suite **747 passed**, full C++ app + standalone ctest **green**.

**DONE:** E-01, E-02, R-01, R-02, R-03, R-04, R-05, R-06, R-07, R-08, R-09, E-03, E-05, E-06, E-08, E-09, R-11, R-12, R-13, R-14 (distance-type + distance-mode; infall correction deferred), R-15, R-16, R-17, R-18, R-19, R-20, R-21 (already-wired via R-02), **R-22 (copilot, full incl. GUI)**, E-04 (first safe extraction `CatalogueWcsUtil.h` + decomposition plan `docs/E04_god_object_decomposition_plan.md`). **R-10 partial** (honest `squeeze_to_3d` warning; label renames deliberately skipped).

**Bugs found & fixed via the new tests/CI** (a bonus of E-01/E-08): `resolve.py` `_asyncio` NameError (endpoint crashed on every call); `test_memmap_cleanup._force_stale` monotonic-origin flake (CI-only failure).

**DEFERRED (with reason):**
- **R-14 infall correction** — the distance-*type* (luminosity/angular→comoving) and distance-*mode* (real-space comoving vs redshift-space cz/H₀) parts are DONE (see fourth pass). The remaining linear-theory *infall correction* (using the loaded velocity field to move redshift-space galaxies to their real-space positions) is deferred: it is speculative, model-dependent, and needs the reconstruction cross-referenced per source.
- **R-10** (minor mislabels) — PARTIAL: the one real honesty gap (silent non-degenerate STOKES/FREQ plane-0 pick in `squeeze_to_3d`) now warns loudly; the rms/EW/CD-matrix items are intentionally left as-is (pure label renames on already-correct values that would break the wire contract with the desktop client for no numeric gain).
- **E-06** (broaden provenance `record_history`), **E-08** (testing blind spots) — incremental hardening.
- **E-04 full decomposition** — deferred to the documented incremental plan: a compile-green-but-runtime-unverified decomposition of a 22k-line/205-member class would likely regress working behavior, so the safe first extraction + plan were delivered instead.

---

## 🔬 Second expert review (post-remediation) — 2026-08-26

The same four-lens panel was re-run on the remediated code to (a) verify the fixes are actually correct and (b) find fresh bugs, including in the new code. Then the findings were themselves Codex-reviewed and fixed.

### Verified correct
All claimed fixes hold. Highlights the panel independently confirmed: **R-13 galactic→supergalactic rotation** matches the canonical de Vaucouleurs matrix to 3.8e-8 with `det=+1` (a proper rotation — no sky mirror); **R-16 streaming divergence** is byte-exact vs `np.gradient` (±1-halo boundary handling correct); **R-19 HI constants** (1.104e24 / 2.356e5) are the standard optically-thin coefficients; **E-02** contract drift-detection genuinely locks the C++/Python header sets through the manifest; **E-03/E-07** admission/guard logic is leak-free; **R-22** copilot honesty (`tool_calls_recorded`), termination, and C++ panel threading are sound.

### New issues found — and FIXED this pass
| Issue | Raised by | Sev | Fix |
|---|---|---|---|
| Copilot query defeated its own DoS controls: timeout released the session slot while the thread kept running, and tool compute bypassed admission | Engineer | HIGH | Dedicated assistant semaphore + hold-both-slots-until-thread-settles (E-03 pattern) + pre-acquire cancel guard |
| `is_per_beam_bunit` rejected `Jy beam-1` (FITS spelling); then an unanchored fix wrongly matched `mJy/beam` (→1000× flux) | Astro / Codex | HIGH | Anchored `^jy/?beam` — accepts `Jy beam-1`, rejects mJy/uJy (falls to honest flux_jy=None) |
| HI `hi_mass_msun` integrated the WHOLE field (noise + all sources) with masking off | Radio | HIGH | Added `threshold_auto` (threaded to worker/router/tool/provenance) + response `warnings`/`n_pixels_summed`/effective mask state |
| Aperture non-finite sum fell back to the WHOLE-IMAGE sum (order-of-magnitude-wrong flux) | Astro | MED | Return NaN (honest) instead |
| Client & server assistant timeouts both 120 s → client abort races the server 504 | Engineer | MED | `/assistant/query` client timeout → 180 s (> server budget) |
| R-11 chirality check missed CD/PC-matrix parity flips (CDELT-only) | Cosmo | LOW-MED | Warn on negative spatial CD/PC determinant |
| HI velocity unit `"m / s"` (astropy spaced) wrongly rejected | Astro | LOW | Normalise whitespace before comparison |
| Fast line-width FWHM could report 0 where weighted M0 ≤ 0 | Astro/Radio | LOW | Blank (NaN) where M0 ≤ 0 |
| R-16 in-memory path was float32 vs streaming float64 (docstring overclaimed byte-identical) | Cosmo | LOW | In-memory now float64 → truly bit-identical |
| E-07 linked token leaked when a fetch was deduped (no watcher) | Engineer | LOW | Consume the token in the dedup early-return |
| Provider rebuilt per request; question unbounded; truncated answer unmarked | Engineer | LOW | Module-cached provider; `max_length=4000`; truncation marker on `stop_reason=max_tokens` |

### New issues DEFERRED (larger / feature-level)
- **Distance-type framework** (Cosmo MED): luminosity/angular-diameter distance columns are placed as the comoving radius (D_L too far by (1+z)²). Real correctness gap; belongs with **R-14** (redshift-space/RSD).
- **PA-aware beam deconvolution** (Radio MED): `_deconvolve_gaussian_beam` subtracts per-axis and keeps the fitted PA; use the full Wild/Condon PA-aware form before quoting deconvolved sizes.
- **Non-square-pixel solid angle** (Radio LOW): `beam_area_pixels` uses mean pixel scale²; use `|CDELT1·CDELT2|` for non-square pixels.
- **Divergence density faH calibration** (Cosmo): `velocity_density_worker` returns raw ∇·v; either apply δ = −∇·v/(faH) or mark uncalibrated (like R-15).
- **Streaming/Dask parity test** (Radio): assert numba ≈ streaming ≈ dask ≈ numpy M1/M2 on a non-uniform-Δv cube so the four paths can't silently diverge.
- **Frame-conversion perf** (Cosmo): `applyFrameToEntries` is O(N) libwcs per toggle; precompute a vectorised rotation for multi-million-row catalogues.
- **Bulk-flow / velocity-dipole diagnostic** (Cosmo): low-cost high-impact cosmic-flows readout now that Planck18 constants are in place.
- **Per-endpoint header-contract assertion** (Engineer minor): `test_required_headers_still_emitted` checks "emitted somewhere", not "by the right endpoint".
- Original deferred still stand: **R-14, R-10, E-04-full**.

---

## 🔧 Third pass (post-second-review remediation) — 2026-08-26

Cleared the second-review deferrals and the remaining feature slices, each Codex-reviewed:
- **PA-aware beam deconvolution** (Wild 1970 α/β/γ form + WCS BPA sky→pixel conversion, `deconvolved_pa_deg` exposed), **non-square-pixel solid angle** (`spatial_pixel_area_arcsec2()` exact via `proj_plane_pixel_area`), **distance-type framework** (luminosity/angular→comoving conversion, `DistanceKind`), **divergence density calibration** (`_linear_density_factor` + `X-Visivo-Calibrated:0` honesty header), **streaming/Dask parity test** — all DONE.
- **R-17** (vorticity streamline colouring): `chkColorByVorticity` (default OFF) computes `mag(Vorticity)` via `vtkArrayCalculator` on the shared `makeStreamActor`, toggle repaints (`rebuildLodActors()`+`Render()`).
- **R-20** (multi-field compare): `velocity_difference_worker` + `POST /v1/velocity/difference_bin` → `|v_a−v_b|` scalar grid. Codex follow-up applied: per-field `box_mpc` with a mismatch warning, unit/frame-mismatch warnings, generic `X-Visivo-Range-Min/Max` headers (not the ∇·v Div family), `X-Visivo-Warnings` forwarded (latin-1-sanitised). Memory limitation (loads both full fields pre-LOD, like the density worker) documented in the worker docstring; streaming/slab path deferred.

## 🌌 Fourth pass (residual backlog, unsupervised) — 2026-08-26

- **R-14** (redshift-space distance mode): new `CatalogueDistanceMode {Comoving, RedshiftSpace}` on the catalogue schema (so scene, hover, table, and shells stay consistent) + a "Distance" selector in the 3D catalogue viewer. `hubbleDistanceMpc(z)=cz/H₀` places sources at the naive redshift-space Hubble distance (bypassing measured/model/override distances); `entryDistanceMpc` falls back to the real-space logic when a source has no usable redshift. Factored the recompute+rebuild out of the Planck18 cosmology branch into `recomputeScenePositionsAndRebuild()` (Codex-flagged P2 fixed there: it now reapplies the shells toggle and refreshes the table so both survive a mode/cosmology recompute). 3 QTest slots (cz/H₀ value + differs-from-comoving, bypasses-override, no-redshift-fallback); C++ suite 30 passing. The *linear-theory infall correction* remains deferred (speculative/model-dependent). Wording clarified per Codex P3: the two-mode shift is a peculiar velocity only against an independent real-space distance.
- **R-10** (partial): `squeeze_to_3d` now warns loudly when it drops a non-degenerate STOKES/FREQ axis (silently keeping plane 0); the rms/EW/CD-matrix label items are intentionally left (renames on already-correct values, wire-contract risk).

---

## 0. Executive summary

The **core numerical machinery is unusually careful** — projection-agnostic WCS via astropy, blank-aware moments, Chan's algorithm for out-of-core variance, two-pass streaming M2 that dodges catastrophic cancellation, baseline subtraction that refuses to run without line-free channels, a real (not decorative) SoFiA-2 validation harness, and a genuine vector-field / cosmic-flows pipeline (streamtracer + basins of attraction).

The real weaknesses are **not** in the hard algorithms — they are in **flux calibration, spectral-frame/unit bookkeeping, and scientific honesty of a few outputs**. These are exactly the areas an "AI copilot" would *amplify* if built first, which is why the copilot is scheduled **after** the correctness fixes.

On the **engineering** side the bones are good — a clean versioned per-endpoint FastAPI backend (121 endpoints, 57 pydantic models), serious out-of-core work with dedicated memmap-cleanup tests, tiered concurrency with backpressure, and RAII guards in the newest linked-view code. The debt is concentrated in **two ~11–22k-line GUI god-objects**, a **client-server contract that is out-of-band and unversioned** (the exact wound that already dropped endpoints once), **no CI test gate**, and **compute that cannot be cancelled or time-bounded**.

**Top fixes before anything else:**
- **Cheap correctness (wrong physical numbers):** ⭐ **R-01** (flux not beam-corrected, off 10–50×), ⭐ **R-02** (`rest_freq_hz` dead / no SPECSYS → Hz where km/s expected), **R-03** (moment-2 is variance not dispersion), **R-08** (no fit uncertainties).
- **Cheap engineering (protect what exists):** **E-01** (wire a CI `pytest`+`ctest` gate — 757 tests are local-only and can rot), **E-02** (lock the client↔backend contract: snapshot OpenAPI + centralize `X-Visivo-*` header names).
- **Blockers for the copilot:** **E-03** (cancellable, time-bounded compute) + R-01/R-02/R-05/R-08/R-11/R-12.

---

## 1. Convergent findings (⭐ raised by ≥2 experts) — HIGHEST PRIORITY

### R-01 ⭐ — Point-source flux is not beam-corrected and is mislabeled `_jy`
- **Severity:** HIGH · **Effort:** XS (~1 line + guard) · **Domains:** Astronomer, Radio
- **Evidence:** `backend/app/compute/photometry.py:282` (`worker_gauss_fit_2d`: `integrated_flux = amp·2π·σx·σy`, returned as `integrated_flux_jy` at `:305-306`); `photometry.py:106,111-113` (`worker_aperture_photometry`: `flux = raw_sum − bkg·npix` returned as `flux_jy`). Correct formula already exists in the repo at `astrometry.py:139` and `validation/lasso_vs_sofia.py:50`.
- **Failure:** On Jy/beam interferometric maps every reported aperture/Gaussian flux is in Jy·pixel/beam — wrong by the beam area in pixels (10–50× for a well-sampled beam), while labeled `_jy`. Any derived flux/mass is off by that factor.
- **Fix:** Shared `to_jy(sum_or_integral, beam_area_px, bunit)` helper; divide by `(π/4ln2)·BMAJ·BMIN/pixΔ²` when BUNIT is Jy/beam; otherwise return an honest unit label (`bunit·pix`) instead of `_jy`. Refuse/annotate when beam is unknown.

### R-02 ⭐ — `rest_freq_hz` ignored; no velocity convention / reference frame (SPECSYS)
- **Severity:** HIGH · **Effort:** M · **Domains:** Astronomer, Radio
- **Evidence:** `backend/app/routers/spectral.py:58,441` declare `rest_freq_hz`; `_compute_linewidth` called at `:454-458` **without** it. Radio convention exists only in `kinematic_model.py:196-218` (radio only; optical/relativistic absent). SPECSYS/VELREF (LSRK/bary/topo) read nowhere. Spectral stacking `spectral.py:407` assumes identical grids with zero frame reconciliation.
- **Failure:** HI FREQ cube → FWHM/EW and line-fit centre come back in **Hz**, not km/s, while the API shape implies velocity; the rest frequency the user passed does nothing. Stacking/comparing cubes silently mixes LSRK and barycentric velocities (tens of km/s for HI).
- **Fix:** Thread `rest_freq_hz` + a convention selector (radio/optical/relativistic) through `worker_linewidth` and the line-fit/moment velocity axis (reuse `_spectral_axis_kms`); read and **record + warn** on SPECSYS; block stacking of mismatched SPECSYS. Reject unused params loudly rather than silently.

### R-04 ⭐ — Moment M1/M2 weighting differs between code paths on non-uniform axes
- **Severity:** LOW-MED · **Effort:** S · **Domains:** Astronomer, Radio
- **Evidence:** Numba path (default for nz>64, `moment.py:309,331`) weights by raw I (`:107-108,137-138`); numpy fallback + Dask weight by I·Δv (`:320-326,636-648`). Self-documented at `moment.py:456-458`.
- **Failure:** Identical for uniform CDELT (common case) but path-dependent M1/M2 on genuinely non-uniform spectral axes (optical-velocity/FELO grids). Same computation should not depend on which backend answered.
- **Fix:** Unify on the I·Δv weighting across all paths (or document as an explicit, tested invariant).

---

## 2. Radio-astronomy correctness

### R-03 — Moment-2 is variance σ², not velocity dispersion σ
- **Severity:** MED-HIGH · **Effort:** S · **Domain:** Astronomer (Radio concurs)
- **Evidence:** `moment.py:174,328-351`, streaming `:502-512`, unit `spectral_unit^2`. UI already honestly labels it "Moment 2 (Variance – NOT dispersion)" (`src/gui/MomentMapController.cpp:295,483`, `RemoteMomentWindow.cpp:118`) — but there is **no** way to get the standard dispersion map.
- **Failure:** Any side-by-side with CASA `immoments(2)` / CARTA / SoFiA / literature disagrees by a square, silently.
- **Fix:** Add a dispersion output `sqrt(max(M2,0))` (km/s) as default or selectable variant; keep variance available.

### R-05 — Beam model is single-plane; no per-channel beam, no primary-beam correction
- **Severity:** MED-HIGH · **Effort:** M · **Domain:** Radio (Astronomer concurs)
- **Evidence:** `BeamMetadata` reads one BMAJ/BMIN/BPA from the primary header (`fits_dataset.py:54-57`). No CASA per-plane `BEAMS` HDU. No PB response / `pb.fits` / PBCOR anywhere (grep-confirmed absent). Deconvolved PA keeps the fitted PA (`photometry.py:174`, "simplified").
- **Failure:** Wideband ALMA/SKA cubes with a varying beam get channel-to-channel wrong beam-area (→ wrong flux); wide-field/mosaic fluxes are PB-uncorrected (biased toward field edges); deconvolved PA wrong when beam PA ≠ source PA.
- **Fix:** Read the `BEAMS` table for per-channel beam; accept an optional PB cube and divide; propagate beam PA into deconvolution.

### R-06 — Higher-order moments (3–10) have no non-Dask streaming path → OOM
- **Severity:** MED · **Effort:** M · **Domain:** Radio
- **Evidence:** `worker_moment` streams only orders 0/1/2 (`moment.py:764`); 3–10 go through `_moment_map_from_array` which materialises the full subset float32 (`:768`) + float64 temporaries (`:361-405`). Only Dask covers 6/8/10 OOC (`dependencies.py:144`).
- **Failure:** With Dask off (single-node default below `_DASK_MIN_BYTES`), a peak/kurtosis map on a >RAM cube balloons to several × the cube and OOMs.
- **Fix:** Add a streaming slab path for orders ≥3 (or route ≥3 through Dask even below the byte threshold).

### R-07 — Per-pixel Gaussian line-width map is a Python `curve_fit` double loop
- **Severity:** MED · **Effort:** L · **Domain:** Radio
- **Evidence:** `spectral.py:113-152` — one `scipy.curve_fit` per pixel (700×700 ≈ 5×10⁵ fits). SNR pre-cut helps on sparse fields (`:116`) but a bright extended HI disk pays full cost.
- **Failure:** Not SKA-scale; slow / non-interactive on dense bright cubes.
- **Fix:** Vectorised/analytic moment-based width as a fast default; reserve full fit for flagged pixels; consider GPU/`numba`. Do **after** R-01/R-02/R-05.

---

## 3. Scientific-honesty / uncertainty

### R-08 — No fit uncertainties propagated (covariance discarded)
- **Severity:** MED · **Effort:** XS · **Domain:** Astronomer
- **Evidence:** `curve_fit` covariance discarded in `worker_line_fit` (`spectral.py:614`) and captured-but-unused in `worker_gauss_fit_2d` (`photometry.py:257`).
- **Failure:** Peak/centre/FWHM/integrated-flux/PA returned as bare point estimates — not measurements; can't be cited.
- **Fix:** `perr = sqrt(diag(pcov))`; return error bars for every fitted quantity. Cheap, high value — **prerequisite for a credible copilot** (a copilot must state uncertainty).

### R-09 — Aperture-photometry uncertainty is a counts/Poisson formula
- **Severity:** MED · **Effort:** S · **Domain:** Astronomer
- **Evidence:** `photometry.py:109`: `flux_err = sqrt(|sum| + npix·|bkg|)` — assumes photon counts; dimensionally meaningless for Jy/beam and ignores intra-beam correlation.
- **Fix:** `σ_flux ≈ rms_local · √(N_pix/N_beam)` (correlated-noise form), rms from the annulus.

### R-10 — Minor mislabels
- **Severity:** LOW · **Effort:** XS · **Domain:** Astronomer
- `astrometry.py:126` reports `np.std` as `"rms"` (it's std about the mean). `spectral.py:44-49` `EW=M0/peak` is a width proxy, not a spectroscopic equivalent width. `squeeze_to_3d` (`fits_dataset.py:157-159`) drops extra axes via `arr[0]` — a non-degenerate Stokes axis silently keeps plane 0 with no guarantee it's Stokes I. `fits_dataset.py:283-287` raw `spacing/cdelt` default to 1.0 for CD/PC-matrix-only headers (mitigated by `effective_cdelt`, but raw consumers would be wrong).

---

## 4. Cosmology / LSS

### R-11 — No WCS / handedness on the velocity grid → silent chirality (mirror) risk
- **Severity:** HIGH (for cosmo use) · **Effort:** M · **Domain:** Cosmologist
- **Evidence:** `backend/app/velocity_field.py:104-113` maps `data[0..2]→+x/+y/+z` in raw order, ignoring WCS/CDELT sign/axis flip; frame literally `"supergalactic (assumed)"`, `box_mpc` user-supplied.
- **Failure:** A reconstruction with a flipped axis or opposite SG handedness produces **mirror-imaged** streamlines/basins/vorticity with no warning — plausible-looking but reversed physics.
- **Fix:** Honor CDELT signs / CTYPE / handedness keyword, or at minimum validate + warn. Do **before** any copilot narrates flow figures.

### R-12 — Local C++ "Planck18" cosmology is mislabeled and ~1% off
- **Severity:** MED · **Effort:** S · **Domain:** Cosmologist
- **Evidence:** `src/gui/Catalogue3DParser.h:162-184` hardcodes `H0=67.74, Ωm=0.3089, ΩΛ=0.6911` (= **Planck 2015**) but the UI labels this branch "Planck18 (local)" (`vtkWindowCatalogue3D_Data.cpp:166-199`); non-Planck18 models defer to the true-astropy backend → inconsistent distances between models the user thinks are reference.
- **Fix:** Route the local "Planck18" branch through the astropy backend (or fix constants to true Planck18); label the local integrator honestly.

### R-13 — Supergalactic frame absent from catalogue coordinate frames
- **Severity:** MED (highest cosmo value/effort ratio) · **Effort:** S-M · **Domain:** Cosmologist
- **Evidence:** Catalogue frame enum is only `{FK5_J2000, Galactic}` (`src/gui/vtkWindowCatalogue3D.h:192`); the velocity field's native frame is supergalactic. Cannot show an RA/Dec galaxy catalogue in the flow field's SG frame.
- **Fix:** Add `Supergalactic` to `CoordFrame`, reuse the existing `wcscon` transform path already used for Galactic. **Biggest scientific payoff per line of code** for cosmic-flows.

### R-14 — Redshift→distance ignores peculiar velocity / redshift-space distortion
- **Severity:** MED (partly inherent) · **Effort:** M-L · **Domain:** Cosmologist
- **Evidence:** `vtkWindowCatalogue3D_Data.cpp:174-179` uses comoving distance directly; no `cz/H₀` mode, no infall/RSD correction. Mitigated: measured distances preferred when present (`Catalogue3DParser.h:298-321`).
- **Fix:** Distance-mode selector (cz/H₀, comoving, measured); optional linear-theory infall correction using the loaded velocity field itself. This is what makes it a *cosmic-flows instrument*, not a viewer.

### R-15 — "Angular power spectrum" is an uncalibrated 2-D image FFT, not a Cₗ
- **Severity:** LOW-MED · **Effort:** M · **Domain:** Cosmologist
- **Evidence:** `backend/app/compute/cosmology.py` — Hann-windowed |FFT|² azimuthal average, no window-power normalization, no beam/mask deconvolution, no mode-mixing; docstring ℓ formula (`:43`) contradicts code (`:118`, code is correct).
- **Fix:** Either honestly rename it a "relative texture spectrum," or add window/mask deconvolution to make it a real Cₗ.

### R-16 — Full-array (`memmap=False`) loads for velocity summary/grid/density
- **Severity:** LOW · **Effort:** M · **Domain:** Cosmologist
- **Evidence:** `velocity_field.py` reads the whole field (`memmap=False`); `velocity_grid_worker`/`velocity_density_worker` compute over the full array pre-LOD. Only `subgrid` is memmapped.
- **Fix:** Out-of-core divergence/magnitude for >256³ fields. Bounded today for CF4 64³/256³.

### R-17 (feature) — Vorticity / shear / tidal-tensor cosmic-web classifier
- **Severity:** — · **Effort:** LOW · **Value:** HIGH (LSS) · **Domain:** Cosmologist
- **Evidence:** VTK vorticity is merely turned **off** (`vtkWindowVectorField.cpp:520`); `vtkGradientFilter` already imported.
- **Opportunity:** Expose curl/shear + ∇v eigen-decomposition (T-web/V-web: knot/filament/sheet/void). Largely already reachable — standard cosmic-web classifier at low effort.

---

## 5. Validation / paper hardening

### R-18 — Lasso-vs-SoFiA validation conflates segmentation with detection
- **Severity:** MED (blocks paper credibility) · **Effort:** S · **Domains:** Radio, Astronomer
- **Evidence:** Seed = SoFiA's own peak voxel (`validation/lasso_vs_sofia.py:92-94`) → measures *segmentation agreement given a correct seed*, not completeness/reliability. README headline (">91% of the SoFiA mask") conflates the two. Sensitivity sweep reports only completeness, not reliability (`lasso_sensitivity.py:34-38,64-73`); lowering nσ 2.5→1.5 to hit "97% completeness" floods into noise with reliability computed nowhere. One cube, 5 sources, no error bars.
- **Fix (to be defensible):** (a) report reliability/Dice at **every** nσ in the sweep; (b) seed-robustness (perturb the click); (c) a second cube; (d) flux uncertainty (needs R-08/R-09). **Keep framing as interactive segmentation, not blind detection.**
- **DONE (a)(b)(c):** the sweep reports completeness, reliability AND Dice at every setting and writes committed `out/sensitivity*.{md,csv}` tables; `lasso_seed_robustness.py` perturbs the click in an interior and a stress stratum; a second cube (WALLABY/ASKAP — different instrument, projection and spectral convention, same SoFiA parameters) is in the registry `validation/datasets.py`, selected with `--dataset`. The second cube changed two claims: `reach` is a cost budget whose default does not transfer between cubes (best Dice 0.76 at reach 32 vs 0.61 at the default 8) — and, measured over all 10 sources, cannot be defaulted from header metadata at all: two auto-rules (growth-curve knee, shell-brightness stop) both lose MORE Dice than the fixed default, and no single fixed value serves both cubes (per-cube optima 8 and 32). It is irreducibly an interactive control, and smoothing-limited faint sources do not segment at any setting (best Dice 0.30) — reported as the boundary of the technique. Click-invariance is a property of compact well-connected sources (Dice CV 0.00–0.18) and degrades on a large low-surface-brightness one (0.36–0.51). **(d) DONE:** integrated fluxes carry uncertainties estimated empirically rather than modelled — each mask's stencil, and for the comparison the SIGNED stencil (+1 lasso / −1 SoFiA), dropped at up to 200 positions lying FULLY outside a dilated SoFiA mask, with σ the scatter of the weighted sums. The analytic rms·√(N·F/B)·Δv form was implemented first and abandoned after review: it assumes a mask many beams across (WALLABY's faint masks are 32–159 voxels against a 28.5-px beam), one cube-wide spectral factor for an irregular 3-D mask, and — for a DIFFERENCE — that an unsigned voxel count stands in for a covariance double sum over SIGNED weights. Measured against the empirical scatter it overestimates σ_Δ by ~1.6–2.3× on HGC 44 and ~1.3× on WALLABY — the disagreeing voxels share a mask boundary, so the signed sum cancels much of their noise — and cannot be checked at all on the one stencil that spans the field, which is the regime where it would matter most. Both are reported side by side. Differences are 1.9–15.4 noise-equivalent σ (HGC 44) and 4.8–6.3 (the four measurable WALLABY sources), so "flux within 8%" is a statement about segmentation. **WALLABY #1 has NO measurable σ** — it spans the field, so its stencil cannot be placed fully on empty sky (0 clean placements in 1600 draws); reported as a dash rather than backed by a model shown to be wrong by up to 2×. Placements must be FULLY clean: an earlier ≥98% gate summed every finite voxel and let real emission into the null, inflating that stencil's scatter ~8%, above the ~5% Monte-Carlo error. **n_σ is explicitly NOT a p-value** — both masks are selected on the data they are integrated over, so boundary voxels carry an Eddington-type selection bias the fixed-mask model does not capture; a formal test would need injected sources or masks from independent data. rms is from emission-free voxels; local rms agrees with global within 2%. **R-18 is now fully closed.**

### R-23 — Gaussian line-fit errors ignore spectral correlation *(DONE)*
- **Severity:** LOW-MED · **Effort:** S-M · **Domains:** Radio, Astronomer
- **Evidence:** `compute/spectral.py` derived every `*_err` from a `curve_fit` covariance that assumes independent channels. R-18(d) measured ρ₁ ≈ 0.74 between adjacent channels on HGC 44, so on a spectrally smoothed cube those errors were optimistic.
- **DONE:** `effective_sample_factor()` measures F = 1 + 2Σρ_k from the fit RESIDUALS (no extra input, and F = 1 leaves an uncorrelated spectrum untouched) and the covariance is inflated by F before any error is derived — so the diagonal errors and the propagated integrated flux all carry it consistently. F is returned as `spectral_correlation_factor` so a user can audit it.
- **Validated by Monte Carlo, not by assertion** (`tests/test_line_fit_correlated_noise.py`): over kernels giving ρ₁ = 0 … 0.96 the uncorrected errors are optimistic by 1.0×, 1.7×, 2.3×, 3.0× while √F predicts 1.00, 1.64, 2.25, 2.72; F recovered from residuals lands within ~7 % in √ terms. The coverage test compares reported 1σ against the true scatter of 120 refits and fails at 2.34× with the correction disabled.

### R-24 — 2-D Gaussian source-fit errors ignore beam correlation *(DONE)*
- **Severity:** MED · **Effort:** M · **Domains:** Radio, Astronomer
- **Evidence:** `worker_gauss_fit_2d` derived its errors from a `curve_fit` covariance that assumes independent pixels, on data whose noise is correlated across the beam. Monte Carlo (source σ = 4 px, 41×41 window, 300 realisations): reported-vs-actual ratios **3.3×/3.7× at 2.5 px beam FWHM, 4.8×/5.9× at 4 px, 5.8×/8.1× at 6 px** (amplitude / integrated flux); white noise 1.05/0.94.
- **Two analytic corrections measured and rejected** before the one that shipped. Beam-area scaling predicts 2.66/4.26/6.39 against those measurements. Summing the residual autocorrelation over the 2-D lag plane over-corrects *and* inflates WHITE noise by 1.4×, because thresholded lags accumulate estimator scatter rather than signal.
- **DONE:** `empirical_fit_errors()` injects the FITTED model at source-free positions in the same image, refits, and returns the full empirical parameter COVARIANCE — full, not just the diagonal, because the integrated flux is propagated through the amp–σx–σy off-diagonals, which is exactly where correlated noise does its worst. Validated against the true scatter at 0.88–1.06 on correlated noise.
- **Source-free is decided by the FRACTION of bright pixels, not by any single one.** Demanding no pixel above 3σ rejects nearly every window (a 41×41 box of pure noise holds ~4 by construction) and biases the null low — this null is meant to BE noise, tails included. Measured: pure-noise windows put ≤1.8 % of pixels beyond 3σ, a window containing a source puts 11–13 %, so the 5 % gate separates them with room to spare.
- **It refuses rather than guesses:** a field too crowded to find clean positions returns nothing, and the result carries `error_method` ("empirical" / "covariance") and `error_trials` so a caller knows which it got. Cost is ~0.07 s on top of the fit, and the estimator's memory is bounded by the fitting window (strided sample for the background, per-window brightness test) rather than by the plane.

### R-25 — Power-spectrum error bars assumed independent Fourier modes *(DONE)*
- **Severity:** LOW-MED · **Effort:** S · **Domains:** Cosmo
- **Found by audit, not by report.** After the same class of defect was fixed twice (R-23 line fit, R-24 source fit) I swept every `*_err` the compute layer returns and checked each one's provenance. All were sound except `cl_err` in `compute/cosmology.py`, which was `std(bin)/√N` over the Fourier modes in a |k| bin — modes the Hann window couples, so it counted more independent samples than exist.
- **Measured** over 40 Gaussian random fields of white and red spectra: reported was **1.7–2.5× too small**, uniformly. That matches √(⟨w⁴⟩/⟨w²⟩²) = 1.94 for a separable 2-D Hann window, now applied and derived from the window array rather than hard-coded.
- **Two further defects surfaced when the fix was checked at the SHIPPED default binning** (n_bins=64, not the coarse n_bins=10 the first measurement used):
  - `rfft2` returns half the plane but the kx=0 column — and the Nyquist column for even nx — still hold BOTH members of each Hermitian pair, so those modes were counted twice. Deduplicating them cut the worst sparse-bin error from 2.6× to ~1.8×.
  - A bin holding fewer than 3 modes has no estimable standard error at all (a 2-point std carries ~50 % uncertainty itself). Those now report **null** instead of a number no correction can rescue, and every bin returns `n_modes` so a caller can see how well determined its error is.
- **Honest range, measured at the default binning:** 0.5–1.6× on bins with more than 8 modes, 0.3–2.0× on sparser ones. The scalar factor is an asymptotic populated-bin result and the docstring now says so rather than claiming it holds for any window or binning.
- **Two fixture traps hit while measuring, named in the test file** because each gave a convincing wrong answer: setting `k[0,0]` to a tiny value instead of zeroing the amplitude injects a DC mode that swamps every bin (spurious 8–27×), and normalising each realisation by its own standard deviation makes low-k modes modulate all bins coherently, reading as an error ratio that grows with ℓ (spurious 12× at high ℓ).
- **The Cℓ values DO change slightly**, and the earlier draft of this entry was wrong to say otherwise: deduplication runs before the bin mean, so a duplicated mode no longer counts twice. The effect is small and in the right direction, but it is a change to the values, not only to the error bar. They remain non-science-grade regardless — no mask deconvolution, no beam/pixel window.

### E-08 route coverage — measured and halved *(partial)*
- **Measured, not estimated:** extracting every `@router` path and grepping the suite (excluding the contract snapshot, which lists them all by construction) showed **37 of 129 routes — 29 % — with no test at all**. Now **10 (8 %)**, in `tests/test_uncovered_routes.py`.
- **What it found.** `/v1/cosmology/distance/batch` returned **0.0 Mpc** for a negative or non-finite redshift while the single-redshift endpoint answers "Redshift z must be >= 0" — and 0.0 Mpc is indistinguishable from a genuine z = 0 at the observer. The response model had documented `list[float | None]` with "None entries mark invalid redshifts" all along; the implementation filled zeros. Now null, so bulk input stays tolerated (one bad row does not fail the request) without inventing a distance.
- **The SAMP operator gate is now exercised on all 12 reachable endpoints** — auth required, non-operator refused in multi-user mode, operator let through. A gate that is never tested is a gate nobody knows works; mutation-verified (replacing `require_operator` with `verify_token` fails them all).
- `kinematic_levels_bin` had a thoroughly tested compute layer and an untested HTTP contract; the binary layout, the header set and the two rejection paths are now pinned.
- **The suite was writing into the user's home.** `_EXPORTS_DIR` defaults to `~/.visivo/exports`, and only one test module opted into an isolating fixture — the directory had accumulated **994 files** from past runs. An autouse fixture in `conftest.py` now redirects it to a tmpdir for every test; a full run leaves the real workspace unchanged (verified by counting before and after).
- **Still untested (10), all needing something a unit test cannot supply:** hips (5 — a live survey), dynspec (3 — a pulsar observation), `/v1/datasets/open_skava` (a .skava file), `POST /create_session` (a GPU).
- **A second defect the coverage found:** `/v1/cube/subvolume_bin` answered **HTTP 500** for a ROI wholly outside the cube — the worker raises a clean `ValueError("Invalid subvolume ROI.")` and the route's only handler was a catch-all 500, while every other binary cube route already maps `ValueError` to 422. Bad user input was surfacing as a server error. Fixed; partial overreach still clamps, and the two contracts now have separate tests.
- **A second sweep** added the binary subvolume path the desktop's parallel slab loader uses (geometry headers vs body length, and the declared range actually bracketing the data), HI products with and without a beam, the spectral baseline, the workspace copy, SAMP upload round-trip, and VBT query. Mutation-verified: inflating the declared width by one fails the subvolume test.
- **An in-band-sentinel sweep** was run at the same time, on the theory that the three found this session (`error_method`, `cl_err` nulls, `distanceMpc`) might be endemic. They are not: every other match is an out-of-domain sentinel (−1 for an index, NaN for blank voxels, a below-minimum value for masked voxels in the volume renderer) that cannot collide with real data. Recorded as a negative result so the sweep is not repeated.
