# 銀行テストパターン集

## 1. トランザクションロールバックテスト

### 1.1 例外時のロールバック検証

```java
@SpringBootTest
@Transactional
class TransferServiceTest {

    @Autowired
    private TransferService transferService;

    @Autowired
    private AccountRepository accountRepository;

    @Test
    void should_rollback_when_insufficient_balance() {
        // Given: 残高1000円の口座
        Account from = accountRepository.save(
            Account.create("FROM-001", new BigDecimal("1000")));
        Account to = accountRepository.save(
            Account.create("TO-001", BigDecimal.ZERO));

        // When: 残高を超える振込を試みる
        assertThatThrownBy(() ->
            transferService.transfer(from.getId(), to.getId(),
                new BigDecimal("2000"))
        ).isInstanceOf(InsufficientBalanceException.class);

        // Then: 両口座の残高が変更されていない
        Account fromAfter = accountRepository.findById(from.getId()).orElseThrow();
        Account toAfter = accountRepository.findById(to.getId()).orElseThrow();
        assertThat(fromAfter.getBalance()).isEqualByComparingTo(new BigDecimal("1000"));
        assertThat(toAfter.getBalance()).isEqualByComparingTo(BigDecimal.ZERO);
    }
}
```

### 1.2 部分更新の検証

```java
@Test
void should_rollback_all_changes_when_second_operation_fails() {
    // Given: 複数の操作を含むトランザクション
    // When: 2番目の操作で失敗
    // Then: 1番目の操作もロールバックされている
}
```

## 2. 監査ログ検証テスト

```java
@SpringBootTest
class AuditLogTest {

    @Autowired
    private AccountService accountService;

    @Autowired
    private AuditLogRepository auditLogRepository;

    @Test
    void should_create_audit_log_on_transfer() {
        // Given
        Long fromId = createTestAccount("1000");
        Long toId = createTestAccount("0");

        // When
        accountService.transfer(fromId, toId, new BigDecimal("500"));

        // Then: 監査ログが記録されている
        List<AuditLog> logs = auditLogRepository
            .findByResourceTypeAndResourceId("Account", fromId.toString());
        assertThat(logs).isNotEmpty();
        assertThat(logs.get(0).getEventType()).isEqualTo("TRANSFER");
        assertThat(logs.get(0).getResult()).isEqualTo("SUCCESS");
    }

    @Test
    void should_create_audit_log_even_on_failure() {
        // Given: 残高不足の口座
        Long fromId = createTestAccount("100");
        Long toId = createTestAccount("0");

        // When: 振込が失敗する
        assertThatThrownBy(() ->
            accountService.transfer(fromId, toId, new BigDecimal("500"))
        );

        // Then: 失敗の監査ログも記録されている
        List<AuditLog> logs = auditLogRepository
            .findByEventType("TRANSFER");
        assertThat(logs).anyMatch(log ->
            "FAILURE".equals(log.getResult()));
    }

    @Test
    void should_not_contain_pii_in_audit_log() {
        // Given & When: 操作を実行
        accountService.transfer(fromId, toId, amount);

        // Then: 監査ログにPIIが含まれていない
        List<AuditLog> logs = auditLogRepository.findAll();
        for (AuditLog log : logs) {
            assertThat(log.getDetails())
                .doesNotContainPattern("\\d{7,}") // 口座番号フル
                .doesNotContain("@") // メールアドレス
                .doesNotContainPattern("\\d{3}-\\d{4}-\\d{4}"); // 電話番号
        }
    }
}
```

## 3. 並行アクセステスト（楽観ロック）

```java
@SpringBootTest
class OptimisticLockTest {

    @Autowired
    private AccountService accountService;

    @Autowired
    private AccountRepository accountRepository;

    @Test
    void should_throw_optimistic_lock_exception_on_concurrent_update() throws Exception {
        // Given: 口座を作成
        Account account = accountRepository.save(
            Account.create("ACC-001", new BigDecimal("10000")));

        // When: 2つのスレッドが同時に更新
        ExecutorService executor = Executors.newFixedThreadPool(2);
        CountDownLatch latch = new CountDownLatch(1);

        Future<?> future1 = executor.submit(() -> {
            latch.await();
            accountService.withdraw(account.getId(), new BigDecimal("3000"));
            return null;
        });

        Future<?> future2 = executor.submit(() -> {
            latch.await();
            accountService.withdraw(account.getId(), new BigDecimal("4000"));
            return null;
        });

        latch.countDown(); // 同時に開始

        // Then: 少なくとも1つがOptimisticLockingFailureExceptionをスロー
        List<Exception> exceptions = new ArrayList<>();
        try { future1.get(); } catch (ExecutionException e) { exceptions.add(e); }
        try { future2.get(); } catch (ExecutionException e) { exceptions.add(e); }

        assertThat(exceptions).hasSizeGreaterThanOrEqualTo(1);
        assertThat(exceptions.get(0).getCause())
            .isInstanceOf(OptimisticLockingFailureException.class);

        executor.shutdown();
    }

    @Test
    void should_maintain_balance_consistency_under_concurrent_access() throws Exception {
        // Given: 残高10000円
        Account account = accountRepository.save(
            Account.create("ACC-001", new BigDecimal("10000")));

        // When: 10スレッドがそれぞれ100円引き出す（リトライ付き）
        ExecutorService executor = Executors.newFixedThreadPool(10);
        List<Future<?>> futures = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            futures.add(executor.submit(() -> {
                accountService.withdrawWithRetry(account.getId(), new BigDecimal("100"));
                return null;
            }));
        }
        for (Future<?> f : futures) { f.get(); }

        // Then: 残高は正確に9000円
        Account result = accountRepository.findById(account.getId()).orElseThrow();
        assertThat(result.getBalance()).isEqualByComparingTo(new BigDecimal("9000"));

        executor.shutdown();
    }
}
```

