# 排他制御パターン — 銀行システム向けリファレンス

## 1. 楽観ロック（Optimistic Locking）

### 1.1 基本パターン — @Version

```java
@Entity
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version
    private Long version;

    @Column(nullable = false, precision = 19, scale = 4)
    private BigDecimal balance;

    // getter, constructor（setterなし）
}
```

### 1.2 楽観ロックの例外ハンドリング

```java
@Service
public class AccountService {

    private static final int MAX_RETRY = 3;

    @Retryable(
        value = OptimisticLockingFailureException.class,
        maxAttempts = MAX_RETRY,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void updateBalance(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        account.addBalance(amount);
        // save時にversionが一致しなければOptimisticLockingFailureExceptionがスローされる
        accountRepository.save(account);
    }
}
```

### 1.3 使い分け指針
- **楽観ロック推奨**: 競合頻度が低い操作（口座情報更新、顧客情報変更）
- **楽観ロック非推奨**: 高頻度な競合が予想される操作（人気商品の在庫減算）

## 2. 悲観ロック（Pessimistic Locking）

### 2.1 SELECT FOR UPDATE

```java
public interface AccountRepository extends JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id IN :ids ORDER BY a.id")
    List<Account> findByIdsForUpdate(@Param("ids") List<Long> ids);
}
```

### 2.2 タイムアウト設定

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")
})
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdForUpdateWithTimeout(@Param("id") Long id);
```

### 2.3 使い分け指針
- **悲観ロック推奨**: 残高更新、振込処理（競合時のリトライコストが高い操作）
- **悲観ロック非推奨**: 参照が主な操作、長時間保持が必要な処理

## 3. デッドロック防止

### 3.1 ロック順序の統一

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // 常にID昇順でロックを取得してデッドロックを防止
        Long firstId = Math.min(fromId, toId);
        Long secondId = Math.max(fromId, toId);

        Account first = accountRepository.findByIdForUpdate(firstId)
            .orElseThrow();
        Account second = accountRepository.findByIdForUpdate(secondId)
            .orElseThrow();

        Account from = fromId.equals(firstId) ? first : second;
        Account to = fromId.equals(firstId) ? second : first;

        from.debit(amount);
        to.credit(amount);
    }
}
```

### 3.2 デッドロック検出と回復

```java
@Service
public class TransferService {

    @Retryable(
        value = {DeadlockLoserDataAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 200, multiplier = 2, random = true)
    )
    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // ロック順序統一 + デッドロック時リトライ
    }
}
```

## 4. 分散ロック

### 4.1 Redis分散ロック

```java
@Service
public class DistributedLockService {

    private final RedisTemplate<String, String> redisTemplate;

    public boolean tryLock(String key, String value, Duration timeout) {
        return Boolean.TRUE.equals(
            redisTemplate.opsForValue()
                .setIfAbsent(key, value, timeout)
        );
    }

    public void unlock(String key, String expectedValue) {
        // Luaスクリプトでアトミックに解放
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            List.of(key), expectedValue
        );
    }
}
```

## 5. パターン選択ガイド

| 状況 | 推奨パターン | 理由 |
|------|-----------|------|
| 口座残高更新 | 悲観ロック | 競合時の整合性が最重要 |
| 顧客情報更新 | 楽観ロック | 競合頻度が低い |
| 振込（2口座） | 悲観ロック + ロック順序統一 | デッドロック防止 |
| バッチ処理 | 楽観ロック + リトライ | 長時間ロック保持を避ける |
| マイクロサービス間 | 分散ロック | プロセス間の排他制御 |
