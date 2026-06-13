# Pixel-first clustering and R-ratio analysis

This directory contains a standalone Python script:

```text
cluster_pixel_analysis.py
```

The script performs a pixel-first version of the waveform/readout analysis from `studyReadout1.1.ipynb`.

The old notebook-style logic was roughly:

```text
one tile hit -> find nearby pixel hits -> compute pixel/tile charge ratio R
```

The new script changes this to:

```text
cluster pixel hits first -> match corresponding tile hits -> compute the same R-style quantities
```

The main goal is to study how pixel charge compares with tile charge after real pixel clustering, and then compare those quantities with truth-level drift information.

---

## What the script does

For each event, the script:

1. Reads pixel and tile waveform hits from a GAMPixPy HDF5 file.
2. Computes the total charge of each pixel waveform.
3. Clusters pixel hits using DBSCAN in normalized `(x, y, t)` space.
4. For each pixel cluster, finds matching tile hits.
5. Computes pixel charge, tile charge, and several pixel/tile charge ratios.
6. Uses segment labels and attribution fractions to match the cluster to truth segments.
7. Computes truth-weighted quantities, including drift length.
8. Writes a full uncut CSV table, plus separate optional cut tables.
9. Optionally writes basic diagnostic plots.

The script does **not** replace the full output with a quality-cut version. The full uncut result is always kept.

---

## Input files

### 1. Hit/readout file

Required.

The hit file should be a GAMPixPy-style HDF5 file with top-level datasets:

```text
pixels
tiles
```

The script expects fields similar to:

For `pixels`:

```text
event id or event_id
pixel x
pixel y
start t or trig t
waveform
label
attribution
```

For `tiles`:

```text
event id or event_id
tile x
tile y
start t or trig t
waveform
label
attribution
```

The waveform is assumed to contain 20 time ticks by default.

### 2. Truth file

Optional, but needed for drift-length comparison.

The truth file should contain a top-level dataset:

```text
segments
```

with fields similar to:

```text
event_id or event id
segment_id or segment id
dE
x, y, z
```

or alternatively start/end positions such as:

```text
x_start, x_end
y_start, y_end
z_start, z_end
```

If a truth file is not provided, the script still runs, but truth and drift columns will be `NaN`.

---

## Important definitions

### Pixel-first clustering

Pixel hits are clustered at the channel/hit level, not by individual waveform time samples.

Each pixel hit is represented by:

```text
x position
y position
start or trigger time
total waveform charge
```

DBSCAN is run event-by-event in normalized coordinates:

```text
x_norm = x / (cluster_eps_space * pixel_pitch)
y_norm = y / (cluster_eps_space * pixel_pitch)
t_norm = t / cluster_eps_time
```

The default DBSCAN parameters are:

```text
cluster_eps_space = 1.5
cluster_eps_time  = 4.0
dbscan_eps        = 1.0
dbscan_min_samples = 2
```

This means neighboring pixels close in both space and time tend to be clustered together.

### Tile matching

After a pixel cluster is found, the script finds corresponding tile hits.

By default:

```text
pixel pitch = 5.0
tile size = 20 * pixel pitch = 100.0
```

So each tile is treated as a square area corresponding to `20 x 20` pixels.

The time readout window is:

```text
waveform_ticks * tick_size
```

with defaults:

```text
waveform_ticks = 20
tick_size = 0.5
```

A tile is matched to a pixel cluster if:

1. the tile spatial area overlaps the pixel-cluster spatial bounding box;
2. the tile time readout window overlaps the cluster time window.

### 3 sigma threshold

The default noise value is:

```text
noise = 50 e
```

The default threshold is:

```text
threshold = 3 * noise = 150 e
```

The script keeps both unthresholded and thresholded charge quantities.

Important charge columns include:

```text
pixel_charge
tile_charge
R_pixel_over_tile
```

and thresholded versions:

```text
pixel_charge_3sd
tile_charge_3sd
R_3sd_pixel_over_tile_rawtile
R_3sd_pixel_over_tile_3sdtile
```

Here:

```text
pixel_charge_3sd = sum of pixel waveform samples above 150 e
tile_charge_3sd  = sum of tile waveform samples above 150 e
```

The script also writes threshold-subtracted quantities:

```text
pixel_charge_above_threshold_subtracted
tile_charge_above_threshold_subtracted
R_above_threshold_subtracted
```

These use `max(waveform - threshold, 0)` rather than simply summing waveform samples above threshold.

---

## Main output files

Suppose you run with:

```text
--tag test_pixel
--output-dir ../detsim_sample/cluster_pixel
```

The main outputs are:

```text
test_pixel_clusters_all.csv
test_pixel_cut_R_0_1.csv
test_pixel_cut_R3sd_rawtile_0_1.csv
test_pixel_matched_tiles_only.csv
test_pixel_summary.json
```

The most important file is:

```text
test_pixel_clusters_all.csv
```

This is the full uncut result. Use this file to study the full behavior of physics quantities such as `R`.

The cut files are only convenience outputs. They do not replace the full table.

---

## Important output columns

### Cluster identity

```text
cluster_id
event_id
n_pixel_hits
n_tile_hits
pixel_global_rows
matched_tile_global_rows
```

### Charge and R-ratio quantities

```text
pixel_charge
tile_charge
R_pixel_over_tile
pixel_charge_3sd
tile_charge_3sd
R_3sd_pixel_over_tile_rawtile
R_3sd_pixel_over_tile_3sdtile
pixel_charge_above_threshold_subtracted
tile_charge_above_threshold_subtracted
R_above_threshold_subtracted
threshold
```