## 4. 冪等性テスト

```java
@SpringBootTest
class IdempotencyTest {

    @Autowired
    private TransferService transferService;

    @Autowired
    private AccountRepository accountRepository;

    @Test
    void should_process_only_once_with_same_idempotency_key() {
        // Given
        String idempotencyKey = UUID.randomUUID().toString();
        Long fromId = createTestAccount("10000");
        Long toId = createTestAccount("0");
        BigDecimal amount = new BigDecimal("1000");

        // When: 同じ冪等キーで2回実行
        TransferResult result1 = transferService.transfer(
            fromId, toId, amount, idempotencyKey);
        TransferResult result2 = transferService.transfer(
            fromId, toId, amount, idempotencyKey);

        // Then: 1回分だけ処理されている
        assertThat(result1.status()).isEqualTo(TransferStatus.COMPLETED);
        assertThat(result2.status()).isEqualTo(TransferStatus.DUPLICATE);

        Account from = accountRepository.findById(fromId).orElseThrow();
        assertThat(from.getBalance()).isEqualByComparingTo(new BigDecimal("9000"));
    }

    @Test
    void should_process_separately_with_different_idempotency_keys() {
        // Given
        Long fromId = createTestAccount("10000");
        Long toId = createTestAccount("0");
        BigDecimal amount = new BigDecimal("1000");

        // When: 異なる冪等キーで2回実行
        transferService.transfer(fromId, toId, amount, UUID.randomUUID().toString());
        transferService.transfer(fromId, toId, amount, UUID.randomUUID().toString());

        // Then: 2回分処理されている
        Account from = accountRepository.findById(fromId).orElseThrow();
        assertThat(from.getBalance()).isEqualByComparingTo(new BigDecimal("8000"));
    }
}
```

## 5. BigDecimalアサーションパターン

```java
class BigDecimalAssertionPatterns {

    @Test
    void should_use_compareTo_not_equals() {
        BigDecimal a = new BigDecimal("1000.00");
        BigDecimal b = new BigDecimal("1000");

        // NG: scaleが異なるためfalse
        // assertThat(a).isEqualTo(b);

        // OK: 数値的に等しいか比較
        assertThat(a).isEqualByComparingTo(b);
    }

    @Test
    void should_specify_rounding_mode() {
        BigDecimal price = new BigDecimal("100");
        BigDecimal taxRate = new BigDecimal("0.08");

        // NG: ArithmeticExceptionの可能性
        // BigDecimal tax = price.multiply(taxRate).divide(BigDecimal.ONE);

        // OK: 丸めモードを明示
        BigDecimal tax = price.multiply(taxRate)
            .setScale(0, RoundingMode.HALF_UP);

        assertThat(tax).isEqualByComparingTo(new BigDecimal("8"));
    }

    @Test
    void should_use_string_constructor() {
        // NG: 浮動小数点の精度問題
        // BigDecimal bad = new BigDecimal(0.1);

        // OK: 文字列コンストラクタ
        BigDecimal good = new BigDecimal("0.1");
        assertThat(good.toString()).isEqualTo("0.1");
    }

    @Test
    void should_verify_negative_amount_rejected() {
        // 金額が負数の場合はバリデーションエラー
        assertThatThrownBy(() ->
            accountService.deposit(accountId, new BigDecimal("-100"))
        ).isInstanceOf(InvalidAmountException.class);
    }

    @Test
    void should_verify_zero_amount_rejected() {
        // 金額が0の場合もバリデーションエラー
        assertThatThrownBy(() ->
            accountService.deposit(accountId, BigDecimal.ZERO)
        ).isInstanceOf(InvalidAmountException.class);
    }
}
```

## 6. テストデータビルダーパターン

```java
public class TestAccountBuilder {

    private String accountNumber = "ACC-" + UUID.randomUUID().toString().substring(0, 8);
    private BigDecimal balance = BigDecimal.ZERO;
    private String holderName = "テスト太郎";
    private AccountStatus status = AccountStatus.ACTIVE;

    public static TestAccountBuilder anAccount() {
        return new TestAccountBuilder();
    }

    public TestAccountBuilder withBalance(String amount) {
        this.balance = new BigDecimal(amount);
        return this;
    }

    public TestAccountBuilder withStatus(AccountStatus status) {
        this.status = status;
        return this;
    }

    public Account build() {
        return new Account(accountNumber, balance, holderName, status);
    }
}

// 使用例
Account account = anAccount().withBalance("10000").build();
```
