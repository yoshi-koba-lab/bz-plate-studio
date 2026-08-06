# Claude-Panel — independent methodological opinion on the fixed stitcher

**Files judged:** `stitcher.py`, `main_stitch_v2.py`
**Date:** 2026-08-06
**Scope:** a judgement on the four implemented fixes, not a re-review of the whole tool. No code was written or run.

## Bottom line

Three of the four fixes (geometry inheritance, the overlap/ambiguity gate, the QC residual report) are sound engineering that make the tool **more honest**, not just prettier — each one now *reports* its own uncertainty. The flat-field fix is the one that still worries me, and for a specific reason: **it was validated on exactly the specimen class that cannot exhibit its failure mode.** The measured numbers are real and good, but they answer the easy question.

Ranked by severity below. The single most important remaining defect is in §5.

---

## (1) Low-percentile-across-tiles shading estimate — acceptable for publication?

**Verdict: acceptable as a disclosed convenience default for *sparse* specimens; not acceptable as an unqualified "needed for quantification" default, and it can bias real biology for confluent/dense specimens.**

What the estimator actually computes (`estimate_flatfield`, lines 511–534): at each pixel position it takes the 25th percentile *across the well's 21–60 tiles*, box-smooths at 12 % of the tile, normalises to mean 1, clips to [0.2, 5.0], and returns a purely **multiplicative** field to divide by. The implied model is: at a fixed field position, the low percentile across tiles is dominated by the tiles where that position happened to land on background, so the percentile tracks `shading × background_floor` and, after smoothing/normalising, recovers relative shading.

That model holds only when its two assumptions hold, and neither is checked:

- **Assumption A — enough tiles expose true background at each field position.** For a **confluent monolayer or dense tissue**, most field positions are covered by specimen in *most* tiles. The 25th percentile is then not the illumination floor but the 25th percentile of `shading × specimen` — it contains real biological structure. Dividing by it **flattens genuine intensity gradients**: any region that is consistently bright across tiles gets divided down toward the mean. That is the opposite of what quantification wants, and it is invisible in the output. There is no guard (e.g. a test that the low percentile is actually flat, or a coefficient-of-variation-across-tiles gate) to detect that background was never observed.

- **Assumption B — shading is multiplicative with negligible additive/offset structure.** For a **fluorescence channel where most of the field is genuinely dark**, the low percentile is dominated by camera offset + dark current + diffuse autofluorescence — which are largely *additive*, not a multiplicative gain on signal. A multiplicative field estimated from the dark floor either (a) is ≈flat after mean-normalisation and does nothing (best case, harmless), or (b) folds an additive stray-light/offset gradient into a multiplicative correction and mis-scales the sparse real signal. It cannot see a true multiplicative vignette that only manifests on the rare bright pixels. So for a mostly-dark channel the correction is at best a no-op and at worst mistreats additive background as gain — bounded by the clip, but wrong in kind.

