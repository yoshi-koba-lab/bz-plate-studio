# KTF / raw-tile start-flow design

This revision supersedes the earlier draft without modifying it.

## Decision

Add an explicit chooser before folder discovery, then keep two typed workflows:

```text
Launch / File > Choose Workflow…
  ├─ View stitched mosaics (.ktf)
  │    → KTF discovery → existing KTF experiment loader/viewer
  └─ Stitch raw image tiles
       → raw structural discovery → raw well selection → existing stitcher
```

Do not infer the mode from a folder and do not put raw `WellTiles` into the KTF model. Use:

- `self._mode`: `None`, `"ktf"`, or `"raw"`;
- `self._experiment`: KTF experiment only;
- `self._raw_experiment`: raw experiment only;
- `self._current_well`: existing KTF selection only;
- `self._current_raw_well`: raw selection only.

The supplied `main_entry.py` is an excerpt: it contains no `MainWindow` declaration, constructor, imports, or `main()`. Its `_scan_root()` comment says omitted startup code calls it. That omitted call site is part of this change.

## 1. Start chooser, return path, and relaunch

Show a reusable modal `StartModeDialog` after the main window first appears. It has two cards:

- **View stitched mosaics (`.ktf`)** — “Open whole-well mosaics already stitched by the microscope.” Button: **Choose KTF Folder…**
- **Stitch raw image tiles** — “Open per-field `X###Y###` tiles and create whole-well images.” Button: **Choose Raw Folder…**

Each card shows its last successfully loaded browse root, if any. Choosing a card opens a directory picker there. Closing the directory picker returns to the chooser; closing a chooser invoked from an existing session returns to that session.

Add **File > Choose Workflow…** to get back later. `Ctrl+O` remains **Open Folder…** for the active workflow. If `self._mode is None`, `Ctrl+O` reopens the workflow chooser. Store both menu actions as attributes so they can be disabled while a stitch is running.

On fresh-launch chooser cancellation, leave the neutral window visible with “Choose File > Choose Workflow… to begin.”

### `QSettings`

- Always show the chooser on relaunch. Omitted startup code must stop calling `_scan_root(last_root)` automatically.
- Add `last_ktf_root` and `last_raw_root`.
- Use legacy `last_root` only as a fallback for KTF when `last_ktf_root` is absent. Do not copy it into raw mode and do not delete the old key.
- A missing/non-directory saved path falls back to `Path.home()` and is not scanned.
- Keep the folder selected in the picker as a pending browse root. Commit that root to the mode-specific key only after a leaf from that scan loads at least one readable well. For a multi-experiment parent/root, this means the parent/root is saved after the first successful leaf load. Corrupt-only candidates do not change settings.
- Cancellation and a completed scan with no candidates do not change settings.

Mode/session changes are also transactional: do not clear the current tree, plate, or viewer until discovery finishes and a selected experiment loads successfully.

## 2. Raw mode

### Accepted folder levels

Raw mode accepts:

- the experiment directory itself;
- any parent holding one or more experiments at any depth;
- a drive or mount root.

Test the selected directory itself before descending. If one experiment is found, select and load it. If several are found, populate the experiment tree and wait for a double-click. A well or `X###Y###` folder alone is outside the accepted contract; explain that the experiment or an ancestor must be selected.

### Raw experiment detection

A candidate must have this direct relationship:

```text
experiment/
  well/
    X<digits>Y<digits>/
      *_X<digits>Y<digits>[_Z<digits>]_CH*.bz.ome.tif[f]
```

Use the same filename contract as the working stitcher:

- position directory: `stitcher.POS_RE.fullmatch()`;
- direct child regular file: `stitcher.TILE_RE.search()`;
- case-insensitive `.tif`/`.tiff` behavior comes from `TILE_RE`;
- ignore `._*` AppleDouble entries;
- do **not** add a directory-X/Y-versus-filename-X/Y equality rule, because current `scan_well()` intentionally derives the position from the filename and does not enforce that rule;
- `.ici` and `.ibc2` may be recorded as experiment metadata but never qualify an experiment;
- `.gci` never qualifies an experiment.

The broad scan is structural. It must not open a TIFF, parse OME XML, or call `discover_wells()` for every directory. Its `RawExperimentSummary` only needs the path, structurally matched direct-well IDs, and `.ici`/`.ibc2` presence. Stop after the first matching tile in each candidate well; do not count every TIFF merely to decorate the tree.

