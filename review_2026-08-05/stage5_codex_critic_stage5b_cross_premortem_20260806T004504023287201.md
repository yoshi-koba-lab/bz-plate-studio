# Codex-Critic Stage-5b CROSS PRE-MORTEM

**Target:** `stitcher.py` as present on 2026-08-06  
**Method:** executable, ground-truthed NumPy/PIL experiments against the current functions, not inspection alone  
**Safety:** experiments were in memory with `PYTHONDONTWRITEBYTECODE=1`; no existing file was edited, replaced, truncated, moved, or deleted; this report is a new uniquely named file

## Executive verdict

| Attack | Risk being tested | Verdict | Strongest measured failure |
|---|---|---|---|
| (a) | A dense/stationary specimen is learned as flat-field | **NOT-RULED-OUT** | At 90% coverage, true bright/dim ratio `1.379` became `0.936`; intensity ordering inverted |
| (b) | Flat-fielding changes geometry or geometric QC | **NOT-RULED-OUT** | A correct `(-1,113)` X step became abut `(0,160)`, a `47.01 px` error |
| (c) | The 12% ambiguity gate rejects a good non-periodic well | **NOT-RULED-OUT** | In a 21-tile connected-well sweep, `60/406` otherwise-correct wells fell back; a native-size case incurred `538 px` X-step error |
| (d) | One bad donor poisons plate geometry | **NOT-RULED-OUT** | An `n=1` false donor made an inherited well's far corner `120 px` wrong; with one good donor the median invented a still-wrong hybrid |
| (e) | Residual QC self-grades and is optimistic | **NOT-RULED-OUT as placement-accuracy QC** | Reported residual median/p95/max `0/0/0` versus independent truth `28/32/32 px` |
| (f) | Float32/NaN/dtype paths corrupt saved pixels | **NOT-RULED-OUT overall** | A uint8 reference forced an end-to-end uint16 channel entirely to `255`; mean-Z also quantized before `blend()` |

The narrow claim that ordinary finite uint8/uint16 flat-field arithmetic itself overflows float32 is **RULED OUT**. The narrow claim that reuse of an estimator automatically forces every residual to zero is also **RULED OUT**. Neither narrow result rescues the overall upgrades because several reachable failure mechanisms remain.

## Test protocol and interpretation

- Tests imported the current `stitcher.py` and exercised `estimate_flatfield`, `apply_flatfield`, `_pair_candidates`, `estimate_steps`, `plate_geometry`, `_subpixel`, `_edge_report`, `blend`, and full `stitch_well` call paths.
- Full-size tests used the stated BZ-X dimensions `1374×1832` and truth `step_x=(-3,1294)`, `step_y=(970,2)` where noted. Scaled tests preserved approximately 29% overlap and the 21-tile 5×5-minus-corners topology.
- Geometry was judged against known origins, not against the estimator's own output.
- For I/O-free end-to-end tests, `_read_plane` was temporarily mapped to in-memory arrays and restored in `finally`; algorithm code was not patched.
- Some sweeps temporarily set `AMBIGUITY_MARGIN=0` at runtime as a control and restored `0.12`. No source file was changed.
- These are constructed counterexamples and sensitivity measurements. Their percentages quantify the stated synthetic regimes, not prevalence on an unknown population of real plates.

## Cross-failure chain

The upgrades can compound rather than fail independently:

```text
camera-coordinate artifact
    -> confidently wrong but plausible "measured" donor
    -> immediately accepted n=1 plate prior
    -> diffuse/non-periodic wells lose their own votes to the ambiguity gate
    -> those wells inherit the poisoned prior
    -> same-data residual reports internal agreement, not external accuracy
```

The broad overlap gate does not stop this example: false `dx=192` in a 320-pixel tile and false `dy=144` in a 240-pixel tile both imply 40% overlap, inside `OVERLAP_RANGE=(0.03,0.75)`.

---

## (a) Dense/stationary biology is absorbed into the flat-field

**Verdict: NOT-RULED-OUT; directly demonstrated.**

