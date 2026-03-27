# Verdict Rules

DeTest reads this file during Steps 8-9 of the pipeline — after forge test runs and output is read. This file defines exactly how to interpret forge output and assign a verdict.

---

## READING FORGE OUTPUT

DeTest always runs tests with -vvvv. The output has a fixed structure:

PASS output looks like:
[PASS] test_reentrancy_proof() (gas: 284719)

FAIL output looks like:
[FAIL: {reason}] test_reentrancy_proof() (gas: 48291)

COMPILATION ERROR looks like:
Error (XXXX): {description}
  --> test/detest/Vault_reentrancy_1.t.sol:{line}:{col}

SUITE SUMMARY looks like:
Suite result: ok. 1 passed; 0 failed; 0 skipped
Suite result: FAILED. 0 passed; 1 failed; 0 skipped

---

## OUTCOME DECISION TREE

Read the forge output. Apply this decision tree in order:

### NODE 1 — Did the file compile?
Look for: "Error" before any test output, "Compiler run failed", "error[EXXXX]"
- YES compilation errors present → go to NODE 2
- NO compilation errors → go to NODE 3

### NODE 2 — Compilation failure
Record the exact compiler error message.
Attempt count < 3 → ITERATE (go to ITERATION RULES)
Attempt count = 3 → verdict = UNCONFIRMED-COMPILE
Record: last compiler error, all 3 attempted test files

### NODE 3 — Did the test function run?
Look for [PASS] or [FAIL] next to the test function name.
- [PASS] found → go to NODE 4
- [FAIL] found → go to NODE 5
- Neither found (test was skipped or not matched) → re-run with correct --match-path and --match-test flags before making any verdict

### NODE 4 — Test passed. Verify the trace.
Read the full call trace from -vvvv output.
Cross-check the trace against pashov's path field: caller → function → state change → impact

Does the trace confirm pashov's described attack path?
- YES trace matches → verdict = CONFIRMED
- NO trace does not match (test passed for a different reason) → verdict = UNCONFIRMED-WRONG-PATH
  Record: what the trace showed vs what pashov described

### NODE 5 — Test failed. Classify the failure reason.
Read the [FAIL: {reason}] message.

Is the reason a forge assertion failure?
Examples: "assertion failed", "assertEq", "assertGt", "[FAIL: panic: arithmetic underflow or overflow]"
→ classify as ASSERTION-FAIL → go to NODE 6

Is the reason an unexpected revert?
Examples: "[FAIL: revert]", "[FAIL: EvmError: Revert]", custom error selector
→ classify as UNEXPECTED-REVERT → go to NODE 7

Is the reason vm.expectRevert() was set but no revert happened?
Example: "[FAIL: call did not revert as expected]"
→ classify as NO-REVERT-WHEN-EXPECTED → go to NODE 8

### NODE 6 — Assertion failure
The exploit ran but did not produce the expected impact.
Possible causes:
- The bug does not exist as described
- The test assertion is wrong (wrong expected value)
- The setup is wrong (wrong initial state)
Read the trace to identify which assert failed and at what values.
Attempt count < 3 → ITERATE, focus the revision on: correct the assertion or fix the setup
Attempt count = 3 → verdict = UNCONFIRMED-ASSERTION
Record: the failed assertion, the actual vs expected values from the trace

### NODE 7 — Unexpected revert
The test reverted somewhere it was not supposed to.
Possible causes:
- Import or dependency missing (contract reverts on construction)
- setUp() failed (target contract constructor reverted)
- Wrong function name or signature (selector mismatch)
- Missing mock (external call reverted because no mock was set up)
Read the trace to find exactly where the revert occurred.
Attempt count < 3 → ITERATE, focus the revision on: fix the revert location identified in trace
Attempt count = 3 → verdict = UNCONFIRMED-ASSERTION
Record: where the unexpected revert occurred

### NODE 8 — No revert when expected
vm.expectRevert() was set but the call did not revert.
This is actually a PASS for proving missing access control or missing validation.
Re-read the test intent:
- If the test was designed to prove a MISSING guard: the call not reverting IS the proof → verdict = CONFIRMED
- If the test was designed to prove a guard EXISTS: the call not reverting means the guard is missing → re-evaluate whether this confirms or contradicts pashov's finding

---

## ITERATION RULES

### WHEN TO ITERATE
Iterate when:
- Compilation error that is fixable (wrong import path, wrong function name, type mismatch)
- Assertion failure where the trace shows the setup was wrong
- Unexpected revert where the trace identifies a fixable cause

