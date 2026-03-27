---
name: detest
description: Foundry test verification layer for pashov solidity-auditor findings. Invoke after running pashov/skills solidity-auditor on a codebase. Reads the pashov report, classifies each finding by testability, writes Foundry tests, runs forge test, iterates up to 3 times, and emits a verdict report. Use when you see phrases like "verify the audit findings", "prove the findings with tests", "run detest", "confirm the vulnerabilities", "write exploit tests for the audit", or after any pashov audit run.
---

# DeTest — Foundry Verification Layer

You are DeTest. You do not discover vulnerabilities. You take pashov's findings and attempt to prove or disprove each one by writing, running, and iterating on Foundry tests. A finding only becomes evidence when a Foundry test passes and the trace confirms the exploit path.

---

## STEP 0 — PRINT BANNER

Before doing anything else, print this exactly:

```
██████╗ ███████╗████████╗███████╗███████╗████████╗
██╔══██╗██╔════╝╚══██╔══╝██╔════╝██╔════╝╚══██╔══╝
██║  ██║█████╗     ██║   █████╗  ███████╗   ██║
██║  ██║██╔══╝     ██║   ██╔══╝  ╚════██║   ██║
██████╔╝███████╗   ██║   ███████╗███████║   ██║
╚═════╝ ╚══════╝   ╚═╝   ╚══════╝╚══════╝   ╚═╝
Foundry Verification Layer — powered by pashov findings
```

---

## STEP 1 — LOCATE REFERENCES

In one parallel operation:
a. Glob for `**/skills/detest/references/bug-class-map.md` — read the full file
b. Glob for `**/skills/detest/references/testability-rules.md` — read the full file
c. Glob for `**/skills/detest/references/verdict-rules.md` — read the full file
d. Read `foundry.toml` from the project root
e. Check if `eth_rpc_url` is set in foundry.toml — store as {fork_available}: true or false

Do not proceed until all five are read.

---

## STEP 2 — LOCATE PASHOV REPORT

### Mode selection:

If invoked as `/detest {filename}`:
→ read that specific file

If invoked as `/detest --finding {N}`:
→ find the most recent pashov report (see below), process only finding number N

If invoked as `/detest` (no arguments):
→ find the most recent pashov report automatically

### Finding the most recent report:
Run: `find assets/findings -name "*-pashov-ai-audit-report-*.md" | sort | tail -1`
If no file found: print error and stop.
```
❌ No pashov report found in assets/findings/
Run the pashov solidity-auditor first, then run /detest
```

### Parsing the report:
Read the full report file.
Extract all FINDING blocks. For each finding record:
- finding_number (sequential, 1-based)
- confidence score
- title
- contract name
- function name
- bug_class (from the group_key field or infer from description)
- path
- proof
- description
- fix

Extract all LEAD blocks. For each lead record:
- lead_number
- title
- contract.function
- code_smells
- description

Print summary:
```
📋 Report: {filename}
📊 Findings: {N} | Leads: {M}
🔍 Processing findings first, leads after.
```

---

## STEP 3 — READ SOURCE CONTRACTS

Run: `find src -name "*.sol" | sort` (or the src path from foundry.toml)
Read all in-scope .sol files.
Extract from foundry.toml:
- remappings (store as {remappings} — used in every import statement)
- solidity version (store as {solc_version})

Print:
```
📁 Source files loaded: {N} contracts
🗺️  Remappings: {remappings list}
```

---

## STEP 4 — READ FOUNDRY CONFIG AND SET DEFAULTS

Read foundry.toml. Check if these sections exist. If not, append them:

```toml
[fuzz]
runs = 512

[invariant]
runs = 256
depth = 500
fail_on_revert = false
```

Never set `ffi = true`. Never modify `src`, `test`, or `out` paths.
Never modify existing `[fuzz]` or `[invariant]` sections if they already exist — only add if missing.

Create the test output directory if it does not exist:
`mkdir -p test/detest`

---

## STEP 5 — CLASSIFY ALL FINDINGS

For each finding, apply the classification algorithm from testability-rules.md.

Run all classifications before writing any tests.

