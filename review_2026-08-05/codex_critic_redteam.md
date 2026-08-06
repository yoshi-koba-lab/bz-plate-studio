# Codex-Critic red-team review: BZ-X stitching

**Review timestamp:** 2026-08-05 22:49:57 JST  
**Files reviewed:** `stitcher.py`, `main_stitch_excerpt.py`  
**Question under test:** Assume the mosaics are subtly wrong; identify mechanisms that can produce a visually convincing but geometrically or quantitatively invalid result.

## Bottom-line verdict

The implementation is not safe to treat as a quantitative stitcher without additional validation. The most likely geometric failure chain for the stated data is:

1. A correct horizontal neighbor can have a real transverse displacement such as `dy=-3`.
2. Phase correlation can generate that correct full-resolution candidate, but `_pair_candidates` ranks it after independently decimating both tiles by two (`stitcher.py:296-307`). An odd displacement cannot be represented exactly on those two decimated sampling grids. The correct match can therefore lose most of its NCC.
3. `_consensus` can then select a repeated-texture/strip alias because vote count dominates score and there is no 29% overlap prior, uniqueness test, or post-placement residual check (`stitcher.py:315-342`). A wrong repeated-pattern shift can pass with a very high score.
4. If no candidate clears `0.15`, the code knowingly substitutes zero overlap, despite the measured overlap being about 29% (`stitcher.py:361-367`). It still writes the mosaic.
5. Whether the chosen global step is slightly wrong, grossly aliased, or merely unable to represent per-tile stage jitter, feathering turns hard discontinuities into smooth double edges. The normal percentile-stretched PNG preview makes this easier to miss (`main_stitch_excerpt.py:136-145`).

The specific suspicion that an already selected `step_x=(-3, dx)` has its shear dropped or applied twice is **ruled out**. The origin formula applies it exactly once. The danger is that the correct odd shift may be scored incorrectly before that point, and that a single integer affine lattice is insufficient even when the selected step is approximately right.

### Status summary

| Area | Verdict | Principal consequence |
|---|---|---|
| (a) One global lattice versus stage error | **NOT RULED OUT** | Smoothly ghosted overlap bands, row/column drift, distorted object coordinates |
| (b) Consensus and `tol=6` | **NOT RULED OUT; demonstrated** | Wrong alias can win with high confidence and no warning |
| (c) Negative `dy` shear application | **RULED OUT after correct selection** | `-3` is applied once; upstream odd-shift scoring and integer quantization remain unsafe |
| (d) uint8 overflow / four-tile brightening | **RULED OUT** | Accumulation is normalized float32 |
| (d) blend ghosting / seam / integer bias | **NOT RULED OUT; integer bias demonstrated** | Double structures, coverage-dependent noise, deterministic `-1` DN errors |
| (e) abutting fallback | **NOT RULED OUT; explicit behavior** | Duplicate specimen bands and inflated physical extent |
| (f) 38–53-slice maximum projection | **NOT RULED OUT; inherent** | N-dependent background, hot pixels, saturation, false spots, lost Z |
| (g) quantitative validity overall | **NOT RULED OUT** | Uncorrected shading, fragile OME axes/calibration, missing provenance and validity masks |

## (a) The global-lattice assumption versus real stage error

**Mechanism.** `estimate_steps` collapses all horizontal edges to one `step_x` and all vertical edges to one `step_y` (`stitcher.py:345-367`). `tile_origins` then forces every tile onto

\[
O(i,j)=i\,s_x+j\,s_y
\]

(`stitcher.py:370-381`). A real stage is better described as

\[
O_{true}(i,j)=i\,s_x+j\,s_y+\delta_{i,j},
\]

where `delta` contains repeatability error, backlash, slow drift, and possibly field-dependent optical distortion. This implementation has nowhere to put `delta`.

With independent absolute positioning jitter, each tile's copy of an overlapping feature is displaced by its own residual. With systematic pitch error, serpentine backlash, or integer/subpixel rounding, the residual can vary coherently or grow across the grid. Feathering averages the displaced copies rather than correcting them. The expected symptoms are:

- double or broadened edges within overlap bands;
- centroids and boundaries that move gradually as the feather weights transfer ownership between tiles;
- up to four displaced copies or an especially blurred feature at a four-tile junction;
- row-dependent offsets under serpentine backlash;
- increasing far-corner error when a fractional true pitch is represented by one integer step.

