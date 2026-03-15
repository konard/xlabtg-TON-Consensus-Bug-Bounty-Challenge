# Vulnerability Report #1: Leader Equivocation Goes Unpunished — Misbehavior Proof Never Generated

## Summary

A Byzantine leader can broadcast two different block candidates for the same slot (equivocation) and face **zero on-chain consequences**. The consensus node detects the second candidate, recognizes the conflict, but silently discards it with no misbehavior proof generated or propagated. This fundamentally undermines the slashing/accountability mechanism.

## Severity

**High** — Safety-critical: The entire accountability model for equivocating leaders is non-functional. A Byzantine leader can safely equivocate across multiple nodes to try to cause split notarization decisions, with no risk of being slashed.

## Affected Code

**File:** `validator/consensus/simplex/consensus.cpp`, lines 178–183

```cpp
if (slot->state->pending_block.has_value()) {
  if (slot->state->pending_block.value()->id != candidate->id) {
    // FIXME: Report misbehavior
  }
  return;
}
```

## Root Cause

When `CandidateReceived` is handled in `ConsensusImpl`, the code correctly detects that the incoming candidate has a different `id` than the one already stored for this slot (meaning the same leader sent two different block proposals for slot `i`). This constitutes a clear Byzantine equivocation.

However, the misbehavior reporting path is entirely absent — replaced with a `// FIXME` comment. Both the detection and the proof data (`slot->state->pending_block` contains the first candidate, `candidate` contains the conflicting second) are available at this point. No `MisbehaviorReport` is published. No proof is serialized.

## Impact

1. **Accountability failure**: A Byzantine leader can equivocate with impunity. The network detects the equivocation locally but never propagates a proof, so the leader cannot be slashed.
2. **Potential fork attempts**: The leader can send candidate A to 34% of validators and candidate B to the other 66%, attempting to split notarization decisions. If successful (due to timing or network partitioning), this can violate consistency.
3. **Quadratic scaling risk**: With up to 100 validators, a sophisticated Byzantine leader could send unique candidate variants to different subsets, flooding each validator with spurious `CandidateReceived` events, each of which triggers an async `try_notarize()` coroutine and a `StoreCandidate` I/O operation — but only the first is processed per-slot on each node individually.

## Reproduction Script (Conceptual)

```
Byzantine Setup:
- Validator B is the current leader for slot s.
- B generates two distinct candidates C1 and C2 for slot s (same slot, different parent_id or different block content).
- B sends C1 to validators V[0..49] and C2 to validators V[50..99].

Expected (correct behavior):
- Any validator that receives C2 after already storing C1 should:
  1. Detect C1.id != C2.id for the same slot.
  2. Construct a ConflictingCandidatesBroadcast (or equivalent misbehavior proof) containing both signed candidates.
  3. Broadcast the proof so the leader can be slashed.

Actual (buggy behavior):
- The second candidate is silently dropped (line 182: `return;`).
- No MisbehaviorReport is published.
- The Byzantine leader faces no consequences.
```

## Fix Direction

At the `// FIXME: Report misbehavior` site, a `ConflictingVotes`-style proof (or a new `ConflictingCandidates` misbehavior type) should be created using both `slot->state->pending_block.value()` (the first candidate) and `candidate` (the conflicting one), then published via `owning_bus().publish<MisbehaviorReport>(candidate->leader, ...)`.

Both candidate objects carry a leader signature (see `Candidate::serialize()` and `CandidateId`), so the proof would be verifiable by all validators.

## References

- `validator/consensus/simplex/consensus.cpp:178–183` (detection without reporting)
- `validator/consensus/simplex/misbehavior.h` (existing misbehavior proof types)
- `validator/consensus/simplex/pool.cpp:128–129` (ConflictingVotes proof for comparison — correctly implemented for votes)
