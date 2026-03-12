# 冪等性パターン — 銀行システム向けリファレンス

## 1. なぜ冪等性が重要か

銀行システムでは以下の理由から冪等性が必須:
- ネットワーク障害によるリトライ
- タイムアウト後の再送信
- メッセージブローカーの重複配信
- ユーザーのダブルクリック

**金銭的影響**: 冪等性がなければ、二重振込・二重引落が発生する

## 2. 冪等キーパターン

### 2.1 基本実装

```java
@Entity
@Table(name = "idempotency_keys")
public class IdempotencyKey {

    @Id
    @Column(length = 36)
    private String key; // UUID

    @Column(nullable = false, length = 50)
    private String resourceType;

    @Column(columnDefinition = "TEXT")
    private String responseBody; // 初回レスポンスのキャッシュ

    @Column(nullable = false)
    private int responseStatus;

    @Column(nullable = false)
    private Instant createdAt;

    @Column(nullable = false)
    private Instant expiresAt;
}
```

### 2.2 冪等性フィルター

```java
@Component
public class IdempotencyFilter extends OncePerRequestFilter {

    private final IdempotencyKeyRepository repository;
    private static final String IDEMPOTENCY_HEADER = "Idempotency-Key";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        if (!isWriteMethod(request.getMethod())) {
            chain.doFilter(request, response);
            return;
        }

        String idempotencyKey = request.getHeader(IDEMPOTENCY_HEADER);
        if (idempotencyKey == null) {
            response.sendError(400, "Idempotency-Key header is required");
            return;
        }

        Optional<IdempotencyKey> existing = repository.findByKeyAndResourceType(
            idempotencyKey, extractResourceType(request));

        if (existing.isPresent()) {
            // 以前のレスポンスを返す
            IdempotencyKey cached = existing.get();
            response.setStatus(cached.getResponseStatus());
            response.getWriter().write(cached.getResponseBody());
            return;
        }

        // リクエストを処理し、レスポンスをキャプチャ
        ContentCachingResponseWrapper wrappedResponse =
            new ContentCachingResponseWrapper(response);
        chain.doFilter(request, wrappedResponse);

        // レスポンスを保存
        repository.save(new IdempotencyKey(
            idempotencyKey,
            extractResourceType(request),
            new String(wrappedResponse.getContentAsByteArray()),
            wrappedResponse.getStatus(),
            Instant.now(),
            Instant.now().plus(Duration.ofHours(24))
        ));

        wrappedResponse.copyBodyToResponse();
    }

    private boolean isWriteMethod(String method) {
        return "POST".equals(method) || "PUT".equals(method) || "PATCH".equals(method);
    }
}
```

## 3. データベースレベルの重複防止

### 3.1 ユニーク制約

```sql
-- 振込テーブル: 同一冪等キーの重複防止
ALTER TABLE transfers
ADD CONSTRAINT uk_transfers_idempotency_key
UNIQUE (idempotency_key);
```

### 3.2 INSERT ... ON CONFLICT

```java
@Repository
public class TransferRepositoryImpl {

    @Transactional
    public TransferResult executeTransfer(Transfer transfer) {
        try {
            transferRepository.save(transfer);
            return TransferResult.created(transfer);
        } catch (DataIntegrityViolationException e) {
            // 重複 — 既存の結果を返す
            Transfer existing = transferRepository
                .findByIdempotencyKey(transfer.getIdempotencyKey())
                .orElseThrow();
            return TransferResult.duplicate(existing);
        }
    }
}
```

## 4. メッセージングの冪等性

### 4.1 消費者側の重複排除

```java
@KafkaListener(topics = "payment-events")
public void handlePaymentEvent(PaymentEvent event) {
    // メッセージIDで重複チェック
    if (processedMessageRepository.existsByMessageId(event.messageId())) {
        log.info("Duplicate message ignored: {}", event.messageId());
        return;
    }

    processPayment(event);

    processedMessageRepository.save(
        new ProcessedMessage(event.messageId(), Instant.now())
    );
}
```

## 5. 冪等キーの有効期限管理

```java
@Scheduled(cron = "0 0 2 * * *") // 毎日午前2時
public void cleanupExpiredKeys() {
    int deleted = idempotencyKeyRepository
        .deleteByExpiresAtBefore(Instant.now());
    log.info("Cleaned up {} expired idempotency keys", deleted);
}
```

## 6. HTTP メソッドと冪等性

| メソッド | 冪等 | 安全 | 対策 |
|---------|------|------|------|
| GET | Yes | Yes | キャッシュ |
| PUT | Yes | No | 全体置換で自然に冪等 |
| DELETE | Yes | No | 存在しない場合も204を返す |
| POST | **No** | No | **冪等キー必須** |
| PATCH | **No** | No | **冪等キー必須** |
