# テストパターン集

言語非依存の疑似コードによるテストパターン集。TDDのREDフェーズでこれらのパターンを参照し、プロジェクトのドメインに適したテストを組み込む。

プロジェクトに該当しないパターンはスキップしてよい。

---

## §1 エラーハンドリング・リカバリテスト

### パターン1.1: 例外発生時の状態整合性確認

```pseudo
// Given: 有効なリソース（初期状態）
resource = createResource(status = "ACTIVE")

// When: 処理中に例外が発生する
mock externalService.call() to throw RuntimeException("Network error")

try:
    service.process(resource.id)
catch:
    pass

// Then: リソースの状態が変更されていないこと（部分更新されていない）
reloadedResource = repository.findById(resource.id)
assert reloadedResource.status == "ACTIVE"
```

### パターン1.2: 部分更新防止（複数リソース更新時）

```pseudo
// Given: 関連する2つのリソース
resourceA = createResource(value = 100)
resourceB = createResource(value = 200)

// When: 2番目の操作で失敗する
mock repository.updateB() to throw DataAccessException("DB error")

try:
    service.transferBetween(resourceA.id, resourceB.id, amount = 50)
catch:
    pass

// Then: どちらも更新されていないこと
assert repository.findById(resourceA.id).value == 100
assert repository.findById(resourceB.id).value == 200
```

---

## §2 データ変更の追跡テスト

### パターン2.1: 状態変更時のイベント記録確認

```pseudo
// Given: 有効なリソースと認証済みユーザー
resource = createResource(status = "DRAFT")
authenticateAs(user = "operator-1")

// When: 状態変更操作を実行する
service.publish(resource.id)

// Then: イベント/ログが記録されていること
events = eventStore.findByResourceId(resource.id)
assert events.size() >= 1

event = events.last()
assert event.action == "PUBLISH"
assert event.userId == "operator-1"
assert event.timestamp != null
```

### パターン2.2: ログに機密データが含まれていないこと

```pseudo
// Given: 機密データを含む操作
user = createUser(
    name     = "John Doe",
    email    = "john@example.com",
    password = "secret123"
)

// When: 操作を実行する
service.updateProfile(user.id, newName = "Jane Doe")

// Then: ログに機密データが平文で含まれていないこと
logOutput = captureLogOutput()
assert "secret123"        not in logOutput
assert "john@example.com" not in logOutput  // メールがPIIの場合
```

---

## §3 並行アクセステスト

### パターン3.1: 同時更新の競合検出

```pseudo
// Given: 有効なリソース（version 0）
resource = createResource(value = 100)

// When: 2つの操作が同時に更新を試みる
// 操作1: リソースを読み取る
resource1 = repository.findById(resource.id)

// 操作2: 先にコミットする
resource2 = repository.findById(resource.id)
resource2.value = 80
repository.save(resource2)  // version 0 → 1

// 操作1: 後からコミットを試みる
resource1.value = 70

// Then: 競合が検出されること
assertThrows ConflictException:
    repository.save(resource1)  // version 0 != 1 → 衝突
```

### パターン3.2: 並行スレッドによる同時更新テスト

```pseudo
// Given: 有効なリソース（value = 1000）
resource = createResource(value = 1000)

// When: 10スレッドが同時にvalue を10ずつ減算する
results = runConcurrently(threadCount = 10):
    try:
        service.decrement(resource.id, amount = 10)
        return "SUCCESS"
    catch ConflictException:
        return "CONFLICT"

// Then: 成功した操作の数だけ値が減っていること
successCount = results.count("SUCCESS")
finalValue = repository.findById(resource.id).value
assert finalValue == 1000 - (10 * successCount)
```

---

## §4 冪等性テスト

### パターン4.1: 同一リクエストの重複実行防止

```pseudo
// Given: 有効なリクエストとリクエストID
requestId = "req-uuid-001"
request = CreateRequest(data = "test-data")

// When: 同じリクエストIDで2回実行する
result1 = service.execute(request, requestId)
result2 = service.execute(request, requestId)

// Then: 2回目は重複として処理されること
assert result1.status == "CREATED"
assert result2.status == "DUPLICATE"
assert result1.resourceId == result2.resourceId
```

### パターン4.2: 異なるリクエストIDは別々に処理されること

```pseudo
// Given: 同じデータだが異なるリクエストID

// When: 異なるリクエストIDで実行する
result1 = service.execute(request, requestId = "req-001")
result2 = service.execute(request, requestId = "req-002")

// Then: それぞれ独立した操作として処理されること
assert result1.status == "CREATED"
assert result2.status == "CREATED"
assert result1.resourceId != result2.resourceId
```

---

## §5 バリデーション・境界値テスト

### パターン5.1: 必須フィールドのバリデーション

```pseudo
// Given: 必須フィールドが欠けたリクエスト

// Then: 必須フィールドなしではエラーになること
assertThrows ValidationException:
    service.create(name = null)

assertThrows ValidationException:
    service.create(name = "")
```

### パターン5.2: 境界値テスト

```pseudo
// Given: 値の範囲制約があるフィールド

// Then: 下限値は許可されること
result = service.create(value = MIN_VALUE)
assert result.status == "SUCCESS"

// And: 上限値は許可されること
result = service.create(value = MAX_VALUE)
assert result.status == "SUCCESS"

// And: 下限未満はエラーになること
assertThrows ValidationException:
    service.create(value = MIN_VALUE - 1)

// And: 上限超過はエラーになること
assertThrows ValidationException:
    service.create(value = MAX_VALUE + 1)
```

### パターン5.3: 文字列長テスト

```pseudo
// Given: 文字列長の制約があるフィールド

// Then: 最大長ちょうどは許可されること
result = service.create(name = "a" * MAX_LENGTH)
assert result.status == "SUCCESS"

// And: 最大長超過はエラーになること
assertThrows ValidationException:
    service.create(name = "a" * (MAX_LENGTH + 1))
```

---

## §6 認証・認可テスト

### パターン6.1: 未認証アクセスの拒否

```pseudo
// Given: 認証なしの状態

// When: 保護されたエンドポイントにアクセスする
response = httpClient.get("/api/v1/protected-resource")

// Then: 401 Unauthorized が返ること
assert response.status == 401
```

### パターン6.2: 権限不足のアクセス拒否

```pseudo
// Given: 一般ユーザーとして認証
authenticateAs(user = "regular-user", role = "USER")

// When: 管理者のみ許可された操作を実行する
response = httpClient.delete("/api/v1/resource/123")

// Then: 403 Forbidden が返ること
assert response.status == 403
```

### パターン6.3: 他ユーザーのリソースへのアクセス拒否

```pseudo
// Given: ユーザーAのリソース
resourceA = createResource(ownerId = "user-a")

// When: ユーザーBがアクセスする
authenticateAs(user = "user-b")
response = httpClient.get("/api/v1/resource/" + resourceA.id)

// Then: アクセスが拒否されること
assert response.status == 403 or response.status == 404
```