- **A structural issue that is independent of specimen density: it is re-estimated *per well* (called inside `stitch_well` on that well's tiles, lines 664 & 687).** A sparse well and a confluent well therefore get *different* fields and are no longer on a common intensity scale. That directly undermines the **cross-well** quantitative comparison the checkbox advertises. A true reference is one field per objective/channel, not one per well.

- **Minor:** the 25th percentile is a biased order statistic for the floor wherever signal is common, introducing a low-spatial-frequency bias correlated with where the specimen tends to sit (e.g. denser toward the well centre); 12 % smoothing reduces but cannot remove a systematic radial correlation. The [0.2, 5.0] clip permits up to a 5× multiplicative distortion — bounded, but large.

**Honest limitation to document:** *"Flat-fielding is image-based: a purely multiplicative field estimated per channel and per well as a low percentile ('darkest common background') across that well's fields. It assumes enough tiles expose true background at each field position and that shading is multiplicative with negligible additive offset. For confluent or spatially structured specimens it can absorb and divide out real intensity gradients; for channels dominated by dark background it mainly tracks the camera offset. Because it is re-estimated per well it does not place different wells on a common intensity scale. It is not a substitute for a measured flat-field reference (dye slide / illumination reference, or BaSiC over a large diverse stack)."*

Also: the GUI label **"Correct illumination (flat-field) — needed for quantification"** (`main_stitch_v2.py:46`) **overclaims**. For a dense specimen, image-based flat-fielding can *harm* quantification. Re-word to "*image-based illumination correction (validate against a reference; may flatten real gradients in confluent specimens)*."

---

## (2) Sub-pixel refinement computed but tiles placed at integer origins — defensible or half-fix?

**Verdict: defensible for cellular morphology/intensity *because the residual is honestly reported*, but it is a genuine half-fix — the code computes the better answer and discards it, and the placement model is a single rigid lattice, not just integer rounding.**

Two facts I want to separate, because they are often conflated:

1. **It is not over-claiming precision.** `_subpixel` (247–287) feeds `_edge_report` (711–747), which writes `residual_median / p95 / max` to `stitch_qc.csv`. Those numbers are the *deviation of the per-edge (sub-pixel) fit from the placed lattice* — i.e. they **disclose the placement error rather than hide it**. A reader who sees "residual_median 1 px, max 3 px" is being told the truth about accuracy. That is the opposite of reporting undelivered precision. Good.

2. **But it is a half-fix in two senses.** (a) The tool computes sub-pixel shifts and throws them away for placement. (b) More importantly, placement is not merely integer — it is a **single global step** (`_consensus` → `.astype(int)`, lines 400–401; `tile_origins`, 478–489). Every tile sits at `i·step_x + j·step_y`. So the reported residual is not just the lost sub-pixel fraction; it is the full **local-vs-global lattice deviation**. A median of 1 px with a max of 3 px means real per-tile stage jitter is being *baked in* at up to 3 px at the worst seams. A classic global least-squares translation optimisation (Preibisch/MIST-style) — or even per-edge integer placement — would reduce this, and the machinery to do it is already present (`_subpixel` gives the measurements). The good news the code earns: **no growth with distance** confirms the rigid lattice does not accumulate error, so the worst case is bounded per tile, not per mosaic.

**So:** integer *single-lattice* placement is adequate for morphology and intensity at cellular scale (objects are tens of px; 1 px median is small; the error is disclosed and non-accumulating). It is **not** adequate for sub-pixel/registration-critical work across seams (colocalisation, PSF-level, tracking), and there the tool computed but did not apply the fix. Recommendation: either promote `_subpixel` into a global optimisation to actually place at the precision it already measures, or keep integer placement but ensure the QC residual is surfaced to the user (it is only in the CSV) and gate seam-sensitive analyses on it. Do not let "sub-pixel seam **check**" be read as sub-pixel **placement**.

---

## (3) Inheriting the plate-median geometry — sound, or masking a mis-acquired well?

**Verdict: sound and well-justified as a fallback, and it is labelled — but the residual safety net is weakest exactly where inheritance is used, and the QC does not record what the inheritance rests on.**

The justification is real and measured: `plate_geometry` (459–475) reports `spread_x/spread_y` (sd 0.5–0.8 px across 8 wells), consistent with one stage program. Inheriting that is far better than abutting at zero overlap (which duplicates a strip of specimen at every seam). Disclosure exists: `step_source="plate"` per well, and `_record_warnings` emits "geometry inherited from the plate." Good.

The narrow-but-real failure mode: a well can fail to self-measure for two different reasons — *low texture / thin overlap / sparse specimen* (common, benign) or *a genuine stage slip* (rare). A true slip usually still produces measurable overlap at a different step, so it is normally *caught* by measurement. The dangerous intersection is a well that **both** slipped **and** is too sparse to measure — it inherits plate geometry and the slip is silently masked.

The honest hole is subtler than "it might mask a slip": **the residual check that would catch a bad inheritance is degraded precisely for inherited wells.** `_edge_report` measures residuals from the well's *own* tiles against the (inherited) step; if the well was too sparse/low-texture to self-measure, `_pair_candidates` will also mostly fail there → few or no edges → `residual_*` come back `None` → the `r95 > 2 px` warning never fires. So the safety net is thinnest exactly when inheritance is invoked.

**What must be reported for it to be honest (partly missing today):**
- `step_source` per well — **present**, good.
- The **number of wells** and **spread** the inherited geometry rests on (`n_x/n_y`, `spread_x/spread_y` exist in the prior dict but are **not** written to `stitch_qc.csv`) — **add these** so a reader can judge the inheritance.
- An explicit flag that inherited-geometry wells have **not** been verified against their own data (residuals may be empty). Today the "plate" warning lacks the "check this well" that "abut" carries (`main_stitch_v2.py:186–188`) — give "plate" the same "check this well" language.

With those additions it is a defensible, disclosed fallback rather than a silent one.

---

## (4) Feather blending unchanged — does flat-fielding change my nearest-tile-centre recommendation?

**Verdict: softened, not reversed. I would still ship nearest-tile-centre for the quantification image and reserve feathering for the cosmetic preview.**

Flat-fielding removes the **strongest** original objection: without it, the two tiles meeting in an overlap sampled the same specimen through *different* parts of the vignette, so averaging blended two mismatched intensity scalings and produced a seam-band DC bias. The measured **overlap-vs-centre bias 0.00 DN** confirms flat-fielding kills that term. That was the DC argument for nearest-centre, and it is now largely answered.

But two of the original reasons are **untouched** by flat-fielding:

- **Noise statistics.** Averaging two tiles in the overlap reduces variance (~√2) within the seam band relative to tile interiors, and feather weights make that a *continuously varying* statistic across the band. Any measurement that assumes uniform noise (thresholding, spot/molecule counting, texture features) sees a spatial artifact along every seam. Flat-fielding does not change this. A hard nearest-centre boundary keeps native single-tile statistics everywhere, with a discontinuity only at the cut.

- **Residual mis-registration.** With integer single-lattice placement and up to 3 px residual (§2), the two tiles in an overlap are **not** perfectly registered. Feathering two mis-registered tiles **ghosts** real structure across the seam (double edges, smeared nuclei); nearest-centre gives a sharp cut instead. For segmentation/morphology a sharp cut is usually less damaging than a ghosted double object. Flat-fielding does not fix registration, so this argument is fully live and is *coupled* to the half-fix in §2.

So my recommendation now: **feathering is fine — even preferable — for the PNG/display preview; nearest-tile-centre (or an explicit "which tile owns each pixel" map) should be the default for the quantification OME-TIFF.** Offer it as an option and state in the docs that the shipped OME-TIFF is feather-blended and what that implies for seam-band statistics. The urgency dropped; the recommendation stands.

---

## (5) Single most important remaining defect + the limitations statement to ship

**Most important remaining defect: the on-by-default, per-well, image-based flat-field is validated *only* on a uniform synthetic specimen — the one case that cannot exhibit its failure mode — while it silently alters the very quantity being measured (intensity) for the common non-uniform case, and re-estimation per well breaks cross-well comparability.**

Why this outranks integer placement (§2) and geometry inheritance (§3): those two are *disclosed* — the QC residual quantifies the placement error, and `step_source` labels inherited wells. The flat-field is different on every axis that matters: it is **on by default**, **labelled "needed for quantification"**, it **modifies intensity** (not geometry), its failure is **specimen-dependent and invisible**, and the headline evidence — **uniform-specimen RMSE 0.71 DN** — is precisely the *best* case for a low-percentile estimator (a common background exists at every field position). The validation demonstrates the estimator recovers a *known flat field on a flat specimen*; it says nothing about a confluent or structured specimen, which is exactly where it can divide out real biology. The proof covers everything except the risk.

Concrete asks, in priority order: (a) validate the flat-field on a **structured/confluent** synthetic phantom and report the intensity distortion, not just uniform RMSE; (b) add a runtime guard (e.g. across-tile CV at each position, or flatness of the low percentile) that warns when background is never exposed and the field may contain specimen; (c) surface that flat-fielding is per-well and re-word the GUI label; (d) offer a shared/measured reference option for cross-well work.

### Limitations statement this tool should ship with

> This tool reassembles BZ-X tiles without proprietary metadata by measuring one rigid tile lattice per well directly from the images. **Placement** is at integer-pixel resolution on a single per-well lattice; the residual between measured seams and that lattice is reported per well in `stitch_qc.csv` (typically ~1 px median, up to ~3 px at the worst seams, non-accumulating with distance). This is adequate for cellular morphology and intensity but **not** for sub-pixel or registration-critical measurements across seams. **Geometry** for a well that cannot measure its own is inherited from the plate-median geometry (labelled `step_source = "plate"`); such wells are not independently verified — inspect them. **Illumination correction** is image-based: a purely multiplicative field estimated per channel and per well as a low percentile across that well's fields, assuming enough tiles expose true background at each position and that shading is multiplicative with negligible additive offset. It was validated on a uniform synthetic specimen (RMSE 0.71 DN); for confluent or spatially structured specimens, where background is rarely exposed, it can absorb and divide out real intensity gradients, and because it is re-estimated per well it does not place different wells on a common intensity scale. It is **not** a substitute for a measured flat-field reference. **Overlaps** are feather-blended, which averages tiles in seam bands — altering local noise statistics and, under the residual mis-registration above, ghosting structure; for rigorous quantification prefer nearest-tile-centre or validate seam regions. Treat all intensities as relative, disclose that flat-fielding was image-based and per-well, and validate against an independent reference before drawing quantitative biological conclusions.

---

## Summary table

| # | Fix | Verdict | Must-do to be publishable |
|---|-----|---------|---------------------------|
| 1 | Low-percentile flat-field | Conditional — biases dense/confluent biology; overclaimed in GUI | Validate on structured phantom; guard for "no background exposed"; re-word label; note per-well scope |
| 2 | Integer placement + QC residual | Defensible (residual disclosed) but a half-fix | Surface QC residual to user; consider global optimisation to *apply* the sub-pixel it measures |
| 3 | Plate-median geometry inheritance | Sound & labelled; safety net weakest where used | Write n-wells/spread to QC; add "check this well" to the "plate" warning |
| 4 | Feather blending (unchanged) | Recommendation softened, not reversed | Nearest-centre for the quantification OME-TIFF; feather for preview |
| 5 | **Most important defect** | Flat-field validated only on the case that can't fail | Ship the limitations paragraph above; validate on non-uniform data |

*Opinion rendered independently on the code as written; measured numbers taken as reported. Additive write only — no existing file modified.*
