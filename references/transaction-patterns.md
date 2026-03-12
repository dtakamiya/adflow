# Spring トランザクションパターン — 銀行システム向けリファレンス

## 1. @Transactional 基本パターン

### 1.1 基本的な使い方
- Service層に配置する（Controller層には配置しない）
- クラスレベルではなくメソッドレベルで適用する
- publicメソッドにのみ有効

```java
@Service
public class AccountService {

    @Transactional
    public void transfer(TransferRequest request) {
        // トランザクション内で実行される
    }

    @Transactional(readOnly = true)
    public Account findById(Long id) {
        // 読み取り専用トランザクション
    }
}
```

### 1.2 rollbackFor の明示的指定
- デフォルトでは RuntimeException と Error のみロールバック
- checked exception もロールバック対象にする

```java
@Transactional(rollbackFor = Exception.class)
public void processPayment(PaymentRequest request) {
    // checked exceptionでもロールバックする
}
```

## 2. Propagation パターン

### 2.1 REQUIRED（デフォルト）
- 既存トランザクションがあれば参加、なければ新規作成
- ほとんどのケースでこれを使用

### 2.2 REQUIRES_NEW
- 常に新規トランザクションを作成
- 監査ログの記録に使用（メイントランザクションが失敗しても監査ログは残す）

```java
@Service
public class AuditLogService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(AuditEvent event) {
        // メイントランザクションとは独立して記録される
        auditLogRepository.save(event);
    }
}
```

### 2.3 MANDATORY
- 既存トランザクションが必須、なければ例外
- トランザクション内でのみ呼ばれるべきメソッドに使用

### 2.4 NOT_SUPPORTED
- トランザクション外で実行
- 外部API呼び出しなど、長時間処理に使用

## 3. 読み取り専用トランザクション

```java
@Transactional(readOnly = true)
public List<Account> findAll() {
    // フラッシュモードがMANUALになり、ダーティチェックが省略される
    // パフォーマンス向上と意図の明示化
    return accountRepository.findAll();
}
```

## 4. 分離レベル

### 4.1 READ_COMMITTED（推奨デフォルト）
```java
@Transactional(isolation = Isolation.READ_COMMITTED)
```
- ダーティリードを防止
- ほとんどの銀行業務で適切

### 4.2 SERIALIZABLE
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
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

```java
@Transactional
public void transfer(TransferRequest request) {
    accountRepository.debit(request.fromAccount(), request.amount());
    accountRepository.credit(request.toAccount(), request.amount());
    // 同一トランザクションでイベントをOutboxに記録
    outboxRepository.save(new OutboxEvent("TRANSFER_COMPLETED", request));
}
```

## 6. アンチパターン

### 6.1 Controllerでの@Transactional
```java
// NG: Controllerにトランザクションを配置
@RestController
public class AccountController {
    @Transactional // ← NG
    @PostMapping("/transfer")
    public ResponseEntity<?> transfer(...) { }
}
```

### 6.2 自己呼び出し（プロキシの罠）
```java
@Service
public class PaymentService {
    public void processAll(List<Payment> payments) {
        payments.forEach(this::processSingle); // ← @Transactionalが効かない！
    }

    @Transactional
    public void processSingle(Payment payment) { }
}
```
**対策**: 別クラスに分離するか、`self`参照を使用する

### 6.3 トランザクション内での外部API呼び出し
```java
@Transactional
public void process(Order order) {
    orderRepository.save(order);
    externalApi.notify(order); // ← NG: タイムアウトでコネクション保持
}
```
**対策**: `@TransactionalEventListener` で非同期化する

## 7. テスト時の注意

```java
@SpringBootTest
@Transactional // テスト後に自動ロールバック
class AccountServiceTest {

    @Test
    void testTransfer() {
        // テストコード — テスト終了時に自動ロールバック
    }

    @Test
    @Commit // ロールバックしたくない場合
    void testWithCommit() { }
}
```