**Verdict: NOT RULED OUT.** There is no per-tile refinement, graph optimization, local warp, edge-residual report, 2x2 cycle-closure check, or final overlap-sharpness validation. Only the global steps and mean scores are returned (`stitcher.py:509-515`), and the GUI removes that geometry record before writing (`main_stitch_excerpt.py:107-119`). A few pixels of error will often look like pleasing smoothing in an overview rather than an obvious seam. A high global NCC does not prove subpixel landmark agreement.

**How to test it.**

1. Create a textured ground-truth canvas and crop a regular 29%-overlap grid with known per-tile perturbations of `±1` through `±6` pixels, plus separate slow row drift and odd/even-row backlash cases.
2. Stitch it and compare against the known canvas at native resolution. Stratify landmark split, centroid error, FWHM, edge MTF, object area, and segmentation count by one-, two-, and four-tile coverage.
3. On real data, retain a full-resolution shift for every neighboring pair. Plot residual vectors `measured_edge_shift - global_step`, their 95th percentile, and residuals versus row, column, and acquisition direction.
4. For each 2x2 tile square, calculate loop closure. The four measured edge vectors should sum to approximately zero. A global score cannot substitute for this test.
5. Blink or false-color the two source overlap crops after placement at 1:1 zoom. Inspect four-way junctions and the far corner, not only the PNG overview.

## (b) `_consensus`, its six-pixel window, and wrong-but-self-consistent steps

**Mechanism 1: vote count dominates match quality.** For each anchor candidate, `_consensus` takes the maximum candidate score from each pair within a rectangular `±6` pixels, then compares candidates lexicographically as `(votes, total)` (`stitcher.py:325-338`). Consequently, a weak displacement appearing in more pairs beats much stronger true displacements whose real stage variation spreads them beyond the tolerance.

A direct unit fixture demonstrates it:

```text
pair 1: true (0,100), NCC .95; common alias (0,130), NCC .20
pair 2: true (0,107), NCC .95; common alias (0,130), NCC .20
pair 3: true (0,114), NCC .95; common alias (0,130), NCC .20
```

With `tol=6`, the true candidates do not form a common three-vote cluster, while the alias does. `_consensus` returns `(0,130)` with confidence `.20`, above `MIN_PAIR_SCORE=.15`. It is therefore wrong and not warned. The seed-window clustering is also non-transitive: values `-6, 0, +6` can all agree around the `0` seed even though the extremes differ by 12 pixels.

**Mechanism 2: alias support is manufactured upstream.** `_pair_candidates` explicitly generates phase-correlation aliases separated by each tested strip width (`stitcher.py:271-293`). It accepts any forward main-axis shift up to a whole tile (`stitcher.py:301-306`), and its overlap-area floor is only 1% of the downsampled tile (`stitcher.py:299-309`). There is no prior around the independently measured 29% overlap. A repeated cell lattice, stripes, or other periodic texture can therefore make a physically implausible shift highly self-consistent.

In an audit fixture with tile width 200, true `step_x=(-3,142)`, a period-20 sinusoidal specimen, and small Gaussian noise, the implementation selected `step_x=(-3,22)` with `peak_x≈0.981`. That is a 120-pixel alias with excellent reported confidence; the GUI would not warn.

**Mechanism 3: the confidence omits important uncertainty.** Empty candidate lists are discarded (`stitcher.py:322`), so confidence has no denominator representing all expected edges. There is no minimum number or fraction of supporting pairs, runner-up margin, held-out validation, expected-overlap check, or comparison of the resulting mosaic dimensions with stage metadata. Exact `(votes,total)` ties keep the first encountered candidate because line 337 uses strict `>`. Candidate generation begins with a set and sorting uses score only, so equal-score tie order is not a scientific criterion.

The returned coordinates are the coordinate-wise median followed by `.astype(int)` (`stitcher.py:341-342`). This truncates rather than estimates a subpixel step; for example, `-2.5` becomes `-2`. A half-pixel pitch error repeated over many columns can become a multi-pixel far-edge error.

**Verdict: NOT RULED OUT; directly demonstrable.** The tolerance and tie policy can lock onto a wrong but self-consistent shift, and `MIN_PAIR_SCORE` does not detect periodic aliases.

