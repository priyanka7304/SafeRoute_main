# SafeRoute — Project Plan

**SafeRoute — Your Path, Your Safety.** A safety-first navigation web application. It retrieves multiple **real road routes** between two places and recommends the **safest practical** one. Every score is shown together with how much evidence supports it.

> **SafeRoute is designed for India-wide operation.** No city or state is hard-coded anywhere in the core system.
> **Status:** Stage 0a (architecture) approved with the decisions in §6. Stage 0b (environment) has not started and is waiting for the owner's go-ahead and the blockers in §7.
> Source-of-truth documents: this file, [ARCHITECTURE.md](ARCHITECTURE.md), [DATA_STRATEGY.md](DATA_STRATEGY.md), [README.md](README.md).

---

## 1. Working rules

1. Build in stages. Each stage builds on the previous working stage and preserves existing functionality.
2. Each stage is independently testable, ships with tests, and is **verified by the owner** before the next one starts.
3. **No stage starts without explicit owner instruction.**
4. Each stage ends with documentation updates (this plan, the decision log, and the data strategy where relevant).
5. Architectural decisions are documented before or with their implementation.
6. Never: fake keys, hard-coded secrets, invented APIs or datasets, unverified data presented as real, straight-line route geometry, aggregated statistics turned into street-level points, or confident-looking scores without evidence.
7. **Out of scope permanently:** CCTV analytics, facial recognition, surveillance functionality, OpenVINO.

---

## 2. Where each architecture topic lives

| # | Topic | Location |
|---|-------|----------|
| — | Geographic scope & four coverage types | ARCHITECTURE §0 |
| 1 | System architecture | ARCHITECTURE §1 |
| 2 | Frontend architecture (incl. TS types) | ARCHITECTURE §2 |
| 3 | Backend architecture | ARCHITECTURE §3 |
| 4 | PostgreSQL design (India hierarchy, provenance) | ARCHITECTURE §4 |
| 5 | Geospatial strategy (corridor/bbox/radius, sparse H3) | ARCHITECTURE §5 |
| 6 | REST API | ARCHITECTURE §6 |
| 7 | Authentication (public search) | ARCHITECTURE §7 |
| 8 | Dataset architecture | ARCHITECTURE §8, DATA_STRATEGY §3–7, §10 |
| 9 | Safety score, coverage and confidence | ARCHITECTURE §9 |
| 10 | ML | ARCHITECTURE §10, DATA_STRATEGY §8 |
| 11 | Google Maps / routing (+ terms) | ARCHITECTURE §11 |
| 12 | Security & privacy | ARCHITECTURE §12 |
| 13 | Environment variables | ARCHITECTURE §13 |
| 14 | Testing | ARCHITECTURE §14 |
| 15 | Deployment (native Windows dev) | ARCHITECTURE §15 |
| 16 | Folder structure | ARCHITECTURE §16 |
| 17 | Roadmap | this document §4 |

---

## 3. Status

| Stage | Name | Status |
|-------|------|--------|
| 0 | Architecture & environment | 0a docs: **approved**; 0b environment: not started |
| 1 | Basic working MVP | not started |
| 2 | India-wide dataset pipeline | not started |
| 3 | ML safety intelligence | not started |
| 4 | Community reporting | not started |
| 5 | Emergency & guardian functionality | not started |
| 6 | Advanced safety intelligence | not started |
| 7 | Testing, security, optimisation & production deployment | not started |

---

## 4. Roadmap

### Stage 0 — Architecture & environment

**0a (done):** repository inspection; PROJECT_PLAN, ARCHITECTURE, DATA_STRATEGY and README.

**0b (after go-ahead):**
- Write `docs/setup-windows.md`: Python 3.12 venv, PostgreSQL + PostGIS (EDB installer + Stack Builder), OSGeo4W GDAL/GEOS, and the `GDAL_LIBRARY_PATH`/`GEOS_LIBRARY_PATH` setup. **No Docker required.**
- Fill `.gitignore`. Add `.editorconfig` and pre-commit hooks (ruff, black, eslint, prettier, gitleaks).
- Django 5.2 skeleton: settings split, `core` (health, readiness, `/config/`), an empty `geo` app with country config loaded from `data/countries/in.yaml`, the custom `accounts.User` migration, and drf-spectacular.
- Vite + React + TypeScript (strict, no-`any` lint) skeleton with Bootstrap, React Router, an Axios instance, and a health page.
- `.env.example` (all variables from ARCHITECTURE §13, no values).
- CI: lint, type check, tests against a PostGIS service, secret scan.

