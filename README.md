<div align="center">

# TraceNet

### One city. Every camera. One connected picture of how traffic moves.

**Smart India Hackathon 2026 — Software Edition**

*Track · Analyze · Connect*

</div>

### City-Wide AI Engine for Multi-Camera ANPR Trajectory Tracking and Urban Traffic Analytics

> **We don't match vehicles on plates alone — TraceNet fuses Plate + Appearance + Road Physics into one Global Vehicle ID across the city.**

| Smart India Hackathon 2026 | |
| --- | --- |
| **Problem Statement ID** | 26127 |
| **Problem Statement title** | City-Wide AI Engine for Multi-Camera ANPR Trajectory Tracking and Urban Traffic Analytics |
| **Theme** | Smart Automation |
| **PS Category** | Software |
| **Team ID** | 176689 |
| **Team name** | Trace Force_A0 |
| **Live deployment** | <https://tracenet-ten.vercel.app/> |
| **Demo video** | <https://youtu.be/lc2fId89qgc> |
| **GitHub** | <https://github.com/KoduruChandrasekhar/TraceNet> |

---

## Contents

> 📚 **More documentation:** [docs/](docs/README.md) — [Judges' guide](docs/JUDGES_GUIDE.md) · [Architecture](docs/ARCHITECTURE.md) · [Fusion algorithm](docs/FUSION_ALGORITHM.md) · [ANPR pipeline](docs/ANPR_PIPELINE.md) · [Privacy & compliance](docs/PRIVACY_AND_COMPLIANCE.md) · [FAQ](docs/FAQ.md)

1. [Executive summary](#1-executive-summary)
2. [The problem](#2-the-problem)
3. [Our solution — four modules](#3-our-solution)
4. [How TraceNet maps to the SIH evaluation criteria](#4-how-tracenet-maps-to-the-sih-evaluation-criteria)
5. [System architecture](#5-system-architecture)
6. [The AI pipeline, stage by stage](#6-the-ai-pipeline-stage-by-stage)
7. [Measured results and design targets](#7-measured-results-and-design-targets)
8. [Platform tour (the web app)](#8-platform-tour-the-web-app)
9. [Security, privacy and responsible use](#9-security-privacy-and-responsible-use)
10. [Technology stack](#10-technology-stack)
11. [Repository layout](#11-repository-layout)
12. [Getting started](#12-getting-started)
13. [API reference](#13-api-reference)
14. [Data model](#14-data-model)
15. [Testing](#15-testing)
16. [Deployment](#16-deployment)
17. [Feasibility, scalability and cost](#17-feasibility-scalability-and-cost)
18. [Impact and beneficiaries](#18-impact-and-beneficiaries)
19. [Risks, mitigations and honest status](#19-risks-mitigations-and-honest-status)
20. [Roadmap](#20-roadmap)
21. [Research, references and licences](#21-research-references-and-licences)

---

## 1. Executive summary

Indian cities already have thousands of CCTV and traffic cameras, but each one works **in isolation**. Following one
vehicle across a city still means a person scrubbing through hours of footage, camera by camera.

**TraceNet is the layer that connects them.** It reads video from many cameras, detects and tracks vehicles,
reads their number plates (ANPR), and then **fuses sightings from different cameras into a single journey per vehicle**
— using plate text, visual appearance (Re-ID) and road-network physics together. On top of that it provides a live
command-centre web app: a city-wide traffic picture, vehicle journey tracking, movement analytics and real-time
alerts for cloned plates, watch-listed vehicles and tampered plates.

| | |
| --- | --- |
| **Input** | Video from existing cameras (RTSP / recorded MP4) |
| **Output** | Vehicle journeys, traffic analytics, live alerts, audit trail |
| **Differentiator** | Multi-signal cross-camera association (MTMC): `S = α·PlateSim + β·cos(Eₐ,E_b) + γ·PhysSim`. Keeps tracking when OCR fails (α → 0, Re-ID + physics fallback) and flags **cloned plates** when the required speed `v_req = d_road/Δt` exceeds `V_max` (150 km/h) |
| **Problem Statement** | PS 26127 — Smart Automation / Software |
| **Runs on** | Commodity hardware: CPU-only OCR at ~60 ms/plate; the API runs in ~110 MB RAM |
| **Deployable today** | Vercel (frontend) + Render (API) + Supabase (PostGIS) — all free tiers; no new hardware |

---

## 2. The problem

| Where it hurts | What happens today |
| --- | --- |
| **Following a vehicle** | Investigators manually stitch a journey together from separate camera systems; takes hours to days. |
| **Understanding traffic** | Every junction reports its own numbers; nobody sees congestion form and spread as one system. |
| **Responding to incidents** | A flagged vehicle is noticed only if the right person is watching the right screen at the right moment. |
| **Plate fraud** | Cloned (duplicate) and tampered number plates go undetected because no one compares sightings *across* cameras. |
| **Planning** | Signals, lanes and routes are planned from sampled counts, not from how people actually move. |
| **Operations** | No single place shows which cameras are healthy, degraded or offline. |

India-specific difficulties that off-the-shelf tools handle poorly: dense mixed traffic (two-wheelers, auto-rickshaws,
buses, LCVs), varied plate fonts and formats (including BH-series), low light and glare, and low-resolution CCTV.

---

## 3. Our solution

TraceNet is built on three ideas:

1. **Think in networks, not cameras.** Every camera is a node; an observation only becomes meaningful next to what
   neighbouring cameras saw.
2. **Think in journeys, not snapshots.** A sequence of sightings across the city is a *journey* — what investigators,
   planners and traffic controllers actually need.
3. **Put everything on one real-time map.** Vehicles, congestion, incidents and alerts on one shared view.

### The four modules (as proposed in our SIH submission)

| Module | Name | What it delivers |
| --- | --- | --- |
| **1** | **High-precision ANPR / OCR** | Reads plates in blur, glare, rain, night, angled and damaged conditions; multi-lane streams; best-frame selection; multi-frame OCR; HSRP grammar. **Target: >90 % plate-read accuracy.** |
| **2** | **Trajectory reconstruction** | Follows one vehicle city-wide: query-based tracking, movement history, timestamps, direction and route on a GIS map, Vehicle Re-ID. |
| **3** | **City traffic analytics** | Real-time heatmaps, traffic density, average speed (road distance ÷ time between cameras), origin–destination patterns, congestion bottlenecks. |
| **4** | **Real-time alert system** | Blacklisted vehicles, suspicious route anomalies, cloned plates — pushed live over WebSocket. |

**Core flow:** `DETECT → OCR → RE-ID → FUSION → GVID → TRAJECTORY → ANALYTICS / ALERTS` (plate unreadable → α → 0 fallback).

**Association rule:** `S = α·PlateSim + β·cos(Eₐ,E_b) + γ·PhysSim`, `α+β+γ = 1`, associate when `S ≥ τ`.
`PlateSim = 1 − Lev/max(|Pa|,|Pb|)`; `PhysSim` from `v_req = d_road/Δt`; all terms in [0,1]. No perfect OCR is required.
In the prototype `α, β, γ = 0.35, 0.40, 0.25` and `τ = 0.70` (`backend/config/fusion.json`).

### Why TraceNet when other solutions exist

| Existing approach | Limitation | TraceNet |
| --- | --- | --- |
| Standalone ANPR (OpenALPR-class) | One camera, no memory | One identity across all cameras |
| Junction ITMS / e-Challan systems | Single-junction snapshots | City-wide O-D flows, congestion and heatmaps |
| Plate-only exact-string matching | Fails on blurred or cloned plates | Tracks without a plate; flags clone candidates |
| Manual CCTV review | Hours of footage per case | Partial-plate search → route (design target < 5 s) |
| MTMC research (CityFlow / AI City) | Research code, no operator product | Appearance association **plus** road-physics clone detection **plus** RBAC, audit log and alerts |

### Core capabilities

| Capability | What it does |
| --- | --- |
| **Vehicle detection & tracking** | YOLO11 + ByteTrack per camera; Indian vehicle classes (two-wheeler, auto-rickshaw, hatchback, SUV, bus, LCV, truck…). |
| **ANPR for Indian plates** | Plate localisation → quality gating → CLAHE enhancement → PaddleOCR → positional character correction → state-code validation → multi-frame consensus voting. |
| **Cross-camera fusion** | Weighted match of *text similarity* (Levenshtein) + *visual similarity* (512-D Re-ID embeddings) + *kinematic feasibility* (OSRM road distance ÷ elapsed time). |
| **Ghost-car tracking** | When the plate is unreadable (glare, night, occlusion) the vehicle is still re-identified by appearance + physics. |
| **Anti-cloning** | Same confident plate at two cameras with an implied speed > 150 km/h → **CRITICAL `CLONED_PLATE`** alert. |
| **Real-time alerts** | `CLONED_PLATE`, `BLACKLIST_HIT` (watchlist), `INVALID_FORMAT` (tampered plate), pushed over WebSocket. |
| **Spatial analytics** | PostGIS trajectories, heat-maps, corridor speed/volume, origin–destination matrix (Polars jobs). |
| **Tamper-evident audit** | Every journey lookup is logged in a SHA-256 hash-chained table; chain integrity is verifiable via API. |
| **Role-based access** | JWT auth with `camera_admin` and `law_enforcement` roles; admin console for users, cameras and system health. |

---

## 4. How TraceNet maps to the SIH evaluation criteria

| SIH criterion | How TraceNet addresses it | Where to look |
| --- | --- | --- |
| **Innovation & originality** | Three-signal fusion (text + appearance + road physics); plate-less "ghost" tracking; physics-based clone detection; hash-chained audit of surveillance queries. | [§6](#6-the-ai-pipeline-stage-by-stage), `backend/fusion/` |
| **Technical feasibility** | Working end-to-end system: ingestion → detection → ANPR → fusion → PostGIS → API → React app. 13 test modules (≈190 test functions). Runs on CPU. | [§12](#12-getting-started), [§15](#15-testing) |
| **Use of technology** | YOLO11, ByteTrack, PaddleOCR (PP-OCRv5/v6, TensorRT-capable), OSNet-compatible Re-ID, FastAPI, PostgreSQL 16 + PostGIS 3.4, Redis, RabbitMQ, MediaMTX, Polars, React 19, Leaflet. | [§10](#10-technology-stack) |
| **Scalability** | Broker mode (RabbitMQ `q.fusion` → separate fusion workers, Redis state + Pub/Sub) scales horizontally; direct mode for a single node. Cameras are config-driven. | [§17](#17-feasibility-scalability-and-cost) |
| **Economic viability** | Works on existing cameras; no proprietary hardware; whole stack is open source; free-tier deployable. | [§17](#17-feasibility-scalability-and-cost) |
| **Social impact** | Faster investigations, fewer plate-fraud blind spots, data-driven traffic management, safer emergency routing. | [§18](#18-impact-and-beneficiaries) |
| **Security & privacy** | RBAC, JWT, purpose-bound design, immutable audit trail, no face recognition. | [§9](#9-security-privacy-and-responsible-use) |
| **User experience** | Responsive command-centre UI, light/dark themes, command palette (keyboard), reduced-motion support, graceful offline demo mode. | [§8](#8-platform-tour-the-web-app) |
| **Honesty / reproducibility** | Every number in the app's *Pipeline* page comes from the repo's own run outputs; limitations are documented below. | [§7](#7-measured-results-and-design-targets), [§19](#19-risks-mitigations-and-honest-status) |

---

## 5. System architecture

```text
 ┌──────────────┐   RTSP / MP4    ┌─────────────────────────────────────────────────────────┐
 │  CCTV feeds  │ ──────────────► │ 1 INGESTION   backend/ingestion   (per-camera workers)  │
 │ (MediaMTX or │                 └───────────────┬─────────────────────────────────────────┘
 │ recorded)    │                                 ▼ frames
 └──────────────┘                 ┌─────────────────────────────────────────────────────────┐
                                  │ 2 DETECT + TRACK  backend/ai   YOLO11 + ByteTrack       │
                                  └───────────────┬─────────────────────────────────────────┘
                                                  ▼ vehicle tracks
                                  ┌─────────────────────────────────────────────────────────┐
                                  │ 3 ANPR  backend/anpr                                    │
                                  │   plate candidate → Q-score → top-K crops → CLAHE →     │
                                  │   PaddleOCR → validation → multi-frame consensus        │
                                  └───────────────┬─────────────────────────────────────────┘
                                                  ▼ Sightings (plate, conf, 512-D embedding, camera, time)
   POST /api/v1/ingest/sightings                  │
                                  ┌───────────────▼─────────────────────────────────────────┐
                                  │ 4 FUSION  backend/fusion                                │
                                  │   RabbitMQ q.fusion → worker(s) → Redis active window   │
                                  │   score = 0.35·text + 0.40·visual + 0.25·physics        │
                                  │   guards • ghost cars • anti-cloning • Global Vehicle ID│
                                  └───────┬───────────────────────────┬─────────────────────┘
                                          ▼ trajectories              ▼ alerts (Redis Pub/Sub)
                    ┌───────────────────────────────┐     ┌──────────────────────────────┐
                    │ 5 PostgreSQL 16 + PostGIS 3.4 │     │ 6 Alert bus  backend/alerts  │
                    │   observations, trajectories, │     │   watchlist • payloads •     │
                    │   alerts, audit, analytics    │     │   /ws/alerts WebSocket       │
                    └───────────────┬───────────────┘     └───────────────┬──────────────┘
                                    ▼                                     ▼
                    ┌─────────────────────────────────────────────────────────────────────┐
                    │ 7 FastAPI  backend/main.py  (JWT + RBAC, spatial API, Polars jobs)  │
                    └───────────────────────────────┬─────────────────────────────────────┘
                                                    ▼
                    ┌─────────────────────────────────────────────────────────────────────┐
                    │ 8 React 19 + Vite web app — Home · Dashboard · Cameras · Tracking · │
                    │   Traffic · How it works · Alerts · Admin                           │
                    └─────────────────────────────────────────────────────────────────────┘
```

### Two runtime modes (so it works on a laptop *and* scales)

| Mode | Transport | When |
| --- | --- | --- |
| **Broker** | RabbitMQ → fusion worker(s) → Redis (state + Pub/Sub) | Production / multiple cameras / separate worker process |
| **Direct** | Fusion runs in the API process, in-memory | Laptop without Docker, free-tier hosting |

`TRACENET_FUSION_TRANSPORT=auto` (default) picks broker when both RabbitMQ and Redis answer at start-up, otherwise direct.

### Graceful degradation

The frontend **never breaks when the backend is absent**: with no `VITE_API_BASE_URL` it runs as a self-contained
demo on a bundled dataset; if the API is asleep, pages fall back to demo data after a 3-second timeout and switch to
live data automatically when the API responds.

---

## 6. The AI pipeline, stage by stage

### 6.1 Ingestion — `backend/ingestion/`
Config-driven (`backend/config/cameras.json`). Each camera is an `mp4` (recorded replay with deterministic timestamps
`ts = replay_start_time + frame_index / fps`) or `rtsp` source behind the same interface. Camera start/stop and health
are exposed over the API.

### 6.2 Detection and tracking — `backend/ai/vehicle_tracker.py`
YOLO11 detection with ByteTrack (`backend/config/bytetrack.yaml`) for stable per-camera track IDs. Writes
`<camera>_detections.json` and an H.264 annotated video for the browser.

**Regional fine-tuning** (`backend/training/yolo/`): builds a YOLO dataset from IISc **UVH-26** (Indian traffic), with
sampling-based class weighting for two-wheelers and auto-rickshaws, an optional **P2 (stride-4) small-object head**
(`yolo11-p2.yaml`) and post-training NMS tuning (`tune_nms.py`).

### 6.3 ANPR — `backend/anpr/`

| Step | Purpose |
| --- | --- |
| Plate candidate search | Lower-vehicle search region, aspect-ratio / rectangularity / skew / character-transition filters |
| **Q-score (no OCR)** | Weighted area + sharpness + exposure + skew; unreadable crops are dropped *before* OCR |
| Top-K crops per track | Only the best K (default 3) crops are read; early-exit when Q ≥ 0.70 |
| CLAHE enhancement | Contrast boost for dusk/glare (bilateral fallback) |
| **Hybrid OCR** | Fast PP-OCRv5-mobile recognition first; escalates to PP-OCRv6-medium only if confidence/length is low |
| Device plan | GPU + TensorRT FP16 → GPU (CUDA) → CPU oneDNN; each rung must pass a real inference before acceptance |
| **Positional validation** | Fixes OCR confusions only where the format expects the other character class (digit↔letter), never blindly |
| Format + state check | `STANDARD` (`TS09AB4521`, `DL8CAK0211`) and `BH` (`22BH1234AA`); real state/UT code required (incl. legacy + `TG`) |
| Watermark rejection | Stock-footage text (Getty, iStock…) is rejected |
| **Consensus** | Multi-frame voting yields one plate + confidence per vehicle |

### 6.4 Cross-camera fusion — `backend/fusion/fusion_engine.py`

```text
score = w_text · text_sim  +  w_visual · visual_sim  +  w_kinematic · physics_ok
        0.35                   0.40                      0.25            (match_threshold = 0.70)
```

| Signal | How it is computed |
| --- | --- |
| `text_sim` | Normalised Levenshtein distance between plates |
| `visual_sim` | Cosine similarity of 512-D Re-ID embeddings (OSNet/VeRi-776 optional; zero-norm safe) |
| `physics_ok` | Directed **OSRM road distance** between cameras ÷ time gap vs 120–150 km/h (fallback: haversine × 1.35) |

**Hard guards** (on top of the score): kinematically impossible links are never made; one camera cannot see one vehicle as
two concurrent transits; two confident plates must agree (≥ 0.80); plate-less links need visual ≥ 0.90; a plate-less
re-link at the *same* camera must follow within 15 s (tracker fragment, not a new visit).

**Ghost cars** — unreadable plate (OCR confidence < 0.70): text weight drops to 0, visual weight rises to 0.75.

**Anti-cloning** — same plate (text_sim ≥ 0.90) at different cameras with implied speed > 150 km/h →
`CLONED_PLATE` / CRITICAL; the two sightings are *not* merged.

**Identity** — `Global Vehicle ID = uuid5(plate)` (or `uuid5("ghost:…")`), deterministic across restarts. A 30-minute active
window (Redis) bounds candidate search to cameras within a 50 km radius; completed vehicles become trajectories.

### 6.5 Alerts — `backend/alerts/`
| Alert | Severity | Trigger |
| --- | --- | --- |
| `CLONED_PLATE` | Critical | Same plate, different cameras, physically impossible speed |
| `BLACKLIST_HIT` | High | Plate on the (database-backed) watchlist |
| `INVALID_FORMAT` | Medium | Plate fails Indian-format / state-code validation (possible tampering) |

Delivered to the browser over `/ws/alerts` (JWT in `?token=`), with a REST feed for history.

### 6.6 Spatial store and analytics — `backend/api/spatial.py`, `backend/analytics/`
PostGIS stores observations as geography points and trajectories as line strings (GiST-indexed). A **Polars** job
(runs every 45 s by default, or via `POST /api/v1/analytics/refresh`) computes per-leg speeds, corridor volume/speed
and a congestion index, plus the origin–destination matrix. It reads via Arrow
(connectorx → ADBC → psycopg fallback).

---

## 7. Measured results and design targets

All figures are produced from this repository's own run outputs (see `src/data/pipeline.js`, `src/data/fieldTests.js`;
shown live on the app's **How it works** page). Hardware: development laptop, **CPU only**.

### OCR speed (per plate crop)

| Read path | Median latency | Plates kept | Exact reads (60-crop test set) |
| --- | --- | --- | --- |
| Baseline: full detection + recognition | 1,850 ms | 45 / 45 | 59 / 60 |
| Accurate: medium-rec hybrid | 295 ms | 44 / 45 | 59 / 60 |
| **Default: mobile-rec + oneDNN hybrid** | **≈ 60 ms** | **45 / 45** | **58 / 60** |

**≈ 30× faster than baseline** with 1 exact-read lost out of 60 — fast enough for real-time CPU-only edge deployment.

### Worked example (real read, CAM-401, two-wheeler)
Three frames voted to `AP09AZ6596` at **94.2 %** confidence (per-frame reads 95.5 / 92.6 / 94.6 %), parsed as
State `AP` · District `09` · Series `AZ` · Number `6596`.

### Field tests
Four recorded Indian traffic clips (768 × 432): flyover junction, rush-hour jam, and others — each reports a funnel
*frames → vehicles tracked → plate located → clear enough to read → plates validated by vote*, vehicle-class mix and
the validated plates with evidence crops. These show where the pipeline loses yield on low-resolution footage (see
[§19](#19-risks-mitigations-and-honest-status)).

### Synthetic ground-truth benchmark
`backend/scripts/generate_dataset.py` + `evaluate_dataset.py` build a multi-camera dataset with **exact ground truth**
(1920×1080 H.264, 6 cameras, seeded) containing every scenario role — `normal`, `ghost`, `clone_a/clone_b`,
`blacklist`, `tampered` — and score fusion (and, with `--e2e`, the whole vision stack) against it. This lets anyone
reproduce precision/recall of journey linking and alert detection without private data.

### Prototype demonstration (seeded demo trace, Module 2)

Plate `TS09AB4521` → GVID `GV-00157`. *Values from one prototype demo trace on seeded data — not a field measurement.*

| Cameras linked | Route reconstructed | Travel time | Average speed | OCR match (seeded) | Detour |
| --- | --- | --- | --- | --- | --- |
| 4 | 19.2 km | 37 min | 31 km/h (road km ÷ time) | 98.7 % | +11 km flagged |

Camera hops: CAM-401 Kukatpally → CAM-402 JNTU → CAM-403 Balanagar → CAM-406 Madhapur, drawn as a timed polyline on
React-Leaflet + OSM with expected-vs-actual route analysis and a playback timeline.

### Design targets (goals — **not yet measured**)

| Target | Value |
| --- | --- |
| Plate-read accuracy | > 90 % (to be validated on clean vs degraded splits) |
| Route lookup latency | < 5 s |
| Heatmap, speed and density refresh | 60 s |
| O-D matrix update | Hourly |
| Alert push latency | < 100 ms |

### Comparison with related systems

| Capability | TraceNet | Standalone ANPR | Junction ITMS / e-Challan | MTMC research |
| --- | :-: | :-: | :-: | :-: |
| Plate OCR in degraded conditions (best-frame + multi-frame + HSRP grammar) | ✅ | ◐ | ◐ | ✗ |
| Cross-camera fusion → one GVID | ✅ | ✗ | ✗ | ◐ |
| Tracking when plate is unreadable (Re-ID fallback, α → 0) | ✅ | ✗ | ✗ | ✅ |
| GIS trajectory reconstruction | ✅ | ✗ | ✗ | ✗ |
| O-D matrices, congestion index, real-time heatmaps | ✅ | ✗ | ◐ | ✗ |
| Real-time blacklist and cloned-plate alerts | ✅ | ◐ | ◐ | ✗ |

✅ supported · ◐ partial · ✗ not supported (as characterised in our submission).

### Footprint
API process ≈ **110 MB RSS**; ML stack is isolated to offline/edge workers.

---

## 8. Platform tour (the web app)

| Page | Question it answers |
| --- | --- |
| 🏠 **Home** | What is the network? Animated camera network, guided single-journey story, interactive city map. |
| 📊 **Dashboard** | What is happening right now? Network speed, congestion, delay, incidents, junction map and day trends. |
| 🎥 **Cameras** | Which cameras are healthy? Camera wall with live/degraded/offline status, hover previews, HLS live feed when MediaMTX is up (falls back to MP4). |
| 🧭 **Tracking** | Where did this vehicle go? Plate search → journey on map + timeline, camera hand-off, Re-ID evidence and parallel-processing view. |
| 🚦 **Traffic** | How is the city moving? Corridors, heat layer, origin–destination flows, and optional Google/TomTom **live road traffic** blend. |
| ⚙️ **How it works** | How does a frame become an insight? Pipeline walkthrough, OCR lab, field tests, with real evidence images. |
| 🚨 **Alerts** | What needs attention? Severity-ranked live feed over WebSocket, each alert placed on the map near its cameras. |
| 🔐 **Admin** | Who can do what? Users and roles, camera management, system health, admin event log (`camera_admin` only). |

**UX details:** light/dark themes, responsive layout, `⌘/Ctrl-K` command palette, reduced-motion support, consistent
traffic colour language (green flowing → yellow building → orange heavy → red severe).

---

## 9. Security, privacy and responsible use

A system that follows vehicles must be built with restraint.

| Control | Implementation |
| --- | --- |
| **Authentication** | JWT with issuer and expiry; tokens valid only while the user exists, is active and still holds the role (5 s directory cache). PBKDF2-SHA256 salted password hashes. |
| **Authorisation (RBAC)** | `camera_admin` (cameras, ingestion, users, watchlist) vs `law_enforcement` (search, tracking, alerts); route-level `require_roles`. |
| **Tamper-evident audit** | Every journey query writes `row_hash = sha256(prev_hash | user | plate | ts | reason)` to `query_audit_log` under an advisory lock; `GET /api/v1/audit/verify` detects any edited or deleted row. Each lookup records a free-text reason. |
| **Purpose binding** | Built for traffic management, public safety and planning — plates and vehicles only; **no facial recognition** and no biometric data. |
| **Data minimisation** | Active tracking window is soft state (30 min, Redis); stock-footage and synthetic data used for demos. |
| **Transport / config** | Secrets only via environment (`.env` git-ignored); CORS allow-list; security headers on the frontend (nosniff, frame-options, referrer policy, locked permissions policy). |
| **Demo data** | Everything shown without a backend is **illustrative** and describes no real person or vehicle. |

Production hardening checklist (also in `DEPLOYMENT.md`): change the seeded demo passwords, set `TRACENET_JWT_SECRET`,
disable the Supabase Data API, move WebSocket auth off query strings behind a trusted proxy.

---

## 10. Technology stack

| Layer | Technologies |
| --- | --- |
| **Computer vision** | Ultralytics **YOLO11** (+ optional P2 head), **ByteTrack**, OpenCV, FFmpeg (imageio-ffmpeg) |
| **OCR** | **PaddleOCR 3.x** (PP-OCRv5-mobile / PP-OCRv6-medium), oneDNN, optional TensorRT FP16 |
| **Re-ID** | 512-D appearance embeddings, cosine matching; OSNet / VeRi-776 weights pluggable |
| **Datasets** | IISc **UVH-26** (Indian traffic, CC-BY-4.0), OSRM / OpenStreetMap road distances |
| **Backend** | **FastAPI**, Uvicorn, Pydantic v2, PyJWT, WebSockets |
| **Messaging / state** | **RabbitMQ** (`q.fusion`), **Redis** (active window, Pub/Sub) — both optional |
| **Database** | **PostgreSQL 16 + PostGIS 3.4**, `pg_trgm`, `fuzzystrmatch`; asyncpg + psycopg |
| **Analytics** | **Polars**, PyArrow, connectorx / ADBC |
| **Streaming** | **MediaMTX** (RTSP :8554, HLS :8888, WebRTC :8889), hls.js |
| **Frontend** | **React 19**, Vite 6, Tailwind CSS 4, Leaflet / react-leaflet / leaflet.heat, Recharts, lucide-react |
| **Live traffic (optional)** | Google Maps traffic layer or TomTom Traffic Flow |
| **DevOps** | Docker Compose, Vercel, Render Blueprint, Supabase, pytest |

---

## 11. Repository layout

```text
SIH-2026/
├── index.html                  App entry
├── package.json · vite.config.js · vercel.json
├── src/                        React frontend
│   ├── pages/                  Home, Dashboard, Cameras, Tracking, Traffic, Pipeline, Alerts, Admin, Login
│   ├── components/             layout · ui · home · map · camera · tracking · traffic · pipeline · anpr
│   ├── context/                Theme + Alerts (WebSocket) state
│   ├── hooks/                  session, motion, text/interaction effects
│   ├── lib/                    api, auth, config, streams, live traffic, Google Maps, simulator
│   ├── data/                   demo network, city graph, pipeline + field-test data
│   └── styles/                 global styles and design tokens
├── public/                     camera clips, annotated videos, hero media, pipeline evidence
├── backend/
│   ├── main.py                 FastAPI app and core routes
│   ├── api/                    auth · users (RBAC) · spatial · routes_analytics · ws_alerts · streams · audit · db
│   ├── ingestion/              camera sources, per-camera workers, manager, CLI
│   ├── ai/                     vehicle tracker, plate detector, OCR engine
│   ├── anpr/                   preprocessing, quality, recognizer, validation, consensus, pipeline, store
│   ├── fusion/                 engine, Re-ID matcher, road network, broker, worker, Redis state, PostGIS writer
│   ├── alerts/                 live service, bus, watchlist, payloads, ingest pipeline
│   ├── analytics/              Polars jobs + scheduler
│   ├── db/                     config + idempotent migrator
│   ├── config/                 cameras · anpr · fusion · road_network · bytetrack
│   ├── training/yolo/          UVH-26 prep, regional fine-tuning, NMS tuning
│   ├── scripts/                demo runner, dataset generation/evaluation, benchmarks, DB seed
│   └── tests/                  13 test modules
├── db/schema.sql               PostGIS schema (idempotent)
├── docker-compose.yml          PostGIS · Redis · RabbitMQ · MediaMTX
├── docker/mediamtx.yml         generated stream config
├── render.yaml                 Render Blueprint (API)
├── DEPLOYMENT.md               Vercel + Render + Supabase guide
└── .env.example · backend/.env.example
```

---

## 12. Getting started

### Option A — Frontend only (30 seconds, no backend)

Requires **Node.js 20+**.

```bash
npm install
npm run dev          # http://localhost:5173
```

Runs on built-in demo data and auto-signs-in as the demo officer.

### Option B — Full stack (API + database)

Requires Python 3.12, Docker.

```bash
# 1. infrastructure (database only is enough; add redis/rabbitmq for broker mode)
docker compose up -d postgis                # or: docker compose up -d   (full live pipeline)

# 2. backend
python -m venv venv && source venv/bin/activate
pip install -r backend/requirements-render.txt      # API only (light)
# pip install -r backend/requirements.txt           # + full ML stack (YOLO, PaddleOCR)
cp backend/.env.example backend/.env
python -m backend.db.migrate                         # apply db/schema.sql
python backend/scripts/seed_db.py --migrate          # optional: demo cameras + ~1,500 observations
uvicorn backend.main:app --port 8000                 # docs at http://localhost:8000/docs

# 3. frontend pointed at the API
echo "VITE_API_BASE_URL=http://localhost:8000" > .env.local
npm run dev
```

### Option C — Run the AI pipeline on video

```bash
python backend/scripts/run_vehicle_tracking.py       # detection + tracking → annotated MP4 + JSON
python backend/scripts/run_automated_anpr_benchmark.py
python backend/scripts/run_demo.py                   # end-to-end: sightings → fusion → alerts
python backend/scripts/prepare_streams.py            # write docker/mediamtx.yml for simulated live CCTV
docker compose up -d                                 # PostGIS + Redis + RabbitMQ + MediaMTX
```

Set `TRACENET_INGEST_SOURCE=rtsp` to read cameras from MediaMTX instead of MP4 files.

### Reproduce the ground-truth benchmark

```bash
python backend/scripts/generate_dataset.py --vehicles 16 --seed 26127
python backend/scripts/evaluate_dataset.py            # add --e2e to score the full vision stack
```

### Demo accounts (local/demo only)

| Username | Password | Role |
| --- | --- | --- |
| `admin` | `admin123` | `camera_admin` |
| `officer` | `police123` | `law_enforcement` |

> Change these before any public deployment.

### Scripts

| Command | Purpose |
| --- | --- |
| `npm run dev` | Dev server with hot reload |
| `npm run build` | Production build to `dist/` |
| `npm run preview` | Serve the production build |

### Environment variables (essentials)

| Variable | Purpose |
| --- | --- |
| `VITE_API_BASE_URL` | Backend origin; empty = self-contained demo |
| `VITE_GOOGLE_MAPS_API_KEY` / `VITE_TOMTOM_API_KEY` | Optional live road traffic overlay |
| `TRACENET_DATABASE_URL` | PostgreSQL DSN |
| `TRACENET_JWT_SECRET` | JWT signing key |
| `TRACENET_FUSION_TRANSPORT` | `auto` · `broker` · `direct` |
| `TRACENET_CORS_ORIGINS` | Allowed frontend origins |

Full list: `.env.example`, `backend/.env.example`, `DEPLOYMENT.md`.

---

## 13. API reference

Interactive docs: `GET /docs` (OpenAPI). Highlights:

| Area | Endpoints |
| --- | --- |
| **Auth** | `POST /api/v1/auth/login` · `GET /api/v1/auth/me` |
| **Health** | `GET /api/health` · `GET /api/v1/db/health` · `GET /api/v1/pipeline/status` |
| **Vehicles / journeys** | `GET /api/v1/vehicles/{plate}/trajectory` · `GET /api/v1/trajectories[/{id}]` · `GET /api/v1/search` · `GET /api/v1/geo/observations` |
| **Ingestion (sightings)** | `POST /api/v1/ingest/sightings` · `POST /api/v1/ingest/reset` |
| **Cameras & streams** | `GET /api/cameras` · `GET /api/ingestion/*` · `POST /api/ingestion/{start,stop}` · `GET /api/v1/streams` |
| **ANPR evidence** | `GET /api/anpr` · `GET /api/anpr/{track_id}` · `GET /api/cameras/{id}/anpr` |
| **Alerts** | `GET /api/v1/alerts/live` · `GET /api/v1/alerts/anomalies` · `GET /api/v1/alerts/bus` · **`WS /ws/alerts`** |
| **Watchlist** | `GET/POST /api/v1/watchlist` · `DELETE /api/v1/watchlist/{plate}` |
| **Analytics** | `GET /api/v1/geo/heatmap` · `/analytics/{summary,hourly,od-matrix,runs}` · `POST /analytics/refresh` |
| **Audit** | `GET /api/v1/audit/verify` |
| **Admin** | `GET/POST /api/v1/admin/users` · `PATCH /api/v1/admin/users/{username}` · `/admin/permissions` · `/admin/events` |

Sighting payloads can be posted by any edge device, so heavy ML can run on a GPU box while the API stays tiny.

---

## 14. Data model

PostgreSQL 16 + PostGIS 3.4 (`db/schema.sql`, idempotent, ULID keys):

| Table | Holds |
| --- | --- |
| `cameras` | Camera registry with geography point, road, sector, direction |
| `vehicle_observations` | One row per sighting: plate, confidence, class, embedding, time, location |
| `global_trajectories` | Fused journeys as chronological waypoints + line geometry |
| `anomaly_alerts` | Cloned-plate / blacklist / invalid-format alerts |
| `query_audit_log` | **Hash-chained** record of every journey lookup |
| `corridor_stats` · `od_matrix` · `analytics_runs` | Polars outputs: speed, volume, congestion index, origin–destination flows |
| `watchlist` | Watch-listed plates |
| `users` · `admin_events` | Accounts/roles and administrative activity |

Extensions: `postgis`, `pg_trgm` (fuzzy plate search), `fuzzystrmatch`.

---

## 15. Testing

```bash
pip install -r backend/requirements.txt
pytest backend/tests
```

13 modules (≈190 test functions) covering: ingestion, ANPR and validation fixes, cross-camera fusion, PostGIS
persistence, Polars analytics, real-time alerts/WebSocket, broker pipeline, RBAC, live video, synthetic dataset, and
upgrade regressions. Redis is faked in tests (`fakeredis`), so most run without Docker.

---

## 16. Deployment

```text
GitHub ─┬─► Vercel    React frontend (static, vercel.json)
        ├─► Render    FastAPI API   (render.yaml, 1 instance / 1 worker)
        └─► Supabase  PostgreSQL + PostGIS (db/schema.sql)
```

Step-by-step instructions, environment variables, deployment order, security notes and free-tier limits are in
**[DEPLOYMENT.md](DEPLOYMENT.md)**. Heavy ML (detection/OCR) runs offline or on an edge/GPU machine and posts
sightings to the deployed API.

---

## 17. Feasibility, scalability and cost

**Feasibility**
- Uses cameras cities already own — no new sensors.
- Per-plate OCR ≈ 60 ms on CPU; GPU/TensorRT paths are auto-selected when available.
- Edge/central split: lightweight API + database in the cloud, ML workers near the cameras.

**Scalability**
- Cameras are configuration, not code; adding a camera is a JSON entry plus a stream.
- RabbitMQ decouples ingest from fusion; fusion workers scale horizontally; Redis keeps the 30-minute active window.
- PostGIS GiST indexes keep spatial queries fast; Polars jobs aggregate in columnar form.
- Candidate search is bounded by time window *and* geographic radius, so cost grows with local density, not city size.

**Operational fit** — works with existing CCTV/ANPR cameras and standard RTSP streams; one interface for search, playback,
maps and alerts; integrates with ICCC/city GIS through open GeoJSON + REST (OGC WMS/WFS via GeoServer is planned);
role-based access for police and ICCC workflows. Sightings are small JSON metadata, not video, so bandwidth and cloud cost stay low.

**Cost** (reference prototype)
| Item | Prototype |
| --- | --- |
| Frontend hosting | Vercel Hobby — ₹0 |
| API | Render free instance (~110 MB RAM) — ₹0 |
| Database | Supabase free (PostGIS) — ₹0 |
| Software licences | All open-source (Apache-2.0 / MIT / AGPL-aware — see [§21](#21-credits-and-licences)) |
| Hardware | Existing cameras + commodity CPU; GPU optional |

---

## 18. Impact and beneficiaries

| Beneficiary | Benefit |
| --- | --- |
| **Traffic control rooms** | See congestion form and spread across junctions; respond before it cascades. |
| **Law enforcement** | Reconstruct a vehicle's journey in seconds instead of hours; automatic watch-list and cloned-plate alerts. |
| **Transport planners** | Real origin–destination and corridor data instead of sampled surveys. |
| **Emergency services** | Road-condition awareness along a route before committing to it. |
| **Camera operators** | Immediate visibility of degraded or offline cameras. |
| **Citizens** | Reduced plate fraud, faster incident response, better-planned roads. |

### Benefits by dimension

| Dimension | Benefit |
| --- | --- |
| **Social** | Faster response to vehicle crime; safer roads; evidence-based city decisions. |
| **Economic** | Reuses existing CCTV/ANPR; open source, no per-camera licence; fewer manual survey hours. |
| **Environmental** | Bottleneck detection → less idling; expected lower fuel burn and CO₂ (goal); data-driven signal timing. |
| **Governance & security** | JWT + RBAC least privilege; tamper-evident SHA-256 audit log; DPDP-aligned retention and auto-purge (design intent). |
| **Scalability** | Camera-by-camera growth; stateless edge, event-driven core; more RabbitMQ consumers to scale out. |
| **Interoperability** | GeoJSON + REST, standard RTSP in, plugs into existing ICCC/city GIS. |

---

## 19. Risks, mitigations and honest status

| Challenge | Level | Mitigation in TraceNet |
| --- | --- | --- |
| Image quality and OCR errors (night, angle, blur, glare, rain, dirty plates) | High | Best-frame selection (Q-score) → top 2–3 frames → CLAHE → PP-OCR → multi-frame vote → HSRP grammar. >90 % is a target, to be validated. |
| Unreadable, missing or cloned plates | High | OSNet-style Re-ID continues plate-less tracks; impossible travel speed flags a clone candidate. |
| Compute and data scale | High | Edge-first FP16 inference; only JSON metadata leaves the edge. |
| Network drops, camera failures | Medium | RabbitMQ durable queues with local buffering, replayed on reconnect; dead cameras flagged. |
| Clock drift, distance errors | Medium | NTP-synced timestamps; OSRM road distance, not straight-line. |
| Privacy, misuse, DPDP compliance | High | Privacy by design: RBAC, SHA-256 audit of every query, auto-purge. |

### Honest status

- **Prototype on recorded footage.** The city camera network is simulated with recorded clips for six Hyderabad
  locations (Kukatpally, JNTU, Balanagar, Madhapur, Gachibowli, KPHB). Live RTSP is supported and tested through MediaMTX
  but not yet against a real municipal feed.
- **Plate yield depends on resolution.** On low-resolution (768×432) rush-hour clips many vehicles are tracked but few
  plates are clear enough to read; this is why ghost-car re-identification exists. Higher-resolution cameras read far more.
- **Re-ID** defaults to a deterministic colour/gradient descriptor (512-D); the learned OSNet/VeRi-776 model is supported but its weights are not bundled. Auto-purge/retention and mandatory reason codes are planned, not built.
- **Hosted free tier** cannot run YOLO/PaddleOCR (RAM); ANPR evidence endpoints return empty there. Ingestion must run
  locally/edge and post sightings.
- **Demo credentials** are public in this repository by design for judging; rotate them for any real deployment.
- Statistics in demo mode are illustrative. Cross-camera accuracy on real-world city data has **not** yet been validated
  at scale; the synthetic ground-truth benchmark is the reproducible evidence we provide today.

---

## 20. Roadmap

- **Wider coverage** — onboard more cameras and junctions (config-driven).
- **Smarter journeys** — better linking in dense traffic, night and rain; bundled OSNet/VeRi-776 Re-ID.
- **Predictive traffic** — forecast congestion 15–60 minutes ahead.
- **Signal coordination** — feed live insight into signal timing.
- **Incident playbooks** — guided responses when an alert fires.
- **Mobile field app** — the same live picture for officers on the ground.
- **Open, anonymised reporting** — aggregate insight for planners and the public.
- **Stronger edge story** — GPU/Jetson edge workers with TensorRT; object-storage evidence.

---

## 21. Research, references and licences

**Key research**

1. ByteTrack — Zhang et al., 2022, ECCV. DOI 10.1007/978-3-031-20047-2_1
2. OSNet — Zhou et al., 2019, ICCV. DOI 10.1109/ICCV.2019.00380
3. PP-OCR — Du et al., 2020. DOI 10.48550/arXiv.2009.09941
4. CityFlow — Tang et al., 2019, CVPR. DOI 10.48550/arXiv.1903.09254
5. VeRi-776 — Liu et al., 2016, ECCV. DOI 10.1007/978-3-319-46475-6_53
6. VehicleID — Liu et al., 2016, CVPR. DOI 10.1109/CVPR.2016.238
7. UVH-26 — Sharma et al., 2025, AIM@IISc. DOI 10.48550/arXiv.2511.02563 (CC-BY-4.0)

**Models, tools and open data:** Ultralytics YOLO11, PaddleOCR / PP-OCRv5, Awiros ANPR-OCR, IISc UVH-26 models,
Torchreid (OSNet), PostGIS, OpenStreetMap / OSRM (© OpenStreetMap contributors, ODbL).

**Our contribution beyond these works:** plate + appearance + road-physics fusion into one Global Vehicle ID,
physics-based cloned-plate detection, and an operator-ready GIS product (RBAC, audit log, alerts).

**Licence note:** Ultralytics YOLO11 is AGPL-3.0 — review before commercial deployment. Stock video on the marketing
pages is for demonstration only.

---

<div align="center">

**TraceNet** — *Track · Analyze · Connect*

Team **Trace Force_A0** · Team ID 176689 · PS 26127 · **Smart India Hackathon 2026**

</div>