After the user selects one experiment, call `stitcher.discover_wells(experiment_dir)` once. This builds the actual `WellTiles` models and verifies at least one readable tile per returned well; it does not promise that every slice is readable. Compare the structural well IDs with `wells.keys()` to report candidate wells that were unusable. Exact causes remain console diagnostics unless `discover_wells()` later gains a structured error result.

### Large-root scan

Run both mode scanners in a cancellable `ExperimentScanWorker`, never on the GUI thread. Preserve the existing KTF scanner’s observable matching, pruning, symlink, and mount traversal semantics; moving it to a worker must not change which KTF experiment directories it returns.

The worker must:

- check cancellation between directory reads;
- avoid following symlink directories, as the current `os.walk()` does by default;
- skip and count permission/read errors;
- prune below a direct-KTF experiment as today;
- prune below a raw experiment after finding its candidate wells;
- keep partial results private and commit only after normal completion;
- throttle progress signals to at most 10 per second (or every 250 directories), showing the current path and directories examined.

If the selected path is a filesystem root/mount root, first warn: “Scanning a whole drive may take several minutes.” Offer **Scan Drive**, **Choose a Narrower Folder**, and **Cancel**. **Scan Drive** remains supported with an indeterminate progress bar and **Cancel Scan** button. There is no fixed depth limit.

### Raw tree, plate, and action

In raw mode the tree header is `Name | Wells | Source`; leaves show the lightweight matched-well count and `Raw tiles`. KTF mode retains `Name | Wells | Channels`. Every leaf stores its path in `Qt.UserRole`, its mode in `Qt.UserRole + 1`, and the picker’s pending browse root in another role/context.

On successful `_load_raw_experiment(folder)`:

1. Set `self._raw_experiment = {name, path, wells, ici_path, ibc2_path}` and `self._current_raw_well = None`; clear the prior KTF display state and set `self._experiment = None`.
2. Populate the plate with `WellTiles` keys. Do not request `KtfInfo.thumbnail_jpeg`. Use a neutral tile-grid placeholder plus text such as `A02 — 21 fields · 30 Z · CH4`.
3. Define counts exactly: fields = `wt.n_tiles` (unique filename-derived X/Y positions), Z = `len(wt.z_values)` (the union of Z indices; missing `_Z` is index 0), and channels = `wt.channels` (uppercase file-level tags). Do not label TIFF-file count as “fields,” and do not imply every field/channel has every Z value.
4. Connect the plate signal to a new mode dispatcher. KTF mode forwards to existing `_on_well_clicked`; raw mode calls `_on_raw_well_selected`, which only sets `self._current_raw_well` and updates a text summary.
5. Hide and clear the Conditions tab in raw mode so stale KTF conditions cannot be displayed or saved against the wrong model.
6. Show a small raw workflow panel in place of the canvas/channel/export area. Its primary action is **Stitch Raw Tiles…**.

Move the existing `b5` stitch action out of the KTF Export group into this raw panel (or create the raw action and hide `b5` in KTF mode). KTF viewing/export buttons remain unchanged, but stitching is entered only through the raw workflow, as required by the mode split.

`StitchDialog` defaults to all wells. “Current well only” is disabled until `self._current_raw_well` is set and then displays that well ID. Starting a new raw experiment always clears that selection.

When stitching starts, set a dedicated stitch-busy state (or set `_busy = True`), disable folder/workflow actions, and retain the worker. `_on_stitch_done()` clears that state and restores the actions. This design does not add an in-progress stitch-cancel UI; the workflow remains locked until `done`.

## 3. Stitched output and reopening

**Do not offer internal reopening in this minimal change.** The viewer is KTL2-specific: `KtfInfo`, embedded thumbnails, `reconstruct_image()`, and `reconstruct_region()` all depend on a KTF header, footer, and tile index. `save_ome_tiff()` writes a `CYX` OME-TIFF, so passing it into those functions is invalid.

After stitching, show a format-neutral completion dialog with succeeded/failed counts, warnings, the output directory, and buttons **Open Output Folder** and **Close**. Implement the first button with `QDesktopServices.openUrl(QUrl.fromLocalFile(out_dir))`. Say which outputs were requested/written; mention Fiji, QuPath, or napari only when OME-TIFF/TIFF was produced. If zero wells succeeded, show a failure state rather than an ordinary success dialog.