**Tests:** health and readiness (PostGIS extension present), custom user migration, country config loads, geographic-neutrality lint test (no city/state literals in core packages), frontend renders the health page. CI green.
**Exit:** the backend and frontend run natively on Windows against local PostGIS, and the owner verifies.

---

### Stage 1 — Basic working MVP

**Scope**
- **Public route planner** (no login): India-restricted Places Autocomplete, `DRIVING` mode, departure time, map with all alternatives, route cards (distance, ETA, score/status, category, **coverage badge**, factors), recommended route highlighted, grey rendering for `NOT_SCORED`.
- **Routing:** `RouteProvider` abstraction plus the Google Routes implementation (alternatives, timeouts, retries, circuit breaker, **no fallback geometry**), `TravelMode` registry (only DRIVING enabled), anonymous throttles and the daily routing cap.
- **Safety engine v1 (provisional):** adaptive resampling, corridor queries, factor providers (police/hospital proximity, lighting, activity, incident exposure; road characteristics wired but data-less until Stage 2), omission of unavailable factors, aggregation, **coverage engine** (ARCHITECTURE §9.4), display rules, like-for-like recommender, scoring profile `1.0.0-DRIVING` marked PROVISIONAL, `confidence.method = NONE`.
- **Data:** consolidated-dataset models with full provenance columns, the `SourceAdapter` interface, the synthetic adapter, and `seed_synthetic` for **development test regions only**, with the SYNTHETIC banner in the UI.
- **Auth:** register, login, refresh, logout, `/me`. Optional: saved routes and preferences (login).

**Out of scope:** real data ingestion, ML, reports, emergency features.

**Tests:** scoring and coverage maths (property-based), display rules, recommender (like-for-like, δ thresholds, insufficient evidence), provider mocks (errors, cap, no fallback geometry), API contract (public access, throttles), geographic-neutrality scenarios (Delhi → Gurugram, Mumbai → Navi Mumbai, Chandigarh → Mohali, Bengaluru → Electronic City, and a zero-data area giving `NOT_SCORED`), synthetic isolation, and frontend components and flows (MSW). One Playwright smoke test.
**Exit:** with owner-provided Google keys, routes between places in several Indian regions display correctly, with road-following polylines, honest coverage and a SYNTHETIC banner. The owner verifies.

---

### Stage 2 — India-wide dataset acquisition & processing pipeline

**Scope**
- Source verification and datasheets (DATA_STRATEGY §5), plus a boundary source for `geo_admin_area`.
- Adapters: `osm` (POIs, road features, `lit`), `government` (aggregate tables, **storage and display only**), `lighting` (VIIRS if adopted; a municipal lighting investigation first), `infrastructure` (only for verified registries).
- Pipeline: validate, normalise, dedupe, admin tagging, idempotent load, `source_coverage` claims, snapshots, quality gates, sparse feature grid build parallelised by state, `rebuild_all` (online and offline).
- Celery + a Redis-compatible broker (on Windows: Memurai or WSL Redis, decided here) for scheduled refreshes.
- Recalibrate the coverage thresholds on real coverage distributions across the test regions. Document the results.
- Endpoints `/data/sources/`, `/data/areas/{id}/statistics/`, `/safety/layers/coverage/`. Frontend "About the data" page and coverage overlay (bbox-limited).

**Tests:** validators, taxonomy, dedupe, idempotency, offline reproducibility, admin tagging, **aggregate-never-becomes-incident**, coverage extents (A=0 vs A=1 with zero records), quality gates, bbox limits.
**Exit:** the test regions are scored from real infrastructure data, coverage levels are honest (most areas are expected to be LIMITED/MEDIUM without incident data), provenance is visible, and a rebuild is reproducible. The owner verifies.

---

### Stage 3 — Machine-learning safety intelligence

**Precondition:** decision on blocker B-5 (label source).
**Scope:** feature builder with `as_of`; K-Means hotspots per analysis region (HDBSCAN challenger); Logistic Regression with temporal and spatial validation, calibration, baselines and bootstrap intervals; applicability domain; model registry; manual activation with production guards; profile v2 adding `hotspot_exposure` and `model_risk` (in-domain only); `confidence` block populated; bbox-limited hotspot/cell layers.
**Tests:** leakage suite, reproducibility, baseline gate, applicability domain (out-of-domain means unavailable), registry and activation guards.
**Exit:** metrics documented; the owner approves activation (or confirms the pipeline-only outcome if no real labels exist).

---

