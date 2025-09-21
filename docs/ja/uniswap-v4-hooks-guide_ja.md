# Uniswap V4 Hook システム完全ガイド

## 概要

Uniswap V4のフックシステムは、プールの動作をカスタマイズするための強力なメカニズムです。フックコントラクトのアドレスの最下位14ビットを調べることで、どの特定のフックを呼び出すかを決定します。

## 14種類のフックとその使用方法

### 1. フック権限システム

V4は合計14のフックフラグを提供：10のコアフック関数と4のデルタ戻り値フラグ：

```solidity
// フックフラグ（アドレスの最下位ビット）
BEFORE_INITIALIZE_FLAG = 1 << 13        // 0010 0000 0000 0000
AFTER_INITIALIZE_FLAG = 1 << 12         // 0001 0000 0000 0000
BEFORE_ADD_LIQUIDITY_FLAG = 1 << 11     // 0000 1000 0000 0000
AFTER_ADD_LIQUIDITY_FLAG = 1 << 10      // 0000 0100 0000 0000
BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9   // 0000 0010 0000 0000
AFTER_REMOVE_LIQUIDITY_FLAG = 1 << 8    // 0000 0001 0000 0000
BEFORE_SWAP_FLAG = 1 << 7               // 0000 0000 1000 0000
AFTER_SWAP_FLAG = 1 << 6                // 0000 0000 0100 0000
BEFORE_DONATE_FLAG = 1 << 5             // 0000 0000 0010 0000
AFTER_DONATE_FLAG = 1 << 4              // 0000 0000 0001 0000
BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3 // 0000 0000 0000 1000
AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2  // 0000 0000 0000 0100
AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1  // 0000 0000 0000 0010
AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0 // 0000 0000 0000 0001
```

### 2. 各フックの詳細な使用例

#### 2.1 コアフック関数（10種類）

##### 2.1.1 初期化フック

**beforeInitialize**
- **目的**: プール初期化前のバリデーションと設定
- **使用例**: 
  - 特定のトークンペアのみ許可
  - 初期手数料率の設定
  - ホワイトリスト制御

**afterInitialize**
- **目的**: プール初期化後の追加セットアップ
- **使用例**: 
  - 自動初期流動性追加
  - イベントログ記録
  - 外部システムへの通知

##### 2.1.2 流動性フック

**beforeAddLiquidity / afterAddLiquidity**
- **目的**: 流動性追加の制御と追跡
- **使用例**:
  - JIT攻撃防止（LiquidityPenaltyHook）
  - 流動性提供者のKYC確認
  - カスタム手数料徴収
  - 最小/最大流動性制限

**beforeRemoveLiquidity / afterRemoveLiquidity**
- **目的**: 流動性削除の制御
- **使用例**:
  - 早期引き出しペナルティ
  - ロック期間の強制
  - 手数料の再分配

##### 2.1.3 スワップフック

**beforeSwap / afterSwap**
- **目的**: スワップの制御と監視
- **使用例**:
  - MEV保護（AntiSandwichHook）
  - 動的手数料調整
  - 取引量制限
  - 指値注文実行（LimitOrderHook）

##### 2.1.4 寄付フック

**beforeDonate / afterDonate**
- **目的**: プール寄付の処理
- **使用例**:
  - 最小寄付額の設定
  - 寄付者報酬
  - 寄付の追跡と記録

#### 2.2 デルタ戻り値フラグ（4種類）

**beforeSwapReturnDelta / afterSwapReturnDelta**
- **目的**: スワップ量の動的調整
- **使用例**:
  - カスタム価格曲線の実装
  - 動的手数料計算
  - スワップ量の修正

**afterAddLiquidityReturnDelta / afterRemoveLiquidityReturnDelta**
- **目的**: 流動性操作後の残高調整
- **使用例**:
  - カスタム手数料実装
  - リベートシステム
  - 自動流動性報酬分配

## 3. 詳細な実装例

### 3.1 LiquidityPenaltyHook（JIT攻撃防止）

```solidity
// 権限設定
Hooks.Permissions({
    afterAddLiquidity: true,              // 流動性追加後に手数料を保持
    afterRemoveLiquidity: true,           // 削除時にペナルティを適用
    afterAddLiquidityReturnDelta: true,   // 動的手数料調整
    afterRemoveLiquidityReturnDelta: true // ペナルティの戻り値
})
```

