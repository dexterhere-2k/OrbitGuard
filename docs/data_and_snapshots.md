# OrbitGuard — Data & Snapshots

This document defines the data structures, ground truth handling, and reproducibility guarantees of the OrbitGuard subnet.

---

## Snapshot Structure

A **snapshot** is a time-stamped collection of orbital elements for a set of tracked objects. Validators construct snapshots from publicly available sources (e.g., GP catalogs, supplemental ephemerides).

```json
{
  "snapshot_id": "orbitguard-snap-20260225-1200Z",
  "epoch": "2026-02-25T12:00:00Z",
  "source": "GP catalog",
  "ipfs_cid": "bafybeig...",
  "objects": [
    {
      "norad_id": 25544,
      "name": "ISS (ZARYA)",
      "tle_line1": "1 25544U 98067A   26056.50000000 ...",
      "tle_line2": "2 25544  51.6416 ..."
    }
  ],
  "prediction_horizon_hours": 72,
  "evaluation_epochs": [
    "2026-02-26T12:00:00Z",
    "2026-02-27T12:00:00Z",
    "2026-02-28T12:00:00Z"
  ],
  "submission_deadline": "2026-02-25T14:00:00Z"
}
```

**Key properties:**

- All miners receive the identical snapshot via its IPFS CID.
- The prediction horizon and evaluation epochs are explicit — no ambiguity.
- The submission deadline enforces that predictions are submitted *before* any ground truth is available.

---

## Prediction Structure

Miners submit a **probabilistic forecast** for each requested object at each evaluation epoch.

```json
{
  "miner_hotkey": "5FHne...",
  "snapshot_id": "orbitguard-snap-20260225-1200Z",
  "submitted_at": "2026-02-25T13:45:00Z",
  "ipfs_cid": "bafybeid...",
  "predictions": [
    {
      "norad_id": 25544,
      "epoch": "2026-02-26T12:00:00Z",
      "state_vector": {
        "x_km": 6700.12, "y_km": 1234.56, "z_km": -789.01,
        "vx_km_s": 1.234, "vy_km_s": 7.654, "vz_km_s": -0.321
      },
      "covariance_6x6": [
        [0.01, 0.0, 0.0, 0.0, 0.0, 0.0],
        "... (symmetric 6×6 matrix)"
      ],
      "conjunction_flags": [
        {
          "secondary_norad_id": 40000,
          "miss_distance_km": 0.85,
          "probability_of_collision": 0.00012
        }
      ]
    }
  ]
}
```

**Key properties:**

- Predictions include both point estimates (state vectors) and uncertainty quantification (covariance).
- Conjunction flags are optional but scored when present — miners are rewarded for correctly identifying close approaches and penalized for false alarms.
- The entire prediction payload is pinned to IPFS at submission time.

---

## Ground Truth Handling

After the prediction horizon elapses, validators retrieve **ground truth** orbital data for the evaluation epochs.

| Ground Truth Source | Resolution | Latency |
|---------------------|-----------|---------|
| Updated GP catalog TLEs | ~1 km accuracy | Hours to days |
| SP3 precision ephemerides | ~cm to m accuracy | Days |
| Conjunction Data Messages (CDMs) | Event-specific | Hours |

**Process:**

1. Validator retrieves ground truth after the evaluation epoch has passed.
2. Ground truth is pinned to IPFS — its CID is included in the evaluation receipt.
3. Scoring is computed by comparing miner predictions against this ground truth.
4. Multiple ground truth sources can be used; the scoring function documents which source is canonical for each metric.

**Important:** Ground truth is never available to miners before submission. The temporal gap between the submission deadline and the ground truth retrieval window is the fundamental anti-gaming mechanism.

---

## Reproducibility Design

OrbitGuard is designed so that any third party can independently verify every evaluation.

**Reproducibility chain:**

```
Snapshot CID  →  Miner Prediction CID  →  Ground Truth CID  →  Evaluation Receipt CID
     │                    │                       │                        │
     ▼                    ▼                       ▼                        ▼
  Identical            Immutable              Canonical               Deterministic
  inputs for           record of              reference               scoring output
  all miners           forecast               data                    with full audit
```

**Requirements for full reproducibility:**

1. All input data is content-addressed and retrievable.
2. The scoring function is deterministic — same inputs always produce same scores.
3. Timestamps are validated against blockchain block heights.
4. Evaluation receipts include all intermediate values (per-object errors, per-metric scores).

---

## Why Time-Delayed Scoring Prevents Gaming

The core insight: **if you cannot see the answer before submitting your prediction, you must actually forecast.**

| Gaming Strategy | Why It Fails |
|-----------------|--------------|
| **Post-hoc fitting** | Submission deadline precedes ground truth availability by hours/days |
| **Copying other miners** | Predictions are submitted (and pinned) independently; no broadcast before deadline |
| **Validator collusion** | Validators do not have ground truth at task time — they retrieve it later |
| **Random guessing** | RMS error scoring penalizes inaccurate predictions; covariance calibration penalizes overconfident guesses |
| **Always predicting "no conjunction"** | Classification precision/recall scoring penalizes both false negatives and false positives |

The time-delayed evaluation model is borrowed from meteorological forecast verification — a well-established methodology for objectively scoring probabilistic predictions.
