# Claude-Panel — Independent Methodological Opinion on the Keyence BZ-X Stitcher

**Reviewer:** Claude-Panel (independent methods/critique seat)
**Date:** 2026-08-05
**Scope reviewed:** `stitcher.py`, `main_stitch_excerpt.py`
**Mode:** Opinion + critique only. No code written or executed.

---

## TL;DR verdict

The engineering is careful and honest about *some* of its own assumptions (vignetting-aware scoring, native-depth reads, atomic writes). But as a tool for **quantitative** microscopy it has one dominant, silent defect and several serious secondary ones. In order of scientific severity:

1. **No flat-field / shading correction anywhere in the output path.** Vignetting is corrected *only for alignment scoring* (`_prep_score`) and is left fully intact in the pixels that get saved. Feather blending then *smooths* the seams, which visually hides the fact that intensities are spatially biased. This is the classic "correct code, wrong science" trap — the mosaic looks clean and is quantitatively wrong everywhere. **CRITICAL.**
2. **Rigid, global, *integer* lattice with no per-tile refinement.** `_consensus` collapses all pairs to one `(dy,dx)` per axis and `astype(int)` throws away the sub-pixel part. This *guarantees* coherent drift and seam misregistration that grows with mosaic size. **HIGH.**
3. **Feather blending is the worst blend choice given (2).** It averages misregistered, non-flat-fielded, differentially-bleached pixels, attenuating and ghosting every feature that spans a seam. **HIGH.**
4. **Max-projection default over 50+ *widefield* Z-slices.** MIP is an extreme-value statistic: background inflation scales with slice count, so it is neither noise-safe nor comparable across datasets. **HIGH (Z users).**
5. **Low-confidence fallback abuts tiles at zero overlap.** With ~29% real overlap, this silently produces a ~29%-too-large mosaic that double-counts a strip at every seam, flagged only as "uncertain." **MEDIUM-HIGH.**

Answers to the six posed questions follow, then the single highest-value change.

---

## (1) Is the global single-step lattice scientifically defensible? Where does it fail?

**Opinion:** Defensible as an *initialisation* and for qualitative whole-well overviews. **Not** defensible as the *final* geometry for quantitative, single-cell-resolution work. A published quantitative stitcher should do per-tile refinement + a global least-squares position solve (the ASHLAR / NIST-MIST / BigStitcher pattern), not a single consensus step.

The code builds a perfectly rigid affine lattice in `tile_origins`:

```
raw[(x,y)] = (i*step_x[0] + j*step_y[0], i*step_x[1] + j*step_y[1])
```

Two design decisions compound here and both hurt quantification:

- **It is a single step.** `_consensus` (lines 315-342) is explicitly built to *discard* per-pair disagreement in favour of one step that "several pairs agree on." That is great for robustness on thin/featureless overlaps, but it means the model structurally *cannot represent* real stage behaviour.
- **It is integer.** `_phase_corr` returns `int` shifts; `_consensus` does `np.median(...).astype(int)`. So the placement of tile `(i,j)` is `i·step + j·step` with an integer step. If the true step is e.g. 1301.4 px, the code uses 1301 and accumulates **0.4 px per tile → several px by the far corner**, in one coherent direction (errors do not cancel).

**Named failure conditions:**

- **Serpentine / boustrophedon scan backlash.** If the BZ-X scans rows in alternating directions, odd and even rows carry opposite mechanical backlash offsets. A single global step averages them and is wrong for *every* row by roughly half the backlash. Raster ("comb") patterns are safer but still accumulate Y backlash.
- **Stage non-repeatability.** Real motorised stages have ±sub-µm to several-µm random positioning error per move. The rigid lattice assumes zero, so each seam carries a random residual → feature doubling at seams.
- **Cumulative sub-pixel drift** (the integer-step problem above), worst at the corner farthest from the origin tile — so large mosaics tear where a reviewer of a small test grid will never look.
- **Optical field distortion / field curvature.** Per-tile magnification varies across the FOV; it is worst at tile edges, exactly where overlaps live. A translation-only lattice cannot correct this and leaves residual, position-dependent misregistration at every seam.
- **Thermal / temporal drift** across a long multi-channel Z acquisition: the effective step at the end of the run differs from the start; the global step is an average that fits neither end.

