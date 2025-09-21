# Uniswap v4 Hooks実装パターン完全ガイド

## セキュリティパターン ★★★

### 1. onlyPoolManagerモディファイア：最も重要なアクセス制御 ★★★

#### 背景と問題

誰でもhook関数を呼び出せる場合、状態の整合性の問題や資金の盗難リスクが発生します。悪意のあるアクターが機密関数を直接呼び出して、hookの状態を操作したり資金を流出させたりする可能性があります。

#### 解決アプローチ

`onlyPoolManager`モディファイアを使用して、PoolManagerコントラクトからの呼び出しのみを許可します。これにより、hook関数が正しいコンテキストと順序で呼び出されることが保証されます。

#### 実装のポイント

- PoolManagerアドレスに対して`msg.sender`をチェック
- すべてのhookコールバック関数に適用
- 明確な失敗通知にカスタムエラーを使用
- 例外なし - 状態を変更する内部ヘルパー関数も保護が必要

#### 関連コード

- [`contracts/src/base/BaseHook.sol`](../contracts/src/base/BaseHook.sol#L66-L69)
  - L66-69: カスタムエラーを含むモディファイア定義
  - L109: beforeInitializeでの使用例
- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L420)
  - L420: unlockCallbackでの必須使用

### 2. クロスプールステート分離：マルチプールhookの必須パターン ★★★

#### 背景と問題

単一のhookが複数のプールで使用される場合、状態の混在により重大なバグやセキュリティ脆弱性が発生する可能性があります。悪意のあるプールが他のプールのデータを操作したりアクセスしたりする可能性があります。

#### 解決アプローチ

PoolIdを主キーとする二重マッピング構造を使用します。これにより各プールの状態を完全に分離し、クロス汚染を防ぎます。

#### 実装のポイント

- `mapping(PoolId => mapping(...))`パターンで完全分離
- プール間でデータが漏洩することはない
- 最小限のガスコスト増加（追加のSLOADが1回）
- 関連するプールデータをグループ化するために構造体の使用を検討

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L79)
  - L79: `_lastAddedLiquidityBlock` - プールごとのブロック番号
  - L84: `_withheldFees` - プールごとの保留手数料

### 3. Tick反復DoSリスク ★★★

#### 背景と問題

大きな価格変動時に、影響を受けるすべてのtickを反復処理すると、無制限のガスを消費する可能性があります。1-tick間隔のプールでは、10%の価格変動で数千のtickの反復処理が必要となり、ガス不足エラーやDoS攻撃の原因となります。

#### 解決アプローチ

反復制限を実装し、代替のデータ構造を検討します。現在の実装にはこれらの保護がなく、注意すべき例として機能しています。

#### 実装のポイント

- 価格上昇/下降に応じた異なるループ方向
- 最大tick数制限が必要（例：MAX_TICKS = 1000）
- 大規模範囲にはイベントまたはオフチェーンインデックスの使用を検討
- スタック深度エラーを防ぐためのメモリ使用量最適化

#### 関連コード

- [`contracts/src/AntiSandwichHook.sol`](../contracts/src/AntiSandwichHook.sol#L99-L117)
  - L99-117: Tick反復 - DoS脆弱性の例
  - 現在の実装では反復に上限がない

### 4. ゼロ流動性チェック：見落とされがちだが重要な検証 ★★☆

#### 背景と問題

ゼロ流動性での操作はゼロ除算エラー、無効な取引、潜在的なDoS攻撃の原因となります。ユーザーは失敗した取引でガスを無駄にしたり、攻撃者が意図的にこれらの条件をトリガーしたりする可能性があります。

#### 解決アプローチ

操作前に必ずゼロをチェックします。特定の問題についてユーザーに明確に通知する適切なカスタムエラーを使用します。

#### 実装のポイント

- 複数の場所での一貫したチェック
- 異なるゼロ条件に対する特定のエラーメッセージ
- ガスの無駄を防ぐための早期検証
- ゼロが復帰するか早期リターンするかを検討

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L164)
  - L164: 寄付前のプール流動性ゼロチェック
  - L64: `NoLiquidityToReceiveDonation`カスタムエラー
- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L247)
  - L247, L336, L401: 各操作でのゼロチェック

### 5. Unlockパターン：安全なコールバックベース実行 ★★☆

#### 背景と問題

PoolManagerとの複雑な相互作用を安全かつアトミックに実行する必要があります。直接呼び出しでは、取引が部分的に失敗した場合にプールが一貫性のない状態になる可能性があります。

