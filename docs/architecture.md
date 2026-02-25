# OrbitGuard — Architecture

This document describes the system components, data flows, and trust mechanisms that comprise the OrbitGuard subnet.

---

## System Components

| Component | Role | Trust Model |
|-----------|------|-------------|
| **Validator** | Publishes orbital snapshots, defines prediction tasks, evaluates miner submissions, sets weights | Staked; must reach consensus with other validators |
| **Miner** | Receives prediction tasks, runs trajectory propagation / ML models, submits probabilistic forecasts | Permissionless; scored purely on output quality |
| **IPFS Layer** | Stores snapshots, submissions, and evaluation receipts with content-addressed immutability | Trustless; content hash guarantees integrity |
| **Bittensor Metagraph** | Maintains validator/miner registration, emission distribution, and weight consensus | On-chain; deterministic |
| **Ground Truth Archive** | Canonical post-hoc orbital data used for scoring (e.g., SP3 ephemerides, updated TLEs) | External; pinned to IPFS after retrieval |

---

## Data Flow

The system operates in discrete **prediction cycles**. Each cycle follows this sequence:

```
Step 1 — SNAPSHOT PUBLICATION
    Validator retrieves current orbital data (TLEs or state vectors)
    Validator pins snapshot to IPFS → receives CID
    Validator broadcasts {CID, object_ids, prediction_horizon, deadline}

Step 2 — MINER FORECASTING
    Miners retrieve snapshot from IPFS using CID
    Miners run propagation models over the prediction horizon
    Miners produce:
      • Predicted state vectors at evaluation epoch(s)
      • 6×6 covariance matrices (position + velocity uncertainty)
      • Conjunction risk probabilities for flagged object pairs
    Miners submit predictions before the deadline

Step 3 — PREDICTION WINDOW ELAPSES
    No scoring occurs during this period
    The system waits for the real-world prediction horizon to pass
    This enforces genuine forecasting — no post-hoc fitting

Step 4 — GROUND TRUTH RETRIEVAL
    Validator retrieves ground truth for the evaluation epoch
    Ground truth is pinned to IPFS → receives CID
    Both snapshot and ground truth are now immutably recorded

Step 5 — EVALUATION & SCORING
    Validator compares miner predictions against ground truth
    Computes objective metrics (RMS error, miss distance, calibration)
    Produces evaluation receipt → pinned to IPFS

Step 6 — WEIGHT SETTING & EMISSION
    Validator aggregates scores into a weight vector
    Weights are committed to the Bittensor metagraph
    TAO emissions flow to miners proportional to their weights
```

---

## Miner–Validator Interaction Model

```
  VALIDATOR                              MINER
     │                                     │
     │──── Publish snapshot (CID) ────────▶│
     │──── Prediction task params ────────▶│
     │                                     │
     │                              ┌──────┴──────┐
     │                              │ Run models   │
     │                              │ Propagate    │
     │                              │ Compute Pc   │
     │                              └──────┬──────┘
     │                                     │
     │◀──── Submit predictions ────────────│
     │                                     │
     │  ┌─────────────────────┐            │
     │  │ Wait for horizon    │            │
     │  │ to elapse           │            │
     │  └─────────────────────┘            │
     │                                     │
     │  ┌─────────────────────┐            │
     │  │ Retrieve ground     │            │
     │  │ truth, evaluate,    │            │
     │  │ set weights         │            │
     │  └─────────────────────┘            │
     │                                     │
     │──── Emission distributed ──────────▶│
```

Miners never see the ground truth before submitting. Validators never see miner predictions before the deadline. This temporal isolation is the core anti-gaming property.

---

## Commit–Reveal Mechanism

Validators use a commit–reveal scheme to prevent **weight copying** — where a lazy validator simply copies another validator's weight assignments instead of independently evaluating miners.

| Phase | Action | Data |
|-------|--------|------|
| **Commit** | Validator publishes `hash(weights ∥ nonce)` on-chain | Only the hash is visible |
| **Reveal** | After the commit window closes, validator reveals `weights` and `nonce` | Hash is verified against the commitment |

**Properties:**

- A validator cannot change weights after committing.
- Other validators cannot derive weights from the hash alone.
- Validators who fail to reveal within the window are penalized.
- This ensures each validator independently evaluates miner performance.

---

## Provenance & IPFS

Every data artifact in OrbitGuard is pinned to IPFS before it is referenced on-chain or used in evaluation.

| Artifact | Pinned When | Purpose |
|----------|-------------|---------|
| Orbital snapshot | At task publication | Ensures all miners work from identical input |
| Miner predictions | At submission | Immutable record of forecast |
| Ground truth | At evaluation time | Canonical reference for scoring |
| Evaluation receipt | After scoring | Audit trail linking inputs → outputs → scores |

**Why IPFS:**

- **Content-addressed:** The CID is a cryptographic hash of the content. Tampering changes the hash.
- **Decentralized storage:** No single entity controls data availability.
- **Deterministic retrieval:** Any party with the CID can independently verify the data.
- **Audit trail:** The chain of CIDs (snapshot → predictions → truth → receipt) forms a complete, verifiable provenance graph.

This provenance chain ensures that any third party can independently reproduce and verify every evaluation decision.