**Bottom line:** keep the global step as the *seed*, then refine each tile against its measured neighbours and solve globally with **sub-pixel** shifts. At minimum, stop rounding the step to integer.

---

## (2) Is measuring geometry on ONE reference plane and reusing it for all channels correct?

**Opinion:** For the *stage* geometry this is **correct and is actually a strength** — do not "fix" it by aligning channels independently. But the docstring claim of channels being **"perfectly registered"** (lines 9-10, 450) is **overstated** and will mislead colocalisation users.

Why reuse is right:
- On a BZ-X, at each field the instrument captures all channels/Z *before* moving the stage. So the stage XY is genuinely identical across channels at a given tile. The only inter-channel shift is optical, not mechanical. Measuring once and reusing therefore **guarantees mutual channel registration** and avoids channel-dependent seams / spurious colocalisation that per-channel alignment would inject. This is the correct call.

Why the "perfect" claim is wrong:
- **Lateral chromatic aberration** (lateral color) shifts and slightly re-magnifies each wavelength within every tile — commonly 0.5-2 px, worse toward field edges, plus an axial focus offset between channels. Reusing one lattice does *not* introduce this, but it also does nothing about it, and "perfectly registered" tells the user there is nothing to worry about. For anyone doing per-pixel colocalisation this is a real, uncorrected error and must be documented (ideally corrected with a per-objective calibration).
- **Reference-plane choice is hardcoded and non-adaptive.** `reference_plane` (436-441) always prefers `CH4` (brightfield) and otherwise the first plane. Brightfield "has most texture" is often true, but for transparent/unstained live specimens brightfield can be low-contrast and a bright DAPI/nuclear channel would register far better. There is no retry-on-another-plane when confidence is low — it only sets `low_confidence` and proceeds. A published tool should pick the reference by measured information content, or measure on 2-3 planes and cross-check for agreement.

**Bottom line:** reuse of one *stage* geometry = correct, keep it. Drop the word "perfectly," add a chromatic-aberration caveat/correction, and make the reference plane adaptive with a cross-check.

---

## (3) Is feather blending appropriate for quantitative fluorescence? What would you do instead?

**Opinion:** Feather blending is fine for a **visualisation** preview and is the *wrong* default for a **quantitative** output. The problem is not "averaging" in the abstract — averaging two unbiased measurements of the same intensity is a legitimate, variance-reducing estimator. The problem is *what* is being averaged here:

1. **Un-flat-fielded tiles (the dominant issue).** Overlaps are where two tile *edges* meet, and edges are the dark, vignetted part of each tile. Averaging two dark edges makes every seam systematically dimmer than tile centres → a periodic "waffle" intensity field across the mosaic. Worse, the feather weight (distance-to-nearest-edge, `_feather` 391-396) is *correlated with the vignetting brightness*, so blending upweights the brighter, more central pixel — an asymmetric, position-dependent bias. Net effect: the mean intensity you measure for a cell depends on **where in the mosaic it fell**. That is a direct corruption of the quantity being measured, and the feather smoothing *camouflages* it.
2. **Averaging across the known misregistration from (1)/§1.** Because the lattice is rigid + integer, seams *are* misaligned. Feather blending then averages misaligned features → attenuated peaks and ghost/doubled structures along every seam. Any object spanning a seam has both its **morphology and its intensity** corrupted. Given known misregistration, feather is the *worst* choice — a hard seam-cut or max would at least not blur.
3. **Photobleaching asymmetry (fluorescence-specific).** The overlap region is exposed twice (once per neighbouring tile) at different times; the second copy is bleached relative to the first. Averaging yields a value matching neither, plus a bleaching gradient across the overlap band.
4. **SNR discontinuity.** Single-sampled vs double-sampled regions have different noise statistics; downstream thresholding/segmentation then behaves differently across the mosaic.