#### 解決アプローチ

`poolManager.unlock`を使用してロックを解除し、コールバック内で処理し、完了時に自動的に再ロックします。これにより、すべての操作が完全に実行されるか、完全に復帰することが保証されます。

#### 実装のポイント

- CallbackData構造で操作タイプを指定
- abi.decodeを使用したタイプセーフな戻り値処理
- 原子性保証 - すべてまたは何もない実行
- パターンに組み込まれた再入攻撃保護

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L292-L301)
  - L292-301: placeOrderでのunlock使用と戻り値デコード
  - L420-428: unlockCallbackでの分岐ロジック

## JIT/MEV保護パターン ★★★

### 6. Uniswap v4でのJIT攻撃防御：革新的な時間ベースペナルティアプローチ ★★★

#### 背景と問題

JIT（Just-in-Time）攻撃は、大きなスワップの直前に流動性を追加して手数料を獲得し、その後すぐに引き出すものです。この寄生的行動は、より多くのリスクを負う長期流動性提供者から収益を奪い、DEXエコシステムの持続可能性を損ないます。

#### 解決アプローチ

時間ベースのペナルティシステムを実装します。指定期間（`blockNumberOffset`）内に流動性を除去する際、手数料に線形減衰ペナルティを適用します。ペナルティは100%から始まり、保護期間中に0%まで減衰します。

#### 実装のポイント

- 一時的手数料保留メカニズムがhookに手数料を保存
- 線形減衰関数：`penalty = fees * (1 - elapsedBlocks / offset)`
- ペナルティは現在のレンジ内LPに寄付され、長期提供者に利益をもたらす
- 典型的なオフセット値：10-100ブロック（Ethereumで2-20分）

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L234-L251)
  - L234-251: `_calculateLiquidityPenalty` - オーバーフロー保護付き線形減衰ペナルティ計算
  - L100-122: `_afterAddLiquidity` - JIT保護期間中の手数料保留
  - L136-180: `_afterRemoveLiquidity` - ペナルティ適用と寄付ロジック

### 7. 非対称MEV防止設計：なぜ買い方向のみを保護するか ★★★

#### 背景と問題

サンドイッチ攻撃では、攻撃者が被害者の前後に取引して価値を抽出します。典型的なパターンは：攻撃者が購入（価格を押し上げ）→ 被害者が購入（インフレした価格で）→ 攻撃者が売却（利益を獲得）です。しかし、すべての方向を保護すると正当な価格発見が阻害されます。

#### 解決アプローチ

買い方向（!zeroForOne）のみをブロック開始価格で固定し、売り方向は通常のxy=k曲線を使用します。これによりサンドイッチ攻撃の経済性を破綻させながら、一部の価格発見を維持します。注：コードコメントではzeroForOneが保護されるとありますが、実装では!zeroForOneが保護されます。

#### 実装のポイント

- 各ブロックの最初のスワップで完全なプール状態を保存
- 買い方向（token1 → token0）のみに固定価格を適用
- 市場価格と固定価格の差を計算
- 差額をプールLPへの補償として寄付

#### 関連コード