### Mechanism

At detector coordinate `p`, a tile approximately contains

```text
I_i(p) = S(p) * B_i(p),
```

where `S` is illumination and `B_i` is biology. For positive `S`, the estimator at `stitcher.py:526-534` computes approximately

```text
q25_i[I_i(p)] = S(p) * q25_i[B_i(p)].
```

If specimen occupancy exceeds 75%, the 25th percentile is still specimen. If biology moves only a few pixels between fields, `q25[B_i(p)]` remains spatially structured in camera coordinates. The large box smoothing removes fine texture but preserves broad expression domains, tissue envelopes, and gradients. Normalization to mean one and clipping cannot tell that structure from illumination. Division then flattens real biology.

### Numeric evidence

Native-size experiment:

- 21 tiles of `1374×1832`;
- `make_well.vignette(strength=0.55)`;
- seed `20260806`;
- non-periodic dense biology formed from broad Gaussian features and asymmetric X/Y trends, background `22`;
- specimen motion only ±8 pixels between tiles, gain SD 1.2%, noise SD 0.6;
- default `lo_pct=25`, `smooth_frac=0.12`.

| Coverage | Field RMSE versus true shade | Object-mean bias | Bright/dim ratio, truth → corrected | Object/background, truth → corrected | Image NRMSE |
|---:|---:|---:|---:|---:|---:|
| 90% | `0.2371` | `-3.70%` | `1.379 → 0.936` | `7.18 → 2.48` | `20.62%` |
| 100% | `0.1217` | `+0.31%` | `1.402 → 1.042` | n/a | `11.42%` |

At 90% coverage, the nominally bright biology became dimmer than the nominally dim biology. Within-object truth/corrected correlation was `-0.135`, regression slope was `-0.117`, and background rose from `22.27` to `62.08`. At 100% coverage the mean looked nearly perfect (`+0.31%`) while biological contrast was badly flattened, so mean-only validation would miss the damage.

A separate `300×400` 80%-coverage run was worse: field RMSE `0.3323`, object bias `-8.30%`, bright/dim `1.375 → 0.892`, object/background `7.26 → 2.30`, NRMSE `28.46%`, and within-object correlation `-0.191`.

An independent stationary-map test gave the same mechanism: with zero to ±24-pixel motion, a true bright/dim ratio near `2.0` became `1.000-1.016`, retaining at most 2.6% of the intended contrast. The correlation between `estimated_field / true_shade` and biology was `0.965-0.996`.

### Fix

1. Prefer an instrument/control-slide flat acquired without specimen.
2. If no control exists, estimate illumination from overlap correspondences: the same world pixel observed at different detector coordinates separates `S` from biology better than a detector-coordinate percentile.
3. Require demonstrated background support at each detector coordinate. Disable correction and emit a QC reason when support is inadequate.
4. Do not enable this low-percentile method by default for confluent or nearly stationary specimens.
5. Report field range/clipping, background-support fraction, tile-coordinate stability, and field/specimen correlation; validate biological contrast on held-out structures, not just the corrected mean.

---

## (b) The flat-field option changes geometry and residuals

**Verdict: NOT-RULED-OUT; directly demonstrated. Ground truth selected the raw geometry in the failure case.**

### Mechanism

This is setting-dependent by construction. `stitcher.py:659-670` flat-fields the reference images before both `estimate_steps()` and `_edge_report()`. Division changes local noise, high-pass content, candidate ranking, NCC, and the hard `MIN_PAIR_SCORE=0.15` decision. An output-intensity option can therefore change the lattice itself.

### Numeric evidence

Control: ordinary full-size `make_well`, seed 0.

- Truth: `step_x=(-3,1294)`, `step_y=(970,2)`.
- Raw: both exact, peaks `0.60785 / 0.92226`.
- Flat-fielded: both exact, peaks `0.61336 / 0.91994`.

Thus failure is not universal. Even with identical geometry, however, a scaled ordinary run changed residual median/p95 from `0.00/0.10 px` raw to `0.00/0.00 px` corrected and edge NCC from `0.919` to `0.993`. The reported QC describes the chosen preprocessing, not an invariant acquisition property.