**What I would do instead (in priority order):**

- **Flat-field first** (see §6) — this alone converts feather blending from "biased" to "legitimate."
- **For quantification, prefer no averaging: nearest-tile-center (Voronoi/seam) assignment** so every output pixel comes from exactly one tile — the one whose centre is nearest and most in-focus. This preserves native single-tile intensity statistics and avoids seam blur entirely. Keep feather as a *display-only* option.
- If blending is retained for quantitative output, do it **only after sub-pixel registration**, and **emit a per-pixel sample-count / provenance map** so downstream tools know which pixels are single vs double sampled.

**Bottom line:** feather = good for the PNG preview, wrong default for the OME-TIFF that people will measure on. Flat-field + single-source (nearest-center) assignment is the quantitatively honest default.

---

## (4) Max-projection across 50+ slices — statistically sound or noise-amplifying?

**Opinion:** **Noise-amplifying and biased**, and especially inappropriate at 50+ slices on a *widefield* instrument. Sound for qualitative display of sparse bright objects; not sound for quantification.

- **MIP is an extreme-value statistic.** The expected maximum of N noisy samples grows with N (≈ σ·√(2 ln N) for Gaussian noise). Over 50 slices the background floor is pulled up substantially and non-uniformly, compressing contrast. Critically, that inflation **depends on the number of Z slices**, so MIP intensities are **not comparable** between wells/experiments acquired with different Z counts.
- **Nonlinear and axially destructive.** MIP discards axial extent: a tall dim structure and a short bright one can project identically. "Total fluorescence" measured on a MIP is not physically meaningful.
- **Widefield makes it worse.** The BZ-X is widefield, not confocal; every in-focus feature dumps out-of-focus haze into neighbouring slices. MIP of 50 hazy slices maximises both the haze contribution and the extreme-value bias.
- The offered alternatives are limited: `mean` (unbiased background, √N noise reduction, but blurs in haze) and `middle` (naïvely assumes the feature sits mid-stack). There is **no focus-based projection**.

