# Pixel-first clustering and R-ratio analysis

## Quick start

Run a short test first:

```bash
python3 cluster_pixel_analysis.py \
  --hit-file ../detsim_sample/gampixpy_fullgeoanatruth-vd-reduced_g4_00_2Mhz_segmentlabel_lowtrig_5mmpitch.h5 \
  --truth-file ../g4_cv1_sample/fullgeoanatruth-vd-reduced_g4_00.h5 \
  --detector-yaml /home/yboxun/NeutrinoGAMPix/detsim_prediction/depth/far_detector_vd.yaml \
  --output-dir ../detsim_sample/cluster_pixel_first_test \
  --tag pixel_first \
  --max-events 10 \
  --truth-lookup event_segment \
  --truth-chunk-size 50000 \
  --make-plots
```

Run with Gaussian-fitted pixel charge and example fit plots:

```bash
python3 cluster_pixel_analysis.py \
  --hit-file ../detsim_sample/gampixpy_fullgeoanatruth-vd-reduced_g4_00_2Mhz_segmentlabel_lowtrig_5mmpitch.h5 \
  --truth-file ../g4_cv1_sample/fullgeoanatruth-vd-reduced_g4_00.h5 \
  --detector-yaml /home/yboxun/NeutrinoGAMPix/detsim_prediction/depth/far_detector_vd.yaml \
  --output-dir ../detsim_sample/cluster_pixel_first_fit_test \
  --tag pixel_first \
  --max-events 10 \
  --truth-lookup event_segment \
  --truth-chunk-size 50000 \
  --fit-waveforms \
  --plot-fit-examples \
  --fit-example-clusters 2 \
  --fit-example-hits-per-cluster 10 \
  --make-plots
```

For a full run, remove `--max-events`. Gaussian fitting is slower, so test with a small number of events before running the full sample.

---

## Overview

`cluster_pixel_analysis.py` performs a pixel-first waveform/readout analysis for GAMPixPy detector-simulation samples.

The analysis flow is:

```text
read pixel and tile waveform hits
  -> cluster pixel hits event by event
  -> match tile hits to each pixel cluster
  -> compute pixel/tile charge ratios
  -> match pixel-cluster labels to truth segments
  -> compute truth-weighted drift and position quantities
  -> write full cluster tables and diagnostic plots
```

The main physics quantity is a pixel-to-tile charge ratio,

```text
R = pixel charge / tile charge
```

computed with several charge definitions, including raw waveform sums, thresholded waveform sums, and optional Gaussian-fitted pixel charge.

The script always keeps the full uncut cluster table. Quality-cut tables are written as additional convenience outputs and do not replace the full sample.

---

## Input files

### Hit/readout file

The hit file is required. It should be a GAMPixPy-style HDF5 file with top-level datasets:

```text
pixels
tiles
```

The `pixels` dataset should contain fields such as:

```text
event id or event_id
pixel x
pixel y
start t or trig t
waveform
label
attribution
```

The `tiles` dataset should contain fields such as:

```text
event id or event_id
tile x
tile y
start t or trig t
waveform
label
attribution
```

The waveform is assumed to contain 20 time ticks by default. This can be changed with `--waveform-ticks`.

### Truth file

The truth file is optional but is needed for truth-position and drift-length columns. It should contain a top-level dataset:

```text
segments
```

with fields such as:

```text
event_id or event id
segment_id or segment id
dE
x, y, z
```

or alternatively start/end position fields:

```text
x_start, x_end
y_start, y_end
z_start, z_end
```

If no truth file is provided, the script still runs, but truth-related columns are empty or `NaN`.

### Detector YAML

The detector YAML is used to compute geometry-based drift information from `drift_volumes`.

Default path:

```text
/home/yboxun/NeutrinoGAMPix/detsim_prediction/depth/far_detector_vd.yaml
```

Override it with:

```bash
--detector-yaml PATH_TO_DETECTOR_YAML
```

---

## Pixel clustering

Pixel hits are clustered at the channel/hit level, not at the individual waveform-sample level.

For each detector event, a pixel hit is represented by:

```text
pixel x
pixel y
start/trigger time
```

DBSCAN is applied event by event using normalized coordinates:

```text
x_norm = pixel_x / (cluster_eps_space * pixel_pitch)
y_norm = pixel_y / (cluster_eps_space * pixel_pitch)
t_norm = pixel_time / cluster_eps_time
```