Ground-truthed failure:

- 21-tile connected topology, tile `120×160`;
- truth `step_x=(-1,113)`, `step_y=(85,1)`;
- fully dense, non-periodic scene made from two independent smoothed Gaussian-random fields, baseline 110;
- vignette strength 0.65, noise SD 6, seed 3;
- estimated flat-field was close to the true shade: RMSE `0.01215`, range `0.5796-1.2861`.

| Processing | Returned X step | X consensus | Source | X-step truth error |
|---|---:|---:|---|---:|
| Raw | `(-1,113)` | `0.150589` | measured | `0 px` |
| Estimated flat-field | `(0,160)` | `0.144064` | abut | `47.01 px` |

Raw also recovered Y exactly with peak `0.169039`. All 16 X pairs survived in both variants and ambiguity drops were zero; the failure was solely a preprocessing-induced transition across `0.15`.

The full-resolution QC contradicted the fallback:

| Pixels and lattice used for QC | Residual median | Residual p95 | Edge NCC median |
|---|---:|---:|---:|
| Raw pixels at true/raw lattice | `0.64` | `1.00` | `0.508` |
| Corrected pixels at true lattice | `0.57` | `0.64` | `0.633` |
| Corrected pixels at returned abut lattice | `23.62` | `47.41` | `0.633` |

The corrected pixels supported the true lattice better in `_edge_report`, yet `_prep_score` consensus rejected it. Ground truth proves raw was right. A plate prior would hide the abut result by changing the source to `plate`, but would not remove the setting dependence.

### Fix

1. Estimate geometry from an immutable raw/high-pass reference; apply specimen-derived flat-fielding only after geometry is fixed.
2. Report raw geometric QC separately from optional corrected-image photometric/seam QC.
3. If raw and corrected candidates are both computed, expose disagreement and never silently let the output-intensity checkbox choose the lattice.
4. Calibrate `MIN_PAIR_SCORE` for the exact preprocessing/noise model, and use support plus uncertainty rather than a single hard threshold.

---

## (c) `AMBIGUITY_MARGIN=0.12` discards correct, non-periodic evidence

**Verdict: NOT-RULED-OUT; directly demonstrated on connected wells and at native size.**

### Mechanism

`_pair_candidates()` applies the uniqueness test at `stitcher.py:366-370` and returns an empty list before cross-edge consensus at `374-401`. This reverses the strongest available disambiguation:

```text
current: each edge -> close rival -> delete all candidates -> multi-edge consensus
needed:  each edge -> retain weighted candidates -> multi-edge consensus -> aggregate uniqueness test
```

Diffuse cells, colonies, and tissue boundaries create broad NCC lobes. Aperiodically placed similar objects create rivals whose locations vary by edge, while the true stage shift is common. The fixed `_TOL=6` can also call two samples from one morphology-dependent broad lobe “distinct.” Correct votes are destroyed before their cross-edge agreement can be used. Additionally, plausibility filtering occurs later in `estimate_steps()` at `431-439`, so `_pair_candidates()` can in principle let a rival veto an edge before the rival is checked against `OVERLAP_RANGE`.

### Numeric evidence: conventional connected 21-tile wells

- 5×5-minus-corners topology: 21 tiles, 16 X edges, 16 Y edges.
- Tile `160×220` (H×W).
- Truth `step_x=(-1,155)`, `step_y=(113,0)`, 29.5%/29.4% overlap.
- Specimen: 1,100 RNG-positioned impulses with lognormal amplitudes, Gaussian-smoothed into diffuse cell/tissue bodies, plus one unique off-centre elliptical envelope and independent per-tile Gaussian acquisition noise.
- No grating, repeated tile, or periodic placement.

| Synthetic regime | Correct with margin 0 | Lost correctness at 0.12 | Fell to abut |
|---|---:|---:|---:|
| Blur radius 30, contrast 15, noise SD 1 | `30/30` | `5/30 (16.7%)` | `4/30 (13.3%)` |
| Blur radius 34, contrast 15, noise SD 1 | `19/30` | `13/19 (68.4%)` | `13/19 (68.4%)` |