Do not offer **Open in Viewer**, and do not make KTF discovery recognize `.ome.tif`. Internal OME-TIFF viewing would require a separate reader backend with preview/region-read methods and a viewer-wide reader interface; that is a separate feature, not an entry-path fix.

## 4. Exact function boundary

### Change in `main_entry.py`

- `StitchDialog.__init__()` — accept the selected raw well or `None`; label/disable “Current well only” correctly.
- `_setup_ui()` — add the raw panel and scan empty/progress state; store the plate tabs; connect `well_clicked` to a mode dispatcher; move/hide `b5` outside KTF mode.
- `_setup_menu()` — add/store **Choose Workflow…** and store the open action for busy-state control.
- `_open_folder()` — route by mode; with no mode, show the chooser.
- `_open_data_root()` — forward the active mode.
- `_find_experiment_dirs()` — preserve its KTF predicate/results and pruning, adding cooperative progress/cancellation for worker execution.
- `_open_path()` — accept explicit mode, start discovery, keep a pending browse root, and commit UI/settings only after successful readable load.
- `_populate_tree()` — keep KTF labels/count logic, but store `"ktf"` and the pending browse root on each leaf.
- `_scan_root()` — keep only as an explicit KTF compatibility alias; remove automatic relaunch use.
- `_on_experiment_selected()` — dispatch by the item’s stored mode and commit the pending browse root only after a successful load.
- `_load_experiment()` — keep KTF parsing/rendering semantics, but stage before replacing current state and return success only when at least one readable well exists.
- `_stitch_raw_tiles()` — use cached `self._raw_experiment["wells"]` and `self._current_raw_well`; remove the `self._experiment` gate and second raw scan; set stitch-busy state.
- `_on_stitch_done()` — clear busy state and show the format-neutral completion/failure dialog.

The omitted startup/constructor code must initialize mode/raw/pending-root/worker state and schedule `_show_start_chooser()` instead of scanning `last_root`.

### Add in `main_entry.py`

- `StartModeDialog`, `_show_start_chooser()`, and `_choose_folder(mode)`;
- `ExperimentScanWorker` plus scan completion/cancellation handlers;
- `_find_raw_experiment_dirs()` and `_populate_raw_tree()`;
- `_load_raw_experiment()` and `_on_raw_well_selected()`;
- `_dispatch_well_clicked()` and `_set_mode()`;
- a settings helper that resolves and commits the two recent-root keys.

The full `WellPlateWidget` implementation is absent from this excerpt. Add an explicit `set_raw_wells(dict[str, WellTiles])`/placeholder mode while retaining existing `set_wells()` behavior for KTF.

### Stay unchanged

- `_select_tree_item()` remains a shared path-only helper.
- Existing KTF well rendering, channel controls, detail-on-zoom, and export handlers remain KTF-only.
- `_reset_well_state()` remains the KTF display reset helper.
- `StitchWorker.run()`, `StitchWorker._write()`, `_on_stitch_progress()`, and all stitch option meanings remain unchanged.
- All of `ktf_reader.py` stays KTF-only, including `is_ktf_file()`, `scan_experiment_folder()`, `KtfInfo`, thumbnails, and reconstruction functions.
- Existing `stitcher.py` functions stay unchanged: `scan_well()`, `discover_wells()`, `stitch_well()`, alignment, projection, flat-fielding, QC, and `save_ome_tiff()`. A new pure structural helper may be added, but it must use the existing `POS_RE`/`TILE_RE` contract.

## 5. Failure and empty states

| State | Behavior |
|---|---|
| Neither format | Give a mode-specific “No KTF experiments” or “No raw-tile experiments” state with **Choose Another Folder** and **Choose Workflow…**. Preserve the current session and settings. |
| Both formats | The chosen mode is authoritative. KTF lists direct-KTF candidates only; raw lists structural raw candidates only. Never merge or auto-switch. |
| Wrong mode | Show the selected mode’s empty state and a route back to the chooser. Do not trigger a second full-drive scan merely to diagnose the alternative. |
| Huge root | Warn, scan in the worker, throttle progress, allow scan cancellation, avoid TIFF decoding, and report inaccessible-directory count. |
| Folder/chooser cancellation | Start nothing and preserve the current session/settings. |
| Scan cancellation | Cooperatively stop, discard partial results, and restore the prior/neutral state; settings stay unchanged. |
| Structural raw candidate, no readable wells | Report “Raw tile names were found, but no readable tile images were found,” include the path, and keep Stitch disabled. |
| Mixed valid/unusable raw wells | Populate/stitch valid wells; report the difference between structural well IDs and returned `WellTiles` IDs as unusable candidate wells. |
| Busy stitch | Workflow/open actions remain disabled until `done`; no hidden mode switch and no promise of mid-stitch cancellation in this change. |
| Zero stitch successes | Show failure with output path and warnings/errors; do not say stitching succeeded. |

