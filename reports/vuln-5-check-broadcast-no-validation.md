# Vulnerability Report #5: FEC Broadcast Accepted Without Pre-Validation — Bandwidth Amplification DoS

## Summary

The private overlay's `check_broadcast` callback — designed to pre-filter FEC broadcasts before fully receiving and reassembling them — unconditionally approves all broadcasts with no validation. This allows any validator (or a single Byzantine validator) to flood the network with arbitrarily large, invalid FEC broadcasts that every peer will fully receive and buffer before rejection, enabling a bandwidth amplification attack.

## Severity

**Medium** — Targeted DoS vector against validators. Requires controlling at least one validator key to participate in the private overlay (authorized key set is restricted to current validators).

## Affected Code

**File:** `validator/consensus/private-overlay.cpp`, lines 141–144

```cpp
void check_broadcast(PublicKeyHash, overlay::OverlayIdShort, td::BufferSlice,
                     td::Promise<td::Unit> promise) override {
  promise.set_value(td::Unit());  // Always accepts
}
```

The `check_broadcast` callback in the overlay library is specifically designed as a pre-accept hook: it fires **before** full FEC reassembly, allowing expensive payloads to be rejected early. By unconditionally approving every broadcast, this safety mechanism is completely bypassed.

## Contrast with `on_overlay_broadcast`

The actual content validation only happens in `on_overlay_broadcast` (lines 158–174), **after** the FEC broadcast has been fully received:

```cpp
void on_overlay_broadcast(PublicKeyHash src, td::BufferSlice data) {
  // ...
  auto maybe_candidate = Candidate::deserialize(std::move(data), bus, peer.idx);
  if (!maybe_candidate.is_ok()) {
    LOG(WARNING) << "MISBEHAVIOR: Failed to deserialize block candidate broadcast: "
                 << maybe_candidate.move_as_error();
    return;
  }
  // ...
}
```

The deserialization failure is logged but no misbehavior proof is generated (another `// FIXME` was noted).

## Attack Vector

The private overlay is restricted to validators in the current session (`authorized_keys` map in `start_up()`). However, a single Byzantine validator is sufficient:

```
Setup:
- Byzantine validator B is part of the current validator set for session S.
- B has the right to broadcast to all other validators via the private overlay.
- B constructs maximally large FEC broadcasts (max_broadcast_size =
  max_block_size + max_collated_data_size + 1MB) with invalid content.
- B continuously sends these broadcasts in a loop.

Effect on each honest validator V:
1. check_broadcast() fires — immediately returns OK (no validation).
2. Overlay layer begins receiving FEC fragments for B's broadcast.
3. All fragments are received and buffered.
4. on_overlay_broadcast() fires, attempts Candidate::deserialize() — fails.
5. Data is discarded. No misbehavior proof generated.
6. B immediately sends another broadcast.

Impact:
- Each honest validator receives up to max_broadcast_size bytes per malicious broadcast.
- With 100 validators, B can amplify its outgoing bandwidth by 99x.
- With a single bad actor, this can saturate the private overlay network.
- Block candidates from honest leaders may be delayed or dropped due to overlay congestion.
```

## Quantification

From `start_up()`:
```cpp
td::uint32 max_broadcast_size = bus.config.max_block_size + bus.config.max_collated_data_size + (1 << 20);
```

Assuming typical TON block parameters:
- `max_block_size`: ~1 MB
- `max_collated_data_size`: ~1 MB
- Additional 1 MB buffer

→ `max_broadcast_size` ≈ 3 MB per broadcast

With 99 other validators receiving each broadcast and no rate limiting: B's 3 MB outgoing → 297 MB received by the network per broadcast. Even at a conservative 1 broadcast/second, this is ~297 MB/s of network load on honest validators.

## Fix Direction

1. **In `check_broadcast()`**: Validate the sender's public key and apply basic size/rate limits before accepting:
   ```cpp
   void check_broadcast(PublicKeyHash src, overlay::OverlayIdShort, td::BufferSlice data,
                        td::Promise<td::Unit> promise) override {
     // Only accept from known validator keys
     if (short_id_to_peer_.find(src) == short_id_to_peer_.end()) {
       promise.set_error(td::Status::Error("Unknown sender"));
       return;
     }
     // Apply rate limiting per sender
     if (rate_limiter_.is_exceeded(src)) {
       promise.set_error(td::Status::Error("Rate limit exceeded"));
       return;
     }
     promise.set_value(td::Unit());
   }
   ```

2. **In `on_overlay_broadcast()`**: Generate a misbehavior proof when deserialization fails (requires collecting signed FEC parts, as noted in the existing `// FIXME` comment).

## References

- `validator/consensus/private-overlay.cpp:141–144` (unconditional broadcast acceptance)
- `validator/consensus/private-overlay.cpp:158–174` (late validation in on_overlay_broadcast)
- `validator/consensus/private-overlay.cpp:44–51` (authorized_keys setup — broadcast is restricted to validators)
- `validator/consensus/private-overlay.cpp:44` (max_broadcast_size calculation)