The second row is conditional on the 19 wells correctly measurable with the gate disabled.

Representative radius-34 seed 3:

- Truth: X `(-1,155)`, Y `(113,0)`.
- Margin 0: X `(0,154)`, peak `0.167888`; Y `(112,0)`, peak `0.173333`; both measured.
- Margin 0.12 rejected `9/16` X and `11/16` Y edges.
- Every rejected list still contained a truth-within-6-pixel candidate; `11/20` rejected edges had truth ranked first.
- Remaining consensus fell to `0.124470/0.130529`, so output became X abut `(0,220)` and Y abut `(160,0)`.
- Median best/rival gaps on rejected edges were only 5.43% X and 5.33% Y. Failure began at margin 0.06 for Y and 0.08 for both axes.

Across a larger scaled condition sweep, 406 of 550 wells were correct with the gate disabled. The gate made 67/406 (`16.5%`) lose correctness and 60/406 (`14.8%`) fall back. It gated 2,912 of 6,600 edges; 2,896/2,912 (`99.45%`) of those lists still contained a truth-within-6 candidate.

### Numeric evidence: native BZ-X size

- Tile `1374×1832`, exact truth `step_x=(-3,1294)`, `step_y=(970,2)`.
- Connected 3×3 grid.
- Twenty randomly located, independently sized, oriented, and intensified elliptical Gaussian colonies; noise SD 1; seed 9.

For X, margin 0 recovered `(0,1295)`, only `3.16 px` from truth, with peak `0.630856`. Margin 0.12 rejected `4/6` X edges. Three rejected winners were within `3.16`, `3.16`, and `4.24 px` of truth, and beat their distinct rivals by only `1.87%`, `2.16%`, and `0.48%`. The surviving consensus was `0.074733`, producing abut `(0,1832)`, a `538 px` X-step error. Supplying the true prior changed only the fallback source to `plate`.

### Does the user notice?

Partially, and not diagnostically:

- The GUI emits generic “geometry inherited from the plate” or “geometry unmeasurable — tiles abutted” warnings.
- `stitch_qc.csv` includes source, `low_confidence`, and `ambiguous_edges`.
- No warning says the 12% gate discarded otherwise consensus-resolvable candidates.
- `ambiguous_edges` conflates this gate with every other empty-candidate cause.
- The plate warning does not say to inspect the inherited well, only the first 12 warnings appear in the modal, and TIFF/PNG outputs contain no QC marker.
- If enough edges survive with peak at least 0.15, the well can remain “measured”; the ambiguity count then lives only in the CSV and triggers no GUI warning.

### Fix

1. Never return `[]` solely because one pair has a close rival.
2. Return candidate clusters plus pair-level gap/uncertainty; downweight ambiguity rather than deleting evidence.
3. Perform cross-edge consensus first, then compare the aggregate winning shift cluster with the aggregate runner-up.
4. Make the same-peak radius feature-scale-aware and apply physical plausibility before a rival can veto evidence.
5. Report rejected/total edges and an explicit fallback reason such as `pair_ambiguity_gate`.
6. Add regression tests with aperiodic random colonies and diffuse tissue, asserting that a common multi-edge shift is not weakened by pair-level uniqueness filtering.

---

## (d) One bad donor can poison the online plate consensus

**Verdict: NOT-RULED-OUT; directly demonstrated through `stitch_well -> plate_geometry -> stitch_well(prior)`.**

### Mechanism

`plate_geometry()` admits a direction using only `peak >= 0.15` at `stitcher.py:466-467`. It has no minimum donor count, expected-step/calibration check, modal cluster, vector-distance outlier rejection, usable-edge threshold, or source/provenance check. It calculates spread but never enforces it. `main_stitch_v2.py:120-150` immediately seeds the prior from the first successful well and writes a well that uses a `plate` prior; only an `abut` result is deferred.

