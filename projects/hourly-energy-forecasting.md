# Deep Learning for Hourly Building Energy Forecasting

**Cedrus Solutions · Deep Learning / Time-Series · Ongoing research — manuscript planned**
**Role:** pipeline design and implementation, model design, training and evaluation.

> This case study describes the engineering approach at a high level. Source code,
> customer data and proprietary implementation details remain private.
> Results are not reported: the work is ongoing and a manuscript is planned.

---

## Problem

An annual energy figure is enough to rank buildings; it is not enough to reason about
them. Peak load, seasonal behaviour, the effect of an occupancy or setpoint change, the
value of a retrofit at the hour it actually bites — all of that lives in the **hourly**
profile.

The objective is an hourly surrogate: given a building's characteristics, its operating
scenario and a weather year, predict hourly heating and cooling power over the season,
fast enough to be used interactively.

## Context

This is a harder problem than the annual one in three ways:

- **Volume.** One building-year is thousands of hourly rows. A training corpus is
  hundreds of millions of rows, which rules out holding a dataset in memory naively.
- **Conditioning.** Hourly demand depends on time-varying inputs — weather, occupancy,
  internal gains, heating and cooling setpoint schedules — not only on static building
  characteristics.
- **Two regimes, not one.** Heating and cooling are different physical problems with
  different drivers and different seasons. Treating them as one target buries both.

## My contribution

I built the full chain: scenario and sampling generation, simulation orchestration,
weather handling, dataset assembly, the training workflow, and the reporting and
inference layers on top.

## Approach / Architecture

```
static sampling ─┐
scenario banks  ─┼→ input organisation → hourly simulation → dataset assembly → training
weather draws   ─┘                          (per building)      (Parquet)        (per target)
```

**Scenario generation.** Banks of occupancy profiles, internal heat gains, and heating
and cooling setpoint schedules are generated once and then held fixed. Fixed inputs live
in a separate tree from regenerated outputs, and the pipeline **fails loudly if a fixed
input is missing** rather than silently regenerating it — otherwise a corpus quietly
stops being comparable to the models trained on it.

**Weather.** Weather years are drawn from a stable source set and extracted alongside the
simulation outputs, so a training row carries the meteorological conditions it was
produced under.

**Dataset assembly.** Hourly simulation outputs, organised scenario inputs and extracted
weather are combined into a Parquet corpus, chunked and compressed, with per-source time
shifts applied so that lagged conditions align with the hour being predicted. Rows with
insufficient valid hourly coverage are dropped.

**Season blocks.** The calendar year is cut into explicit heating and cooling windows.
Each target trains only on its own regime — heating on the winter blocks, cooling on a
configurable summer window.

**Models.** The production path is a configurable PyTorch MLP: arbitrary hidden-layer
stack, ReLU/GELU/SiLU activation, optional batch-norm or layer-norm, optional dropout.
**LSTM** and **XGBoost** variants were implemented and evaluated against it — the
sequence model consumes `(batch, time, features)` windows and regresses from the final
hidden state. The simpler conditioned regressor was retained for the current workflow;
the sequence models remain the natural comparison for the planned manuscript.

## Engineering decisions

- **Sample windows, not whole years.** Training draws 168-hour (one-week) subsets at
  random offsets inside a season block. This keeps batches diverse, bounds memory, and
  acts as augmentation over the phase of the week.

- **Split by building, stratified.** Train/validation/test are split at the **building**
  level and stratified — by surface and by scenario — never by row. A random row split
  would leak the same building across splits and produce a meaningless score.

- **Huber loss over MSE.** Hourly power has heavy tails; squared error lets a handful of
  extreme hours dictate the fit.

- **Mixed precision, large batches, resumable runs.** AMP with a gradient scaler, batch
  sizes in the thousands, periodic checkpoints carrying optimiser and best-state, and
  resume-by-run-name. A multi-hour run that dies at epoch 30 does not start over.

- **ReduceLROnPlateau + early stopping + best-state restore**, with TensorBoard logging
  per run.

- **Evaluation on two levels.** Pointwise regression metrics (RMSE, MAE, R²) *and*
  per-building aggregated metrics over the reconstructed window — because a model can be
  respectable hour by hour and still get a building's seasonal total wrong. An explicit
  energy-conservation scaling factor is computed and reported, so systematic over- or
  under-prediction of the seasonal total is visible as its own quantity rather than
  hidden inside an average.

- **Feature-dictionary caching keyed by data fingerprint.** Scaling statistics are cached
  against a key built from the corpus contents and the active filters, so a re-run reuses
  them only when it is genuinely the same corpus.

- **Operational layer on top.** Beyond training, the repository carries single-building
  inference, auxiliary-consumption modelling and report generation — the path from a
  trained model to something an engineer reads.

## Evaluation

Test-set RMSE / MAE / R², per-building metrics over reconstructed profiles, temporal
overlays of predicted vs simulated profiles, global scatter, and the energy-conservation
factor. Comparison across model families (MLP / LSTM / gradient boosting) is part of the
planned write-up.

**Status: ongoing research.** No performance figures are published here, and nothing in
this work has been submitted or peer-reviewed yet.

## Technologies

Python · PyTorch · scikit-learn · XGBoost · NumPy · pandas · PyArrow / Parquet ·
TensorBoard · YAML-driven configuration · Matplotlib · seaborn · ruff

## What I learned

- **The split is the experiment.** Getting building-level stratified splitting right
  changed the measured difficulty of the problem more than any architecture change did.
- **Two metrics, two audiences.** Hourly error is for the model; seasonal-total error is
  for the engineer who has to trust it. Reporting only one hides the failure the other
  would catch.
- **Freeze your fixed inputs, loudly.** The most expensive class of bug here is the one
  where regenerated "fixed" data silently invalidates every earlier comparison.

## Links

Private — company codebase. Manuscript planned.
