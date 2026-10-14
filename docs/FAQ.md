# Frequently asked questions

### What problem does TraceNet solve?
Cameras work in silos. TraceNet links their sightings into one **Global Vehicle ID** so police can see a full route in seconds
and planners can see how the city moves. (PS 26127.)

### How is this different from normal ANPR?
Normal ANPR reads a plate and forgets it. TraceNet remembers across cameras and uses three signals (plate, appearance, road
physics), so it still works when OCR is wrong or the plate is hidden, and it can flag cloned plates.

### How do you detect a cloned plate?
Same confident plate at two cameras, with a required speed `distance_by_road / time` above 150 km/h. Distances come from OSRM
road routing, so winding roads do not cause false alarms. See [FUSION_ALGORITHM.md](FUSION_ALGORITHM.md).

### What if the plate cannot be read?
The vehicle becomes a "ghost" track: the plate weight goes to zero and appearance + physics decide the match, subject to
strict thresholds (visual ≥ 0.90, physically feasible).

### Is the >90 % OCR accuracy proven?
No. It is a **design target**. What we measured: ~60 ms median CPU latency, 58/60 exact reads on our 60-crop test set, and a
worked 94.2 % consensus read. Validation on clean vs degraded splits is future work, and we say so openly.

### Are the demo numbers real?
The `TS09AB4521` trace (4 cameras, 19.2 km, 37 min, 98.7 %) comes from **seeded** data, and the cameras are recorded clips of
Hyderabad junctions standing in for a city network. The code path (ingest → fusion → PostGIS → API → map) is the real one.

### Does it need new cameras?
No. It uses existing CCTV/ANPR via standard RTSP, and sends small JSON, not video.

### Can it scale to a whole city?
The design is camera-by-camera: stateless fusion workers behind a durable RabbitMQ queue with Redis state; search is bounded by
time window and a 50 km radius. The hosted demo is single-process; broker mode is implemented and tested, and a city-scale
load test has not been run yet.

### Is the Re-ID model learned?
By default a deterministic colour/gradient descriptor is used. A learned OSNet (VeRi-776) extractor is supported when weights
and `torchreid` are present, but weights are not bundled. See [FUSION_ALGORITHM.md](FUSION_ALGORITHM.md#re-id-embeddings).

### What about privacy?
RBAC, JWT, password hashing and a SHA-256 hash-chained audit log are implemented. No faces or biometrics. Auto-purge and
mandatory reason codes are planned, not built. Details in [PRIVACY_AND_COMPLIANCE.md](PRIVACY_AND_COMPLIANCE.md).

### Why can't the free hosted version run the AI?
YOLO and PaddleOCR need several GB of RAM. Detection and OCR run locally or on an edge/GPU machine and POST sightings to the
deployed API. The hosted API itself uses ~110 MB.

### What happens when the backend is down?
The frontend falls back to bundled demo data after 3 seconds and switches to live data when the API responds.

### Which Indian plate formats are supported?
Standard state plates (e.g. `TS09AB4521`, `DL8CAK0211`) and BH series (`22BH1234AA`), validated against real state/UT codes with
position-aware OCR correction.

### How do I run it?
`npm install && npm run dev` for the demo UI in 30 seconds. Full instructions in the root README and
[JUDGES_GUIDE.md](JUDGES_GUIDE.md).

### How do I add a camera?
Add an entry to `backend/config/cameras.json` (id, coordinates, source), add its distances to
`backend/config/road_network.json` (or let the haversine fallback apply), and start ingestion.

### What licences are involved?
Mostly Apache/MIT/BSD. Ultralytics YOLO11 is AGPL-3.0 — review it before a commercial deployment. UVH-26 is CC-BY-4.0.
OpenStreetMap data is ODbL.