- [`contracts/src/AntiSandwichHook.sol`](../contracts/src/AntiSandwichHook.sol#L147-L171)
  - L147-171: `_getTargetUnspecified` - 方向ベース処理分岐
  - L154-164: `Pool.swap` - ブロック開始状態でのシミュレーション
- [`contracts/src/AntiSandwichHookImpl.sol`](../contracts/src/AntiSandwichHookImpl.sol#L12-L22)
  - L12-22: `_afterSwapHandler` - 手数料収集処理

## ステート管理パターン ★★★

### 8. 一時的ストレージ（EIP-1153）のDeFi革命 ★★★

#### 背景と問題

通常のストレージは高いガスコストがかかります（SSTORE: 20,000ガス）。トランザクション内でのみ必要な一時データの保存は非効率で高価です。さらに、永続ストレージは再入攻撃リスクを作成し、明示的なクリーンアップが必要です。

#### 解決アプローチ

EIP-1153の一時的ストレージ（TSTORE/TLOAD）を活用します。データはトランザクション終了時に自動的にクリアされ、ガスコストが大幅に削減されます（~100ガス）。これにより自然な再入保護が提供され、状態汚染リスクが排除されます。

#### 実装のポイント

- beforeSwap→afterSwap間での永続ストレージなしの状態受け渡し
- 即座のリセットによりスワップ間での状態汚染を防止
- 再入攻撃への自然な耐性
- モディファイアやtry/catchブロックでの明示的クリーンアップが不要

#### 関連コード

- [`contracts/src/fee/BaseDynamicAfterFee.sol`](../contracts/src/fee/BaseDynamicAfterFee.sol#L63-L86)
  - L63-86: `_transientTargetUnspecifiedAmount` - 読み取り/書き込みヘルパー関数
  - L98-111: `_beforeSwap` - ターゲット値の一時保存
  - L133-144: `_afterSwap` - 安全のための即座リセットパターン

### 9. Position.calculatePositionKey：衝突フリーポジション管理 ★★☆

#### 背景と問題

同じtick範囲での異なるユーザーのポジションや、同一ユーザーの複数ポジションを区別する必要があります。単純な連結では衝突が発生したり、複雑な文字列操作が必要になったりする可能性があります。

#### 解決アプローチ

4つのパラメータ（sender、tickLower、tickUpper、salt）をkeccak256でハッシュします。これにより決定論的で衝突耐性のある32バイト識別子を作成します。衝突確率は事実上ゼロです（2^256の可能値）。

#### 実装のポイント

- 決定論的計算によりオフチェーン計算が可能
- saltパラメータにより同一範囲でのユーザーごとの複数ポジションを許可
- 固定32バイトキー長により一定のガスコスト
- 一貫性のためにパラメータの順序が重要

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L109)
  - L109: 流動性追加時のキー計算
  - L145: 流動性除去で同じキーを使用
  - saltパラメータがポジション管理の柔軟性を提供

### 10. 効率的なSlot0パッキング：256ビットでの設計思想 ★★☆

#### 背景と問題

プールの主要状態変数を別々に保存すると、複数のSLOAD（各3,000ガス）が必要になります。価格、tick、手数料に個別にアクセスすると最低12,000ガスがかかります。

#### 解決アプローチ

4つの重要な値を1つの256ビットスロットにパックします。単一のSLOAD（3,000ガス）ですべての情報を取得します。慎重なビット配分により精度の損失を防ぎます。

#### 実装のポイント

- sqrtPriceX96: 160ビット - Q64.96形式の価格の平方根
- tick: 24ビット - 現在のtick（±800万の範囲をサポート）
- protocolFee: 24ビット - プロトコル手数料（ベーシスポイントの100分の1単位）
- lpFee: 24ビット - LP手数料（同じ精度）
- 将来の拡張用に24ビットが未使用

#### 関連コード

- [`contracts/src/AntiSandwichHook.sol`](../contracts/src/AntiSandwichHook.sol#L94)
  - L94: `Slot0.wrap(poolManager.extsload(...))` - 直接ストレージ読み取り
  - L119-120: `poolManager.getFeeGrowthGlobals` - 追加状態取得

## タイプセーフティパターン ★★☆

### 11. OrderIdライブラリ：カスタムタイプによるタイプセーフティ ★★☆

#### 背景と問題

uint256を直接使用すると、異なる目的のIDを混同するリスクがあります。OrderIdを期待する関数が誤ってPoolIdを受け取り、サイレント失敗を引き起こす可能性があります。コンパイル時のタイプチェックはプリミティブ型では役に立ちません。

#### 解決アプローチ

Solidity 0.8.8のカスタムタイプ機能を活用します。強力なタイピングのために`type OrderId is uint232`を定義します。これにより実行時オーバーヘッドなしにコンパイル時保証を作成します。

#### 実装のポイント

- 事故を防ぐために明示的なタイプ変換（wrap/unwrap）が必要
- タイプ混同バグをコンパイル時にキャッチ
- ゼロ実行時ガスコスト - 純粋にコンパイル時機能
- `equals`や`increment`などのタイプ固有メソッドを追加可能

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L24-L42)
  - L24-42: `OrderIdLibrary` - カスタムタイプ定義
  - L32-34: `equals` - タイプセーフな比較関数
  - L37-41: `unsafeIncrement` - uncheckedでのオーバーフロー耐性インクリメント

### 12. SafeCast：安全なタイプ変換 ★★☆

#### 背景と問題

タイプ変換のオーバーフロー/アンダーフローは予期しない動作やセキュリティ脆弱性を引き起こします。int256(-1)をuint256に変換すると2^256-1になり、エラーではありません。サイレント失敗によりデバッグが困難になります。

#### 解決アプローチ

SafeCastライブラリは変換時に値が適合することを検証します。オーバーフロー/アンダーフロー時に明確なエラーでリバートします。一般的なケースに対して明示的で安全な変換メソッドを提供します。

#### 実装のポイント

- uint256→uint128のような縮小変換では上位ビットをチェック
- int→uintのような符号変換では負の値をチェック
- 説明的なリバート理由での明示的エラー処理
- 潜在的バグコストと比較してガスオーバーヘッドは最小限

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L244)
  - L244: `SafeCast.toUint128` - オーバーフローチェック付き手数料量変換
  - L54: using宣言 - `using SafeCast for uint256`

### 13. BalanceDeltaエレガンス：デュアルトークン管理パターン ★☆☆

#### 背景と問題

DEXは常に2つのトークンを同時に処理します。それらを別々に管理すると冗長なコード、重複したロジック、エラーの可能性が増加します。両方のトークンの量が一貫して処理されることを保証する必要があります。

#### 解決アプローチ

BalanceDeltaタイプは1つの構造体で2つのトークンデルタを管理します。演算子オーバーロードにより直感的な操作を可能にします。これにより両方のトークン量が常に一緒に処理されることが保証されます。

#### 実装のポイント

- 加算/減算操作の自然な記法
- 簡潔な符号反転実装
- 必要時のamount0/amount1への簡単な分解
- 一方のトークンの処理を忘れることを防止

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L151)
  - L151: `totalFees = feeDelta + withheldFees` - 自然な加算
  - L170: `liquidityPenalty - withheldFees` - 減算操作
  - L175: `toBalanceDelta(-amount0, -amount1)` - 符号反転