Default clustering parameters are:

```text
pixel_pitch          = 5.0
cluster_eps_space    = 1.5
cluster_eps_time     = 4.0
dbscan_eps           = 1.0
dbscan_min_samples   = 2
```

The corresponding command-line options are:

```bash
--pixel-pitch 5.0
--cluster-eps-space 1.5
--cluster-eps-time 4.0
--dbscan-eps 1.0
--dbscan-min-samples 2
```

To allow isolated single-pixel hits to become clusters, use:

```bash
--dbscan-min-samples 1
```

---

## Tile matching

After a pixel cluster is found, tile hits from the same event are matched to that cluster.

A tile is treated as a square region with side length:

```text
tile_size = tile_size_pixels * pixel_pitch
```

The default is:

```text
tile_size_pixels = 20
pixel_pitch = 5
tile_size = 100
```

The time readout window is:

```text
readout_window = waveform_ticks * tick_size
```

with defaults:

```text
waveform_ticks = 20
tick_size = 0.5
readout_window = 10
```

A tile is matched to a pixel cluster if:

1. the tile spatial area overlaps the pixel-cluster spatial bounding box;
2. the tile readout time window overlaps the cluster time window.

The number of matched tile channels is saved in:

```text
n_tile_hits
```

When `--make-plots` is used, the script writes:

```text
plots/hist_n_tile_hits.png
```

This plot shows how many tile channels each pixel cluster is matched to.

---

## Charge definitions and R ratios

### Raw waveform charge

Raw charges are simple waveform sums:

```text
pixel_charge = sum of all pixel waveform samples in the cluster
tile_charge  = sum of all matched tile waveform samples
R_pixel_over_tile = pixel_charge / tile_charge
```

### 3-sigma thresholded charge

The default noise value is:

```text
noise = 50 e
```

The default threshold is:

```text
threshold = 3 * noise = 150 e
```

The corresponding options are:

```bash
--noise 50
--threshold-sigma 3
```

The thresholded charge columns are:

```text
pixel_charge_3sd
tile_charge_3sd
R_3sd_pixel_over_tile_rawtile
R_3sd_pixel_over_tile_3sdtile
```

where:

```text
pixel_charge_3sd = sum of pixel waveform samples with charge > threshold
tile_charge_3sd  = sum of tile waveform samples with charge > threshold
```

The script also writes threshold-subtracted quantities:

```text
pixel_charge_above_threshold_subtracted
tile_charge_above_threshold_subtracted
R_above_threshold_subtracted
```

These use:

```text
sum(max(waveform - threshold, 0))
```

instead of summing the original samples above threshold.

---

## Gaussian-fitted pixel charge

Gaussian fitting is enabled with:

```bash
--fit-waveforms
```

When enabled, the script performs two related fits:

1. **Summed cluster waveform fit**

   The script sums all pixel waveforms in a cluster and fits the summed waveform with:

   ```text
   A * exp(-0.5 * ((t - mu) / sigma)^2)
   ```

   Similar summed-waveform fit columns are also written for matched tile waveforms.

2. **Individual pixel-hit waveform fits**

   The script also fits each pixel-hit waveform in the cluster individually. The fitted charge is then summed over the pixel hits.

The individual-fit columns are:

```text
pixel_individual_fit_charge
pixel_individual_fit_charge_3sd
pixel_individual_fit_charge_above_threshold_subtracted
pixel_individual_fit_success_count
pixel_individual_fit_fail_count
```

The main fitted-threshold charge column is:

```text
pixel_individual_fit_charge_3sd
```

This is the sum over individual pixel-hit Gaussian fits of the fitted charge in the region where:

```text
Gaussian(t) > threshold
```

with the default threshold equal to 150 e.

The corresponding fitted R-ratio columns are:

```text
R_pixel_individual_fit3sd_over_rawtile
R_pixel_individual_fit3sd_over_3sdtile
```

The script also writes summed-cluster-fit comparison ratios:

```text
R_pixel_summed_fit3sd_over_rawtile
R_pixel_summed_fit3sd_over_3sdtile
```

### Gaussian-fit example plots

Use:

```bash
--fit-waveforms \
--plot-fit-examples \
--fit-example-clusters 2 \
--fit-example-hits-per-cluster 10
```

