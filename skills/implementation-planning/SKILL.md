---
name: implementation-planning
description: システム設計書を入力として、TDD対応の実装計画書を作成する。Phase分割、TDDステップ付きTask定義、銀行固有チェックポイント、Gradle/Mavenコマンドを含む。
argument-hint: "[design-name] - 対象設計書（例: transfer-service）"
allowed-tools: Read, Write, Glob, Grep, Bash(ls *), Bash(find *), Bash(./gradlew *), Bash(./mvnw *)
---

# 実装計画書作成スキル

## 利用可能な設計書一覧
!`ls docs/design/*.md 2>/dev/null || echo "設計書なし"`

## ビルドシステム
!`if [ -f build.gradle ] || [ -f build.gradle.kts ]; then echo "Gradle"; elif [ -f pom.xml ]; then echo "Maven"; else echo "不明"; fi`

## 引数の処理

- `$ARGUMENTS` が提供された場合: その内容を対象設計書名として使用し、`docs/design/` から該当ファイルを検索する
- `$ARGUMENTS` が空の場合: 上記の「利用可能な設計書一覧」を提示し、ユーザーに対象設計書を選択してもらう

## 前提条件

- 関連する設計書が `docs/design/` に存在すること
- 関連するADRが `docs/adr/` に存在すること

## ワークフロー

### Step 1: 設計書の読込と分析

1. 設計書から以下を抽出する:
   - コンポーネント一覧と依存関係
   - API仕様
   - データモデル
   - トランザクション設計
   - 監査ログ設計

**エラーハンドリング:**
- 設計書が存在しない場合、`/design` の実行を提案する
- 設計書が複数ある場合、一覧を提示して選択を求める

### Step 2: 既存コードベースの分析

1. ビルドシステムを特定する（Gradle / Maven）
2. テストフレームワークを特定する
3. 既存のテストパターンを参照する
4. パッケージ構成を確認する

**エラーハンドリング:**
- ビルドシステムが検出できない場合、ユーザーにGradle/Mavenのいずれかを確認する

### Step 3: Phase分割

設計書のコンポーネントを実装Phaseに分割する:

1. **Phase 1**: ドメイン層（エンティティ、バリューオブジェクト）
2. **Phase 2**: リポジトリ層（Repository インターフェース、実装）
3. **Phase 3**: サービス層（ビジネスロジック、トランザクション制御）
4. **Phase 4**: プレゼンテーション層（Controller、バリデーション）
5. **Phase 5**: 横断的関心事（監査ログ、排他制御、冪等性）
6. **Phase 6**: インテグレーションテスト

### Step 4: TDDタスク分解

各Phaseを具体的なTaskに分解し、それぞれにTDDステップを付与:

```
Task N.M: {タスク名}
├── RED: テストファイルとテストケースの定義
│   ├── テストファイルパス
│   ├── テストケース一覧（Given/When/Then形式）
│   └── テストコード概要
├── GREEN: 最小実装
│   ├── 実装ファイルパス
│   └── 実装項目一覧
├── REFACTOR: リファクタリング項目
└── 検証コマンド（gradle/maven）
```

### Step 5: 銀行固有チェックポイントの挿入

各Phase完了時の確認項目を追加:
- トランザクション設計の準拠
- 監査ログの実装
- 排他制御の適用
- 冪等性の確保
- 金額計算の安全性（BigDecimal）

### Step 6: 依存関係図の生成

Task間の依存関係をMermaid図で表現する

### Step 7: 計画書生成

1. `templates/implementation-plan-template.md` を使用する
2. `docs/plans/{kebab-case-name}-plan.md` にファイルを作成する

**エラーハンドリング:**
- `docs/plans/` が存在しない場合は `mkdir -p docs/plans` で作成する
- テンプレートが見つからない場合、ビルトイン構造で代替する

### Step 8: 承認ゲート

ユーザーに計画書の内容を確認してもらう:
- 修正が必要な場合は修正する
- 承認された場合、次のステージ（`/bank-tdd`）への進行を提案する

## ビルドシステム検出

```
1. `build.gradle` or `build.gradle.kts` → Gradle
   検証コマンド: `./gradlew test --tests "{TestClass}"`
2. `pom.xml` → Maven
   検証コマンド: `./mvnw test -Dtest="{TestClass}"`
```
