# Vulnerability Report #2: ConflictingCandidateAndCertificate Misbehavior Proof Contains No Evidence

## Summary

When the Pool detects a Byzantine leader proposing a block candidate that conflicts with an already-existing notarization or finalization certificate, it correctly calls `ConflictingCandidateAndCertificate::create(...)` — but the resulting proof object is **completely empty**. The misbehavior type stores no fields, carries no evidence, and cannot be verified by any validator. This makes the entire `ConflictingCandidateAndCertificate` accountability path non-functional.

## Severity

**High** — The misbehavior detection fires, a report is published, but it is unverifiable. Any receiving validator must reject an empty proof. The Byzantine leader cannot be slashed for presenting a candidate that contradicts a finalized or notarized block.

## Affected Code

**File:** `validator/consensus/simplex/misbehavior.h`, lines 28–36

```cpp
class ConflictingCandidateAndCertificate : public Misbehavior {
 public:
  static MisbehaviorRef create() {
    return td::make_ref<ConflictingCandidateAndCertificate>();
  }

  ConflictingCandidateAndCertificate() {
  }
};
```

The class has **no fields**. Compare with `ConflictingVotes` which correctly stores `vote1_` and `vote2_`:

```cpp
class ConflictingVotes : public Misbehavior {
  // ...
  td::BufferSlice vote1_;
  td::BufferSlice vote2_;
};
```

**File:** `validator/consensus/simplex/pool.cpp` — multiple call sites with missing arguments:

```cpp
// Line 646-647:
return resolve_with(ConflictingCandidateAndCertificate::create(
    /* candidate, last_finalization_cert */));

// Line 656:
return resolve_with(ConflictingCandidateAndCertificate::create(/* candidate, notarization_cert(slot) */));

// Line 666-667:
return resolve_with(ConflictingCandidateAndCertificate::create(
    /* candidate, notarization_cert(first_nonfinalized_slot_ - 1) */));

// Line 676-677:
return resolve_with(ConflictingCandidateAndCertificate::create(
    /* candidate, notarization_cert(slot) */));
```

In all four cases, the evidence (the conflicting candidate and the certificate that proves its invalidity) is mentioned **only in comments** but never actually passed to `create()` because `create()` accepts no arguments.

## Root Cause

The `ConflictingCandidateAndCertificate` class was stubbed out (likely a skeleton created during initial development) and never completed. The `create()` factory was not given parameters, so the four call sites in `maybe_resolve_request()` cannot pass the required evidence even though the local variables holding it are available at each call site.

Specifically, in `maybe_resolve_request()`:
- `request_.candidate_for_proof` holds the conflicting candidate.
- `last_finalized_block_` / `first_nonfinalized_slot_` allow retrieving the relevant certificate.
- `last_final_cert_` holds the last finalization certificate.
- Per-slot notarization certificates are accessible via `slot->state->certs`.

All necessary data for a complete, verifiable proof is available at each call site.

## Impact

1. **Accountability gap**: A Byzantine validator that presents candidates for already-finalized or already-notarized slots cannot be punished.
2. **Subtle consensus confusion**: If a Byzantine leader presents a candidate for a slot whose parent conflicts with the finalized chain, the Pool detects it and resolves the `WaitForParent` request with a misbehavior — but the proof cannot be acted upon. The detection is correct but the consequence is nil.
3. **Cascade risk**: If the misbehavior reporting pipeline does downstream signature verification on the proof, receiving an empty `ConflictingCandidateAndCertificate` could cause crashes or assertion failures depending on how the `Misbehavior` base class is handled.

## Reproduction Scenario

```
Byzantine Setup:
- Slot s has been finalized with block B_s (FinalCert exists).
- Byzantine leader L sends a new candidate C' for slot s with a different parent.

Expected:
- Pool's maybe_resolve_request detects: next_slot_after_parent < first_nonfinalized_slot_.
- Pool creates a ConflictingCandidateAndCertificate proof containing C' and the existing FinalCert.
- Proof is broadcast; L is slashed.

Actual:
- Pool calls ConflictingCandidateAndCertificate::create() with no arguments.
- Resulting MisbehaviorRef contains an empty object.
- No evidence is stored or propagated.
- L faces no consequences.
```

## Fix Direction

1. Add fields to `ConflictingCandidateAndCertificate`:
   ```cpp
   class ConflictingCandidateAndCertificate : public Misbehavior {
    public:
     static MisbehaviorRef create(CandidateRef candidate, td::BufferSlice certificate) {
       return td::make_ref<ConflictingCandidateAndCertificate>(
           std::move(candidate), std::move(certificate));
     }
     // ...
    private:
     CandidateRef candidate_;      // The conflicting candidate (with leader signature)
     td::BufferSlice certificate_; // The certificate that proves the conflict
   };
   ```

2. Update each `create()` call site in `maybe_resolve_request()` to pass the actual candidate and certificate evidence.

## References

- `validator/consensus/simplex/misbehavior.h:28–36` (empty proof class)
- `validator/consensus/simplex/pool.cpp:646,656,666,676` (call sites with missing arguments)
- `validator/consensus/simplex/pool.cpp:14–26` (ConflictingVotes — correct implementation for comparison)
