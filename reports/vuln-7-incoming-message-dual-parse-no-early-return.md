# Vulnerability Report #7: Incoming Protocol Messages Parsed Twice — CPU Waste and Ambiguous TL Encoding Risk

## Summary

In `PoolImpl::handle(IncomingProtocolMessage)`, every incoming message is parsed first as a `tl::vote` and then **unconditionally** as a `tl::certificate`, regardless of whether the first parse succeeded. This means every valid vote message is also subjected to a failed certificate deserialization attempt. More critically, if a crafted message can be parsed as both types simultaneously (due to TL encoding ambiguity), both processing paths execute — a vote is recorded **and** a certificate is processed from the same bytes.

## Severity

**Medium** — The dual-parse creates both a performance issue (wasted CPU on every received vote) and a potential logic integrity issue if TL encoding is not strictly disjoint.

## Affected Code

**File:** `validator/consensus/simplex/pool.cpp`, lines 371–424

```cpp
void handle(BusHandle, std::shared_ptr<const IncomingProtocolMessage> message) {
    auto maybe_tl_vote = fetch_tl_object<tl::vote>(message->message.data, true);
    if (maybe_tl_vote.is_ok()) {
      // ... parse and handle vote ...
      handle_vote(message->source.get_using(bus), std::move(vote));
    }
    // ⚠️ No early return here — continues to certificate parsing even if vote was valid

    auto maybe_tl_certificate = fetch_tl_object<tl::certificate>(message->message.data, true);
    if (maybe_tl_certificate.is_ok()) {
      // ... parse and handle certificate ...
      handle_certificate(maybe_tl_certificate.move_as_ok()).start().detach();
    }
  }
```

**Critical observation**: After a successful vote parse and `handle_vote()` call, the code does **not** `return`. It falls through to the certificate parsing attempt using the same `message->message.data`.

## Analysis

### Performance Impact (Confirmed)

Every valid vote message results in:
1. `fetch_tl_object<tl::vote>()` — success, O(data_size) parse
2. `handle_vote()` — vote recorded
3. `fetch_tl_object<tl::certificate>()` — **failure** (different TL tag), O(data_size) parse attempt

This doubles the deserialization CPU cost for every vote. Under high validator counts (100 validators × 3 vote types × ~10,000 slots), this represents significant wasted computation.

### TL Encoding Ambiguity Risk (Potential)

TL (Type Language) uses 4-byte magic numbers to distinguish types. If `tl::vote` and `tl::certificate` have different magic numbers (which they should in a correct TL schema), crafted ambiguity is impossible. However, the risk depends on:

1. Whether the TON TL schema is machine-generated and guaranteed disjoint (likely, but not verified here).
2. Whether malformed data can pass both `fetch_tl_object<tl::vote>()` and `fetch_tl_object<tl::certificate>()` in some edge case of the deserializer (e.g., permissive parsing modes).

The `true` parameter in `fetch_tl_object<T>(data, true)` may indicate "boxed" TL parsing, where the type magic is checked. If so, collision is unlikely. But the design creates an unnecessary attack surface.

### Logic Bug Risk

If a message is processed as both a vote and a certificate:
- A `NotarizeVote` signed by validator V would be recorded in `slot.votes[V]`.
- The same data also interpreted as a certificate would add multiple validator signatures (from `cert->signatures`) to the same slot — potentially including forged entries or weight double-counting.

Even if TL magic prevents the collision today, the lack of an early-return `guard` is a latent design defect.

## Reproduction

```cpp
// A message that successfully parses as tl::vote:
// (This is the normal case for every vote in the network)
auto normal_vote_msg = create_valid_vote_message(...);
// Result:
// 1. Parsed as vote -> handle_vote() called -> OK
// 2. Parsed as certificate -> fetch_tl_object fails -> harmless but wasteful

// Theoretical crafted message (if TL magic is not strictly enforced):
// A message starting with a byte prefix that satisfies both TL parsers.
// Result: Both vote and certificate processing paths execute for the same message.
```

## Fix Direction

Add an early return after successful vote handling:

```cpp
auto maybe_tl_vote = fetch_tl_object<tl::vote>(message->message.data, true);
if (maybe_tl_vote.is_ok()) {
  // ... handle vote ...
  handle_vote(message->source.get_using(bus), std::move(vote));
  return;  // <-- Add this: a message is either a vote OR a certificate, not both
}

auto maybe_tl_certificate = fetch_tl_object<tl::certificate>(message->message.data, true);
// ...
```

This eliminates the redundant deserialization and makes the exclusivity invariant explicit in code.

## References

- `validator/consensus/simplex/pool.cpp:371–424` (dual-parse without early return)
- `validator/consensus/simplex/pool.cpp:386` (handle_vote — called but not followed by return)
