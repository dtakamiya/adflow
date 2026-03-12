---
name: bank-tdd
description: 銀行システム向けTDD（テスト駆動開発）。RED→GREEN→REFACTORサイクルに銀行固有のテストパターン（トランザクションロールバック、監査ログ検証、並行アクセス、冪等性、BigDecimalアサーション）を組み込む。
argument-hint: "[task-id] - 対象Task（例: 1.1 or TransferService）"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(./gradlew *), Bash(./mvnw *), Bash(ls *), Bash(find *)
---

# 銀行TDDスキル

## 銀行テストパターン
!`cat ${CLAUDE_SKILL_DIR}/bank-testing-patterns.md 2>/dev/null || echo "パターンファイルなし"`

## ビルドシステム
!`if [ -f build.gradle ] || [ -f build.gradle.kts ]; then echo "Gradle"; elif [ -f pom.xml ]; then echo "Maven"; else echo "不明"; fi`

## 引数の処理

- `$ARGUMENTS` が提供された場合: その内容を対象Task IDまたはクラス名として使用し、実装計画書から該当Taskを検索する
- `$ARGUMENTS` が空の場合: ユーザーに実装対象のTaskまたはクラスを確認する

## 前提条件

- 実装計画書が `docs/plans/` に存在すること（推奨）
- ビルドが通る状態であること

## ワークフロー

### Step 1: 実装計画の読込

1. 実装計画書がある場合、対象Taskを特定する
2. ない場合、ユーザーに実装対象を確認する
3. 対象Taskから以下を抽出:
   - テストケース一覧
   - 実装ファイルパス
   - 検証コマンド

### Step 2: RED — テスト作成

1. テストファイルを作成する
2. Given/When/Then 形式でテストケースを記述する
3. 上記「銀行テストパターン」を参照して必要なテストを追加する
4. テストを実行して **失敗することを確認する**

```bash
./gradlew test --tests "{TestClass}" # or ./mvnw test -Dtest="{TestClass}"
```

**重要**: テストが失敗しない場合は進まない。テストの書き方を見直す。

**エラーハンドリング:**
- ビルドエラー（コンパイルエラー等、テスト失敗ではない）の場合は停止してエラー内容を報告し、修正してからリトライする
- ビルドシステムが検出できない場合、ユーザーに確認する

### Step 3: GREEN — 最小実装

1. テストを通すための最小限のコードを実装する
2. 過度な設計をしない — テストが通ることだけを目指す
3. テストを実行して **成功することを確認する**

```bash
./gradlew test --tests "{TestClass}"
```

**重要**: テストが成功しない場合は進まない。実装を修正する。

### Step 4: REFACTOR — リファクタリング

1. コードの重複を除去する
2. 命名を改善する
3. SOLID原則に沿っているか確認する
4. 銀行固有の品質基準を確認:
   - BigDecimal が使用されているか
   - @Transactional が適切か
   - 監査ログが実装されているか
5. テストを再実行して **変わらず成功することを確認する**

### Step 5: 繰り返し

次のテストケース / 次のTaskに進み、Step 2〜4を繰り返す

### Step 6: カバレッジ確認

```bash
./gradlew jacocoTestReport # or equivalent
```

目標: 80%以上

**エラーハンドリング:**
- JaCoCoが未設定の場合、カバレッジレポートをスキップし「JaCoCoが未設定のためカバレッジレポートを生成できません」と警告する

## 銀行固有テストの組み込みルール

以下の条件に該当する場合、自動的に対応するテストパターンを追加する:

| 実装内容 | 必須テスト | 参照 |
|---------|----------|------|
| 金額フィールドがある | BigDecimalアサーション | bank-testing-patterns.md §5 |
| @Transactional がある | ロールバックテスト | bank-testing-patterns.md §1 |
| 状態変更操作がある | 監査ログ検証テスト | bank-testing-patterns.md §2 |
| @Version がある | 並行アクセステスト | bank-testing-patterns.md §3 |
| 冪等キーがある | 冪等性テスト | bank-testing-patterns.md §4 |
