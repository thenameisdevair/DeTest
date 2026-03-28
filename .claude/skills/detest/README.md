# DeTest

**Foundry test verification layer for pashov/skills solidity-auditor findings.**

DeTest takes the output of a pashov security audit and attempts to prove or disprove each finding by writing, running, and iterating on Foundry tests. It does not discover vulnerabilities — it produces evidence for or against findings that already exist.

---

## What's new in v0.2.0

DeTest now works on **any Solidity repo** — Foundry, Hardhat, Truffle, or plain contracts.

It automatically creates an isolated Foundry workspace in /tmp/, copies your contracts in, resolves dependencies, and runs everything there. Your original repo is never modified.

---

## Prerequisites

- [pashov/skills solidity-auditor](https://github.com/pashov/skills) must be installed and must have already run on the target codebase
- Foundry must be installed (`forge --version` should work)
- Internet access for `forge install` (dependency resolution pulls from GitHub)
- For FORK category findings: `eth_rpc_url` must be set in `foundry.toml`

No `foundry.toml` required in the target repo. DeTest creates its own.

---

## Install

```bash
git clone https://github.com/thenameisdevair/DeTest.git
mkdir -p ~/.claude/skills
cp -r DeTest/.claude/skills/detest ~/.claude/skills/detest
```

---

## Usage — New Audit Target (any repo type)

**Step 1 — Clone the target repo**
```bash
cd ~/Documents && git clone https://github.com/{owner}/{repo}.git && cd {repo}
```

**Step 2 — Install pashov skill locally**
```bash
cp -r ~/Documents/lendvest-smart-staking/.claude/skills/solidity-auditor .claude/skills/solidity-auditor
```

**Step 3 — Install DeTest skill locally**
```bash
cp -r ~/.claude/skills/detest .claude/skills/detest
```

**Step 4 — Open Claude Code**
```bash
claude
```

**Step 5 — Run pashov (discovery)**
```
run the solidity auditor with all different agents possible on the codebase
```

**Step 6 — Run DeTest (verification)**
```
/detest
```

DeTest will automatically detect the repo type, set up an isolated Foundry workspace, resolve dependencies, and begin verification.

---

## Supported Repo Types

| Type | Detection | Notes |
|---|---|---|
| Foundry | `foundry.toml` present | Fastest path — remappings reused directly |
| Hardhat | `hardhat.config.ts` or `.js` | Contracts copied from `contracts/` or `src/` |
| Truffle | `truffle-config.js` | Contracts copied from `contracts/` |
| Node | `package.json` only | Imports scanned for dependency detection |
| Plain | Just `.sol` files | Dependencies inferred from import statements |

---

## Dependency Resolution

DeTest resolves dependencies in three tiers:

**Tier 1 — Automatic**
Known packages (OpenZeppelin, Solmate, Solady, Uniswap, Chainlink, Aave, PRB Math, LayerZero, and more) are installed automatically via `forge install`. No user input needed.

**Tier 2 — Interactive**
Unknown packages prompt you once:
```
⚠️  Unknown dependency: @custom/library
    GitHub repo? (owner/repo format, or press Enter to skip):
```

**Tier 3 — Partial scope**
If a dependency cannot be resolved, the affected contracts are excluded from scope. Other contracts continue normally. Excluded contracts produce `INCONCLUSIVE` verdicts with a clear reason.

---

## What DeTest produces

For each pashov finding, DeTest produces one of four verdicts:

| Verdict | Meaning |
|---|---|
| ✅ CONFIRMED | A Foundry test passes and trace confirms the exploit path |
| ❌ UNCONFIRMED | Test attempted up to 3 times, no passing result |
| ⚠️ INCONCLUSIVE | Bug class untestable, no RPC configured, or contract excluded |
| ⏭️ SKIPPED | LEAD finding, single attempt failed to compile |

Results are saved to:
`assets/findings/{project-name}-detest-verification-{YYYYMMDD-HHMMSS}.md`

Test files are written to:
`{workdir}/test/detest/{ContractName}_{bugClass}_{findingNumber}.t.sol`

---

## Invocation modes

```
/detest                        — auto-detect latest pashov report, process all findings
/detest {filename}             — use a specific pashov report file
/detest --finding {N}          — process only finding number N
```

---

## What DeTest cannot do

- Discover new vulnerabilities
- Prove mempool-dependent attacks (front-running, sandwiching)
- Prove off-chain collusion (DVN collusion, solver collusion)
- Replace manual security review
- Guarantee production exploitability from a passing test
- Resolve private GitHub repos as dependencies
- Handle circular contract dependencies

---

## Pipeline

```
pashov /solidity-auditor   →   discovers vulnerabilities
         ↓
/detest                    →   proves or disproves with Foundry tests
         ↓
hackenproof-triage         →   scope check, duplicate check, severity, submission
```

---

## References

- `references/bug-class-map.md` — 46 bug classes mapped to Foundry tools and test patterns
- `references/testability-rules.md` — classification algorithm for each finding
- `references/verdict-rules.md` — how to interpret forge output and assign verdicts
- `references/dependency-map.md` — npm/import names mapped to forge install targets
- `references/workspace-setup-rules.md` — Phase 0 workspace setup algorithm

---

## Versions

| Version | Description |
|---|---|
| v0.2.0 | Isolated Foundry workspace — works on any repo type |
| v0.1.0 | Initial release — Foundry repos only |

---

## Relationship to SPECTRA

DeTest is a standalone skill. SPECTRA (autonomous audit agent) and DeTest share the philosophy that findings must be grounded in passing tests — but they operate independently. DeTest is downstream of pashov. SPECTRA is a full pipeline that includes its own hypothesis generation.