A single systematic false match can therefore become the plate. With two disagreeing donors, componentwise `np.median(...).astype(int)` creates a midpoint geometry that may belong to neither donor.

### Numeric evidence

Bad donor construction, seed 55:

- 4×4 grid, 16 tiles of `240×320`, 24 neighbor edges.
- Independent truth: `step_x=(0,224)`, `step_y=(168,0)`.
- Real scene: global non-periodic Gaussian random specimen plus acquisition noise.
- Camera-coordinate artifact: one iid random `240×128` patch copied into detector columns `0:128` and `192:320`, and one iid random `96×320` patch copied into rows `0:96` and `144:240`. These are duplicated non-periodic random patches, not a grating.

The actual functions returned:

- false `step_x=(0,192)`, peak `0.499512`;
- false `step_y=(144,0)`, peak `0.504324`;
- source measured, `low_confidence=False`;
- 24 usable edges, ambiguity 0, NCC median `0.506`;
- residual median/p95/max `0/0/0`.

`plate_geometry([bad_donor])` accepted exactly that prior with `n_x=n_y=1` and spread 0. A second, unmeasurable checker-like well inherited the false steps with source `plate`, peak 0, edges 0, residual `None`, and 24 ambiguous edges.

Consequences for the inherited 4×4 well:

- per-X-step error `32 px`, per-Y-step error `24 px`;
- far `(3,3)` tile displacement `120 px`;
- false output shape `672×896` versus true `744×992`.

Adding one clean measured donor did not repair consensus. The two admitted donors produced the componentwise midpoint:

- plate X `(0,208)`, spread 16;
- plate Y `(156,0)`, spread 12;
- next inherited well was still wrong by 16/12 pixels per step and `60 px` at the far corner.

Only after two good donors plus the bad donor did the median return truth. This is especially dangerous online: earlier inherited outputs are not retroactively rebuilt after the prior improves.

### Does the user notice?

- The poisoned donor has no warning: it is measured, above threshold, ambiguity 0, and residual p95 0.
- An inherited target gets only “geometry inherited from the plate.” Because the warning chain uses `elif`, that message masks the generic low-confidence warning.
- Plate donor IDs, `n`, spread, disagreement, and target-to-prior deviation are absent from the per-well geometry record and CSV.
- The target's 24 ambiguous edges appear in CSV but trigger no warning; residual is `None`, so the residual threshold cannot warn.

### Fix

1. Measure all candidate donor wells first; construct a stable plate model only afterward.
2. Require at least three agreeing measured donors, or one trusted instrument/metadata calibration. Reject an `n=2` disagreement rather than averaging it.
3. Cluster the two-dimensional step vectors and use the dominant tight mode; do not take an unconditional componentwise median across modes.
4. Enforce maximum spread/deviation and minimum usable-edge support. Treat a camera-coordinate common template as a possible systematic artifact.
5. Carry donor IDs, donor count, spread, expected-step deviation, and consensus status into every inherited geometry and warning.
6. Do not write inherited mosaics until consensus is trusted; rebuild earlier inherited wells if the consensus changes materially.

---

## (e) Same-estimator residual is internal consistency, not placement accuracy

**Verdict: NOT-RULED-OUT as an accuracy claim. The stronger claim that reuse mechanically forces a low residual is RULED OUT.**

### Mechanism

`estimate_steps()` fits a lattice from `_pair_candidates`. `_edge_report()` at `stitcher.py:718-746` re-runs `_pair_candidates` on the same images, takes the first winner, optionally refines it with `_subpixel`, and compares it with the fitted lattice. Ambiguous edges are omitted from residuals entirely.

This metric validly describes internal agreement among accepted matches. It is not independent evidence that those matches correspond to the same world features. Fitting and scoring the same edges introduces finite-sample optimism and selection bias; a shared systematic alias can make both stages agree perfectly while being wrong together.

### Numeric evidence: refuting the tautology

