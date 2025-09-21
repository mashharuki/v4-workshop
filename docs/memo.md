# トライしてみた記録

## Hooksのデプロイ

```bash
forge script script/01_DeployAndSave.s.sol \
    --rpc-url $UNICHAIN_RPC \
    --private-key $PK \
    --broadcast \
    --legacy \
    --ffi \
    -vvv
```

```bash
[⠊] Compiling...
[⠑] Compiling 71 files with Solc 0.8.26
[⠘] Solc 0.8.26 finished in 1.63s
Compiler run successful with warnings:
Warning (2394): Transient storage as defined by EIP-1153 can break the composability of smart contracts: Since transient storage is cleared only at the end of the transaction and not at the end of the outermost call frame to the contract within a transaction, your contract may unintentionally misbehave when invoked multiple times in a complex transaction. To avoid this, be sure to clear all transient storage at the end of any call to your contract. The use of transient storage for reentrancy guards that are cleared at the end of the call is safe.
   --> lib/openzeppelin-contracts/contracts/utils/TransientSlot.sol:108:13:
    |
108 |             tstore(slot, value)
    |             ^^^^^^

Script ran successfully.

== Return ==
result: struct DeployHooksWithMiner.DeploymentResult DeploymentResult({ liquidityPenaltyHook: 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503, antiSandwichHook: 0x0000000000000000000000000000000000000000, limitOrderHook: 0x0000000000000000000000000000000000000000, liquidityPenaltySalt: 0x0000000000000000000000000000000000000000000000000000000000000000, antiSandwichSalt: 0x0000000000000000000000000000000000000000000000000000000000000000, limitOrderSalt: 0x0000000000000000000000000000000000000000000000000000000000000000 })

== Logs ==
  LiquidityPenaltyHook deployed with salt: 17055
  ===== Hook Deployment Results =====
  LiquidityPenaltyHook: 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503
  ===================================
  
========== DEPLOYMENT COMPLETE ==========
  Please update deployments/unichain-sepolia.json with:
  
  LiquidityPenaltyHook:
    address: 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503
    salt: 0x0000000000000000000000000000000000000000000000000000000000000000
  
  AntiSandwichHookImpl:
    address: 0x0000000000000000000000000000000000000000
    salt: 0x0000000000000000000000000000000000000000000000000000000000000000
  
  LimitOrderHook:
    address: 0x0000000000000000000000000000000000000000
    salt: 0x0000000000000000000000000000000000000000000000000000000000000000
  =========================================

  
Deployment JSON:
  {
  "network": "unichain-sepolia",
  "chainId": 1301,
  "poolManager": "0x2000d755f9e4F3c77E0C9dfb6f84a609E2A0f0fd",
  "hooks": {
    "LiquidityPenaltyHook": {
      "address": "0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503",
      "deploymentBlock": 31575463,
      "salt": "0x0000000000000000000000000000000000000000000000000000000000000000"
    },
    "AntiSandwichHookImpl": {
      "address": "0x0000000000000000000000000000000000000000",
      "deploymentBlock": 31575463,
      "salt": "0x0000000000000000000000000000000000000000000000000000000000000000"
    },
    "LimitOrderHook": {
      "address": "0x0000000000000000000000000000000000000000",
      "deploymentBlock": 31575463,
      "salt": "0x0000000000000000000000000000000000000000000000000000000000000000"
    }
  },
  "deployedAt": "1758427891"
}
  
Deployment addresses saved to script/.deployment.env

## Setting up 1 EVM.

==========================

Chain 1301

Estimated gas price: 0.001000253 gwei

Estimated total gas used for script: 2605088

Estimated amount required: 0.000002605747087264 ETH

==========================

##### unichain-sepolia
✅  [Success] Hash: 0xc3c022f1fe2e6fb3c624c88a851340e902a8ac6af7b2145123ab65af8a474139
Block: 31575476
Paid: 0.000001886514167361 ETH (1886037 gas * 0.001000253 gwei)

✅ Sequence #1 on unichain-sepolia | Total Paid: 0.000001886514167361 ETH (1886037 gas * avg 0.001000253 gwei)
                                                                                                                                     

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.

Transactions saved to: /workspaces/v4-workshop/contracts/broadcast/01_DeployAndSave.s.sol/1301/run-latest.json

Sensitive values saved to: /workspaces/v4-workshop/contracts/cache/01_DeployAndSave.s.sol/1301/run-latest.json
```

