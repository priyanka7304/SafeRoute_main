# SafeRoute — Data Strategy

> **SafeRoute is designed for India-wide operation.**
> **Status:** Stage 0, approved with owner decisions (PROJECT_PLAN.md §6).
> This document covers the consolidated SafeRoute dataset: sources, provenance, the India-wide adapter architecture, the handling of aggregated statistics, the pipeline, synthetic-data policy, feature engineering, ML data rules, and development test regions.
> Companion documents: [ARCHITECTURE.md](ARCHITECTURE.md), [PROJECT_PLAN.md](PROJECT_PLAN.md).

---

## 1. Principles

1. **No fabricated data presented as real.** Every record traces back to a documented source.
2. **Never turn aggregated statistics into street-level data.** If an official table says a district had N incidents, SafeRoute never distributes those N incidents onto streets and never invents coordinates (§6).
3. **Missing is not safe.** An area without data is "no evidence", and that lowers coverage. It is never treated as low risk.
4. **"No records" is different from "no source".** Each source declares its coverage extent, so the system can tell "a source covers this area and recorded few incidents" from "no source covers this area".
5. **Proxies are labelled as proxies.** Night-time brightness is not street-light quality, and venue density is not crowd measurement.
6. **Reproducibility.** Raw downloads are immutable and checksummed, and the database can be rebuilt from raw files plus code.
7. **Geographic neutrality.** Adding a new state or city means adding a source adapter or manifest. The safety engine is not rewritten.
8. **Licences are respected.** Attribution is displayed, and share-alike obligations are tracked.

---

## 2. Data reality for India

