# Workspace Setup Rules — Phase 0

DeTest reads this file at the start of every run before touching anything. Phase 0 creates an isolated Foundry workspace from any repo type — Foundry, Hardhat, or plain contracts. The original repo is never modified.

---

## PHASE 0 OVERVIEW

Phase 0 has 7 steps executed in order. If any step fails, DeTest reports the failure precisely and stops. It never proceeds to classification or test writing with a broken workspace.

STEP 0.1 — Detect repo type
STEP 0.2 — Create isolated working directory
STEP 0.3 — Copy contracts into workspace
STEP 0.4 — Detect dependencies
STEP 0.5 — Resolve dependencies
STEP 0.6 — Generate foundry.toml
STEP 0.7 — Verify workspace with forge build

---

## STEP 0.1 — DETECT REPO TYPE

Check for the following files in the target repo root, in order:

1. foundry.toml → FOUNDRY repo
2. hardhat.config.ts or hardhat.config.js → HARDHAT repo
3. truffle-config.js → TRUFFLE repo
4. package.json (with no hardhat config) → NODE repo
5. None of the above → PLAIN repo (just .sol files)

Print detection result:
```
🔍 Repo type detected: {FOUNDRY | HARDHAT | TRUFFLE | NODE | PLAIN}
```

For FOUNDRY repos: read existing foundry.toml remappings and src path — use them directly in workspace setup. This is the fastest path.

For all other types: proceed through full workspace setup.

---

## STEP 0.2 — CREATE ISOLATED WORKING DIRECTORY

Run:
```bash
mkdir -p /tmp/detest-{project-name}-{YYYYMMDD-HHMMSS}
```

Store this path as {workdir}. All subsequent operations happen inside {workdir}.

Run inside {workdir}:
```bash
forge init --no-commit --no-git .
```

This creates the standard Foundry structure:
- {workdir}/src/
- {workdir}/test/
- {workdir}/lib/
- {workdir}/foundry.toml
- {workdir}/script/

Delete the default Counter.sol and Counter.t.sol that forge init creates — they are noise.

Print:
```
📁 Workspace created: {workdir}
```

---

## STEP 0.3 — COPY CONTRACTS INTO WORKSPACE

Find all .sol files in the target repo. Apply the same exclude pattern as pashov:
- Skip: interfaces/, lib/, mocks/, test/, node_modules/
- Skip: *.t.sol, *Test*.sol, *Mock*.sol

Copy all included .sol files into {workdir}/src/ preserving directory structure.

Example:
- target repo: src/vault/Vault.sol → {workdir}/src/vault/Vault.sol
- target repo: contracts/Token.sol → {workdir}/src/Token.sol

For repos where contracts live in contracts/ (Hardhat default) instead of src/:
- Copy from contracts/ into {workdir}/src/
- Flatten one level if needed to preserve relative imports

Print:
```
📋 Contracts copied: {N} files
   Source path: {original path}
   Workspace path: {workdir}/src/
```

---

## STEP 0.4 — DETECT DEPENDENCIES

Collect dependency information from all available sources. Read them all — do not stop at the first one found.

SOURCE A — foundry.toml (if FOUNDRY repo)
Read [profile.default] remappings array. Each entry is already a valid remapping. Store all of them.

SOURCE B — remappings.txt (if present in repo root)
Each line is a remapping in format: prefix/=path/. Store all of them.

SOURCE C — package.json (if present)
Read dependencies and devDependencies. Extract all package names. Cross-reference against dependency-map.md. Separate into:
- KNOWN: in dependency-map.md and SKIP is false → needs forge install
- SKIP: in SKIP LIST → ignore silently
- UNKNOWN: not in dependency-map.md → needs user resolution

SOURCE D — Import statement scanning (always run this)
Read every .sol file copied into {workdir}/src/. Extract every import statement. Parse the import prefix (everything before the first / after any @ symbol). Cross-reference against dependency-map.md. Add any new UNKNOWN prefixes not already found in SOURCE C.

Print dependency summary:
```
📦 Dependencies detected:
   Known (auto-resolve):   {N} packages
   Skip (dev tooling):     {N} packages
   Unknown (need input):   {N} packages
```

---

## STEP 0.5 — RESOLVE DEPENDENCIES

### 5a — Install forge-std first (always)
```bash
cd {workdir} && forge install foundry-rs/forge-std --no-commit
```
forge-std is always required. Install it unconditionally regardless of what the source repo uses.

### 5b — Install known dependencies
For each KNOWN dependency from dependency-map.md:
```bash
cd {workdir} && forge install {FORGE TARGET} --no-commit
```

If forge install succeeds: store the REMAPPING from dependency-map.md.
If forge install fails: mark as EXCLUDED, record the forge error, continue to next dependency. Do not stop the run.

