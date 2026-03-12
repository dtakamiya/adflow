# 排他制御パターン — ミッションクリティカルシステム向けリファレンス

## 1. 楽観ロック（Optimistic Locking）

### 1.1 基本パターン — バージョンフィールド（楽観ロック）

```pseudo
// スキーマ定義: エンティティ "Account"
class Account:

    id: Long                           // 主キー（自動採番）
    version: Long                      // バージョンフィールド（楽観ロック）
    balance: Decimal(precision=19, scale=4)  // NOT NULL、高精度小数型

    // getter, constructor（setterなし）
```

### 1.2 楽観ロックの例外ハンドリング

```pseudo
class AccountService:

    MAX_RETRY = 3

    // リトライ設定: 楽観ロック例外発生時、最大3回、初期遅延100ms、指数バックオフ×2
    // トランザクション宣言
    function updateBalance(accountId: Long, amount: Decimal):
        account = accountRepository.findById(accountId)
            ?? throw AccountNotFoundException(accountId)
        account.addBalance(amount)
        // 保存時にバージョンが一致しなければ楽観ロック例外がスローされる
        accountRepository.save(account)
```

### 1.3 使い分け指針
- **楽観ロック推奨**: 競合頻度が低い操作（口座情報更新、顧客情報変更）
- **楽観ロック非推奨**: 高頻度な競合が予想される操作（人気商品の在庫減算）

## 2. 悲観ロック（Pessimistic Locking）

### 2.1 SELECT FOR UPDATE

```pseudo
// リポジトリインターフェース: Account エンティティ
interface AccountRepository:

    // クエリアノテーション: 排他ロック（PESSIMISTIC_WRITE）
    // SELECT a FROM Account a WHERE a.id = :id
    function findByIdForUpdate(id: Long): Optional<Account>

    // クエリアノテーション: 排他ロック（PESSIMISTIC_WRITE）
    // SELECT a FROM Account a WHERE a.id IN :ids ORDER BY a.id
    function findByIdsForUpdate(ids: List<Long>): List<Account>
```

### 2.2 タイムアウト設定

```pseudo
// クエリアノテーション: 排他ロック（PESSIMISTIC_WRITE）
// クエリヒント: ロックタイムアウト 3000ms
// SELECT a FROM Account a WHERE a.id = :id
function findByIdForUpdateWithTimeout(id: Long): Optional<Account>
```

### 2.3 使い分け指針
- **悲観ロック推奨**: 残高更新、振込処理（競合時のリトライコストが高い操作）
- **悲観ロック非推奨**: 参照が主な操作、長時間保持が必要な処理

## 3. デッドロック防止

### 3.1 ロック順序の統一

```pseudo
class TransferService:

    // トランザクション宣言
    function transfer(fromId: Long, toId: Long, amount: Decimal):
        // 常にID昇順でロックを取得してデッドロックを防止
        firstId  = min(fromId, toId)
        secondId = max(fromId, toId)

        first  = accountRepository.findByIdForUpdate(firstId)  ?? throw NotFound
        second = accountRepository.findByIdForUpdate(secondId) ?? throw NotFound

        from = (fromId == firstId) ? first : second
        to   = (fromId == firstId) ? second : first

        from.debit(amount)
        to.credit(amount)
```

### 3.2 デッドロック検出と回復

```pseudo
class TransferService:

    // リトライ設定: デッドロック例外発生時、最大3回、初期遅延200ms、指数バックオフ×2（ランダム揺らぎあり）
    // トランザクション宣言
    function transfer(fromId: Long, toId: Long, amount: Decimal):
        // ロック順序統一 + デッドロック時リトライ
```

## 4. 分散ロック

### 4.1 Redis分散ロック

```pseudo
class DistributedLockService:

    redisClient: RedisClient

    function tryLock(key: String, value: String, timeout: Duration): Boolean:
        // SET key value NX PX timeout（存在しない場合のみセット）
        return redisClient.setIfAbsent(key, value, timeout)

    function unlock(key: String, expectedValue: String):
        // Luaスクリプトでアトミックに解放
        script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        """
        redisClient.executeScript(script, keys=[key], args=[expectedValue])
```

## 5. パターン選択ガイド

| 状況 | 推奨パターン | 理由 |
|------|-----------|------|
| 口座残高更新 | 悲観ロック | 競合時の整合性が最重要 |
| 顧客情報更新 | 楽観ロック | 競合頻度が低い |
| 振込（2口座） | 悲観ロック + ロック順序統一 | デッドロック防止 |
| バッチ処理 | 楽観ロック + リトライ | 長時間ロック保持を避ける |
| マイクロサービス間 | 分散ロック | プロセス間の排他制御 |
