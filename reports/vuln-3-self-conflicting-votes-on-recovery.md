# Vulnerability Report #3: State Machine Bug Allows Local Node to Cast Conflicting Votes

## Summary

A code bug (acknowledged by the developers themselves in a comment) can cause a validator node to sign and broadcast two contradictory votes for the same slot during its normal operation. On recovery (restart), the node detects these self-conflicting votes in the database but uses a `tolerate_conflicts = true` flag as a workaround rather than fixing the root cause. The underlying state machine invariant that prevents double-voting is therefore not reliably enforced.

## Severity

**High** — A validator node casting conflicting votes for the same slot is a safety violation: it simultaneously contributes weight to two incompatible outcomes. If the conflicting votes are observed by other validators, a `ConflictingVotes` misbehavior proof will be generated against the affected validator (who may not even be Byzantine — just buggy).

## Affected Code

**File:** `validator/consensus/simplex/pool.cpp`, lines 532–533

```cpp
LOG_CHECK(validator != owning_bus()->local_id || tolerate_conflicts)
    << "We produced conflicting votes! Conflict occured for " << vote.vote;
```

This `LOG_CHECK` asserts that we (the local validator) should never produce conflicting votes — unless `tolerate_conflicts` is set.

**File:** `validator/consensus/simplex/pool.cpp`, lines 338

```cpp
handle_our_vote(vote, /*tolerate_conflicts=*/true, /*suppress_vote_broadcast=*/true).start().detach();
```

During bootstrap (node restart), all saved votes are replayed with `tolerate_conflicts=true`.

**File:** `validator/consensus/simplex/consensus.cpp`, lines 54–60

```cpp
auto notar_fn = [&](const NotarizeVote& notar_vote) {
  if (slot->state->voted_notar.has_value() && slot->state->voted_notar != notar_vote.id) {
    // Note that a bug might have caused conflicting votes to be casted. In this case, Pool
    // will crash but not before database durably stores the votes, so in bootstrap_votes we
    // might observe conflicts. In such a case, at least let's not corrupt local per-slot
    // invariants.
    LOG(WARNING) << "Dropping corrupted " << notar_vote;
    return;
  }
```

The developer comment explicitly acknowledges: *"a bug might have caused conflicting votes to be cast"*.

## Root Cause Analysis

The `ConsensusImpl` state machine in `consensus.cpp` tracks per-slot voting state: `voted_notar`, `voted_skip`, `voted_final`. These guard against double-voting at the `ConsensusImpl` level. However, the `PoolImpl` in `pool.cpp` maintains a *separate*, independent per-validator voting state (`Tsentrizbirkom`). These two state machines can desynchronize.

Specifically, the sequence that can lead to conflicting votes:

1. `ConsensusImpl` receives a `CandidateReceived` event for slot `s` and dispatches `try_notarize(slot)` as a detached async coroutine.
2. Before `try_notarize()` completes (it awaits `WaitForParent`, then `ResolveState`, then `ValidationRequest` — all async), a `SkipVote` or `FinalizeVote` may be issued for slot `s` by the `alarm()` handler or by `process_notarization_observed()`.
3. When `try_notarize()` finally completes (after the awaits), the `voted_notar` guard prevents issuing a `NotarizeVote` only if `voted_notar` was already set during the awaits. But the vote-type conflict checks in `Tsentrizbirkom::check_invariants()` exist precisely because this race is not perfectly prevented.
4. The conflicting votes are written to the database. On next restart, the node discovers them and uses `tolerate_conflicts=true` to paper over the issue.

## Impact

1. **Inadvertent self-slashing**: A non-Byzantine validator node can generate a `ConflictingVotes` misbehavior proof against itself, which remote nodes will correctly process. Depending on the slashing rules, this causes an honest validator to be penalized.
2. **Safety risk**: If conflicting votes are broadcast (before the crash/restart that triggers `tolerate_conflicts`), they can be observed by other nodes and contribute to conflicting certificate assembly, weakening the 2/3 threshold assumption.
3. **Liveness degradation**: A node that crashes and re-starts with `tolerate_conflicts=true` silently drops one of the conflicting votes, potentially dropping the vote that was already partially propagated, causing its weight to be "split" across two incompatible outcomes in different nodes' views.

## Reproduction Scenario

```
Trigger conditions:
- Validator V is not the leader for slot s.
- V receives a valid candidate C for slot s.
- V's alarm() fires for slot s (timeout) concurrently with the ongoing try_notarize(s) coroutine.
  - Both alarm() and try_notarize() are actor-local, but the async awaits in try_notarize()
    interleave with alarm() callbacks.
- alarm() issues SkipVote{s} and sets voted_skip=true.
- try_notarize() completes its awaits and issues NotarizeVote{C.id}.

Result:
- Both SkipVote{s} and NotarizeVote{C.id} are signed and broadcast by V.
- Tsentrizbirkom::check_invariants() detects: finalize + skip -> ConflictingVotes.
- LOG_CHECK fires; node crashes (if not bootstrap path).
- On restart: tolerate_conflicts=true silently drops the conflicting vote.
```

## Fix Direction

The root fix must ensure the `ConsensusImpl` voting state guards (`voted_notar`, `voted_skip`, `voted_final`) are checked **after** all async awaits in `try_notarize()`, not just before:

```cpp
td::actor::Task<> try_notarize(State::SlotRef slot) {
  // ... awaits ...
  auto validation_result = co_await owning_bus().publish<ValidationRequest>(...);

  // Re-check slot state after all awaits — the slot might have been skipped during awaiting
  if (slot.state->voted_skip || slot.state->voted_notar.has_value()) {
    co_return {};  // Abort: slot state changed while we were awaiting
  }

  slot.state->voted_notar = candidate->id;
  owning_bus().publish<BroadcastVote>(NotarizeVote{candidate->id});
  // ...
}
```

The same check should be applied after `WaitForParent` and `ResolveState` awaits.

## References

- `validator/consensus/simplex/pool.cpp:532–533` (LOG_CHECK acknowledging the bug)
- `validator/consensus/simplex/pool.cpp:338` (tolerate_conflicts workaround on bootstrap)
- `validator/consensus/simplex/consensus.cpp:54–60` (developer comment acknowledging the bug)
- `validator/consensus/simplex/consensus.cpp:210–237` (try_notarize — multiple async awaits without re-checking voted state)
- `validator/consensus/simplex/pool.cpp:209–217` (check_invariants)
