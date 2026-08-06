# Implementation plan: quantitative and failure-safe Keyence stitching upgrades

Created: 2026-08-06 00:16:52 JST  
Target: `stitcher.py`  
Status: implementation plan only; no source code is changed by this document.

## Inputs and decisions

This plan is based on the current `stitcher.py`, the Claude panel report, the Codex red-team report, both versions of the field-standard report (the final differs only in citation corrections), and the supplied measurements from real Keyence BZ-X data.

The measured data change one recommendation from the general reviews: retain the two-vector rigid lattice as the final geometric model for this upgrade. High-confidence edge residuals have median 1 px, maximum 3 px, and no growth with distance from the grid origin. There is therefore no evidence here for a per-tile pose solve. Preserve every edge measurement and expose the residuals so this assumption remains testable; revisit a per-tile robust solve only if residual growth, row/column structure, or cycle errors appear.

The four decisions are:

1. Use a BaSiC-style rank-2 low-rank plus sparse flat/dark-field estimator for each saved `Plane.key`, not a pixelwise median/percentile. Twenty-one to fifty-nine independent XY fields are enough for the rank-2 model and a 10/11-image split validation at the smallest well. Intensity sorting, reweighted L1 residuals, and spatial smoothness are materially safer than a median field when dense tissue occupies most frames.
2. Separate geometry measurement from geometry resolution. Measure all wells first, form an equal-well robust plate consensus, then let an unmeasurable axis inherit the complete plate vector. Never turn “unmeasurable” into abutment inside `estimate_steps`.
3. Use an inclusive, uniform hard overlap support of 5–60% before candidate scoring/top-K and again in consensus. Do not narrow the universal gate around 29%. Add explicit periodic ambiguity detection so an alias that remains inside 5–60% cannot be called a confident local measurement.
4. Refine accepted edge translations with the Guizar-Sicairos local upsampled DFT at a factor of 10, retain floating-point lattice vectors/origins, render the fractional origins, and write a versioned JSON sidecar containing one record for every expected neighbor edge.

