# Codex-Critic Stage-5b CROSS PRE-MORTEM — QA-corrected final

**Target:** current `stitcher.py` in this workspace  
**Date:** 2026-08-06  
**Method:** executable, ground-truthed NumPy/PIL experiments against the implemented functions  
**Safety:** all experiments were in memory with `PYTHONDONTWRITEBYTECODE=1`; no existing file was edited, overwritten, truncated, moved, or deleted; this is a new uniquely named file

## Executive verdict

| Attack | Verdict | Strongest measured evidence |
|---|---|---|
| (a) Dense/stationary specimen enters the estimated flat-field | **NOT-RULED-OUT** | At 90% coverage, true bright/dim ratio `1.379` became `0.936`, inverting biological intensity ordering |
| (b) Flat-fielding changes geometry or reported residuals | **NOT-RULED-OUT** | Correct X `(-1,113)` became abut `(0,160)`, `47.01 px` wrong |
| (c) The 12% ambiguity test needlessly drops a real non-periodic well | **NOT-RULED-OUT** | In a connected 21-tile diffuse-specimen regime, `13/19` gate-off-correct wells fell back; a native-size case incurred a `538 px` X-step error |
| (d) One bad donor poisons plate inheritance | **NOT-RULED-OUT** | An `n=1` false donor made an inherited 4×4 well `120 px` wrong at the far corner; one good donor then produced a still-wrong midpoint lattice |
| (e) Same-estimator QC is optimistically self-confirming | **NOT-RULED-OUT as accuracy QC** | Reported residual median/p95/max `0/0/0` while independent truth was `28/32/32 px` |
| (f) Float32/NaN/negative/dtype paths corrupt output | **NOT-RULED-OUT overall** | A uint8 reference forced an end-to-end uint16 plane entirely to `255`; mean-Z also quantized before `blend()` |

Two narrower allegations were **RULED OUT**:

1. Finite uint8/uint16 pixels corrected by an estimator-produced field cannot overflow float32 or spontaneously become negative/NaN.
2. Reusing `_pair_candidates` does not mechanically force every residual to zero.

Those narrow results do not rescue the upgrades. Reachable counterexamples remain for every requested attack.

## Protocol

- I called the current `estimate_flatfield`, `apply_flatfield`, `_pair_candidates`, `estimate_steps`, `plate_geometry`, `_subpixel`, `_edge_report`, `blend`, and full `stitch_well` paths.
- Full-size tests used tile `1374×1832` and truth `step_x=(-3,1294)`, `step_y=(970,2)` where stated. Scaled tests preserved approximately 29% overlap and used either the 21-tile 5×5-minus-corners topology or an explicitly labelled 3×3 topology.
- “Correct well” in the ambiguity sweeps means both directions were measured and within 6 pixels of truth.
- Geometry was scored against known origins, not the estimator's own output.
- I/O-free end-to-end tests temporarily mapped `_read_plane` to in-memory arrays and restored it in `finally`; source code was not patched.
- Some controls temporarily set `AMBIGUITY_MARGIN=0` at runtime, then restored `0.12`.
- Frequencies below describe the constructed synthetic regimes, not prevalence on unknown real plates.

## Cross-failure chain

```text
camera-coordinate artifact
  -> confidently wrong but physically plausible measured donor
  -> immediately admitted n=1 plate prior
  -> diffuse/non-periodic wells lose correct votes to the pair ambiguity gate
  -> those wells inherit the poisoned prior
  -> same-data residual reports agreement with the artifact, not truth
```

The overlap gate does not stop the demonstrated donor: false `dx=192` in a 320-pixel tile and false `dy=144` in a 240-pixel tile both imply 40% overlap, inside `(0.03,0.75)`.

---

## (a) Dense or nearly stationary biology is learned as shading

**Verdict: NOT-RULED-OUT — directly demonstrated.**

### Mechanism

For detector coordinate `p`, let `I_i(p)=S(p)B_i(p)`, with illumination `S` and biology `B_i`. Because `S` is positive,

```text
q25_i[I_i(p)] = S(p) q25_i[B_i(p)].
```

