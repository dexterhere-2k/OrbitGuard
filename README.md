# OrbitGuard — Submission

---

## Title

**OrbitGuard: Decentralized Satellite Trajectory Forecasting & Collision Risk Assessment**

## Tagline

*An incentivized AI subnet that turns orbital safety into a competitive forecasting market.*

---

## Problem Statement

Low Earth Orbit is increasingly congested — over 10,000 active satellites and millions of debris fragments share limited orbital space. Conjunction assessment (predicting whether two objects will collide) currently depends on a small number of centralized operators with limited capacity and no adversarial validation. As launch cadence accelerates, this centralized model does not scale. The consequence of failure is cascading debris generation (Kessler Syndrome), which could permanently degrade access to LEO.

---

## Solution

OrbitGuard structures orbital trajectory prediction as a decentralized forecasting competition on a Bittensor subnet.

- **Validators** publish time-stamped orbital snapshots and prediction horizons.
- **Miners** submit probabilistic trajectory forecasts — state vectors, covariance matrices, and conjunction risk probabilities.
- **Evaluation** occurs after the prediction horizon elapses, scoring miners against archived ground truth.
- **Rewards** are distributed proportionally to objective forecast quality via an emission vector.

Predictions are immutably recorded on IPFS. Every evaluation is deterministic and auditable.

---

## Architecture Summary

```
Validator → Publish Snapshot (IPFS)
         → Define Prediction Task
              ↓
Miners   → Retrieve Snapshot
         → Run Propagation Models
         → Submit Predictions (IPFS)
              ↓
         [Prediction Horizon Elapses]
              ↓
Validator → Retrieve Ground Truth (IPFS)
         → Score Predictions (RMS, Calibration, Pc)
         → Commit-Reveal Weight Setting
         → Distribute Emissions
```

**Key mechanisms:**

- Time-delayed evaluation prevents post-hoc fitting.
- Commit-reveal prevents validator weight copying.
- IPFS provenance ensures full auditability.

---

## Tokenomics Summary

**Emission vector:** `[52428, 13107]` (scaled to 65,535)

| Component | Allocation | Weight |
|-----------|-----------|--------|
| Trajectory Accuracy | 80% | 52,428 |
| Conjunction & Calibration | 20% | 13,107 |

**Incentive alignment:**

- Miners earn TAO proportional to forecast accuracy.
- Trajectory accuracy is weighted 4× higher than conjunction scoring, reflecting the priority hierarchy: accurate orbits are the prerequisite for reliable conjunction assessment.
- A timeliness multiplier provides marginal bonus for early submissions.

---

## Links

| Resource | URL |
|----------|-----|
| Documentation | `./docs/` |
| Architecture | [`architecture.md`](./architecture.md) |
| Scoring Design | [`scoring_and_incentives.md`](./scoring_and_incentives.md) |
| Overview | [`overview.md`](./overview.md) |
