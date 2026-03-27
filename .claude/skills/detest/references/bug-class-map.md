# Bug Class → Foundry Tool Map

This file maps every pashov bug_class tag to its test category, required Foundry tools, test type, and a precise description of what the test must prove. DeTest reads this file before writing any test.

---

## HOW TO USE THIS FILE

1. Extract bug_class from the pashov finding
2. Find the matching entry below
3. Read: CATEGORY, TOOLS, TEST TYPE, PROVE THIS
4. Write the test accordingly
5. If bug_class is not listed here, use the proof field and path field to infer the closest match

---

## CATEGORY: STANDARD

### reentrancy
TOOLS: attacker contract with malicious fallback/receive + vm.prank + vm.deal
TEST TYPE: unit
PROVE THIS: attacker calls target function, target makes external call before updating state, attacker re-enters and drains funds or double-spends. Assert attacker balance increased beyond what a single honest call would produce.
PATTERN:
  - Deploy AttackerContract that implements receive() or fallback() with re-entry logic
  - vm.deal(attacker, initialAmount)
  - vm.prank(attacker); target.deposit{value: initialAmount}()
  - vm.prank(attacker); target.withdraw() — triggers re-entry in AttackerContract.receive()
  - assertGt(attacker.balance, initialAmount) — proves funds drained

### cross-function-reentrancy
TOOLS: attacker contract + vm.prank + shared state variable tracking
TEST TYPE: unit
PROVE THIS: Function A makes external call before syncing state that Function B reads. Attacker re-enters through Function B during Function A's external call. Assert state inconsistency.

### read-only-reentrancy
TOOLS: attacker contract + vm.mockCall on the view function being manipulated
TEST TYPE: unit
PROVE THIS: External contract's view function (e.g. get_virtual_price) returns transitional/wrong value when called mid-execution. Assert that a protocol using that view function gets incorrect data during the re-entry window.

### erc777-reentrancy
TOOLS: MockERC777 contract with tokensToSend hook + vm.prank
TEST TYPE: unit
PROVE THIS: Deploy MockERC777 that calls back into target during transfer. Target updates state after transfer. Assert double-spend or fund drain.
MOCK NEEDED: MockERC777 implementing IERC777 tokensToSend hook

### erc721-callback-reentrancy
TOOLS: MockERC721Receiver + vm.prank
TEST TYPE: unit
PROVE THIS: Target calls safeTransferFrom or safeMint before updating state. MockERC721Receiver.onERC721Received re-enters and exploits stale state.

### erc1155-callback-reentrancy
TOOLS: MockERC1155Receiver + vm.prank
TEST TYPE: unit
PROVE THIS: Same pattern as erc721 but for ERC1155 onERC1155Received or onERC1155BatchReceived.

### cross-function-reentrancy
TOOLS: attacker contract + two separate entry points
TEST TYPE: unit
PROVE THIS: Two functions share state. Function A's nonReentrant does not protect Function B. Enter through B while A is mid-execution.

### access-control
TOOLS: vm.prank(nonAuthorizedAddress) + vm.expectRevert OR assert success when it should fail
TEST TYPE: unit
PROVE THIS (two sub-cases):
  Sub-case A — function has no guard: vm.prank(attacker); target.privilegedFunction() — assert it SUCCEEDS when it should revert
  Sub-case B — wrong guard: vm.prank(wrongRole); target.privilegedFunction() — assert it SUCCEEDS when it should revert
NOTE: vm.expectRevert() tests that something DOES revert. To prove missing access control, assert the call SUCCEEDS without expectRevert — the passing test IS the proof.

### tx-origin-auth
TOOLS: AttackerContract that calls target as intermediary + vm.prank
TEST TYPE: unit
PROVE THIS: target uses require(tx.origin == owner). Deploy AttackerContract. vm.prank(owner); attacker.attack() — inside attack(), call target.privilegedFunction(). tx.origin is owner, msg.sender is attacker. Assert the call succeeds through the intermediary.