Print after each install:
```
  ✅ {IMPORT PREFIX} → installed
  ❌ {IMPORT PREFIX} → failed: {forge error — one line}
```

### 5c — Resolve unknown dependencies interactively
For each UNKNOWN dependency, prompt the user once:
```
⚠️  Unknown dependency: {prefix}
    Not in dependency-map.md.
    GitHub repo? (owner/repo format, or press Enter to skip):
```

If user provides owner/repo:
- Run: forge install {owner/repo} --no-commit
- If succeeds: inspect {workdir}/lib/{repo-name}/ structure to determine the correct remapping
  - Look for src/, contracts/, or root .sol files to determine the base path
  - Generate remapping: {prefix}/=lib/{repo-name}/{detected-base-path}/
- If fails: mark as EXCLUDED, record error

If user presses Enter:
- Mark prefix as EXCLUDED
- Note: "Consider adding to dependency-map.md for future runs"

### 5d — Mark affected contracts as EXCLUDED
For each EXCLUDED dependency:
- Scan all .sol files in {workdir}/src/
- Identify every file that imports the excluded prefix
- Mark those files as EXCLUDED
- Any pashov finding whose contract field matches an EXCLUDED file gets verdict INCONCLUSIVE with reason: "Contract excluded from scope — dependency {prefix} could not be resolved"

Print exclusion summary if any:
```
⚠️  Excluded from scope ({N} contracts):
    {filename} — missing dependency: {prefix}
```

---

## STEP 0.6 — GENERATE foundry.toml

Write {workdir}/foundry.toml with:

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
test = "test"
remappings = [
  "forge-std/=lib/forge-std/src/",
  {all successfully resolved remappings},
]

[fuzz]
runs = 512

[invariant]
runs = 256
depth = 500
fail_on_revert = false
```

For FOUNDRY repos: merge the original remappings with newly installed ones. Never duplicate a remapping entry.

Never set ffi = true.
Never copy compiler version from original repo — let forge detect it from pragma statements.

---

## STEP 0.7 — VERIFY WORKSPACE WITH forge build

Run:
```bash
cd {workdir} && forge build 2>&1
```

### If forge build succeeds:
```
✅ Workspace ready. forge build passed.
   Contracts in scope: {N}
   Excluded contracts: {N}
   Proceeding to classification.
```
Phase 0 complete. Proceed to STEP 1 (load references).

### If forge build fails:
Read the compiler errors carefully. Classify each error:

ERROR TYPE A — Import not found
The error names a specific import path that could not be resolved.
Action: identify which prefix is missing, add it to the UNKNOWN list, go back to STEP 0.5c for that specific prefix. One retry only.

ERROR TYPE B — Syntax error or version mismatch
The error is in a .sol file itself (not an import issue).
Action: mark that specific contract as EXCLUDED, remove it from {workdir}/src/, run forge build again. If build passes after removal: continue with reduced scope. Document the exclusion.

ERROR TYPE C — Widespread failures (more than 30% of contracts failing)
Too many contracts failing to be useful.
Action: stop Phase 0. Print:
```
❌ Workspace setup failed. forge build produced {N} errors across {M} contracts.
   This may indicate a complex monorepo structure or unsupported dependency.

   Errors saved to: {workdir}/build-errors.txt

   Manual resolution options:
   1. Add missing dependencies to .claude/skills/detest/references/dependency-map.md
   2. Run /detest from inside a pre-configured Foundry repo
   3. Manually copy contracts and foundry.toml to a Foundry workspace and run /detest there
```

---

## FAILURE MODES AND RESPONSES

| Failure | Response |
|---|---|
| forge init fails | Stop. Print: "forge not installed or not in PATH. Run: curl -L https://foundry.paradigm.xyz | bash" |
| No .sol files found | Stop. Print: "No Solidity contracts found in target repo." |
| All dependencies EXCLUDED | Stop. Print: "No contracts remain in scope after dependency resolution." |
| forge build ERROR TYPE C | Stop with manual resolution options (see above) |
| Single contract syntax error | Exclude contract, continue |
| forge install rate limited | Wait 5 seconds, retry once. If still fails, mark EXCLUDED. |

---

## WORKSPACE CLEANUP

After DeTest completes its full run (verdict report written):
Print:
```
🧹 Workspace: {workdir}
   Keep for inspection? (yes to keep, Enter to delete):
```

If user presses Enter: run rm -rf {workdir}
If user types yes: leave it. Print the path so user can inspect test files.

---

## PRINT SUMMARY AT END OF PHASE 0

Before proceeding to classification, always print:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚙️  Phase 0 Complete — Workspace Ready
   Repo type:          {type}
   Working directory:  {workdir}
   Contracts in scope: {N}
   Contracts excluded: {N}
   Dependencies installed: {N}
   Dependencies skipped:   {N}
   forge build:        ✅ PASSED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