This saves individual-pixel waveform fit examples for the first few clusters. With the default values above, it writes at most:

```text
2 clusters * 10 pixel hits per cluster = 20 plots
```

The plots are saved under:

```text
plots/gaussian_fit_examples/
```

Each plot shows:

```text
pixel waveform
Gaussian fit
3-sigma threshold line
fitted charge
fitted charge above 3 sigma
fit sigma
```

This is intended as a lightweight diagnostic and does not plot every pixel hit.

---

## Truth matching and drift quantities

Pixel hits contain:

```text
label
attribution
```

The `label` field gives truth segment IDs associated with the hit, and `attribution` gives the corresponding contribution/weight.

For truth-weighted quantities, the script requires both:

```text
attribution > 0
waveform charge > 0
```

for a label entry to contribute. Therefore, for example:

```text
label       = [1,   2,   3]
attribution = [0.5, 0.5, 0]
```

or

```text
attribution = [0.5, 0.5, -1]
```

or

```text
attribution = [0.5, 0.5, -999]
```

will use only labels `1` and `2`. Label `3` is ignored.

Truth segment matching can be controlled with:

```bash
--truth-lookup event_segment
```

Available modes are:

```text
event_segment
segment_id
event_segment_then_segment_id
```

Recommended for event-based detector samples:

```bash
--truth-lookup event_segment
```

This matches truth as:

```text
(event_id, segment_id)
```

The `segment_id` mode matches only by segment ID and is mainly useful for compatibility checks.

### Drift definition

The geometry-based drift value is computed from the truth segment midpoint:

```text
segment_midpoint = [(x_start + x_end)/2,
                    (y_start + y_end)/2,
                    (z_start + z_end)/2]
```

and detector geometry:

```text
drift = dot(segment_midpoint - anode_center, -drift_axis)
```

where `anode_center` and `drift_axis` come from:

```text
detector_config["drift_volumes"]["volume_0"]
```

The default drift mode is:

```bash
--drift-mode signed_volume0
```

No absolute-value correction is applied in this mode.

The main drift columns are:

```text
drift_length_attr_avg
drift_length_dE_avg
```

where:

```text
drift_length_attr_avg = attribution-weighted average over matched truth segments
drift_length_dE_avg   = dE-weighted average over matched truth segments
```

Diagnostic drift columns are also saved:

```text
drift_geometry_attr_avg
drift_geometry_dE_avg
drift_325_minus_x_attr_avg
drift_325_minus_x_dE_avg
drift_geometry_signed_attr_avg
drift_geometry_signed_dE_avg
```

---

## Output files

Suppose the command uses:

```bash
--tag pixel_first
--output-dir ../detsim_sample/cluster_pixel_first_test
```

The main output files are:

```text
pixel_first_clusters_all.csv
pixel_first_cut_R_0_1.csv
pixel_first_cut_R3sd_rawtile_0_1.csv
pixel_first_matched_tiles_only.csv
pixel_first_summary.json
```

If `--fit-waveforms` is used, the script also writes:

```text
pixel_first_cut_Rfit3sd_rawtile_0_1.csv
```

The most important file is:

```text
pixel_first_clusters_all.csv
```

This is the full uncut cluster table. Use this file for studying the full behavior of `R`, drift length, tile multiplicity, and fitted-charge quantities.

The cut files are convenience outputs only.

---

## Important output columns

### Cluster identity and matching

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

### Gaussian-fit quantities

Present when `--fit-waveforms` is used:

```text
pixel_fit_amp
pixel_fit_mu
pixel_fit_sigma
pixel_fit_charge
pixel_fit_charge_3sd
pixel_fit_success

tile_fit_amp
tile_fit_mu
tile_fit_sigma
tile_fit_charge
tile_fit_charge_3sd
tile_fit_success

pixel_individual_fit_charge
pixel_individual_fit_charge_3sd
pixel_individual_fit_success_count
pixel_individual_fit_fail_count

R_fit_pixel_over_tile
R_pixel_individual_fit3sd_over_rawtile
R_pixel_individual_fit3sd_over_3sdtile
R_pixel_summed_fit3sd_over_rawtile
R_pixel_summed_fit3sd_over_3sdtile
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
truth_y_attr_avg
truth_z_attr_avg
truth_x_dE_avg
truth_y_dE_avg
truth_z_dE_avg
drift_length_attr_avg
drift_length_dE_avg
segment_ids
segment_weights
```

