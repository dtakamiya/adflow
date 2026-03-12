# 監査ログパターン — 銀行システム向けリファレンス

## 1. 基本原則

- **5W1H**: Who(誰が), What(何を), When(いつ), Where(どこで), Why(なぜ — 可能な場合), How(どのように)
- **Append-only**: 監査ログは追記のみ、更新・削除不可
- **独立トランザクション**: メイン処理が失敗しても監査ログは残す
- **PII除外**: 個人識別情報はマスキングまたは除外
- **改ざん防止**: ハッシュチェーン等で改ざんを検出可能にする

## 2. 監査ログエンティティ

```java
@Entity
@Table(name = "audit_logs")
@Immutable // Hibernateの変更検知を無効化
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Instant timestamp;

    @Column(nullable = false, length = 50)
    private String eventType;

    @Column(nullable = false, length = 100)
    private String userId;

    @Column(nullable = false, length = 50)
    private String action;

    @Column(nullable = false, length = 100)
    private String resourceType;

    @Column(length = 100)
    private String resourceId;

    @Column(columnDefinition = "TEXT")
    private String details; // JSON形式、PIIマスキング済み

    @Column(nullable = false, length = 10)
    private String result; // SUCCESS / FAILURE

    @Column(length = 45)
    private String ipAddress;

    @Column(nullable = false, length = 36)
    private String correlationId; // リクエスト追跡用UUID

    @Column(nullable = false, length = 64)
    private String previousHash; // 改ざん防止用ハッシュチェーン

    @Column(nullable = false, length = 64)
    private String hash;

    // コンストラクタのみ（setterなし — イミュータブル）
}
```

## 3. 監査ログサービス

```java
@Service
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(AuditEvent event) {
        String previousHash = auditLogRepository.findLatestHash()
            .orElse("GENESIS");

        AuditLog auditLog = AuditLog.builder()
            .timestamp(Instant.now())
            .eventType(event.eventType())
            .userId(event.userId())
            .action(event.action())
            .resourceType(event.resourceType())
            .resourceId(event.resourceId())
            .details(maskPii(event.details()))
            .result(event.result())
            .ipAddress(event.ipAddress())
            .correlationId(event.correlationId())
            .previousHash(previousHash)
            .hash(computeHash(previousHash, event))
            .build();

        auditLogRepository.save(auditLog);
    }

    private String maskPii(String details) {
        // 口座番号: 末尾4桁以外をマスク
        // 氏名: 姓のみ表示
        // メール: ドメイン以外をマスク
        return PiiMasker.mask(details);
    }

    private String computeHash(String previousHash, AuditEvent event) {
        String data = previousHash + event.timestamp() + event.eventType()
            + event.userId() + event.action();
        return DigestUtils.sha256Hex(data);
    }
}
```

## 4. AOP による自動監査ログ

```java
@Aspect
@Component
public class AuditAspect {

    private final AuditLogService auditLogService;

    @Around("@annotation(audited)")
    public Object audit(ProceedingJoinPoint joinPoint, Audited audited) throws Throwable {
        String userId = SecurityContextHolder.getContext()
            .getAuthentication().getName();
        String correlationId = MDC.get("correlationId");

        try {
            Object result = joinPoint.proceed();
            auditLogService.log(AuditEvent.success(
                audited.eventType(),
                userId,
                audited.action(),
                audited.resourceType(),
                extractResourceId(joinPoint, audited),
                correlationId
            ));
            return result;
        } catch (Exception e) {
            auditLogService.log(AuditEvent.failure(
                audited.eventType(),
                userId,
                audited.action(),
                audited.resourceType(),
                extractResourceId(joinPoint, audited),
                correlationId,
                e.getMessage()
            ));
            throw e;
        }
    }
}

// カスタムアノテーション
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    String eventType();
    String action();
    String resourceType();
    int resourceIdParam() default -1;
}
```

## 5. 使用例

```java
@Service
public class AccountService {

    @Audited(
        eventType = "ACCOUNT",
        action = "TRANSFER",
        resourceType = "Account",
        resourceIdParam = 0
    )
    @Transactional
    public TransferResult transfer(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        // ビジネスロジック — 監査ログは自動記録
    }
}
```

## 6. PIIマスキングルール

| データ種別 | マスキング方式 | 例 |
|----------|-------------|-----|
| 口座番号 | 末尾4桁のみ表示 | ****1234 |
| 氏名 | 姓のみ表示 | 山田** |
| メールアドレス | ドメインのみ表示 | ****@example.com |
| 電話番号 | 末尾4桁のみ表示 | ****5678 |
| 住所 | 都道府県のみ表示 | 東京都*** |
| 金額 | マスキングしない（業務上必要） | ¥1,000,000 |

## 7. 改ざん検証

```java
@Service
public class AuditIntegrityService {

    public boolean verifyChain() {
        List<AuditLog> logs = auditLogRepository.findAllOrderByIdAsc();
        String previousHash = "GENESIS";

        for (AuditLog log : logs) {
            if (!log.getPreviousHash().equals(previousHash)) {
                return false; // チェーン破損
            }
            String computed = computeHash(previousHash, log);
            if (!log.getHash().equals(computed)) {
                return false; // 改ざん検出
            }
            previousHash = log.getHash();
        }
        return true;
    }
}
```
