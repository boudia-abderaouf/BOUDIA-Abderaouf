# Annual Building Energy Consumption Modeling

**Cedrus Solutions · Machine Learning / Energy Analytics · Production pipeline**
**Role:** pipeline design and implementation, model training, evaluation and release tooling.

> This case study describes the engineering approach at a high level. Source code,
> customer data and proprietary implementation details remain private.

---

## Problem

Estimating a building's annual heating and cooling demand properly requires dynamic
thermal simulation: a physics engine, a fully specified building, and minutes of
compute per variant. That is far too slow to sit behind an interactive product, and it
cannot be run over a portfolio of assets on demand.

The goal was a **surrogate model**: something that reproduces the simulator's answer
in milliseconds, across climate zones and countries, with a measured and reproducible
gap to the physics it replaces.

## Context

The training data does not exist in the wild — it has to be manufactured. That makes
the project as much a *data-generation and orchestration* problem as a modeling one:
sample a plausible building population, simulate every sample, collect the results,
and only then train. Each stage can fail on its own schedule, and a simulation run is
expensive enough that silently losing or corrupting results is the main risk.

## My contribution

I built the end-to-end pipeline: parameter sampling, simulation orchestration, result
collection, training, evaluation, and the versioning/release layer that makes runs
comparable over time.

## Approach / Architecture

```
parameter sampling  →  physics simulation  →  result collection  →  training  →  evaluation
   (Monte-Carlo)        (per zone, batched)     (per zone dataset)    (per zone)    (+ reports)
```

**Sampling.** A configurable per-parameter distribution registry — uniform, normal,
and discrete draws with explicit probabilities, some of them multiplicative on a
surface or geometry term — produces a synthetic building population. Every parameter
of the building schema (envelope, geometry, systems, usage) has a declared
distribution rather than an ad-hoc range in code.

**Simulation.** Each sampled building is pushed through a commercial dynamic thermal
simulation engine, driven programmatically and run in chunks. The orchestration layer
detects truncated result chunks and re-simulates them, enforces per-simulation
timeouts, and dismisses blocking GUI dialogs so an unattended batch actually finishes.

**Training.** A multi-layer perceptron regressor per climate zone, on ~22–24 features
covering envelope, geometry, systems and usage, with min-max scaling computed from a
shared dictionary so that all zones of a country are scaled identically. Categorical
insulation type is one-hot encoded. Targets are annual heating and cooling demand.

**Orchestration.** Prefect flows with task-level dependencies, so a zone starts training
as soon as its own dataset lands rather than waiting for the whole generation to finish.
Runs are tracked in MLflow, grouped by country/zone/target, with plots logged as artifacts.

**Geographic structure.** `continent → country → climate zone` is first-class throughout,
including a worldwide zone registry and country groups, so a new country is configuration
rather than code.

## Engineering decisions

- **Versioning is separate for data and for models.** A dataset version and a training
  version are independent labels. Retraining on an existing dataset, or regenerating data
  for an unchanged trainer, are both first-class operations instead of a destructive
  overwrite. Manifests record generated samples, simulation results and per-zone datasets,
  with SHA-256 fingerprints of the code that produced them.

- **A local model registry, independent of MLflow.** Releases are snapshotted into a
  content-addressed registry and compared to the previous release for the same country.
  Comparison produces Markdown, CSV and JSON reports plus a PDF with prediction scatter
  plots — so "is the new model better?" has a reproducible artifact rather than a
  recollection. Release can be triggered automatically at the end of a country's training.

- **Metrics chosen for the decision they inform.** Beyond MAE and R², the evaluation
  reports the share of predictions within ±1 / ±5 / ±10 kWh, a mean relative error capped
  to keep near-zero denominators from dominating, and the share of predictions inside a
  ±5 % margin. An aggregate R² hides exactly the failure mode that matters here — a small
  set of buildings being badly wrong.

- **Per-chunk error analysis.** Relative and absolute error are also reported per chunk
  and as bias box-plots, which surfaces systematic bias by building regime rather than
  averaging it away.

- **A GPU training path that stays deployable.** An experimental trainer fits the network
  in PyTorch on GPU (batch-norm, large batches, dataset resident in VRAM), then folds
  batch-norm and target de-normalisation into the final layer and transplants the weights
  into a strictly equivalent scikit-learn estimator. Training speed without changing the
  serving artifact.

## Evaluation

- Train / validation / test splits per zone, with a fixed seed.
- MAE, R², capped mean relative error, in-margin rate, absolute/relative error
  distributions, predicted-vs-true scatter, per-chunk bias.
- Version-to-version comparison against a shared reference dataset, with an explicit
  status per zone/target rather than a single headline number.
- Unit tests covering the versioning, release and comparison logic.

Quantitative results are internal and not published here.

## Technologies

Python · scikit-learn (MLPRegressor) · PyTorch · NumPy · pandas · SciPy · Prefect ·
MLflow · Pydantic · joblib · Matplotlib · pytest · Docker · Poetry · loguru

## What I learned

- **Generating the dataset is the project.** The model was the least uncertain part;
  everything expensive lived in getting trustworthy simulation output at scale.
- **Reproducibility has to be designed in, not added.** Separating dataset and training
  versions, and fingerprinting the code, is what made a six-month-old run explainable.
- **A comparison report beats a metric.** The release-vs-previous artifact is what
  actually gets used when deciding to ship a model.

## Links

Private — company codebase.