Primary method references: [BaSiC](https://doi.org/10.1038/ncomms14836) and [Guizar-Sicairos upsampled-DFT registration](https://doi.org/10.1364/OL.33.000156).

## Resulting data flow

The batch path becomes a memory-bounded two-pass workflow:

1. `discover_wells()` remains unchanged.
2. `prepare_plate(...)` inventories compatible channel/shape/dtype cohorts and fits one shading model per `Plane.key` from raw, unprojected sample frames.
3. For every well, load the reference plane, correct every raw Z slice, perform the requested Z reduction, measure every expected +X/+Y edge, apply the overlap gate and ambiguity test, and refine accepted edges to 0.1-pixel sampling. Discard reference images after recording the well estimate.
4. Aggregate locally valid well axes into plate vectors. Resolve each well axis independently as `local`, `plate_consensus`, or `nominal_overlap`.
5. Stitch each plane: `_read_plane` -> flat/dark correction in `float32` -> `_z_reduce` -> fractional-origin feather fusion -> one final round/clip/cast.
6. Save the OME-TIFF and `<OME filename>.stitch-qc.json` together. The caller must retain `result["__geometry__"]` long enough to pass it to `save_ome_tiff(..., qc=...)`.

This intentionally rereads reference frames in pass 2 rather than caching all wells in memory.

## Internal types and compatibility boundary

Add the following internal dataclasses; exact names may change, but their information must not be collapsed:

- `OverlapPrior(min_fraction=0.05, max_fraction=0.60, nominal_x=0.29, nominal_y=0.29, source="keyence_protocol")`.
- `ShadingModel`: plane/cohort identity, native `flatfield`, native `darkfield`, fit settings, sample IDs, convergence/stability metrics, model ID, and status.
- `EdgeMeasurement`: one expected adjacency, all candidate/refinement/QC fields listed later.
- `AxisEstimate`: nullable local step, scores, expected/supporting edge counts, support fraction, ambiguity, subpixel status, and failure reason.
- `WellGeometryEstimate`: local X/Y estimates and all edges; it contains no fallback.
- `ResolvedWellGeometry`: applied X/Y vectors plus source and plate/nominal provenance.
- `PlateCalibration`: shading models, all local estimates, plate vectors/scatter/donors, and resolved well geometries.

Preserve the current public surface:

- `estimate_steps(images)` still returns exactly `(step_x, step_y, px, py)`. It becomes a wrapper over `_estimate_steps_detailed`. Components become floats, which is the intended precision improvement. A standalone unmeasurable direction uses the configured nominal-overlap fallback, never abutment.
- `stitch_well(wt, planes=None, z_mode="max", progress=None, cancel=None, ...)` keeps its existing positional parameters and plane-result dictionary. Add new options only after `*`, for example `plate_calibration=None`, `shading_models=None`, `overlap_prior=None`, and `shading_mode="basic"`.
- A standalone `stitch_well` self-fits shading models from that well and resolves failed geometry from the explicit nominal prior. Plate callers use the new `prepare_plate(...)`/`stitch_plate(...)` path.
- Preserve all existing `__geometry__` fields. Extend that dictionary; do not add another sentinel key because the current caller assumes every non-`__geometry__` entry is plane data.
- `save_ome_tiff(path, planes_data, pixel_um=0.0)` remains valid. Add only keyword-only `qc=None`; the normal application path supplies it.
- `tile_origins`, `mosaic_size`, and `blend` continue accepting the current arguments, while also accepting float origins.

The current GUI worker needs a small caller change even though the algorithms live in `stitcher.py`: prepare the plate before its per-well loop, pass the resolved calibration to `stitch_well`, do not discard `__geometry__` before saving, and call `save_ome_tiff(..., qc=geo)`.

## 1. Flat-field and shading correction

### Concrete estimator

Use the BaSiC spatial model, separately for every saved `Plane.key` (channel plus page), not merely for the file-level `CH` tag:

\[
Y_i(p) = B_i S(p) + D(p) + R_i(p).
\]

`S` is a positive multiplicative flat field, `D` is an additive dark field, `B_i` is a frame-level scalar used only to identify the model, and `R_i` is the biological/foreground residual. The shared background matrix has rank at most two (`S B^T + D 1^T`). Fit it with intensity sorting, a reweighted L1 residual, and Fourier/spatial smoothness on `S` and `D` using the BaSiC LADMAP solver.

Do not enable BaSiC's time-lapse/baseline correction and do not apply `B_i` to the output. With dense fluorescence, per-frame baseline normalization can remove real well-to-well or tile-to-tile biology. Only the spatial `S` and `D` are applied.

Use a thin adapter around a pinned, tested BaSiCPy release rather than a home-grown median fallback. The package is not currently installed in this workspace, so adding and locking that runtime dependency is an explicit implementation prerequisite. The following settings must be covered by an adapter test (the names match the BaSiCPy 1.1 API):

```text
get_darkfield=True
sort_intensity=True
fitting_mode="ladmap"
working_size=[192, 256]
smoothness_flatfield=1.0
smoothness_darkfield=1.0
sparse_cost_darkfield=0.01
autosegment=False
```

If a newer BaSiCPy release is selected, pin it and prove parameter/output parity; do not silently substitute a simple median if import or fit fails.

### Sampling and model scope

- Cohort key: `(Plane.key, tile_shape, source dtype, pixel calibration/objective when present)`. Never pool unlike optical configurations.
- Take one native, raw middle-Z frame from every distinct XY field. Do not feed a maximum, mean, or stitched image to the estimator.
- In a plate run, stratify samples by well and XY position, round-robin across wells, and cap a model at 128 frames for bounded memory. In a standalone well, use all 21–59 fields.
- Skip unreadable frames and frames with more than 1% saturated pixels; record every skip.
- Convert to `float32` and scale by the source dtype maximum. Fit at 192x256, then bilinearly upsample `S` and `D` to 1374x1832 (or the discovered native shape).
- Normalize `mean(S)=1`; adjust the fitted scalar factor consistently. Store native-resolution `S` and `D` as `float32`.

This choice is right for 21–59 fields because the model has only two shared smooth spatial components, while even the minimum well permits deterministic checkerboard halves of 10 and 11 independent XY fields. Dense specimen occupancy is not itself disqualifying: the tissue changes with XY position while detector-fixed shading remains at the same sensor coordinates. The known failure is biology locked to the same detector coordinates in every field, not simply “a lot of tissue.”

### Application order and numeric behavior

Immediately after `_read_plane()` returns each raw Z slice, apply:

\[
C_z(p) = M\,\frac{I_z(p)/M-D(p)}{S(p)},
\]

where `M` is 255 or 65535. Keep `C_z` as `float32`, including negative and above-range values, through `_z_reduce` and `blend`. Do not clamp or cast per slice: early clipping would change a maximum/mean projection and reintroduce bias. At the end of fusion only, use `np.rint`, clip to the source dtype range, and cast once. Record low- and high-clipped fractions.

The same model is applied to every raw Z slice of that plane. Corrected reference slices feed alignment as well as saved pixels. This fixes the current asymmetry where only `_prep_score` is protected from vignetting.

### Integration points

- Add `_fit_shading_model(samples, cohort)` and `_apply_shading(img, model)` next to `_read_plane`.
- Refactor the nested `stitch_well.load()` so `_apply_shading` is called before appending a slice to `stack`, before ragged-stack cropping and `_z_reduce`.
- Fit/pass models before building `ref_imgs`; `_pair_candidates` then sees corrected projected reference tiles.
- Let `blend(..., output_dtype=source_dtype)` accumulate corrected floats and perform the sole final quantization.
- Put per-plane shading provenance under `result["__geometry__"]["shading"]`.

### Failure mode and response

Validate each model before it can touch saved pixels:

- at least 20 distinct usable XY fields, so both checkerboard halves have at least 10;
- solver converged and all `S`, `D` values are finite;
- `p01(S) > 0.10` and `p99(S) < 4.0`;
- checkerboard half-fit stability: `median(abs(S_A/S_B - 1)) <= 0.05` and `p95 <= 0.10`;
- half-fit dark-field RMSE `<= 5/255` in normalized units (scale equivalently for uint16).

Repeated detector-locked structures, too few usable frames, saturation, or insufficient diversity can make `S` and `D` non-identifiable. On validation failure, first try a compatible plate-pooled model. If no validated model exists, default quantitative mode raises `ShadingEstimationError` before any OME-TIFF is saved. An explicit `shading_mode="off"` preserves the legacy uncorrected path for a deliberate diagnostic/export, but sets `shading.status="disabled"`, `low_confidence=true`, and a warning; it must never be labelled corrected.

### Proof tests and pass criteria

1. **Measured uniform-field severity, N=21 and N=59.** Generate/use a true value of 120 whose raw field spans 47–152 and has a 29% overlap with an 11% raw overlap-to-centre deficit. Reopen the saved OME-TIFF. Across valid pixels require RMSE `<=2 DN`, absolute centre-versus-edge median difference `<=2 DN`, and absolute overlap-versus-tile-centre median difference `<=2 DN`.
2. **Dense specimen.** Parameterize 21 and 59 tiles with 80–100% foreground, different full-field textures at every XY position, known smooth `S` spanning `47/120` to `152/120`, and known `D`. On held-out nonsaturated frames require flat-field normalized RMSE `<=3%`, dark-field RMSE `<=2 DN`, corrected-pixel RMSE `<=3 DN`, and object-mean dependence on detector radius `<=2%`.
3. **Identity calibration.** With `S=1,D=0`, uint8 and uint16 saved output differs from the legacy arithmetic by at most 1 DN, dtype is unchanged, and a constant field is bit-exact after the new round-to-nearest cast.
4. **Ordering sentinel.** Instrument `_z_reduce`: every received slice must already be `float32` and equal `(I/M-D)/S*M` within `1e-5` before projection. Reopened output must be within 1 DN of the analytically projected truth.
5. **Fit failure.** A too-small or detector-locked sample set must fail the half-split validation, raise/flag the documented error, and produce no unflagged OME-TIFF.

## 2. Plate-level geometry fallback

### Separate measurement from resolution

The present fallback in `estimate_steps` destroys the crucial distinction between “measured step equal to a tile width” and “no measurement.” Remove those two abutting assignments from the detailed path.

Add `_estimate_steps_detailed(images, prior) -> WellGeometryEstimate`. For every axis it returns either a measured float vector or `step=None` plus a failure reason. `estimate_steps` remains a four-value compatibility wrapper and performs nominal resolution only at its outer boundary.

Define expected edges from `wt.positions`, before checking whether images loaded. For each +X/+Y axis:

```text
support_fraction = edges supporting winning mode / all expected adjacency edges
```

The denominator includes missing, unreadable, flat, and empty-candidate edges. This prevents the current behavior in which one surviving edge can masquerade as well-level confidence.

A well axis may donate to the plate only if it:

- is a genuinely local measurement, never an inherited/default value;
- has at least 3 supporting edges;
- has `support_fraction >= 0.50`;
- has consensus NCC at least the existing `MIN_PAIR_SCORE`;
- is inside the overlap prior;
- is not periodic/runner-up ambiguous; and
- has enough successful subpixel refinements to produce a float vector.

Sparse wells with informative area at or below 40% should therefore fail locally and become recipients, not donors.

### Plate aggregation

Resolve X and Y independently. A well can retain local X and inherit Y. Aggregate only wells in the same plate and calibration/tile-shape cohort.

For each axis:

1. Give every donor well equal weight; a 59-tile well must not dominate a 21-tile well.
2. Take the component-wise median of the donor two-vectors.
3. For each vector component, compute `sigma_robust = 1.4826 * MAD`.
4. Reject a donor vector if either component differs from the provisional median by more than `max(3 px, 4*sigma_robust)` for that component.
5. Recompute the component-wise median from inliers. Store component MAD/scatter and every rejected donor/reason.

The 3 px floor admits the measured high-confidence edge range; the 4-MAD rule rejects a gross periodic mode. One or two inlier donors may still rescue other wells because this is far safer than abutment, but report `single_donor` or `limited_support`. Three or more inlier donors report `robust_supported`.

### Resolution hierarchy

For each well axis, apply this order:

1. Its own valid local two-vector (`geometry_source_axis="local"`).
2. The plate two-vector, including its cross-axis shear (`"plate_consensus"`).
3. A trusted explicit nominal overlap (`"nominal_overlap"`). For this Keyence protocol, 29% gives:

```text
step_x = (0.0, 1832 * 0.71) = (0.0, 1300.72)
step_y = (1374 * 0.71, 0.0) = (975.54, 0.0)
```

Unknown nominal transverse terms are zero. If an entire plate is unmeasurable and neither nominal acquisition geometry nor trustworthy stage positions are available, raise `GeometryUnmeasurableError` before `tile_origins`, `blend`, or saving. Never invent `(0, tile_width)`/`(tile_height, 0)`.

Keep legacy `low_confidence=true` whenever either applied axis is inherited or nominal, even when the plate consensus is strong. A sparse well must remain visibly identified as not locally validated.

### Integration points

- Add `measure_well_geometry(...)`, `resolve_plate_geometry(...)`, and a `prepare_plate(...)` coordinator around `estimate_steps`/`stitch_well`.
- `prepare_plate` performs pass 1 and stores estimates, not mosaics.
- `stitch_well(..., plate_calibration=...)` uses pre-resolved vectors instead of re-resolving them.
- `tile_origins` and all channels use exactly the same resolved vectors.
- Preserve `peak_x/peak_y` as the local well scores; do not replace them with donor scores.

Extend `__geometry__` with:

```text
local_step_x, local_step_y
geometry_source_x, geometry_source_y
local_failure_x, local_failure_y
geometry_validated_locally_x, geometry_validated_locally_y
plate_step_x, plate_step_y
plate_donor_wells_x, plate_donor_wells_y
plate_donor_count_x, plate_donor_count_y
plate_rejected_wells_x, plate_rejected_wells_y
plate_scatter_x, plate_scatter_y
plate_status_x, plate_status_y
nominal_overlap_x, nominal_overlap_y
fallback_used
plate_unmeasurable_x, plate_unmeasurable_y
```

### Failure mode and response

Plate inheritance assumes wells share the same acquisition geometry. Mixing objectives, pixel calibration, tile shape, or separately acquired plates would create a wrong but precise consensus, so cohort checks are mandatory. A one-donor plate is useful but not robust; it is always low-confidence. Whole-plate failure uses the declared 29% nominal only for this known protocol and is explicitly `nominal_unvalidated`; it is never reported as measured.

### Proof tests and pass criteria

1. **Eight-well rescue.** Truth `step_x=(-3.2,1300.72)`, `step_y=(975.54,2.1)`; seven donors have the measured 0.5/0.8 px scatter and one well is wholly uninformative. Require every plate-vector component within 1.0 px of truth. The failed well must exactly receive the stored plate vectors; overlap X is `531.28 +/-1 px`, overlap Y is `398.46 +/-1 px`, sources are `plate_consensus`, and `fallback_used`/`low_confidence` are true.
2. **Poisoned donor.** Add one 120 px outlier. It must be listed as rejected and the final plate vector must remain within 1.0 px of truth.
3. **Partial-axis rescue.** A well with valid local X and failed Y must report `source_x=local`, `source_y=plate_consensus` and preserve its local X exactly.
4. **Whole-plate failure.** With nominal 0.29, the two vectors above must match within `1e-6`, every well must be `nominal_unvalidated`, and no overlap may be zero. With nominal/stage geometry absent, the documented exception must occur before any OME/sidecar write.
5. **API regression.** Existing positional `stitch_well` calls and four-value unpacking of `estimate_steps` still work. No failure path ever resolves to `(0,tw)` or `(th,0)`.

## 3. Physical overlap prior and periodic ambiguity

### Prior definition

The universal physical prior is a truncated uniform support, inclusive at both endpoints:

\[
0.05 \le f_{overlap} \le 0.60.
\]

For an X neighbor, `f_overlap = 1 - dx/tile_width`; for a Y neighbor, `1 - dy/tile_height`. Thus the main-axis step must satisfy:

```text
0.40 * axis_length <= main_step <= 0.95 * axis_length
```

For production tiles this is X `732.8..1740.4` and Y `549.6..1305.3` px. For the stripe fixture with width 200, valid `dx` is `80..190`: true 142 remains valid and the observed wrong 22 is physically impossible because it implies 89% overlap.

Do not hard-bound the transverse component; that would risk rejecting valid camera angle/shear. Keep the existing actual-overlap-area/patch-size checks.

`nominal=0.29` is acquisition provenance and a resolution fallback, not a universal hard gate. A different acquisition may pass any nominal inside 5–60% without changing the supported interval.

### Entry into `_pair_candidates` and `_consensus`

1. Add `prior=DEFAULT_OVERLAP_PRIOR` to `_pair_candidates`.
2. Generate aliases as now, then apply the main-axis prior before `_score_shift`, sorting, and `[:top]`. This ordering matters: impossible aliases must not consume all six candidate slots and push the true shift out.
3. Carry `implied_main_overlap_fraction` and rejection counts into edge QC.
4. Pass axis, tile shape, and prior into `_consensus`; reapply the gate to every anchor and voter as defense against externally constructed or legacy candidate lists.
5. Include empty/unmeasurable edges in the well support denominator.

Do not let a nominal expected overlap turn an intrinsically ambiguous periodic edge into a “local measurement.” Cluster all physically allowed candidate modes using the existing 6 px basin tolerance. If a distinct runner-up more than 6 px away has both:

```text
runner_up_support >= 0.80 * best_support
runner_up_mean_ncc >= best_mean_ncc - 0.02
```

mark that axis `ambiguous_candidate`. A plate or nominal prior may then place it, but its source remains inherited/nominal and `low_confidence` stays true. This handles allowed-range periodic aliases as well as the observed out-of-range dx=22 alias.

When two nonambiguous modes are otherwise exactly tied, an optional trusted expected step can break the tie by smallest main-axis distance, but it does not alter the 5–60% hard support.

### Failure mode and response

A pure 1-D periodic signal cannot identify an absolute translation modulo its period. The correct behavior is not to force the highest NCC mode; it is to call the local axis ambiguous and use independent plate geometry. If the whole plate contains only ambiguous stripes, use the declared nominal 29% vector and report `nominal_unvalidated`. Two-dimensional lattices contain orthogonal phase information and should continue to measure locally.

### Proof tests and pass criteria

1. **Gate unit test.** At width 200, candidates 80 and 190 must be accepted; 79.9, 190.1, and observed alias 22 must be rejected. Assert rejection occurs before top-K ranking.
2. **Measured stripe regressions.** At true `dx=142`, run periods 10, 20, and 58. `dx=22` must never enter the accepted set. The local result must either be within 1 px of truth or be `ambiguous_candidate`; after plate/nominal resolution the applied dx must be `142 +/-1 px`. An inherited result remains low-confidence. A confidence-0.99 result at dx=22 is an unconditional failure.
3. **Full supported range.** With nonperiodic texture, sweep 5%, 10%, 29%, 45%, and 60% overlap in both axes and transverse shifts from -3 to +3 px. Every endpoint/interior case must remain accepted; coarse main-axis error `<=1 px`, and after upgrade 4 `<=0.2 px`.
4. **Two-dimensional lattice.** Existing 2-D periodic fixtures remain accepted, have no prior rejection, and recover each main component within 1 px coarse / 0.2 px refined.
5. **QC.** The edge record must expose raw and in-prior candidate counts, implied overlap, and `outside_overlap_prior`/`ambiguous_candidate` reasons.

## 4. Subpixel refinement and per-edge QC

### Upsampled-DFT method

Keep `_phase_corr` and the multi-strip `_pair_candidates` as the coarse integer/alias search. After the overlap prior and consensus identify the winning coarse basin, choose the best candidate in that basin for every expected edge and call:

```text
_refine_shift_dft(a, b, coarse_shift, upsample_factor=10)
```

The new helper implements the Guizar-Sicairos local matrix-multiply DFT in NumPy:

1. Extract the equal-sized full-resolution overlap patches implied by `coarse_shift`, eroded by a 2 px guard.
2. Use flat-field-corrected images. High-pass at `ds=1`, subtract each mean, and apply a separable 2-D Hann window.
3. Compute `R = FFT(A) * conj(FFT(B))` and normalize `R /= abs(R) + eps`.
4. Evaluate the inverse DFT only on a 15x15 region (`ceil(1.5*10)` samples per axis) centered on zero residual using matrix multiplication. Do not allocate a 10-times zero-padded FFT.
5. If peak coordinate relative to the region center is `q`, set `delta=q/10` and return `coarse_shift+delta`. Preserve the convention: the returned `[dy,dx]` is tile B origin minus tile A origin.
6. Reject the refinement if the local peak touches the search boundary, values are nonfinite, either overlap dimension is under 32 px, or the refined result violates the overlap prior. A boundary peak means the coarse candidate was not in the right integer basin; the local search may not jump to another periodic alias.

An upsample factor of 10 gives a 0.1-pixel sampling grid. It is estimator precision, not a claim that every biological edge is accurate to 0.1 px.

Compute the final local axis vector as the coordinate-wise median of valid refined edges supporting the winning nonambiguous mode. Remove `_consensus(...).astype(int)`. Keep all edge translations for residual QC, but place tiles on the single refined lattice because the supplied real-data residuals show no distance growth.

### Fractional placement

Returning float steps while slicing at integer origins would be a no-op. Make the following rendering changes:

- `tile_origins` uses `float64` and normalizes by the floating minima.
- `mosaic_size` uses `ceil(max_origin + tile_shape)` for each axis.
- In `blend`, split each origin into floor and fractional `(fy,fx)`.
- Bilinearly splat both `image*feather` and `feather` to the four neighboring integer offsets with weights `(1-fy)(1-fx)`, `fy(1-fx)`, `(1-fy)fx`, and `fy*fx`.
- Divide accumulated intensity by accumulated weight. Translating numerator and weight identically preserves constants.
- Use `np.rint`, clip, and cast once after all tiles have accumulated.

Integer origins remain equivalent to the existing placement apart from fixing its downward truncation bias.

### Refinement failure

Flat/sparse overlaps remain unmeasurable; periodic alias selection must already have been rejected by upgrade 3. A coarse-valid edge whose local DFT fails records `subpixel_status="unavailable"` and its failure enum. It may be retained as a flagged coarse diagnostic, but it does not count as a refined donor edge. If refined support falls below the local-axis threshold, upgrade 2 supplies plate geometry. Rotation, distortion, focus changes, and local specimen motion remain outside the model and appear as edge residuals rather than being hidden.

### QC file and exact schema

When `qc` is supplied, `save_ome_tiff` writes one UTF-8 JSON object at:

```text
Path(str(ome_path) + ".stitch-qc.json")
```

For example, `A01.ome.tif` produces `A01.ome.tif.stitch-qc.json`. Use `schema_version="stitcher-qc/1.0"`. JSON numbers must be finite; unavailable values are `null`, never `NaN` or infinity.

Required top-level fields:

```text
schema_version, created_utc, output_ome_tiff, well
tile_shape_px_yx, output_shape_px_yx, source_dtype
reference_plane {key, channel, page, label}
z_projection_mode
shading_models[] {plane_key, model_id, cohort, method, status,
                   sample_count, working_size_px_yx, converged,
                   flatfield_p01_p50_p99, darkfield_min_median_max,
                   split_flat_median_rel, split_flat_p95_rel,
                   split_dark_rmse_dn, clipped_low_fraction,
                   clipped_high_fraction}
geometry_source_x, geometry_source_y
step_x_px_yx, step_y_px_yx, local_step_x_px_yx, local_step_y_px_yx
plate_step_x_px_yx, plate_step_y_px_yx
plate_donor_wells_x, plate_donor_wells_y
plate_rejected_wells_x, plate_rejected_wells_y
plate_scatter_x, plate_scatter_y, plate_status_x, plate_status_y
nominal_overlap_fraction_xy, fallback_used, low_confidence
tile_origins[] {tile_xy, origin_px_yx}
overlap_x_px, overlap_y_px
overlap_prior {min_fraction, max_fraction, nominal_xy, source}
registration_method, upsample_factor, subpixel_grid_px, shift_convention
edge_summary_by_axis
accepted_edge_ncc {median, p05, min}
residual_norm_px_by_axis {median, p95, max}
residual_vs_distance_slope_px_per_tile_by_axis
unreadable, warning_codes, edges[]
```

`edge_summary_by_axis` contains `expected`, `loaded`, `candidate_generated`, `in_prior`, `accepted`, `rejected`, `unmeasurable`, and `used_for_consensus` counts for X and Y.

Every expected +X/+Y adjacency derived from `wt.positions` gets exactly one edge object, even when a file is missing or the edge is unmeasurable. Required per-edge fields are:

```text
edge_id
axis
tile_a_xy, tile_b_xy
tile_a_source, tile_b_source
edge_midpoint_grid_xy
distance_from_grid_origin_tiles
status                         # accepted | rejected | unmeasurable
rejection_reason               # nullable enum
candidate_count_raw
candidate_count_in_prior
coarse_shift_px_yx
refined_shift_px_yx
subpixel_delta_px_yx
subpixel_status
prior_step_px_yx
prior_delta_px_yx
implied_overlap_px_yx
implied_main_overlap_fraction
overlap_sample_count
coarse_registration_ncc
post_refinement_registration_ncc
post_refinement_corrected_intensity_ncc
runner_up_ncc
ncc_margin
used_for_well_consensus
lattice_step_px_yx
lattice_residual_px_yx
lattice_residual_norm_px
```

Define `lattice_residual_px_yx = lattice_step_px_yx - refined_shift_px_yx`. `distance_from_grid_origin_tiles` is the Euclidean distance from the minimum grid index to the edge midpoint. The top-level residual-growth slope is the ordinary least-squares slope of residual norm against that distance.

Allowed `rejection_reason` values in schema v1 are:

```text
missing_tile
unreadable_tile
low_information
no_candidate
outside_overlap_prior
weak_ncc
ambiguous_candidate
dft_nonfinite
dft_peak_at_boundary
dft_overlap_too_small
consensus_outlier
```

For plate-fallback wells, attempted edges are still exported, `used_for_well_consensus=false`, and residuals are computed against the inherited vector wherever a usable refined edge exists.

### Proof tests and pass criteria

1. **DFT estimator.** At least 50 band-limited random textures with known fractional shifts, including negative transverse components: maximum component error `<=0.10 px` and shift-vector RMSE `<=0.08 px`.
2. **Fractional 5x5 lattice.** Gapped grid, 29% overlap, fractional main and cross-axis terms: every recovered step component within 0.10 px and maximum origin error across the grid `<=0.75 px`.
3. **Constant fusion.** Fractional-origin 2x2 uint8 and uint16 mosaics: every covered pixel is bit-exact and there are no interior coverage holes.
4. **Feature fidelity.** Fractionally shifted Gaussian beads crossing seams: centroid error `<=0.20 px` and uint8 mosaic RMSE `<=1 DN` in seam regions.
5. **Real eight-well regression.** Accepted-edge lattice residual median `<=1 px`, maximum `<=3 px`, and absolute residual-growth slope `<=0.1 px per grid tile`. These are checks against the supplied observed behavior, not new universal thresholds.
6. **Schema integrity.** A fixture containing accepted, flat, missing, and prior-rejected edges must produce exactly the expected adjacency count; all required fields are present; no JSON NaNs exist; recomputed residuals and summary quantiles match within `1e-6`.

## Cross-upgrade ordering constraints

Priority and execution dependency are not identical:

1. Implement and validate shading first. It must be active before Z projection, candidate generation, and saved-pixel fusion.
2. Implement the measurement/resolution split and plate coordinator next, but do not enable donor aggregation with the old candidate logic.
3. Activate the 5–60% gate and ambiguity rule before any well can become a plate donor; otherwise the confident stripe alias can poison the consensus.
4. Add subpixel edge refinement before the final per-well and across-well medians, so plate vectors retain fractional precision.
5. Resolve every well only after all local measurements are complete. Then calculate float origins and stitch all channels with the same geometry.
6. Export QC last because it consumes shading, candidate, refinement, local, plate, resolution, and rendering outcomes.

## Definition of done

The implementation is complete only when all four upgrade-specific test groups pass and an end-to-end real-data run demonstrates all of the following:

- saved pixels, not merely the alignment scorer, use validated per-plane flat/dark fields before Z projection;
- a sparse well never abuts and clearly reports whether it inherited plate or nominal geometry;
- none of the period-10/20/58 stripe fixtures can produce an unflagged wrong step;
- valid 5% and 60% overlap endpoints remain measurable;
- fractional origins are actually rendered rather than rounded away;
- every expected neighbor edge appears in the JSON sidecar; and
- the existing public call shapes and result layout continue to work.
