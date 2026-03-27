# Testability Classification Rules

This file defines the algorithm DeTest uses to classify each pashov finding before writing any test. Classification is a hard gate — no test is written until a category is assigned.

---

## CLASSIFICATION ALGORITHM

### Step 1 — Extract the bug_class field
Read the bug_class tag from the pashov finding. This is the primary classification input.

If bug_class is missing or blank:
- Fall through to Step 2 using the title and description fields
- If those are also insufficient: classify as INCONCLUSIVE, reason: "bug_class missing, insufficient signal for classification"

### Step 2 — Look up bug_class in bug-class-map.md
Open references/bug-class-map.md and find the exact matching entry.

If found: use the CATEGORY from that entry. Proceed to Step 4.

If not found: proceed to Step 3.

### Step 3 — Infer category from proof and path fields
Read the proof field (concrete values/trace) and path field (call sequence) and apply this decision tree:

  Does the exploit require real deployed protocol state (live AMM, live oracle, live lending pool)?
  → FORK

  Does the exploit require a fake external contract that simulates a dependency?
  → MOCK

  Does the exploit require observing mempool ordering, off-chain actors, or async timing between chains?
  → UNTESTABLE

  Does the exploit require only Foundry cheatcodes (vm.prank, vm.warp, vm.deal, vm.store)?
  → STANDARD

  If none of the above can be determined from proof and path:
  → INCONCLUSIVE, reason: "bug_class not mapped, proof insufficient for test construction"

### Step 4 — Apply FORK downgrade check
If classification is FORK:
- Check foundry.toml for eth_rpc_url
- Check environment for FOUNDRY_RPC_URL
- If neither is present: downgrade to INCONCLUSIVE, reason: "fork required, no RPC configured"
- If present: retain FORK classification and record the RPC source

---

## UNTESTABLE PATTERNS — DEFINITIVE LIST

The following patterns can never be proven with on-chain deterministic Foundry execution. Any finding whose core exploit mechanism matches one of these five patterns must be classified UNTESTABLE → INCONCLUSIVE.

### Pattern 1: Mempool and Transaction Ordering
Covers: mempool-front-running, deployment-front-running, sandwich-attack

Forge executes transactions sequentially in a single EVM instance. There is no mempool. Transaction ordering cannot be influenced or observed within a test.

IMPORTANT DISTINCTION: If the underlying vulnerability is a missing parameter (e.g. minAmountOut = 0), that is testable as STANDARD. Only the ordering attack vector itself is UNTESTABLE.

Reason string: "requires mempool observation — transaction ordering cannot be tested in Forge"

### Pattern 2: Validator and Block Producer Manipulation
Covers: validator-prevrandao-manipulation, validator-MEV, block-producer-bias

vm.prevrandao exists and can set the PREVRANDAO value. However, setting it in a test does not prove that a validator can influence it in production — it only proves the code reads the value. The threat model requires a colluding validator, which is an off-chain actor.

Reason string: "requires colluding block producer — vm.prevrandao sets the value but cannot prove production exploitability"

### Pattern 3: Off-Chain Actor Collusion
Covers: dvn-collusion, solver-collusion, oracle-operator-collusion, relayer-collusion

These vulnerabilities require coordinated decisions by multiple off-chain human actors (DVNs, solvers, relayers) who choose to behave maliciously. There is no on-chain proof possible — the attack is a governance/trust assumption, not a code path.

Reason string: "requires off-chain actor collusion — no on-chain proof possible"

### Pattern 4: Async Cross-Chain Timing
Covers: cross-chain-state-lag, lzRead-delivery-race, bridge-finality-race

The vulnerability depends on the timing window between two cross-chain events (e.g. an lzRead query and its lzReceive delivery). Forge executes atomically in a single block context. The async gap between cross-chain messages cannot be simulated — the state can be read but the race condition window cannot be proven.

Reason string: "requires async cross-chain timing window — Forge executes atomically, race condition cannot be simulated"

### Pattern 5: Economic Continuous Properties and Build-Time Concerns
Covers: economic-lvr (loss-versus-rebalancing), bytecode-verification-mismatch, compiler-version-mismatch

Economic LVR: Loss-versus-rebalancing is a continuous property of AMMs relative to external markets over time. The mechanism can be demonstrated but profitability cannot be proven on-chain — it depends on external price feeds and arbitrageur behavior.

Bytecode/build concerns: These are tooling and build pipeline issues, not runtime properties. No on-chain test can verify that deployed bytecode matches source.

Reason string (LVR): "economic property requiring external market comparison — profitability cannot be proven on-chain"
Reason string (build): "build/tooling concern — not a runtime property, no on-chain test possible"

---

## CLASSIFICATION OUTPUT FORMAT

Before writing any test, DeTest prints the full classification summary to the console. The user must confirm before DeTest proceeds.

Format:

```
DeTest — Classification Summary
Source: {pashov-report-filename}
Findings: {total count}

  #  | Title                          | Contract.Function         | Category    | Test Type
 ----|--------------------------------|---------------------------|-------------|----------
  1  | {title}                        | {Contract}.{function}     | STANDARD    | unit
  2  | {title}                        | {Contract}.{function}     | MOCK        | unit
  3  | {title}                        | {Contract}.{function}     | FORK        | unit
  4  | {title}                        | {Contract}.{function}     | STANDARD    | fuzz
  5  | {title}                        | {Contract}.{function}     | UNTESTABLE  | —
  6  | {title}                        | {Contract}.{function}     | INCONCLUSIVE| —

Testable:     {N} findings (STANDARD + MOCK + FORK)
Untestable:   {N} findings (UNTESTABLE + INCONCLUSIVE)

Proceed? [y/n]
```

Processing order: findings are processed highest confidence score first. Within the same score, STANDARD before MOCK before FORK.

LEADs are listed after all FINDINGs. They are labeled LEAD in the Category column and get one attempt only.

---

## EDGE CASES

### Finding has multiple bugs in one
If a single pashov finding describes two independent vulnerabilities:
- Classify based on the primary vulnerability (highest impact sub-bug)
- Note the secondary in the test comments
- Write one test that attempts to prove both in sequence

### Finding is access control + reentrancy combined
Classify as STANDARD. Write a test that:
1. First proves access control missing (can call without privilege)
2. Then proves reentrancy (re-enter during the unguarded call)

### Finding references both local contracts and live protocols
If the core vulnerability is in a local contract but it interacts with a live oracle:
- Classify as MOCK (mock the live oracle with vm.mockCall)
- Only upgrade to FORK if the realistic exploit requires actual live protocol state to be meaningful

### Finding proof field is empty or too vague
If proof field is empty or contains only "TBD" or generic descriptions:
- Classify as INCONCLUSIVE
- Reason: "proof field insufficient — cannot construct test without concrete exploit values or trace"

### Same bug class appears in multiple findings
Each finding gets its own test file. Never combine two findings into one test.
The test file naming includes the finding number: test/detest/{Contract}_{bugClass}_{findingNumber}.t.sol
