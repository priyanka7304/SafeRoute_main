# SafeRoute — Architecture

> **SafeRoute is designed for India-wide operation.**
> **Status:** Stage 0 architecture, approved with the owner decisions recorded in PROJECT_PLAN.md §6.
> **Source of truth:** this document together with [PROJECT_PLAN.md](PROJECT_PLAN.md) and [DATA_STRATEGY.md](DATA_STRATEGY.md).
> Any change to an architectural decision is recorded in the [Decision Log](#18-decision-log) before or with the code that implements it.

---

## Table of Contents

0. [Geographic Scope and the Four Kinds of Coverage](#0-geographic-scope-and-the-four-kinds-of-coverage)
1. [System Architecture](#1-system-architecture)
2. [Frontend Architecture](#2-frontend-architecture)
3. [Backend Architecture](#3-backend-architecture)
4. [PostgreSQL Database Design](#4-postgresql-database-design)
5. [Geospatial Data Strategy](#5-geospatial-data-strategy)
6. [REST API Design](#6-rest-api-design)
7. [Authentication Architecture](#7-authentication-architecture)
8. [Dataset Architecture](#8-dataset-architecture) (summary; detail in DATA_STRATEGY.md)
9. [Safety-Score, Coverage and Confidence Architecture](#9-safety-score-coverage-and-confidence-architecture)
10. [ML Architecture](#10-ml-architecture)
11. [Google Maps / Routing Architecture](#11-google-maps--routing-architecture)
12. [Security and Privacy Architecture](#12-security-and-privacy-architecture)
13. [Environment-Variable Strategy](#13-environment-variable-strategy)
14. [Testing Strategy](#14-testing-strategy)
15. [Deployment Architecture](#15-deployment-architecture)
16. [Complete Folder Structure](#16-complete-folder-structure)
17. [Stage-by-Stage Roadmap](#17-stage-by-stage-roadmap)
18. [Decision Log](#18-decision-log)

---

## Non-negotiable invariants

| # | Invariant | How it is enforced |
|---|-----------|------------------|
| I-1 | Every route shown to a user follows real roads. Route geometry comes only from the routing provider's polyline. | There is no geometry-construction code in the backend. On provider failure the API returns an error, never a straight line. A test asserts this. |
| I-2 | Data is never presented as real unless its provenance is known. | Every record carries `source`, `data_origin`, `verification_status` and timestamps (DATA_STRATEGY §4). |
| I-3 | SYNTHETIC data is only for development and testing. It is labelled everywhere and never mixed with real data. | Default managers exclude it, production settings refuse to start with it enabled, and the UI shows a banner (DATA_STRATEGY §7). |
| I-4 | Secrets never appear in source control. | `.env` is gitignored, only `.env.example` is committed, and CI runs secret scanning. |
| I-5 | **Aggregated statistics are never turned into street-level points.** District/state/city counts are never spread onto streets or given invented coordinates. | Aggregates live in a separate table with no point geometry (§4.2). A test asserts that no incident row is ever derived from an aggregate row. |
| I-6 | Missing evidence is reported, never filled in. Unavailable factors are **omitted**, not defaulted. When evidence is insufficient, **no numeric score is shown**. | The coverage model (§9.4) and display rules (§9.6). |
| I-7 | The Safety Score is a **provisional, relative index**. It is never presented as an objective probability of being safe. | API field `methodology_status: "PROVISIONAL"`, plus UI copy review. |
| I-8 | Scores are reproducible. Each response records the scoring-profile version, the model version (if any) and the data snapshot. | API response and `route_result`. |
| I-9 | **No city- or state-specific assumptions in core code.** Country, bounds, timezone and data sources come from configuration and data. | Code review, plus tests that exercise several regions across India, including cross-state routes. |
| I-10 | No surveillance features: no CCTV analytics, no facial recognition, no tracking of third parties. | Scope rule (PROJECT_PLAN §1). |
| I-11 | Coverage (how much evidence exists) and confidence (statistical/model certainty) are separate concepts with separate fields. | API contract (§6.3), tests. |

---

## 0. Geographic Scope and the Four Kinds of Coverage

**Production scope: all of India.** Users can search between any two places in India that the routing provider covers, for example Delhi → Gurugram, Mumbai → Navi Mumbai, Chandigarh → Mohali, or Bengaluru → Electronic City. Routes that cross state or UT boundaries are normal.

The deployment country is itself configuration (`SAFEROUTE_COUNTRY_CODE=IN`), so there is no hidden assumption that could block expansion later.

SafeRoute keeps four kinds of coverage apart and never lets one stand in for another:

| # | Coverage type | Question it answers | Determined by | Shown to the user as |
|---|---------------|--------------------|---------------|----------------------|
| 1 | **Routing coverage** | Can we get real road routes here? | The routing provider (Google Routes API) returns routes or not | Routes appear, or "No route found" |
| 2 | **Safety-data coverage** | How much legitimate safety evidence exists along this route? | The coverage model (§9.4): availability, geographic resolution, recency and volume of each factor's data along the route | Coverage level badge: **High / Medium / Limited / Insufficient**, with per-factor detail |
| 3 | **Community-data coverage** | How many credible SafeRoute user reports exist along this route? | Count and recency of *verified* community reports in the route corridor (Stage 4+) | Part of the coverage detail ("3 verified community reports in the last 90 days") |
| 4 | **Development/test dataset coverage** | Which areas are loaded in *development and CI* data? | Configured development test regions (DATA_STRATEGY §9) | Never shown in production. It is a development convenience only, **not a product restriction**. |

**A route can have full routing coverage and insufficient safety-data coverage at the same time.** That is expected in many parts of India, and the product must handle it honestly (§9.6).

---

## 1. System Architecture

### 1.1 Overview

SafeRoute is a **modular monolith**: one Django project with bounded-context apps, plus a separate React + TypeScript single-page app. Background work runs in Celery workers (from Stage 2). Real-time features use Django Channels (from Stage 5). See ADR-001.

```
                          ┌────────────────────────────────────────┐
                          │            Browser (SPA)               │
                          │ React + TS + Router + Axios + Bootstrap│
                          │ Google Maps JS API (display, Places    │
                          │ Autocomplete restricted to India)      │
                          └───────────────┬────────────────────────┘
                                          │ HTTPS /api/v1/*  (public + JWT)
                                          │ WSS   /ws/*      (Stage 5)
                          ┌───────────────▼────────────────────────┐
                          │     Reverse proxy (nginx / platform)   │
                          └───────────────┬────────────────────────┘
        ┌─────────────────────────────────▼──────────────────────────────────┐
        │                     Django + DRF                                   │
        │ accounts  geo  routing  safety  datasets  ml  reports*  emergency* │
        │              │        │                                            │
        │              │        ├─ Scoring engine ─┐                         │
        │              │        └─ Coverage engine ┼─► corridor/bbox/radius  │
        │              │                           │   PostGIS queries       │
        │              └─ RouteProvider ──► Google Routes API (server key)   │
        └───────┬───────────────────────┬───────────────────────┬────────────┘
     ┌──────────▼─────────┐   ┌─────────▼────────┐   ┌──────────▼───────────┐
     │ PostgreSQL+PostGIS │   │  Redis           │   │ Celery workers/beat  │
     │ system of record   │   │ cache, broker,   │   │ source adapters,     │
     │ (India-wide)       │   │ channels layer*  │   │ grid builds, ML,     │
     └────────────────────┘   └──────────────────┘   │ SOS fan-out*         │
                                                     └──────────┬───────────┘
                                        ┌───────────────────────▼──────────┐
                                        │ Source adapters (DATA_STRATEGY): │
                                        │ government, osm, infrastructure, │
                                        │ lighting, community              │
                                        └──────────────────────────────────┘
   * = later stage
```

### 1.2 Request flow: "find safest route" (public, no login)

1. The user picks an origin and destination with Places Autocomplete, restricted to India. The frontend sends `place_id`s or coordinates, the travel mode (MVP: `DRIVING`) and the departure time to `POST /api/v1/routes/safe-routes/`.
2. The backend validates the input: an allowed travel mode, and endpoints inside the configured country (§5.2).
3. The **`RouteProvider`** (Google implementation) requests alternatives. It returns up to about 3 real road routes, each with a polyline, distance and duration.
4. For each route, the backend:
   1. decodes the polyline;
   2. resamples it *along the road geometry* (adaptive spacing, §5.4);
   3. runs **corridor queries** in PostGIS for the POIs, incidents and reports relevant to that corridor only;
   4. maps the samples to H3 cells and reads the cell features.
5. The **scoring engine** computes a per-segment and per-route risk from the factors that are available, and **omits unavailable factors**.
6. The **coverage engine** computes safety-data coverage and community-data coverage for each route. The **confidence** block is filled only when a statistical model is used (Stage 3+).
7. The **recommender** compares routes like-for-like (§9.7) and picks the safest *practical* route, or states that no meaningful safety comparison is possible.
8. The frontend draws all polylines on the Google Map, highlights the recommendation, and shows score + coverage badges and factor explanations.

### 1.3 Quality attributes

| Attribute | Target |
|-----------|--------|
| Scoring + coverage latency (excluding Google), urban route | p95 < 300 ms for 3 routes |
| Long inter-city route (e.g. > 300 km) | p95 < 1.5 s (adaptive sampling, §5.4) |
| Browser memory | The browser never loads bulk safety data. It only receives the route results and bbox-limited map layers. |
| Graceful degradation | If safety data is missing, routes are still returned and the coverage is stated honestly |
| Extensibility | Adding a new state/city dataset means a new source adapter plus a manifest. The safety engine does not change. |

---

## 2. Frontend Architecture

### 2.1 Stack

| Concern | Choice |
|--------|--------|
| Build | **Vite** (Create React App is deprecated) |
| Language | **TypeScript**, `strict: true`. ESLint `@typescript-eslint/no-explicit-any: error`. Any exception needs a justified inline disable. |
| UI | React 19, **Bootstrap 5.3** + react-bootstrap |
| Routing | React Router (`createBrowserRouter`) |
| HTTP | Axios, with one configured instance and interceptors |
| Maps | Google Maps JavaScript API via `@vis.gl/react-google-maps`. The version is verified at Stage 1. |
| Server state | TanStack Query |
| Forms | react-hook-form + zod |
| Tests | Vitest, React Testing Library, MSW; Playwright for E2E |

### 2.2 Structure

```
src/
  app/          # shell, providers, router
  api/          # axios instance, interceptors, typed endpoint functions
  types/        # domain & API types (§2.6)
  auth/         # AuthContext, ProtectedRoute, login/register
  features/
    route-planner/   # Stage 1 (public)
    safety-layer/    # Stage 3 (bbox-limited overlays)
    reports/         # Stage 4 (login)
    emergency/       # Stage 5/6 (login)
  components/   # RiskBadge, CoverageBadge, ScoreGauge, DataNotice, ...
  maps/         # map provider, polyline rendering, colour scales
  hooks/ utils/ styles/ config/
```

Rules:
- **No safety logic runs in the browser.** The frontend only displays backend results.
- **No city names, coordinates or regions are hard-coded.** The initial map viewport comes from the backend config endpoint (`/api/v1/config/`) or the user's location with consent. The default is a country-level view.
- Map layers are always requested for the **current map bounds** and zoom (§6.2). Nothing is preloaded.

### 2.3 Pages

| Path | Page | Access | Stage |
|------|------|--------|-------|
| `/` | Route planner: map, search, route comparison, safety info | **Public** | 1 |
| `/login`, `/register` | Auth | Public | 1 |
| `/saved` | Saved routes / search history (opt-in) | Login | 1 (optional) |
| `/settings` | Preferences (e.g. default travel mode, practicality limits) | Login | 1 (optional) |
| `/about/data` | Data sources, licences, coverage explanation | Public | 2 |
| `/map/safety` | Bbox-limited hotspot and coverage overlays | Public | 3 |
| `/reports`, `/reports/new` | Community reports (viewing may be public; submitting needs login) | Login to submit | 4 |
| `/emergency`, `/contacts`, `/trip/:id` | SOS, contacts, trips | Login | 5 |
| `/guardian/:token` | Guardian live view (scoped share token) | Token | 5 |

### 2.4 Map rendering

- All alternatives are drawn at once. The recommended route is thicker and drawn on top. Colours follow risk category, with a dash pattern and text label so colour is never the only signal.
- A route whose coverage is **Insufficient** is drawn in a **neutral grey**, never green. Grey means "unknown", not "safe".
- Per-segment colouring uses the `segments` array. Segments with no evidence are grey.

### 2.5 Auth handling

The access token is held in memory only. The refresh token is in an httpOnly, Secure, SameSite=Strict cookie. A single-flight refresh interceptor handles expiry. Public pages work without any token.

### 2.6 Core TypeScript types (sketch; finalised in Stage 1 from the OpenAPI schema)

```ts
type TravelMode = 'DRIVING' | 'WALKING' | 'TWO_WHEELER' | 'BICYCLING';
type RiskCategory = 'LOW' | 'MODERATE' | 'ELEVATED' | 'HIGH';
type CoverageLevel = 'HIGH' | 'MEDIUM' | 'LIMITED' | 'INSUFFICIENT';
type ScoreStatus = 'SCORED' | 'SCORED_LIMITED_EVIDENCE' | 'NOT_SCORED';
type DataOrigin = 'VERIFIED_SOURCE' | 'DERIVED' | 'COMMUNITY' | 'SYNTHETIC';

interface LatLng { lat: number; lng: number }
interface LocationInput { placeId?: string; point?: LatLng }

interface FactorResult {
  key: string; label: string;
  status: 'AVAILABLE' | 'UNAVAILABLE' | 'NOT_APPLICABLE';
  risk: number | null;             // 0..1, null when unavailable
  evidenceQuality: number | null;  // 0..1, see §9.4
  explanation: string;
}
interface SafetyCoverage {
  level: CoverageLevel; index: number;
  uncoveredLengthShare: number;
  byFactor: { key: string; availability: number; resolution: number; recency: number; volume: number }[];
  community: { verifiedReports: number; windowDays: number } | null;
}
interface ModelConfidence {
  method: 'NONE' | 'BOOTSTRAP_INTERVAL';
  interval: [number, number] | null;
  applicability: 'IN_DOMAIN' | 'OUT_OF_DOMAIN' | 'NOT_APPLICABLE';
}
interface RouteSafety {
  status: ScoreStatus; score: number | null; category: RiskCategory | null;
  factors: FactorResult[]; segments: { fromIdx: number; toIdx: number; risk: number | null }[];
}
interface RouteOption {
  routeId: string; encodedPolyline: string; distanceM: number; durationS: number;
  providerLabels: string[]; safety: RouteSafety; coverage: SafetyCoverage;
  confidence: ModelConfidence; isRecommended: boolean; isPractical: boolean;
}
interface SafeRoutesResponse {
  requestId: string; travelMode: TravelMode; departureTime: string; timeBucket: string;
  routes: RouteOption[];
  recommendation: { routeId: string | null; basis: 'SAFETY' | 'NO_MEANINGFUL_DIFFERENCE' | 'INSUFFICIENT_EVIDENCE'; reason: string };
  scoring: { profileVersion: string; modelVersion: string | null; snapshotId: string | null; methodologyStatus: 'PROVISIONAL' };
  dataNotice: { syntheticDataUsed: boolean; sources: string[] };
}
interface User { id: string; email: string; fullName: string }
// Report, EmergencyContact, Trip, MapLayerFeature, etc. are defined in their stages.
```

Snake_case JSON is converted to camelCase in the API layer, in one place.

---

## 3. Backend Architecture

### 3.1 Stack

| Concern | Choice |
|--------|--------|
| Language | Python **3.12** (installed on the dev machine; Python 3.8 is end-of-life) |
| Framework | **Django 5.2 LTS** + Django REST Framework |
| Geospatial | **GeoDjango** on **PostGIS** |
| Auth | `djangorestframework-simplejwt` + token blacklist |
| API schema | `drf-spectacular` |
| Config | `django-environ` |
| Jobs | Celery + Redis (Stage 2) |
| Real-time | Django Channels (Stage 5) |
| HTTP client | `httpx` |
| Geo/data | `shapely` 2, `pyproj`, `h3` (v4), `polyline`, `pandas`, `numpy`, `geopandas`, `pyosmium` (Stage 2), `rasterio` (Stage 2), `scikit-learn` (Stage 3) |

### 3.2 Django apps

| App | Responsibility | Stage |
|-----|----------------|-------|
| `core` | Base models, health checks, exception handler, pagination, public config endpoint | 0/1 |
| `accounts` | Custom `User` (email login), profile, preferences, JWT views | 1 |
| `geo` | Country config, **administrative hierarchy** (country → state/UT → district → city/locality), H3 utilities, projection helpers, admin tagging service | 1 (country config) → 2 (hierarchy data) |
| `routing` | `RouteProvider` abstraction, Google implementation, travel-mode registry, safe-routes endpoint, saved routes | 1 |
| `safety` | Factor providers, scoring engine, **coverage engine**, recommender, scoring profiles, H3 feature grid | 1 → 3 → 6 |
| `datasets` | Consolidated SafeRoute dataset: sources, ingestion runs, records, aggregates; **source adapters**; pipeline | 1 (models + synthetic) → 2 |
| `ml` | Features, K-Means, Logistic Regression, registry, inference, applicability domain | 3 → 6 |
| `reports` | Community reports, credibility, moderation | 4 |
| `emergency` | Contacts, SOS, notifier, discreet emergency interface | 5 → 6 |
| `tracking` | Trips, pings, check-ins, guardian sessions, deviation detection | 5 → 6 |

### 3.3 Layering inside each app

`views` (HTTP only) → `serializers` → `services/` (business logic) → `selectors.py` (read queries) → `models`. External calls go through `providers/` or `sources/` adapters. Celery `tasks.py` files are thin wrappers around services.

### 3.4 Extension points (so later stages need no core rewrite)

| Interface | Purpose | Adding something new means… |
|-----------|---------|-----------------------------|
| `RouteProvider` (`routing/providers/base.py`) | `compute_alternatives(origin, dest, mode, departure) -> list[ProviderRoute]`, `supported_modes()` | a new provider class. The frontend is unaffected. |
| `TravelMode` registry | Maps internal modes to provider values and to a scoring profile per mode | enabling a mode in config and adding its scoring profile |
| `FactorProvider` (`safety/factors/base.py`) | `key`, `compute(corridor, time_bucket) -> FactorResult` including evidence quality | one class plus a profile entry |
| `RiskModel` | Rule-based (Stage 1) and ML (Stage 3) models behind the same interface | registering a model |
| `SourceAdapter` (`datasets/sources/base.py`) | `fetch`, `parse`, `normalize` into the SafeRoute schema, `coverage_extent` | one adapter plus a manifest. **The core engine does not change.** |
| Domain events | `route_computed`, `trip_started`, `sos_triggered`, `report_verified` | subscribing in the new app |
| `Notifier` | SMS/email/push | a provider implementation |

---

## 4. PostgreSQL Database Design

PostgreSQL 16+ with **PostGIS**. Geometry is stored in **SRID 4326**. Distances use `geography` or a local UTM projection chosen per computation (§5.3). All geometries have GIST indexes.

### 4.1 Entity overview

```
geo_admin_area (self-referencing hierarchy: COUNTRY → STATE_UT → DISTRICT → CITY → LOCALITY)
      ▲ (optional FKs, derived by point-in-polygon)
      │
datasets_data_source ─< datasets_ingestion_run ─┬─< datasets_incident          (geolocated only)
         │                                      ├─< datasets_poi
         │                                      ├─< datasets_road_feature
         │                                      ├─< datasets_lighting_observation
         │                                      └─< datasets_aggregate_statistic (NO point geometry)
         └─< datasets_source_coverage (extent where the source claims completeness)

datasets_dataset_snapshot ─< safety_cell_feature (sparse H3 grid)
safety_scoring_profile     ml_model ─< ml_model_metric    ml_hotspot
accounts_user ─< routing_saved_route ─< routing_route_result
              ─< reports_report (Stage 4) ─< reports_report_vote
              ─< emergency_contact / emergency_sos_event (Stage 5)
              ─< tracking_trip ─< tracking_location_ping (partitioned)
```

### 4.2 Tables

**`geo_admin_area`**: the India administrative hierarchy
| column | type | notes |
|---|---|---|
| id | bigint PK | |
| level | enum | `COUNTRY`, `STATE_UT`, `DISTRICT`, `SUBDISTRICT`, `CITY`, `LOCALITY` |
| name, name_local | varchar | |
| parent_id | FK self, null | |
| official_code | varchar null | e.g. the Local Government Directory (LGD) code, if the boundary source provides it (verified in Stage 2) |
| geom | MultiPolygon(4326) null | from a documented boundary source; used **internally** for tagging and aggregates only (see §12.6) |
| source_id | FK | provenance of the boundary |
| valid_from / valid_to | date null | districts get created and split over time |

**`datasets_data_source`**
| column | type | notes |
|---|---|---|
| id, slug | uuid, varchar UNIQUE | |
| name, publisher, url, license, attribution_text | | |
| publisher_type | enum | `GOVERNMENT`, `POLICE`, `OPEN_COMMUNITY` (e.g. OSM), `SCIENTIFIC` (e.g. NASA), `SAFEROUTE_USERS`, `SAFEROUTE_SYNTHETIC` |
| adapter_category | enum | `government`, `osm`, `infrastructure`, `lighting`, `community`, `synthetic` |
| native_geo_resolution | enum | `POINT`, `SNAPPED_POINT`, `LINE`, `RASTER`, `LOCALITY`, `CITY`, `DISTRICT`, `STATE`, `COUNTRY` |
| refresh_cadence | interval null | expected update frequency, used by the recency score |
| notes | text | known limitations |

**`datasets_source_coverage`**: *where* a source claims to have data. This is essential for telling "no incidents recorded" apart from "no data here".
| id | source_id | admin_area_id null | geom MultiPolygon null | factor_keys varchar[] | valid_from, valid_to | completeness_note |

**`datasets_ingestion_run`**: source, started/finished, status, raw_uri, raw_sha256, upstream_updated_at, row counts (fetched/valid/rejected/duplicate/inserted/updated), rejection_report jsonb, **processing_version** (pipeline code version + taxonomy version).

**Common provenance columns** on every consolidated record table (`incident`, `poi`, `road_feature`, `lighting_observation`, `aggregate_statistic`):
| column | notes |
|---|---|
| source_id, ingestion_run_id, source_record_id | `UNIQUE (source_id, source_record_id)` |
| **data_origin** | `VERIFIED_SOURCE`, `DERIVED`, `COMMUNITY`, `SYNTHETIC` (definitions in DATA_STRATEGY §4.2) |
| **verification_status** | `VERIFIED`, `UNVERIFIED`, `PENDING_REVIEW`, `REJECTED`, `NOT_APPLICABLE` |
| **original_geo_resolution** | as published by the source (it may be coarser than the source default) |
| location_precision_m | null when not a point |
| **source_timestamp** | when the event happened or the record was valid, per the source |
| **source_updated_at** | the source's own "last updated", when published |
| **retrieved_at** | when SafeRoute collected it |
| **processing_version** | pipeline/taxonomy version that produced the row |
| state_id, district_id, city_id | FK `geo_admin_area`, nullable, **derived** by point-in-polygon (for filtering and reporting, not scoring) |
| raw_hash | change detection |

**`datasets_incident`**: geolocated incidents only (`geom Point NOT NULL`)
plus `category`, `source_category`, `severity` (provisional taxonomy), `occurred_at`, `occurred_at_precision`, `h3_r9`, `dedupe_group_id`.

**`datasets_poi`**: `kind` (`POLICE`, `HOSPITAL`, `HOSPITAL_EMERGENCY`, `CLINIC`, `FIRE_STATION`, `PHARMACY`, `TRANSIT_STOP`, `ACTIVITY_VENUE`, `STREET_LAMP`, …), `name`, `geom Point`, `attributes jsonb` (e.g. `opening_hours`, `emergency`).

**`datasets_road_feature`**: `geom LineString`, `road_class`, `lit` (`YES`/`NO`/`LIMITED`/`UNKNOWN`), `sidewalk`, `lanes`, `maxspeed`, tags jsonb. Road characteristics come from OSM.

**`datasets_lighting_observation`**: per H3 cell and period aggregation of raster lighting proxies (`h3_r8`, `period`, `radiance_mean`, `radiance_p90`, `quality_flag`).

**`datasets_aggregate_statistic`**: **administrative-level statistics, with no point geometry**
| id | source_id | admin_area_id (FK, NOT NULL) | admin_level | indicator (e.g. `IPC_CRIMES_TOTAL`) | indicator_label (verbatim) | period_start/end | value | unit | population_denominator null | + provenance columns |
There is deliberately no `geom` column, so aggregates **cannot** be mistaken for street-level data (I-5).

**`datasets_dataset_snapshot`**: frozen set of ingestion runs, `includes_synthetic`, profiling report, quality-gate results.

**`safety_cell_feature`**: the **sparse** H3 feature grid (§5.5)
| column | notes |
|---|---|
| h3_r9 (bigint), snapshot_id | PK |
| h3_r5 | coarse parent, for partitioning and bulk filtering |
| static features | `lit_road_share`, `lit_tag_coverage`, `lamp_density`, `activity_density`, `road_class_mix`, `lighting_radiance` (all nullable) |
| incident features | `inc_recency_weighted`, `inc_count_12m`, `inc_bucket_profile` jsonb (nullable) |
| evidence | per-factor `availability/resolution/recency/volume` jsonb (§9.4) |
| model outputs | `model_risk_by_bucket` jsonb, `hotspot_exposure` (Stage 3, nullable) |

Rows exist only for cells containing roads or data. Cells that don't appear are treated as "no evidence", never as "safe".

**`safety_scoring_profile`**: `version`, `travel_mode`, `weights`, `params`, `status` (`PROVISIONAL`/`REVIEWED`), `is_active`, `notes`.

**`routing_saved_route`** (login, opt-in): user, origin/destination as supplied by the user (place_id / point), travel_mode, created_at.
**`routing_route_result`**: saved_route, distance_m, duration_s, score, status, coverage_level, profile/model/snapshot versions, factors jsonb. **No provider polyline is persisted** (§11.5).

**`ml_model`**, **`ml_model_metric`**, **`ml_hotspot`**: as in §10.4. Every model records its **analysis region** (admin areas or polygon) and **applicability domain**.

**Stage 4–6 tables:** `reports_report` (with `data_origin=COMMUNITY`, `verification_status`, credibility, expiry, geom stored fuzzed), `reports_report_vote`, `reports_moderation_action`, `emergency_contact`, `emergency_sos_event`, `tracking_trip`, `tracking_location_ping` (time-partitioned, retention), `tracking_checkin`, `tracking_guardian_access`.

### 4.3 Scale and indexing (India-wide)

- GIST on all geometry. B-tree on `h3_r9`, `h3_r5`, `occurred_at`, admin FKs.
- Partial indexes `WHERE data_origin <> 'SYNTHETIC'` on the record tables.
- Large tables (`incident`, `location_ping`) are partitioned: incidents by `occurred_at` range once volume warrants it, pings by month.
- `safety_cell_feature` can be list-partitioned by `h3_r5` parent or by state if it grows beyond comfortable single-table size. This is decided on measured volume in Stage 2 or 6.
- Ingestion is idempotent through `UNIQUE (source_id, source_record_id)` upserts.

---

## 5. Geospatial Data Strategy

### 5.1 Principles

- **Scoring uses coordinates and spatial queries**, not administrative boundaries. Admin areas are for tagging, filtering, aggregates and reporting.
- **No region constants in code.** Country config (code, bounds, timezone, projection strategy) is data or configuration.
- The browser only ever receives data for the **requested route corridor** or the **current map bounds**.

### 5.2 Country config

`geo` app, `CountryConfig` loaded from `data/countries/in.yaml`, which holds the ISO code, timezone (`Asia/Kolkata`), a coarse bounding box for early validation, and the boundary source reference.
- Stage 1 validates origin/destination against the coarse bbox. Places Autocomplete is also restricted to region code `in`.
- From Stage 2, once admin boundaries are loaded, validation uses the country polygon.
- Routes that leave India (e.g. near borders) are not specially handled in the MVP. Safety evidence outside India is simply "no coverage".

### 5.3 CRS and projections

- Storage: EPSG:4326.
- Distances: PostGIS `geography`.
- Local metric work (buffering, resampling, K-Means input) uses the **UTM zone computed from the data's centroid**. India spans several UTM zones (roughly 42N–47N), so the zone is never fixed. K-Means is never run on raw degrees.

### 5.4 Route processing

- Decode the provider polyline and resample *along it*. Spacing is **adaptive**: 50 m for routes up to 25 km, growing linearly to a maximum of 500 m, with a cap of about 3,000 samples per route. This keeps Delhi → Mumbai-length routes tractable.
- **Corridor:** the route line buffered by a radius set per factor (e.g. 150 m for incidents and reports, 5 km for the police/hospital search). This is one PostGIS query per factor family, using `ST_DWithin` on geography with GIST.

### 5.5 H3 feature grid (sparse)

- **H3 resolution 9** (≈ 0.1 km² cells) for local features. Resolution 8 is used for coarse raster proxies (VIIRS ≈ 500 m).
- The grid is **sparse**: only cells with road features or source data get rows. A dense India grid would be about 3.3 M km² ÷ 0.1 km² ≈ 30 M cells, and most of them are irrelevant. The sparse approach stores road-bearing cells only.
- Time-of-day dependence is kept in compact per-cell profiles (jsonb by time bucket), not in one row per bucket.
- **Proximity features** (police, hospital) are computed **at request time** from corridor POIs, not precomputed across India. That is cheap because POIs are few and indexed.

### 5.6 Query patterns supported

| Pattern | Used by | PostGIS |
|--------|---------|---------|
| Route corridor | scoring, coverage | `ST_DWithin(geom::geography, :route_line::geography, :r)` |
| Bounding box | map layers | `geom && ST_MakeEnvelope(...)` with a max area and zoom-dependent aggregation (coarser H3 at low zoom) |
| Nearby radius | point safety, SOS nearest help | `ST_DWithin` + KNN `ORDER BY geom <-> :pt LIMIT n` |
| Point-in-polygon | admin tagging at ingestion | `ST_Contains` on `geo_admin_area` |

---

## 6. REST API Design

### 6.1 Conventions

- `/api/v1/`, JSON, snake_case, ISO-8601 UTC, coordinates as `{lat, lng}`.
- Errors in an RFC 7807-style body. OpenAPI 3 via drf-spectacular.
- Throttles: separate anonymous and authenticated scopes. There is also a **global daily cap on routing-provider calls** to protect billing (§12).

### 6.2 Endpoints

| Method | Path | Access | Stage | Purpose |
|--------|------|--------|-------|---------|
| GET | `/health/`, `/health/ready/` | public | 0 | Liveness / readiness |
| GET | `/config/` | public | 1 | Public client config: enabled travel modes, country, default viewport, feature flags |
| POST | `/auth/register/`, `/auth/token/`, `/auth/token/refresh/`, `/auth/logout/` | public/cookie | 1 | Auth |
| GET/PATCH | `/auth/me/` | login | 1 | Profile and preferences |
| POST | `/routes/safe-routes/` | **public** (anon-throttled) | 1 | **Core:** alternatives + safety + coverage + recommendation |
| GET/POST/DELETE | `/routes/saved/` | login | 1 (opt) | Saved routes |
| GET | `/safety/point/?lat=&lng=&time=` | public | 3 | Nearby-radius safety and coverage |
| GET | `/safety/layers/cells/?bbox=&zoom=&time_bucket=` | public | 3 | Bbox-limited grid (GeoJSON), zoom-aggregated |
| GET | `/safety/layers/hotspots/?bbox=` | public | 3 | Hotspots within the bbox |
| GET | `/safety/layers/coverage/?bbox=&zoom=` | public | 2 | Safety-data coverage map |
| GET | `/data/sources/` | public | 2 | Source registry and attributions |
| GET | `/data/areas/{id}/statistics/` | public | 2 | Aggregate statistics for an admin area, **labelled with their resolution** |
| GET | `/admin-api/ingestion-runs/`, `/admin-api/models/` | staff | 2/3 | Ops |
| … | `/reports/…` | view public / submit login | 4 | Community reports |
| … | `/emergency/…`, `/trips/…`, `WS /ws/trips/{id}/` | login / token | 5–6 | Emergency, guardian |

### 6.3 Core contract: `POST /api/v1/routes/safe-routes/`

Request:
```json
{
  "origin":      { "place_id": "…" },
  "destination": { "lat": 0.0, "lng": 0.0 },
  "travel_mode": "DRIVING",
  "departure_time": "2026-10-01T16:00:00Z",
  "max_detour_ratio": 1.3
}
```

Response `200` (abridged):
```json
{
  "request_id": "uuid",
  "travel_mode": "DRIVING",
  "time_bucket": "NIGHT",
  "routes": [
    {
      "route_id": "r0",
      "encoded_polyline": "…",
      "distance_m": 18250,
      "duration_s": 2280,
      "provider_labels": ["DEFAULT_ROUTE"],
      "safety": {
        "status": "SCORED",
        "score": 78,
        "category": "MODERATE",
        "factors": [
          { "key": "lighting", "status": "AVAILABLE", "risk": 0.41, "evidence_quality": 0.55,
            "explanation": "Lighting inferred from map tags on 46% of route length" },
          { "key": "verified_incident_exposure", "status": "UNAVAILABLE", "risk": null,
            "evidence_quality": 0.0, "explanation": "No geolocated incident source covers this area" }
        ],
        "segments": [ { "from_idx": 0, "to_idx": 22, "risk": 0.31 } ]
      },
      "coverage": {
        "level": "LIMITED",
        "index": 0.34,
        "uncovered_length_share": 0.12,
        "by_factor": [ { "key": "lighting", "availability": 1.0, "resolution": 0.9, "recency": 1.0, "volume": 0.6 } ],
        "community": { "verified_reports": 0, "window_days": 90 }
      },
      "confidence": { "method": "NONE", "interval": null, "applicability": "NOT_APPLICABLE" },
      "is_practical": true,
      "is_recommended": true
    }
  ],
  "recommendation": {
    "route_id": "r0",
    "basis": "SAFETY",
    "reason": "Higher score on shared factors (lighting, police proximity); coverage is Limited"
  },
  "scoring": { "profile_version": "1.0.0-DRIVING", "model_version": null, "snapshot_id": "…", "methodology_status": "PROVISIONAL" },
  "data_notice": { "synthetic_data_used": false, "sources": ["osm-india-…"] }
}
```

Errors: `400` validation (including a disabled travel mode or a point outside the country), `422` no route found, `502` provider error (**no fallback geometry**), `503` routing budget cap reached or provider disabled, `429` throttled.

---

## 7. Authentication Architecture

| Aspect | Decision |
|------|----------|
| Public (no login) | Map, place search, **route search and comparison**, safety and coverage information, data source pages, map layers |
| Login required | Saved routes and preferences, submitting community reports, emergency contacts, SOS, trips, Guardian Mode, any private data |
| Mechanism | JWT (`simplejwt`). Access token 15 min, in memory only. Refresh token 7 days, rotated and blacklisted, in an httpOnly Secure SameSite=Strict cookie scoped to `/api/v1/auth/`. |
| CSRF | Required on the cookie-based refresh and logout endpoints |
| Passwords | Argon2 + Django validators |
| User model | Custom `accounts.User` (email as username) in the first migration |
| Anonymous abuse | Per-IP throttle on safe-routes, a global daily routing cap, and optional CAPTCHA/Turnstile evaluated in Stage 7 if abuse appears |
| Brute force | Scoped throttle + lockout (`django-axes`, evaluated in Stage 1) |
| Authorization | `IsAuthenticated`, object-level `IsOwner`, `IsStaff`, and (Stage 5) `IsGuardianOfTrip` with expiring, revocable grants |

---

## 8. Dataset Architecture

The full specification is in **[DATA_STRATEGY.md](DATA_STRATEGY.md)**. In summary:

- SafeRoute builds its own **consolidated dataset**. It combines legitimate sources into one SafeRoute schema and keeps full provenance on every record (source, original geographic resolution, data origin, collection timestamp, source update timestamp, verification status, processing version).
- **Source adapters** live in `backend/apps/datasets/sources/{government,osm,infrastructure,lighting,community,synthetic}/`. Each adapter has a committed manifest in `data/sources/<slug>.yaml`. Adding a new state or city dataset means one new adapter (or just a new manifest for an existing adapter type).
- **Geolocated records** (incidents, POIs, road features, lighting cells) and **administrative aggregates** are stored in separate tables. Aggregates are never converted to points (I-5).
- The pipeline runs: fetch → raw zone (immutable, checksummed) → validate → normalise → deduplicate → admin tagging → load (idempotent upsert) → snapshot → build the sparse feature grid.
- **Ingestion scope is configuration.** Development uses test-region clips. Production uses the whole country. The code is the same in both.

---

## 9. Safety-Score, Coverage and Confidence Architecture

### 9.1 Three separate outputs per route

| Output | Meaning | Available from |
|--------|---------|----------------|
| **Safety Score** (0–100, or null) | A provisional, relative index of risk exposure along the whole route, computed from the factors that have evidence. Higher means lower estimated relative risk. | Stage 1 (synthetic dev data), Stage 2 (real data) |
| **Safety-data coverage** (level + index) | How much legitimate evidence supports the score: availability, resolution, recency and volume, per factor, along the route | Stage 1 (methodology), Stage 2 (real) |
| **Model confidence** | Statistical certainty of an ML prediction, and whether the route is inside the model's applicability domain | Stage 3+. Before that, `method: "NONE"`. It is **never faked**. |

A route can have a high score with limited coverage. The UI always shows the two together.

### 9.2 Factors

All weights are **provisional** (see §9.5). A factor with no evidence on a segment is **omitted** for that segment. It is never set to a default value.

| Key | Evidence source | Risk normalisation (provisional) | Stage |
|-----|-----------------|----------------------------------|-------|
| `verified_incident_exposure` | Geolocated incidents with `data_origin=VERIFIED_SOURCE`, severity-weighted, recency-decayed (τ = 180 days) | Empirical percentile **within the source's coverage extent** (comparisons are only made inside one source's area) | 2 (only where a source exists) |
| `hotspot_exposure` | K-Means hotspots within an analysis region | intensity-weighted membership/distance | 3 |
| `model_risk` | Logistic Regression probability (in-domain only) | calibrated probability → percentile | 3 |
| `community_reports` | **Verified** community reports, credibility-weighted, short decay | percentile, weight capped | 4 |
| `police_proximity` | police POIs in the corridor | `min(1, d / 3 km)` | 1 |
| `hospital_proximity` | hospitals with emergency services, else any hospital | `min(1, d / 5 km)` | 1 |
| `lighting` | OSM `lit` tags, street-lamp density, VIIRS proxy (§DATA_STRATEGY 5.4) | `1 − lighting_index`. Full weight at night, 0.25× in daylight. | 1 (synthetic) / 2 |
| `activity` | density of OSM activity venues and transit stops (a **proxy** for people nearby) | `1 − percentile` | 1 (synthetic) / 2 |
| `road_characteristics` | OSM road class, sidewalk presence (for future walking), lanes | rule table per travel mode | 2 |
| time of day | not a separate factor. It selects the time bucket and modulates lighting, activity and incident profiles | Stage 1 basic, Stage 6 improved | 1 → 6 |

**Aggregated official statistics** (e.g. district-level counts) are **not a street-level factor**. They are stored and can be shown as labelled "area context". Using them in the score is **disabled by default**, and would only be enabled through a separately documented, resolution-appropriate method after Stage 2 analysis (DATA_STRATEGY §6).

### 9.3 Aggregation

1. Cell/segment risk: `R = Σ_f w_f · r_f / Σ_{f available} w_f`, over available factors only.
2. Route risk: `R_route = α · length-weighted mean(R_seg) + (1 − α) · P90(R_seg)`, with α = 0.7 (provisional). Segments with no available factor are excluded from the risk calculation and counted in `uncovered_length_share`.
3. `score = round(100 · (1 − R_route))`, subject to the display rules in §9.6.
4. Categories: 80–100 `LOW`, 60–79 `MODERATE`, 40–59 `ELEVATED`, 0–39 `HIGH` (provisional cut-offs).

### 9.4 Safety-data coverage methodology

For each factor *f* and route sample *s*, the **evidence quality** is:

```
q(f,s) = A(f,s) × Res(f,s) × Rec(f,s) × Vol(f,s)        each in [0,1]
```

| Component | Definition (provisional parameters) |
|-----------|--------------------------------------|
| **A — availability** | 1 if at least one source for *f* declares coverage of this location (`datasets_source_coverage`), else 0. *This is what separates "no incidents recorded" (A=1, low count) from "no incident data here" (A=0).* |
| **Res — resolution** | By the source's `original_geo_resolution`: exact point ≤ 50 m → 1.0; snapped/≤ 250 m → 0.8; line/raster ≤ 500 m → 0.7; locality → 0.3; city → 0.15; district → 0.1; state → 0.05 |
| **Rec — recency** | 1.0 if the data is within its expected refresh cadence; otherwise `exp(−(age − cadence)/cadence)` |
| **Vol — volume** | For statistical factors (incidents, reports): `min(1, N_area / N_min)`, where N_area is the count of records within the source's coverage near the route (e.g. H3 k-ring 3) and N_min is a provisional 30. For infrastructure factors this is 1 if the source's completeness check passed for that area, otherwise 0.5. |

Route-level values:
- Per-factor quality `Q_f` = length-weighted mean of `q(f,s)`.
- **Coverage index** `C = Σ_f w_f · Q_f / Σ_f w_f`, taken over **all factors in the active profile**, not only the available ones. Missing factors therefore lower coverage.
- The factor groups are **incident evidence** (`verified_incident_exposure`, `community_reports`, `model_risk`) and **infrastructure evidence** (proximity, lighting, activity, road characteristics).

**Coverage levels** (thresholds provisional, to be recalibrated on real coverage distributions in Stage 2):

| Level | Conditions (all must hold) | Plain-language meaning |
|-------|----------------------------|------------------------|
| **HIGH** | C ≥ 0.70; incident-evidence group quality ≥ 0.6; infrastructure group available on ≥ 90% of route length | Multiple kinds of recent, fine-grained evidence, including verified incident data |
| **MEDIUM** | C ≥ 0.45; infrastructure group available on ≥ 80% of length; incident evidence may be coarse or community-only | Good infrastructure evidence, with partial incident evidence |
| **LIMITED** | C ≥ 0.20; infrastructure evidence on ≥ 50% of length | Mostly infrastructure proxies; no reliable incident evidence |
| **INSUFFICIENT** | otherwise, or `uncovered_length_share` > 0.5 | Too little evidence to compare safety |

Community-data coverage is reported separately (count and recency of verified reports in the corridor). It feeds the coverage index only through the `community_reports` factor once Stage 4 credibility gates are live.

### 9.5 Weights (v1, provisional)

> These weights are **initial judgements, chosen only to make the mechanism work and testable**. No scientific basis is claimed for them. Stage 2 data analysis and Stage 3 model evaluation will decide whether they change. Every change creates a new profile version.

| Factor | DRIVING v1.0.0 |
|---|---|
| verified_incident_exposure | 0.30 |
| lighting | 0.20 |
| activity | 0.15 |
| police_proximity | 0.15 |
| hospital_proximity | 0.10 |
| road_characteristics | 0.10 |

Later factors (`hotspot_exposure`, `model_risk`, `community_reports`) are added in new profile versions. Other travel modes get their own profiles when they are enabled; for example, walking would likely weight lighting and sidewalks higher, and that will be documented when it is introduced.

### 9.6 Display rules (no confident-looking scores without evidence)

| Coverage level | `safety.status` | Score shown? | UI treatment |
|----------------|-----------------|--------------|--------------|
| HIGH / MEDIUM | `SCORED` | yes | score + coverage badge |
| LIMITED | `SCORED_LIMITED_EVIDENCE` | yes, with a prominent "Limited data" badge and the factors used | recommendation text says "based on infrastructure indicators only" |
| INSUFFICIENT | `NOT_SCORED` | **no**; `score: null` | "Not enough safety data for this route"; route drawn grey; distance and time still shown |

Every score also carries the "provisional methodology" notice (I-7).

### 9.7 Recommender: safest *practical* route, compared like-for-like

1. **Practical** means `duration ≤ fastest × max_detour_ratio` (default 1.30) **and** `duration − fastest ≤ 20 min`. Both are configurable, and logged-in users can set them in preferences.
2. **Like-for-like comparison:** routes are compared on a **comparison score** that uses only the factors available on *all* candidate routes. This prevents a route from winning just because it passes through an area with less data.
3. **Meaningful-difference threshold:** a route is recommended on safety only if its comparison score beats the others by at least δ, where δ depends on coverage (provisional: HIGH 3, MEDIUM 5, LIMITED 8 points). Otherwise `basis: NO_MEANINGFUL_DIFFERENCE` and the fastest practical route is suggested.
4. If every route is `NOT_SCORED`, then `basis: INSUFFICIENT_EVIDENCE`, and the routes are listed by time with no safety recommendation.
5. A non-practical route that is much safer (≥ 15 points on the comparison score) is shown as "Safest overall (longer)", but it is not the default.
6. Stage 6 improves this trade-off (for example user-adjustable safety/time preference and Pareto presentation).

### 9.8 Performance

- Corridor queries are one per factor family, indexed. H3 feature lookup is one `WHERE h3_r9 = ANY(:cells)` query.
- Caching (Stage 6/7): Redis cache of cell features and corridor POIs keyed by H3 cells. Route results are cached only briefly and only as our own computed outputs (§11.5).

---

## 10. ML Architecture

The data rules and leakage checklist are in DATA_STRATEGY §8. **No model is trained before Stage 3.**

### 10.1 Where ML can apply in India

ML needs **legitimately geolocated labels**: verified geolocated incident sources or, later, sufficient verified community reports. Such labels will exist only in some areas of India. So:
- Models are trained per **analysis region**, meaning an area where a qualifying label source exists (an admin area set or polygon). Nothing in code is city-specific. Analysis regions are data rows.
- Each model stores its **applicability domain**: the analysis region plus the feature ranges seen in training.
- **Outside the domain, `model_risk` is unavailable.** The model is never extrapolated across India, and `confidence.applicability = OUT_OF_DOMAIN`.
- Before any such region exists, Stage 3 delivers the full ML pipeline, tested on SYNTHETIC data (not activatable in production), and activation waits until real labels exist (see PROJECT_PLAN blocker B-5).

### 10.2 K-Means (hotspots)

Input: incidents or verified reports in the training window, projected to the local UTM zone and standardised, with optional severity × recency weights. We sweep k and select using silhouette, Davies–Bouldin, the elbow and a minimum cluster size. Clusters above an intensity threshold count as hotspots. HDBSCAN is evaluated as a challenger and reported, and K-Means stays the production method unless the owner approves a switch. Stage 6 adds scheduled **dynamic hotspot updates** with stability tracking.

### 10.3 Logistic Regression (risk)

The unit is H3 cell × time bucket × month. The label is ≥ 1 relevant incident in month t+1, using features from ≤ t only. The model is a scikit-learn `Pipeline` (scaler/encoder + `LogisticRegression`). We use temporal splits, a spatial-block robustness check, calibration and mandatory baselines. The primary metric is PR-AUC. Coefficients drive the factor explanations. **Model confidence** comes from bootstrap prediction intervals (Stage 3) and the applicability-domain check. It is kept separate from coverage.

### 10.4 Registry and lifecycle

`ml_model` stores name, version, algorithm, hyperparameters, feature list + `FEATURE_SET_VERSION`, snapshot, analysis region, applicability domain, periods, metrics (plus baselines), artifact URI + SHA-256, code version, `trained_on_synthetic`, and `activated_by/at`. Activation is manual. Synthetic-trained models are blocked in production. Inference is batch (grid rebuild), so there is no model call on the request path.

---

## 11. Google Maps / Routing Architecture

### 11.1 Services

| Service | Called from | Purpose | Stage |
|---------|-------------|---------|-------|
| Maps JavaScript API | browser | map, polylines | 1 |
| Places API (New) Autocomplete | browser | India-restricted place search (`includedRegionCodes: ["in"]`) | 1 |
| **Routes API** `computeRoutes` | **backend only** | real road alternatives | 1 |
| Geocoding API | backend, optional | resolving a dropped pin | 1 (opt) |

Google designates the Directions API as "Legacy". The Routes API is its successor. Field names, SKUs and pricing are verified against the official docs at Stage 1 and recorded in `docs/integrations/google.md`.

### 11.2 Provider abstraction

```
routing/providers/base.py      RouteProvider (ABC), ProviderRoute dataclass
                               (encoded_polyline, distance_m, duration_s, labels, warnings)
routing/providers/google.py    GoogleRoutesProvider
routing/travel_modes.py        TravelMode enum + registry:
                                 DRIVING      → Routes API "DRIVE"         (enabled in MVP)
                                 TWO_WHEELER  → "TWO_WHEELER"              (future; availability varies by region)
                                 WALKING      → "WALK"                     (future)
                                 BICYCLING    → "BICYCLE"                  (future; coverage limited)
```

Enabled modes come from `SAFEROUTE_ENABLED_TRAVEL_MODES` (MVP: `DRIVING`). Enabling a mode later means config, a scoring profile for that mode, and tests. It needs no redesign.

### 11.3 Routes API usage

- `POST https://routes.googleapis.com/directions/v2:computeRoutes`, with headers `X-Goog-Api-Key` (server key from env) and `X-Goog-FieldMask` (mandatory; minimal: `routes.distanceMeters,routes.duration,routes.polyline.encodedPolyline,routes.routeLabels`).
- Body: `origin`, `destination`, `travelMode`, `computeAlternativeRoutes: true`, `departureTime`, `routingPreference` for traffic-aware driving (this affects the SKU; decided at Stage 1), `regionCode: "in"`, `languageCode`.
- Known constraints (re-verified at Stage 1): at most about 3 alternatives, and none are computed when intermediate waypoints are given.
- **Candidate expansion (Stage 6, cost-gated, off by default):** extra requests via waypoints chosen from the safety grid, to widen the options. Every candidate is still a real provider route.

### 11.4 Resilience

Timeouts of 3 s connect and 10 s read. Two retries with jitter, on 5xx and 429 only. A circuit breaker. The key is never logged. **No fallback geometry.**

### 11.5 Google Maps Platform terms considerations (to be confirmed by the owner; not legal advice)

| Topic | Design response |
|-------|-----------------|
| Caching/storage of Google content is restricted | Provider polylines are not persisted. Only user-supplied inputs (place IDs, which Google allows to be stored, and user-entered coordinates) and **our own computed scores** are stored. |
| Google content should be shown on a Google map | All routes are displayed on the Google Maps JS map |
| Attribution | Google's map attribution is kept intact. Our data-source attributions are shown separately. |
| Key security | Separate browser and server keys with API and referrer/IP restrictions, plus quotas and budget alerts |
| Billing exposure from anonymous use | Anonymous throttles plus `GOOGLE_ROUTES_DAILY_CAP`. When the cap is hit, the API returns 503 with a clear message. |

### 11.6 Keys

| Key | Restrictions |
|-----|-------------|
| `VITE_GOOGLE_MAPS_BROWSER_KEY` (public by nature) | HTTP-referrer restricted; Maps JavaScript API and Places API (New) only |
| `GOOGLE_MAPS_SERVER_KEY` (secret) | IP-restricted where the platform allows; Routes API (and Geocoding) only |

The owner creates both keys. They are never generated, committed or logged by the project.

---

## 12. Security and Privacy Architecture

| Area | Controls |
|------|----------|
| Transport | HTTPS, HSTS, secure cookies |
| AuthN/AuthZ | §7. IDOR tests for every user-owned resource. |
| Anonymous endpoints | Per-IP throttles, a global routing budget cap, request-size limits, bbox area limits on layers, and bot mitigation evaluated in Stage 7 |
| Input validation | Serializers, coordinate ranges, country check, allowed travel modes |
| Injection / XSS | ORM and parameterised SQL. React escaping. Reports rendered as plain text. |
| CSP / CORS | Strict CSP allowing only Google Maps domains. CORS allowlist. |
| Secrets | Env or secret manager only. gitleaks in CI and pre-commit. |
| Dependencies | pip-audit, npm audit, Dependabot |
| **Personal data (India)** | Location data is personal data. The design follows data minimisation: anonymous route searches are **not stored**; saved routes are opt-in; retention limits apply (route history 90 days by default, trip pings 30 days after the trip). No raw coordinates go into logs. Compliance with India's **Digital Personal Data Protection Act, 2023** and its rules is to be reviewed before Stage 5 and production (owner and legal). |
| Community reports | Moderation, credibility, rate limits, location fuzzing, no third-party PII |
| Emergency features | Consent-based guardian access that can be revoked and expires. Audited SOS delivery. The discreet emergency interface (Stage 6) must not weaken authentication. |
| **12.6 Map boundaries** | The depiction of India's external boundaries is legally regulated in India. SafeRoute **does not draw country or state boundary lines** on user-facing maps, and relies on the Google base map. Admin polygons are used internally only. Any future boundary display needs a legal check and an officially compliant boundary source. |
| Out of scope by rule | CCTV analytics, facial recognition, surveillance of third parties (I-10) |

---

## 13. Environment-Variable Strategy

- 12-factor config via `django-environ` and Vite `import.meta.env`.
- `.env.example` is committed with names and comments only. `backend/.env` and `frontend/.env.local` are gitignored. Production values come from the platform's secret manager.
- Settings split: `base/dev/test/prod`. `prod` **fails fast** if secrets are missing, if `DEBUG` is on, or if `SAFEROUTE_ALLOW_SYNTHETIC=true`.
- Every `VITE_*` variable is **public**, so only the restricted browser key and public URLs go there.

| Variable | Scope | Secret | Notes |
|----------|-------|--------|-------|
| `DJANGO_ENV`, `DJANGO_SETTINGS_MODULE`, `DJANGO_DEBUG`, `DJANGO_ALLOWED_HOSTS` | backend | no | |
| `DJANGO_SECRET_KEY` | backend | **yes** | |
| `DATABASE_URL` | backend | **yes** | `postgis://…` |
| `GDAL_LIBRARY_PATH`, `GEOS_LIBRARY_PATH` | backend | no | **native Windows** paths to OSGeo4W DLLs |
| `REDIS_URL` | backend | yes | Stage 2+ |
| `CORS_ALLOWED_ORIGINS`, `CSRF_TRUSTED_ORIGINS` | backend | no | |
| `JWT_ACCESS_LIFETIME_MIN`, `JWT_REFRESH_LIFETIME_DAYS` | backend | no | 15 / 7 |
| `GOOGLE_MAPS_SERVER_KEY` | backend | **yes** | owner-provided |
| `GOOGLE_ROUTES_DAILY_CAP` | backend | no | billing guard |
| `ANON_SAFE_ROUTES_RATE`, `USER_SAFE_ROUTES_RATE` | backend | no | e.g. `30/hour` |
| `SAFEROUTE_COUNTRY_CODE` | backend | no | `IN` |
| `SAFEROUTE_ENABLED_TRAVEL_MODES` | backend | no | `DRIVING` |
| `SAFEROUTE_INGESTION_SCOPE` | pipeline | no | `country:IN` in prod; `test-regions` in dev |
| `SAFEROUTE_ALLOW_SYNTHETIC` | backend | no | `true` only in dev/test |
| `DATA_RAW_DIR`, `ARTIFACT_STORAGE` | backend | no | |
| `EARTHDATA_TOKEN` | pipeline | **yes** | only if VIIRS is used (manual step in DATA_STRATEGY §5.4) |
| `DATA_GOV_IN_API_KEY` | pipeline | **yes** | only if a data.gov.in API source is adopted (owner registers) |
| `SENTRY_DSN` | both | semi | optional |
| `VITE_API_BASE_URL`, `VITE_GOOGLE_MAPS_BROWSER_KEY`, `VITE_GOOGLE_MAPS_MAP_ID` | frontend | public | |
| Stage 5: `NOTIFIER_*` | backend | **yes** | provider TBD |

---

## 14. Testing Strategy

| Layer | Tooling | Focus |
|------|---------|-------|
| Backend unit | pytest, pytest-django, hypothesis | scoring maths (bounds, monotonicity, P90), **coverage maths** (A=0 vs low count, resolution tiers, thresholds), display rules, like-for-like recommender, adaptive resampling, travel-mode registry |
| Backend integration | pytest-django on **local PostGIS** (native in dev, service container in CI), factory_boy | corridor/bbox/radius queries, admin tagging, idempotent ingestion, grid build, migrations |
| Provider | respx with **hand-written fixture responses** matching the documented schema | alternatives, zero routes, errors, budget cap, **no fallback geometry** |
| API contract | APIClient + OpenAPI snapshot | public vs login access, throttles, response shape, IDOR |
| **Geographic neutrality** | pytest | Fixtures from **several representative regions across India** (e.g. Delhi → Gurugram cross-state, Mumbai → Navi Mumbai, Chandigarh → Mohali cross-UT/state, Bengaluru → Electronic City) with synthetic fixture routes; a region with **zero data** must yield `NOT_SCORED`; a grep test fails on city/state literals in core packages |
| **Aggregate safety** (I-5) | pytest | ingesting an aggregate never creates `incident` rows; aggregates have no geometry |
| Data quality | pandera + checks | coordinates, timestamps, duplicates, provenance completeness |
| ML | pytest | leakage suite, reproducibility, baselines, applicability domain, activation guards |
| Frontend | Vitest + RTL + MSW, `tsc --noEmit`, ESLint (no `any`) | planner, badges (score + coverage), grey "not scored" rendering, synthetic banner |
| E2E | Playwright | public route search smoke test (Stage 1), full suite (Stage 7) |
| Security / perf | bandit, pip-audit, npm audit, gitleaks; ZAP and Locust (Stage 7) | |

CI never calls real Google or data sources. Live tests are marked `@pytest.mark.live` and run manually. Coverage gates: ≥ 85% for `safety`, `routing/services`, `datasets/pipeline` and `ml`; ≥ 70% overall.

---

## 15. Deployment Architecture

### 15.1 Local development: **native Windows, no Docker required**

| Component | Install |
|-----------|---------|
| Python 3.12 | already installed; project venv `backend/.venv` |
| Node.js 24 | already installed |
| PostgreSQL 16/17 | EDB Windows installer |
| PostGIS | installed through the EDB **Stack Builder** (Spatial Extensions) |
| GDAL / GEOS / PROJ (for GeoDjango) | **OSGeo4W** installer. `GDAL_LIBRARY_PATH` / `GEOS_LIBRARY_PATH` point at its DLLs. The version must be one Django 5.2 supports (checked at setup). |
| Redis (Stage 2+) | Celery needs a broker. On native Windows, options are Memurai (a Redis-compatible Windows service) or Redis in WSL. Decided at Stage 2. Stages 0–1 need no Redis. |

The step-by-step guide is written in Stage 0b as `docs/setup-windows.md`. Docker support may be added later for reproducible deployment, but it is **optional** and never required for local development.

### 15.2 Environments

local (native), CI (Linux runners with a PostGIS service), staging, production. Synthetic data is allowed only in local and CI.

### 15.3 Production (provider decided before Stage 7)

- CDN or static hosting for the React build.
- HTTPS load balancer in front of backend containers (≥ 2), plus Celery worker and beat.
- Managed PostgreSQL with PostGIS, managed Redis, and object storage for raw data and artifacts.
- An **India hosting region** is preferred for latency and data-residency considerations.
- Migrations run as a release step. Daily backups with point-in-time recovery. Structured logs, error tracking and uptime checks.

---

## 16. Complete Folder Structure

This is the target structure. Folders are created in the stage that needs them.

```
SafeRoute_main/
├── README.md  PROJECT_PLAN.md  ARCHITECTURE.md  DATA_STRATEGY.md
├── .env.example  .gitignore  .gitattributes  .editorconfig  .pre-commit-config.yaml
├── .github/workflows/ci.yml
├── docs/
│   ├── setup-windows.md          # Stage 0b
│   ├── adr/                      # future decisions
│   ├── integrations/google.md    # verified API details
│   ├── data-sources/<slug>.md    # per-source datasheets
│   └── methodology/{scoring.md,coverage.md}   # provisional methodology, versioned
├── backend/
│   ├── manage.py  pyproject.toml  requirements/{base,dev,prod}.txt
│   ├── config/settings/{base,dev,test,prod}.py  urls.py  asgi.py  wsgi.py  celery.py
│   ├── apps/
│   │   ├── core/
│   │   ├── accounts/
│   │   ├── geo/                  # country config, admin hierarchy, H3/projection utils
│   │   ├── routing/
│   │   │   ├── providers/{base.py,google.py}
│   │   │   ├── travel_modes.py  services/  serializers.py  views.py  tests/
│   │   ├── safety/
│   │   │   ├── factors/{base.py,incidents.py,proximity.py,lighting.py,activity.py,road.py,...}
│   │   │   ├── engine/{resample.py,corridor.py,aggregate.py,recommender.py}
│   │   │   ├── coverage/{evidence.py,levels.py}
│   │   │   ├── grid/build.py  tests/
│   │   ├── datasets/
│   │   │   ├── sources/
│   │   │   │   ├── base.py               # SourceAdapter interface
│   │   │   │   ├── government/           # data.gov.in / NCRB aggregates, state portals
│   │   │   │   ├── osm/                  # POIs, road features, lit tags
│   │   │   │   ├── infrastructure/       # non-OSM facility registries (if verified)
│   │   │   │   ├── lighting/             # VIIRS, municipal lighting data (if found)
│   │   │   │   ├── community/            # promotion of verified reports (Stage 4)
│   │   │   │   └── synthetic/            # dev/test only
│   │   │   ├── pipeline/{validate.py,normalize.py,dedupe.py,admin_tag.py,load.py,snapshot.py}
│   │   │   ├── taxonomy.yaml
│   │   │   ├── management/commands/{ingest_source,build_feature_grid,rebuild_all,seed_synthetic}.py
│   │   │   └── tests/
│   │   ├── ml/  reports/  emergency/  tracking/
│   └── tests/                    # cross-app, geographic-neutrality tests
├── frontend/
│   ├── package.json  vite.config.ts  tsconfig.json  eslint.config.js  index.html
│   └── src/ (see §2.2)   tests/  e2e/
├── data/
│   ├── countries/in.yaml         # committed: country config
│   ├── sources/<slug>.yaml       # committed: source manifests
│   ├── test_regions/<slug>.geojson   # committed: DEV/TEST clip areas (not product limits)
│   ├── raw/ interim/ processed/  # gitignored
├── ml/notebooks/  ml/artifacts/  # artifacts gitignored
└── scripts/
```

---

## 17. Stage-by-Stage Roadmap

Detail is in PROJECT_PLAN.md §4.

| Stage | Name |
|------|------|
| 0 | Architecture & environment (native Windows) |
| 1 | MVP: public route search, Google road alternatives (DRIVING), provisional scoring + coverage on SYNTHETIC dev data, auth |
| 2 | India-wide dataset pipeline: source adapters, admin hierarchy, provenance, coverage extents, real infrastructure data |
| 3 | ML safety intelligence (per analysis region, applicability domain) |
| 4 | Community reporting (India-wide, credibility gates) |
| 5 | Emergency & guardian functionality |
| 6 | Advanced safety intelligence |
| 7 | Testing, security, optimisation, production deployment |

---

## 18. Decision Log

| ID | Decision | Rationale | Status |
|----|----------|-----------|--------|
| ADR-001 | Modular monolith (Django) + SPA | Scale and team size; clear app boundaries | Accepted |
| ADR-002 | PostGIS + GeoDjango | Spatial indexes, corridor/bbox/radius queries | Accepted |
| ADR-003 | **Sparse** H3 res-9 feature grid; proximity computed at request time | India-wide scale without a 30 M-cell dense grid | Accepted (revised for India) |
| ADR-004 | Google Routes API, server-side, behind the `RouteProvider` abstraction | Real road routes; key secrecy; provider independence | Accepted |
| ADR-005 | Provider polylines are not persisted | Google terms (owner to confirm) | Accepted |
| ADR-006 | Rule-based provisional scoring first; ML in Stage 3 behind the same interface | No labelled data yet | Accepted |
| ADR-007 | Route risk = α·mean + (1−α)·P90 | Dangerous stretches cannot be averaged away | Accepted (provisional parameters) |
| ADR-008 | JWT: in-memory access token + rotating httpOnly refresh cookie | XSS resilience | Accepted |
| ADR-009 | `data_origin` + `verification_status` on every record; synthetic hidden by default and blocked in prod | Provenance and separation | Accepted |
| ADR-010 | Vite + React + **TypeScript** (strict, no `any`) | Owner approved (A-6) | Accepted |
| ADR-011 | K-Means on projected coordinates, per analysis region; HDBSCAN as challenger | Brief requires K-Means | Accepted |
| ADR-012 | Logistic Regression with temporal validation, baselines, applicability domain | Leakage prevention; no extrapolation across India | Accepted |
| ADR-013 | **Native Windows** local dev; Docker optional later | Owner decision (A-3) | Accepted |
| ADR-014 | Python 3.12 / Django 5.2 LTS | 3.8 is end-of-life | Accepted |
| ADR-015 | **Public route search**; login only for personal features | Owner decision (A-5) | Accepted |
| ADR-016 | **India-wide scope**; country via config; no city/state constants in code | Owner decision (A-2 revised) | Accepted |
| ADR-017 | Four coverage concepts kept distinct; coverage ≠ confidence | Owner requirement; honesty | Accepted |
| ADR-018 | Aggregate statistics in a separate, geometry-free table; never spread to streets | Owner requirement (I-5) | Accepted |
| ADR-019 | Insufficient coverage → no numeric score; like-for-like comparison; coverage-dependent δ | Avoids confident-looking scores without evidence | Accepted (provisional thresholds) |
| ADR-020 | DRIVING only in the MVP; `TravelMode` registry and per-mode scoring profiles | Owner decision | Accepted |
| ADR-021 | Stage 6 = Advanced Safety Intelligence; no CCTV, facial recognition or surveillance | Owner decision (A-1) | Accepted |
| ADR-022 | No boundary lines drawn by SafeRoute on user-facing maps | Indian boundary-depiction rules | Accepted |
| ADR-023 | Anonymous routing protected by throttles + a global daily cap | Public access with bounded billing | Accepted (values need owner input) |
