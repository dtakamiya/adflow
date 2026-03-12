---
name: stack-planning
description: 仕様書を入力として、スタックPR（積み上げ型の小さなPR）の実装計画書を作成する。PRの分割、TDDステップ付きTask定義、ドメイン品質チェックポイント、ビルドシステムコマンドを含む。
argument-hint: "[spec-name or feature-name] - 対象仕様書または機能名（例: transfer-service）"
allowed-tools: Read, Write, Glob, Grep, Bash(ls *), Bash(find *), Bash(mkdir *), Bash(./gradlew *), Bash(./mvnw *), Bash(npm *), Bash(npx *), Bash(pytest *), Bash(cargo *), Bash(go *), Bash(dotnet *), Bash(make *)
---

# スタックPR計画作成スキル

## アクティブなワークフロー（stack-plan待ち）
!`for d in docs/*/; do if [ -f "$d.adflow-context.md" ] && grep -q "stack-plan" "$d.adflow-context.md" 2>/dev/null; then echo "--- $(basename $d) ---"; cat "$d.adflow-context.md"; echo; fi; done 2>/dev/null || echo "対象ワークフローなし"`

## 利用可能な仕様書一覧
!`ls docs/*/*-spec.md 2>/dev/null || echo "仕様書なし"`

## ビルドシステム
!`if [ -f build.gradle ] || [ -f build.gradle.kts ]; then echo "Gradle"; elif [ -f pom.xml ]; then echo "Maven"; elif [ -f package.json ]; then echo "Node.js"; elif [ -f pyproject.toml ] || [ -f setup.py ]; then echo "Python"; elif [ -f Cargo.toml ]; then echo "Rust"; elif [ -f go.mod ]; then echo "Go"; elif ls *.csproj >/dev/null 2>&1 || [ -f *.sln ]; then echo ".NET"; elif [ -f Makefile ]; then echo "Make"; else echo "不明"; fi`

## 引数の処理

- `$ARGUMENTS` が提供された場合:
  - 仕様書名の場合: `docs/{名前}-*/02-spec.md` で検索し、該当する機能名ディレクトリを特定
  - 機能名の場合: `docs/{機能名}/.adflow-context.md` を直接読み込む
- `$ARGUMENTS` が空の場合: 以下の自動検出ロジックを実行する

### 前段ドキュメント自動検出

1. `docs/*/` 配下の `.adflow-context.md` をスキャンする
2. 「現在のステージ」が `stack-plan` のコンテキストファイルを検索する
3. 該当が1件: そのワークフローの仕様書を自動選択する
4. 該当が複数: 一覧を提示してユーザーに選択してもらう
5. 該当なし: 旧方式のフォールバック（`docs/*-spec.md` から最新を候補提示 or `/spec` の実行を提案）

## 前提条件

- 関連する仕様書が `docs/{feature-name}/02-spec.md` に存在すること
- 関連するADRが `docs/{feature-name}/01-adr.md` に存在すること

## ワークフロー

### Step 1: 仕様書の読込と分析

1. コンテキストファイルから仕様書とADRのファイルパスを取得する
2. 仕様書を **Read ツールで全文読み込む**
3. ADRも **Read ツールで全文読み込む**
4. 以下を抽出し計画の入力コンテキストとして保持:
   - コンポーネント一覧と依存関係（仕様書 §2）
   - API仕様（仕様書 §4）
   - データモデル（仕様書 §5）
   - トランザクション設計（仕様書 §6）
   - 監査ログ設計（仕様書 §7）
5. 計画書の「関連ドキュメント」セクションに正確なリンクを自動挿入

**エラーハンドリング:**
- 仕様書が存在しない場合、`/spec` の実行を提案する
- 仕様書が複数ある場合、一覧を提示して選択を求める

### Step 2: 既存コードベースの分析

1. ビルドシステムを特定する（上記「ビルドシステム」セクション参照）
2. テストフレームワークを特定する
3. 既存のテストパターンを参照する
4. パッケージ構成を確認する

