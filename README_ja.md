# Uniswap V4 ワークショップ - EDCON 2025

このリポジトリは、Uniswap V4 Hooksワークショップのためのモノレポです。スマートコントラクト、V4 SDK統合、インデクサー、ダッシュボードが含まれています。

## 🎯 ワークショップ概要

このワークショップでは、Uniswap V4の革新的な「Hooks」機能をデプロイしてテストし、Hookの完全な開発ライフサイクル（開発 → デプロイ → テスト → 分析）を体験します。

**所要時間**: 35-55分  
**対象者**: 基本的なSolidityの知識を持つ開発者

## 🎯 ワークショップの目標

このワークショップを通じて以下のことを習得できます：

1. Uniswap V4 Hooksの仕組みの理解
2. HookMinerを使用した適切なアドレスデプロイの体験
3. V4 SDKを使ったプール操作の習得
4. LiquidityPenaltyHookを用いたJIT攻撃防止の検証
5. Envioインデクサーを使った分析手法の学習

## 📅 タイムテーブル

| 時間      | セクション          | 内容                                                                    |
| --------- | ------------------- | ----------------------------------------------------------------------- |
| 0-5分     | 環境セットアップ    | リポジトリのクローン、依存関係のインストール、環境変数の設定              |
| 5-15分    | Hookデプロイ        | HookMinerを使用したLiquidityPenaltyHookのデプロイ                       |
| 15-25分   | プール作成          | デプロイしたHookを使用したプールの作成                                  |
| 25-40分   | ハンズオン操作      | V4 SDKを使用した流動性の追加・削除とスワップの実行                      |
| 40-45分   | 分析とまとめ        | インデクサーとダッシュボードを使った結果確認、質疑応答                   |

## 🏗️ システムアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ブロックチェーン (Unichain)                     │
│  ・Hooks (LiquidityPenaltyHook, etc.)                               │
│  ・PoolManager, PositionManager                                     │
│  ・Universal Router                                                 │
└─────────────────────┬──────────────┬────────────────────────────────┘
                      │              │
                      │              │
                イベント監視     トランザクション実行
                      │              │
                      ▼              ▼
┌─────────────────────────────┐  ┌───────────────────────────────────┐
│     インデクサー (Envio)     │  │      V4 SDK スクリプト             │
│  http://localhost:8080      │  │  ・04-add-liquidity.ts            │
│  ・イベント収集              │  │  ・05-swap-universal-router.ts    │
│  ・データ永続化              │  │  ・06-remove-liquidity.ts         │
│    (PostgreSQL)             │  │  ・check-pool-state.ts            │
│  ・GraphQL API               │  │                                  │
└──────────────┬──────────────┘  └─────────────────┬─────────────────┘
               │                                   │
               │ GraphQL API                       │ 操作
               │                                   │
               ▼                                   │
┌─────────────────────────────┐                    │
│      ダッシュボードアプリ    │                    │
│   http://localhost:3000     │                    │
│  ・統計表示                 │                    │
│  ・TVL分析                  │                    │
│  ・Hook動作の               │                    │
│    可視化                   │                    │
└──────────────┬──────────────┘                    │
               │                                   │
               │ 読み取り専用                      │
               ▼                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  ユーザー（ワークショップ参加者）                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 🔨 Foundryデプロイフロー