## トークン処理パターン ★★☆

### 14. CurrencySettlerパターン：安全なトークン決済実装 ★★☆

#### 背景と問題

トークン転送にはERC20、ネイティブトークン、ERC-6909の異なる処理が必要です。それぞれに固有のインターフェースとエッジケースがあります。エラーを防ぎコードの重複を減らすための統一インターフェースが必要です。

#### 解決アプローチ

CurrencySettlerライブラリは`take`（受け取り）と`settle`（支払い）操作による抽象化を提供します。固有の要件を尊重しながら、一貫したインターフェースですべてのトークンタイプを処理します。

#### 実装のポイント

- msg.valueとsyncを使用したネイティブトークン処理
- 仮想残高のためのERC-6909 burn/mintサポート
- ゼロ量の早期リターン（一部のトークンはゼロ転送でリバート）
- hookと外部アドレスからの転送を区別

#### 関連コード

- [`contracts/src/utils/CurrencySettler.sol`](../contracts/src/utils/CurrencySettler.sol#L31-L51)
  - L31-51: `settle` - 統一支払い実装
  - L39-42: syncを使用したネイティブトークン特別処理
  - L61-68: `take` - 統一受け取り実装

### 15. ERC-6909：実際のトークンを保持しない管理 ★★☆

#### 背景と問題

大量のトークンを保有するhookはセキュリティリスクを作成し、攻撃者のハニーポットになります。しかし、リミット注文、手数料徴収、その他機能のために一時的な保管が必要です。

#### 解決アプローチ

仮想会計にERC-6909（マルチトークン標準）を使用します。PoolManagerが仮想トークン所有権を追跡し、hookは実際のトークンを保持しません。実際の転送は最終決済時のみ発生します。

#### 実装のポイント

- 仮想発行に`mint`、仮想燃焼に`burn`
- 最終`take`操作時のみ実際のトークン移動
- 複数トークンのガス効率的なバッチ管理
- hookとPoolManager間の承認要件を排除

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L470-L478)
  - L470-478: 注文約定時のMint - 仮想トークン発行
  - L434-444: 引き出し時のBurn→take - 仮想から実体への変換
  - L436-438: キャンセル時の手数料mint - 手数料の仮想保持

### 16. Donate関数：価格影響なしでLPへの報酬分配 ★☆☆

#### 背景と問題

収集した手数料（MEV防止やペナルティから）をLPに公平に分配する必要があります。流動性の追加はプール価格を変更します。直接転送には個々のLPシェアの追跡が必要です。

#### 解決アプローチ

`poolManager.donate`を使用してプールにトークンを寄付します。流動性に基づいて既存のLPシェアに比例して自動分配します。流動性は追加されないため価格影響はありません。

#### 実装のポイント

- 現在レンジ内のLPのみが寄付を受け取る
- プールがゼロ流動性の場合（受取人なし）はリバート
- amount0/amount1を独立して指定可能
- 公平な分配のためにfee growth globalsを更新

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L166-L168)
  - L166-168: `poolManager.donate` - LPへのペナルティ寄付
  - L164: 流動性チェック - 寄付受取人の存在確認