### missing-modifier
TOOLS: vm.prank(address(0xdead)) — any non-privileged address
TEST TYPE: unit
PROVE THIS: Call state-changing function from arbitrary address. Assert it succeeds and state changes when it should be restricted.

### integer-overflow
TOOLS: fuzz test with uint256 parameters + bound()
TEST TYPE: fuzz
PROVE THIS: Arithmetic operation produces incorrect result at boundary values. Use bound() to focus on values near type(uintN).max. Assert the result is incorrect or reverts unexpectedly.
ALSO: Add a concrete unit test with the exact values from pashov's proof field.

### integer-underflow
TOOLS: fuzz test + bound() near zero values
TEST TYPE: fuzz
PROVE THIS: Subtraction produces incorrect result when b > a in unchecked block. Assert result is astronomically large (wraps) or causes downstream incorrect behavior.

### unsafe-downcast
TOOLS: unit test with exact overflowing value from proof field
TEST TYPE: unit
PROVE THIS: uint128(largeUint256) truncates silently. Set up state where a uint256 value exceeds type(uint128).max, cast it, assert the stored value is wrong.

### precision-loss
TOOLS: fuzz test with bound() + concrete unit test
TEST TYPE: fuzz + unit
PROVE THIS: Division before multiplication causes truncation. Assert (a / b) * c != (a * c) / b for values in the proof field.

### off-by-one
TOOLS: unit test at the exact boundary value
TEST TYPE: unit
PROVE THIS: Loop or range check includes/excludes the boundary incorrectly. Test with length, length-1, and length+1. Assert the unexpected case.

### signature-replay
TOOLS: unit test, call same signed payload twice
TEST TYPE: unit
PROVE THIS: First call succeeds. Second call with identical calldata also succeeds when it should revert. No nonce or nonce not incremented.
PATTERN:
  - Construct valid signed message
  - target.executeWithSig(payload, sig) — first call, should succeed
  - target.executeWithSig(payload, sig) — second call, should revert but doesn't
  - assertEq(count, 2) — proves double execution

### missing-nonce
TOOLS: same as signature-replay
TEST TYPE: unit

### signature-malleability
TOOLS: unit test with both (v,r,s) and (v',r,s') variants
TEST TYPE: unit
PROVE THIS: Compute the malleable signature variant. Assert both pass ecrecover and return the same address. Assert both are accepted by the target.

### missing-chainid
TOOLS: vm.chainId(differentChainId) + replay
TEST TYPE: unit
PROVE THIS: Sign message on chainId 1. vm.chainId(137). Replay the same signature. Assert it is accepted on the different chain.

### cross-chain-replay (without bridge)
TOOLS: vm.chainId() + same signature
TEST TYPE: unit
PROVE THIS: Same as missing-chainid.

### erc20-return-value
TOOLS: MockNoReturnERC20 that mimics USDT (returns void)
TEST TYPE: unit
PROVE THIS: Deploy MockNoReturnERC20. Call target function that uses require(token.transfer(...)). Assert the function reverts even when the transfer actually succeeded, OR that it silently ignores a failed transfer.
MOCK NEEDED: MockNoReturnToken — transfer() and transferFrom() return nothing (void)

### fee-on-transfer
TOOLS: MockFeeOnTransferToken
TEST TYPE: unit
PROVE THIS: Deploy MockFeeOnTransferToken that charges 1% on transfer. Call target.deposit(100). Assert target recorded 100 but only received 99. Difference = accounting error.
MOCK NEEDED: MockFeeOnTransferToken — transferFrom deducts fee before sending

### zero-amount-transfer
TOOLS: MockRevertOnZeroToken + unit test
TEST TYPE: unit
PROVE THIS: Deploy MockRevertOnZeroToken that reverts on amount=0. Set up state where amount rounds to zero (e.g. tiny deposit with fee). Assert the distribution function reverts, blocking all users.

### storage-collision-proxy
TOOLS: vm.store + vm.load + direct slot inspection
TEST TYPE: unit
PROVE THIS: Read slot 0 of proxy. Read slot 0 of implementation (via delegatecall context). Assert they overlap. vm.store(proxyAddress, bytes32(0), bytes32(maliciousValue)). Assert implementation reads the corrupted value through delegatecall.

