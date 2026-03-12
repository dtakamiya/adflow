# ドメイン品質テストパターン集

## 1. トランザクションロールバックテスト

### 1.1 例外時のロールバック検証

```
// 統合テスト（トランザクション付き）
test should_rollback_when_insufficient_balance:
    // Given: 残高1000円の口座
    from = repository.save(Account.create("FROM-001", Decimal("1000")))
    to   = repository.save(Account.create("TO-001",   Decimal("0")))

    // When: 残高を超える振込を試みる
    assert throws InsufficientBalanceException:
        transferService.transfer(from.id, to.id, Decimal("2000"))

    // Then: 両口座の残高が変更されていない
    fromAfter = repository.findById(from.id)
    toAfter   = repository.findById(to.id)
    assert fromAfter.balance == Decimal("1000")  // 値比較
    assert toAfter.balance   == Decimal("0")
```

### 1.2 部分更新の検証

```
test should_rollback_all_changes_when_second_operation_fails:
    // Given: 複数の操作を含むトランザクション
    // When: 2番目の操作で失敗
    // Then: 1番目の操作もロールバックされている
```

## 2. 監査ログ検証テスト

```
// 統合テスト
test should_create_audit_log_on_transfer:
    // Given
    fromId = createTestAccount("1000")
    toId   = createTestAccount("0")

    // When
    accountService.transfer(fromId, toId, Decimal("500"))

    // Then: 監査ログが記録されている
    logs = auditLogRepository.findByResourceTypeAndResourceId("Account", fromId)
    assert logs is not empty
    assert logs[0].eventType == "TRANSFER"
    assert logs[0].result    == "SUCCESS"

test should_create_audit_log_even_on_failure:
    // Given: 残高不足の口座
    fromId = createTestAccount("100")
    toId   = createTestAccount("0")

    // When: 振込が失敗する
    assert throws:
        accountService.transfer(fromId, toId, Decimal("500"))

    // Then: 失敗の監査ログも記録されている
    logs = auditLogRepository.findByEventType("TRANSFER")
    assert any(log.result == "FAILURE" for log in logs)

test should_not_contain_pii_in_audit_log:
    // Given & When: 操作を実行
    accountService.transfer(fromId, toId, amount)

    // Then: 監査ログにPIIが含まれていない
    logs = auditLogRepository.findAll()
    for log in logs:
        assert not matches(log.details, r"\d{7,}")          // 口座番号フル
        assert "@" not in log.details                        // メールアドレス
        assert not matches(log.details, r"\d{3}-\d{4}-\d{4}") // 電話番号
```

## 3. 並行アクセステスト（楽観ロック）

```
test should_throw_optimistic_lock_exception_on_concurrent_update:
    // Given: 口座を作成
    account = repository.save(Account.create("ACC-001", Decimal("10000")))

    // When: 2つのスレッド/ゴルーチン/タスクが同時に更新
    exceptions = []
    run_concurrently([
        () -> accountService.withdraw(account.id, Decimal("3000")),
        () -> accountService.withdraw(account.id, Decimal("4000")),
    ], on_error: exceptions.append)

    // Then: 少なくとも1つが楽観ロック例外をスロー
    assert len(exceptions) >= 1
    assert exceptions[0] is 楽観ロック例外

test should_maintain_balance_consistency_under_concurrent_access:
    // Given: 残高10000円
    account = repository.save(Account.create("ACC-001", Decimal("10000")))

    // When: 10並列でそれぞれ100円引き出す（リトライ付き）
    run_concurrently(10, () ->
        accountService.withdrawWithRetry(account.id, Decimal("100"))
    )

    // Then: 残高は正確に9000円
    result = repository.findById(account.id)
    assert result.balance == Decimal("9000")  // 値比較
```

## 4. 冪等性テスト

```
test should_process_only_once_with_same_idempotency_key:
    // Given
    idempotencyKey = generateUUID()
    fromId = createTestAccount("10000")
    toId   = createTestAccount("0")
    amount = Decimal("1000")

    // When: 同じ冪等キーで2回実行
    result1 = transferService.transfer(fromId, toId, amount, idempotencyKey)
    result2 = transferService.transfer(fromId, toId, amount, idempotencyKey)

    // Then: 1回分だけ処理されている
    assert result1.status == COMPLETED
    assert result2.status == DUPLICATE

    from = repository.findById(fromId)
    assert from.balance == Decimal("9000")  // 値比較

test should_process_separately_with_different_idempotency_keys:
    // Given
    fromId = createTestAccount("10000")
    toId   = createTestAccount("0")
    amount = Decimal("1000")

    // When: 異なる冪等キーで2回実行
    transferService.transfer(fromId, toId, amount, generateUUID())
    transferService.transfer(fromId, toId, amount, generateUUID())

    // Then: 2回分処理されている
    from = repository.findById(fromId)
    assert from.balance == Decimal("8000")  // 値比較
```

## 5. 高精度小数型アサーションパターン

```
test should_use_value_comparison_not_identity:
    a = Decimal("1000.00")
    b = Decimal("1000")

    // NG: スケールや内部表現が異なるため同一性比較は誤り
    // assert a is b

    // OK: 数値的に等しいか値比較
    assert a == b  // 値比較

test should_specify_rounding_mode:
    price   = Decimal("100")
    taxRate = Decimal("0.08")

    // NG: 丸めモードを指定しないと精度エラーの可能性
    // tax = price * taxRate

    // OK: 丸めモードを明示して計算
    tax = (price * taxRate).round(scale=0, mode=丸めモード.HALF_UP)

    assert tax == Decimal("8")  // 値比較

test should_use_string_literal_constructor:
    // NG: 浮動小数点リテラルから生成すると精度が失われる
    // bad = Decimal(0.1)  // 0.1000000000000000055511151231257827021181583404541015625...

    // OK: 文字列リテラルから生成
    good = Decimal("0.1")
    assert str(good) == "0.1"

test should_verify_negative_amount_rejected:
    // 金額が負数の場合はバリデーションエラー
    assert throws InvalidAmountException:
        accountService.deposit(accountId, Decimal("-100"))

test should_verify_zero_amount_rejected:
    // 金額が0の場合もバリデーションエラー
    assert throws InvalidAmountException:
        accountService.deposit(accountId, Decimal("0"))
```

## 6. テストデータビルダーパターン

```
// テストデータ生成用ビルダー
builder AccountBuilder:
    accountNumber = "ACC-" + generateUUID()[0:8]
    balance       = Decimal("0")
    holderName    = "テスト太郎"
    status        = AccountStatus.ACTIVE

    withBalance(amount: string) -> self:
        self.balance = Decimal(amount)
        return self

    withStatus(s: AccountStatus) -> self:
        self.status = s
        return self

    build() -> Account:
        return Account(accountNumber, balance, holderName, status)

// 使用例
account = AccountBuilder().withBalance("10000").build()
```
