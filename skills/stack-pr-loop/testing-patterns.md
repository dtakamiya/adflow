# ドメイン品質テストパターン — ミッションクリティカルシステム向け

言語非依存の疑似コードによるテストパターン集。TDDのREDフェーズでこれらのパターンを参照し、ドメイン品質テストを組み込む。

---

## §1 トランザクションロールバックテスト

### パターン1.1: 例外発生時のロールバック確認

```pseudo
// Given: 有効な口座（残高 1000）
account = createAccount(balance = Decimal("1000"))

// When: 振込処理中に例外が発生する
mock externalService.notify() to throw RuntimeException("Network error")

try:
    transferService.transfer(account.id, toAccount.id, Decimal("500"))
catch:
    pass

// Then: 残高が変更されていないこと
reloadedAccount = accountRepository.findById(account.id)
assert reloadedAccount.balance == Decimal("1000")
```

### パターン1.2: 部分コミット防止（複数テーブル更新時）

```pseudo
// Given: 送金元口座（残高 1000）と送金先口座（残高 500）
fromAccount = createAccount(balance = Decimal("1000"))
toAccount   = createAccount(balance = Decimal("500"))

// When: 入金処理（2番目の操作）で失敗する
mock accountRepository.credit() to throw DataAccessException("DB error")

try:
    transferService.transfer(fromAccount.id, toAccount.id, Decimal("300"))
catch:
    pass

// Then: 出金も入金もどちらもコミットされていないこと
assert accountRepository.findById(fromAccount.id).balance == Decimal("1000")
assert accountRepository.findById(toAccount.id).balance   == Decimal("500")
```

### パターン1.3: 独立トランザクション（監査ログはロールバックされない）

```pseudo
// Given: 有効な口座
account = createAccount(balance = Decimal("1000"))

// When: メイン処理が失敗する
try:
    transferService.transfer(account.id, invalidAccountId, Decimal("500"))
catch:
    pass

// Then: メイン処理はロールバックされるが、監査ログは記録されている
assert accountRepository.findById(account.id).balance == Decimal("1000")
auditLogs = auditLogRepository.findByResourceId(account.id)
assert auditLogs.size() >= 1
assert auditLogs.last().result == "FAILURE"
```

---

## §2 監査ログ検証テスト

### パターン2.1: 状態変更時の監査ログ記録確認

```pseudo
// Given: 有効な口座と認証済みユーザー
account = createAccount(balance = Decimal("1000"))
authenticateAs(user = "operator-1")

// When: 状態変更操作を実行する
transferService.transfer(account.id, toAccount.id, Decimal("500"))

// Then: 監査ログが記録されていること
auditLogs = auditLogRepository.findByCorrelationId(currentCorrelationId())
assert auditLogs.size() == 1

log = auditLogs.first()
assert log.eventType    == "ACCOUNT"
assert log.action       == "TRANSFER"
assert log.userId       == "operator-1"
assert log.resourceType == "Account"
assert log.resourceId   == account.id.toString()
assert log.result       == "SUCCESS"
assert log.timestamp    != null
assert log.correlationId != null
```

### パターン2.2: 監査ログにPIIが含まれていないこと

```pseudo
// Given: PII（個人識別情報）を含む操作
account = createAccount(
    ownerName    = "山田太郎",
    email        = "yamada@example.com",
    phoneNumber  = "090-1234-5678",
    balance      = Decimal("1000")
)

// When: 状態変更操作を実行する
accountService.updateProfile(account.id, newName = "山田花子")

// Then: 監査ログのdetailsにPIIが平文で含まれていないこと
log = auditLogRepository.findLatest()
details = log.details

assert "山田太郎"        not in details
assert "山田花子"        not in details
assert "yamada@example.com" not in details
assert "090-1234-5678"  not in details
// マスキング済みの値は含まれてよい
assert "山田**" in details or "****@example.com" in details  // OK
```

### パターン2.3: ハッシュチェーンの整合性検証

```pseudo
// Given: 複数の監査ログが存在する
transferService.transfer(account1.id, account2.id, Decimal("100"))
transferService.transfer(account2.id, account3.id, Decimal("200"))
transferService.transfer(account3.id, account1.id, Decimal("50"))

// When: チェーンの整合性を検証する
result = auditIntegrityService.verifyChain()

// Then: チェーンが正常であること
assert result == true

// And: 各ログのハッシュが前のログのハッシュを参照していること
logs = auditLogRepository.findAllOrderByIdAsc()
assert logs[0].previousHash == "GENESIS"
for i in 1..logs.size()-1:
    assert logs[i].previousHash == logs[i-1].hash
```