デプロイされたアドレスを確認

```bash
cat script/.deployment.env
```

```bash
LIQUIDITY_PENALTY_HOOK=0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503
ANTI_SANDWICH_HOOK=0x0000000000000000000000000000000000000000
LIMIT_ORDER_HOOK=0x0000000000000000000000000000000000000000
```

コントラクトコードを確認する

```bash
cast code $(cat script/.deployment.env | grep LIQUIDITY_PENALTY_HOOK | cut -d'=' -f2) --rpc-url $UNICHAIN_RPC | wc -c
16931
```

foundryでテストを実行(Unichain Sepoliaからフォークしてテスト)

```bash
forge test --fork-url $UNICHAIN_RPC -vvv
```

```bash
[⠊] Compiling...
[⠆] Compiling 40 files with Solc 0.8.26
[⠔] Solc 0.8.26 finished in 3.28s
Compiler run successful with warnings:
Warning (2394): Transient storage as defined by EIP-1153 can break the composability of smart contracts: Since transient storage is cleared only at the end of the transaction and not at the end of the outermost call frame to the contract within a transaction, your contract may unintentionally misbehave when invoked multiple times in a complex transaction. To avoid this, be sure to clear all transient storage at the end of any call to your contract. The use of transient storage for reentrancy guards that are cleared at the end of the call is safe.
  --> lib/v4-core/src/libraries/CurrencyReserves.sol:23:13:
   |
23 |             tstore(CURRENCY_SLOT, 0)
   |             ^^^^^^

Warning (2072): Unused local variable.
  --> tests/LiquidityPenaltyHook.t.sol:68:9:
   |
68 |         uint256 gasUsed = gasStart - gasleft();
   |         ^^^^^^^^^^^^^^^

Warning (2072): Unused local variable.
  --> tests/LiquidityPenaltyHook.t.sol:89:9:
   |
89 |         uint256 gasUsed = gasStart - gasleft();
   |         ^^^^^^^^^^^^^^^

Warning (2072): Unused local variable.
   --> tests/LiquidityPenaltyHook.t.sol:126:9:
    |
126 |         uint256 gasUsed = gasStart - gasleft();
    |         ^^^^^^^^^^^^^^^

Warning (2018): Function state mutability can be restricted to view
  --> tests/LiquidityPenaltyHook.fork.t.sol:12:5:
   |
12 |     function setUp() public {
   |     ^ (Relevant source part starts here and spans across multiple lines).

Warning (2018): Function state mutability can be restricted to pure
   --> tests/utils/HookTestBase.sol:169:5:
    |
169 |     function logBalanceDelta(string memory label, BalanceDelta delta) internal view {
    |     ^ (Relevant source part starts here and spans across multiple lines).

Warning (3860): Contract initcode size is 124352 bytes and exceeds 49152 bytes (a limit introduced in Shanghai). This contract may not be deployable on Mainnet. Consider enabling the optimizer (with a low "runs" value!), turning off revert strings, or using libraries.
  --> tests/LiquidityPenaltyHook.t.sol:18:1:
   |
18 | contract LiquidityPenaltyHookTest is HookTestBase, BalanceDeltaAssertions {
   | ^ (Relevant source part starts here and spans across multiple lines).


Ran 4 tests for tests/LiquidityPenaltyHook.fork.t.sol:LiquidityPenaltyHookForkTest
[PASS] test_Fork_EdgeCases() (gas: 32049)
Logs:
  Skipping fork test - set FORK=true to run
  Forked Unichain mainnet at block: 31575641
  Using PoolManager at: 0x1F98400000000000000000000000000000000004
  Chain ID: 1301

[PASS] test_Fork_GasCosts() (gas: 32005)
Logs:
  Skipping fork test - set FORK=true to run
  Forked Unichain mainnet at block: 31575641
  Using PoolManager at: 0x1F98400000000000000000000000000000000004
  Chain ID: 1301

[PASS] test_Fork_JITAttackPrevention() (gas: 32028)
Logs:
  Skipping fork test - set FORK=true to run
  Forked Unichain mainnet at block: 31575641
  Using PoolManager at: 0x1F98400000000000000000000000000000000004
  Chain ID: 1301

[PASS] test_Fork_LegitimateLPNoPenalty() (gas: 32006)
Logs:
  Skipping fork test - set FORK=true to run
  Forked Unichain mainnet at block: 31575641
  Using PoolManager at: 0x1F98400000000000000000000000000000000004
  Chain ID: 1301

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 5.38s (6.25s CPU time)

Ran 5 tests for tests/LiquidityPenaltyHook.t.sol:LiquidityPenaltyHookTest
[PASS] test_AddLiquidity() (gas: 235435)
Logs:
  
=== Testing Add Liquidity ===
  Add liquidity delta
    Amount0: -2995354955910781
    Amount1: -2995354955910781
  Add liquidity gas used: 230875

[PASS] test_MultipleLiquidityOperationsSameBlock() (gas: 389490)
Logs:
  
=== Testing Multiple Liquidity Operations Same Block ===
  First add
    Amount0: -2995354955911
    Amount1: -2995354955911
  Second add
    Amount0: -5981737760510
    Amount1: 0
  First remove (with penalty)
    Amount0: 2995354955910
    Amount1: 2995354955910
  Second remove (with penalty)
    Amount0: 5981737760509
    Amount1: 0

[PASS] test_PenaltyBasisPointsConfiguration() (gas: 130175270)
Logs:
  
=== Testing Penalty Configuration ===
  
Testing penalty rate: 5 basis points
    Penalty0: 1
    Penalty1: 1
  
Testing penalty rate: 10 basis points
    Penalty0: 1
    Penalty1: 1
  
Testing penalty rate: 50 basis points
    Penalty0: 1
    Penalty1: 1
  
[OK] Penalty rates configuration works correctly

[PASS] test_RemoveLiquidityNoPenalty() (gas: 239866)
Logs:
  
=== Testing Remove Liquidity No Penalty ===
  Add liquidity delta
    Amount0: -2995354955910781
    Amount1: -2995354955910781
  Advanced to block 31575636
  Remove liquidity delta
    Amount0: 2995354955910780
    Amount1: 2995354955910780
  Remove liquidity no penalty gas used: 62913
  Net amounts without penalty:
    Net amount0: -1
    Net amount1: -1

[PASS] test_RemoveLiquidityWithPenalty() (gas: 236316)
Logs:
  
=== Testing Remove Liquidity With Penalty ===
  Add liquidity delta
    Amount0: -2995354955910781
    Amount1: -2995354955910781
  Remove liquidity delta
    Amount0: 2995354955910780
    Amount1: 2995354955910780
  Remove liquidity with penalty gas used: 62915
  Net amounts after penalty:
    Net amount0: -1 (should be negative due to penalty)
    Net amount1: -1 (should be negative due to penalty)

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 9.91s (1.37s CPU time)

Ran 2 test suites in 11.23s (15.29s CPU time): 9 tests passed, 0 failed, 0 skipped (9 total tests)
```