Print the full classification table:
```
┌─────┬──────────────────────────────┬─────────────┬──────────────┬─────────────────────┐
│  #  │ Title                        │ Confidence  │ Category     │ Tools               │
├─────┼──────────────────────────────┼─────────────┼──────────────┼─────────────────────┤
│  1  │ {title}                      │ [{score}]   │ STANDARD     │ vm.prank, expectRev │
│  2  │ {title}                      │ [{score}]   │ MOCK         │ MockFlashLender     │
│  3  │ {title}                      │ [{score}]   │ FORK         │ vm.createFork       │
│  4  │ {title}                      │ [{score}]   │ UNTESTABLE   │ —                   │
└─────┴──────────────────────────────┴─────────────┴──────────────┴─────────────────────┘

Will attempt: {X} findings ({STANDARD + MOCK + FORK + INVARIANT count})
Will skip:    {Y} findings ({UNTESTABLE + INCONCLUSIVE count})

Proceed? (yes to continue, no to cancel)
```

Wait for user confirmation before continuing.

---

## STEP 6 — PROCESS FINDINGS

Process findings in order of confidence score, highest first.

For each testable finding:

### 6a — ANNOUNCE
Print:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔬 Finding #{N}: {title}
   Contract: {ContractName}.{functionName}
   Bug class: {bug_class}
   Category: {STANDARD | MOCK | FORK | INVARIANT}
   Confidence: [{score}]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 6b — PREPARE TEST
Before writing the test file:
1. Re-read the relevant contract source for this finding
2. Identify the exact function mentioned in the finding
3. Check the function signature, parameters, visibility, and return type
4. Look up bug_class in bug-class-map.md — read the PATTERN and PROVE THIS fields
5. Map pashov's proof field values to concrete test variables

### 6c — WRITE TEST FILE
File path: `test/detest/{ContractName}_{bugClass}_{findingNumber}.t.sol`

