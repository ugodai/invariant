# Invariant Verification

**Proof of Invariant (PoI)** is a protocol architecture for verifying structural invariants in distributed ledgers based on directed acyclic graphs (causal DAGs), detecting double-signing (equivocation), and ensuring deterministic ordering and conditional finality of the distributed ledger.

## Table of Contents
- [Core Architecture](#core-architecture)
- [Protocol Mechanics (How PoI Works)](#protocol-mechanics-how-poi-works)
- [Key Properties & Threat Model]
- [Repository Structure]
- [Integration & SDK]
- [Citation](#citation)
- [License](#license)

## Core Architecture

**Proof of Invariant (PoI)** decouples topological graph admissibility and causal validation from state execution. Rather than relying on heavy global consensus rounds, PoI enforces structural rules at the graph level, enabling concurrent transaction submission with deterministic state convergence.

```
┌─────────────────────────────┐
│      Validator Proposals    │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│     Inductive DAG Rules     │
│ Parent Closure · Sequences  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│     Equivocation Tracker    │
│     O(1) Sequence Guard     │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│      Anchor Commitment      │
│     Future Beacon Salt      │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│     Deterministic State     │
│       S = Execute(D, S₀)    │
└─────────────────────────────┘
```

### Key Architectural Pillars

* **Causal Delta Graph (Causal DAG):** Transactions and state updates are submitted as content-addressed *deltas*     
  `d = \langle \text{ChainID}, r, h, P(d), \text{seq}, \text{payload}, \sigma \rangle`      
  referencing prior parents, forming a directed acyclic graph.
* **Inductive Topological Safety:** Graph acyclicity is guaranteed through content-addressed parent references `P(d)`, explicit self-parent chain rules, and strict round-height progression.
* **Equivocation Guard:** An `O(1)` sequence-indexed guard tracks validator activity and detects multi-branch double-signing attempts without graph traversal overhead.
* **Anchor-Based Segment Ordering:** A deterministic round-and-anchor ordering scheme (extending DAG-based consensus models) establishes global linear sequence across asynchronous views.
* **Future-Beacon Salting:** Per-round future beacons salt transaction identifiers during canonical sorting to neutralize proposer grinding and ordering manipulation.


## Protocol Mechanics (How PoI Works)

PoI processes, orders, and finalizes state transitions through five sequential phases:

### 1. Invariant & Delta Verification
When a node receives a delta `d` from validator `v_i`, it recomputes all structural fields before insertion:
* **Self-Chain Rule:** `P(d)` must contain exactly one self-parent `s(d)` signed by `v_i`, where `\text{seq}(d) = \text{seq}(s(d)) + 1` and `r(d) > r(s(d))`.
* **Round & Quorum Rule:** `r(d) = 1 + \max_{p \in P(d)} r(p)`, requiring `P(d)` to include parent deltas from at least `Q` distinct signers at round `r(d) - 1`.
* **Height Rule:** `h(d) = 1 + \max_{p \in P(d)} h(p)`.

### 2. `O(1)` Equivocation Tracking & Ejection
To detect validators issuing conflicting deltas (double-signing):
* Nodes maintain an in-memory index `I: (\text{ValidatorID}, \text{seq}) \to \text{DeltaID}`.
* If `I(v_i, \text{seq})` already holds a different hash for the same sequence number (`\text{DeltaID}(d_a) \neq \text{DeltaID}(d_b)`), both signed deltas are retained as cryptographic evidence, triggering immediate validator role ejection.
* **Committed-Head Pruning:** To maintain bounded memory, index entries behind the latest committed checkpoint sequence (`\text{seq}_c`) are safely pruned.

### 3. Anchor Selection & Direct Commit
* Each round `r` designates a deterministic anchor candidate slot: `\text{slot}(r) = r \pmod N`.
* An anchor `A_r` is **directly committed** when at least `f + 1` deltas in round `r+1` explicitly include `A_r` in their causal history.
* Unreferenced or uncommitted anchors are deterministically derived as skipped without requiring additional network consensus rounds.

### 4. Canonical Sorting & Future-Beacon Salting
Once an anchor `A_r` is committed, its uncommitted causal history forms a DAG segment:
1. **Topological Extraction:** The segment's deltas are extracted in causal DAG order.
2. **Salted Sorting:** Within identical heights, deltas are ordered by `\text{Hash}(\text{DeltaID} \parallel \text{Beacon}_{r})`, where `\text{Beacon}_{r}` is a round-indexed future beacon. This prevents malicious proposers from grinding hashes to manipulate transaction priority.

### 5. Deterministic Execution & Conditional Finality
* Nodes execute the linear sequence deterministically:      
  `S_{t+1} = \text{Execute}(\text{Canonicalize}(\mathcal{D}), S_t)`.
* **Conflict Resolution:** In the event of conflicting transactions or double-spend attempts, state transitions are processed in exact canonical sequence; conflicting later transactions trigger explicit execution reverts (`STATUS_REVERTED`).
* **Checkpointing:** Periodic quorum-signed checkpoint certificates over committed anchors freeze historical execution prefixes, guaranteeing conditional finality.

## Citation
#### Academic Reference
If you use or reference **Proof of Invariant (PoI)**, please cite the project:
```bibtex
@misc{shvets2026poi,
  author       = {Shvets, Oleksandr A.},
  title        = {{Proof of Invariant (PoI): Structural Invariant Verification in Causal DAG Ledgers}},
  year         = {2026},
  howpublished = {\url{https://github.com/ugodai/invariant}},
  note         = {Technical specification}
}
```
Repository: <https://github.com/ugodai/invariant>

## License

Copyright © 2026 Oleksandr A. Shvets   
This draft paper is licensed under the Creative Commons Attribution-NonCommercial 4.0 International License (CC BY-NC 4.0). The author intends to release the final version under a less restrictive license (such as CC BY 4.0).
