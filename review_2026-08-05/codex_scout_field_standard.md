# Field-standard review of `stitcher.py` for microscope tile mosaics

Generated: 2026-08-05 22:48:49 JST  
Reviewed implementation: [`stitcher.py`](./stitcher.py)  
Scope: whole-slide and multi-well mosaics acquired on a regular XY grid with approximately 30% overlap.

## Executive judgment

`stitcher.py` is a thoughtful **fast affine-lattice stitcher**, but it is **not a field-standard robust stitcher for unattended whole-slide or plate-scale work**. Its multi-width strip search, explicit strip-alias candidates, high-pass NCC scoring, robust consensus, native-bit-depth handling, and reuse of one reference geometry across channels are all sound ideas. The decisive limitation is that it throws away the neighbor-specific registrations after estimating one fixed 2-D step vector for X and one fixed 2-D step vector for Y. Every tile is then forced onto that lattice at an integer-pixel coordinate.

The accepted microscopy tools do not generally make that final simplification. NIST MIST, Preibisch's Fiji Grid/Collection Stitching, and ASHLAR measure neighbor-specific translations, reject or constrain unreliable edges, and derive an individual global position for every tile. BaSiC supplies the complementary flat-/dark-field correction that registration and feathering cannot provide.

The fixed-step approximation is defensible only for a calibrated, repeatable, planar acquisition in which edge-wise validation demonstrates subpixel residuals with no row-, column-, scan-direction-, or distance-dependent trend. It is best used as an initialization or fallback model. For a large research-grade mosaic, per-tile positions should be solved globally from all trustworthy horizontal and vertical overlaps, with the regular grid or stage model acting as a prior.

The code's feathering is already a form of linear distance-weighted blending. It is adequate when geometry and radiometry are correct. Multiband blending is optional for qualitative presentation, not a prerequisite for correct microscopy stitching and not a substitute for flat-field correction.

## 1. What constitutes the field standard

There is no single ISO-style algorithm or universal NCC cutoff for microscope stitching. The practical field standard is the common workflow embodied by the established, published tools:

1. Correct sensor-/illumination-fixed shading, preferably from measured flat and dark references or retrospectively per channel with BaSiC.
2. Use stage metadata, nominal grid geometry, and overlap as a physical prior to build a graph of genuinely adjacent tiles.
3. Register every informative adjacent overlap independently, usually by phase correlation followed by real-overlap correlation scoring. Test multiple Fourier peaks/aliases, use windowing or feature-enhancing filters where appropriate, and refine to subpixel precision.
4. Reject or down-weight weak, implausible, or globally inconsistent pair registrations. Do not assume that phase correlation's largest peak is valid merely because a peak always exists.
5. Compute a globally consistent position for each tile using all reliable constraints, a graph method, or a robust global solve. Use stage/grid geometry to constrain or place uninformative tiles.
6. Correct radiometry before fusion, then use a documented overlap blend. Export per-edge scores, global residuals, outliers, loop-closure errors, and tile positions so correctness can be audited.

### Established methods and what they add