**How to test it.**

1. Make `_consensus` unit tests with the three-pair fixture above, permute pair and candidate order, and sweep `tol` from 1 to 20. Record mode support, support fraction, score sum, spread, and runner-up margin.
2. Generate stripes, checkerboards, and cell-like lattices across a range of periods and noise levels. Require exact recovery of a known approximately 71%-of-tile pitch.
3. Test sparse and nearly featureless wells, including only one measurable neighbor pair. A high score from one edge must not be represented as well-level confidence.
4. On real data, plot every candidate mode and enforce a plausible overlap interval centered on the independently measured 29%, unless trustworthy stage metadata says otherwise.
5. Validate the selected geometry on held-out neighbors at full resolution and reject it if the residual distribution or ambiguity margin fails a predeclared threshold.

### A separate scorer defect that is especially relevant to `dy=-3`

Candidate ranking uses `sa=a[::2,::2]` and `sb=b[::2,::2]`, then scores `round(dy/2), round(dx/2)` (`stitcher.py:215-224`, `296-307`). These independently decimated arrays have compatible sampling phases only for even displacements. If a component is odd, no integer shift of the decimated arrays can compare the exact same original pixels. The high-pass operation then emphasizes precisely the high-frequency content most damaged by that phase mismatch.

An in-memory audit using exact crops from one random canvas produced the following at the known displacement (`192x256` tiles, `dx=150`):

| True `dy` | Full-resolution NCC | Current DS=2 NCC | Rank of correct candidate |
|---:|---:|---:|---:|
| `-4` | `1.000` | `0.998` | 1 |
| `-3` | `1.000` | `-0.017` | 5 |
| `-2` | `1.000` | `0.999` | 1 |

Natural microscope data are smoother than white-noise texture, so the collapse need not be this large, but the mechanism is exact and the high-pass makes it relevant. Odd `dx` has the same problem. The correct candidate may fall out of the retained top six, trigger the abutting fallback, or lose to a strip alias.

**Targeted test:** take one real horizontal pair for which full-resolution phase correlation reports `dy=-3`. Score the same candidate (i) at full resolution, (ii) with current independent decimation, (iii) after antialiased resizing with a subpixel-capable shift, and (iv) by aligning first and only then subsampling the common overlap. Repeat after synthetically translating one tile by one pixel. A one-pixel parity change should not radically reorder candidates.

## (c) Sign conventions and the negative transverse shift in `step_x`

**Specific mechanism proposed:** a `step_x` such as `(-3,1301)` might lose its negative `dy`, apply it with the wrong sign, or apply it twice when origins are normalized.

**Verdict for that specific mechanism: RULED OUT, provided the step was selected correctly.** The tuple convention is consistently `(dy,dx)` in `_score_shift`, `_phase_corr`, and `_pair_candidates` (`stitcher.py:235-262`, `271-312`). Horizontal filtering requires positive `dx` but does not prohibit negative `dy` (`stitcher.py:301-305`). `tile_origins` computes

```text
y = i*step_x[0] + j*step_y[0]
x = i*step_x[1] + j*step_y[1]
```

exactly once (`stitcher.py:375-378`). Subtracting global minima afterward is only a translation and cannot change neighbor differences (`stitcher.py:379-381`). `blend` consumes the resulting `(oy,ox)` directly (`stitcher.py:407-414`).

For three columns and `step_x=(-3,1301)`, the pre-normalization y coordinates are `0,-3,-6`; normalization makes them `6,3,0`. Neighbor differences remain `-3`. Constant affine shear is represented, not dropped or doubled.

**Important residual risks that are NOT ruled out:**

- The DS=2 parity defect above can mis-rank a correct odd `-3` before origin construction.
- Integer-only phase peaks and median truncation cannot represent a fractional shear or pitch; systematic rounding error can accumulate across a large grid.
- A single cross-axis term cannot represent row/column-dependent shear, backlash, or distortion.
- Main-axis direction is hard constrained to positive `dx` for `+X` and positive `dy` for `+Y`. If the vendor indices run in the opposite physical direction for a dataset, the real candidate is filtered and fallback/alias behavior follows.

**How to test it.**