## 高度実装パターン ★★☆

### 17. DEXでのリミット注文実装：単一Tick流動性の活用 ★★★

#### 背景と問題

従来のAMMはリミット注文をサポートしません - ユーザーは市場価格でのみ取引可能です。中央集権的注文マッチングやオフチェーンコンポーネントなしに、DEXでCEXのような取引体験が必要です。

#### 解決アプローチ

Uniswap v4のレンジ外流動性特性を活用します。単一tick範囲（最小幅）に流動性を配置して特定価格での実行を保証します。プールの価格発見メカニズムが注文マッチングエンジンになります。

#### 実装のポイント

- 単一tick幅流動性のみ（例：tick 1000から1010）
- 一方向トークン預入：売り注文はtoken0預入、買い注文はtoken1預入
- 価格がtickを横切る際の自動実行 - キーパー不要
- ガス効率のため複数ユーザーの注文が単一tickに集約

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L477-L516)
  - L477-516: `_handlePlaceCallback` - 単一tick流動性追加
  - L557-594: `_fillOrder` - 価格横断時の自動実行
  - L246-315: `placeOrder` - ユーザー向け注文作成インターフェース

### 18. Pool.swapシミュレーション：状態変更なしでの結果計算 ★★☆

#### 背景と問題

AntiSandwichHookは実際のプールを変更せずに、ブロック開始価格での取引結果を知る必要があります。手数料や保護措置を決定するための正確な価格計算が必要です。

#### 解決アプローチ

保存されたチェックポイント状態と`Pool.swap`ライブラリ関数を使用して理論値を計算します。メモリ状態で純粋関数として実行し、実際のプールは変更されません。

#### 実装のポイント

- 完全にメモリ状態で計算
- 実際のプールストレージへの変更なし
- 正確な価格と手数料計算
- 複数シナリオの効率的シミュレーション

#### 関連コード

- [`contracts/src/AntiSandwichHook.sol`](../contracts/src/AntiSandwichHook.sol#L154-L164)
  - L154-164: `Pool.swap` - チェックポイント状態でのシミュレーション
  - L87: `_lastCheckpoint.state` - 保存された状態構造の使用

### 19. 手数料と元本の分離：LP公平性の保護 ★★☆

#### 背景と問題

リミット注文で、注文をキャンセルするユーザーが他者の獲得手数料を盗む可能性があります。手数料が適切に追跡されない場合、ユーザーは注文を素早く配置・キャンセルして蓄積された手数料をすくい取れます。

#### 解決アプローチ

キャンセル時に残りの注文配置者に手数料を分配します。最後のキャンセラーのみが手数料も受け取ります。チェックポイントを追跡して、ユーザーが参加後に獲得した手数料のみを受け取ることを保証します。

#### 実装のポイント

- `removingAllLiquidity`フラグが手数料分配を決定
- 手数料ミントプロセスが手数料を元本から分離
- principalDelta計算メソッドがコンテキストに基づいて切り替え
- チェックポイントシステムが手数料スキミング攻撃を防止

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L532-L550)
  - L532-550: 条件付きルーティングを含む手数料分配ロジック
  - L541-545: 残りの注文配置者への手数料ミント
  - L547-549: 最後の注文がすべての残り手数料を受け取る

## ガス最適化パターン ★★☆

### 20. uncheckedブロックの適切な使用：安全なガス最適化 ★★☆

#### 背景と問題

Solidity 0.8以降、算術演算には自動オーバーフローチェックがあり、操作あたり約35ガスのコストが増加します。事前検証によりオーバーフローが不可能な場合、これらのチェックは無駄です。

#### 解決アプローチ

オーバーフローが証明的に不可能な場合、`unchecked`ブロックを使用して冗長チェックをスキップします。なぜオーバーフローが発生しないかをインラインコメントで常に文書化します。

#### 実装のポイント

- uncheckedブロック前の必須前提条件検証
- 監査者のための安全性根拠をコメントで文書化
- 算術演算で20-30%のガス節約
- ユーザー入力や予測不可能な値には決して使用しない

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L253-L263)
  - L253-263: `_calculateLiquidityPenalty` - 検証された前提条件での計算
  - L157: 前提条件 - `current - last < offset`の保証によりオーバーフロー防止

### 21. 早期リターンパターン：基本的ガス最適化 ★☆☆

#### 背景と問題