### storage-layout-shift
TOOLS: vm.load on specific slots before and after
TEST TYPE: unit
PROVE THIS: Deploy V1. Record values at key slots with vm.load. Deploy V2 with shifted layout. Assert slot values are misread under V2's layout.

### time-manipulation-timelock
TOOLS: vm.warp
TEST TYPE: unit
PROVE THIS: vm.warp(block.timestamp + timelockPeriod - 1) — assert action fails. vm.warp(block.timestamp + timelockPeriod + 1) — assert action succeeds. Proves the timelock boundary.

### time-manipulation-deadline
TOOLS: vm.warp
TEST TYPE: unit
PROVE THIS: vm.warp(deadline + 1). Assert the expired deadline is not enforced (if the bug is missing deadline check) or correctly enforced (if testing the fix).

### time-manipulation-vesting
TOOLS: vm.warp
TEST TYPE: unit
PROVE THIS: Warp to various points in the vesting schedule. Assert claimed amounts match or deviate from expected.

### array-delete-gap
TOOLS: unit test
TEST TYPE: unit
PROVE THIS: Add elements to array. delete array[index]. Iterate over array. Assert the zeroed slot is still present (length unchanged) and causes incorrect behavior (zero-address recipient, phantom distribution, etc.).

### duplicate-array-input
TOOLS: unit test with repeated IDs
TEST TYPE: unit
PROVE THIS: Call function with [tokenId, tokenId] (same ID twice). Assert claimed amount is 2x what it should be, or state was modified twice.

### merkle-proof-reuse
TOOLS: unit test, two different callers using same proof
TEST TYPE: unit
PROVE THIS: Alice generates valid merkle proof. vm.prank(alice); target.claim(proof, amount) — succeeds. vm.prank(bob); target.claim(proof, amount) — also succeeds when it should fail. Leaf does not bind to msg.sender.

### commit-reveal-frontrun
TOOLS: vm.prank (two different addresses) + same commitment hash
TEST TYPE: unit
PROVE THIS: Alice submits commitment hash (without msg.sender in hash). Bob copies Alice's hash. Bob submits reveal before Alice. Assert Bob's reveal is accepted and Alice's is blocked/replayed.

### erc4626-rounding
TOOLS: unit test with specific deposit/withdraw sequence
TEST TYPE: unit
PROVE THIS: deposit(a) then redeem(shares) returns more than a. Or: previewDeposit returns more than deposit actually mints. Assert the round-trip profit.

### erc4626-inflation-attack
TOOLS: unit test, two depositors
TEST TYPE: unit
PROVE THIS: Attacker deposits 1 wei (gets 1 share). Attacker donates large amount directly to vault (token.transfer to vault, bypassing deposit). Victim deposits large amount. Assert victim receives 0 shares due to rounding. Assert attacker can redeem for victim's funds.

### missing-slippage
TOOLS: unit test with minAmountOut = 0
TEST TYPE: unit
PROVE THIS: Call swap/deposit with minAmountOut = 0 or no slippage parameter. Manipulate the pool state before the call (vm.store or mock). Assert the user receives far less than expected with no revert.

### self-liquidation
TOOLS: vm.prank (borrower address calling liquidate on themselves)
TEST TYPE: unit
PROVE THIS: Borrower takes position. Position becomes undercollateralized. vm.prank(borrower); target.liquidate(borrower, ...) — assert it succeeds. Assert borrower collects the liquidation bonus on their own position.

### repeated-liquidation
TOOLS: unit test, call liquidate twice on same position
TEST TYPE: unit
PROVE THIS: First liquidation zeroes the position. Second liquidation call on the same position succeeds again (no status check). Assert state inconsistency or double bonus.

### reward-double-claim
TOOLS: unit test, call claimRewards twice
TEST TYPE: unit
PROVE THIS: After first claim, pendingReward not zeroed. Second claim pays same amount. Assert total received = 2x expected.