1. Assert for every grid point that `origin(i+1,j)-origin(i,j)==step_x` and `origin(i,j+1)-origin(i,j)==step_y`, both before and after normalization.
2. Crop a known canvas at `step_x=(-3,1301)` and a cross-coupled `step_y=(976,2)`. Place point landmarks in every overlap and verify exact coincidence at full resolution.
3. Compare `_score_shift(a,b,-3,dx)` with `_score_shift(a,b,+3,dx)` on the same known pair to verify sign independently of candidate ranking.
4. Check directory-index direction against OME/stage positions when available; do not infer it solely from the imposed positive-direction filter.

## (d) Feather blending, uint8 accumulation, and four-tile intersections

### Overflow and simple overlap-count brightening

**Verdict: RULED OUT.** `acc` and `wgt` are float32 (`stitcher.py:403-405`); each tile is converted to float before weighted addition, and `acc/wgt` is computed before conversion back to integer (`stitcher.py:413-420`). Thus four tiles do not overflow an 8-bit accumulator or create a fourfold brightness jump. For perfectly registered, photometrically identical inputs, the intended operation is a normalized convex average.

### Deterministic integer undercount

**Verdict: NOT RULED OUT; reproduced.** The final positive float is cast with bare `.astype(np.uint8)` or `.astype(np.uint16)` (`stitcher.py:416-419`), which truncates rather than rounds. Float32 multiply/divide can also produce `99.99999` when the mathematically exact answer is 100.

Using the production tile shape `(1374,1832)`, constant-valued uint8 tiles of 100, and a 2x2 grid with `step_x=(0,1301)` and `step_y=(976,0)`:

- one isolated tile returned 99 in 146,776 of 2,517,168 pixels (5.83%);
- the 2x2 mosaic returned 99 in 696,865 of 7,362,550 pixels (9.46%);
- in the 398x531 four-way rectangle, 53,601 of 211,338 pixels (25.36%) were 99.

The amplitude is only one DN in this fixture, so it is nearly invisible, but its feather-correlated spatial pattern is real. It can alter threshold decisions and low-count fluorescence statistics. Mean Z projection has a related truncation at `stitcher.py:432`.

### Ghosting, seam bias, and noise statistics

**Verdict: NOT RULED OUT.** `_feather` assigns a pyramidal distance-to-nearest-edge weight (`stitcher.py:391-396`) and `blend` averages all copies (`stitcher.py:407-415`). It has no deghosting, exposure/gain matching, flat-fielding, seam selection, or registration-uncertainty model.

- A geometric residual produces two smoothly mixed structures in an ordinary overlap and as many as four at an X/Y overlap intersection. Peaks are attenuated, edges broaden, and centroids can move with the weights even without an obvious seam.
- Tile-fixed vignetting and acquisition-time gain/bleaching differences make the weighted value depend on which tile contributes. Normalization removes the feather amplitude, not the optical shading field.
- Noise variance changes with effective coverage. Equal independent two-way and four-way averages reduce background standard deviation by about `1/sqrt(2)` and `1/2`, respectively, while singly covered pixels retain one exposure. A uniform global threshold therefore has coverage-dependent false-positive behavior.
- No coverage, effective-weight, or provenance image is saved, so downstream analysis cannot mask or model these regimes.

**How to test it.**

1. Make constant uint8 and uint16 identity tests for every relevant constant, grid size, and overlap. Require bit-exact constancy and map residuals by coverage. Compare the current cast with rounding before conversion.
2. Stitch synthetic beads and one-pixel lines with known `±1` to `±6` pixel residuals. Measure peak, FWHM, centroid, object area, and split/merge rate separately in one-, two-, and four-tile regions.
3. Stitch independent flat-noise tiles and compare variance and threshold exceedances against effective sample number `sum(w)^2/sum(w^2)`.
4. Use a uniform fluorescent slide and measured per-tile flat fields to quantify center, edge, two-way seam, and four-way-junction intensity.

## (e) `MIN_PAIR_SCORE=0.15` and the abutting fallback

**Mechanism.** When either consensus is missing or its mean selected NCC is below `.15`, `estimate_steps` substitutes `(0, tile_width)` or `(tile_height,0)` (`stitcher.py:361-367`). For 1832x1374 tiles with 29% overlap, the approximate true pitch is 1301x976. The fallback therefore discards about 531 horizontal and 398 vertical overlap pixels per neighbor.

