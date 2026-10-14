# Architecture

## 1. Principles

1. **Edge computes, core correlates.** Heavy vision runs near the cameras; only small JSON *sightings* travel to the core.
2. **Event-driven.** Sightings are messages; fusion workers are stateless consumers backed by Redis, so adding capacity means
   adding consumers.
3. **Degrade, don't fail.** Every infrastructure dependency (Redis, RabbitMQ, database, backend) has a fallback.
4. **Config over code.** Cameras, thresholds and road distances live in JSON.
5. **Auditable by default.** Every journey lookup is hash-chained.

## 2. Component view

| # | Component | Code | Responsibility |
| --- | --- | --- | --- |
| 1 | Ingestion | `backend/ingestion/` | Per-camera workers reading RTSP or recorded MP4; deterministic timestamps; start/stop/health |
| 2 | Detection + tracking | `backend/ai/vehicle_tracker.py` | YOLO11 + ByteTrack → per-camera track IDs |
| 3 | ANPR | `backend/anpr/`, `backend/ai/` | Plate localisation, Q-score gating, CLAHE, PaddleOCR, validation, consensus |
| 4 | Fusion | `backend/fusion/` | Cross-camera association, GVID, clone detection |
| 5 | Persistence | `backend/fusion/postgres_writer.py`, `db/schema.sql` | PostGIS observations and trajectories |
| 6 | Alerts | `backend/alerts/` | Watchlist, payload shaping, in-process bus, WebSocket fan-out |
| 7 | Analytics | `backend/analytics/` | Polars jobs: leg speeds, corridors, O-D, scheduler |
| 8 | API | `backend/main.py`, `backend/api/` | FastAPI: auth/RBAC, spatial, analytics, streams, admin, audit |
| 9 | Web app | `src/` | React 19 + Leaflet command centre |

## 3. Data flow

```text
camera frame
  → YOLO11 detect → ByteTrack local track id
  → plate candidate → Q-score → top-K crops → CLAHE → PaddleOCR → validate → multi-frame consensus
  → Sighting { camera, time, plate, ocr_confidence, 512-D embedding | vehicle crop, class }
  → POST /api/v1/ingest/sightings
      broker mode : RabbitMQ q.fusion → fusion worker → Redis (active window) → result / alerts on Pub/Sub
      direct mode : in-process FusionEngine
  → trajectory (PostGIS) + alerts (WebSocket) → analytics (Polars) → dashboard
```

The sighting contract lets any device that can POST JSON act as an edge node. The embedding may be sent as base64
float32, or the server can compute it from a base64 PNG/JPEG vehicle crop.

## 4. Deployment topologies

| Topology | Components | Use |
| --- | --- | --- |
| **Demo (static)** | Vercel only, bundled data | Judging without any backend |
| **Hosted** | Vercel + Render (direct mode, 1 worker) + Supabase PostGIS | Public demo, free tier |
| **Local full** | Docker: PostGIS, Redis, RabbitMQ, MediaMTX + API + optional separate fusion worker | Development, live-stream simulation |
| **City scale (design)** | Many edge nodes → durable RabbitMQ → N fusion workers → Redis GEO → PostGIS cluster | Target architecture |

### Why single worker on the hosted tier
The fusion engine, alert bus and watchlist cache live in process memory in direct mode, so the hosted API runs one
instance. Broker mode externalises that state to Redis/RabbitMQ and removes the restriction.

## 5. Failure behaviour

| Failure | Behaviour |
| --- | --- |
| RabbitMQ/Redis down at start | `auto` transport falls back to direct mode |
| Database unreachable | Built-in demo accounts still authenticate; API routes report `db/health` status |
| Backend asleep/unreachable (frontend) | 3 s timeout, pages render bundled demo data, retry on next load |
| MediaMTX not running | Cameras page plays recorded MP4 instead of HLS |
| Camera read errors | Worker tolerates up to `max_consecutive_read_errors` (15) and reports status |

## 6. Key design decisions

| Decision | Why |
| --- | --- |
| Fuse three signals, not plates alone | OCR fails on blur, glare and night; appearance and physics keep the track alive |
| OSRM road distance, not straight line | One-way roads and detours change feasible speed; asymmetric directed distances |
| Deterministic GVID (`uuid5` of plate) | Same vehicle gets the same ID across restarts and workers |
| Polars analytics off the request path | Aggregation never blocks API latency; Arrow-native reads |
| PostGIS | Spatial indexing and GIS interoperability (GeoJSON, WMS/WFS later) |
| Hash-chained audit with advisory lock | Tamper evidence without an external ledger; concurrency-safe |
| Metadata-only upstream | Bandwidth, cost and privacy |
