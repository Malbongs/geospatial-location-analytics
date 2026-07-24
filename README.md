# Geospatial Location Intelligence Engine

**Nationwide demand–supply location analysis for retail siting — built with spatial statistics, from data pipeline to scoring engine.**

![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-150458?logo=pandas&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-013243?logo=numpy&logoColor=white)
![GeoPandas](https://img.shields.io/badge/GeoPandas-139C5A)
![Shapely](https://img.shields.io/badge/Shapely-3776AB)
![PyProj](https://img.shields.io/badge/pyproj-2C7BB6)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?logo=scikitlearn&logoColor=white)
![PostGIS](https://img.shields.io/badge/PostGIS-008BB9?logo=postgresql&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy%202.0-D71F00)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)

> Code-private project. This document presents methodology, scale, and role only — no code or screenshots.

## Overview

I designed and built — solo — a spatial analytics engine that scores the viability of **retail store locations** across an entire country at the administrative-district level. Given any of **3,523 districts across 250 municipalities**, it estimates how much unmet consumer demand a new store could realistically capture, by modeling demand, competition, accessibility, and cost-efficiency as distinct spatial forces rather than lumping them into a single heuristic.

The core idea is that "a good location" is not one number you can eyeball. It is the interaction of *where people are*, *where existing supply already is*, *how far people will actually travel*, and *what it costs to operate there*. Each of those is a measurable spatial quantity, and each has an established method in the geography and spatial-epidemiology literature. This engine implements those methods correctly — 2SFCA / E2SFCA for accessibility, the Huff probabilistic choice model for demand allocation — rather than approximating them.

The whole thing is roughly **12,353 lines of Python across 61 files**, fed by **19 heterogeneous public-data ETL connectors** normalized onto a single nationwide spatial grid.

## Methodology

The engine produces a composite **0–100 location score** from four axes, then layers six auxiliary indicators on top. The four axes are combined with **profile-specific weights**, and the weights themselves are *learned* rather than guessed (see Validation).

### The Four Axes

| Axis | Meaning | Drives |
|------|---------|--------|
| **D — Demand** | Spatial consumer demand potential in the catchment | How many customers exist |
| **C — Competitive Opportunity** | Gap between demand and existing supply | Whether the market is saturated or open |
| **A — Accessibility** | Spatially-decayed reachability of supply from demand | How easily people can actually get there |
| **R — Cost-efficiency** | Return relative to operating cost | Whether the economics work |

Composite score = weighted aggregation of D, C, A, R, normalized to 0–100.

### Accessibility: 2SFCA → E2SFCA

Accessibility is computed with the **Two-Step Floating Catchment Area (2SFCA)** method and its enhanced variant **E2SFCA** (Luo & Qi, 2009).

**Step 1 — supply-to-demand ratio.** For each supply location *j*, sum the demand of every population location *k* within the catchment:

```
R_j = S_j / Σ_k ( P_k · W_kj )
```

**Step 2 — accessibility at each demand point.** For each demand location *i*, sum the ratios of all supply within its catchment:

```
A_i = Σ_j ( R_j · W_ij )
```

The **enhanced** step in E2SFCA is the weighting function `W`. Instead of a hard binary catchment, I apply a **Gaussian distance-decay kernel** so nearby supply counts far more than supply at the edge:

```
G(d) = exp( −d² / (2·σ²) ),   σ = 4 km,   hard cutoff at 15 km
```

This gives a distance-sensitive accessibility surface rather than a step function that treats a store 1 km away the same as one 14 km away.

### Demand Allocation: Huff Probabilistic Choice Model

Raw demand has to be *allocated* — a customer surrounded by several options doesn't send all their spend to the nearest one. I use the **Huff model**, the standard spatial-interaction gravity model for retail patronage:

```
P(i → j) = ( A_j / D_ij^β ) / Σ_k ( A_k / D_ik^β ),   β = 2.0
```

Summing each destination's captured probability-mass over all origins yields **expected demand** per location.

### Competitive Opportunity and Unmet Demand

```
Unmet demand = Demand × Competitive opportunity
```

A district scores high only when it has *both* real demand *and* room for that demand to be served.

### Auxiliary Indicators (6)

1. **Market vitality** — net business formation (openings minus closures) as a momentum signal.
2. **Closure risk** — local churn/failure pressure.
3. **Commercial-district typology** — **K-means clustering** into archetypes, cluster count validated by **silhouette score**.
4. **Specialized demand** — demand tilt for particular segments.
5. **Expected market size** — projected addressable market volume.
6. Ranking diagnostics via **Spearman rank correlation**, implemented directly in NumPy.

## Data Pipeline

Nineteen **public-data ETL connectors** pull from heterogeneous government OpenAPIs, each with its own schema, coordinate convention, key format, and update cadence.

1. **Extract** — 19 connectors against public OpenAPIs, with per-source retry, rate-limit handling, and pagination.
2. **Normalize** — reconcile every source onto a single administrative-district key space (250 municipalities, 3,523 districts).
3. **Spatial join** — reproject to a common CRS (**pyproj**), then join geometries (**GeoPandas / Shapely**) onto district boundaries.
4. **Load** — persist to **PostGIS** via **SQLAlchemy 2.0 + GeoAlchemy2**, keeping geometry first-class.
5. **Compute** — run the scoring engine over the unified nationwide table.

The hard part isn't any single source — it's making 19 sources with nothing in common agree on *what district a thing is in* and *what that thing means*, across the whole country.

## Tech Stack

| Layer | Tools |
|-------|-------|
| **Core numerics** | Python, pandas, NumPy |
| **Geospatial** | GeoPandas, Shapely, pyproj |
| **Machine learning** | scikit-learn — StandardScaler, KMeans, silhouette; Spearman hand-implemented in NumPy |
| **Spatial database** | PostGIS |
| **ORM / geo-ORM** | SQLAlchemy 2.0, GeoAlchemy2 |
| **API** | FastAPI |
| **Data sources** | Multiple public-data OpenAPIs (19 connectors) |

## Scale

| Metric | Value |
|--------|-------|
| Python source | ~12,353 LOC across 61 files |
| ETL connectors | 19 public-data sources |
| Geographic coverage | 250 municipalities, nationwide |
| Analytical units | 3,523 administrative districts |
| Composite axes | 4 (Demand, Competition, Accessibility, Cost-efficiency) |
| Auxiliary indicators | 6 |
| Team size | 1 (design + implementation) |

## Technical Highlights

- **Textbook spatial methods, implemented correctly.** 2SFCA, E2SFCA with a Gaussian decay kernel, and the Huff gravity model are implemented to their published formulations — not approximated with ad-hoc distance buckets.
- **Learned weights, not guessed weights.** The four-axis weighting is optimized against real outcome data, so the composite reflects what actually correlates with success.
- **19 sources, one grid.** Nineteen incompatible public APIs standardized and spatially joined onto 3,523 districts nationwide.
- **Geometry as a first-class citizen.** PostGIS + GeoAlchemy2 keep spatial relationships queryable end-to-end.
- **Honest uncertainty engineering.** The model avoids circular logic, treats areas outside data coverage as *neutral* rather than fabricating a score, and explicitly flags low-confidence outputs.

## Validation

- **Weight optimization.** Four-axis weights selected by **grid search** against real outcome targets (observed openings / survival).
- **Cross-validation.** Optimization ran under **5-fold cross-validation** to guard against overfitting.
- **Parameter back-testing.** Spatial parameters — decay σ, catchment cutoff, normalization — chosen by back-testing, not defaulted.
- **Rank-quality checks.** Output rankings audited with **Spearman rank correlation** against reference signals.

**Stated limits (honest by design):** districts outside a source's coverage are held neutral (never imputed); low-confidence outputs are labeled; the engine is decision *support* and does not claim to predict any individual store's success deterministically.

## My Role

Sole designer and engineer, end to end:

- The **spatial methodology** — selecting and correctly implementing 2SFCA / E2SFCA / Huff, and designing the four-axis composite.
- The **data engineering** — all 19 ETL connectors, nationwide district normalization, CRS handling, and PostGIS/GeoAlchemy2 spatial persistence.
- The **calibration science** — grid search, 5-fold cross-validation, and parameter back-testing.
- The **honesty layer** — coverage handling, confidence labeling, and circular-logic prevention.

One person, ~12,353 lines, 3,523 districts, from raw public APIs to a calibrated nationwide scoring engine.