スマートコントラクトのデプロイプロセスは、Foundryのスクリプト機能を使用します：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        開発者マシン                                 │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                      Foundryスクリプト                     │     │
│  │  01_DeployAndSave.s.sol → HookMinerでフックをデプロイ       │     │
│  │  02_CreatePool.s.sol    → デプロイしたフックでプールを作成  │     │
│  │  03_ShowPoolInfo.s.sol  → デプロイ情報を表示              │     │
│  └──────────────────────┬─────────────────────────────────────┘     │
│                         │                                           │
│  ┌──────────────────────▼─────────────────────────────────────┐     │
│  │                   HookMiner                                │     │
│  │  ・権限に基づく決定論的アドレスの計算                      │     │
│  │  ・CREATE2デプロイ用のソルトを検索                         │     │
│  │  ・アドレスにフックフラグをエンコード（下位20ビット）       │     │
│  └──────────────────────┬─────────────────────────────────────┘     │
│                         │                                           │
│  ┌──────────────────────▼─────────────────────────────────────┐     │
│  │              CREATE2 Deployer                              │     │
│  │  ・計算されたアドレスでフックをデプロイ                    │     │
│  │  ・権限ビットとアドレスの一致を保証                        │     │
│  │  ・デプロイ情報を.envファイルに保存                        │     │
│  └──────────────────────┬─────────────────────────────────────┘     │
└─────────────────────────┼───────────────────────────────────────────┘
                          │ デプロイ & 初期化
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Unichainブロックチェーン                        │
│  ・決定論的アドレスにデプロイされたHooks                             │
│  ・フック統合で初期化されたプール                                    │
│  ・V4 SDK連携の準備完了                                             │
└─────────────────────────────────────────────────────────────────────┘
```

**主要なデプロイ手順:**

1. **環境セットアップ**

   ```bash
   cd contracts
   source .env
   export ETH_FROM=$(cast wallet address --private-key $PK)
   ```

2. **Hookデプロイ** (01_DeployAndSave.s.sol)

   - HookMinerを使用してアドレスを計算
   - CREATE2を介してデプロイして決定論的アドレスを生成
   - アドレスを`scripts/.deployment.env`に保存

3. **プール作成** (02_CreatePool.s.sol)

   - デプロイされたフックアドレスを読み取り
   - 適切なフック統合でプールを作成
   - プール情報を`scripts/.pool.env`に保存

4. **検証** (03_ShowPoolInfo.s.sol)
   - すべてのデプロイ情報を表示
   - フック権限とプール設定を確認

## 🪝 Uniswap V4 Hooksとは？

### V4の革新的なアーキテクチャ

Uniswap V4は「シングルトン」アーキテクチャを採用し、すべてのプールが単一のPoolManagerコントラクトによって管理されます。これにより以下が可能になります：

- **ガス効率の向上**: プール間の最適化されたマルチホップスワップ
- **カスタマイザビリティ**: Hooksを通じたプール動作の自由な拡張
- **資本効率**: フラッシュアカウンティングによる一時的な借用

### Hookの機能

Hooksは、プールのライフサイクルの特定のポイントで実行されるカスタムロジックです：

```
┌─────────────────┐
│   ユーザー操作  │
└────────┬────────┘
         │
    ┌────▼─────┐     ┌──────────────┐
    │ プール   ├────►│ フックコント │
    │ マネージャー │◄────┤ ラクト      │
    └──────────┘     │ (あなたの    │
                     │ ロジック)    │
                     └──────────────┘