It does not crop those duplicate specimen regions away. It places the second tile after the first, so specimen coordinates jump backward by the overlap at each boundary and the common specimen band appears twice. Mosaic extent is inflated by approximately

```text
(number_of_columns - 1) * 531 pixels
(number_of_rows    - 1) * 398 pixels
```

while the original pixel size is retained. Each tile remains internally sharp; repetitive tissue can therefore make the result resemble a larger, plausible well rather than a corrupted one.

**Verdict: NOT RULED OUT; this is the explicit fallback.** A completely silent GUI fallback is ruled out: `low_confidence` becomes true and a post-run warning is shown (`stitcher.py:509-516`; `main_stitch_excerpt.py:110-119`, `201-217`). That safeguard is inadequate because the file is written first and counted as a successful well. The warning says alignment was uncertain; it does not say that zero overlap was substituted and specimen content was duplicated. Geometry, score, fallback state, and expected-versus-actual dimensions are not embedded in the OME-TIFF.

The odd-shift scorer defect makes fallback more likely for the stated `dy=-3`. Conversely, a wrong periodic alias above `.15` avoids both fallback and warning, which is worse.

**How to test it.**

1. Crop a 3x3 grid with exact 29% overlap from a global phantom. Put unique numbered fiducials inside every overlap strip.
2. Make the reference plane flat or featureless to force fallback while keeping another channel informative.
3. Assert output dimensions, count every fiducial, and measure the backward coordinate jump and physical extent. Verify that the saved OME contains no fallback marker.
4. Add threshold-boundary fixtures at NCC `0.149` and `0.151`, plus a high-scoring periodic false match. Neither result should be accepted for quantitative output merely because it lies on one side of a single absolute threshold.

## (f) Maximum Z projection across 38–53 slices

**Mechanism.** `_z_reduce` applies a raw per-pixel maximum (`stitcher.py:425-433`), and maximum projection is both the API default and the first/default GUI choice (`stitcher.py:444`; `main_stitch_excerpt.py:27-31`, `63-67`). It selects an extreme noise sample as readily as a focused biological signal.

For independent Gaussian background, the expected maximum of 38 and 53 samples is approximately the baseline plus 2.1 and 2.3 standard deviations, respectively. Real camera noise is not perfectly independent or Gaussian, but hot-pixel survival, upper-tail background, and saturation probability still rise with the slice count. A maximum is nonlinear and not an estimator of integrated fluorescence. Values are not comparable across stacks with different numbers of slices or different noise, and axially separate objects can become apparently colocalized in 2D.

There are four compounding implementation details:

1. `wt.z_values` is a well-wide union (`stitcher.py:124-131`), while `load` silently skips missing files and unreadable slices (`stitcher.py:456-474`). Adjacent tiles/channels can therefore take maxima over different N, producing tile-dependent background and hot-pixel probability. A missing file is not recorded as unreadable.
2. Projection is performed per tile before blending (`stitcher.py:474`, `500-506`). For positive normalized weights,

   \[
   \sum_i w_i\max_z I_i(z) \geq \max_z\sum_i w_i I_i(z).
   \]

   The inequality is strict when neighboring tiles peak at different Z. For two equal-weight tiles with Z values `[10,0]` and `[0,10]`, the current order returns 10 whereas blend-each-Z-then-project returns 5. This creates a coverage-dependent upward bias.
3. The reference geometry is measured from the same selected projection (`stitcher.py:478-490`). A 53-slice brightfield maximum need not be the best focused or most registerable plane.
4. The saved OME declares only `CYX` and records neither projection mode, source Z indices/counts, nor rejected slices (`stitcher.py:522-533`). The irreversible loss of Z is not machine-readable downstream.

**Verdict: NOT RULED OUT; noise amplification is inherent.** It may make sparse fluorescence look brighter and more complete while increasing false puncta, object width, background, and saturation.

**How to test it.**

1. Project real dark/background stacks at N=`1, 38, 53`; compare background mean/median, 99th and 99.9th percentiles, hot-pixel count, saturation, and false-positive object count.
2. Inject known beads and rare hot pixels. Compare maximum, mean, middle, a robust percentile, and focus-selected projection against the ground truth.
3. Delete selected Z files in a controlled copy and measure tile-to-tile background as effective N changes. Also verify that the “middle” mode selects the intended physical Z, not merely the middle of the remaining list.
4. Compare current project-then-blend with stitch-each-Z-then-project; map the difference by coverage count.
5. Estimate geometry independently from maximum, middle, mean, and a focus-selected reference slice. Inspect step stability and full-resolution residuals.