At `stitcher.py:526-534`, occupancy above 75% means the 25th percentile can still be specimen. When the specimen moves only a few pixels, `q25(B)` remains spatially structured in camera coordinates. The 12%-scale box smoothing removes fine detail but preserves broad tissue/expression gradients. Normalizing and clipping cannot distinguish these from illumination, so division flattens biology.

### Numeric evidence

Native-size run:

- 21 tiles, `1374×1832`;
- `make_well.vignette(strength=0.55)`;
- seed `20260806`;
- non-periodic broad Gaussian biology plus asymmetric X/Y trends, background 22;
- specimen movement only ±8 pixels, gain SD 1.2%, noise SD 0.6;
- default `lo_pct=25`, `smooth_frac=0.12`.

| Coverage | Field RMSE vs true shade | Object bias | Bright/dim truth → corrected | Object/background truth → corrected | Image NRMSE |
|---:|---:|---:|---:|---:|---:|
| 90% | `0.2371` | `-3.70%` | `1.379 → 0.936` | `7.18 → 2.48` | `20.62%` |
| 100% | `0.1217` | `+0.31%` | `1.402 → 1.042` | n/a | `11.42%` |

At 90% coverage, bright biology became dimmer than nominally dim biology. Within-object truth/corrected correlation was `-0.135`, slope `-0.117`, and background rose from `22.27` to `62.08`. At 100% coverage the mean looked nearly perfect while real contrast was flattened; mean-only validation misses the damage.

A `300×400` 80%-coverage control gave field RMSE `0.3323`, object bias `-8.30%`, bright/dim `1.375 → 0.892`, object/background `7.26 → 2.30`, NRMSE `28.46%`, and object correlation `-0.191`.

An independent stationary-map run found that zero to ±24-pixel motion changed a true bright/dim ratio near 2.0 to `1.000-1.016`, retaining at most 2.6% of contrast. Correlation of `estimated_field / true_shade` with biology was `0.965-0.996`.

### Fix

1. Prefer an instrument/control-slide flat acquired without specimen.
2. Otherwise estimate illumination from overlap correspondences, where the same world pixel is observed at different detector coordinates.
3. Require demonstrated per-coordinate background support; disable and flag correction when support is inadequate.
4. Do not enable this percentile method by default for confluent or nearly stationary specimens.
5. Report field range/clipping, support fraction, camera-coordinate stability, and field/specimen correlation; validate held-out biological contrast, not only mean intensity.

---

## (b) Flat-fielding changes geometry and geometric QC

**Verdict: NOT-RULED-OUT — directly demonstrated; ground truth selected raw geometry in the failure case.**

### Mechanism

`stitcher.py:659-670` corrects `ref_imgs` before both `estimate_steps()` and `_edge_report()`. Division changes local noise, high-pass content, candidate rankings, NCC, and the hard `MIN_PAIR_SCORE=0.15` transition. An output-intensity option can therefore choose a different lattice.

### Numeric evidence

Ordinary full-size `make_well`, seed 0, was a control:

- truth X/Y `(-3,1294)/(970,2)`;
- raw recovered exact steps, peaks `0.60785/0.92226`;
- corrected recovered exact steps, peaks `0.61336/0.91994`.

Failure is not universal. Yet even when steps stayed exact in a scaled ordinary run, residual median/p95 changed from `0.00/0.10 px` raw to `0.00/0.00 px` corrected, and edge NCC from `0.919` to `0.993`. QC is preprocessing-dependent.

Ground-truthed failure:

- connected 21-tile grid, tile `120×160`;
- truth X/Y `(-1,113)/(85,1)`;
- fully dense, non-periodic scene formed from two independent smoothed Gaussian-random fields;
- vignette strength 0.65, noise SD 6, seed 3;
- estimated field was accurate versus true shade: RMSE `0.01215`, range `0.5796-1.2861`.

| Processing | X result | X consensus | Source | Error |
|---|---:|---:|---|---:|
| Raw | `(-1,113)` | `0.150589` | measured | `0 px` |
| Estimated flat-field | `(0,160)` | `0.144064` | abut | `47.01 px` |