```

**14の拡張ポイント**:

- `beforeInitialize` / `afterInitialize`
- `beforeAddLiquidity` / `afterAddLiquidity`
- `beforeRemoveLiquidity` / `afterRemoveLiquidity`
- `beforeSwap` / `afterSwap`
- `beforeDonate` / `afterDonate`
- デルタ戻り値フラグ（手数料とスワップ量の調整用）

## 📦 ワークショップでカバーする3つのHooks

### 1. LiquidityPenaltyHook - JIT攻撃防止

**問題**: Just-In-Time（JIT）攻撃

- 攻撃者が大きなスワップの直前に流動性を追加
- スワップ手数料を獲得した直後に流動性を削除
- 長期的な流動性提供者から収益を奪う

**解決策**: 時間ベースのペナルティ

```solidity
penalty = fees * (1 - (currentBlock - lastAddedBlock) / blockNumberOffset)
```

早期の流動性撤退にペナルティを課し、時間の経過とともに減少（10ブロック後に0%）。

### 2. AntiSandwichHook - MEV保護

ブロック内の価格操作を制限することでサンドイッチ攻撃を防止。各ブロックの開始時に価格をチェックポイントし、同一ブロック内での有利な価格での取引を防止。

### 3. LimitOrderHook - 疑似指値注文

特定の価格に到達したときに自動的に実行される、オンチェーン指値注文を可能にします。

## 🚀 クイックスタート

### 1. 環境変数の設定

```bash
# または手動で各.envファイルを作成
cp contracts/.env.example contracts/.env
cp apps/indexer/.env.example apps/indexer/.env
```

**contracts/.envの必須設定**:

- `PK`: デプロイ用の秘密鍵（絶対にコミットしないこと！）
- `ETHERSCAN_API_KEY`: コントラクト検証用

**apps/indexer/.envの必須設定**:

- `ENVIO_API_TOKEN`: Envio APIトークン（オプション）
  - ローカル開発のみの場合は不要
  - 本番環境や開発コンソール使用時に必要（https://envio.dev/console）
  - https://envio.dev/app/api-tokens でトークンを作成

### 2. 依存関係のインストール

```bash
# Bunを使用してインストール
bun install
```

gitSubmoduleのインストール

```bash
git submodule update --init --recursive
```

### 3. インデクサーのセットアップと開始（初回のみ）

```bash
# インデクサーをセットアップ
cd apps/indexer
pnpm install  # インデクサーはpnpmが必要
bun run codegen
bun run dev
```

### 4. ダッシュボードの開始

```bash
# 新しいターミナルで、ダッシュボードを開始（http://localhost:3000）
cd apps/dashboard
pnpm install
bun run dev
```

**注意**:

- インデクサーの動作にはDocker Desktopが必要です
- インデクサーの準備が完了すると、GraphQLエンドポイントがhttp://localhost:8080/v1/graphqlで利用可能になります
- 2回目以降の実行では、インデクサーディレクトリで単純に`bun run dev`を使用できます

## 📁 プロジェクト構造

```
uniswap-v4-workshop/
├── contracts/        # Uniswap V4 Hooksスマートコントラクト
│   ├── src/         # Hook実装（LiquidityPenaltyHook等）
│   └── script/      # デプロイと操作スクリプト
├── scripts/         # V4 SDK統合スクリプト
│   ├── utils/       # 共通ユーティリティ
│   ├── 04-add-liquidity.ts      # 流動性追加
│   ├── 05-swap-universal-router.ts # スワップ実行
│   └── 06-remove-liquidity.ts    # 流動性削除
├── apps/
│   ├── indexer/     # ブロックチェーンインデクサー（Envio）
│   └── dashboard/   # 分析ダッシュボード（Next.js）
├── deployments/     # ネットワーク別コントラクトアドレス
└── docs/           # ワークショップドキュメント
```

## 🛠️ 利用可能なコマンド

```bash
# コントラクト関連
bun run contracts:build       # コントラクトをビルド
bun run v4:deploy            # HookMinerでフックをデプロイ
bun run v4:pool              # デプロイしたフックでプールを作成
bun run v4:info              # プール情報を表示

# V4 SDK操作
bun run v4:check             # プール状態を確認
bun run v4:add               # 流動性を追加（Position NFTを作成）
bun run v4:swap              # Universal Router経由でスワップを実行
bun run v4:remove            # 流動性を削除

# アプリケーション起動
bun run indexer:dev          # インデクサーを開始（Dockerが必要）
bun run dashboard            # ダッシュボードを開始
```

### スクリプトの詳細

**Foundryスクリプト**:

```solidity
// v4:deploy (01_DeployAndSave.s.sol) - Hookデプロイ
- HookMinerを使用して適切なアドレスでデプロイ
- CREATE2による決定論的アドレス生成
- デプロイ結果を.deployment.envに保存

// v4:pool (02_CreatePool.s.sol) - プール作成
- デプロイしたHookを使用してプールを作成
- ETH/USDCペア、0.3%手数料設定
- プール情報を.pool.envに保存

// v4:info (03_ShowPoolInfo.s.sol) - プール情報表示
- Hookアドレスと有効な権限を表示
- プールIDと現在価格
- エクスプローラーリンクを生成
```

**V4 SDKスクリプト**:

```typescript
// v4:check (check-pool-state.ts) - プール状態確認
- 現在価格、流動性、手数料を表示
- Hookアドレスと設定を確認

// v4:add (04-add-liquidity.ts) - 流動性追加
- Position NFTを作成
- Permit2を使用したガスレス承認
- 指定された価格範囲に流動性を提供

// v4:swap (05-swap-universal-router.ts) - スワップ実行
- V4 Quoterを使用した見積もり（リバートデータ処理）
- V4Plannerでアクションを構築
- Universal Router経由で実行

