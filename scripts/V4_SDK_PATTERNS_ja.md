# Uniswap V4 SDK 使用パターン

このドキュメントは、UIアプリケーションから抽出されたUniswap V4 SDKの主要なパターンと使用例をまとめたものです。

## 基本概念

### 1. プールキー構造

V4は一意のキー構造を使用してプールを識別します：

```typescript
interface PoolKey {
  currency0: string;  // 小さいトークンアドレス
  currency1: string;  // 大きいトークンアドレス
  fee: number;        // 手数料ティア（例：3000 = 0.3%）
  tickSpacing: number; // プールのティック間隔
  hooks: string;      // フックコントラクトアドレス
}
```

### 2. V4Planner - トランザクションの構築

V4Plannerはスワップトランザクションを構築する主要な方法です：

```typescript
import { V4Planner, Actions } from '@uniswap/v4-sdk';

// プランナーインスタンスを作成
const planner = new V4Planner();

// スワップトランザクションを構築
planner.addAction(Actions.SWAP_EXACT_IN_SINGLE, [
  poolKey,          // PoolKey構造体
  true,             // zeroForOne（方向）
  amountIn,         // 入力量
  minAmountOut,     // 最小出力量
  MAX_UINT256,      // 期限
  recipient,        // 受取人アドレス
  hookData || '0x', // オプションのフックデータ
]);

// 入力トークンを決済
planner.addAction(Actions.SETTLE_ALL, [
  inputToken,
  maxAmount,
  false  // コントラクトからの支払いではない
]);

// 出力トークンを受け取る
planner.addAction(Actions.TAKE_ALL, [
  outputToken,
  minAmount
]);

// 最終化してエンコードされたデータを取得
const { actions, params } = planner.finalize();
```

### 3. Universal Routerとの統合

すべてのV4スワップはUniversalRouterを通じて実行されます：

```typescript
const universalRouterInterface = new ethers.Interface([
  'function execute(bytes calldata commands, bytes[] calldata inputs, uint256 deadline) external payable'
]);

const swapTx = await universalRouter.execute(
  '0x0b',           // V4_SWAPコマンド
  [encodedParams],  // V4Plannerの出力
  deadline,
  { value: ethValue }
);
```

### 4. ポジション管理

V4PositionManagerを使用した流動性の追加：

```typescript
const calls = [
  encodeFunctionData({
    abi: V4PositionManagerABI,
    functionName: 'modifyLiquidities',
    args: [encodedParams, deadline]
  })
];

// ネイティブETHの処理
if (isNativeToken) {
  calls.push(
    encodeFunctionData({
      abi: V4PositionManagerABI,
      functionName: 'sweep',
      args: [NATIVE_TOKEN_ADDRESS, recipient]
    })
  );
}

const multicallTx = await positionManager.multicall(calls);
```

### 5. プール状態の読み取り

StateViewコントラクトを通じてプール状態を読み取り：

```typescript
const stateViewInterface = new ethers.Interface([
  'function getSlot0(PoolKey memory poolKey) external view returns (uint160 sqrtPriceX96, int24 tick, uint32 protocolFee, uint32 lpFee)',
  'function getLiquidity(PoolKey memory poolKey) external view returns (uint128)',
  'function getPoolPrice(PoolKey memory poolKey, uint256 amountIn, bool zeroForOne) external view returns (uint256)'
]);

const slot0 = await stateView.getSlot0(poolKey);
const liquidity = await stateView.getLiquidity(poolKey);
```

### 6. Permit2との統合

Permit2を使用したバッチトークン承認：

```typescript
const permitData = {
  details: [
    {
      token: token0,
      amount: amount0,
      expiration: MAX_UINT48,
      nonce: nonce0
    },
    {
      token: token1,
      amount: amount1,
      expiration: MAX_UINT48,
      nonce: nonce1
    }
  ],
  spender: positionManager.address,
  sigDeadline: deadline
};

const signature = await signer._signTypedData(
  permitDomain,
  permitTypes,
  permitData
);
```

## コントラクトアドレス

### メインネット（Unichain）
- PoolManager: `0x1F98400000000000000000000000000000000004`
- Universal Router: `0x5C60712b43ddC773d82F5AFD59e88C9eDc7AE926`
- V4PositionManager: `0x5C60712b43ddC773d82F5AFD59e88C9eDc7AE926`
- StateView: `0x16F98400000000000000000000000000000000fC`
- Permit2: `0x000000000022D473030F116dDEE9F6B43aC78BA3`

### テストネット（Unichain Sepolia）
- PoolManager: `0x2000d755f9e4F3c77E0C9dfb6f84a609E2A0f0fd`
- Universal Router: `0x97C5c088644fA3F576D7993d8e3516cFA7361f3a`
- V4PositionManager: `0xEcFe3cC2c893df3faa1c06a0a0612cb8a0Aba0b2`
- StateView: `0x4e5F2b7B97e0667AD969dF4CDB93d64691ad8c23`
- Permit2: `0x000000000022D473030F116dDEE9F6B43aC78BA3`

## 共通ユーティリティ

### ネイティブトークンの処理
```typescript
const NATIVE_TOKEN_ADDRESS = '0x0000000000000000000000000000000000000000';

function isNativeToken(address: string): boolean {
  return address === NATIVE_TOKEN_ADDRESS || 
         address === '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';
}
```

### トークンのソート
```typescript
function sortTokens(tokenA: string, tokenB: string): [string, string] {
  return BigInt(tokenA) < BigInt(tokenB) ? [tokenA, tokenB] : [tokenB, tokenA];
}
```

### プールキーの検証
```typescript
function validatePoolKey(key: PoolKey): boolean {
  return BigInt(key.currency0) < BigInt(key.currency1);
}
```

## フック統合

フックはプールの動作を変更し、プールキーで指定されます：

```typescript
// フック付きプールの作成
const poolKey = {
  currency0: token0,
  currency1: token1,
  fee: 3000,
  tickSpacing: 60,
  hooks: hookAddress  // デプロイしたフックコントラクト
};

// フックデータをスワップで渡すことができます
const hookData = ethers.AbiCoder.defaultAbiCoder().encode(
  ['uint256'], 
  [customValue]
);
```

## エラーハンドリング

処理すべき一般的なV4エラー：
- `Pool not initialized` - プールを最初に作成する必要があります
- `Invalid pool key` - トークンが正しくソートされていません
- `Hook validation failed` - フック権限がデプロイメントと一致しません
- `Insufficient liquidity` - スワップに十分な流動性がありません

## テストパターン

### フォークテスト
```typescript
// テスト用にメインネットフォークを使用
const provider = new ethers.JsonRpcProvider('http://localhost:8545');
```

### テストトークンのデプロイ
```typescript
const testTokenFactory = new ethers.ContractFactory(
  TestERC20ABI,
  TestERC20Bytecode,
  signer
);
const token = await testTokenFactory.deploy('Test', 'TEST', 18);
```

## V3からの移行

V3との主な違い：
- プールアドレスの代わりにプールキー
- 個別のプールの代わりにシングルトンPoolManager
- カスタムロジック用のフック統合
- ガス効率のための一時的ストレージ
- 直接スワップ仕様（ルーティングなし）