Raw Y was exact with peak `0.169039`. All 16 X pairs survived and ambiguity drops were zero in both variants. Flat-fielding alone moved consensus below 0.15.

| QC pixels/lattice | Median | p95 | NCC median |
|---|---:|---:|---:|
| Raw at true/raw lattice | `0.64` | `1.00` | `0.508` |
| Corrected at true lattice | `0.57` | `0.64` | `0.633` |
| Corrected at returned abut lattice | `23.62` | `47.41` | `0.633` |

The corrected full-resolution QC actually favored the true lattice, while high-pass consensus rejected it. A plate prior can replace abut, but cannot remove this setting dependence.

### Fix

1. Fit geometry on an immutable raw/high-pass reference; apply specimen-derived correction only after origins are fixed.
2. Separate raw geometric QC from optional corrected-image photometric QC.
3. If both candidates are computed, expose disagreement and never let an output correction checkbox silently select the lattice.
4. Calibrate confidence for the exact preprocessing/noise model and use uncertainty/support rather than one hard score.

---

## (c) `AMBIGUITY_MARGIN=0.12` drops correct non-periodic evidence

**Verdict: NOT-RULED-OUT — directly demonstrated in connected 21-tile wells and at native size.**

### Mechanism

At `stitcher.py:366-370`, `_pair_candidates()` returns `[]` before cross-edge consensus at `374-401`:

```text
current: each edge -> close rival -> delete all candidates -> global consensus
needed:  each edge -> retain weighted candidates -> global consensus -> aggregate uniqueness
```

Diffuse cells and tissue boundaries create broad NCC lobes. Similar but aperiodically located objects create edge-specific rivals. The rivals vary, whereas the stage step is common. The fixed `_TOL=6` can also call samples from one broad morphology-dependent lobe “distinct.” Correct evidence is deleted before consensus can exploit its agreement. Physical plausibility is applied later at `431-439`, so a rival can also veto before that later filter.

### Connected 21-tile experiment

- 5×5-minus-corners topology, 16 X plus 16 Y edges;
- tile `160×220` (H×W);
- truth X/Y `(-1,155)/(113,0)`, 29.5%/29.4% overlap;
- 1,100 randomly positioned, lognormally weighted impulses Gaussian-smoothed into diffuse cell/tissue bodies;
- one unique off-centre elliptical envelope and independent per-tile noise;
- no grating, repeated tile, or periodic placement.

| Regime | Correct at margin 0 | Lost at 0.12 | Fell to abut |
|---|---:|---:|---:|
| Blur radius 30, contrast 15, noise SD 1 | `30/30` | `5/30 (16.7%)` | `4/30 (13.3%)` |
| Blur radius 34, contrast 15, noise SD 1 | `19/30` | `13/19 (68.4%)` | `13/19 (68.4%)` |

The radius-34 denominator excludes 11 wells not correctly measurable even with the gate disabled.

Representative radius-34 seed 3:

- truth X/Y `(-1,155)/(113,0)`;
- margin 0 returned X `(0,154)`, peak `0.167888`, and Y `(112,0)`, peak `0.173333`, both measured;
- margin 0.12 rejected `9/16` X and `11/16` Y edges;
- all 20 rejected lists still held a truth-within-6 candidate; truth ranked first in `11/20`;
- surviving consensus fell to `0.124470/0.130529` and returned abut `(0,220)/(160,0)`;
- rejected best/rival gaps had medians only 5.43% X and 5.33% Y; failure began at margin 0.06 for Y and 0.08 for both axes.

### Broader scaled 3×3 sweep

This was a **9-tile, 12-edge-per-well** sweep, separate from the 21-tile experiment. Of 550 conditions/seeds, 406 wells were correct with the gate disabled. The 0.12 gate made `67/406 (16.5%)` lose correctness and `60/406 (14.8%)` fall back. It gated `2,912/6,600` edges; `2,896/2,912 (99.45%)` gated lists still contained a truth-within-6 candidate.

### Native BZ-X confirmation