// v4:remove (06-remove-liquidity.ts) - 流動性削除
- Position NFTから流動性を引き出し
- LiquidityPenaltyHookのペナルティを処理
- 部分的または完全な削除をサポート
```

## 🔧 Hook固有のデプロイプロセス

V4 Hooksの最も独特な側面は、**権限情報がアドレス自体にエンコードされている**ことです：

```
フックアドレス: 0x0050E651C7b8662f4E17C589B6387db8f7488503
                                                      ^^^^
                                        最下位ビット（右側）が権限を表す
```

例: `0x...8503`の最下位14ビット

- バイナリ: `1000 0101 0000 0011`
- これは以下の権限を示します：
  - ビット0 (0x1): AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA
  - ビット1 (0x2): AFTER_ADD_LIQUIDITY_RETURNS_DELTA
  - ビット8 (0x100): AFTER_REMOVE_LIQUIDITY
  - ビット10 (0x400): AFTER_ADD_LIQUIDITY

### HookMinerの動作原理

1. **権限宣言**:

```solidity
function getHookPermissions() public pure returns (Hooks.Permissions memory) {
    return Hooks.Permissions({
        afterAddLiquidity: true,
        afterRemoveLiquidity: true,
        afterAddLiquidityReturnDelta: true,
        afterRemoveLiquidityReturnDelta: true,
        // ... その他はすべてfalse
    });
}
```

2. **アドレス計算**:

```solidity
// HookMinerはCREATE2を使用して適切なアドレスを見つける
uint160 flags = Hooks.AFTER_ADD_LIQUIDITY_FLAG |
                Hooks.AFTER_REMOVE_LIQUIDITY_FLAG | ...;

(address hookAddress, bytes32 salt) = HookMiner.find(
    CREATE2_DEPLOYER,
    flags,
    bytecode,
    constructorArgs
);
```

3. **検証**: 権限はvalidateHookAddress()で検証されます。

## 📊 Envioを使用したインデックス作成

### イベント処理フロー

```
PoolManager → イベント発行 → Envio インデクサー → PostgreSQL → GraphQL → ダッシュボード
```

### 追跡する主要イベント

1. **Initialize**: プール作成（Hookアドレスを含む）
2. **Swap**: 取引実行とHookの介入
3. **ModifyLiquidity**: LP操作とペナルティ適用
4. **Donate**: 手数料の寄付

## 📚 ドキュメント

- [ワークショップ45分ガイド（日本語）](./docs/japanese/workshop-45min.md) - 完全なワークショップガイド
- [アーキテクチャ詳細](./docs/ARCHITECTURE.md)
- [コントラクトREADME](./contracts/README.md)

## 🏆 Uniswap関連グラント

ワークショップ完了後は、Unichain Hook Grantへの応募をぜひご検討ください！

### グラント概要

- **主催者**: Uniswap Foundation & Atrium
- **対象**: Unichain上で動作する革新的なHooks
- **カテゴリー**: DeFiイノベーション、流動性最適化、MEV保護、新しいAMM設計

## 🔗 関連リンク

- [Uniswap V4 ドキュメント](https://docs.uniswap.org/contracts/v4/overview)
- [V4 SDK ドキュメント](https://docs.uniswap.org/sdk/v4/overview)
- [V4 Core リポジトリ](https://github.com/Uniswap/v4-core)
- [OpenZeppelin Hooks](https://github.com/OpenZeppelin/uniswap-hooks)
- [Hooks監査レポート](https://blog.openzeppelin.com/openzeppelin-uniswap-hooks-v1.1.0-rc-1-audit)
- [Hookデータ標準ガイド](https://www.uniswapfoundation.org/blog/developer-guide-establishing-hook-data-standards-for-uniswap-v4)
- [Unichainドキュメント](https://docs.unichain.org)
- [Envio HyperSyncドキュメント](https://docs.envio.dev/docs/HyperSync/overview)
- [Envio Uniswap V3 Analytics](https://github.com/enviodev/uniswap-v3-analytics)
- [Envio Uniswap V4 Indexer](https://github.com/enviodev/uniswap-v4-indexer)
- [日本コミュニティ](https://t.me/uniswapjp)
- [Uniswap V4 Dojo](https://t.me/c/1793969856/1)
- [DEX Analytics](https://dexanalytics.org/schemas/jit-liquidity-events)

---

**Uniswap V4でDeFiの未来を構築**