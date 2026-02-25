# OrbitGuard — Scoring & Incentives

This document defines the objective metrics, emission vector, and economic design of the OrbitGuard subnet.

---

## Scoring Dimensions

Miners are scored on two weighted categories: **Trajectory Accuracy** and **Conjunction & Calibration**.

### Trajectory Accuracy (80% of emissions)

| Metric | Definition | Computation |
|--------|-----------|-------------|
| **RMS Position Error** | Root-mean-square of Euclidean distance between predicted and true position vectors | `√(Σ ‖r_pred − r_true‖² / N)` across all objects and epochs |
| **RMS Velocity Error** | Root-mean-square of velocity vector difference | `√(Σ ‖v_pred − v_true‖² / N)` |
| **Along-Track / Cross-Track / Radial Decomposition** | Error decomposed into orbital reference frame components | Enables diagnosis of systematic propagation biases |

**Score mapping:** Raw RMS errors are mapped to a `[0, 1]` score via a monotonically decreasing function. Lower error → higher score. The mapping function is calibrated against baseline propagator performance (e.g., SGP4) so that miners must exceed naive propagation to earn meaningful rewards.

### Conjunction & Calibration (20% of emissions)

| Metric | Definition | Computation |
|--------|-----------|-------------|
| **Miss Distance Error** | Absolute error in predicted closest-approach distance for flagged conjunctions | `‖d_pred − d_true‖` |
| **Collision Probability Calibration** | How well predicted probabilities match observed outcome frequencies | Reliability diagram analysis; Brier score |
| **Classification Precision** | Fraction of flagged conjunctions that were actual close approaches | `TP / (TP + FP)` |
| **Classification Recall** | Fraction of actual close approaches that were correctly flagged | `TP / (TP + FN)` |

**Calibration matters:** A miner who predicts `Pc = 0.01` should be correct ~1% of the time. Systematically overconfident or underconfident probability estimates are penalized.

---

## Emission Vector

Emissions are distributed using a two-component weight vector scaled to the Bittensor standard of 65,535 total:

| Component | Weight | Fraction | Raw Value |
|-----------|--------|----------|-----------|
| Trajectory Accuracy | 80% | 0.80 | **52,428** |
| Conjunction & Calibration | 20% | 0.20 | **13,107** |
| **Total** | **100%** | **1.00** | **65,535** |

**Emission vector:** `[52428, 13107]`

The 80/20 split reflects the priority hierarchy: accurate trajectory propagation is the foundational capability, while conjunction assessment is a higher-order derivative that depends on accurate trajectories.

---

## Composite Score

A miner's total score for a prediction cycle:

```
S_total = 0.80 × S_trajectory + 0.20 × S_conjunction

Where:
  S_trajectory ∈ [0, 1]  — normalized trajectory accuracy score
  S_conjunction ∈ [0, 1] — normalized conjunction & calibration score
```

Weights are then set proportional to `S_total` across all miners:

```
w_i = S_total_i / Σ S_total_j    (for all miners j)
```

---

## Timeliness Multiplier

Miners who submit predictions earlier within the submission window receive a timeliness bonus. This incentivizes prompt forecasting, which has operational value for real-world conjunction assessment.

```
T = 1.0 + α × (deadline − submission_time) / window_duration

Where:
  α = timeliness bonus coefficient (e.g., 0.1 for a 10% max bonus)
  T ∈ [1.0, 1.0 + α]
```

**Adjusted score:** `S_adjusted = S_total × T`

The timeliness multiplier is small relative to accuracy scoring — it breaks ties between miners of similar quality, not the overall ranking order.

---

## Anti-Gaming Economic Design

| Mechanism | Purpose |
|-----------|---------|
| **Time-delayed evaluation** | Predictions scored against future ground truth; no post-hoc fitting |
| **IPFS-pinned submissions** | Immutable record; predictions cannot be retroactively modified |
| **Commit-reveal for validators** | Prevents weight copying between validators |
| **Baseline normalization** | Scores calibrated against naive propagator; random guessing yields ≈ 0 |
| **Covariance scoring** | Miners cannot inflate scores by submitting artificially tight uncertainties |
| **Precision/recall balance** | Miners cannot game conjunction scoring by always predicting "collision" or "no collision" |
| **Stake requirements** | Economic cost to register as miner or validator; Sybil resistance |

The incentive structure rewards miners who invest in better models, more compute, and domain expertise — not miners who exploit scoring loopholes.