**エラーハンドリング:**
- ビルドシステムが検出できない場合、ユーザーにビルドシステムを確認する

### Step 3: スタックPRの分割（PRごとの境界定義）

仕様書の要件をレビュアーが確認しやすいように、独立した小さな「スタックPR」単位に分割する:

1. **PR 1**: ドメイン層（エンティティ、バリューオブジェクト）
2. **PR 2**: リポジトリ層（Repository インターフェース、実装）※親: PR 1
3. **PR 3**: サービス層（ビジネスロジック、トランザクション制御）※親: PR 2
4. **PR 4**: プレゼンテーション層（Controller、バリデーション）※親: PR 3
5. **PR 5**: 横断的関心事と統合テスト（監査ログ、排他制御など）※親: PR 4

各PRは「前のPRに依存して積み上げる（スタック）」前提で定義する。

### Step 4: 各PR内のTDDタスク分解

各PR（Phase）を具体的なTask（コミット単位）に分解し、それぞれにTDDステップを付与:

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
└── 検証コマンド（ビルドシステムに応じたコマンド）
```

### Step 5: ドメイン品質チェックポイントの挿入

各Phase完了時の確認項目を追加:
- トランザクション設計の準拠
- 監査ログの実装
- 排他制御の適用
- 冪等性の確保
- 金額計算の安全性（高精度小数型）
- トランザクション宣言の適用
- バージョンフィールド（楽観ロック）の確認

### Step 6: 依存関係図の生成

Task間の依存関係をMermaid図で表現する

### Step 7: 計画書生成

1. `templates/implementation-plan-template.md` を使用する
2. `docs/{feature-name}/03-plans.md` にファイルを作成する

**エラーハンドリング:**
- `docs/{feature-name}/` が存在しない場合は `mkdir -p docs/{feature-name}` で作成する
- テンプレートが見つからない場合、ビルトイン構造で代替する

### Step 8: AI自己レビュー（自動品質チェック）

生成したスタックPR計画に対して、AI自身で以下の品質を独自検証する:
- [ ] 各PRがレビュー可能な小さな単位に分割されているか
- [ ] 各PRごとの依存関係（親ブランチ）が明確になっているか
- [ ] すべてのTDDタスクにRED(テスト)/GREEN(実装)の指定があるか
- [ ] トランザクションや監査ログなどのドメイン品質チェックポイントが含まれているか
- [ ] 検証コマンドがビルドシステムと合致しているか

問題があれば、ユーザーに提示する前に自律的に修正を行う。

### Step 9: 承認ゲート（人間によるバリデーション）

自己レビューで品質を満たした計画書をユーザーに提示し、内容を確認してもらう:
- 修正が必要な場合は修正する
- 承認された場合、Step 10 に進む

### Step 10: コンテキスト更新

1. `docs/{feature-name}/.adflow-context.md` を更新する
2. 実装計画の成果物を記録: ファイルパス `docs/{feature-name}/03-plans.md`、ステータス「完了」
3. 現在のステージを `stack-loop` に更新する
4. ユーザーに案内する: **「次のステージ `/stack-loop` を実行するか、`/clear` して後で再開できます」**
   - **重要**: `/workflow` からの実行でない限り、自律的に次のコマンドを実行してはいけません。必ずユーザーからの明示的なコマンド実行（または進行の承認）を待ってください。

## ビルドシステム検出

```
1. build.gradle / build.gradle.kts → Gradle
   検証コマンド: ./gradlew test --tests "{TestClass}"
2. pom.xml → Maven
   検証コマンド: ./mvnw test -Dtest="{TestClass}"
3. package.json → Node.js
   検証コマンド: npm test / npx jest
4. pyproject.toml / setup.py → Python
   検証コマンド: pytest
5. Cargo.toml → Rust
   検証コマンド: cargo test
6. go.mod → Go
   検証コマンド: go test ./...
7. *.csproj / *.sln → .NET
   検証コマンド: dotnet test
8. Makefile → Make
   検証コマンド: make test
```
