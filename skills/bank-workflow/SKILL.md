---
name: bank-workflow
description: 銀行システム開発の5段階ワークフロー（ADR→設計書→実装計画→TDD→コードレビュー）をオーケストレーションする。各ステージの成果物存在チェックと承認ゲート管理を行う。
argument-hint: "[feature] [--from=stage] - 機能名と開始ステージ"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git *), Bash(./gradlew *), Bash(./mvnw *), Bash(ls *), Bash(find *)
---

# 銀行ワークフロー オーケストレーションスキル

## 既存成果物
!`echo "=== ADR ==="; ls docs/adr/*.md 2>/dev/null || echo "なし"; echo "=== 設計書 ==="; ls docs/design/*.md 2>/dev/null || echo "なし"; echo "=== 計画 ==="; ls docs/plans/*.md 2>/dev/null || echo "なし"`

## ビルドシステム
!`if [ -f build.gradle ] || [ -f build.gradle.kts ]; then echo "Gradle"; elif [ -f pom.xml ]; then echo "Maven"; else echo "不明"; fi`

## 引数の処理

`$ARGUMENTS` をパースする:

1. `--from=` パラメータがある場合: 開始ステージを指定する
   - 有効な値: `adr`, `design`, `plan`, `tdd`, `review`
   - 無効な値の場合: 「有効なステージ: adr, design, plan, tdd, review」を提示して再入力を求める
2. `--from=` 以外の引数: 機能名として使用する
3. `$ARGUMENTS` が空の場合: ユーザーに機能名を確認する

**例:**
- `/bank-workflow 振込機能` → 機能名「振込機能」、Stage 1 (ADR) から開始
- `/bank-workflow 振込機能 --from=tdd` → 機能名「振込機能」、Stage 4 (TDD) から開始
- `/bank-workflow --from=design` → 機能名をユーザーに確認、Stage 2 から開始

## ワークフロー概要

```
Stage 1: ADR        → docs/adr/{NNN}-{title}.md
    ↓ [承認ゲート]
Stage 2: 設計書      → docs/design/{name}-design.md
    ↓ [承認ゲート]
Stage 3: 実装計画    → docs/plans/{name}-plan.md
    ↓ [承認ゲート]
Stage 4: TDD実装    → src/main/java/... + src/test/java/...
    ↓ [テスト全通過]
Stage 5: レビュー    → レビュー結果 + 修正
```

## ワークフロー実行

### Step 1: 機能の特定と開始ステージの決定

1. 引数から機能名と開始ステージを取得する
2. 上記の「既存成果物」に基づき、最後の完了ステージを自動検出する
3. `--from=` 指定がない場合、既存成果物から最適な開始ステージを提案する

### Step 2: 成果物存在チェック

以下のディレクトリをスキャンして既存の成果物を確認:

```
docs/adr/           → ADRファイルの有無
docs/design/        → 設計書ファイルの有無
docs/plans/         → 実装計画書ファイルの有無
src/test/java/      → テストファイルの有無
src/main/java/      → 実装ファイルの有無
```

既存の成果物がある場合:
- ユーザーに現在のステージを提示する
- 途中のステージから再開するか確認する

### Step 3: Stage 1 — ADR作成

1. `/adr` スキルを呼び出す（機能名を引数として渡す）
2. ADRが生成されたことを確認する
3. **承認ゲート**: ユーザーにADRの内容を確認してもらう
   - 「承認」→ Stage 2 へ進む
   - 「修正」→ ADRを修正して再確認
   - 「スキップ」→ ADRなしで Stage 2 へ（非推奨の旨を伝える）

**エラーハンドリング:** ADR作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 4: Stage 2 — 設計書作成

1. `/design` スキルを呼び出す（ADR番号を引数として渡す）
2. 設計書が生成されたことを確認する
3. **承認ゲート**: ユーザーに設計書の内容を確認してもらう

**エラーハンドリング:** 設計書作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 5: Stage 3 — 実装計画書作成

1. `/plan` スキルを呼び出す（設計書名を引数として渡す）
2. 実装計画書が生成されたことを確認する
3. **承認ゲート**: ユーザーに実装計画書の内容を確認してもらう

**エラーハンドリング:** 計画書作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 6: Stage 4 — TDD実装

1. `/bank-tdd` スキルを呼び出す（実装計画書のTaskに沿って実行）
2. 各Task完了時に進捗を報告する
3. **テストゲート**: すべてのテストが通過することを確認する

```bash
./gradlew test  # or ./mvnw test
```

**エラーハンドリング:** テスト失敗の場合は修正を試みる。ビルドエラーの場合はエラー内容を表示し「リトライ / 中止」を提案する

### Step 7: Stage 5 — コードレビュー

1. `/bank-review` スキルを呼び出す
2. CRITICAL / HIGH の指摘がある場合:
   - 修正を提案する
   - ユーザーの承認後に修正を適用する
   - 修正後にテストを再実行する
3. すべての指摘が解消されるまで繰り返す

### Step 8: 完了

1. 成果物の一覧を表示する:
   - ADR: `docs/adr/{NNN}-{title}.md`
   - 設計書: `docs/design/{name}-design.md`
   - 実装計画: `docs/plans/{name}-plan.md`
   - テスト: `src/test/java/...`
   - 実装: `src/main/java/...`
2. コミットを提案する

## 途中再開

ワークフローは任意のステージから再開可能:

```
/bank-workflow {feature} --from=design    → Stage 2 から開始
/bank-workflow {feature} --from=plan      → Stage 3 から開始
/bank-workflow {feature} --from=tdd       → Stage 4 から開始
/bank-workflow {feature} --from=review    → Stage 5 から開始
```
