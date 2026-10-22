# Invariant Verification
**An invariant** is a formally defined property or relationship that must remain valid under the permitted operations of a system.    
**Proof of Invariant (PoI)** is a protocol architecture for verifying Structural Invariants in Distributed Ledgers based on Directed Acyclic Graphs (causal DAGs). It is designed to enable:
* **Equivocation detection**: detecting double-signing by the same validator.
* **Deterministic ordering**: establishing a deterministic order for ledger events.
* **Conditional finality**: establishing finality when explicitly defined protocol conditions are satisfied.

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

1. **Causal Delta Graph (Causal DAG)**    
    Transactions and state updates are submitted as content-addressed deltas:
    ```
    d = <ChainID, r, h, P(d), seq, payload, sigma>
    ```
    Each delta references its prior parents, forming a directed acyclic graph (DAG).
2. **Inductive Topological Safety**    
    Graph acyclicity is guaranteed through content-addressed parent references `(P(d))`, explicit self-parent chain rules, and strict round-height progression.
3. **Equivocation Guard**    
    An `O(1)` sequence-indexed guard tracks validator activity and detects multi-branch double-signing attempts without graph traversal overhead.
4. **Anchor-Based Segment Ordering**     
    A deterministic round-and-anchor ordering scheme establishes a global linear sequence across asynchronous views.
5. **Future-Beacon Salting**     
    Per-round future beacons salt transaction identifiers during canonical sorting to neutralize proposer grinding and ordering manipulation.

## Protocol Mechanics (How PoI Works)

PoI processes, orders, and finalizes state transitions through five sequential phases.

#### 1. Invariant and Delta Verification

When a node receives a delta d from validator `v_i`, it recomputes all structural fields before insertion.

##### Self-Chain Rule

`P(d)` must contain exactly one self-parent `s(d)` signed by `v_i`, satisfying:
```
seq(d) = seq(s(d)) + 1
r(d) > r(s(d))
```
##### Round and Quorum Rule

The round of a delta is defined as:
```
r(d) = 1 + max(r(p) for p in P(d))
```
Additionally, `P(d)` must include parent deltas from at least `Q` distinct signers at round `r(d) - 1`.

##### Height Rule

The height of a delta is defined as:
```
h(d) = 1 + max(h(p) for p in P(d))
```
#### 2. `O(1)` Equivocation Tracking and Ejection

To detect validators issuing conflicting deltas (double-signing), nodes maintain an in-memory index:
```
I: (ValidatorID, seq) -> DeltaID
```
If `I(v_i, seq)` already holds a different hash for the same sequence number, such that:
```
DeltaID(d_a) != DeltaID(d_b)
```
both signed deltas are retained as cryptographic evidence, triggering immediate validator role ejection.

##### Committed-Head Pruning

To maintain bounded memory, index entries behind the latest committed checkpoint sequence seq_c are safely pruned.

#### 3. Anchor Selection and Direct Commit

Each round r designates a deterministic anchor candidate slot:
```
slot(r) = r mod N
```
An anchor `A_r` is directly committed when at least `f + 1` deltas in round `r + 1` explicitly include `A_r` in their causal history.

Unreferenced or uncommitted anchors are deterministically derived as skipped, without requiring additional network consensus rounds.

#### 4. Canonical Sorting and Future-Beacon Salting

Once an anchor `A_r` is committed, its uncommitted causal history forms a DAG segment.

##### Topological Extraction

The segment’s deltas are extracted in causal DAG order.

##### Salted Sorting

Within identical heights, deltas are ordered by:
```
Hash(DeltaID || Beacon_r)
```
Here, Beacon_r is a round-indexed future beacon.

This prevents malicious proposers from grinding hashes to manipulate transaction priority.

#### 5. Deterministic Execution and Conditional Finality

Nodes execute the linear sequence deterministically:
```
S_(t+1) = Execute(Canonicalize(D), S_t)
```
##### Conflict Resolution
In the event of conflicting transactions or double-spend attempts, state transitions are processed in the exact canonical sequence. Conflicting later transactions trigger explicit execution reverts (`STATUS_REVERTED`).

##### Checkpointing
Periodic quorum-signed checkpoint certificates over committed anchors freeze historical execution prefixes, guaranteeing conditional finality.

## Citation
#### Academic Reference
If you use or reference **Proof of Invariant (PoI)**, please cite the project:
```bibtex
@techreport{shvets2026poi,
  author      = {Shvets, Oleksandr A.},
  type        = {Cryptographic Protocol Specification},
  title       = {{Proof of Invariant (PoI): Structural Invariant Verification in Causal DAG Ledgers}},
  year        = {2026},
  url         = {https://github.com/ugodai/invariant}
}
```
Repository: <https://github.com/ugodai/invariant>

## License
#### Code License
All code associated with this work will be released under the **Apache License, Version 2.0**.

_Copyright &copy; 2026 **Oleksandr A. Shvets**_   
**The draft paper(s)** is licensed under the Creative Commons **Attribution-NonCommercial** 4.0 International License.     
The author intends to release the final version under a less restrictive license (such as **CC BY 4.0**).