条件が満たされない時に処理を続行すると、不要な計算とストレージ読み取りでガスが無駄になります。深いネストもコードの監査を困難にします。

#### 解決アプローチ

条件を早期にチェックして不要な処理をスキップします。最も安価なチェックから最も高価なチェックの順で並べます。これによりガスを節約しコードの可読性を向上します。

#### 実装のポイント

- 最も安価なチェックを最初に実行（ストレージ読み取り前のブールフラグなど）
- 早期リターンで処理を終了
- より良い可読性のためのネスト深度削減
- ストレージクリーンアップからのガス還付を検討

#### 関連コード

- [`contracts/src/fee/BaseDynamicAfterFee.sol`](../contracts/src/fee/BaseDynamicAfterFee.sol#L139-L141)
  - L139-141: 高価な操作前のapplyTargetチェック
- [`contracts/src/utils/CurrencySettler.sol`](../contracts/src/utils/CurrencySettler.sol#L33)
  - L33, L65: amount=0ケースの早期リターン

### 22. エラー設計哲学：revertメッセージからカスタムエラーへ ★☆☆

#### 背景と問題

`require(condition, "Error message")`は文字列をオンチェーンで保存し、デプロイメントに大きなガスコストがかかります（文字あたり約200ガス）。エラータイプも不明確で追加データを運べません。

#### 解決アプローチ

特定のエラータイプをカスタムエラーで定義します。これらは4バイトセレクタにコンパイルされ、デバッグ用パラメータを含められます。エラー処理を改善しながらガスを節約します。

#### 実装のポイント

- 4バイトセレクタのみがオンチェーンで保存
- try/catchブロックでのタイプセーフなエラー処理
- リバートあたり約2,000ガスの節約
- デバッグ用の関連値を含められる

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L56-L64)
  - L56-64: 説明的名前でのエラー定義
  - L90, L164: 特定のコンテキストでの使用例
- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L151-L163)
  - L151-163: 異なる失敗モードに対する5つの異なるエラータイプ

## ユーティリティパターン ★★☆

### 23. Hookアドレスマイニング：CREATE2の創造的使用 ★★☆

#### 背景と問題

Uniswap v4では、hook権限（使用できるコールバック）はコントラクトアドレスの下位14ビットで決定されます。デプロイメント前に特定のビットパターンを持つアドレスを見つける必要があります。

#### 解決アプローチ

CREATE2の決定論的アドレス生成を活用します。必要なビットパターンを持つアドレスを見つけるためにsalt値をブルートフォースします。デプロイメント前にアドレスを事前計算します。

#### 実装のポイント

- 適切なsaltを見つけるため最大160,000回の反復
- ガス見積もりのためにデプロイメント前にアドレスを計算
- 検索時間とデプロイメント柔軟性のバランス
- 特定の権限組み合わせを対象にできる

#### 関連コード