## 6. Tests with pass criteria

1. **Fresh launch:** exactly two workflow cards appear and no scan begins before a choice.
2. **No active mode:** closing the launch chooser leaves a neutral window; `Ctrl+O` reopens the chooser.
3. **Settings matrix:** legacy-only `last_root` seeds KTF but not raw; `last_ktf_root` overrides legacy; missing paths fall back home without scanning; a successful load writes only the active mode key and saves the originally selected experiment/parent/root; invalid, corrupt-only, and cancelled cases write neither key.
4. **KTF regression:** experiment, parent, deep root, and (where supported) nested-mount fixtures return the same KTF set and grouping as current `_find_experiment_dirs()`; a KTF leaf dispatches to `scan_experiment_folder()` and existing viewing/export passes.
5. **Direct raw experiment:** the selected directory itself is tested, found once, and auto-loaded without KTF calls.
6. **Verified real layout:** `080626-3month-Isousa/` loads `A02`, `A03`, `C02`, and `C03` with no `.ktf`; displayed field/Z/channel values exactly equal the direct `discover_wells()` baseline, including the confirmed 21-position, 30-Z-value, `CH4` result.
7. **Parent and drive root:** multiple/deep raw experiments appear once; no well or position directory appears as an experiment. A whole-drive scan remains responsive and finds the same candidates as a synchronous fixture oracle.
8. **Structural positives:** case variants, `.tif`/`.tiff`, optional Z, and multiple `TILE_RE` channels qualify under a matching `POS_RE` directory. Detection imposes no extra coordinate-equality contract beyond the current stitcher.
9. **Structural negatives:** `.ici`/`.ibc2` alone, `.gci` alone, a position directory without a matching tile filename, AppleDouble files, and a directory merely named like a TIFF do not qualify.
10. **Broad-scan cost:** instrumented discovery opens zero TIFF payloads, stops after structural proof per well, emits progress no faster than the throttle, and commits no partial results after **Cancel Scan**. Permission errors are counted, not fatal.
11. **Raw model/counts:** fields equal `wt.n_tiles`, Z equals `len(wt.z_values)`, and channels equal `wt.channels` for multi-channel, missing-slice, optional-Z, and non-contiguous-Z fixtures.
12. **Mixed wells:** one valid and one corrupt-only candidate well loads the valid plate cell, reports the other as unusable, and allows stitching only the valid dictionary.
13. **Well dispatch/isolation:** a KTF plate click reaches existing `_on_well_clicked`; a raw click never does, sets only `_current_raw_well`, shows no KTF thumbnail/reconstruction, and leaves Conditions hidden/cleared.
14. **Raw selection reset:** loading a second raw experiment clears the previous current well; “Current well only” stays disabled until a new selection.
15. **Stitch handoff/busy state:** all-well and current-well choices pass the expected cached `WellTiles` dictionary, never require `self._experiment`, perform no second discovery, and keep workflow/open actions disabled until `done` restores them.
16. **Completion variants:** `both`, `ometiff`, `png`, and `split` each produce format-accurate completion text; warnings/failures and zero-success are distinct; **Open Output Folder** targets the exact output directory; no internal-viewer action appears.
17. **Both/neither:** a combined-layout folder follows only the chosen mode; an unrelated folder gives the right empty state without clearing the previous session or changing settings.
18. **Cancellation matrix:** chooser, folder picker, scan, stitch-options dialog, and output picker cancellation start no unintended load/worker and preserve the prior applicable state.
19. **Mode isolation:** raw discovery/load/stitch never calls `ktf_reader.scan_experiment_folder()`; KTF load/view never receives `WellTiles`; stitched `.ome.tif` output is not classified as KTF.
