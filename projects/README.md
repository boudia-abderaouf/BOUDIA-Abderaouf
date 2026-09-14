# Project case studies

Six pieces of work, written up as engineering case studies rather than repository
listings: the problem, the decisions, the evaluation, and what I personally built.

> Each of these also has an illustrated version on the
> [portfolio site](https://boudia-abderaouf.github.io/BOUDIA-Abderaouf/#work), with
> architecture diagrams and — for the published research — the results charts.

| Project | Domain | Type | Status |
|---|---|---|---|
| [Verifiable Document Intelligence for Building Energy Audits](document-intelligence.md) | LLM systems · RAG · structured extraction | Industrial — case study only | In production |
| [Annual Building Energy Consumption Modeling](annual-energy-model.md) | Machine learning · surrogate modeling · MLOps | Industrial — case study only | In production |
| [Deep Learning for Hourly Building Energy Forecasting](hourly-energy-forecasting.md) | Deep learning · time series | Industrial R&D — case study only | Ongoing, manuscript planned |
| [National Building Data Platform](dbt-data-platform.md) | Analytics engineering · dbt · PostGIS | Industrial — case study only | In production |
| [Fall Risk Assessment Using Gait Analysis](fall-risk-assessment.md) | Signal processing · machine learning | Research | Submitted, under review |
| [Enhanced Interactive Segmentation for Borehole Images](interseg-wesam.md) | Computer vision · domain adaptation | Research | Published — IEEE IGARSS 2025 |

## How to read these

Each case study follows the same structure — problem, context, my contribution,
approach, engineering decisions, evaluation, technologies, what I learned.

Four of the six describe work done inside a company codebase. For those, the write-up
covers **techniques and reasoning only**: no source code, no prompts, no customer or
site identifiers, no real building measurements, no internal module, schema or
infrastructure names. Where a quantitative result is given, it is either a
pipeline-level engineering metric (latency, token cost, corpus size) or a figure
already published in a peer-reviewed paper.

Where an experiment did not produce a clean win, that is stated as such.

## Classification

**Public case study only** — technique describable, code stays private:
document intelligence · annual energy model · hourly forecasting · dbt data platform

**Research publication** — paper plus high-level implementation detail:
fall risk assessment · interactive segmentation for borehole images
