# 検証ループ（Verification Loop）パターン

PR作成前やマイルストーン到達時に実行する、6フェーズの品質検証パターン。
`/stack-loop` の Step 4（ローカル検証）で自動的に参照される。

---

## 検証フェーズ

### Phase 1: ビルド検証

プロジェクトが正常にビルドできることを確認する。

```pseudo
// ビルドシステムに応じたコマンドを実行
build_result = execute(build_command)  // npm run build, ./gradlew build, cargo build, etc.
assert build_result.exitCode == 0, "ビルドが失敗しています"
```

**判定基準**: ビルドエラーが0件であること。

### Phase 2: 型チェック

静的型システムによる型安全性を確認する。

```pseudo
// 型チェックコマンドを実行
type_result = execute(type_check_command)  // npx tsc --noEmit, mypy, etc.
assert type_result.errors == 0, "型エラーがあります"
```

**判定基準**: 型エラーが0件であること。

### Phase 3: Lint チェック

コーディング規約への準拠を確認する。

```pseudo
// Lintコマンドを実行
lint_result = execute(lint_command)  // eslint, ruff, clippy, etc.
assert lint_result.errors == 0, "Lintエラーがあります"

// 自動修正可能な場合は修正を適用
if lint_result.fixable > 0:
    execute(lint_command + " --fix")
```

**判定基準**: エラーが0件であること。警告は許容するが、新規追加の警告は確認する。

### Phase 4: テストスイート

全テストの実行とカバレッジの確認。

```pseudo
// テストスイートを実行
test_result = execute(test_command + " --coverage")

// 結果の検証
assert test_result.failures == 0, "テストが失敗しています"
assert test_result.coverage >= COVERAGE_THRESHOLD, "カバレッジが不足しています"
```

**判定基準**:
- テスト失敗: 0件
- カバレッジ: プロジェクト基準に準拠（推奨: 80%以上）
- 新規コードのカバレッジ: 90%以上を目標

### Phase 5: セキュリティスキャン

シークレットの漏洩や `console.log` の残存を検出する。

```pseudo
// シークレットの検出
secrets = grep_patterns([
    /(?:api[_-]?key|secret|password|token)\s*[:=]\s*['"][^'"]+['"]/i,
    /(?:AKIA|AIza)[A-Za-z0-9]{16,}/,  // AWS/GCP キー形式
])
assert secrets.count == 0, "シークレットが検出されました"

// デバッグ出力の検出（本番コード内のみ、テストファイルは除外）
debug_outputs = grep_in_source([
    /console\.log\(/,
    /print\(/,        // Python（logging以外）
    /println!\(/,     // Rust（debug用）
])
// 警告として報告（ブロックはしない）
```

**判定基準**: シークレット検出は即座にブロック。デバッグ出力は警告。

### Phase 6: 差分レビュー

変更ファイルの最終確認。

```pseudo
// 変更ファイルの確認
changed_files = git_diff("--name-only")

for file in changed_files:
    // 意図しない変更の検出
    assert file in expected_changes, "意図しないファイル変更: " + file

    // エラーハンドリングの欠落検出
    if file.is_source_code():
        diff = git_diff(file)
        check_error_handling(diff)
        check_resource_cleanup(diff)
```

**判定基準**:
- 意図しないファイル変更がない
- エラーハンドリングが適切
- リソースのクリーンアップが行われている

---

## 検証結果のフォーマット

```markdown
# 検証ループ結果

| Phase | 状態 | 詳細 |
|-------|------|------|
| ビルド | PASS/FAIL | {詳細} |
| 型チェック | PASS/FAIL/SKIP | {詳細} |
| Lint | PASS/FAIL | {詳細} |
| テスト | PASS/FAIL | {通過数}/{全体数}, カバレッジ: {N}% |
| セキュリティ | PASS/WARN/FAIL | {詳細} |
| 差分レビュー | PASS/WARN | {詳細} |

**総合判定**: READY / NOT READY
```

## 実行タイミング

- `/stack-loop` の各PR完了時（Step 4）
- 大きなリファクタリング後
- PR作成前の最終チェック
- 長時間の開発セッション中（15分ごと推奨）
