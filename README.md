# Agroforestry project
Climate station analyses


Data & QC pipeline (staged, idempotent)

All steps are versioned (*_vNNN.csv) and keep row_id so merges are traceable. We prefer conservative masking and keep reasons as qc_* flags.

Step 1 – Parse & normalize
Parse timestamps; standardize node IDs; tag station_type (agroforestry/open/reference); basic diagnostics.

Stage 2–4 – Soil moisture pre-QC
Range checks, known-broken mapping, flatline detection; write *_clean and qc_sm_* flags.

Step 5R – Soil moisture regimes (conservative)
Detect long monotonic ramps (WARN) and runaway variance (CRIT) on probe time series. Only a whitelisted probe auto-masked for runaway.

Step 5PD – Soil moisture pair-divergence
Detect sustained divergence of a probe from its sibling (same node, other depth) or a manual neighbor composite (from neighbors_manual.csv).
Robust rolling medians/MAD, edge-relaxed run gating, and per-row mapping of CRIT_pair_diverge.

Step 6PD – Soil temperature pair-divergence (aggressive)
Same idea as 5PD plus self-based anomaly fallback (spikes, fast rates, cross-depth inconsistencies) so we still catch issues when references are weak/missing.

Step 7a/7b/7c – Air QC
Temperature (self-checks), Humidity (lenient) with daylight context (implausible flat/sticky runs masked only if long or in daylight), and Pressure cross-checks. Preferred columns exported as temperature__pref, humidity__pref, pressure__pref.

Step 8 – Radiation & wind range/order
Range and ordering checks; preferred *_clean retained.

Step 9 – Assemble analysis_ready_vNNN.*
Join the latest valid outputs, enforce known-broken masking, build __pref for all variables, and export ok_* booleans + all qc_* flags. Also writes a Parquet copy.

Key artifacts

af_clean_v01/analysis_ready_vNNN.csv (and .parquet) — the single source for analyses.

neighbors_manual.csv — optional lookup of neighbor node IDs for composite references.

Feature engineering (what & why)

VPD (from temperature and humidity): captures atmospheric drying power that drives plant water loss; more actionable than RH alone.

PWP risk (share of nodes below a soil-moisture threshold per site-day): operational KPI for irrigation/stress alerts—quantifies how much of a site is in critical dryness on that day.

Distance gradient & side: encodes perpendicular distance to the tree row (1–4–8–16.5 m) and Left/Right relative to rows to capture shading/wind asymmetries.

Seasons & DOY: seasonal context for stratified summaries and between-season contrasts.

Node-equal daily means (for gradient work): avoid coverage bias across nodes.

Analysis design
Q1 — AF vs Open (paired daily)

Build daily site means and restrict to days with both AF & Open present (paired).

Statistics: paired sign-flip/permutation tests; bootstrap CIs; FDR across variables.

Plots: seasonal curves, AF–Open difference curves with CIs, ECDFs, paired scatters.

Q2 — Gradient within AF

Re-weight to node-equal daily means per distance bin (1–4–8–16.5 m) and side.

Tests: robust OLS/Wald omnibus for gradient; pairwise bin contrasts with multiple-testing control; Left–Right asymmetry checks.

Plots: distance profiles (per season), effect bars (e.g., 1 m–8 m), across-row smooths, and compact significance matrices.

Q3 — Soil moisture stress (PWP)

Compute site-day PWP risk = fraction of nodes below threshold.

Compare AF vs Open pooled and by season; show green (AF) vs brown (Open) bars with in-bar counts; test differences (e.g., exact/chi-square).

What we found (high-level)

Sides aren’t symmetric: Left vs Right near rows can behave differently—don’t pool them blindly.

Distance matters (season-dependent): near-row bins often differ from mid-field (e.g., summer air temp, soil moisture); effects can flip with season and wind/radiation context.

Open field shows higher PWP risk in aggregate in this dataset, though magnitude varies by season.

QC pipeline is doing its job: masked segments are traceable and derived from explicit rules (no black boxes).

See outputs/figs/ and outputs/tables/ for exact plots/tables per question.

How to run
Option A — Colab

Open each notebook in notebooks/ and run top-to-bottom. Upload or mount data, set DATA_ROOT, and execute.

Option B — Local
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt   # pandas, numpy, statsmodels, matplotlib, scipy, pyarrow, etc.
jupyter lab


Run 01_data_cleaning.ipynb to produce af_clean_v01/analysis_ready_vNNN.*, then proceed with 02_…, 03_…, 04_….

Reproducibility & conventions

Versioned outputs (_vNNN) and row_id joins keep merges safe.

Preferred series are __pref; QC flags are never dropped.

Plots export PNG + PDF with deterministic filenames.

Randomized procedures (permutations/bootstraps) set seeds inside notebooks.

Limitations & next steps

Coverage imbalance: some nodes have gaps—handled by node-equal means but still worth monitoring.

Wind direction / canopy not fully modeled yet—could refine seasonal gradients.

Threshold choices (e.g., PWP) are configurable—sensitivity analysis recommended.

Extend models to hierarchical mixed effects and incorporate year & weather covariates for gradient inference.