- tile `1374×1832`, truth X/Y `(-3,1294)/(970,2)`;
- connected 3×3 grid;
- 20 randomly located, independently sized/oriented/intensified elliptical Gaussian colonies;
- noise SD 1, seed 9.

For X, margin 0 recovered `(0,1295)`, `3.16 px` from truth, peak `0.630856`. Margin 0.12 rejected `4/6` X edges. Three rejected winners were `3.16`, `3.16`, and `4.24 px` from truth and beat their rivals by only `1.87%`, `2.16%`, and `0.48%`. Surviving consensus was `0.074733`, so without a prior it returned abut `(0,1832)`, a `538 px` step error. With the true prior supplied, it inherited `(-3,1294)` with source `plate`, but the well remained locally unmeasured.

### Does the user notice?

Only generically:

- GUI warnings say “geometry inherited from the plate” or “geometry unmeasurable — tiles abutted.”
- CSV includes source, `low_confidence`, and `ambiguous_edges`.
- Nothing says the 12% gate discarded consensus-resolvable evidence; `ambiguous_edges` conflates all empty-candidate causes.
- Only the first 12 warnings appear, output images embed no QC marker, and surviving measured wells get no GUI warning for a high ambiguity fraction.

### Fix

1. Return candidates plus uncertainty/gap; do not hard-return `[]` for pair ambiguity.
2. Downweight ambiguous pairs, cluster across all edges, then compare aggregate winner and runner-up.
3. Make same-peak tolerance feature-scale-aware and apply physical plausibility before veto.
4. Report rejected/total edges and an explicit fallback reason such as `pair_ambiguity_gate`.
5. Regression-test aperiodic diffuse tissue so a strong common shift cannot be weakened by pair-level uniqueness.

---

## (d) One bad donor poisons online plate consensus

**Verdict: NOT-RULED-OUT — directly demonstrated through `stitch_well -> plate_geometry -> stitch_well(prior)`.**

### Mechanism

Admission at `stitcher.py:466-467` is peak-only (`>=0.15`). Current plate/abut fallback directions are indirectly excluded because their retained peaks remain below 0.15, but a confidently false `measured` donor has no calibration, vector-outlier, donor-count, edge-support, or explicit provenance check. Spread is computed but unenforced.

`main_stitch_v2.py:120-150` starts using the first admitted/confident measured donor direction immediately. A well using `plate` is written; only `abut` is deferred. With two disagreeing donors, componentwise median manufactures a midpoint that can belong to neither.

### Numeric evidence and exact construction

- seed 55; 4×4 grid; 16 tiles of `240×320`; 24 edges;
- truth X/Y `(0,224)/(168,0)`;
- `flatfield=False`, `subpixel=True`;
- global scene `N(0,30)`;
- base tile `clip(110 + 0.35*scene_crop + artifact + N(0,0.5), 0,255).astype(uint8)`;
- detector artifact `qx ~ N(0,25)`, shape `240×128`, copied into columns `0:128` and `192:320`;
- detector artifact `qy ~ N(0,25)`, shape `96×320`, copied into rows `0:96` and `144:240`.

These are duplicated non-periodic random patches, not a grating. The actual functions returned:

- false X `(0,192)`, peak `0.499512`;
- false Y `(144,0)`, peak `0.504324`;
- source measured, `low_confidence=False`;
- 24 edges, ambiguity 0, NCC median `0.506`;
- residual median/p95/max `0/0/0`.

`plate_geometry([bad])` accepted that prior with `n_x=n_y=1`, spread 0.

The inheritance target used identical periodic, hence unmeasurable, tiles:

```text
clip(110 + 35 sin(2*pi*x/16) + 35 sin(2*pi*y/16), 0,255).astype(uint8)
```

It inherited the false X/Y with peak 0, source `plate`, edges 0, residual `None`, and 24 ambiguous edges.

- per-step errors: X 32 px, Y 24 px;
- far `(3,3)` displacement: `120 px`;
- false shape `672×896`; true shape `744×992`.

