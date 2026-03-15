# Vulnerability Report #4: Invalid Block Candidate Accepted Without Leader Accountability

## Summary

When block validation fails for a received candidate (i.e., the leader produced a semantically invalid block), the consensus node correctly rejects the block but **does not generate or report any misbehavior** against the leader. A Byzantine leader can repeatedly broadcast invalid block candidates — causing every validator to perform expensive block validation work — with no accountability consequence.

## Severity

**Medium-High** — The combination of silent rejection and lack of misbehavior reporting creates both an accountability gap and a potential resource exhaustion vector.

## Affected Code

**File:** `validator/consensus/simplex/consensus.cpp`, lines 224–228

```cpp
auto validation_result = co_await owning_bus().publish<ValidationRequest>(parent.state, candidate);

if (validation_result.has<CandidateReject>()) {
  LOG(WARNING) << "Candidate " << candidate->id
               << " is rejected: " << validation_result.get<CandidateReject>().reason;
  // FIXME: Report misbehavior
  co_return {};
}
```

## Root Cause

The `ValidationRequest` result includes a reason string when the candidate is rejected (`CandidateReject.reason`). The candidate itself carries the leader's digital signature (the `Signed<CandidateId>` structure within `Candidate`). Both pieces of information are available at the `// FIXME` site.

However, no misbehavior type for "invalid block candidate" exists in `misbehavior.h`. The path from detection to reporting has never been implemented.

## Impact

1. **Accountability gap**: A Byzantine leader can produce invalid block candidates without any on-chain consequence. Repeated invalid proposals across multiple leader windows allow targeted liveness degradation.

2. **Resource exhaustion (superlinear DoS)**: The `try_notarize()` coroutine for each received candidate performs:
   - `StoreCandidate` (disk I/O)
   - `WaitForParent` (depends on skip certificate availability — may wait)
   - `ResolveState` (state resolution)
   - `ValidationRequest` (full block validation — computationally expensive)

   A Byzantine leader can produce `slots_per_leader_window` invalid candidates per window, each costing O(block_size) validation work on all validators. With 100 validators and no slashing disincentive, this is a sustainable DoS.

3. **Slot starvation**: Each invalid candidate for slot `s` sets `pending_block` for that slot, preventing any other candidate (even from a legitimate subsequent leader) from being considered for that slot, since:
   ```cpp
   if (slot->state->pending_block.has_value()) {
     // second candidate is dropped (or equivocation FIXME)
     return;
   }
   ```
   This effectively allows a Byzantine leader to "poison" a slot by sending an invalid candidate first, ensuring the slot ends in a skip.

## Reproduction Scenario

```
Setup:
- Byzantine validator B is the leader for window W (slots s to s+N-1).
- B generates cryptographically valid but semantically invalid block candidates
  (e.g., with wrong prev_block_hash, invalid transactions, or wrong seqno).
- B broadcasts these invalid candidates for each slot in W.

Expected (correct behavior):
- Validators validate each candidate, detect it is invalid.
- Each validator generates a "InvalidBlockCandidate" misbehavior proof containing:
  - The candidate (with B's leader signature proving B produced it)
  - The validation failure reason
- Proof is broadcast; B is slashed.
- Skip votes are issued for poisoned slots.

Actual (buggy behavior):
- Validators perform expensive validation (ValidationRequest).
- Each validator silently drops the invalid candidate (co_return {}).
- No MisbehaviorReport is published.
- Slot s is marked with pending_block = invalid_candidate, blocking future candidates.
- Skip votes are eventually issued (by alarm()), wasting the slot.
- B faces no consequences and retains its leader position for the next cycle.
```

## Fix Direction

1. Define a new misbehavior type `InvalidBlockCandidate`:
   ```cpp
   class InvalidBlockCandidate : public Misbehavior {
    public:
     static MisbehaviorRef create(CandidateRef candidate) {
       return td::make_ref<InvalidBlockCandidate>(std::move(candidate));
     }
    private:
     CandidateRef candidate_;  // Carries leader's signature — verifiable
   };
   ```

2. At the `// FIXME: Report misbehavior` site in `try_notarize()`:
   ```cpp
   if (validation_result.has<CandidateReject>()) {
     owning_bus().publish<MisbehaviorReport>(
         candidate->leader, InvalidBlockCandidate::create(candidate));
     co_return {};
   }
   ```

## References

- `validator/consensus/simplex/consensus.cpp:224–228` (validation rejection without misbehavior)
- `validator/consensus/simplex/consensus.cpp:183–189` (pending_block is set before validation — slot "poisoning")
- `validator/consensus/simplex/misbehavior.h` (no InvalidBlockCandidate type exists)
