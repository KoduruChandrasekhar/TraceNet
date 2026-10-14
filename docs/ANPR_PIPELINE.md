# ANPR pipeline for Indian plates

Code: `backend/anpr/`, `backend/ai/` · tuning: `backend/config/anpr.json` · tests: `test_phase2_anpr.py`, `test_anpr_fixes.py`

## Flow

```text
vehicle track (YOLO11 + ByteTrack)
  → plate candidate search
  → Q-score (no OCR needed)         ← discard unreadable crops early
  → top-K crops per track (K = 3)
  → CLAHE contrast enhancement (bilateral fallback)
  → OCR (hybrid fast → accurate)
  → normalise + positional correction + format/state validation
  → multi-frame consensus vote
  → one plate + confidence per vehicle
```

## 1. Vehicle detection and tracking

YOLO11 detects vehicles; ByteTrack keeps stable per-camera IDs. Tracks shorter than 3 hits, or vehicles smaller than
60 × 40 px, are ignored. For Indian traffic, `backend/training/yolo/` can fine-tune the detector on IISc **UVH-26** with
sampling-based class weighting for two-wheelers and auto-rickshaws, an optional P2 (stride-4) head for small objects, and
NMS tuning.

## 2. Plate candidate search

Searches the lower part of each vehicle (40 %), filtering by aspect ratio (2.5–5.5; relaxed for motorcycles), rectangularity,
skew (≤ 30°) and character-transition count, then removes duplicates and plates not owned by the vehicle (≥ 80 % overlap).

## 3. Quality score (Q-score)

`Q` combines plate **area**, **sharpness**, **exposure** and **skew** (weights 0.35 / 0.45 / 0.20 / 0.30, configurable).
Crops below minimum size (40 × 12 px), sharpness (25), skew (20°) or `Q < 0.20` are dropped **before** OCR, which saves
compute and avoids polluting the vote with garbage. Only the best 3 crops are read; reading stops early if `Q ≥ 0.70`.

## 4. Enhancement

CLAHE (Contrast-Limited Adaptive Histogram Equalisation) improves dusk, glare and shadow conditions; a bilateral-filter
variant is the fallback if the first read is weak.

## 5. Hybrid OCR

| Stage | Model | Role |
| --- | --- | --- |
| Fast | PP-OCRv5 **mobile** recognition (plate crop is a single line, detection skipped) | Median ≈ 60 ms on CPU |
| Accurate | PP-OCRv6 **medium** det + rec | Used only when confidence < 0.80 or fewer than 4 characters |

Device plan, tried in order at startup (each rung must pass a real inference): GPU + TensorRT FP16 → GPU (CUDA) → CPU oneDNN.

| Measured on CPU (60-crop set) | Median | Exact reads |
| --- | --- | --- |
| Baseline full detection + recognition | 1,850 ms | 59 / 60 |
| Medium-rec hybrid | 295 ms | 59 / 60 |
| **Default mobile-rec + oneDNN hybrid** | **≈ 60 ms** | **58 / 60** |

## 6. Validation (`validation.py`)

| Format | Regex | Example |
| --- | --- | --- |
| Standard | `^[A-Z]{2}[0-9]{1,2}[A-Z]{0,3}[0-9]{4}$` | `TS09AB4521`, `DL8CAK0211` |
| BH series | `^[0-9]{2}BH[0-9]{4}[A-Z]{1,2}$` | `22BH1234AA` |

- The first two letters must be a real state/UT code (legacy codes included; `TG` for Telangana).
- **Positional correction**: first the format is fixed, then a character is swapped only where that position expects the
  other class (digit in the state code → letter; letter in the serial number → digit; e.g. `0→O`, `8→B`, `1→I` in letter positions; `O→0`, `B→8`, `Z→2` in digit positions). Characters
  are never substituted blindly.
- Stock-footage watermark text (Getty, iStock…) is rejected.
- Failing plates are flagged `INVALID_FORMAT` and can raise an alert (possible tampering).

## 7. Multi-frame consensus (`consensus.py`)

Reads of one transit are grouped when within edit distance 1 of each other (so one misread character never splits a vehicle into two plates). The largest group wins by majority, and the most frequent string in it is the plate. If no string is more frequent, same-length reads get a confidence-weighted per-character vote.

Real example (CAM-401, track 4): reads `AP09AZ6596` (95.5 %), `AP.09AZ6596` (92.6 %), `AP.09AZ6596` (94.6 %) → **AP09AZ6596 at
94.2 %**, parsed as State `AP` · District `09` · Series `AZ` · Number `6596`. The *How it works → OCR lab* page shows the
actual frames, crops and enhanced crops.

## 8. Known limits

- Low-resolution footage (e.g. 768 × 432) loses plate yield: many vehicles are tracked but few crops pass the quality gate.
  The *Field tests* page shows each funnel honestly. Fusion's ghost-car path exists to cover that gap.
- The >90 % accuracy goal is a **target** pending evaluation on clean vs degraded splits.
- Night and heavy rain need regional training data; the fine-tuning scripts are provided but results are not claimed.
