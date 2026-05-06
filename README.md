# Certum

Four parsers. C, Rust, Go, Solidity. One IR. No tree-sitter, no LLM anywhere in the pipeline.

From the IR we emit Dafny with bounded integer subset types, so Z3 catches overflow directly. The hard part was the CEGIS loop: verify with Dafny, Z3 spits out a counterexample, we feed it back into the semantic layer to refine the invariants, re-emit, re-verify. Rinse until proof or bug.

When it verifies, the same IR drives backends for Lean 4, Coq, F*, and ACSL.

We caught L2 gas fee overflow, packet buffer OOB, and validator memory pool bounds violations. All from plain C source, no annotations.

It's not magic. The IR is the moat. The rest is just a feedback loop between Z3 and a pattern-matching invariant generator.

---

TEST 1: Network packet reader (unchecked offset overflow)

TRANSLATE    C to Dafny         OK: 1178 chars generated
CEGIS        3 iterations max   BUG_FOUND in 0.1s
                                 BOUNDS_VIOLATION: index out of range
RESULT       Bug found. Fix the C before shipping.

TEST 2: Ethereum gas fee calculation (integer overflow)

TRANSLATE    C to Dafny         OK: 999 chars generated
CEGIS        3 iterations max   BUG_FOUND in 0.0s
                                 BOUNDS_VIOLATION: value does not satisfy int64_t constraints
RESULT       Bug found. Z3 found the exact overflow input.

TEST 3: Solana validator memory pool (loop + array)

TRANSLATE    C to Dafny         OK: 1094 chars generated
CEGIS        5 iterations max   VERIFIED in 0.0s
OUTPUT       Lean 4: 20 lines   Coq: 26 lines   F*: 21 lines   ACSL: 36 lines

TEST 4: Correct array initialization

TRANSLATE    C to Dafny         OK: 1023 chars generated
CEGIS        5 iterations max   VERIFIED in 0.0s
OUTPUT       Lean 4: 20 lines   Coq: 26 lines   F*: 21 lines   ACSL: 33 lines

---

Stack: pycparser / recursive-descent to Unified IR to Semantic Analysis to CEGIS to Dafny/Z3 to Lean 4 / Coq / F* / ACSL