- [`contracts/src/utils/HookMiner.sol`](../contracts/src/utils/HookMiner.sol#L23-L41)
  - L23-41: `find` - saltブルートフォース実装
  - L35-38: ビットマスクチェック - 14ビット権限検証
  - L48-56: `computeAddress` - CREATE2アドレス公式

### 24. StateLibrary：効率的プールステートアクセス ★☆☆

#### 背景と問題

個別のgetter関数を通じて様々なプールステート変数にアクセスするのは非効率です。複数の外部呼び出しはガスコストと遅延を増加させます。

#### 解決アプローチ

StateLibraryは低レベルアクセスの抽象化を提供します。複数の値が必要な場合、ストレージスロットを直接読み取れます。効率性を改善しながらタイプセーフティを維持します。

#### 実装のポイント

- `_getPoolStateSlot`でスロットを計算
- 生データに`extsload`で直接読み取り
- 適切なキャストでタイプセーフティを維持
- 可能な場合は関連読み取りをバッチ処理

#### 関連コード

- [`contracts/src/AntiSandwichHook.sol`](../contracts/src/AntiSandwichHook.sol#L94)
  - L94: 効率性のための直接Slot0読み取り
  - L121: ライブラリを使用した流動性取得
- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L665)
  - L665: `getSlot0`使用例

### 25. Actions定数：マルチコール操作の標準化 ★☆☆

#### 背景と問題

より良いUXとガス効率のために1つのトランザクションで複数の操作を実行する必要があります。操作タイプは異なるコントラクト間で統一管理が必要です。

#### 解決アプローチ

Actions定数がすべての操作を数値的に定義します。ルーターがこれらの定数を解釈して対応する操作を実行します。柔軟な操作構成を可能にします。

#### 実装のポイント

- ガス効率のための数値操作識別
- 将来の操作に対する拡張可能な設計
- 単一バイト比較による効率的ディスパッチ
- すべての周辺コントラクト間での一貫性

#### 関連コード

- [`contracts/src/libraries/Actions.sol`](../contracts/src/libraries/Actions.sol#L10-L42)
  - L10-19: 流動性操作
  - L21-25: スワップ操作
  - L29-42: 決済操作

## 特殊実装パターン ★☆☆

### 26. Tick丸め落とし穴：負数の特別処理 ★★☆

#### 背景と問題

Tickは価格範囲境界を表します。負のtickの不正な丸めは無効な範囲や価格計算エラーを引き起こします。標準除算はゼロに向かって丸めますが、tick数学は負の無限大に向かう丸めが必要です。

#### 解決アプローチ

負のtickには負の無限大（より小さい値）に向かって丸めます。通常の除算はゼロに向かって丸めるため、負の余りに対する特別調整が必要です。

#### 実装のポイント

- 余りに`tick % tickSpacing != 0`をチェック
- 負の余りの場合、圧縮値から1を減算
- 正と負の範囲間での価格一貫性を保証
- リミット注文と範囲ポジション計算に重要

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L727-L732)
  - L727-732: `getTickLower` - 負tick処理
  - L730: 条件チェック - 負で割り切れない場合

### 27. FullMathライブラリ：DeFiの必須精度計算 ★★☆

#### 背景と問題

`a * b / c`計算で、最終結果がfitしても`a * b`がuint256をオーバーフローする可能性があります。例：中間乗算が2^256を超える手数料シェア計算。

#### 解決アプローチ

FullMath.mulDivは内部で512ビット中間値を使用します。精度を維持しながらオーバーフローを防ぎます。Remco Bloemenの実装に基づいています。

#### 実装のポイント

- 512ビット精度で内部乗算を処理
- 正確性のため同じ操作で除算を実行
- 数学的正確性とガス効率のバランス
- 手数料計算と価格変換に重要

#### 関連コード

- [`contracts/src/LiquidityPenaltyHook.sol`](../contracts/src/LiquidityPenaltyHook.sol#L254-L258)
  - L254-258: ペナルティ計算での使用
  - 分子: `feeDelta * (offset - elapsed)`
  - 分母: `offset`

### 28. TickMath：価格とTick変換 ★☆☆

#### 背景と問題

Uniswap v4は価格をtick（1.0001の累乗）として表現します。sqrtPriceX96形式とtick間の頻繁な変換が必要です。素朴な実装はガス高となります。

#### 解決アプローチ

TickMathライブラリはビット操作と事前計算定数を使用した最適化変換を提供します。EVM用に最適化された対数計算。

#### 実装のポイント

- `getTickAtSqrtPrice`: バイナリサーチで価格→tick
- `getSqrtPriceAtTick`: ビットシフトでtick→価格
- MIN_TICK/MAX_TICK境界検証
- 一般的な価格範囲でガス最適化

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L746-L749)
  - L746-749: `getTick`関数での使用
  - sqrt価格から現在のtickを計算

### 29. exactInput vs exactOutput：非対称手数料適用ロジック ★☆☆

#### 背景と問題

スワップには「入力量指定」と「出力量指定」の2タイプがあり、ユーザーの指定量を保持するために異なる手数料適用方法が必要です。

#### 解決アプローチ

Uniswap v4は`amountSpecified`の符号でスワップタイプを区別します：

- **負の値 (< 0)**: exactInput - 入力量指定（例：「1 ETHを売る」）
- **正の値 (> 0)**: exactOutput - 出力量指定（例：「1000 USDCを買う」）

手数料は常に「非指定」側に適用されます：

- exactInput: 手数料は出力量を減少
- exactOutput: 手数料は入力量を増加

#### 実装のポイント

```solidity
// 符号からスワップタイプを決定
bool exactInput = params.amountSpecified < 0;

// 非指定通貨を識別
// zeroForOne: token0→token1スワップ方向
// amountSpecified < 0: exactInput（入力指定）
// 両方が同時にtrue/falseの場合、token1が非指定（出力）
(Currency unspecified, int128 unspecifiedAmount) =
    (params.amountSpecified < 0 == params.zeroForOne)
    ? (key.currency1, delta.amount1())  // token1が非指定
    : (key.currency0, delta.amount0()); // token0が非指定
```

#### 具体例：

- **exactInput + zeroForOne（ETH→USDC、1 ETHを売る）**

  - `amountSpecified = -1e18`（負）
  - `zeroForOne = true`
  - 非指定 = USDC（出力側）
  - 手数料はUSDC出力を減少

- **exactOutput + !zeroForOne（USDC→ETH、1 ETHを買う）**
  - `amountSpecified = 1e18`（正）
  - `zeroForOne = false`
  - 非指定 = USDC（入力側）
  - 手数料はUSDC入力を増加

#### 関連コード

- [`contracts/src/fee/BaseDynamicAfterFee.sol`](../contracts/src/fee/BaseDynamicAfterFee.sol#L151)
  - L151: `bool exactInput = params.amountSpecified < 0` - 符号ベース決定
  - L147-149: 非指定通貨と量の識別
  - L161-175: exactInput/exactOutputの異なる手数料計算

### 30. Salt戦略：使用するとき、使用しないとき ★☆☆

#### 背景と問題

同じtick範囲で複数ポジションを望む同一ユーザーには識別子が必要です。異なるユースケースには異なるポジション管理戦略が必要です。

#### 解決アプローチ

- 個別管理：ポジションごとに固有saltを使用
- Hook管理：統一管理にsalt=0を使用
- 柔軟なポジション識別戦略を可能にする

#### 実装のポイント

- LimitOrderHookはすべてのユーザーにsalt=0を使用（hook集合管理）
- 通常のLPは個別ポジションに任意のsalt値を使用
- Saltがポジションキーの一意性を保証
- 異なるユースケースのsalt生成戦略を検討

#### 関連コード

- [`contracts/src/LimitOrderHook.sol`](../contracts/src/LimitOrderHook.sol#L491)
  - L491: `salt: 0` - Hook統一管理
  - L109, L145: saltパラメータを使用するcalculatePositionKey

### 31. Currency型順序：なぜCurrencyにサイズ関係があるか ★☆☆

#### 背景と問題

異なる順序の同じトークンペアが異なるプールを作成し、流動性を断片化させます。プールの一意性のために標準的な順序が必要です。

#### 解決アプローチ

トークンアドレスを数値的に比較し、常により小さいアドレスをcurrency0として設定します。これにより入力順序に関係なく決定論的で一意なプール識別を作成します。

#### 実装のポイント

- アドレス数値比較（アドレスは160ビット整数）
- 入力順序に関係なくプールの一意性を保証
- プール発見を簡素化する標準的順序
- Uniswap v2/v3パターンと一貫

#### 関連コード

- Currencyライブラリ内で暗黙的に使用
- PoolKey構造がcurrency0 < currency1を強制

### 32. Hook権限設計：アドレスビットマスクの活用 ★★☆

#### 背景と問題

各hookがどのコールバックを使用できるかを従来のアクセス制御で管理するのはガス集約的で非柔軟です。

#### 解決アプローチ

hookアドレスの下位14ビットを権限フラグとして使用します。各ビットが特定のコールバックの権限を表します。権限はアドレス自体に固有です。

#### 実装のポイント

- 各ビットが特定のコールバック権限に対応
- 希望する権限にHookMinerでアドレスを事前計算
- 単一AND操作でガス効率的権限チェック
- 不変権限により実行時改ざんを防止

#### 関連コード

- [`contracts/src/utils/HookMiner.sol`](../contracts/src/utils/HookMiner.sol#L10)
  - L10: `FLAG_MASK = 0x3FFF` - 14ビットマスク定数
  - L35-38: ビットマスクでの権限チェックロジック
- [`contracts/src/base/BaseHook.sol`](../contracts/src/base/BaseHook.sol#L99-L101)
  - L99-101: `validateHookAddress` - デプロイ時検証

### 33. HookFeeイベント：標準化された手数料追跡 ★★☆

#### 背景と問題

Hookは様々なメカニズム（MEV保護、動的手数料など）を通じて手数料を収集しますが、異なるhook間での手数料追跡は一貫していませんでした。オフチェーン監視と会計のための標準化されたイベント発行が必要でした。

#### 解決アプローチ

`IHookEvents`インターフェースで標準化`HookFee`イベントを定義します。手数料を収集するすべてのhookが一貫したパラメータでこのイベントを発行し、エコシステム全体での統一された手数料追跡を可能にします。

#### 実装のポイント

```solidity
event HookFee(
    bytes32 indexed poolId,
    address indexed sender,
    uint128 feeAmount0,
    uint128 feeAmount1
);
```