- Clean random specimen at true X/Y `(0,224)/(168,0)`: correct estimate and residual 0.
- Passing the deliberately wrong `(0,192)/(144,0)` lattice into `_edge_report()` on those same clean images produced residual median `28`, p95/max `32`, over 24 edges with NCC 1.

Therefore calling the same estimator does not automatically force zero.

With known random per-tile jitter in 20 independent 5×5 wells, the reported residual tracked independently computed per-edge displacement reasonably: mean fitted/known median ratio `0.988` (range `0.831-1.203`) and only `0.112 px` average lowering. A 12-well train/held-out split measured mean training median `7.37 px` versus held-out `7.78 px`: ordinary fit-data optimism was about 5.5%, with held-out larger in 8/12 wells.

### Numeric evidence: catastrophic shared failure

The camera-coordinate donor from (d) produced:

| Metric | `_edge_report` | Independent ground truth |
|---|---:|---:|
| Residual median | `0 px` | `28 px` |
| Residual p95 | `0 px` | `32 px` |
| Residual max | `0 px` | `32 px` |
| Far-corner displacement | not reported | `120 px` |

All 24 edges chose the same false detector artifact, ambiguity was 0, source was measured, and NCC median was `0.506`. The estimator and its QC self-confirmed the same systematic mechanism, so the GUI emitted no warning.

The ambiguity behavior adds another optimistic selection: edges for which `_pair_candidates()` returns empty contribute only to `ambiguous_edges`; they do not enter median, p95, max, or NCC. A low residual can therefore describe only the easy selected subset.

### Fix

1. Rename the fields to `internal_lattice_residual_*`; do not present them alone as seam-placement accuracy.
2. Add independent validation against stage metadata, a trusted calibration, expected plate step, or a separately estimated signal/frequency band.
3. Fit and validate on disjoint edges/features/channels. Holdout reduces ordinary fit optimism, although it will not catch a detector artifact shared by every edge.
4. Estimate and remove/whiten the common camera-coordinate template before both fitting and validation.
5. Report accepted/total edge fraction, ambiguous fraction, weak-NCC count, expected-step deviation, and a seam montage or sampled overlap error.
6. Preserve the current residual as a useful internal-consistency diagnostic, but do not let it certify external truth.

---

## (f) Float32, NaN, negative, saturation, and dtype paths

**Verdict: NOT-RULED-OUT overall. Ordinary finite same-depth unsigned overflow is RULED OUT; other reachable corruption paths are demonstrated.**

### What is ruled out

For finite uint8/uint16 input and an `estimate_flatfield()`-generated field:

- field is bounded to `[0.2,5.0]`;
- maximum corrected uint8 is `255/0.2 = 1275`;
- maximum corrected uint16 is `65535/0.2 = 327675`;
- even 60 fully coincident maximum uint16 tiles accumulate below 20 million;
- positive finite unsigned pixels divided by a positive finite field cannot become negative or NaN.

These values are far below float32 overflow. A valid uint16 single-tile result matched a float64 reference and clipped rather than wrapping.

### Demonstrated remaining paths

#### 1. Unreported output saturation

- `60000 / 0.67 = 89552` saved as uint16 became `65535`.
- `65535 / 0.2 = 327675` became `65535`.
- uint8 `200/0.67 = 298.5` and `255/0.2 = 1275` both became `255`.

Clipping is preferable to wraparound, but quantitative information above the native ceiling is still lost and no saturation count is reported.

#### 2. Mixed channel bit depth is quantized using the reference plane

`stitch_well()` captures `src_dtype` once from the reference image at `stitcher.py:661` and passes it to every plane at `691-692`.

An end-to-end in-memory well with a uint8 CH4 reference and a uint16 signal plane of approximately `40000-43000` returned the entire signal plane as uint8 value `255`, with flat-field both off and on. A direct four-value proxy `[100,300,1000,50000]` correctly stayed uint16 when its own dtype was used, but became `[100,255,255,255]` under the uint8 reference dtype: 75% clipped.

#### 3. Mean-Z violates “quantizes once”

`_z_reduce(..., "mean")` casts the mean back to the stack's integer dtype before flat-fielding.

