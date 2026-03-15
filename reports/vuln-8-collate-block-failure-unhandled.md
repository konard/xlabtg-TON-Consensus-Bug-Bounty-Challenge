# Vulnerability Report #8: Unhandled collate_block Failure Silently Kills Leader's Entire Window

## Summary

In `BlockProducerImpl::generate_candidates()`, the `collate_block` call is performed with `co_await` but the result is not wrapped with `.wrap()` to handle errors. If `collate_block` returns an error (e.g., due to resource exhaustion, collator service failure, or network issues), the unhandled exception propagates up and **terminates the entire leader window coroutine** silently. The local validator stops producing blocks for all remaining slots in its leader window without notifying any peer. Peers eventually time out and issue skip votes for all remaining slots.

## Severity

**Medium** — Liveness degradation. This is self-inflicted (no Byzantine actor needed), but is also exploitable: a colluding collator service can deliberately fail `collate_block` to cause the leader to silently drop its window.

## Affected Code

**File:** `validator/consensus/block-producer.cpp`, lines 115–129

```cpp
// FIXME: What to do if collate_block suddenly fails?
CollateParams params{ ... };
auto block_candidate = co_await td::actor::ask(bus.manager, &ManagerFacade::collate_block,
                                               std::move(params),
                                               cancellation_source_.get_cancellation_token());
// ⚠️ No error handling: if collate_block returns an error, co_await throws,
// the coroutine unwinds, and generate_candidates() exits silently.
```

The `// FIXME` comment acknowledges the issue.

## Contrast with Correct Error Handling Pattern

Elsewhere in the codebase, `co_await` calls use `.wrap()` to convert errors into `td::Result`:

```cpp
// candidate-resolver.cpp:271
auto result = co_await resolve_candidate_inner(id, state).wrap();
if (result.is_ok()) { ... } else { /* handle error */ }
```

The `collate_block` await lacks this pattern.

## Impact

1. **Liveness degradation**: Every slot remaining in the leader's window after the failure is silently abandoned. Other validators wait for `first_block_timeout_ms` before issuing skip votes, stalling the chain for up to `slots_per_leader_window * first_block_timeout_ms` milliseconds.

2. **Timeout multiplier escalation**: After a window with all-skip slots, `first_block_timeout_s_` is multiplied by `first_block_timeout_multipler` (consensus.cpp:118):
   ```cpp
   first_block_timeout_s_ = std::min(first_block_timeout_s_ * bus.first_block_timeout_multipler,
                                     bus.first_block_max_timeout_s);
   ```
   Repeated collation failures can escalate timeouts for all subsequent windows, compounding the liveness impact.

3. **Exploitable via colluder**: A malicious collator service (external to the validator, providing blocks via `collate_block`) can deliberately return errors when a specific validator is the leader, causing that validator to silently fail its window repeatedly. This is a targeted liveness attack that does not require the attacker to control a validator key.

## Reproduction Scenario

```
Setup:
- Validator V is the leader for window W.
- V uses an external collator service C.
- C is controlled by an attacker or is simply buggy.

Trigger:
- V's generate_candidates() calls collate_block for slot s.
- C returns an error (timeout, invalid state, or deliberate failure).

Result:
- co_await throws the error.
- generate_candidates() coroutine unwinds.
- current_leader_window_ is NOT reset to nullopt (the cleanup at line 161-163 is unreached).
  Actually: if generate_candidates() exits via exception, the destructors run but
  current_leader_window_ remains set. If OurLeaderWindowStarted fires again, the
  CHECK(current_leader_window_ < event->start_slot) at line 49 may crash.
- No block is produced for slots s through end_slot-1.
- Other validators wait, then skip all abandoned slots.
```

**Potential secondary crash**: If `generate_candidates()` exits via unhandled exception while `current_leader_window_` remains set to `window`, and a new `OurLeaderWindowStarted` event arrives for the same window:
```cpp
void handle(BusHandle, std::shared_ptr<const OurLeaderWindowStarted> event) {
    CHECK(current_leader_window_ < event->start_slot);  // May fail if window unchanged
```

## Fix Direction

Wrap the `collate_block` call with `.wrap()` error handling:

```cpp
auto block_candidate_r = co_await td::actor::ask(bus.manager, &ManagerFacade::collate_block,
    std::move(params), cancellation_source_.get_cancellation_token()).wrap();

if (block_candidate_r.is_error()) {
  LOG(WARNING) << "collate_block failed for slot " << slot << ": " << block_candidate_r.error();
  // Option 1: Skip this slot and continue the window
  owning_bus().publish<BroadcastVote>(SkipVote{slot}).start().detach();  // or signal peers somehow
  ++slot;
  continue;
  // Option 2: Abort the window and let peers time out naturally
  break;
}
auto block_candidate = block_candidate_r.move_as_ok();
```

The chosen recovery strategy (skip slot vs. abort window) should be defined by protocol policy, but either is preferable to silent unhandled failure.

## References

- `validator/consensus/block-producer.cpp:115–129` (FIXME + unhandled co_await)
- `validator/consensus/block-producer.cpp:49` (CHECK that may fail on re-entry)
- `validator/consensus/simplex/consensus.cpp:116–121` (timeout multiplier escalation logic)
- `validator/consensus/simplex/candidate-resolver.cpp:271` (correct .wrap() error handling pattern)
