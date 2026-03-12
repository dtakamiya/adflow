# トランザクションパターン — ミッションクリティカルシステム向けリファレンス

## 1. トランザクション宣言 基本パターン

### 1.1 基本的な使い方
- サービス層に配置する（プレゼンテーション層には配置しない）
- クラスレベルではなくメソッドレベルで適用する
- 公開メソッドにのみ有効

```pseudo
class AccountService:

    // トランザクション宣言
    function transfer(request: TransferRequest):
        // トランザクション内で実行される

    // 読み取り専用トランザクション宣言
    function findById(id: Long): Account
        // 読み取り専用トランザクション
```

### 1.2 例外発生時のロールバック明示指定
- デフォルトでは非チェック例外のみロールバック
- チェック例外もロールバック対象にする

```pseudo
// トランザクション宣言（すべての例外でロールバック）
function processPayment(request: PaymentRequest):
    // checked exceptionでもロールバックする
```

## 2. トランザクション伝播パターン

### 2.1 REQUIRED（デフォルト）
- 既存トランザクションがあれば参加、なければ新規作成
- ほとんどのケースでこれを使用

### 2.2 REQUIRES_NEW（独立トランザクション）
- 常に新規トランザクションを作成
- 監査ログの記録に使用（メイントランザクションが失敗しても監査ログは残す）

```pseudo
class AuditLogService:

    // トランザクション宣言（独立トランザクション）
    function log(event: AuditEvent):
        // メイントランザクションとは独立して記録される
        auditLogRepository.save(event)
```

### 2.3 MANDATORY
- 既存トランザクションが必須、なければ例外
- トランザクション内でのみ呼ばれるべきメソッドに使用

### 2.4 NOT_SUPPORTED
- トランザクション外で実行
- 外部API呼び出しなど、長時間処理に使用

## 3. 読み取り専用トランザクション

```pseudo
// 読み取り専用トランザクション宣言
function findAll(): List<Account>:
    // ダーティチェックが省略される
    // パフォーマンス向上と意図の明示化
    return accountRepository.findAll()
```

## 4. 分離レベル

### 4.1 READ_COMMITTED（推奨デフォルト）
```pseudo
// トランザクション宣言（分離レベル: READ_COMMITTED）
```
- ダーティリードを防止
- ほとんどのミッションクリティカル業務で適切

### 4.2 SERIALIZABLE
```pseudo
// トランザクション宣言（分離レベル: SERIALIZABLE）
```
- 最も厳格 — 残高更新など金銭的に重要な操作に使用
- パフォーマンス影響に注意

## 5. 分散トランザクション

### 5.1 Sagaパターン
- マイクロサービス間のトランザクション整合性
- 各サービスのローカルトランザクション + 補償トランザクション

```
振込サービス → 出金(口座A) → 入金(口座B)
                  ↓失敗時          ↓失敗時
              補償不要        出金取消(口座A)
```

### 5.2 Outboxパターン
- イベント発行の信頼性を確保
- トランザクション内でOutboxテーブルに書き込み、非同期で発行

```pseudo
// トランザクション宣言
function transfer(request: TransferRequest):
    accountRepository.debit(request.fromAccount, request.amount)
    accountRepository.credit(request.toAccount, request.amount)
    // 同一トランザクションでイベントをOutboxに記録
    outboxRepository.save(OutboxEvent("TRANSFER_COMPLETED", request))
```

## 6. アンチパターン

### 6.1 プレゼンテーション層でのトランザクション宣言

```pseudo
// NG: プレゼンテーション層にトランザクションを配置
class AccountController:
    // トランザクション宣言 ← NG
    function transfer(...): Response
```

### 6.2 自己呼び出し（プロキシの罠）

```pseudo
class PaymentService:
    function processAll(payments: List<Payment>):
        payments.forEach(this.processSingle)  // ← トランザクションが効かない！

    // トランザクション宣言
    function processSingle(payment: Payment):
        ...
```
**対策**: 別クラスに分離するか、自己参照を使用する

### 6.3 トランザクション内での外部API呼び出し

```pseudo
// トランザクション宣言
function process(order: Order):
    orderRepository.save(order)
    externalApi.notify(order)  // ← NG: タイムアウトでコネクション保持
```
**対策**: トランザクションイベントリスナーで非同期化する

## 7. テスト時の注意

```pseudo
// テストフレームワークのアノテーション
// トランザクション宣言（テスト後に自動ロールバック）
class AccountServiceTest:

    function testTransfer():
        // テストコード — テスト終了時に自動ロールバック

    // ロールバックしたくない場合: コミット指定
    function testWithCommit():
        ...
```
