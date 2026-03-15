# Vulnerability Report #9: Causality Violation in Block Candidate Silently Dropped — No Misbehavior

## Summary

When a block candidate is received with a `parent_id` whose slot number is greater than or equal to the candidate's own slot number (a causality violation — the "parent" is not actually earlier in the chain), the consensus node detects this and silently drops the candidate with another `// FIXME: report misbehavior` comment. This allows a Byzantine leader to send causality-violating candidates to probe validator behavior or trigger edge cases without any accountability.

## Severity

**Low-Medium** — The candidate is correctly rejected (no safety impact), but the lack of misbehavior reporting allows a Byzantine leader to send unlimited causality-violating candidates with no consequences. It also prevents downstream analysis of Byzantine behavior patterns.

## Affected Code

**File:** `validator/consensus/simplex/consensus.cpp`, lines 173–176

```cpp
if (candidate->parent_id.has_value() && candidate->parent_id->slot >= candidate->id.slot) {
  // FIXME: report misbehavior
  return;
}
```

## Root Cause

The check is correct — a block's parent must come from an earlier slot. If `parent_id.slot >= candidate.slot`, the candidate cannot be a valid successor to its claimed parent. This is a provably Byzantine action because the leader signs the `CandidateId` (which includes both the slot and parent information).

However, unlike the `ConflictingVotes` path (which correctly generates a misbehavior proof), this path has no proof generation.

## Impact

1. **Probe attacks with no cost**: A Byzantine leader can send causality-violating candidates to test validator responses, measure network timing, or stress the `slot_at()` / `pending_block` tracking, with no risk of being detected on-chain.

2. **Slot state interference**: Although the candidate is rejected before `pending_block` is set, the rejection happens after `slot_at()` succeeds and various slot state checks run. A high volume of such candidates could contribute to CPU load.

3. **Interaction with WaitForParent**: The `CHECK` in `PoolImpl::process(WaitForParent)` at line 435:
   ```cpp
   CHECK(!candidate->parent_id.has_value() || candidate->parent_id->slot < candidate->id.slot);
   ```
   This means if somehow a causality-violating candidate reaches `WaitForParent` (e.g., via a different code path), it would trigger a hard crash (`CHECK` failure) rather than a graceful rejection. The current path in `ConsensusImpl` prevents this, but the defense-in-depth is fragile.

## Reproduction Scenario

```
Setup:
- Byzantine leader B is responsible for slot s.
- B creates a candidate for slot s with parent_id.slot = s (self-referential) or parent_id.slot = s+5 (future parent).
- B broadcasts this candidate.

Expected:
- Validators detect the causality violation.
- Validators generate a misbehavior proof (candidate contains B's leader signature).
- B is reported and potentially slashed.

Actual:
- Validators detect the causality violation (line 173).
- Validators silently return (line 175).
- B is never reported. Slot s eventually gets a skip vote.
```

## Fix Direction

Similar to Vulnerability #1, a new misbehavior type or reuse of `InvalidBlockCandidate` (from Vulnerability #4) can be generated here:

```cpp
if (candidate->parent_id.has_value() && candidate->parent_id->slot >= candidate->id.slot) {
  owning_bus().publish<MisbehaviorReport>(
      candidate->leader,
      InvalidBlockCandidate::create(candidate));  // or a dedicated CausalityViolation type
  return;
}
```

## References

- `validator/consensus/simplex/consensus.cpp:173–176` (causality check with FIXME)
- `validator/consensus/simplex/pool.cpp:435` (CHECK that guards against this in Pool — would crash if reached)