Adding one clean donor yielded componentwise midpoint X `(0,208)` with spread 16 and Y `(156,0)` with spread 12. The next inherited target remained 16/12 pixels wrong per step and `60 px` wrong at the corner. Only after two good plus one bad donor did median return truth; earlier inherited outputs were not rebuilt.

### Does the user notice?

- The bad donor has no warning: measured, above threshold, ambiguity 0, residual p95 0.
- The target says only “geometry inherited from the plate”; the `elif` chain masks the separate low-confidence wording.
- Donor IDs/count/spread/disagreement are absent from the well record and CSV.
- Its 24 ambiguous edges appear only in CSV and residual is `None`, so residual thresholding cannot warn.

### Fix

1. Measure donors first; use a prior only after a stable plate model exists.
2. Require at least three agreeing measured donors or trusted calibration. Reject `n=2` disagreement rather than averaging.
3. Cluster two-dimensional step vectors and select a tight dominant mode; enforce spread/deviation and usable-edge support.
4. Carry donor IDs, count, spread, expected-step deviation, and consensus status into inherited QC/warnings.
5. Delay inherited output and rebuild it if consensus changes materially.

---

## (e) Residual is internal consistency, not placement accuracy

**Verdict: NOT-RULED-OUT as accuracy QC; the claim that reuse automatically makes residual zero is RULED OUT.**

### Mechanism

`estimate_steps()` fits from `_pair_candidates`. At `stitcher.py:718-746`, `_edge_report()` re-runs the same estimator on the same images, takes the first winner, optionally performs `_subpixel`, and compares it with the fitted lattice. Empty/ambiguous edges are omitted from median, p95, max, and NCC.

That legitimately measures agreement among accepted matches. It is not independent evidence that the matches identify the same world features. Fit/test reuse adds finite-sample and selection optimism; a shared alias makes fitting and QC wrong together.

### Numeric evidence

The tautology is false: on clean random data with true X/Y `(0,224)/(168,0)`, passing deliberately wrong `(0,192)/(144,0)` into `_edge_report()` produced median `28`, p95/max `32`, 24 edges, NCC 1.

Ordinary optimism was modest:

- 20 independent jittered 5×5 wells: mean fitted/independent-known median ratio `0.988`, range `0.831-1.203`, average lowering `0.112 px`;
- 12-well train/holdout split: training median `7.37 px`, holdout `7.78 px`, 5.5% higher on holdout; holdout exceeded train in 8/12.

But the exact camera-artifact run from (d), using `flatfield=False` and `subpixel=True`, was catastrophic:

| Metric | Report | Independent truth |
|---|---:|---:|
| Median | `0` | `28 px` |
| p95 | `0` | `32 px` |
| Max | `0` | `32 px` |
| Far-corner displacement | absent | `120 px` |

All 24 edges selected the same false camera pattern, ambiguity was 0, NCC median `0.506`, source measured, and the GUI emitted no warning.

### Fix

1. Rename these fields `internal_lattice_residual_*`; preserve them, but do not present them alone as accuracy proof.
2. Validate independently against stage metadata, trusted calibration/expected step, or a separate signal/frequency band.
3. Fit and validate on disjoint edges/features/channels. Holdout reduces ordinary optimism but cannot catch an artifact common to every edge.
4. Estimate/remove or whiten the camera-coordinate common template.
5. Report accepted/total and ambiguous fractions, weak-NCC count, expected-step deviation, and sampled seam visual QC.

---

## (f) Float32, NaN, negative, saturation, and dtype paths

**Verdict: NOT-RULED-OUT overall; finite same-depth unsigned float32 overflow is RULED OUT.**

### Ruled-out standard arithmetic path

For finite uint8/uint16 and a generated field clipped to `[0.2,5]`:

- max corrected uint8: `1275`;
- max corrected uint16: `327675`;
- even 60 coincident maximum uint16 tiles accumulate below 20 million;
- division by a positive finite field cannot create negative or NaN values.

These are far below float32 overflow. A valid uint16 result matched a float64 reference and clipped, never wrapped.

### Demonstrated corruption/loss paths

