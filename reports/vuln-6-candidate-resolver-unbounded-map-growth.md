# Vulnerability Report #6: CandidateResolver State Map Grows Without Bound — Memory Exhaustion

## Summary

The `CandidateResolverImpl` maintains a `std::map<CandidateId, CandidateState>` called `state_` that is populated whenever a candidate ID is referenced (via `ResolveCandidate`, `StoreCandidate`, or `NotarizationObserved` events) but is **never pruned**. A Byzantine validator can cause this map to grow without bound by broadcasting candidates for arbitrary future slot numbers, exhausting validator memory.

## Severity

**Medium** — Memory exhaustion DoS. The per-entry cost is a `CandidateAndCert` struct plus awaiter vectors; a sustained attack can OOM the validator process.

## Affected Code

**File:** `validator/consensus/simplex/candidate-resolver.cpp`

The map grows in three handlers:

1. `process(ResolveCandidate)` — line 147:
   ```cpp
   CandidateState &state = state_[request->id];
   ```

2. `process(StoreCandidate)` — line 166:
   ```cpp
   auto &state = state_[request->id];
   ```

3. `handle(NotarizationObserved)` — line 193:
   ```cpp
   auto &state = state_[event->id];
   ```

None of these ever remove entries from `state_`. There is no equivalent to `ConsensusState::notify_finalized()` being called on `CandidateResolverImpl`'s map.

## Contrast with Pool and ConsensusImpl

Both `PoolImpl` (via `ConsensusState::notify_finalized()`) and `ConsensusImpl` (via `state_->notify_finalized()`) prune their slot state maps when slots are finalized:

```cpp
// pool.cpp (handle_typed_saved_certificate for FinalCert):
state_->notify_finalized(id.slot);

// ConsensusState::notify_finalized():
first_non_finalized_slot_ = std::max(first_non_finalized_slot_, slot + 1);
while (!slots_.empty() && slots_.begin()->first < first_non_finalized_slot_) {
  slots_.erase(slots_.begin());  // Entries are pruned
}
```

`CandidateResolverImpl` has no such pruning mechanism.

## Attack Vector

The attack leverages the slot-too-new check in `ConsensusImpl` (which prevents overly future candidates from being processed there) but does **not** prevent `CandidateResolverImpl` from accumulating state for these IDs.

However, even under normal conditions, the map grows monotonically:
- Each slot creates an entry in `state_` when its candidate is received.
- When the slot is finalized, `FinalizationObserved` is published by `PoolImpl`, but `CandidateResolverImpl` does **not** subscribe to it.
- The entry for the finalized slot remains in `state_` indefinitely.

With `< 10,000 slots` per session and candidate data per slot including `CandidateRef` (potentially multi-MB block data), this is a guaranteed O(session_length) memory leak per session.

Additionally, a Byzantine leader can submit candidates for future slots (up to `max_leader_window_desync` windows ahead), each creating a `CandidateState` entry with unresolved `resolve_awaiters` — keeping dangling promise objects alive until the session ends.

## Impact

1. **Memory leak per session**: Assuming 10,000 slots and ~1 KB metadata per `CandidateState` (without block data), the map accumulates ~10 MB of unreclaimable metadata per session. With block data cached in-memory, this is much larger.
2. **DoS amplification**: A Byzantine leader generating maximum-size candidates for future slots within the allowed desync window (`max_leader_window_desync`) causes entries to accumulate with unresolved awaiters. Each unresolved entry holds a `std::vector<Promise>` and a `CandidateAndCert` object.
3. **Crash risk**: Over long-running sessions, OOM conditions may cause the validator process to be killed by the OS.

## Fix Direction

Add subscription to `FinalizationObserved` in `CandidateResolverImpl` and prune entries for finalized slots:

```cpp
template <>
void handle(BusHandle, std::shared_ptr<const FinalizationObserved> event) {
  // Prune all entries for slots <= finalized slot
  auto it = state_.begin();
  while (it != state_.end() && it->first.slot <= event->id.slot) {
    // Cancel any pending awaiters before erasing
    for (auto &p : it->second.resolve_awaiters) {
      p.set_error(td::Status::Error(ErrorCode::cancelled, "slot finalized"));
    }
    for (auto &p : it->second.store_awaiters) {
      p.set_error(td::Status::Error(ErrorCode::cancelled, "slot finalized"));
    }
    it = state_.erase(it);
  }
}
```

## References

- `validator/consensus/simplex/candidate-resolver.cpp:147,166,193` (map insertions with no pruning)
- `validator/consensus/simplex/state.h:48–53` (ConsensusState::notify_finalized — correct pruning pattern)
- `validator/consensus/simplex/pool.cpp:800–813` (handle_typed_saved_certificate for FinalCert — notifies ConsensusState)