## (g) Other ways the result can look right but be quantitatively unusable

### 1. Uncorrected tile shading is preserved in the scientific output

**Verdict: NOT RULED OUT; directly implied.** `_prep_score` explicitly removes low-frequency vignetting for registration (`stitcher.py:215-224`), but this correction is never applied to the images passed to `blend`. Thus the code knows a repeatable illumination field exists and only hides it from the scorer. In singly covered regions, feather multiplication and division cancel, leaving raw vignetting unchanged. In overlaps, values from differently shaded tile coordinates are averaged. Feathering can smooth the visual transition while retaining a lattice-periodic intensity field.

This is likely the largest non-geometric threat to quantitative fluorescence: otherwise identical cells can acquire different intensity, background, and segmentation probability depending on their coordinate modulo the tile grid.

**Test:** stitch a uniform fluorescent slide or blank field; fold the mosaic coordinates modulo 1832x1374; plot the median intensity field and its Fourier spectrum at the tile frequencies. Compare object intensity versus distance from tile center/edge and before/after a validated flat-field correction.

### 2. Multi-page OME-TIFF axes are assumed rather than parsed

**Verdict: NOT RULED OUT.** `_ome_planes` treats every PIL frame as a channel-like `Plane` and maps OME `<Channel>` elements to raw page index (`stitcher.py:69-94`). Z is inferred only from `_Z###` in the filename (`stitcher.py:121-124`). `_read_plane` seeks that fixed page while the loader iterates filename-derived Z (`stitcher.py:186-201`, `456-474`). No code interprets `DimensionOrder`, `SizeC`, `SizeZ`, `SizeT`, `TiffData`/IFD mapping, series, samples-per-pixel, or pyramidal structure. Output then declares every selected page to be a C plane in `CYX` (`stitcher.py:522-527`).

The approach is correct only if every top-level page is exactly one channel, page order is identical in every file, and all Z is exclusively encoded in filenames. If pages are internal Z or T, the output can present slices/timepoints as plausible “channels.” If a multi-sample RGB/pseudocolor array is returned, `_read_plane` takes `max(axis=2)` (`stitcher.py:197-199`), which is not generally a quantitative intensity recovery.

There is also a discovery bug: if `_ome_planes` cannot open the first sample, it returns a truthy one-plane fallback (`stitcher.py:69-77`), so the “try up to five samples” loop immediately stops (`stitcher.py:132-140`). One bad first tile can hide valid extra fluorescence pages in later files. A readable but anomalous one-page first tile is likewise trusted.

**Test:** inspect every representative source pattern with `tifffile.TiffFile.series[*].axes/shape` or Bio-Formats and reconcile every IFD with OME `TiffData FirstC/FirstZ/FirstT`. Build sentinel OME fixtures with a unique value for every `(C,Z,T)` and assert plane labels and values. Put an unreadable or anomalous one-page file first, followed by valid three-page files, and verify discovery completeness.

### 3. “One geometry” does not mean channels are perfectly registered

**Verdict: NOT RULED OUT.** Reusing one stage lattice is sensible and prevents channel-specific tile geometry, but it does not correct lateral chromatic aberration, wavelength-dependent magnification, filter/camera offsets, time-dependent sample drift, or z-dependent parallax. The docstring claim that this keeps channels “perfectly registered” (`stitcher.py:9-10`, `448-450`) is stronger than the implementation.

**Test:** image multicolor registration beads, measure per-channel displacement and local affine residual across tile centers and edges, and repeat over time/acquisition order. Report rather than assume the colocalization error.

### 4. Missing data become zeros or unequal projections without a validity mask

**Verdict: NOT RULED OUT.** Missing files are skipped (`stitcher.py:460-466`); ragged Z stacks are silently top-left cropped to a common size (`stitcher.py:469-474`); `blend` emits numeric zero wherever total weight is zero (`stitcher.py:415`). The saved image has no validity/coverage mask. A missing tile, smaller plane, or blank canvas area can be interpreted as true zero fluorescence. Geometry uses the union of positions across channels, so a channel with missing positions can retain a full-sized zero-filled canvas.