### パターン2.4: 処理失敗時も監査ログが記録されること

```pseudo
// Given: 残高不足の口座
account = createAccount(balance = Decimal("100"))

// When: 残高不足で振込が失敗する
try:
    transferService.transfer(account.id, toAccount.id, Decimal("500"))
catch InsufficientBalanceException:
    pass

// Then: 失敗の監査ログが記録されていること
log = auditLogRepository.findLatest()
assert log.result == "FAILURE"
assert log.action == "TRANSFER"
assert "InsufficientBalance" in log.details or "残高不足" in log.details
```

---

## §3 並行アクセステスト

### パターン3.1: 楽観ロック競合の検出

```pseudo
// Given: 有効な口座（残高 1000、version 0）
account = createAccount(balance = Decimal("1000"))

// When: 2つのトランザクションが同時に更新を試みる
// トランザクション1: 口座を読み取る
account1 = accountRepository.findById(account.id)

// トランザクション2: 先にコミットする
account2 = accountRepository.findById(account.id)
account2.debit(Decimal("200"))
accountRepository.save(account2)  // version 0 → 1

// トランザクション1: 後からコミットを試みる
account1.debit(Decimal("300"))

// Then: 楽観ロック例外がスローされること
assertThrows OptimisticLockException:
    accountRepository.save(account1)  // version 0 != 1 → 衝突
```

### パターン3.2: バージョンインクリメントの確認

```pseudo
// Given: 有効な口座（version 0）
account = createAccount(balance = Decimal("1000"))
assert account.version == 0

// When: 更新操作を実行する
accountService.updateBalance(account.id, Decimal("500"))

// Then: バージョンがインクリメントされていること
updated = accountRepository.findById(account.id)
assert updated.version == 1

// When: さらに更新する
accountService.updateBalance(account.id, Decimal("-200"))

// Then: バージョンが再びインクリメントされること
updated2 = accountRepository.findById(account.id)
assert updated2.version == 2
```

### パターン3.3: 並行スレッドによる同時更新テスト

```pseudo
// Given: 有効な口座（残高 10000）
account = createAccount(balance = Decimal("10000"))

// When: 10スレッドが同時に100ずつ引き出す
results = runConcurrently(threadCount = 10):
    try:
        accountService.withdraw(account.id, Decimal("100"))
        return "SUCCESS"
    catch OptimisticLockException:
        return "CONFLICT"

// Then: 成功した操作の数だけ残高が減っていること
successCount = results.count("SUCCESS")
finalBalance = accountRepository.findById(account.id).balance
assert finalBalance == Decimal("10000") - (Decimal("100") * successCount)

// And: 一部はコンフリクトで失敗していること（競合が発生する場合）
// ※ リトライ実装がある場合は全て成功する可能性もある
assert successCount >= 1
```

---

## §4 冪等性テスト

### パターン4.1: 同一冪等キーによる重複実行の防止

```pseudo
// Given: 有効な振込リクエストと冪等キー
idempotencyKey = "txn-uuid-001"
request = TransferRequest(from = account1.id, to = account2.id, amount = Decimal("500"))

// When: 同じ冪等キーで2回実行する
result1 = transferService.execute(request, idempotencyKey)
result2 = transferService.execute(request, idempotencyKey)

// Then: 2回目は重複として処理され、1回分のみ反映されること
assert result1.status == "CREATED"
assert result2.status == "DUPLICATE"
assert result1.transferId == result2.transferId

// And: 残高は1回分のみ変動していること
assert accountRepository.findById(account1.id).balance == Decimal("500")   // 1000 - 500
assert accountRepository.findById(account2.id).balance == Decimal("1500")  // 1000 + 500
```

### パターン4.2: 異なる冪等キーは別々に処理されること

```pseudo
// Given: 有効な口座（残高 1000）
account = createAccount(balance = Decimal("1000"))

// When: 異なる冪等キーで同じ操作を実行する
result1 = transferService.execute(request, idempotencyKey = "key-001")
result2 = transferService.execute(request, idempotencyKey = "key-002")

// Then: それぞれ独立した操作として処理されること
assert result1.status == "CREATED"
assert result2.status == "CREATED"
assert result1.transferId != result2.transferId
```

