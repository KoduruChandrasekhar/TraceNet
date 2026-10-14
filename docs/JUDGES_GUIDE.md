# Judges' guide — evaluate TraceNet in 10 minutes

## 0. What you are looking at

TraceNet turns isolated CCTV/ANPR cameras into one city-wide system. A vehicle seen by several cameras is fused into a
single **Global Vehicle ID (GVID)** using **plate text + visual appearance + road-network physics**, then used for
journey tracking, traffic analytics and live alerts.

## 1. Zero-setup path (2 minutes)

Open the live deployment: <https://tracenet-ten.vercel.app/>

The site auto-signs-in as the demo police officer. If the backend is asleep (free tier) pages show built-in demo data
and switch to live data when the API wakes (up to ~1 minute).

| Page | Try this | What it proves |
| --- | --- | --- |
| **Home** | Scroll the guided journey story; switch map layers | The "network, not cameras" idea |
| **Tracking** | Search `TS09AB4521` (or a partial plate) | Route, timestamps, direction, detour on the GIS map; camera hand-off and Re-ID evidence |
| **Traffic** | Open corridors, heat layer, O-D flows | Module 3 — city-wide analytics from trajectories |
| **Alerts** | Watch the feed | Module 4 — live WebSocket alerts, each placed on the map |
| **Cameras** | Hover tiles | Camera wall with health status; recorded Hyderabad clips |
| **How it works** | Open *OCR lab* and *Field tests* | Real evidence images, real latencies, honest funnels |
| **Admin** (log in as `admin`) | Users, roles, cameras | RBAC and administration |

Demo accounts (public by design): `admin` / `admin123` (camera_admin), `officer` / `police123` (law_enforcement).

## 2. Local end-to-end demo (≈5 minutes)

Requires Docker, Python 3.12, Node 20+. See the README for installation.

```bash
docker compose up -d postgis
python -m backend.db.migrate
python backend/scripts/seed_db.py --migrate
uvicorn backend.main:app --port 8000          # terminal 1
VITE_API_BASE_URL=http://localhost:8000 npm run dev   # terminal 2
python backend/scripts/run_demo.py --fast     # terminal 3 — scripted scenario
```

`run_demo.py` posts scripted sightings over HTTP exactly as cameras would, while an officer session listens on the
WebSocket. Expected results:

| Scenario | Plate | Expected outcome |
| --- | --- | --- |
| **Hero trace** | `TS09EA1234` | One GVID across CAM-410 → 406 → 411 → 401, even with a 1-character OCR error at CAM-411 |
| **Ghost car** | `TS08UB5678` | Plate unread at CAM-406 (`NO_PLATE_DETECTED`) yet re-linked by appearance + physics |
| **Cloned plate** | `MH12AB9999` | Same plate at two far-apart cameras 30 s apart → `CLONED_PLATE` (critical) |
| **Blacklist** | `DL01XY0001` | On the seeded watchlist → `BLACKLIST_HIT` |
| **Tampered plate** | `TS0XX12` | Fails format validation → `INVALID_FORMAT` |

The script then verifies trajectories, received alerts, the audit log and refreshes the Polars analytics.

## 3. Things worth verifying yourself

1. **Audit chain** — as `officer`, look up a plate, then call `GET /api/v1/audit/verify`. Editing any row in
   `query_audit_log` makes verification fail from that row onward.
2. **RBAC** — an `officer` token calling `POST /api/ingestion/start` gets **403**; no token gets **401**.
3. **Fusion tuning** — all thresholds are in `backend/config/fusion.json`; tests in `backend/tests/test_phase3_fusion.py`.
4. **Reproducible benchmark** — `python backend/scripts/generate_dataset.py && python backend/scripts/evaluate_dataset.py`
   scores fusion and alerts against exact ground truth.
5. **Tests** — `pytest backend/tests`.

## 4. Mapping to our submission

| Submission claim | Where to see it |
| --- | --- |
| Module 1 — high-precision ANPR | [ANPR_PIPELINE.md](ANPR_PIPELINE.md), *How it works → OCR lab* |
| Module 2 — trajectory reconstruction | *Tracking* page, `backend/api/spatial.py` |
| Module 3 — traffic analytics | *Traffic* page, `backend/analytics/polars_jobs.py` |
| Module 4 — real-time alerts | *Alerts* page, `backend/alerts/`, `WS /ws/alerts` |
| Fusion innovation | [FUSION_ALGORITHM.md](FUSION_ALGORITHM.md) |
| Privacy by design | [PRIVACY_AND_COMPLIANCE.md](PRIVACY_AND_COMPLIANCE.md) |

## 5. Please note (honest status)

- Design targets (>90 % OCR accuracy, <5 s lookup, <100 ms alert push, 60 s heatmap refresh) are **goals, not yet measured**.
- The demo trace (`TS09AB4521`, 19.2 km, 37 min, 98.7 %) comes from **seeded data**, not a field deployment.
- By default Re-ID uses a deterministic colour/gradient descriptor; the learned OSNet/VeRi-776 model is supported but
  its weights are not bundled. See [FUSION_ALGORITHM.md](FUSION_ALGORITHM.md#re-id-embeddings).