### reward-stale-debt
TOOLS: unit test, deposit then check earned before rewardDebt is set
TEST TYPE: unit
PROVE THIS: New depositor earns rewards for periods before their deposit. Assert earned() returns non-zero immediately after deposit with no time passage.

### phantom-rewards
TOOLS: unit test
TEST TYPE: unit
PROVE THIS: Position is liquidated but reward accounting not cleared. Assert subsequent reward calculation still accrues to the liquidated position.

### uups-missing-auth
TOOLS: vm.prank(address(0xdead)) + call upgradeTo
TEST TYPE: unit
PROVE THIS: vm.prank(nonOwner); target.upgradeTo(maliciousImpl) — assert it SUCCEEDS. Proves _authorizeUpgrade has no access control.

### proxy-initialization-frontrun
TOOLS: vm.prank(attacker) + call initialize before legitimate owner
TEST TYPE: unit
PROVE THIS: Deploy proxy without init calldata. vm.prank(attacker); proxy.initialize(attacker) — assert it succeeds. Attacker is now owner.

### create2-zero-check
TOOLS: unit test
TEST TYPE: unit
PROVE THIS: Set up conditions where CREATE2 deployment fails (e.g. insufficient balance, salt collision). Call the function. Assert the returned address is used despite being address(0). Assert subsequent operations on address(0) silently succeed or cause wrong behavior.

### ecrecover-zero-check
TOOLS: unit test with malformed signature
TEST TYPE: unit
PROVE THIS: Pass a signature of all zeros or invalid (v,r,s). ecrecover returns address(0). Assert the function accepts address(0) as a valid signer (because permissions[address(0)] is set or the check is missing).

### arbitrary-delegatecall
TOOLS: vm.prank + crafted calldata targeting an ERC20 the contract holds approvals for
TEST TYPE: unit
PROVE THIS: Call the function with user-supplied target = ERC20 address, data = transferFrom(victim, attacker, balance). Assert funds are stolen via the delegatecall.

### arbitrary-external-call
TOOLS: vm.prank + crafted target and calldata
TEST TYPE: unit
PROVE THIS: Call the function with target = token contract, data = transfer(attacker, contractBalance). Assert attacker receives funds held by the calling contract.

### array-out-of-bounds
TOOLS: unit test at arr.length or arr.length+1
TEST TYPE: unit
PROVE THIS: Call function that iterates with <= instead of <. Assert OOB access reverts unexpectedly (or in <0.8 produces wrong value).

### missing-zero-address-check
TOOLS: vm.prank + pass address(0) as parameter
TEST TYPE: unit
PROVE THIS: Call privileged setter with address(0). Assert it succeeds and address(0) is now owner/admin. Subsequent operations are broken.

### emit-missing
TOOLS: vm.recordLogs + unit test
TEST TYPE: unit
PROVE THIS: Call function that should emit event. Use vm.recordLogs() before call. After call, read Vm.Log[] entries. Assert the expected event was not emitted.
NOTE: This proves the event is missing, not a security bug per se. Only test if pashov rates this as a finding.

### abi-encodePacked-collision
TOOLS: unit test with two different inputs that produce same hash
TEST TYPE: unit
PROVE THIS: Construct two different (a,b) pairs where abi.encodePacked(a,b) == abi.encodePacked(a',b'). Assert both are accepted as valid (same hash key). Prove the collision enables permission bypass or double-spend.

---

## CATEGORY: MOCK

### flash-loan-callback-validation
TOOLS: MockFlashLender + vm.prank
TEST TYPE: unit
PROVE THIS (two sub-cases per vector 65):
  Sub-case A: Call onFlashLoan directly (not through lender). vm.prank(attacker); target.onFlashLoan(attacker, token, amount, fee, data) — assert it succeeds without a real flash loan.
  Sub-case B: Deploy MockFlashLender. Initiate a real flash loan but with crafted initiator/token/amount parameters. Assert the callback accepts them.
MOCK NEEDED:
  contract MockFlashLender {
    function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data) external returns (bool) {
      // transfer tokens to receiver
      // call receiver.onFlashLoan(address(this), token, amount, fee, data)
      // verify repayment
    }
  }