### パターン4.3: 冪等キーの有効期限テスト

```pseudo
// Given: 冪等キーで処理を実行する
idempotencyKey = "txn-uuid-expired"
result1 = transferService.execute(request, idempotencyKey)
assert result1.status == "CREATED"

// When: 有効期限（24時間）を過ぎてから同じキーで実行する
advanceTimeBy(Duration.ofHours(25))
cleanupService.cleanupExpiredKeys()

result2 = transferService.execute(request, idempotencyKey)

// Then: 新しい操作として処理されること
assert result2.status == "CREATED"
assert result2.transferId != result1.transferId
```

---

## §5 高精度小数型テスト

### パターン5.1: 文字列コンストラクタの使用確認

```pseudo
// Given/When: 金額を生成する

// Then: 文字列コンストラクタで生成した値が正確であること
amount = Decimal("0.1")
assert amount.toString() == "0.1"

// And: 浮動小数点コンストラクタでは精度が失われることを確認
// ※ このテストは「なぜ文字列コンストラクタが必要か」を示すドキュメント的テスト
badAmount = Decimal(0.1)  // double → Decimal
assert badAmount.toString() != "0.1"  // "0.1000000000000000055511151231257827021181583404541015625" 等
```

### パターン5.2: 加算・減算における丸め誤差の回避

```pseudo
// Given: 典型的な丸め誤差パターン
a = Decimal("0.1")
b = Decimal("0.2")

// When: 加算する
result = a + b

// Then: 正確に 0.3 であること（浮動小数点の 0.1+0.2 != 0.3 問題が発生しない）
assert result == Decimal("0.3")
assert result.compareTo(Decimal("0.3")) == 0

// And: 金融計算の典型例
price    = Decimal("19.99")
tax      = Decimal("1.60")
total    = price + tax
assert total == Decimal("21.59")
```

### パターン5.3: 丸めモードの明示的指定

```pseudo
// Given: 除算で無限小数が発生するケース
amount = Decimal("10.00")
divisor = Decimal("3")

// When: 丸めモードを指定して除算する
result_half_up   = amount.divide(divisor, scale = 2, roundingMode = HALF_UP)
result_half_even = amount.divide(divisor, scale = 2, roundingMode = HALF_EVEN)
result_down      = amount.divide(divisor, scale = 2, roundingMode = DOWN)

// Then: 各丸めモードで期待通りの結果であること
assert result_half_up   == Decimal("3.33")
assert result_half_even == Decimal("3.33")
assert result_down      == Decimal("3.33")

// And: 丸めモード未指定では例外がスローされること
assertThrows ArithmeticException:
    amount.divide(divisor, scale = 2)  // 丸めモード未指定 → 例外
```

### パターン5.4: 境界値テスト（ゼロ・負数・上限）

```pseudo
// Given: 金額バリデーション付きのサービス

// Then: ゼロは許可されないこと
assertThrows InvalidAmountException:
    transferService.transfer(from, to, Decimal("0"))

// And: 負数は許可されないこと
assertThrows InvalidAmountException:
    transferService.transfer(from, to, Decimal("-100"))

// And: 上限を超える金額は許可されないこと
assertThrows InvalidAmountException:
    transferService.transfer(from, to, Decimal("1000000001"))  // 上限: 10億

// And: 上限ちょうどは許可されること
result = transferService.transfer(from, to, Decimal("1000000000"))
assert result.status == "SUCCESS"

// And: 小数点以下の精度が保たれること
result = transferService.transfer(from, to, Decimal("0.0001"))
assert result.status == "SUCCESS"
```

### パターン5.5: 通貨コード検証

```pseudo
// Given: 通貨コード付きの金額オブジェクト
jpyAmount = Money(amount = Decimal("1000"), currency = "JPY")
usdAmount = Money(amount = Decimal("10.50"), currency = "USD")

// Then: 異なる通貨間の演算は禁止されること
assertThrows CurrencyMismatchException:
    jpyAmount + usdAmount

// And: 同一通貨の演算は許可されること
result = jpyAmount + Money(Decimal("500"), "JPY")
assert result.amount   == Decimal("1500")
assert result.currency == "JPY"

// And: JPYは小数点以下0桁であること
assert jpyAmount.scale() == 0

// And: USDは小数点以下2桁であること
assert usdAmount.scale() == 2
```
