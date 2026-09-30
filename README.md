# SafeRoute – Your Path, Your Safety

**SafeRoute is designed for India-wide operation.**

SafeRoute is a safety-aware navigation web application. It gets several
**real road routes** between any two places in India from Google, gives each
route an explainable **Safety Score (0–100)** together with a **Data Coverage**
level, and recommends the safest *practical* route. The recommendation weighs
safety against the extra distance and time.

> The Safety Score is a **provisional, relative comparison index**. It is
> **not** an objective probability of being safe and never a guarantee of safety.
> Wherever there is too little safety evidence, SafeRoute says so and does not
> show a score.

## Status

**Stage 0 – Architecture approved.** Environment setup (Stage 0b) is next.
No application code exists yet.

| Document | Purpose |
|----------|---------|
| [PROJECT_PLAN.md](PROJECT_PLAN.md) | Stages, scope, decisions, open items |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System, API, database, scoring/coverage, security, deployment, decision log |
| [DATA_STRATEGY.md](DATA_STRATEGY.md) | Sources, provenance, source adapters, synthetic-data policy, ML data rules |

## Coverage: four different things

| Coverage | Meaning |
|----------|---------|
| Routing coverage | Whether the routing provider (Google) can return real road routes |
| Safety-data coverage | How much legitimate safety evidence exists along a route (High / Medium / Limited / Insufficient) |
| Community-data coverage | How many verified SafeRoute user reports exist along a route (Stage 4+) |
| Development/test coverage | Which sample regions are loaded for development and tests. This is **not** a product restriction. |

A route can be fully routable while its safety evidence is limited. The app
shows both.

## Technology stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + **TypeScript** (Vite), React Router, Axios, Bootstrap 5, Google Maps JavaScript API |
| Backend | Python 3.12, Django 5.2 LTS, Django REST Framework, JWT (simplejwt), GeoDjango |
| Database | PostgreSQL + PostGIS |
| Routing | Google Routes API (server-side), behind a provider abstraction; MVP travel mode: driving |
| Data / ML | pandas, NumPy, GeoPandas, H3, scikit-learn (K-Means, Logistic Regression) |

## Local development

Development runs **natively on Windows**: a Python virtual environment,
PostgreSQL + PostGIS, and GDAL/GEOS from OSGeo4W. Docker is **not** required.
Setup instructions will be added in Stage 0b (`docs/setup-windows.md`).

## Data honesty

- Every record keeps its source, original geographic resolution, data origin,
  collection and source-update timestamps, verification status and
  processing version.
- Street-level crime data is **not** fabricated. Official aggregated statistics
  (e.g. at district level) are never spread onto streets.
- Crime/incident data is used only where a verified geolocated source exists.
- Development/test data is labelled **SYNTHETIC**, kept apart from real
  data, and blocked in production.

## Attribution

Map data and routes © Google. Safety-layer data sources and their licenses
(for example © OpenStreetMap contributors, ODbL) will be listed in the app's
"About the data" page from Stage 2 onwards.
