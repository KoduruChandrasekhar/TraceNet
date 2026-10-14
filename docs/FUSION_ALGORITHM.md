# Cross-camera fusion: plate + appearance + road physics

Code: `backend/fusion/fusion_engine.py` · tuning: `backend/config/fusion.json` · tests: `backend/tests/test_phase3_fusion.py`

## The idea

Plate-only matching breaks when OCR makes one wrong character, when the plate is hidden, or when two cars share a plate.
TraceNet scores every candidate match on three independent signals and requires them to agree.

```text
S = w_text · text_sim + w_visual · visual_sim + w_kinematic · physics_ok        (w sum to 1)
    0.35               0.40                      0.25                           associate when S ≥ 0.70
```

| Signal | Definition |
| --- | --- |
| `text_sim` | `1 − Levenshtein(a, b) / max(len(a), len(b))` |
| `visual_sim` | cosine similarity of the two 512-D appearance embeddings |
| `physics_ok` | from implied speed `v = d_road / Δt`: 1.0 up to 120 km/h, falling linearly to 0.0 at 150 km/h, 0 beyond |

`d_road` is the **directed OSRM driving distance** between the two cameras (`backend/config/road_network.json`;
fallback `haversine × 1.35` for unknown pairs).

## Candidate selection

A sighting is compared only with vehicles in the **active window** (30 minutes by event time) whose last camera is
within **50 km**. Older vehicles are expired and their trajectories completed.

## Hard guards (applied on top of the score)

1. A kinematically impossible link (`physics_ok == 0`) never associates two sightings.
2. The same camera cannot see one vehicle as two concurrent transits.
3. Two confident plates must agree (`text_sim ≥ 0.80`).
4. Plate-less links need `visual_sim ≥ 0.90`.
5. A plate-less re-link at the **same** camera must happen within 15 s (a tracker fragment). Later, a similar car is more
   likely a different one.

## Ghost cars (unreadable plate)

If the plate is missing, invalid or below 0.70 OCR confidence, the text weight is redistributed onto appearance:
`w_text = 0`, `w_visual = 0.75`, `w_kinematic = 0.25`. The vehicle keeps its journey and receives a ghost identity.

## Cloned-plate detection

If the **same confident plate** (`text_sim ≥ 0.90`) appears at **different cameras** and the implied speed exceeds
**150 km/h**, the second sighting is **not merged**; a `CLONED_PLATE` alert (CRITICAL) is raised. Because the check uses road
distance and event time, a legitimate vehicle driving normally never triggers it, and the distance error from winding
roads does not cause false positives the way straight-line distance would.

## Identity

`Global Vehicle ID = uuid5(NAMESPACE_DNS, plate)`, or `uuid5(NAMESPACE_DNS, "ghost:…")` for plate-less vehicles.
Deterministic IDs stay stable across restarts and across workers.

Decision order: exact active plate → same vehicle; otherwise best candidate with `S ≥ 0.70` that passes all guards;
otherwise a new vehicle.

## Re-ID embeddings

`backend/fusion/reid_matcher.py` provides two extractors behind one interface:

| Extractor | When used | Nature |
| --- | --- | --- |
| **OSNet** (`osnet_x1_0`, e.g. VeRi-776 weights) | `torchreid` installed **and** weights found | Learned vehicle Re-ID embedding |
| **Handcrafted fallback** | default when weights are absent | Deterministic HSV-histogram + gradient-orientation descriptor (256 + 256 dims) |

The fallback separates vehicles of different colour/shape but cannot tell two similar white hatchbacks apart. That is why
plate-less links require a high visual threshold (0.90) **and** physical feasibility. The OSNet weights are not bundled
in this repository; installing them is the supported upgrade path.

## Worked example

Two sightings of a hatchback: CAM-410 at 18:45:00 reads `TS09EA1234`; CAM-411 at 19:08:00 reads `TS09EA1Z34` (one OCR error).

- `text_sim = 1 − 1/10 = 0.90`
- `visual_sim ≈ 0.95` (same vehicle crop family)
- road distance 10.19 km (OSRM table) over 23 min → 26.6 km/h → `physics_ok = 1.0`
- `S = 0.35·0.90 + 0.40·0.95 + 0.25·1.0 = 0.945 ≥ 0.70` → same GVID, appended to the route.

Same plate at CAM-410 and CAM-401 (11.7 km apart) 30 s later → 1,409 km/h → `physics_ok = 0`, `text_sim = 1.0` → not merged,
`CLONED_PLATE` raised.

(The numbers are illustrative of the formula, using real inter-camera distances from `road_network.json`.)

## Tuning

Every parameter above is a key in `backend/config/fusion.json` (`w_text`, `w_visual`, `w_kinematic`, `match_threshold`,
`speed_ok_kmh`, `speed_max_kmh`, `clone_text_similarity`, `ghost_*`, `active_window_seconds`, `candidate_radius_km`).
The synthetic ground-truth benchmark (`generate_dataset.py` + `evaluate_dataset.py`) can be re-run after any change.