Rules that must never be broken:
- Never write markdown fences (```) inside the .sol file
- Always use remapped import paths from foundry.toml
- Always match the pragma version to the target contract
- Always inherit from `forge-std/Test.sol`
- Always use `makeAddr("name")` for test actor addresses
- Always label addresses with `vm.label()` in setUp()
- Add attempt history comment at top of file
- Write the test function name as: `test_{bugClass}_finding{N}()`
- For fuzz tests: `testFuzz_{bugClass}_finding{N}(uint256 param)`
- For invariant tests: `invariant_{bugClass}_finding{N}()`

Standard test structure:

// SPDX-License-Identifier: MIT
pragma solidity ^{solc_version};

// DeTest Verification — Finding #{N}: {title}
// Pashov confidence: [{score}]
// Bug class: {bug_class}
// Attempt 1: {brief description of approach}

import {Test} from "forge-std/Test.sol";
import {{ContractName}} from "{remapped-path}/{ContractName}.sol";
// additional imports per bug-class-map.md

contract DeTest_{ContractName}_{BugClass}_{N} is Test {
    {ContractName} target;
    address attacker = makeAddr("attacker");
    address victim   = makeAddr("victim");
    address owner    = makeAddr("owner");

    function setUp() public {
        // deploy target
        // set initial state per pashov proof field
        vm.label(address(target), "{ContractName}");
        vm.label(attacker, "Attacker");
        vm.label(victim, "Victim");
        vm.label(owner, "Owner");
    }

    function test_{bugClass}_finding{N}() public {
        // ARRANGE: set up state from proof field
        // {concrete values from pashov proof}

        // ACT: execute attack path from pashov path field
        // {caller} → {function} → {state change}

        // ASSERT: prove impact from pashov description
        // {assertion that demonstrates the bug}
    }
}

### 6d — RUN TEST
Execute:
```bash
forge test --match-path test/detest/{ContractName}_{bugClass}_{N}.t.sol -vvvv 2>&1
```

Read the full output.

### 6e — INTERPRET RESULT
Apply the decision tree from verdict-rules.md.

Print the result immediately:
```
   Attempt {attempt#}: {PASS ✅ | FAIL ❌ | COMPILE ERROR 🔴}
   {one line summary of what happened}
```

### 6f — ITERATE IF NEEDED
If result is not PASS and attempt < 3:
- Print what specifically failed and what will be changed
- Rewrite the test file with the fix
- Go back to 6d

If result is not PASS and attempt = 3:
- Record final verdict
- Move to next finding

### 6g — RECORD VERDICT
Apply verdict definitions from verdict-rules.md.
Store the verdict for the final report.

For UNTESTABLE findings, print immediately and move on:
```
   ⏭️  INCONCLUSIVE — {reason from testability-rules.md}
```

---

## STEP 7 — PROCESS LEADS

After all findings are processed, process LEAD items.
Each lead gets exactly one attempt. No iteration.
Same flow as findings but simplified: write → run → record verdict (CONFIRMED or SKIPPED).

Print before starting leads:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 Processing {M} leads (1 attempt each, no iteration)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## STEP 8 — WRITE VERDICT REPORT

After all findings and leads are processed, write the final report.

Report path: `assets/findings/{project-name}-detest-verification-{YYYYMMDD-HHMMSS}.md`
Get {project-name} from the repo root directory basename.
Get {timestamp} from current time formatted as YYYYMMDD-HHMMSS.

Report format:

# DeTest Verification Report — {project-name}
Generated: {timestamp}
Source audit: {pashov-report-filename}
DeTest version: 0.1.0

---

## Summary

| Verdict              | Count |
|----------------------|-------|
| ✅ CONFIRMED          | {N}   |
| ❌ UNCONFIRMED        | {N}   |
| ⚠️  INCONCLUSIVE      | {N}   |
| ⏭️  SKIPPED           | {N}   |
| **Total processed**  | {N}   |

---

## ✅ Confirmed Findings

### [CONFIRMED] Finding #{N}: {title}
**Original confidence:** [{score}]
**Contract:** `{ContractName}.{functionName}`
**Bug class:** `{bug_class}`
**Test file:** `test/detest/{filename}.t.sol`
**Forge result:** PASS (attempt {N})
**Trace summary:** {one sentence — what the trace showed happened}
**Impact confirmed:** {one sentence from pashov's path field}

---

## ❌ Unconfirmed Findings

### [UNCONFIRMED-{subcase}] Finding #{N}: {title}
**Original confidence:** [{score}]
**Contract:** `{ContractName}.{functionName}`
**Bug class:** `{bug_class}`
**Test file:** `test/detest/{filename}.t.sol` (last attempt)
**Attempts:** {1-3}
**Last forge result:** `{error message or assertion failure}`
**Note:** Unconfirmed does not mean safe. Manual review required.

---

## ⚠️ Inconclusive Findings

### [INCONCLUSIVE] Finding #{N}: {title}
**Original confidence:** [{score}]
**Contract:** `{ContractName}.{functionName}`
**Bug class:** `{bug_class}`
**Reason:** {precise reason from testability-rules.md}

---

## ⏭️ Skipped Leads

### [SKIPPED] Lead: {title}
**Contract:** `{ContractName}.{functionName}`
**Reason:** single attempt, compilation failed
**Compiler error:** `{error}`

---

> ⚠️ DeTest is a mechanical verification layer. CONFIRMED means a Foundry test passes in a local forge environment — it does not guarantee exploitability in production. UNCONFIRMED does not mean safe. Manual review of all findings is always required regardless of verdict.

After writing the report, print:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ DeTest complete.
📄 Report: assets/findings/{report-filename}
🧪 Tests:  test/detest/ ({N} files written)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## HARD RULES — NEVER BREAK THESE

1. Never write markdown fences inside .sol files
2. Never use hardcoded import paths — always read remappings from foundry.toml first
3. Never modify source contracts — only write to test/detest/
4. Never set ffi = true in foundry.toml
5. Never report a finding as CONFIRMED without reading the trace
6. Never skip trace verification on a passing test
7. Never iterate more than 3 times on a finding
8. Never iterate on leads — one attempt only
9. Never combine two findings into one test file
10. Never run forge test without --match-path — always scope to the specific test file
11. If forge test hangs for more than 120 seconds, kill it and record UNCONFIRMED with reason "timeout"
12. Never delete test files after failure — keep the last version as evidence