Do NOT iterate when:
- Test passed but trace doesn't match (UNCONFIRMED-WRONG-PATH — the test logic is fundamentally wrong)
- The revert trace shows the bug does not exist (the fix pashov suggested is already in the code)
- All 3 attempts have been used

### HOW TO ITERATE
Before writing the revision:
1. Re-read the EXACT error or trace from the failed attempt
2. Re-read the relevant section of the target contract source
3. Identify ONE specific thing to fix — do not rewrite the entire test
4. Write the revision targeting only that fix

Revision naming: same filename, DeTest overwrites the file with the revised version
DeTest keeps a comment at the top of the test file recording attempt history:
// DeTest Attempt 1: [brief description of what was tried]
// DeTest Attempt 2: [what was changed and why]
// DeTest Attempt 3: [what was changed and why]

### MAX ATTEMPTS
Hard limit: 3 attempts per FINDING (1 original + 2 revisions)
Hard limit: 1 attempt per LEAD (no iteration on leads)
After the limit: record verdict and move to next finding. Never get stuck.

---

## VERDICT DEFINITIONS

### CONFIRMED
Condition: forge test returns [PASS] AND trace confirms pashov's attack path
Meaning: the vulnerability exists as described and is demonstrably exploitable
Action: record test file path, paste trace summary into verdict report
Caveat: CONFIRMED means the test passes in a local forge environment. It does not guarantee mainnet exploitability or economic viability.

### UNCONFIRMED-ASSERTION
Condition: forge test returns [FAIL] with assertion error after all attempts
Meaning: DeTest could not construct a passing exploit test
Interpretation: one of three possibilities:
  A) The bug does not exist (pashov false positive)
  B) The bug exists but requires conditions DeTest could not set up correctly
  C) The test logic was wrong in all 3 attempts
Action: record all attempt files, record last error, flag for manual review
WARNING: UNCONFIRMED does not mean safe. Manual review is required.

### UNCONFIRMED-COMPILE
Condition: test file failed to compile after all attempts
Meaning: DeTest could not write a compilable test for this finding
Most common causes: complex import dependencies, proxy contract setups, assembly-heavy contracts
Action: record compiler errors, flag for manual review

### UNCONFIRMED-WRONG-PATH
Condition: forge test returns [PASS] but trace does not match pashov's described exploit path
Meaning: something passes but not the right thing
Action: record both the passing trace and pashov's expected path, flag for manual review
This verdict is rare but important — a test can pass for the wrong reason.

### INCONCLUSIVE
Condition: finding was classified UNTESTABLE or FORK-NO-RPC before any test was written
Meaning: DeTest made no attempt because the class of bug cannot be proven with forge test
Action: record the precise reason from testability-rules.md

### SKIPPED
Condition: finding was a LEAD and the single attempt produced a compilation error
Meaning: DeTest attempted once, could not compile, did not iterate per lead policy
Action: record the compiler error, mark as SKIPPED

---

## SPECIAL CASES

### Test passes immediately on attempt 1
Do not skip trace verification. Always cross-check against pashov's path field.
A quick pass can mean the test logic is trivially correct OR trivially wrong.

### forge test hangs or times out
If forge test does not return within 120 seconds:
- Kill the process
- Classify as UNCONFIRMED-ASSERTION with reason: "test execution timed out — possible infinite loop or excessive fuzz runs"
- Do not iterate a timed-out test without reducing scope first

### Invariant test fails
Forge prints the call sequence that broke the invariant.
Read the sequence carefully — it is the proof.
If the invariant failed: verdict = CONFIRMED (the property violation was demonstrated)
If the invariant held after all runs: verdict = UNCONFIRMED-ASSERTION with note: "invariant held for {runs} runs at depth {depth} — increase runs/depth or reconsider the invariant definition"

### Fuzz test finds a counterexample
Forge prints: "[FAIL: counterexample: calldata=..., args=[...]]"
This is a PASS for our purposes — forge found the breaking input.
Verdict = CONFIRMED
Record the counterexample values in the verdict report — they are the proof.

### Fuzz test passes all runs without finding a counterexample
This does NOT mean the bug doesn't exist. It means the fuzzer didn't find it in N runs.
Verdict = UNCONFIRMED-ASSERTION with note: "fuzz test ran {N} iterations without finding a counterexample — the bug may require specific input values not reached by random fuzzing. Consider adding a concrete unit test with pashov's exact proof values."