続いてプールを作成する

```bash
# ETH/USDCプール（0.3%手数料）を作成
forge script script/02_CreatePool.s.sol \
    --rpc-url $UNICHAIN_RPC \
    --private-key $PK \
    --broadcast \
    --legacy \
    -vvv
```

```bash
[⠊] Compiling...
No files changed, compilation skipped
Traces:
  [2452535] → new CreatePool@0x5b73C5498c1E3b4dbA84de0F1833c4a029d90519
    └─ ← [Return] 12139 bytes of code

  [28280] CreatePool::run()
    ├─ [0] VM::envUint("PK") [staticcall]
    │   └─ ← [Return] <env var value>
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] 0x51908F598A5e0d8F1A3bAbFa6DF76F9704daD072
    ├─ [0] console::log("==== Creating Uniswap V4 Pool ====") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Deployer:", 0x51908F598A5e0d8F1A3bAbFa6DF76F9704daD072) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Network: Unichain") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("\n== V4 Contract Addresses ==") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("PoolManager:", 0x1F98400000000000000000000000000000000004) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("StateView:", 0x86e8631A016F9068C3f085fAF484Ee3F5fDee8f2) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::readFile("./script/.deployment.env") [staticcall]
    │   └─ ← [Return] <file>
    ├─ [0] VM::split("LIQUIDITY_PENALTY_HOOK=0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503\nANTI_SANDWICH_HOOK=0x0000000000000000000000000000000000000000\nLIMIT_ORDER_HOOK=0x0000000000000000000000000000000000000000\n", "\n") [staticcall]
    │   └─ ← [Return] ["LIQUIDITY_PENALTY_HOOK=0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503", "ANTI_SANDWICH_HOOK=0x0000000000000000000000000000000000000000", "LIMIT_ORDER_HOOK=0x0000000000000000000000000000000000000000", ""]
    ├─ [0] VM::split("LIQUIDITY_PENALTY_HOOK=0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503", "=") [staticcall]
    │   └─ ← [Return] ["LIQUIDITY_PENALTY_HOOK", "0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503"]
    ├─ [0] VM::parseAddress("0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503") [staticcall]
    │   └─ ← [Return] 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503
    ├─ [0] console::log("Loaded LiquidityPenaltyHook from deployment:", 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("\n== Pool Configuration ==") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Currency0:", 0x0000000000000000000000000000000000000000) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Currency1:", 0x078D782b760474a361dDA0AF3839290b0EF57AD6) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Pair: ETH/USDC") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Fee:", 3000) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Tick Spacing:", 60) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Hook:", 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Hook Name: LiquidityPenaltyHook") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Initial Price: From V3 TWAP") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("\n== Pool ID ==") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::toString(0x05eff95fbbf872cc1577d069a9f1175907967f2a4538ea303555067b1590f723) [staticcall]
    │   └─ ← [Return] "0x05eff95fbbf872cc1577d069a9f1175907967f2a4538ea303555067b1590f723"
    ├─ [0] console::log("Pool ID:", "0x05eff95fbbf872cc1577d069a9f1175907967f2a4538ea303555067b1590f723") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("\n== Checking if pool exists ==") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] 0x86e8631A016F9068C3f085fAF484Ee3F5fDee8f2::getSlot0(0x05eff95fbbf872cc1577d069a9f1175907967f2a4538ea303555067b1590f723) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Revert] call to non-contract address 0x86e8631A016F9068C3f085fAF484Ee3F5fDee8f2



== Logs ==
  ==== Creating Uniswap V4 Pool ====
  Deployer: 0x51908F598A5e0d8F1A3bAbFa6DF76F9704daD072
  Network: Unichain
  
== V4 Contract Addresses ==
  PoolManager: 0x1F98400000000000000000000000000000000004
  StateView: 0x86e8631A016F9068C3f085fAF484Ee3F5fDee8f2
  Loaded LiquidityPenaltyHook from deployment: 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503
  
== Pool Configuration ==
  Currency0: 0x0000000000000000000000000000000000000000
  Currency1: 0x078D782b760474a361dDA0AF3839290b0EF57AD6
  Pair: ETH/USDC
  Fee: 3000
  Tick Spacing: 60
  Hook: 0x6aa4b614E4b5FE2fabC7211ee530AF6becC6C503
  Hook Name: LiquidityPenaltyHook
  Initial Price: From V3 TWAP
  
== Pool ID ==
  Pool ID: 0x05eff95fbbf872cc1577d069a9f1175907967f2a4538ea303555067b1590f723
  
== Checking if pool exists ==
Error: script failed: call to non-contract address 0x86e8631A016F9068C3f085fAF484Ee3F5fDee8f2
```

プールの状態表示

```bash
forge script script/03_ShowPoolInfo.s.sol \
    --rpc-url $UNICHAIN_RPC \
    -vvv
```