### Stage 4 — Community reporting (India-wide)
- Logged-in users submit reports anywhere in India: category, location (stored fuzzed), time, text.
- Moderation queue, credibility scoring (corroboration, reporter history), expiry, abuse limits.
- **Reports do not affect scoring until the credibility gates pass.** After that they feed the separate, weight-capped `community_reports` factor and community-data coverage. They stay distinguishable (`data_origin = COMMUNITY`).
- **Tests:** permissions and IDOR, rate limits, credibility rules, no influence before verification, provenance preserved.

### Stage 5 — Emergency & guardian functionality
- Emergency contacts (with consent), SOS button, trips started from a chosen route, location pings (partitioned, with retention), Guardian Mode live view (Channels, expiring and revocable share links), automatic safety check-ins.
- `Notifier` interface; SMS/email/push provider chosen at this stage (A-8).
- DPDP Act 2023 compliance review before release.
- **Tests:** SOS delivery audit, notifier fallback, guardian access expiry and revocation, WebSocket auth, retention jobs.

### Stage 6 — Advanced safety intelligence
- **Time-of-day-aware scoring:** richer per-bucket profiles, weekday/weekend split, activity patterns by hour where data supports it.
- **Dynamic hotspot updates:** scheduled re-clustering with stability tracking and versioned hotspot sets.
- **Improved route-risk prediction:** model refinements and challengers evaluated against the Stage 3 baselines (adopted only if better and approved).
- **Route deviation detection:** during a trip, distance from the chosen route (held in session memory, not persisted) triggers alerts and check-ins.
- **Voice SOS** where the web platform realistically supports it (Web Speech API in supporting browsers, active only while the app is open and foregrounded). Limitations are documented and a manual fallback is always available.
- **Emergency bypass / discreet emergency interface:** a quick-trigger, low-visibility SOS flow that does not weaken authentication or the audit trail.
- **Improved safety/distance/time trade-offs:** user-adjustable preference, Pareto presentation, and waypoint-based candidate expansion (cost-gated).
- **Performance optimisation:** Redis caching of cell features and corridor POIs, query tuning, table partitioning if volumes require it.
- **Excluded:** OpenVINO, CCTV analytics, facial recognition, surveillance functionality.

### Stage 7 — Testing, security, optimisation & production deployment
- Full E2E suite, Locust load tests, OWASP ZAP baseline, CSP hardening, staff MFA, bot mitigation for anonymous endpoints if needed.
- Production infrastructure (India region preferred), backups, monitoring, runbook, retention jobs.
- Legal reviews completed (Google terms, DPDP, boundary depiction).
- **Exit:** production checklist signed off by the owner.

---

## 5. Assumptions

1. **Production scope is India.** The country is configured (`SAFEROUTE_COUNTRY_CODE=IN`), so other countries remain possible later.
2. Google provides road routing across India. Availability of modes other than DRIVING varies and will be verified when each is enabled.
3. Safety-data coverage will be **uneven across India**. Many routes will be LIMITED or INSUFFICIENT until more verified sources or community reports exist, and the product states this openly.
4. There is a single developer/owner initially, and a web app only (responsive).
5. The owner provides the Google Cloud project, billing and keys.
6. Python 3.12 and Node 24 are used, both already installed. git is installed but not on the PowerShell PATH (it works from Git Bash).
7. The Safety Score is a provisional relative index, not a probability of being safe.

---

## 6. Owner decisions (resolved)

| ID | Decision |
|----|----------|
| A-1 | Stage 6 = **Advanced Safety Intelligence** (scope in §4). Excludes OpenVINO, CCTV analytics, facial recognition and surveillance. |
| A-2 | **India-wide** scope. No hard-coded cities or states. Crime/incident data is optional per area. Aggregates are never converted to street-level data. Chandigarh Tricity is **only a development test region**. |
| A-3 | **Native Windows** local development (venv, PostgreSQL + PostGIS, OSGeo4W). Docker optional later. |
| A-4 | Stage 1 uses clearly labelled SYNTHETIC development data. It is never presented as real. |
| A-5 | **Public route search.** Login only for personal features (reports, contacts, Guardian Mode, saved routes and preferences). |
| A-6 | **React + TypeScript** (strict, typed interfaces, no `any` unless justified). |
| A-7 | Pipeline supports VIIRS as a documented **proxy**. Stage 1 does not depend on it. Manual Earthdata steps are documented (DATA_STRATEGY §5.4). Local lighting data is investigated first in Stage 2. |
| A-9 | `readme.m` renamed to **README.md**. |
| A-11 | Practicality limits (≤ 30% and ≤ 20 min longer than the fastest route) are documented as **provisional**. |
| A-12 | Factor weights, severities, coverage thresholds and δ are **provisional methodology**, not established facts. They are revisited after the Stage 2 and 3 analysis. |
| A-13 | MVP travel mode **DRIVING**. WALKING, TWO_WHEELER and BICYCLING can be added through the `TravelMode` registry. |
| A-14 | Google terms considerations are documented (ARCHITECTURE §11.5). The owner confirms the terms when setting up the account. |
| Deferred | A-8 notification provider (Stage 5); A-10 hosting provider (before Stage 7). |

