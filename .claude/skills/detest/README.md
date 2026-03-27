# DeTest

**Foundry test verification layer for pashov/skills solidity-auditor findings.**

DeTest takes the output of a pashov security audit and attempts to prove or disprove each finding by writing, running, and iterating on Foundry tests. It does not discover vulnerabilities — it produces evidence for or against findings that already exist.

---

## Prerequisites

- [pashov/skills solidity-auditor](https://github.com/pashov/skills) must be installed and must have already run on the target codebase
- Foundry must be installed (`forge --version` should work)
- The project must have a valid `foundry.toml`
- For FORK category findings: `eth_rpc_url` must be set in `foundry.toml`

---

## Install

```bash
git clone https://github.com/thenameisdevair/DeTest.git
cp -r DeTest/.claude/skills/detest ~/.claude/skills/detest
```

---

## Usage

Run pashov first:
```
run the solidity auditor with all different agents possible on the codebase
```

Then run DeTest:
```
/detest
```

Or target a specific report:
```
/detest assets/findings/myproject-pashov-ai-audit-report-20250327-120000.md
```

Or target a single finding:
```
/detest --finding 3
```

---

## What DeTest produces

For each pashov finding, DeTest produces one of four verdicts:

| Verdict | Meaning |
|---|---|
| CONFIRMED | A Foundry test passes and trace confirms the exploit path |
| UNCONFIRMED | Test attempted up to 3 times, no passing result |
| INCONCLUSIVE | Bug class cannot be proven with on-chain tests |
| SKIPPED | LEAD finding, single attempt failed to compile |

Results are saved to:
`assets/findings/{project-name}-detest-verification-{YYYYMMDD-HHMMSS}.md`

Test files are written to:
`test/detest/{ContractName}_{bugClass}_{findingNumber}.t.sol`

---

## What DeTest cannot do

- Discover new vulnerabilities
- Prove mempool-dependent attacks (front-running, sandwiching)
- Prove off-chain collusion
- Replace manual security review
- Guarantee production exploitability from a passing test

---

## References

- `references/bug-class-map.md` — maps pashov bug_class tags to Foundry tools and test patterns
- `references/testability-rules.md` — classification algorithm for each finding
- `references/verdict-rules.md` — how to interpret forge output and assign verdicts

---

## Relationship to SPECTRA

DeTest is a standalone skill. SPECTRA (autonomous audit agent) and DeTest share the philosophy that findings must be grounded in passing tests — but they operate independently. DeTest is downstream of pashov. SPECTRA is a full pipeline that includes its own hypothesis generation.
