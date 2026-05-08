# Certum

Four parsers. C, Rust, Go, Solidity. One IR.

From the IR we emit Dafny with bounded integer subset types, so Z3 catches overflow directly. The hard part was the CEGIS loop: verify with Dafny, Z3 spits out a counterexample, we feed it back into the semantic layer to refine the invariants, re-emit, re-verify. Rinse until proof or bug.

When it verifies, the same IR emits Lean 4, Coq, F*, and ACSL artifacts; those become external-checker claims only when their native checkers accept them.

---

## Pipeline

Source hits a compiler frontend (Clang AST for C, rustc MIR for Rust, solc for Solidity, hand-written recursive-descent for Go). The frontend emits into a typed, source-spanned formal IR. Every construct carries a memory region tag, stack, heap, static, MMIO, ghost, external.

The memory model runs separation logic over a flat byte-addressable array. Pointer pairs get classified into an aliasing matrix: MUST_ALIAS, MAY_ALIAS, NO_ALIAS. Struct layouts computed with natural alignment. Double pointers resolved recursively. Use-after-free, null dereference, signed overflow, flagged as UB.

Spec generation is deterministic. Interval analysis infers integer ranges via abstract interpretation over the interval domain. Loop invariants are pattern-matched from induction variables and guard conditions. Symbolic execution collects path conditions. External calls hit the harness library — 50 built-in models for libc, allocators, syscalls, crypto primitives, networking, threading, blockchain environment, serialization. Every harness is an explicit assumption in the receipt.

The CEGIS loop runs `parse -> ir -> semantic -> emit -> verify -> refine -> repeat`. Z3 returns SAT with concrete counterexample values. The invariant refiner strengthens annotations for the next iteration. Sixteen bug classes, each with its own refinement strategy. Loop terminates when Z3 says UNSAT on all paths or when it can't, and you have a real bug.

---

## What it verifies

**Integer overflow.** Every integer type maps to a bounded Dafny subset. `int32_t` becomes `x: int | -2^31 <= x < 2^31`. Z3 finds the exact overflow input — which multiplication, which addition, at what values.

**Out-of-bounds access.** Array, slice, and pointer accesses carry explicit bounds checks. If Z3 can't prove `0 <= i < arr.Length`, you get the exact index and bound.

**Pointer aliasing.** Conservative and sound. `char*`/`void*` alias everything. Same base type may alias. Assignment creates alias relationships. Frame conditions: non-aliased pointers get `requires p != q` and `modifies only {written_set}`.

**Concurrency.** Thread interleaving via `reads`/`modifies` clauses. Race detection: two writers on shared memory without synchronization. Deadlock detection: lock order graph cycle analysis. Goroutines, async/await, channels, atomics.

**Distributed consensus.** Raft as a Dafny ghost state machine. Quorum intersection: any two majorities share at least one node — proved by cardinality. Leader election safety: at most one leader per term.

**Cryptographic protocols.** Symbolic model with perfect-crypto axioms and Dolev-Yao adversary. Signal Double Ratchet, TLS 1.3. Forward secrecy, replay protection. Symbolic model means the protocol logic is verified assuming the crypto is perfect.

---

## Proof receipts

Every verification produces a JSON receipt:

```
source_hash      SHA-256 of original source
normalized_hash  SHA-256 after strip + normalize
ir_hash          Deterministic hash of formal IR
dafny_hash       SHA-256 of generated Dafny
claim_level      VERIFIED / REFUTED / MODEL_BOUNDARY / UNVERIFIED
counterexample   Exact values and execution path
replay_command   Shell command to reproduce
```

Replay verifies all hashes match. Run it on your machine. Run it in Docker. Same source, same hashes, same result.

Receipts distinguish implementation verification, symbolic models, emitted artifacts, external checker results, and model boundaries, so no generated artifact is presented as stronger than the evidence attached to it.

---

## Multi-backend

Dafny/Z3 is primary. The same IR cross-checks against CBMC (bounded model checking for C), Frama-C/WP (ACSL contracts), direct SMT-LIB2 export, and Kani (Rust). Disagreement between backends nullifies the claim and triggers deeper investigation.

---

## Stack

`Clang / rustc MIR / solc AST -> Formal IR -> Interval Analysis + Separation Logic -> CEGIS -> Dafny 4.8.1 / Z3 4.12.2 -> CBMC / Frama-C / SMT-LIB / Kani -> Proof Receipt -> Lean 4 / Coq / F* / ACSL`

Deterministic end-to-end.

---

## License

Proprietary. All rights reserved.