**Test:** remove one Z, one whole channel tile, and one reference tile in controlled fixtures. Require a completeness report and compare the saved data/metadata with a coverage mask. Downstream measurement software should be able to distinguish invalid from biological zero.

### 5. Physical scale and processing provenance are fragile

**Verdict: NOT RULED OUT.** `tile_pixel_um` reads only the first indexed file, uses a narrow decimal-only regex for `PhysicalSizeX`, ignores `PhysicalSizeY` and the declared unit, and returns zero on failure (`stitcher.py:155-164`). Output assigns that one number to both X and Y (`stitcher.py:525-534`). Separate TIFFs are written without the OME calibration path (`main_stitch_excerpt.py:127-135`). Geometry steps, scores, fallback use, coverage, unreadable/missing slices, and Z projection mode are not persisted.

**Test:** use anisotropic X/Y sizes, nanometer units, scientific notation, and a bad first file; reopen each format in Fiji/Bio-Formats and assert physical dimensions and units. Require machine-readable processing history including projection, source Z count, origins, edge residuals, confidence, fallback state, and coverage.

### 6. The preview is optimized to look good, not to expose defects

**Verdict: NOT RULED OUT.** PNG generation ignores zero pixels when choosing the 1st and 99.5th percentile display range and stretches every plane independently (`main_stitch_excerpt.py:136-145`). This can conceal zero holes, background shifts, saturation, cross-well intensity differences, and small seam biases. A smooth composite is not a geometry or radiometry test.

**Test:** always generate fixed-window views, raw overlap differences, red/cyan neighbor overlays, checkerboards, coverage maps, and residual-vector maps alongside the pretty preview. Blind-review both sets.

### 7. Cancellation and output naming can yield plausible incomplete data

**Verdict: NOT RULED OUT.** If cancellation occurs between planes, `stitch_well` can return the channels accumulated so far before adding `__geometry__` (`stitcher.py:494-516`). The worker can treat a nonempty partial result as success and write it (`main_stitch_excerpt.py:107-125`). Split files use only `p.label` in the filename, so duplicate labels collide (`main_stitch_excerpt.py:127-135`). These are completeness/provenance failures rather than geometric ones, but they can leave a plausible-looking partial dataset.

**Test:** cancel deterministically after the first plane and require no “successful” output. Use duplicate channel labels and assert unique, non-overwriting destinations and a complete manifest.

## Minimum acceptance tests before downstream quantification

1. **Axis audit:** prove the actual the vendor file layouts map correctly from OME `(C,Z,T)` to exported planes; fail closed on an unrecognized layout.
2. **Known-geometry phantom:** recover approximately 29% overlap from nonperiodic, periodic, sparse, odd-shift, jittered, and serpentine grids. Include the observed `dy=-3` and require parity-invariant scoring.
3. **Residual QC:** retain every neighbor measurement, require adequate support and uniqueness, solve/refine per-tile positions, and publish residual/cycle-closure maps. A fallback mosaic should not be released as quantitative data.
4. **Photometric identity:** constant tiles must remain bit-exact; a uniform slide must remain spatially uniform after stitching; noise and signal statistics must be reported by coverage.
5. **Z audit:** quantify background/false-object behavior at 1, 38, and 53 slices and compare both projection/blend orders. Preserve Z or explicitly record the irreversible projection.
6. **Output audit:** reopen every saved format and verify channel identity, dimensions, physical calibration, complete channel/tile coverage, projection provenance, geometry/fallback state, and a validity map.

## Overall decision

For qualitative whole-well viewing, some outputs may be serviceable after manual native-resolution inspection. For cell counting, morphology, spatial distances, fluorescence intensity, thresholding, or colocalization, the current implementation is **not validated**. Smooth seams and a high single NCC score are specifically insufficient evidence: the code contains mechanisms that can make a wrong lattice look confident, make an invalid zero-overlap fallback look sharp, and make radiometrically biased pixels look visually polished.

The first fixes to prioritize are: remove the odd-shift downsampling failure; use the measured 29%/stage metadata as a geometric constraint; retain per-edge shifts and solve/refine per-tile positions at subpixel precision; refuse rather than silently abut on failed geometry; flat-field before projection/blending; and persist coverage, geometry residuals, fallback state, Z provenance, and validity in the output.