**What I would do instead:** default to **Extended Depth of Field (EDF/EDOF)** — pick, per pixel, the value from the *sharpest* slice by a local focus measure (this is what Keyence's own "full focus" does). EDF selects a *real acquired intensity* rather than a noise-inflated maximum, so it avoids background bias and is far more defensible quantitatively, while giving the all-in-focus image users expect from a widefield Z-stack. Keep `mean`/`sum` for users who need linear axial integration, keep MIP as an explicit qualitative option, and **document that MIP intensities are not comparable across differing slice counts**. (Also: all projections should be computed on flat-fielded tiles — see §6.)

---

## (5) The most important thing this gets WRONG that the Codex reviewers might miss

**The single most important defect: the pipeline produces a mosaic that *looks* seamless but is never flat-fielded, so every quantitative intensity is spatially biased — and the feather blend actively hides it.**

Why code-focused reviewers will likely miss it:
- `_prep_score` high-passes the images and the comment (215-224) explains vignetting is "the same in every tile." A reviewer sees high-pass handling and vignetting-awareness and mentally checks the box. But `_prep_score` feeds **only the alignment scorer** — it never touches the pixels written by `blend`/`save_ome_tiff`. The output retains full per-tile shading.
- Feather blending removes the *visible* seam discontinuity, so the rendered mosaic and PNG preview look correct. A reviewer inspecting output sees a smooth image and concludes "stitching works." The defect is invisible without an intensity-vs-position analysis of a flat specimen.
- It is "correct code, wrong science": no line is buggy in isolation; the *scientific pipeline* is missing a stage.

**A very close second, same category:** the `astype(int)` step (§1). A code reviewer reads "shifts are integers, pixels are integers — fine." The *scientific* consequence — guaranteed coherent sub-pixel drift that tears large mosaics at the far corner while passing on small test grids — is a domain judgment they may not make.

**Honorable mention Codex may rationalise as reasonable:** the low-confidence fallback `step_x=(0, tw)` / `step_y=(th, 0)` in `estimate_steps` (364-366) **abuts tiles at zero overlap** when the true overlap is ~29%. That is not a graceful degradation — it produces a mosaic ~29% too large per axis with a duplicated strip at every seam, i.e. it *double-counts* a large fraction of the specimen, and only raises a soft "alignment uncertain" warning rather than refusing. A reviewer may read it as a sensible default; scientifically it is a silent data-fabrication risk. The honest fallback is the **nominal stage step from the BZ-X/OME metadata** (the file already exposes `PhysicalSizeX`; nominal positions are available too) or an explicit refusal — never an abut.

---

## (6) Single change with the highest scientific value for a published tool

**Add a self-calibrating flat-field (shading) correction, applied per channel before Z-projection and before blending — with a documented option to output non-blended, non-projected data for quantification.**

Rationale for ranking it first:
- It fixes the variable actually being quantified — **intensity** — which is currently biased *everywhere*, not just at seams.
- It benefits **every** dataset: single-Z or Z-stack, two tiles or two hundred, any channel. It is not a niche fix.
- It is the **prerequisite that makes the existing feather blend and the projections legitimate.** Flat-fielded tiles average correctly; flat-fielded slices project without the seam-darkening artifact.
- It can be **estimated from the data the tool already loads** — a BaSiC-style (Peng et al., *Nat. Commun.* 2017) flatfield+darkfield model fit from the population of tiles, no calibration slide or proprietary metadata required, which fits this project's "no Analyzer needed" ethos.

Runner-up (if the seat were "highest value for *geometric* fidelity"): switch the rigid integer lattice to **per-tile pairwise registration with a global sub-pixel least-squares position solve** (ASHLAR/MIST-style). High value, but more work, and for whole-well overviews the rigid lattice is often "good enough" — whereas the intensity bias is wrong for *everyone*.

Cheap, high-honesty companion changes (low effort, real value): stop rounding the step to integer; make the low-confidence fallback use nominal metadata or refuse rather than abut; emit a per-pixel sample-count/provenance map; and correct the docstrings ("perfectly registered", vignetting handled) so users don't over-trust the output.

---

## Severity-ranked summary

| # | Issue | Where | Severity | Fix |
|---|-------|-------|----------|-----|
| 1 | No flat-field correction on output; feather hides it | `_prep_score` used for scoring only; `blend` | **Critical** | Self-calibrating (BaSiC-style) shading correction per channel before projection/blend |
| 2 | Rigid, global, integer lattice; no per-tile refinement → coherent drift/misregistration | `_consensus` `astype(int)`, `tile_origins` | **High** | Sub-pixel per-tile registration + global least-squares solve; at minimum drop integer rounding |
| 3 | Feather averages misregistered/un-flat-fielded/bleached pixels | `_feather`, `blend` | **High** | Flat-field first; prefer nearest-tile-center (no averaging) for quantitative output; feather = preview only |
| 4 | MIP default over 50+ widefield slices (N-dependent background inflation) | `_z_reduce` default `max` | **High (Z)** | Default to EDF/best-focus; keep mean/sum; document MIP non-comparability |
| 5 | Low-confidence fallback abuts at zero overlap → ~29% double-count | `estimate_steps` fallback | **Medium-High** | Use nominal metadata step or refuse; never abut |
| 6 | "Channels perfectly registered" overstated; no chromatic-aberration caveat/correction | docstrings; no lateral-color handling | **Medium** | Correct wording; document/correct lateral chromatic aberration for colocalisation |
| 7 | Reference plane hardcoded (CH4), no adaptive retry on low confidence | `reference_plane` | **Low-Medium** | Choose reference by measured information content; cross-check across planes |

*Prepared independently for the review panel. Additive write only; no existing file modified.*