---

## 7. Remaining blockers / owner inputs

| ID | Needed before | Question |
|----|---------------|----------|
| **B-1** | Stage 0b | Local installs: PostgreSQL + PostGIS (EDB installer + Stack Builder) and OSGeo4W are **not installed**. Will you install them following the guide I write, or should I attempt command-line installs (e.g. `winget`)? PostGIS through Stack Builder is interactive. |
| **B-2** | Stage 1 verification | Google Cloud project with billing; a **browser key** (referrer-restricted: Maps JS + Places New) and a **server key** (Routes API). The Routes API must be enabled. Keys go only in your local `.env` files. |
| **B-3** | Stage 1 | Anonymous cost guards: proposed **30 route searches/hour per IP** and a **global cap of 1,000 Routes API calls/day** for development. What monthly Google budget should the production cap reflect? |
| **B-4** | Stage 1 | Confirm the **development test regions**: Chandigarh Tricity, Delhi–Gurugram, Mumbai–Navi Mumbai, Bengaluru–Electronic City, plus one low-data area. |
| **B-5** | Stage 3 | ML label source. Without a verified geolocated incident source in India, can Stage 3 end as "pipeline built and tested, not activated", with activation later once verified sources or sufficient community reports exist? *(Optionally, a public geolocated dataset from outside India could be used purely as a **methodology benchmark**, never for Indian scores. This needs your approval.)* |
| **B-6** | Before production | Legal and compliance reviews: Google Maps Platform terms, the DPDP Act 2023 and its rules, and India map-boundary depiction rules. Not blocking now. |

---

## 8. External APIs and services

| Service | Stage | Account/key | Cost |
|---------|-------|-------------|------|
| Google Maps JavaScript API | 1 | browser key | paid beyond free usage |
| Google Places API (New) Autocomplete | 1 | browser key | paid |
| Google Routes API | 1 | server key | paid |
| Google Geocoding API (optional) | 1 | server key | paid |
| OpenStreetMap via Geofabrik | 2 | none | free (ODbL attribution) |
| data.gov.in (if API access is used) | 2 | owner-registered API key | free |
| State/city open-data portals | 2+ | per portal | free (verify) |
| NASA Earthdata (VIIRS), if adopted | 2 | free Earthdata login + token (manual) | free |
| PostgreSQL + PostGIS | 0 local / 7 managed | — / hosting | free / paid |
| Redis-compatible broker | 2 local / 7 managed | — / hosting | free / paid |
| Object storage | 2/7 | hosting | low |
| SMS/email/push provider | 5 | TBD (A-8) | paid |
| Error tracking (optional) | 7 | TBD | free tier / paid |
| GitHub Actions | 0 | existing repo `priyanka7304/SafeRoute_main` | free tier |

---

## 9. Key risks

| Risk | Mitigation |
|------|-----------|
| No geolocated incident data in most of India | Coverage model; honest LIMITED/INSUFFICIENT; community reports; per-source onboarding |
| Uneven OSM completeness | Per-area completeness checks feed the `Vol` component; coverage is lowered, not faked |
| Users over-trusting scores | Coverage badges, provisional notice, grey "not scored" routes, never "safe" wording |
| Routes crossing areas with different coverage | Like-for-like comparison on shared factors |
| Google billing from anonymous use | Throttles, daily cap, minimal field masks, budget alerts |
| Reporting bias in official statistics | Aggregates excluded from scoring by default; documented caveats |
| Community-report bias and abuse | Credibility gates, moderation, capped weight |
| India-wide data volume | Sparse grid, corridor/bbox queries, per-state parallel builds, partitioning |
| GeoDjango on native Windows | OSGeo4W guide; CI on Linux catches environment drift |
| Privacy of location data | No storage of anonymous searches, opt-in history, retention limits, DPDP review |

---

## 10. Definition of done (every stage)

- [ ] Scope implemented; earlier stages still work (full suite green)
- [ ] New tests for the stage
- [ ] Invariants I-1 … I-11 (ARCHITECTURE) hold
- [ ] No secrets in the diff
- [ ] Docs updated (status, decision log, data strategy)
- [ ] Owner verified the stage and authorised the next one