### flash-loan-governance
TOOLS: MockFlashMintToken + vm.prank + vm.roll
TEST TYPE: unit
PROVE THIS: Deploy MockFlashMintToken that mints on flashLoan and burns after. Borrow massive amount. Vote in governance using borrowed balance. Assert vote counted. Repay. Proposal passes.
MOCK NEEDED: MockFlashMintToken that implements ERC3156 and returns balance during callback

### bridge-message-spoofing
TOOLS: Call receive function directly bypassing bridge
TEST TYPE: unit
PROVE THIS: Call target.lzReceive(srcChainId, fakeSource, nonce, maliciousPayload) directly from attacker address. Assert it succeeds — proves missing endpoint validation.
No mock needed — call the function directly with vm.prank(attacker).

### lzcompose-spoofing
TOOLS: Direct call to lzCompose
TEST TYPE: unit
PROVE THIS (vector 5, two sub-cases):
  Sub-case A: vm.prank(attacker); target.lzCompose(attacker, guid, message, executor, options) — assert succeeds (msg.sender != endpoint not checked)
  Sub-case B: vm.prank(address(endpoint)); target.lzCompose(fakeSrcOApp, guid, message, executor, options) — assert succeeds (from address not validated against trusted OApp)

### cross-chain-replay-bridge
TOOLS: MockBridge that processes same message twice
TEST TYPE: unit
PROVE THIS (vector 48, three sub-cases):
  Sub-case A: Call processMessage twice with same hash. Assert second succeeds (no replay tracking).
  Sub-case B: vm.chainId(differentChain). Call processMessage. Assert it succeeds on wrong chain (no destChainId check).
  Sub-case C: Craft two messages from different source chains that hash to same value. Assert both process (source chain not in hash).

### erc777-hook-reentrancy
TOOLS: MockERC777
TEST TYPE: unit
PROVE THIS: Deploy MockERC777 where tokensToSend calls back into target. Deposit MockERC777. Trigger withdrawal. During tokensToSend hook, re-enter and call deposit again. Assert double-credit.
MOCK NEEDED:
  contract MockERC777 is IERC20 {
    ITargetContract target;
    function transfer(address to, uint256 amount) external returns (bool) {
      // call target.deposit() again before completing transfer
      target.deposit(amount);
      // then complete transfer
      return true;
    }
  }

### non-standard-erc20-void-return
TOOLS: MockNoReturnToken
TEST TYPE: unit
PROVE THIS: Deploy MockNoReturnToken (transfer returns void like USDT). Call target function that uses require(token.transfer(...)). Assert function reverts even though transfer succeeded. Proves USDT-style tokens break the protocol.
MOCK NEEDED:
  contract MockNoReturnToken {
    mapping(address => uint256) public balanceOf;
    function transfer(address to, uint256 amount) external { // no return value
      balanceOf[msg.sender] -= amount;
      balanceOf[to] += amount;
    }
    function transferFrom(address from, address to, uint256 amount) external { // no return value
      balanceOf[from] -= amount;
      balanceOf[to] += amount;
    }
  }

### non-standard-erc20-usdt-approve
TOOLS: MockUSDTToken
TEST TYPE: unit
PROVE THIS: Deploy MockUSDTToken that reverts on approve when allowance != 0. Target calls token.approve(spender, amount) without first calling approve(spender, 0). Assert it reverts.
MOCK NEEDED: MockUSDTToken where approve reverts if current allowance != 0 and new amount != 0

### oracle-mock-staleness
TOOLS: MockChainlinkAggregator
TEST TYPE: unit
PROVE THIS: Deploy MockChainlinkAggregator that returns updatedAt = block.timestamp - MAX_STALENESS - 1 and a valid price. Call target function that uses the oracle. Assert it uses the stale price without reverting (missing staleness check).
MOCK NEEDED:
  contract MockChainlinkAggregator {
    int256 public price;
    uint256 public updatedAt;
    function latestRoundData() external view returns (uint80, int256, uint256, uint256, uint80) {
      return (1, price, 0, updatedAt, 1);
    }
  }

