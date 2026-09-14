# National Building Data Platform — Analytics Engineering

**Cedrus Solutions · Analytics Engineering / Data Modeling · Production pipeline**
**Role:** dbt project architecture, SQL modeling, spatial indexing and run automation.

> This case study describes the engineering approach at a high level. Warehouse
> configuration, credentials and internal schema details remain private.

---

## Problem

France publishes a national building database (BDNB) that consolidates building
geometry, addresses, cadastral parcels, ownership, energy-performance certificates (DPE)
and aggregated local energy-delivery data. It is authoritative and it is public — and in
raw form it is unusable for analysis: dozens of interrelated source tables, inconsistent
typing, and a volume that makes national-scale joins impractical without deliberate
partitioning.

The platform needed analysis-ready tables: **one row per building group**, enriched with
its addresses, parcels, certificate data and energy indicators, refreshed as new versions
of the source database are released.

## Context

Two constraints shaped the design:

- **Scale forces partitioning.** Building the whole country as one table is not viable.
  Each French *département* is built as its own materialisation — around 95 runs — which
  keeps every individual build bounded and independently re-runnable.
- **The source schema moves.** Source tables and columns change between releases, so
  source declarations and model scaffolding have to be generated rather than hand-written.

## My contribution

I built the dbt project: layer architecture, the SQL transformations, the macro layer
that generates source declarations and model configuration, the spatial indexing
strategy, the data-quality tests, and the automation that runs the whole country.

## Approach / Architecture

Standard dbt layering, applied strictly:

```
sources/        declared source tables, generated per source-database version
  ↓
staging/        one model per source entity — typing, renaming, safe casts, cleanup
  ↓
intermediate/   joins and enrichment — building group assembled from its components
  ↓
marts/          the analysis-ready fact table: one row per building group, per département
```

- **Staging** normalises each raw entity: addresses, building constructions, building
  groups, parcels, owners, DPE records, and the aggregated electricity / gas / heat-network
  delivery tables. Materialised as tables, with safe numeric casting so a single malformed
  value does not fail a whole build.

- **Intermediate** assembles the building group and its energy-delivery aggregates,
  collapsing one-to-many relations into per-group structures.

- **Marts** produces the final fact. Address lists are exploded with a lateral `unnest`
  and fall back to the building's canonical address when the list is empty; parcels are
  aggregated per group; the result is exactly one row per building-group identifier.

- **Parameterised by département.** The target is a run variable, so `dbt run --vars
  '{"dep": "..."}'` builds one département and a shell loop builds the country.

## Engineering decisions

- **Indexes as post-hooks, not as an afterthought.** Every mart build creates its own
  B-tree indexes on the join keys and **GiST indexes on the geometry columns** (building
  group, address, aggregated parcels) in a post-hook, each guarded so a failure on one
  index does not abort the build. Downstream spatial queries are the entire point of the
  table; an unindexed geometry column makes it useless.

- **Session guards on every model.** Pre-hooks set a lock timeout, a statement timeout and
  an idle-in-transaction timeout. On a warehouse this size, a build that blocks forever
  while holding a lock is a worse outcome than a build that fails.

- **Code generation over hand-maintenance.** Macros generate source declarations and model
  YAML scaffolding from the live source database, and resolve the source name from the
  configured source-database version. When the upstream release changes, the declarations
  are regenerated rather than patched by hand.

- **Tests that encode the contract.** The grain is asserted (`not_null` + `unique` on the
  building-group key), relationships back to the parent intermediate models are tested, and
  address uniqueness is tested at `warn` severity — it is a quality signal, not a build
  stopper. Column-level descriptions and types are documented in the schema files, so
  `dbt docs` produces real documentation and lineage rather than an empty graph.

- **Source version as a project variable.** The upstream database version is declared once
  in the project configuration and flows through the source-resolution macros.

## Evaluation

- dbt tests on grain, nullability and referential integrity, run per département.
- dbt-generated documentation and lineage graph.
- Dependencies pinned: `dbt_utils` and `codegen` at explicit versions.

## Technologies

dbt · SQL · PostgreSQL · PostGIS (GiST spatial indexes, Lambert-93 geometries) ·
Jinja macros · dbt_utils · dbt codegen · Bash automation

## Wider data work in the same programme

The same programme included adjacent Python work I contributed to: statistical analysis
and Random-Forest modeling of tertiary buildings, cross-referencing of geographic and
energy datasets over PostgreSQL/PostGIS with MapServer (WMS/WFS) publication, KNN-based
imputation and integrity reporting across public data sources, and property-value
prediction with XGBoost on the French open transaction dataset (DVF).

## What I learned

- **Partitioning is a modeling decision, not an ops detail.** Making the département the
  unit of build is what made the whole thing operable, testable and re-runnable.
- **Generated sources age better than written ones.** The macro layer paid for itself at
  the first upstream schema change.
- **Timeouts are part of correctness** on a warehouse that other people are also using.

## Links

Private — company codebase. The underlying source database (BDNB) and the property
transaction dataset (DVF) are French public open data.
