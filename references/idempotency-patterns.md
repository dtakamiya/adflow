# 冪等性パターン — ミッションクリティカルシステム向けリファレンス

## 1. なぜ冪等性が重要か

ミッションクリティカルシステムでは以下の理由から冪等性が必須:
- ネットワーク障害によるリトライ
- タイムアウト後の再送信
- メッセージブローカーの重複配信
- ユーザーのダブルクリック

**金銭的影響**: 冪等性がなければ、二重振込・二重引落が発生する

## 2. 冪等キーパターン

### 2.1 基本実装

```pseudo
// スキーマ定義: テーブル名 "idempotency_keys"
class IdempotencyKey:

    key: String            // 主キー、UUID（最大36文字）
    resourceType: String   // NOT NULL, 最大50文字
    responseBody: Text     // 初回レスポンスのキャッシュ
    responseStatus: Int    // NOT NULL
    createdAt: Instant     // NOT NULL
    expiresAt: Instant     // NOT NULL
```

### 2.2 冪等性フィルター

```pseudo
// リクエストフィルター（書き込みメソッドのみ適用）
class IdempotencyFilter:

    repository: IdempotencyKeyRepository
    IDEMPOTENCY_HEADER = "Idempotency-Key"

    function doFilter(request, response, chain):
        if not isWriteMethod(request.method):
            chain.doFilter(request, response)
            return

        idempotencyKey = request.getHeader(IDEMPOTENCY_HEADER)
        if idempotencyKey == null:
            response.sendError(400, "Idempotency-Key header is required")
            return

        existing = repository.findByKeyAndResourceType(
            idempotencyKey, extractResourceType(request))

        if existing is present:
            // 以前のレスポンスを返す
            cached = existing.get()
            response.setStatus(cached.responseStatus)
            response.write(cached.responseBody)
            return

        // リクエストを処理し、レスポンスをキャプチャ
        wrappedResponse = ResponseWrapper(response)
        chain.doFilter(request, wrappedResponse)

        // レスポンスを保存
        repository.save(IdempotencyKey(
            key          = idempotencyKey,
            resourceType = extractResourceType(request),
            responseBody = wrappedResponse.getCapturedBody(),
            responseStatus = wrappedResponse.status,
            createdAt    = Instant.now(),
            expiresAt    = Instant.now() + Duration.ofHours(24)
        ))

        wrappedResponse.flushToClient()

    private function isWriteMethod(method: String): Boolean:
        return method in ["POST", "PUT", "PATCH"]
```

## 3. データベースレベルの重複防止

### 3.1 ユニーク制約

```sql
-- 振込テーブル: 同一冪等キーの重複防止
ALTER TABLE transfers
ADD CONSTRAINT uk_transfers_idempotency_key
UNIQUE (idempotency_key);
```

### 3.2 重複時は既存結果を返す

```pseudo
// リポジトリ実装
class TransferRepositoryImpl:

    // トランザクション宣言
    function executeTransfer(transfer: Transfer): TransferResult:
        try:
            transferRepository.save(transfer)
            return TransferResult.created(transfer)
        catch データ整合性違反例外 as e:
            // 重複 — 既存の結果を返す
            existing = transferRepository
                .findByIdempotencyKey(transfer.idempotencyKey)
                ?? throw NotFound
            return TransferResult.duplicate(existing)
```

## 4. メッセージングの冪等性

### 4.1 消費者側の重複排除

```pseudo
// メッセージリスナー: トピック "payment-events"
function handlePaymentEvent(event: PaymentEvent):
    // メッセージIDで重複チェック
    if processedMessageRepository.existsByMessageId(event.messageId):
        log.info("Duplicate message ignored: {}", event.messageId)
        return

    processPayment(event)

    processedMessageRepository.save(
        ProcessedMessage(event.messageId, Instant.now())
    )
```

## 5. 冪等キーの有効期限管理

```pseudo
// スケジュール実行: 毎日午前2時
function cleanupExpiredKeys():
    deleted = idempotencyKeyRepository.deleteByExpiresAtBefore(Instant.now())
    log.info("Cleaned up {} expired idempotency keys", deleted)
```

## 6. HTTP メソッドと冪等性

| メソッド | 冪等 | 安全 | 対策 |
|---------|------|------|------|
| GET | Yes | Yes | キャッシュ |
| PUT | Yes | No | 全体置換で自然に冪等 |
| DELETE | Yes | No | 存在しない場合も204を返す |
| POST | **No** | No | **冪等キー必須** |
| PATCH | **No** | No | **冪等キー必須** |
