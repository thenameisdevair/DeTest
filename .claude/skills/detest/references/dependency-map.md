# Dependency Map — npm/import name → forge install target

DeTest reads this file during Phase 0 (workspace setup) to resolve contract dependencies automatically. When a dependency is found in package.json, foundry.toml, remappings.txt, or inferred from import statements, DeTest looks it up here first before asking the user.

---

## LOOKUP FORMAT

Each entry has:
- IMPORT PREFIX: the string that appears in import statements or package.json
- FORGE TARGET: the GitHub owner/repo passed to forge install
- REMAPPING: the exact remapping line to add to foundry.toml
- SKIP: if true, this is a dev tool or JS library — skip silently, do not attempt forge install

---

## KNOWN DEPENDENCIES

### OpenZeppelin

IMPORT PREFIX: @openzeppelin/contracts
FORGE TARGET: OpenZeppelin/openzeppelin-contracts
REMAPPING: @openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
SKIP: false

IMPORT PREFIX: @openzeppelin/contracts-upgradeable
FORGE TARGET: OpenZeppelin/openzeppelin-contracts-upgradeable
REMAPPING: @openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/
SKIP: false

### Solmate / Solady

IMPORT PREFIX: solmate
FORGE TARGET: transmissions11/solmate
REMAPPING: solmate/=lib/solmate/src/
SKIP: false

IMPORT PREFIX: solady
FORGE TARGET: Vectorized/solady
REMAPPING: solady/=lib/solady/src/
SKIP: false

### Uniswap

IMPORT PREFIX: @uniswap/v2-core
FORGE TARGET: Uniswap/v2-core
REMAPPING: @uniswap/v2-core/=lib/v2-core/
SKIP: false

IMPORT PREFIX: @uniswap/v2-periphery
FORGE TARGET: Uniswap/v2-periphery
REMAPPING: @uniswap/v2-periphery/=lib/v2-periphery/
SKIP: false

IMPORT PREFIX: @uniswap/v3-core
FORGE TARGET: Uniswap/v3-core
REMAPPING: @uniswap/v3-core/=lib/v3-core/
SKIP: false

IMPORT PREFIX: @uniswap/v3-periphery
FORGE TARGET: Uniswap/v3-periphery
REMAPPING: @uniswap/v3-periphery/=lib/v3-periphery/
SKIP: false

IMPORT PREFIX: @uniswap/v4-core
FORGE TARGET: Uniswap/v4-core
REMAPPING: @uniswap/v4-core/=lib/v4-core/src/
SKIP: false

IMPORT PREFIX: permit2
FORGE TARGET: Uniswap/permit2
REMAPPING: permit2/=lib/permit2/src/
SKIP: false

### Chainlink

IMPORT PREFIX: @chainlink/contracts
FORGE TARGET: smartcontractkit/chainlink
REMAPPING: @chainlink/contracts/=lib/chainlink/contracts/
SKIP: false

### Aave

IMPORT PREFIX: @aave/core-v3
FORGE TARGET: aave/aave-v3-core
REMAPPING: @aave/core-v3/=lib/aave-v3-core/contracts/
SKIP: false

IMPORT PREFIX: aave-v3-core
FORGE TARGET: aave/aave-v3-core
REMAPPING: aave-v3-core/=lib/aave-v3-core/
SKIP: false

### PRB Math

IMPORT PREFIX: @prb/math
FORGE TARGET: PaulRBerg/prb-math
REMAPPING: @prb/math/=lib/prb-math/src/
SKIP: false

### LayerZero

IMPORT PREFIX: @layerzerolabs/lz-evm-protocol-v2
FORGE TARGET: LayerZero-Labs/LayerZero-v2
REMAPPING: @layerzerolabs/lz-evm-protocol-v2/=lib/LayerZero-v2/packages/layerzero-v2/evm/protocol/contracts/
SKIP: false

### forge-std (always required)

IMPORT PREFIX: forge-std
FORGE TARGET: foundry-rs/forge-std
REMAPPING: forge-std/=lib/forge-std/src/
SKIP: false

---

## SKIP LIST — dev tooling, JS libraries, non-Solidity packages

These appear in package.json but have no forge equivalent. DeTest silently skips them.

hardhat
hardhat-gas-reporter
hardhat-etherscan
@nomicfoundation/hardhat-toolbox
@nomicfoundation/hardhat-ethers
@nomicfoundation/hardhat-chai-matchers
@nomiclabs/hardhat-ethers
@nomiclabs/hardhat-waffle
@nomiclabs/hardhat-etherscan
ethers
ethers-v6
viem
wagmi
chai
mocha
typescript
ts-node
dotenv
prettier
prettier-plugin-solidity
solhint
eslint
@typechain/hardhat
typechain

---

## UNKNOWN DEPENDENCY HANDLING

If an import prefix is not in this file:

1. Extract the prefix — everything before the first / in the import path
   Example: import "@custom/library/Token.sol" → prefix is @custom/library

2. Check if it matches any entry in the SKIP LIST → skip silently

3. If not in SKIP LIST → prompt the user:
   ⚠️  Unknown dependency: {prefix}
       Not in dependency-map.md.
       GitHub repo? (format: owner/repo, or press Enter to skip):

4. If user provides owner/repo:
   - Run: forge install {owner/repo} --no-commit
   - Generate remapping by inspecting the installed lib/ directory structure
   - Add to workdir foundry.toml remappings
   - Continue

5. If user presses Enter (skips):
   - Mark all contracts that import this prefix as EXCLUDED
   - Any pashov finding referencing an EXCLUDED contract gets verdict: INCONCLUSIVE
   - Reason: "dependency {prefix} could not be resolved — contract excluded from scope"
   - Continue with remaining contracts

6. If forge install fails even after user provides owner/repo:
   - Same as skip — mark as EXCLUDED, document the forge error, continue

---

## ADDING NEW ENTRIES

When DeTest successfully resolves an unknown dependency via user input, it should note at the end of its run:

"Consider adding {prefix} → {owner/repo} to dependency-map.md for future runs."

DeTest does not auto-modify this file — the user updates it manually.