| Data type | What is realistically available | Consequence |
|-----------|--------------------------------|-------------|
| Crime/incident statistics | Official national statistics (NCRB's *Crime in India*, with tables republished on data.gov.in) are, as far as I know, published at **state/UT, district and selected-city level**, not as geolocated incidents. | Stored as **aggregates** (§6). They are not a street-level factor. |
| Street-level incident data | **No uniform, open, verified, geolocated incident dataset is known for India.** Some state or city portals may publish something. Each candidate must be verified individually in Stage 2 and after. | `verified_incident_exposure` is **optional** and available only where a verified source exists. Elsewhere it is marked `UNAVAILABLE`. |
| Police stations, hospitals, fire stations, pharmacies | OpenStreetMap (nationwide, completeness varies). Possibly official registries (to be verified). | Available nationally with variable completeness, which is measured per area. |
| Road network and characteristics | OpenStreetMap (nationwide) | Available. Tag completeness (e.g. `lit`, `sidewalk`) varies. |
| Lighting | OSM `lit` tags and street lamps (likely sparse in many areas); satellite night-time lights (VIIRS, nationwide, coarse); possibly municipal/smart-city datasets (to investigate) | Lighting is a proxy with explicit limitations (§5.4). |
| Community reports | SafeRoute users (Stage 4+) | Coverage grows over time. Reports stay distinguishable from verified records. |

**Routing coverage and safety-data coverage are different.** Google may route anywhere in India, while safety evidence will be uneven. The coverage model (ARCHITECTURE §9.4) makes this visible to users.

---

## 3. Source-adapter architecture

### 3.1 Layout

```
backend/apps/datasets/sources/
  base.py              # SourceAdapter interface
  government/          # official aggregates (e.g. NCRB tables via data.gov.in), state/city open-data portals
  osm/                 # POIs (police, hospitals, fire, pharmacies, transit, venues, lamps), road features
  infrastructure/      # non-OSM facility registries, if a legitimate open one is verified
  lighting/            # VIIRS night-time lights; municipal lighting datasets if found
  community/           # promotes verified SafeRoute reports into the consolidated dataset (Stage 4)
  synthetic/           # DEV/TEST ONLY generators, clearly labelled
data/sources/<slug>.yaml   # committed manifest per source instance
```

### 3.2 `SourceAdapter` interface

```python
class SourceAdapter(ABC):
    slug: str
    adapter_category: Literal["government", "osm", "infrastructure", "lighting", "community", "synthetic"]
    record_type: Literal["incident", "poi", "road_feature", "lighting", "aggregate_statistic"]

    def fetch(self, scope: IngestionScope, raw_dir: Path) -> RawArtifact: ...
    def parse(self, raw: RawArtifact) -> pd.DataFrame: ...
    def normalize(self, df: pd.DataFrame) -> pd.DataFrame: ...          # → SafeRoute canonical schema
    def coverage_extent(self, df: pd.DataFrame) -> list[CoverageClaim]:  # where the source claims completeness
```

- **`IngestionScope`** is configuration: `country:IN`, a list of admin areas, or `test-regions`. The same adapter serves all of these.
- **One adapter type, many manifests.** For example, a generic `government/csv_table` or `government/ogd_api` adapter plus one manifest per dataset. A new state dataset in a supported format needs **only a manifest**. A new format needs one adapter.

### 3.3 Manifest (`data/sources/<slug>.yaml`)

```yaml
slug: osm-india
name: OpenStreetMap India extract
publisher: OpenStreetMap contributors (via Geofabrik)
publisher_type: OPEN_COMMUNITY
adapter: osm/pbf
record_types: [poi, road_feature]
url: <verified download URL, recorded in Stage 2>
license: ODbL-1.0
attribution: "© OpenStreetMap contributors"
native_geo_resolution: POINT
refresh_cadence: P1M
coverage: { country: IN }
data_origin: VERIFIED_SOURCE
notes: Completeness varies by area; measured per area during ingestion.
```

---

## 4. Consolidated SafeRoute schema and provenance

### 4.1 Required provenance on every record

| Field | Meaning |
|-------|---------|
| `source` | FK to `data_source` (publisher, URL, licence) |
| `source_record_id` | the upstream identifier, or a stable hash if none exists |
| `original_geo_resolution` | resolution as published: `POINT`, `SNAPPED_POINT`, `LINE`, `RASTER`, `LOCALITY`, `CITY`, `DISTRICT`, `STATE`, `COUNTRY` |
| `data_origin` | see §4.2 |
| `verification_status` | `VERIFIED`, `UNVERIFIED`, `PENDING_REVIEW`, `REJECTED`, `NOT_APPLICABLE` |
| `source_timestamp` | when the event occurred or the data was valid, per the source |
| `source_updated_at` | the source's own last-updated date, when published |
| `retrieved_at` | when SafeRoute collected it |
| `processing_version` | pipeline + taxonomy version that produced the row |
| `ingestion_run` | the run (raw file checksum, counts) |
| geographic metadata | `geom` (lat/lng) for geolocated records, and **derived** `state`, `district`, `city/locality` FKs via point-in-polygon when boundary data exists. Aggregates carry an `admin_area` FK instead of geometry. |

### 4.2 `data_origin` definitions

| Value | Meaning | Examples |
|-------|---------|----------|
| `VERIFIED_SOURCE` | Ingested from a documented, legitimate external source whose provenance is known. *It means the source is verified, not that each row is guaranteed accurate.* | OSM POIs, an official portal's incidents, NCRB aggregate tables, VIIRS rasters |
| `DERIVED` | Computed by SafeRoute from other records, with the inputs traceable | H3 cell features, hotspots, admin tagging, lighting indices |
| `COMMUNITY` | Submitted by SafeRoute users | Stage 4 reports (and incidents promoted from verified reports) |
| `SYNTHETIC` | Generated for development and testing only | Seeded demo incidents and POIs |

### 4.3 Administrative hierarchy

`geo_admin_area` stores COUNTRY → STATE_UT → DISTRICT → (SUBDISTRICT) → CITY → LOCALITY.
- The boundary source is chosen and verified in Stage 2. Candidates are OSM administrative boundaries (ODbL) and official sources where the licence allows. Where available we record official codes, such as Local Government Directory codes.
- Boundaries are used **internally** for tagging, filtering and linking aggregates. They are not drawn as boundary lines on user maps (ARCHITECTURE §12.6).
- Districts change over time (new districts get created), so areas carry `valid_from/valid_to`, and aggregates link to the area valid in their period.
- **Safety calculations use coordinates and spatial queries**, not admin boundaries.

### 4.4 Validation rules (every geolocated row)

| Rule | Action |
|------|--------|
| lat ∈ [−90, 90], lng ∈ [−180, 180] | reject `coord_out_of_range` |
| not null island (within 1 km of 0,0) | reject `null_island` |
| inside the configured country (bbox in Stage 1, polygon from Stage 2) and inside the ingestion scope | drop `out_of_scope` (counted) |
| lat/lng swap suspicion | fix only for sources documented to swap; otherwise reject |
| precision < 3 decimals | flag, set `location_precision_m` |
| timestamps parseable, not in the future, within the source's range | reject `bad_timestamp` |
| required fields present | reject `missing_required` |
| category mapped to taxonomy | map to `OTHER` + log (never silently dropped) |

A run whose rejection rate exceeds the threshold (default 10%) is marked `PARTIAL` and needs review.

### 4.5 Taxonomy (provisional)

`taxonomy.yaml` maps each source's labels to canonical categories, a **provisional severity (1–5)**, and a travel-safety relevance flag. The severities are **methodological choices, not established facts**. They are reviewed with the owner and revisited after the Stage 2 and 3 analysis. Initial canonical categories: `VIOLENT_ASSAULT`, `SEXUAL_OFFENCE`, `HARASSMENT`, `ROBBERY`, `SNATCHING`, `THEFT_PERSON`, `WEAPONS`, `PUBLIC_DISORDER`, `VEHICLE_CRIME`, `PROPERTY_CRIME`, `OTHER`. Original labels are always kept verbatim.

### 4.6 De-duplication

1. In-source: `UNIQUE (source_id, source_record_id)` + upsert. A changed `raw_hash` updates the row.
2. No-ID sources: a hash of (category, rounded location, timestamp).
3. Cross-source incidents: same category within 50 m and 30 min gives a shared `dedupe_group_id`, counted once. Thresholds are provisional.
4. POIs: the same facility mapped twice (node + area, or OSM + registry) merges when within 100 m with matching name or kind. Provenance of both is kept.

---

## 5. Source catalogue (candidates; each must pass verification)

> "Candidate" means the source category is known to exist, but its current access method, licence, schema, coverage and freshness **must be verified in Stage 2**, and written up in `docs/data-sources/<slug>.md` before ingestion. **Nothing here is claimed as verified yet.**

### 5.1 Nationwide

| Category | Candidate | Records | Resolution | Licence (to verify) |
|----------|-----------|---------|------------|---------------------|
| osm | OpenStreetMap India extract from Geofabrik (`.osm.pbf`; confirm whether to use the India extract or its sub-regions) | police, hospitals (+ `emergency=yes`), clinics, fire stations, pharmacies, transit stops, activity venues, street lamps, road features (`highway`, `lit`, `sidewalk`, `lanes`) | point / line | ODbL 1.0 |
| osm | OSM administrative boundaries | `geo_admin_area` | polygon | ODbL 1.0 |
| lighting | NASA Black Marble VIIRS night-time lights (§5.4) | radiance per H3 r8 cell | ≈ 500 m raster | NASA data policy (verify citation terms) |
| government | NCRB *Crime in India* tables (via data.gov.in or NCRB publications) | `aggregate_statistic` | state / district / city | Government Open Data License – India (verify per dataset) |
| infrastructure | Official health-facility or police-station directories, **if** an openly licensed, geocoded version is verified (e.g. candidate datasets on data.gov.in, or national registries if they offer open bulk access) | POIs | point | verify |

### 5.2 State / city level (discovered incrementally)

State open-data portals, city smart-city portals and police open-data pages may publish useful datasets such as police station lists, streetlight inventories or incident data. **No such source is assumed.** Each one found goes through the same verification checklist:

1. Is it published by the responsible authority or a legitimate open-data platform?
2. Is the licence explicit and does it permit our use?
3. What is its geographic resolution? Are coordinates present, and how precise?
4. What time period does it cover, and how is it updated?
5. What area does it cover? (This becomes `source_coverage`.)
6. Does it contain personal data? If so, it is not ingested unless lawful and necessary.

Each verified source becomes a manifest (and an adapter only if the format is new).

### 5.3 Explicitly excluded

- News or social-media scraping (reliability, copyright, terms-of-service problems).
- Storing Google Places data as a factor source (Google caching terms).
- Any CCTV, camera or facial data (I-10).
- Coordinates generated from aggregates (I-5).

### 5.4 Lighting: proxies and their limits

| Proxy | What it measures | Limitations |
|-------|------------------|-------------|
| OSM `lit=yes/no` on roads | a mapper's observation that a road is lit | Often sparse in India. Coverage is measured as `lit_tag_coverage`, and unknown is not "unlit". |
| OSM `highway=street_lamp` | mapped lamp positions | Very incomplete in most areas. Used only where its completeness check passes. |
| **VIIRS night-time lights** | upward radiance seen from orbit at ≈ 500 m resolution | Brightness **does not equal street-light quality or personal safety**. It includes commercial, residential and vehicle light. Its coarse resolution mixes lit and dark streets. Cloud, moon and seasonal effects apply. Composites are not hour-specific. It is used as a **coarse lighting/activity proxy** with low evidence resolution (0.7). |
| Municipal/smart-city lighting data | streetlight inventories or status, if published | Availability unknown. **Investigated in Stage 2 before VIIRS is made the final lighting feature.** |

**Stage 1 does not depend on VIIRS.**

**Manual steps for VIIRS (only if adopted in Stage 2; the owner does these, and no credentials are ever created or stored by the project):**
1. Create a free **NASA Earthdata Login** account at `https://urs.earthdata.nasa.gov/`.
2. Once signed in, generate a **user token** from your Earthdata Login profile (the "Generate Token" page).
3. Put it in your local `backend/.env` as `EARTHDATA_TOKEN=<your token>`. It is never committed.
4. If the download endpoint requires it, accept the data-provider application's terms (e.g. LAADS DAAC) when first prompted after login.
5. The pipeline then downloads the Black Marble monthly/annual products (candidate product IDs such as VNP46A3/VNP46A4, confirmed in Stage 2) for the tiles covering the ingestion scope.

Exact product choice, tile list and endpoint are verified and recorded in `docs/data-sources/viirs-black-marble.md` in Stage 2.

---

## 6. Aggregated statistics: how they may and may not be used

| Allowed | Not allowed |
|---------|-------------|
| Store them in `aggregate_statistic`, linked to the admin area and period, with the verbatim indicator label and source | Distributing counts onto streets, H3 cells or random coordinates |
| Display them as labelled **"area context"** (e.g. "District-level statistic, 2022, source: NCRB"), shown with its resolution | Presenting a district figure as if it described a specific street |
| Compute rates only where a documented population denominator is available, labelled as such | Ranking or comparing districts without caveats about reporting differences between jurisdictions |
| **Optional future use in scoring**: only a documented method that works at the aggregate's own resolution (e.g. a very low-weight area-level prior with resolution quality 0.05–0.1). It is **disabled by default** and needs owner approval after Stage 2 analysis. | Letting an aggregate raise a route's coverage level to HIGH or MEDIUM |

Reason: reported-crime statistics reflect reporting and recording practices as well as underlying risk, and they vary between states. Their resolution cannot support street-level conclusions.

---

## 7. Synthetic data policy

Synthetic data is allowed **only** in development and tests.

| Rule | Enforcement |
|------|-------------|
| Belongs to a `SAFEROUTE_SYNTHETIC` source with a `synthetic-` slug; `data_origin = SYNTHETIC` on every row | seed command, DB check trigger |
| Default ORM managers exclude synthetic rows; `.with_synthetic()` must be called explicitly | managers + tests |
| Any response influenced by synthetic data sets `data_notice.synthetic_data_used = true`, and the UI shows a persistent **"SYNTHETIC DEVELOPMENT DATA — not real safety information"** banner | serializer + UI tests |
| `SAFEROUTE_ALLOW_SYNTHETIC=true` is required to seed or use it; production settings refuse to start with it set | settings guard + test |
| Synthetic data is generated **only inside development test regions** (§9), seeded and reproducible | command |
| Synthetic data is **never derived from real aggregates**, to avoid "realistic-looking" fake street data | code review + test |
| Models trained on synthetic data are flagged and cannot be activated in production | registry guard |

**Stage 1** uses synthetic data only to demonstrate the provisional score and coverage interface. The resulting scores are never presented as real-world safety information.

---

## 8. Feature engineering and ML data rules

### 8.1 Spatial and temporal units

- H3 res 9 for local features (sparse grid), res 8 for raster proxies.
- Time buckets (**local time**, taken from the country config): `LATE_NIGHT` 00–05, `EARLY_MORNING` 05–08, `MORNING` 08–12, `AFTERNOON` 12–17, `EVENING` 17–20, `NIGHT` 20–24. A weekday/weekend split is a Stage 6 candidate.
- Sources with only month or year precision never feed time-bucket features.

### 8.2 Features (per cell; all nullable; evidence components stored alongside)

`inc_recency_weighted`, `inc_count_12m`, `inc_bucket_profile` (only where a verified geolocated incident source exists); `lit_road_share`, `lit_tag_coverage`, `lamp_density`, `lighting_radiance`; `activity_density`; `road_class_mix`, `sidewalk_share`; `hotspot_exposure` and `model_risk_by_bucket` (Stage 3); `community_verified_reports_90d` (Stage 4). Proximity features are computed at request time from the corridor. The feature code is versioned (`FEATURE_SET_VERSION`).

### 8.3 ML labels in an India-wide system

- Labels must come from **legitimately geolocated** records: a verified incident source, or (after Stage 4) verified community reports with enough volume. **Aggregates are never labels for cell-level models.**
- Models are trained per **analysis region** where labels exist. They record their applicability domain and never produce `model_risk` outside it (ARCHITECTURE §10.1).
- If no qualifying label source exists by Stage 3, the ML pipeline is built and tested on synthetic data, but it is not activated in production (PROJECT_PLAN blocker B-5).

### 8.4 Leakage checklist (each item gets an automated test)

| Risk | Prevention |
|------|-----------|
| Random splits across time | time-ordered train/validation/test only |
| Features using future data | feature builder takes `as_of`; a test injects a future incident and asserts no feature change |
| Hotspots fit on test data | K-Means refit per fold on data ≤ fold end |
| Preprocessing fit on all data | sklearn `Pipeline` fit on the training fold only |
| Spatial autocorrelation | additional spatial-block evaluation (H3 parent groups) |
| Duplicates inflating labels | dedupe groups counted once |
| Label/feature window overlap | asserted disjoint (features ≤ t, label t+1) |
| Synthetic in evaluation | excluded by default; flagged if used |
| Threshold tuned on test | tuned on validation; test evaluated once per version |
| Community-report feedback loop (reports cluster where users are) | report-derived labels are weighted by user-density-aware credibility and evaluated separately; documented limitation |

### 8.5 Metrics and registry

Primary metric PR-AUC, plus ROC-AUC, Brier, log-loss and calibration bins, broken down per time bucket. Mandatory baselines are the previous-period indicator and the historical per-cell rate. K-Means is evaluated with silhouette, Davies–Bouldin, inertia curve and temporal stability. The registry record contains: name, version, algorithm, hyperparameters, feature list and version, snapshot, analysis region, applicability domain, periods, metrics, artifact URI and SHA-256, code version, random seed, `trained_on_synthetic`, and activation details.

---

## 9. Development and test dataset coverage (not a product restriction)

To keep development fast, local and CI environments **do not download or process all of India**. Ingestion scope is configuration (`SAFEROUTE_INGESTION_SCOPE`):

| Environment | Scope |
|-------------|-------|
| local dev | `test-regions`: OSM data clipped to a few representative regions |
| CI | small committed fixtures (synthetic, or tiny ODbL-attributed OSM clips) |
| staging/production | `country:IN` |

**Proposed development test regions** (they cover different states/UTs and include cross-boundary routes; owner to confirm, blocker B-4):

| Test region | Why |
|-------------|-----|
| Chandigarh Tricity (Chandigarh UT, Mohali – Punjab, Panchkula – Haryana) | cross-UT/state routes in a small area; owner's local knowledge helps sanity checks |
| Delhi – Gurugram | cross-state (Delhi NCT → Haryana), dense metro |
| Mumbai – Navi Mumbai | large metro, coastal geometry |
| Bengaluru – Electronic City | southern metro, long urban corridors |
| One low-data area (e.g. a rural district stretch) | proves `LIMITED` / `INSUFFICIENT` coverage behaviour |

Test-region polygons live in `data/test_regions/` and are labelled **development-only**. Tests assert that the core code runs identically with any region set, and that no city or state literal appears in core packages.

---

## 10. Pipeline

```
fetch (adapter) ─► raw zone  data/raw/<slug>/<UTC timestamp>/  (immutable, sha256)
  ─► parse ─► validate (§4.4) ─► normalise (schema, taxonomy, UTC, resolution)
  ─► deduplicate (§4.6) ─► admin tagging (point-in-polygon, derived)
  ─► load (idempotent upsert; provenance columns) ─► source coverage claims
  ─► snapshot (+ profiling report, quality gates)
  ─► build sparse feature grid (+ evidence components per factor)
```

| Command | Purpose |
|---------|---------|
| `manage.py ingest_source <slug> [--scope …] [--from-raw <path>]` | run one source |
| `manage.py create_snapshot --latest` | freeze runs |
| `manage.py build_feature_grid --snapshot <id> [--scope …]` | build grid (parallelisable per state through Celery) |
| `manage.py rebuild_all [--scope …] [--offline]` | full reproducible rebuild |
| `manage.py seed_synthetic --region <test-region> --seed 42` | dev only |

**Quality gates before a snapshot activates:** rejection rates, POI completeness sanity checks per area (reported, and they lower the infrastructure `Vol` component where they fail), `lit` tag coverage, provenance completeness at 100%, and a profiling report. If a scheduled refresh fails, **the last good snapshot stays active**.

---

## 11. Community reports as data (Stage 4 preview)

- Reports are stored in `reports_report` with `data_origin = COMMUNITY`. They are never mixed with `VERIFIED_SOURCE` records.
- They influence scoring **only after** credibility and moderation mechanisms are live, and only through the separate `community_reports` factor with a capped weight.
- Only reports that pass the credibility rules count toward community-data coverage.
- Reports work throughout India from day one. Coverage grows wherever users contribute.

---

## 12. Open data questions (Stage 2)

1. Which boundary source to use for `geo_admin_area` (licence, currency, official codes)?
2. Is any verified geolocated incident source available anywhere in India? Each one is evaluated individually.
3. Are there openly licensed official registries for hospitals and police stations with coordinates?
4. What is OSM completeness (POIs, `lit`, `sidewalk`) across the test regions? This sets the infrastructure `Vol` thresholds.
5. Are there municipal or smart-city streetlight datasets, and what are their licences?
6. How should the NCRB aggregate tables be modelled (indicator names, period, city vs district units)? This is storage and display only; scoring use is disabled.
