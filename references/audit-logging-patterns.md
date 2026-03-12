# 監査ログパターン — ミッションクリティカルシステム向けリファレンス

## 1. 基本原則

- **5W1H**: Who(誰が), What(何を), When(いつ), Where(どこで), Why(なぜ — 可能な場合), How(どのように)
- **Append-only**: 監査ログは追記のみ、更新・削除不可
- **独立トランザクション**: メイン処理が失敗しても監査ログは残す
- **PII除外**: 個人識別情報はマスキングまたは除外
- **改ざん防止**: ハッシュチェーン等で改ざんを検出可能にする

## 2. 監査ログエンティティ

```pseudo
// スキーマ定義: テーブル名 "audit_logs"、変更検知無効（Append-only）
class AuditLog:

    id: Long                // 主キー（自動採番）
    timestamp: Instant      // NOT NULL
    eventType: String       // NOT NULL, 最大50文字
    userId: String          // NOT NULL, 最大100文字
    action: String          // NOT NULL, 最大50文字
    resourceType: String    // NOT NULL, 最大100文字
    resourceId: String      // 最大100文字
    details: Text           // JSON形式、PIIマスキング済み
    result: String          // NOT NULL, "SUCCESS" / "FAILURE"
    ipAddress: String       // 最大45文字
    correlationId: String   // NOT NULL, リクエスト追跡用UUID（最大36文字）
    previousHash: String    // NOT NULL, 改ざん防止用ハッシュチェーン（最大64文字）
    hash: String            // NOT NULL, 最大64文字

    // コンストラクタのみ（setterなし — イミュータブル）
```

## 3. 監査ログサービス

```pseudo
class AuditLogService:

    auditLogRepository: AuditLogRepository

    // トランザクション宣言（独立トランザクション）
    function log(event: AuditEvent):
        previousHash = auditLogRepository.findLatestHash()
            ?? "GENESIS"

        auditLog = AuditLog(
            timestamp     = Instant.now(),
            eventType     = event.eventType,
            userId        = event.userId,
            action        = event.action,
            resourceType  = event.resourceType,
            resourceId    = event.resourceId,
            details       = maskPii(event.details),
            result        = event.result,
            ipAddress     = event.ipAddress,
            correlationId = event.correlationId,
            previousHash  = previousHash,
            hash          = computeHash(previousHash, event)
        )

        auditLogRepository.save(auditLog)

    private function maskPii(details: String): String:
        // 口座番号: 末尾4桁以外をマスク
        // 氏名: 姓のみ表示
        // メール: ドメイン以外をマスク
        return PiiMasker.mask(details)

    private function computeHash(previousHash: String, event: AuditEvent): String:
        data = previousHash + event.timestamp + event.eventType
             + event.userId + event.action
        return sha256Hex(data)
```

## 4. AOP による自動監査ログ

```pseudo
// アスペクト定義
class AuditAspect:

    auditLogService: AuditLogService

    // @Audited アノテーション付きメソッドの前後に自動実行
    function audit(joinPoint, auditedAnnotation):
        userId        = SecurityContext.getCurrentUser().name
        correlationId = RequestContext.get("correlationId")

        try:
            result = joinPoint.proceed()
            auditLogService.log(AuditEvent.success(
                auditedAnnotation.eventType,
                userId,
                auditedAnnotation.action,
                auditedAnnotation.resourceType,
                extractResourceId(joinPoint, auditedAnnotation),
                correlationId
            ))
            return result
        catch Exception as e:
            auditLogService.log(AuditEvent.failure(
                auditedAnnotation.eventType,
                userId,
                auditedAnnotation.action,
                auditedAnnotation.resourceType,
                extractResourceId(joinPoint, auditedAnnotation),
                correlationId,
                e.message
            ))
            throw e

// カスタムアノテーション定義（メソッドに付与して自動監査ログを有効化）
annotation Audited:
    eventType: String
    action: String
    resourceType: String
    resourceIdParam: Int = -1
```

## 5. 使用例

```pseudo
class AccountService:

    // 自動監査ログ: eventType="ACCOUNT", action="TRANSFER", resourceType="Account"
    // トランザクション宣言
    function transfer(fromAccountId: Long, toAccountId: Long, amount: Decimal): TransferResult:
        // ビジネスロジック — 監査ログは自動記録
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

```pseudo
class AuditIntegrityService:

    function verifyChain(): Boolean:
        logs = auditLogRepository.findAllOrderByIdAsc()
        previousHash = "GENESIS"

        for log in logs:
            if log.previousHash != previousHash:
                return false  // チェーン破損
            computed = computeHash(previousHash, log)
            if log.hash != computed:
                return false  // 改ざん検出
            previousHash = log.hash

        return true
```