- uint16 Z values `1000` and `1001` became `1000` rather than `1000.5`;
- division by `0.2` then saved `5000`;
- true quantize-once behavior is `rint(1000.5/0.2) = 5002`.

Measured error: `-2 DN` in this two-slice example.

#### 4. Non-finite/invalid values poison rather than mask

`apply_flatfield()` validates neither field shape, positivity, nor finiteness. With image value 100 and field entries `[0,-1,NaN,Inf]`, it produced `[Inf,-100,NaN,0]`. When blended with a valid overlapping value 50 and saved to uint16, the result was `[65535,0,0,25]`.

One NaN in one of four float tiles made `estimate_flatfield()` return an all-ones field after the smoothed profile mean became non-finite; the source NaN then survived correction and poisoned its blend pixel. Integer output cast it to 0 with a warning; float output remained NaN. `_read_plane()` accepts non-uint images by converting them to float32, so this is reachable for float TIFFs even though ordinary unsigned tiles are safe.

#### 5. General float input and accumulation precision

- Four coincident arbitrary float tiles `[+3e38,+3e38,-3e38,-3e38]` have mathematical mean zero, but sequential float32 accumulation overflowed and quantized to 65535. This is not reachable from a valid corrected uint16 field, but the public float path is unbounded.
- For 21 coincident random uint16 tiles with random valid fields, float32 versus float64 accumulation changed 28 of 20,480 pixels (`0.1367%`) by at most `1 DN`.
- Passing `out_dtype="uint16"` as a string misses the `dtype == np.uint16` branch and returns float32; `np.dtype("uint16")` works. Internal calls currently pass a dtype object, but the function API is fragile.

### Fix

1. Track and use each plane's own source dtype; reject or explicitly convert inconsistent tile dtypes within a plane.
2. Keep mean-Z in float32/float64 and quantize only after projection, correction, and blending.
3. Validate field shape, finiteness, and strict positivity before division.
4. Exclude non-finite samples from both accumulator and weight; report invalid input/output pixel counts.
5. Count and report clipped pixels, and permit float32 or wider integer output for quantitative corrected data.
6. Normalize `out_dtype` with `np.dtype(out_dtype)`; consider float64 accumulation for exact uint16 final-DN behavior.

---

## Recommended repair order

1. **Stop biological damage first:** disable specimen-derived flat-field by default for dense data; decouple geometry from the output correction option.
2. **Make plate inheritance fail closed:** do not seed from one well; require a tight multi-donor mode or trusted calibration and preserve provenance.
3. **Move ambiguity handling after global consensus:** retain candidate evidence and test uniqueness at the aggregate level.
4. **Re-label and augment QC:** treat residuals as internal consistency and add independent geometry/expected-step validation plus usable-edge coverage.
5. **Repair numeric contracts:** per-plane dtype, floating mean-Z, finite masks, saturation reporting, normalized output dtype.

## Minimum regression suite before acceptance

1. Native-size 80%, 90%, and 100% dense/stationary specimens must preserve object ordering, contrast, and background within declared tolerances; mean intensity alone is insufficient.
2. Toggling flat-field must not change the measured lattice. If both raw and corrected geometry are retained for diagnostics, disagreement must block or warn explicitly.
3. Connected aperiodic diffuse-cell wells must retain true candidates through global consensus; test the measured gate-loss rates above.
4. One wrong donor and an `n=2` disagreement must not produce a usable prior or a midpoint lattice.
5. A common camera-coordinate artifact must fail independent validation even if internal residual is zero.
6. Mixed uint8/uint16 channels, uint16 saturation, mean-Z halves, NaN/Inf float TIFFs, and string/dtype output requests must have explicit tested behavior.

## Bottom line

All four upgrades have a concrete failure mechanism. The most serious combination is not a crash: it is a quantitatively altered image or geometrically wrong mosaic that still looks internally confident. The current warnings reveal some fallbacks, but they do not expose why evidence was rejected, whether the plate prior is trustworthy, or whether a zero residual merely confirms a shared systematic alias.