### oracle-mock-negative-price
TOOLS: MockChainlinkAggregator returning answer = -1
TEST TYPE: unit
PROVE THIS: Set MockChainlinkAggregator.price = -1. Call target. Assert negative price is used without check, causing wrong calculation.

---

## CATEGORY: FORK

### chainlink-staleness-live
TOOLS: vm.createSelectFork + vm.warp + real Chainlink aggregator address
TEST TYPE: unit
PROVE THIS: Fork mainnet. Get the real aggregator for the asset. vm.warp(block.timestamp + MAX_STALENESS + 1). Call target function that reads the oracle. Assert stale data is accepted (or assert it reverts to confirm the fix).
REQUIRES: eth_rpc_url in foundry.toml

### spot-price-oracle-manipulation
TOOLS: vm.createSelectFork + real Uniswap pool + flash loan
TEST TYPE: unit
PROVE THIS: Fork mainnet at specific block. Manipulate the real pool reserve ratio using deal() to simulate a large swap. Assert target's price calculation is affected. Proves live price oracle is exploitable.
REQUIRES: eth_rpc_url + fork_block_number

### rebasing-token-accounting
TOOLS: vm.createSelectFork + real stETH address + rebase simulation
TEST TYPE: unit
PROVE THIS: Fork mainnet. Get actual stETH balance of target. Simulate a rebase by calling vm.store on stETH's balance mapping. Assert target's cached balance diverges from actual balanceOf.
REQUIRES: eth_rpc_url

### lst-depeg
TOOLS: vm.createSelectFork + vm.mockCall on price feed
TEST TYPE: unit
PROVE THIS: Fork mainnet. vm.mockCall on the stETH/ETH price feed to return 0.90 (depeg). Assert target uses 1:1 pricing instead. Assert the borrowing/collateral calculation is wrong.
REQUIRES: eth_rpc_url

---

## CATEGORY: INVARIANT

### solvency-invariant
TOOLS: invariant_ test + handler pattern + ghost variables
TEST TYPE: invariant
PROVE THIS: Define invariant: address(vault).balance >= vault.totalDeposits(). Run random sequences of deposit/withdraw. Assert the invariant is violated.
USE WHEN: pashov's finding involves "vault insolvency", "rounding dust", "accumulated error"

### supply-conservation-invariant
TOOLS: invariant_ test + handler
TEST TYPE: invariant
PROVE THIS: Define invariant: token.totalSupply() == sum of all balances. Run mint/burn sequences. Assert conservation is violated.
USE WHEN: "cross-chain supply accounting" (vector 150), "totalSupply inflation"

### access-control-invariant
TOOLS: invariant_ test + handler with multiple actors
TEST TYPE: invariant
PROVE THIS: Define invariant: only owner can call privilegedFunction. Run random call sequences from non-owner actors. Assert the invariant is violated (non-owner succeeds).
USE WHEN: access control bug that might only manifest after specific state transitions

### withdrawal-solvency-invariant
TOOLS: invariant_ test + handler
TEST TYPE: invariant
PROVE THIS: Define invariant: every depositor can always withdraw their full deposit. Run deposit/withdraw sequences. Assert some depositor cannot withdraw.
USE WHEN: "vault insolvency via rounding" (vector 85), "share redemption asymmetry" (vector 173)

---

## BUG CLASSES NOT IN THIS MAP

If the pashov finding has a bug_class not listed above:

1. Read the proof field carefully — it contains concrete values and a trace
2. Read the path field — it gives the exact call sequence
3. Map to the closest category using this logic:
   - Does it require impersonation? → STANDARD, use vm.prank
   - Does it require time travel? → STANDARD, use vm.warp/vm.roll
   - Does it require a fake external contract? → MOCK
   - Does it require real deployed protocol state? → FORK
   - Does it require proving a property holds across many call sequences? → INVARIANT, use invariant_ test
   - Does it require observing mempool or off-chain actors? → UNTESTABLE
4. If still unclear after reading proof and path: classify as INCONCLUSIVE with reason "bug_class not mapped, proof insufficient for test construction"