### Pixel-cluster geometry

```text
pixel_centroid_x
pixel_centroid_y
pixel_centroid_t
pixel_width_x
pixel_width_y
pixel_width_t
pixel_span_x
pixel_span_y
pixel_span_t
pixel_time_min
pixel_time_max
```

### Matched tile geometry

```text
tile_centroid_x
tile_centroid_y
tile_centroid_t
tile_width_x
tile_width_y
tile_width_t
tile_span_x
tile_span_y
tile_span_t
tile_time_min
tile_time_max
```

### Truth matching

```text
n_truth_segments
dominant_segment_id
dominant_segment_weight
truth_weight_sum
truth_dE_sum
truth_x_attr_avg
truth_x_dE_avg
drift_length_attr_avg
drift_length_dE_avg
segment_ids
segment_weights
```

The attribution-weighted drift length is usually the first quantity to check:

```text
drift_length_attr_avg
```

The `dE`-weighted version is:

```text
drift_length_dE_avg
```

---

## How to run

### Quick test run

From the directory containing the script:

```bash
python cluster_pixel_analysis.py \
  --hit-file ../detsim_sample/gampixpy_fullgeoanatruth-vd-reduced_g4_00_2Mhz_segmentlabel_lowtrig_5mmpitch_presample.h5 \
  --truth-file ../g4_cv1_sample/fullgeoanatruth-vd-reduced_g4_00.h5 \
  --output-dir ../detsim_sample/cluster_pixel \
  --tag test_pixel \
  --max-events 100 \
  --make-plots
```

### Full run

Remove the event limit:

```bash
python cluster_pixel_analysis.py \
  --hit-file ../detsim_sample/gampixpy_fullgeoanatruth-vd-reduced_g4_00_2Mhz_segmentlabel_lowtrig_5mmpitch_presample.h5 \
  --truth-file ../g4_cv1_sample/fullgeoanatruth-vd-reduced_g4_00.h5 \
  --output-dir ../detsim_sample/cluster_pixel \
  --tag full_pixel \
  --make-plots
```

### Run without truth matching

```bash
python cluster_pixel_analysis.py \
  --hit-file ../detsim_sample/gampixpy_fullgeoanatruth-vd-reduced_g4_00_2Mhz_segmentlabel_lowtrig_5mmpitch_presample.h5 \
  --output-dir ../detsim_sample/cluster_pixel \
  --tag no_truth_test
```

Truth-related columns will be empty or `NaN`.

---

## Useful options

### Change the pixel/tile geometry

```bash
--pixel-pitch 5.0
--tile-size-pixels 20.0
```

The tile side length is computed as:

```text
tile_size = pixel_pitch * tile_size_pixels
```

### Change the readout time window

```bash
--tick-size 0.5
--waveform-ticks 20
```

The readout window is:

```text
waveform_ticks * tick_size
```

### Change the clustering behavior

```bash
--cluster-eps-space 1.5
--cluster-eps-time 4.0
--dbscan-eps 1.0
--dbscan-min-samples 2
```

If you want isolated single-pixel hits to become clusters, use:

```bash
--dbscan-min-samples 1
```

### Change the threshold

The default is `3 * 50 e = 150 e`:

```bash
--noise 50
--threshold-sigma 3
```

For a different threshold, change either value. For example, a `5 sigma` threshold with the same noise is:

```bash
--threshold-sigma 5
```

### Limit runtime for debugging

```bash
--max-events 100
```

or:

```bash
--max-pixel-hits-per-event 5000
```

The second option keeps only the largest-charge pixel hits in each event, so it should be used only for testing.

### Fit summed waveforms

```bash
--fit-waveforms
```

This adds Gaussian-fit columns for the summed pixel and tile waveforms. It requires `scipy`.

---

## Python dependencies

Required:

```bash
pip install numpy pandas h5py scikit-learn
```

Optional:

```bash
pip install matplotlib scipy
```

`matplotlib` is only needed for `--make-plots`.

`scipy` is only needed for `--fit-waveforms`.

---

## Recommended workflow

First run a small test:

```bash
python cluster_pixel_analysis.py \
  --hit-file YOUR_HIT_FILE.h5 \
  --truth-file YOUR_TRUTH_FILE.h5 \
  --output-dir test_output \
  --tag test \
  --max-events 20 \
  --make-plots
```

Then inspect:

```text
test_output/test_clusters_all.csv
test_output/test_summary.json
test_output/plots/
```

Check these quantities first:

```text
n_pixel_hits
n_tile_hits
R_pixel_over_tile
R_3sd_pixel_over_tile_rawtile
R_3sd_pixel_over_tile_3sdtile
drift_length_attr_avg
```

After confirming the matching and clustering look reasonable, run the full sample without `--max-events`.

---

## Notes and caveats

- The main result is the uncut cluster table. Do not use only the cut tables for physics-behavior studies.
- DBSCAN parameters may need tuning depending on event density, time scale, and noise conditions.
- Tile matching currently uses overlap of bounding boxes and readout windows. This is intentionally simple and close to the original matching idea.
- Truth matching uses segment `label` and `attribution` from the pixel hits. The weights are based on waveform charge multiplied by attribution fraction.
- The default drift definition is `anode_x - truth_x`, with `anode_x = 325`. Change `--anode-x` if needed.