| Method | Accepted algorithmic role | What a naive fixed-step implementation omits |
|---|---|---|
| **NIST MIST** | Registers every horizontal and vertical adjacent pair with Fourier phase correlation. For real images it examines multiple correlation peaks, enumerates the four wrapped 2-D translations associated with each peak, and selects candidates using NCC on the actual overlap. It estimates nominal overlap, camera/stage-axis angle, stage repeatability, and actuator backlash; refines pair translations inside the physically plausible stage-error region; and uses an NCC-weighted graph/maximum spanning tree to obtain per-tile global positions. [Chalfoun et al., 2017](https://doi.org/10.1038/s41598-017-04567-y); [official NIST overview](https://pages.nist.gov/MIST/) | Multiple-peak disambiguation; row-/column-dependent mechanical behavior; a stage repeatability/backlash model; physically constrained local refinement; per-edge confidence and positions; graph-based global placement; diagnostic pairwise/global-position files. MIST explicitly notes that even a good stage has about 1–2 µm repeatability and that requested overlap can vary materially. |
| **Preibisch / Fiji Grid-Collection Stitching** | Finds a translation for each overlapping pair using phase correlation, validates candidates by direct correlation on the implied overlap, and gives each tile its own translation. It globally minimizes the sum of pairwise transfer errors by least squares, avoiding sequential propagation, and removes inconsistent links using correlation and displacement tests. Current Fiji supports grid/stage priors, arbitrary collections, subpixel placement, multiple channels/time points, and linear/nonlinear distance-weighted fusion. [Preibisch, Saalfeld & Tomancak, 2009](https://doi.org/10.1093/bioinformatics/btp184); [official Fiji documentation](https://imagej.github.io/plugins/image-stitching) | A joint all-edge consistency solve; loop redundancy; link rejection based on global displacement; individual tile coordinates; subpixel interpolation; auditable average and maximum displacement. |
| **ASHLAR** | Builds the adjacency graph from Bio-Formats stage positions, registers the expected overlaps with phase correlation, and refines to 0.1-pixel precision using the Guizar-Sicairos method. It uses Laplacian/LoG preprocessing to suppress autocorrelation, scores the aligned overlap with NCC, derives a dataset-specific false-match threshold from 1,000 random non-neighbor pairs, and applies a physical translation limit. Accepted edges form a minimum-spanning forest; a fitted stage-position model places disconnected or uninformative tiles. ASHLAR also registers imaging cycles directly to a common reference and writes pyramidal OME-TIFF. [Muhlich et al., 2022](https://doi.org/10.1093/bioinformatics/btac544); [official computational method](https://labsyspharm.github.io/ashlar/overview/DetCompMethods.html) | Stage-driven adjacency; decorrelation filtering; data-adaptive false-match calibration; physical shift bounds; subpixel transforms; per-edge corrections and per-tile positions; model-based recovery of sparse/background tiles; direct-to-reference cycle alignment. ASHLAR's spanning tree can itself accumulate error along long paths, but it still retains neighbor-specific shifts rather than imposing one fixed step. |
| **BaSiC** | A radiometric preprocessing method, not a registration algorithm. It retrospectively estimates a multiplicative flat field and additive dark field from an image collection using low-rank and sparse decomposition, and can also correct temporal baseline drift. It should be estimated separately for each channel/optical configuration and applied to raw linear-intensity tiles before registration and blending. [Peng et al., 2017](https://doi.org/10.1038/ncomms14836) | Spatially varying gain/vignetting, camera offset/dark signal, and temporal background drift. Mean subtraction, high-pass scoring, scalar normalization, and feathering cannot invert this image-formation model or restore quantitative intensity. |

These methods are complementary. BaSiC or measured flat-/dark-field correction addresses radiometry; MIST, Fiji, or ASHLAR addresses geometry and global placement; blending only composes already corrected and registered tiles.

## 2. What `stitcher.py` does well

- `_read_plane()` preserves native 8-/16-bit intensity rather than coercing through an 8-bit luminance conversion (`stitcher.py:186–201`).
- `_pair_candidates()` searches overlap strips at 12%, 20%, 30%, 45%, and 60% and explicitly adds the main-axis `-1/0/+1` strip-width aliases (`stitcher.py:265–312`). This is a useful mitigation for the circular ambiguity created by using cropped strips, and the 30% hypothesis matches the stated acquisition well.
- Candidates are ranked by NCC on downsampled high-pass images (`stitcher.py:215–244`, `294–310`). This reduces low-frequency illumination bias and uses the real implied overlap rather than trusting the raw phase peak alone.
- `_consensus()` seeks a shift cluster supported by many neighbor pairs instead of trusting one noisy pair (`stitcher.py:315–342`). For an exceptionally regular stage, that robust lattice estimate can outperform a sequential chain of noisy measurements.
- The two fixed basis vectors contain cross-axis components. Thus a constant camera-to-stage angle or global grid shear can be represented: X motion may include `dy`, and Y motion may include `dx` (`stitcher.py:370–381`). The implementation is more capable than a scalar overlap percentage.
- Measuring geometry once on a textured reference plane and reusing it for all channels (`stitcher.py:436–506`) is good practice when channels are intrinsically co-registered. MIST likewise supports reassembling other channels from a previously computed position model. Channel-specific chromatic or camera offsets still require calibration.
- The output fusion preserves the source integer depth and normalizes accumulated weights (`stitcher.py:399–420`).

## 3. Material departures from the standard

### 3.1 Local measurements are collapsed into a fixed integer lattice

The central assumption is stated directly in `_consensus()`: “Every neighbour pair in a plate shares the same step” (`stitcher.py:315–320`). `estimate_steps()` reduces all horizontal edges to one vector and all vertical edges to another (`stitcher.py:345–367`), after which `tile_origins()` enforces

\[
p(i,j)=i\,s_x+j\,s_y.
\]

No neighbor-specific translation survives into placement (`stitcher.py:370–381`, `490–492`). There is no per-tile correction, stage-repeatability/backlash model, weighted graph, robust global pose solve, loop closure, or spatial residual map.

This choice has one real benefit: it avoids a random-walk accumulation of independent edge noise. It replaces that problem with two others:

- Every real local stage error remains as a seam mismatch because a tile cannot move away from the lattice.
- Any systematic error in a basis vector grows linearly with grid distance. At tile index `n`, a step bias `delta` produces approximately `(n-1) * delta` far-edge drift.

For example, a 2048-pixel tile at exactly 30% overlap has a nominal step of 1433.6 pixels. Restricting that step to 1434 pixels leaves a 0.4-pixel systematic error per interval; across 20 intervals, the far edge is displaced by about 8 pixels even before mechanical error is considered. Subpixel estimation alone does not solve local stage variation, but integer-only global steps make systematic accumulation especially easy.

### 3.2 Phase-correlation disambiguation is useful but incomplete

`_phase_corr()` takes only the single largest circular-correlation peak and returns an integer position (`stitcher.py:247–262`). The multi-width/`+/-1` strip aliases are a good local heuristic, but they do not replace the standard safeguards:

- inspect several strong peaks, not only one per strip width;
- enumerate all physically possible 2-D wrapped translations;
- use a stage/nominal-overlap search bound;
- taper or pad crop boundaries to reduce spectral leakage;
- measure peak separation or peak-to-sidelobe quality;
- reject candidates that violate the global neighbor graph.

MIST and Preibisch explicitly inspect multiple phase peaks and validate their wrapped candidates in real space. ASHLAR adds windowing/decorrelation, a physical shift limit, and a dataset-adaptive false-match test.

### 3.3 Illumination handling affects scoring only

The high-pass copy in `_prep_score()` is used only to rank candidates (`stitcher.py:215–224`, `294–310`). The raw, uncorrected strips still generate the phase-correlation candidates, and the raw tiles are blended into the output. Mean subtraction does not remove spatially varying multiplicative vignetting or additive dark-field structure.

The comment that vignetting is “the same in every tile” is only half the issue. The shading pattern is fixed in **sensor coordinates**, while the same specimen point moves from one sensor edge to the other in adjacent tiles. A strong center-to-edge field can therefore dominate sparse biological texture, favor zero or another broad false peak, and produce periodic intensity bands in the final mosaic. BaSiC or measured flat-/dark-field correction should precede both registration and fusion.

### 3.4 Confidence reporting cannot establish correctness

`MIN_PAIR_SCORE = 0.15` is an undocumented fixed heuristic (`stitcher.py:265–268`). The reported `peak_x` and `peak_y` are not phase-correlation peaks: they are mean NCC values for members supporting the winning consensus (`stitcher.py:315–342`, `509–515`). The code does not retain or report:

- every edge's selected shift and NCC;
- how many possible edges voted for the consensus;
- a non-neighbor/null NCC distribution;
- horizontal versus vertical residual distributions;
- rejected edges, loop errors, spatial trends, or held-out validation;
- peak-to-second-peak separation.

A good majority can mask a bad row or sparse region. Moreover, if a direction is unmeasurable or its mean NCC is below 0.15, the code silently changes the step to tile abutment—zero overlap—despite the known approximately 30% acquisition (`stitcher.py:361–366`). It flags low confidence afterward, but it still produces a geometrically implausible mosaic. A field-standard implementation should use stage/nominal geometry as the low-information fallback or stop that stitch as unvalidated, not silently discard 30% overlap.

### 3.5 Integer-only placement omits a visibly useful refinement

The raw correlation maximum, consensus median, and all origins are integers (`stitcher.py:247–262`, `341–342`, `370–381`). There is no local peak fit, upsampled DFT, or fractional-pixel resampling. [Guizar-Sicairos, Thurman & Fienup](https://doi.org/10.1364/OL.33.000156) provide the standard efficient subpixel refinement. ASHLAR uses 0.1-pixel precision and found a discernible improvement over integer placement, with diminishing returns beyond 0.1 pixel.

Even before other errors, rounding an otherwise exact 2-D translation has an RMS quantization floor of about 0.29 pixel per axis (about 0.41 pixel in 2-D for uniformly distributed fractional offsets). Reusing a rounded step turns that local quantization into a systematic grid-scale bias.

## 4. Known phase-correlation failure modes and this code

| Failure mode | Why it happens | Standard mitigation | Assessment of this code |
|---|---|---|---|
| **Peak aliasing / wraparound** | DFT correlation is circular; shifts are modulo crop size. Repeated cells, periodic plate features, sparse signal, and noise yield several plausible peaks. Crop boundaries add spectral leakage. | Expected stage/overlap bounds; tapering/padding; multiple peaks and all wrapped candidates; score the true overlap; reject weak/globally inconsistent edges. | Multiple strip widths and `+/-1` main-axis aliases are valuable, but only one peak per strip is retained, no 2-D alias enumeration or window is used, and no stage/global consistency test follows. |
| **Vignetting dominates correlation** | The illumination field is fixed to the detector, not the specimen. A shared gradient or edge falloff can correlate more strongly than sparse overlap content. | Flat-/dark-field correction (measured or BaSiC); gradient/Laplacian/LoG registration images; correlate only expected overlaps; validate on corrected intensities. | High-pass NCC ranking helps, but phase candidates and output intensities remain uncorrected. Feathering can hide a seam but cannot recover quantitative intensity. |
| **Translation-only assumption** | Basic phase correlation cannot directly solve relative rotation, magnification, shear, perspective, nonlinear lens distortion, focus-dependent warp, or specimen motion. | Translation is acceptable for a rigid planar sample, fixed optics, and a repeatable modern stage; otherwise calibrate distortion or fit affine/similarity/non-rigid transforms as physically justified. | The two basis vectors model constant grid angle/shear, but tile pixels are never rotated or warped and position-dependent distortion cannot be represented. |
| **Error across a large grid** | Sequential edge errors grow roughly as `sqrt(n)` when independent and as `n` when biased. A fixed lattice avoids the former but any step bias grows as `n`, while local deviations remain at every seam. | All-edge weighted/robust global positions, loop closure, stage priors, outlier rejection, and held-out edge validation. | No sequential random walk, which is good; however, no per-tile solve or loop closure exists, and integer/systematic step error grows with distance. |
| **Subpixel accuracy** | The discrete peak supplies only an integer shift; interpolation and sampling affect fractional estimates. | Local upsampled DFT or a validated peak fit, then subpixel resampling; ASHLAR uses 0.1-pixel precision. | Entirely integer. This is a material weakness for cellular/punctate data and large grids. |
| **Stage backlash / repeatability** | Position error changes with travel direction and after reversals. Serpentine scans can have alternating row offsets; good stages still have finite repeatability. | Read stage positions; model repeatability/backlash and camera angle; retain edge-specific corrections; compare residuals by direction, row, and column. MIST was designed around this problem. | One X vector and one Y vector force both scan directions and every row/column to agree. Direction-dependent or post-reversal errors are averaged away rather than corrected. |

## 5. Is one global step acceptable?

Strictly, this implementation uses **two** global 2-D basis vectors—one reused for every `+X` neighbor and one for every `+Y` neighbor. That is more expressive than one scalar step, but far less expressive than one optimized position per tile.

### Conditions under which it can be acceptable

The approximation can be acceptable for a fast preview or validated production path when all of the following hold:

- the sample is rigid, planar, and static;
- camera, objective, focus, and magnification remain fixed;
- the stage is closed-loop or otherwise repeatable to well below the allowed image-space residual;
- overlap is uniform and all tiles share the same dimensions;
- scan-direction reversal produces no detectable backlash pattern;
- shading has been corrected and overlaps have sufficient non-periodic texture;
- the grid is modest enough that fractional-step bias does not become significant;
- independent edge-wise measurements show no row/column/direction trend and satisfy the residual targets below, including at the far edges.

### Conditions under which it breaks

It breaks or becomes scientifically risky with serpentine backlash, row-/column-dependent step, thermal drift, open-loop stage repeatability, fractional pixel pitch accumulated over many fields, incorrect nominal overlap, nonlinear stage calibration, lens distortion, slide tilt/focus change, local specimen motion, sparse/background overlaps, repeated texture, missing acquisition metadata, or channel-/cycle-specific optical transforms.

For whole-slide or large multi-well mosaics, the default should therefore be per-tile optimization. A suitable model keeps a 2-D origin `p_i` for every tile and uses the regular lattice/stage mapping `A s_i` as a prior, for example:

\[
\min_{p,A}\;\sum_{(i,j)\in E} w_{ij}\,\rho\!\left(\left\|(p_j-p_i)-t_{ij}\right\|\right)
+\lambda\sum_i\left\|p_i-A s_i\right\|^2,
\]

where `t_ij` is the measured neighbor translation, `w_ij` comes from calibrated match confidence, `rho` is a robust loss, and `s_i` is the nominal grid or stage coordinate. This distributes inconsistent measurements, rejects outliers, preserves a physically plausible grid, and still places sparse tiles. With approximately 30% overlap, a regular grid supplies abundant horizontal and vertical constraints for this solve.

## 6. Is feather blending adequate?

Yes—**when alignment and radiometry are already correct**. “Feather blending” is normally linear distance-weighted blending, so feather versus linear blending is not a meaningful opposition. Preibisch's formulation uses distance-to-edge weights: exponent `alpha=0` gives averaging, `alpha=1` gives linear feathering, and larger exponents give a steeper nonlinear blend. The code's normalized distance-to-nearest-border weights (`stitcher.py:391–420`) are in this accepted baseline family.

Linear feathering is generally adequate for a static, flat-field-corrected microscopy grid with subpixel placement. It reduces an abrupt boundary but cannot fix:

- doubled/blurred objects from geometric error;
- vignetting, dark-field offset, exposure/gain changes, or photobleaching;
- moving cells or focus-dependent content;
- quantitative bias already present in the source tiles.

[Burt–Adelson multiband blending](https://doi.org/10.1145/245.247) blends low spatial frequencies over a broad zone and fine detail over a narrow zone. It is useful for a **qualitative display mosaic** when broad illumination/color seams remain after the best available radiometric correction. It is not required by MIST, Fiji, or ASHLAR. For quantitative fluorescence it can be less desirable because the result is no longer a simple spatially weighted average of measured intensities. It also cannot make an incorrect registration correct. The microscopy-specific study by [Piccinini & Bevilacqua](https://doi.org/10.1155/2018/7082154) likewise distinguishes blending from radiometric/vignetting correction.

Recommended policy: correct flat/dark field first; use simple linear feathering for the quantitative mosaic; optionally generate a separately labeled multiband visualization if cosmetic low-frequency seams remain.

## 7. Plausible numerical acceptance ranges

These are engineering QC bands, not universal physical constants. Pixel size, optical resolution, modality, signal occupancy, preprocessing, and the exact correlation definition matter. Report the metric definition and distributions, not only a single mean.

### Geometric residual

For an informative neighbor edge `(i,j)`, define the global consistency residual as

\[
r_{ij}=(p_j-p_i)-t_{ij}.
\]

For a least-squares/robust global solution, report the Euclidean norm of this vector for every retained and rejected horizontal/vertical edge. For a spanning-tree placement, validate on non-tree edges or re-register the final overlaps; tree-edge residuals are zero by construction and cannot prove accuracy.

| Residual level | Practical interpretation for a well-textured 30%-overlap microscopy grid |
|---|---|
| Median **0.1–0.3 px**, upper tail about **<=0.5 px** | Excellent; consistent with a high-quality subpixel pipeline. |
| Median **0.3–0.5 px**, 95th percentile about **<=1 px** | Strong and normally suitable for cellular/punctate work. |
| Median **0.5–1.0 px**, 95th percentile **1–2 px** | Often acceptable for routine overview mosaics, but inspect sharp structures and downstream measurement sensitivity. |
| Persistent **1–2 px** local residuals or a spatial/directional trend | Marginal for single-cell or punctate analysis; investigate the transform/stage/shading model. |
| **>=2–3 px** on informative overlaps | Generally visible and a failure for cell-scale quantitative work. |

These bands are anchored by established guidance rather than presented as a formal standard: the [official Fiji documentation](https://imagej.net/imagej-wiki-static/Image_Stitching) says average and maximum global displacements should be around or below 1 pixel when there is no major alignment error, and the paper gives successful example adjustments around 0.39–0.77 pixels on average and 0.64–1.18 pixels maximum. ASHLAR uses 0.1-pixel estimator precision, reports a discernible gain over integer placement, and reported approximately 0.2-pixel median local consistency on one benchmark. Published colony-centroid or object-position errors, such as MIST's task-level measurements, include segmentation, acquisition, and other effects and must not be substituted for seam residuals.

### Normalized cross-correlation

For zero-mean NCC evaluated on the valid, corrected, post-shift overlap:

| NCC | Interpretation |
|---|---|
| **>0.7** | Fiji's documented typical range for a good registration; strong evidence when the overlap is textured and shading/repetition are controlled. |
| **0.3–0.7** | Ambiguous/permissive. It can be valid for sparse or noisy fluorescence, but needs a physical shift bound, a distinct correlation peak, calibrated null distribution, and global/loop consistency. |
| **<0.3** | Ordinarily weak evidence. Accept only after dataset-specific validation; otherwise reject or use the stage/grid prior. |

There is deliberately no universal pass threshold. ASHLAR estimates the false-match boundary per dataset as the 99th percentile of NCC from 1,000 random non-neighbor pairs (an empirical one-sided `p=0.01`) and combines it with a physical translation limit. A repeated pattern or fixed shading field can also produce a high NCC for the wrong shift, so NCC is necessary evidence, not proof.

The code's `0.15` cutoff is therefore too permissive for unattended acceptance and is especially weak because it is applied to an aggregate consensus score rather than each edge. Its high-pass/downsampled NCC is not numerically identical to Fiji's regression measure, so the exact replacement threshold must be calibrated on representative the vendor data; the methodological conclusion remains that `0.15` alone cannot certify a correct stitch.

At minimum, a correct run should report median, 5th percentile, and minimum accepted-edge NCC; median and 95th-percentile residual by direction; rejected-edge count; largest loop-closure error; row/column/direction trends; and overlay crops from representative seams and four-tile junctions.

## 8. Recommended implementation priorities

1. **Retain every neighbor measurement.** Store multiple candidate shifts, the selected subpixel shift, NCC, peak-separation quality, and physical plausibility for every horizontal and vertical edge.
2. **Use a physical prior.** Read OME/vendor stage positions if available; otherwise use grid indices and the nominal 30% overlap. Treat the current consensus basis vectors as an affine-lattice prior, not final positions.
3. **Improve pair registration.** Correct flat/dark field, window/filter registration images, inspect multiple peaks/2-D aliases, and refine the winner with an upsampled DFT to about 0.1-pixel numerical precision.
4. **Calibrate and reject edges.** Use a non-neighbor NCC null distribution or a validated modality-specific rule, a stage/overlap shift bound, and robust global inconsistency tests.
5. **Solve per-tile positions globally.** Use robust weighted least squares over all good edges with lattice/stage regularization, or a documented MIST/ASHLAR-style graph strategy. Preserve loop edges for validation even if a tree is used for placement.
6. **Fail safely.** If a direction is unmeasurable, fall back to known stage/nominal 30% geometry and mark the result unvalidated, or stop. Do not silently switch to zero-overlap abutment.
7. **Keep linear feathering as the default.** Add multiband fusion only as an explicitly non-quantitative display option. Do not use blending to mask geometric or radiometric defects.
8. **Export QC and positions.** Save per-edge diagnostics, final tile coordinates, residual/outlier tables, and a pyramidal OME-TIFF suitable for whole-slide inspection.

## Bottom line

For a small, demonstrably rigid and repeatable grid, the present global-step approach may generate a visually good result and its robust consensus can be useful. For general whole-slide or multi-well plate stitching, particularly over many fields, it is below the standard established by MIST, Preibisch/Fiji, and ASHLAR because it has no subpixel neighbor-specific placement, stage-aware edge validation, per-tile global optimization, or residual-based QC. BaSiC/flat-field correction is also missing. Feather blending is not the blocker; geometry, illumination correction, and evidence of correctness are.

## Primary references

- Chalfoun, J. et al. (2017). [MIST: Accurate and Scalable Microscopy Image Stitching Tool with Stage Modeling and Error Minimization](https://doi.org/10.1038/s41598-017-04567-y). *Scientific Reports* 7, 4988. See also the [official NIST MIST site](https://pages.nist.gov/MIST/) and [official user guide](https://github.com/usnistgov/MIST/wiki/User-Guide).
- Preibisch, S., Saalfeld, S. & Tomancak, P. (2009). [Globally optimal stitching of tiled 3D microscopic image acquisitions](https://doi.org/10.1093/bioinformatics/btp184). *Bioinformatics* 25, 1463–1465. See also the [official Fiji Image Stitching documentation](https://imagej.github.io/plugins/image-stitching).
- Muhlich, J. L. et al. (2022). [Stitching and registering highly multiplexed whole-slide images of tissues and tumors using ASHLAR](https://doi.org/10.1093/bioinformatics/btac544). *Bioinformatics* 38, 4613–4621. See also the [official ASHLAR computational method](https://labsyspharm.github.io/ashlar/overview/DetCompMethods.html).
- Peng, T. et al. (2017). [A BaSiC tool for background and shading correction of optical microscopy images](https://doi.org/10.1038/ncomms14836). *Nature Communications* 8, 14836.
- Guizar-Sicairos, M., Thurman, S. T. & Fienup, J. R. (2008). [Efficient subpixel image registration algorithms](https://doi.org/10.1364/OL.33.000156). *Optics Letters* 33, 156–158.
- Burt, P. J. & Adelson, E. H. (1983). [A multiresolution spline with application to image mosaics](https://doi.org/10.1145/245.247). *ACM Transactions on Graphics* 2, 217–236.
- Piccinini, F. & Bevilacqua, A. (2018). [Colour Vignetting Correction for Microscopy Image Mosaics Used for Quantitative Analyses](https://doi.org/10.1155/2018/7082154). *BioMed Research International* 2018, Article 7082154.