1. **Unreported saturation.** `60000/0.67=89552` and `65535/0.2=327675` both saved as `65535`; uint8 `200/0.67` and `255/0.2` both saved as `255`. Native-ceiling information is lost without a clipped-pixel count.

2. **Mixed channel dtype.** `stitch_well()` takes `src_dtype` once from the reference at `stitcher.py:661` and passes it to every plane at `691-692`. An end-to-end uint8-reference/uint16-signal well with signal values about `40000-43000` returned the entire signal plane as uint8 `255`, with flat-field off and on. A direct `[100,300,1000,50000]` proxy became `[100,255,255,255]`: 75% clipped.

3. **Mean-Z prequantization.** `_z_reduce(...,"mean")` casts to integer before correction. uint16 Z values `1000,1001` became `1000`; divide by 0.2 then saved `5000`, while true quantize-once gives `rint(1000.5/0.2)=5002`. Error: `-2 DN`.

4. **Invalid/non-finite poisoning.** With image value 100 and field `[0,-1,NaN,Inf]`, `apply_flatfield()` produced `[Inf,-100,NaN,0]`. Blending with valid overlapping value 50 into uint16 produced `[65535,0,0,25]`. One NaN in one of four float tiles caused the smoothed field estimate to fall back to all ones, after which the source NaN still poisoned output. `_read_plane()` permits float TIFF input, so this is reachable outside ordinary unsigned tiles.

5. **General float input and precision.** Four coincident arbitrary float tiles `[+3e38,+3e38,-3e38,-3e38]` mathematically average to zero but overflowed sequential float32 accumulation and quantized to 65535. This is unreachable from valid corrected uint16 but the public float path is unbounded. For 21 coincident random uint16/valid-field tiles, float32 versus float64 changed `28/20,480 (0.1367%)` pixels by at most 1 DN.

6. **Fragile `out_dtype`.** String `"uint16"` misses `dtype == np.uint16` and returns float32; `np.dtype("uint16")` works. Internal calls use dtype objects, but the function contract is unsafe.

### Fix

1. Track each plane's source dtype; reject or explicitly normalize inconsistent tile dtypes within a plane.
2. Preserve mean-Z as float and quantize only after projection, correction, and blend.
3. Validate field shape, finiteness, and strict positivity.
4. Mask non-finite samples out of both accumulator and weight; report invalid-pixel counts.
5. Count/report clipping and offer float32 or wider output for quantitative corrected data.
6. Normalize `out_dtype=np.dtype(out_dtype)`; use float64 accumulation if exact final uint16 DN matters.

---

## Repair priority

1. **Stop biological damage:** disable specimen-derived correction by default for dense data and decouple alignment from output correction.
2. **Make inheritance fail closed:** require trusted calibration or a tight multi-donor mode; preserve consensus provenance.
3. **Move ambiguity after global evidence aggregation:** retain candidates and compare aggregate clusters.
4. **Reframe QC:** keep internal residual, add independent geometry validation and usable-edge coverage.
5. **Repair numeric contracts:** per-plane dtype, floating mean-Z, finite masks, saturation reporting, normalized dtype.

## Minimum acceptance regressions

1. Native-size 80%, 90%, and 100% dense/stationary specimens preserve object ordering, contrast, and background within declared tolerances.
2. Flat-field on/off cannot change lattice; any diagnostic disagreement blocks or warns explicitly.
3. Connected aperiodic diffuse-cell wells retain correct candidates through aggregate consensus.
4. One false donor and an `n=2` disagreement cannot create a usable prior or midpoint lattice.
5. A camera-coordinate artifact fails independent validation even when internal residual is zero.
6. Mixed uint8/uint16, uint16 saturation, mean-Z halves, NaN/Inf float TIFFs, and string/dtype output requests have explicit tested behavior.

## Bottom line

All four upgrades have concrete failure mechanisms. The most dangerous result is not a crash; it is a biologically altered image or geometrically wrong mosaic that remains internally confident. Current warnings reveal some fallbacks but do not identify ambiguity-gate rejection, establish that a plate prior is trustworthy, or show that a zero residual can merely confirm a shared systematic alias.
