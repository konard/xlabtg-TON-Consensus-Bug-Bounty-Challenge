# TON Consensus Bug Bounty — Vulnerability Reports

This directory contains vulnerability reports for the TON Blockchain Simplex consensus implementation (`validator/consensus/` in the `testnet` branch of `ton-blockchain/ton`).

## Findings Summary

| # | Report | Severity | Category | File |
|---|--------|----------|----------|------|
| 1 | [Leader Equivocation Goes Unpunished](./vuln-1-leader-equivocation-no-misbehavior.md) | **High** | Accountability / Byzantine | `simplex/consensus.cpp:178–183` |
| 2 | [ConflictingCandidateAndCertificate Proof is Empty](./vuln-2-conflicting-candidate-and-certificate-empty-proof.md) | **High** | Accountability / Incomplete Implementation | `simplex/misbehavior.h:28–36`, `simplex/pool.cpp:646,656,666,676` |
| 3 | [Node Can Self-Generate Conflicting Votes](./vuln-3-self-conflicting-votes-on-recovery.md) | **High** | Safety / State Machine Race | `simplex/pool.cpp:532–533`, `simplex/consensus.cpp:54–60` |
| 4 | [Invalid Block Candidate — No Leader Accountability](./vuln-4-invalid-block-candidate-no-misbehavior.md) | **Medium-High** | Accountability / DoS | `simplex/consensus.cpp:224–228` |
| 5 | [FEC Broadcast Pre-Filter Bypassed](./vuln-5-check-broadcast-no-validation.md) | **Medium** | DoS / Bandwidth Amplification | `private-overlay.cpp:141–144` |
| 6 | [CandidateResolver Map Grows Without Bound](./vuln-6-candidate-resolver-unbounded-map-growth.md) | **Medium** | Memory Exhaustion / Resource Leak | `simplex/candidate-resolver.cpp:147,166,193` |
| 7 | [Incoming Messages Parsed Twice](./vuln-7-incoming-message-dual-parse-no-early-return.md) | **Medium** | Logic / Performance | `simplex/pool.cpp:371–424` |
| 8 | [Unhandled collate_block Failure](./vuln-8-collate-block-failure-unhandled.md) | **Medium** | Liveness / Error Handling | `block-producer.cpp:115–129` |
| 9 | [Causality Violation Silently Dropped](./vuln-9-causal-violation-silent-drop.md) | **Low-Medium** | Accountability / Byzantine | `simplex/consensus.cpp:173–176` |

## Methodology

All findings are derived from direct source code analysis of the `testnet` branch of `ton-blockchain/ton` at the `validator/consensus/` path (focusing on `simplex/`). Each report includes:

- **Exact file and line numbers** referencing the upstream code
- **Root cause analysis** explaining the bug
- **Impact assessment** describing the attack scenario
- **Reproduction scenario** explaining how to trigger the bug
- **Fix direction** suggesting how to address the issue

No external tooling, fuzzing, or runtime instrumentation was used — these are static analysis findings. All findings can be verified by reading the referenced source files.

## Key Architectural Observations

1. **Misbehavior infrastructure is incomplete**: The `ConflictingVotes` proof type is correctly implemented and used. But `ConflictingCandidateAndCertificate` has no fields and all call sites pass no arguments. Multiple locations use `// FIXME: Report misbehavior` placeholders that have never been implemented.

2. **The tolerate_conflicts bootstrap flag acknowledges a known safety bug**: The developer comment at `pool.cpp:55-58` explicitly states that *"a bug might have caused conflicting votes to be cast"*, and `tolerate_conflicts=true` is used as a recovery workaround rather than fixing the root cause (the async race in `try_notarize()`).

3. **CandidateResolverImpl has no lifecycle pruning**: Unlike `PoolImpl` and `ConsensusImpl`, the candidate resolver never removes finalized entries from its state map.

4. **broadcast pre-filtering is disabled**: The `check_broadcast` hook — specifically designed for early rejection of invalid FEC broadcasts — unconditionally accepts all broadcasts, removing the intended DoS protection layer.