**動作原理**:
1. 流動性追加時にブロック番号を記録
2. 設定期間内（blockNumberOffset）の削除にペナルティを適用
3. ペナルティは時間経過とともに線形減少
4. 徴収されたペナルティは他のLPに分配

**パラメータ調整**:
- `blockNumberOffset`: 10-100ブロック（流動性と資産タイプに基づいて調整）
- 低流動性プール: 高い値を推奨
- 高流動性プール: 小さい値でも効果的

### 3.2 AntiSandwichHook（MEV保護）

```solidity
// 権限設定
Hooks.Permissions({
    beforeSwap: true,           // スワップ前にチェックポイントを確認
    afterSwap: true,            // スワップ後に状態を更新
    afterSwapReturnDelta: true  // 価格制限を適用
})
```

**動作原理**:
1. ブロック開始時の価格をチェックポイントとして保存
2. 同一ブロック内のスワップは開始価格より有利な価格で実行不可
3. zeroForOne方向は通常のxy=k曲線を使用
4. !zeroForOne方向は固定価格で実行

**考慮事項**:
- MEV耐性により裁定取引が減少
- ブロック開始価格が市場価格から乖離する可能性
- 大きな価格変動時のガス消費量に注意

### 3.3 LimitOrderHook（疑似指値注文）

```solidity
// 権限設定
Hooks.Permissions({
    afterInitialize: true,  // 初期化後にtickLowerLastを設定
    afterSwap: true        // スワップ後に注文を実行
})
```

**動作原理**:
1. 現在価格外のティックに流動性を配置
2. 価格がティックを越えたときに注文を実行
3. 実行後、フックが資金を保持
4. ユーザーは後で引き出し可能

**使用方法**:
- `placeOrder`: 注文を配置
- `cancelOrder`: 未約定注文をキャンセル
- `withdraw`: 約定済み注文を引き出し

## 4. フック組み合わせパターン

### 4.1 基本保護システム

```solidity
// JIT保護 + MEV保護
contract ProtectedPool {
    // LiquidityPenaltyHookとAntiSandwichHookの機能を統合
    function getHookPermissions() returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeSwap: true,
            afterSwap: true,
            afterAddLiquidity: true,
            afterRemoveLiquidity: true,
            afterSwapReturnDelta: true,
            afterAddLiquidityReturnDelta: true,
            afterRemoveLiquidityReturnDelta: true,
            // その他はfalse
        });
    }
}
```

### 4.2 高度な取引システム

```solidity
// 指値注文 + 動的手数料
contract AdvancedTradingHook {
    // 市場状況に基づく手数料調整と指値注文機能
    function getHookPermissions() returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            afterInitialize: true,
            beforeSwap: true,
            afterSwap: true,
            beforeSwapReturnDelta: true,
            // その他はfalse
        });
    }
}
```

## 5. 実装時の考慮事項

### 5.1 イベント

- イベントを発行する標準的な方法があります
- 詳細: [Hook Data Standards Guide](https://www.uniswapfoundation.org/blog/developer-guide-establishing-hook-data-standards-for-uniswap-v4)

### 5.2 msg.senderの処理

フック内で元のmsg.senderにアクセスするには特別な処理が必要

- 詳細: https://docs.uniswap.org/contracts/v4/guides/accessing-msg.sender-using-hook

### 5.3 ETH（ネイティブトークン）の処理

流動性ポジションでETHを扱う際はスイープ処理が必要

**重要な脆弱性例**:
- 詳細: https://x.com/electisec/status/1921211750185054216

### 6. エラー調査ツール

**エラーセレクタ調査用の4byte Directory使用法**:
- URL: https://www.4byte.directory/
- エラー発生時に返されるbytes4セレクタを検索
- 例: `0x7939f424` → `TransferFromFailed()`

使用方法:
1. トランザクションリバートデータから最初の4バイトを抽出
2. 4byte Directoryで検索
3. エラー関数シグネチャを確認

### 7. よくある問題

1. **HookAddressNotValid エラー**
   - 原因: アドレスのビットパターンが権限と一致しない
   - 解決策: HookMinerで正しいアドレスを再計算

2. **ガス制限エラー**
   - 原因: フック処理が複雑すぎる
   - 解決策: 処理を簡素化するかオフチェーンに移行

3. **予期しない動作**
   - 原因: フック実行順序の誤解
   - 解決策: beforeとafter処理フローを確認

4. **不明なエラー**
   - [4byte Directory](https://www.4byte.directory/)でエラーセレクタを検索
   - エラーの意味を理解してデバッグ