---

## Diagnostic plots

With:

```bash
--make-plots
```

the script writes plots under:

```text
OUTPUT_DIR/plots/
```

Important plots include:

```text
hist_n_tile_hits.png
hist_R_pixel_over_tile.png
hist_R_3sd_pixel_over_tile_rawtile.png
hist_drift_length_attr_avg.png
R_vs_drift_length_attr_avg.png
```

When Gaussian fitting is enabled, additional plots include:

```text
hist_R_pixel_individual_fit3sd_over_rawtile.png
hist_R_pixel_individual_fit3sd_over_3sdtile.png
hist_pixel_individual_fit_success_count.png
hist_pixel_individual_fit_fail_count.png
R_fit3sd_rawtile_vs_drift_length_attr_avg.png
```

When example fit plotting is enabled, individual pixel-fit examples are saved under:

```text
OUTPUT_DIR/plots/gaussian_fit_examples/
```

---

## Runtime and memory notes

### Event limit

Use:

```bash
--max-events 10
```

or another small value for testing. This limits the number of detector events processed.

### Truth loading

The truth file can be large. The script first scans the selected detector events to collect the needed segment labels, then loads only matching truth rows in chunks.

The truth chunk size is controlled by:

```bash
--truth-chunk-size 50000
```

A smaller value uses less memory but may be slower.

### Pixel-hit cap for debugging

Use:

```bash
--max-pixel-hits-per-event 5000
```

only for quick debugging. This keeps the largest-charge pixel hits in each event and can bias physics distributions.

### Gaussian fitting

Gaussian fitting can be significantly slower than waveform summing. A small test run is recommended before a full run:

```bash
--max-events 10 --fit-waveforms
```

---

## Dependencies

Required:

```bash
pip install numpy pandas h5py scikit-learn
```

Optional but recommended:

```bash
pip install matplotlib scipy
```

`matplotlib` is required for `--make-plots`.

`scipy` is required for `--fit-waveforms`.

The geometry drift calculation requires `gampixpy` and access to the detector YAML.

---

## Recommended workflow

1. Run a small sample without Gaussian fitting:

   ```bash
   python3 cluster_pixel_analysis.py \
     --hit-file YOUR_HIT_FILE.h5 \
     --truth-file YOUR_TRUTH_FILE.h5 \
     --detector-yaml YOUR_DETECTOR.yaml \
     --output-dir test_output \
     --tag test \
     --max-events 10 \
     --truth-lookup event_segment \
     --truth-chunk-size 50000 \
     --make-plots
   ```

2. Inspect:

   ```text
   test_output/test_clusters_all.csv
   test_output/test_summary.json
   test_output/plots/hist_n_tile_hits.png
   test_output/plots/hist_R_3sd_pixel_over_tile_rawtile.png
   test_output/plots/hist_drift_length_attr_avg.png
   ```

3. Run a small sample with Gaussian fitting:

   ```bash
   python3 cluster_pixel_analysis.py \
     --hit-file YOUR_HIT_FILE.h5 \
     --truth-file YOUR_TRUTH_FILE.h5 \
     --detector-yaml YOUR_DETECTOR.yaml \
     --output-dir test_output_fit \
     --tag test \
     --max-events 10 \
     --truth-lookup event_segment \
     --truth-chunk-size 50000 \
     --fit-waveforms \
     --plot-fit-examples \
     --fit-example-clusters 2 \
     --fit-example-hits-per-cluster 10 \
     --make-plots
   ```

4. Check the Gaussian-fit example plots and fitted-charge columns.

5. After the settings look reasonable, run the full sample without `--max-events`.

---

## Notes and caveats

- The full uncut table is the primary output. Do not rely only on cut tables for physics-behavior studies.
- DBSCAN parameters may need tuning depending on event occupancy, time scale, and noise conditions.
- Tile matching uses overlap of spatial bounding boxes and readout windows.
- The `segment_id` truth-lookup mode can mix truth segments across events if segment IDs are not globally unique. For event-based samples, `--truth-lookup event_segment` is recommended.
- Geometry-based drift is signed by default. The signed distribution should be checked against the detector coordinate convention before applying physical cuts.
