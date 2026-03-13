# adflow

ADR駆動のAI駆動開発ワークフロープラグイン for Claude Code。

アーキテクチャ決定（ADR）を起点に、仕様書→スタックPR計画→実装ループ（TDD・自動レビュー・PR作成）までを一気通貫で進める8段階ワークフローを提供します。各ステージの成果物が次のステージの入力となり、設計と実装の一貫性を保ちます。

ミッションクリティカルシステム向けのドメイン品質チェック（トランザクション安全性、監査ログ、排他制御、冪等性、高精度小数型）をビルトインで提供しています。

## ワークフロー

8段階のワークフローを提供します。各段階はスラッシュコマンドで個別に実行でき、`/workflow` で一気通貫の実行も可能です。

```
/adr → /spec → /stack-plan → /stack-loop
  ↓       ↓         ↓            ↓
 ADR    仕様書   スタックPR計画  実装ループ(TDD/Review/PR)
```

### コマンド一覧

| コマンド | 説明 | 成果物 |
|---------|------|--------|
| `/adr [title]` | Architecture Decision Record 作成とAI自己レビュー | `docs/NNNN-title/01-adr.md` |
| `/spec [adr-number]` | 仕様書作成（Mermaid図・API仕様・データモデル）とAI自己レビュー | `docs/NNNN-title/02-spec.md` |
| `/stack-plan [spec-name]` | スタックPR実装計画書作成（PR分割・Task定義）とAI自己レビュー | `docs/NNNN-title/03-plans.md` |
| `/stack-loop [feature]` | 実装ループ（ブランチ作成→TDD→ローカル検証→自己レビュー→PR作成） | ブランチ + コミット + PR |
| `/workflow [feature]` | 全8段階を順番に実行 | 上記すべて |

`/workflow` は `--from=` パラメータで途中のステージから再開できます:

```
/workflow 振込機能 --from=stack-loop    # Stage 8 (実装ループ) から再開
```

## インストール

### 前提条件

- [Claude Code](https://claude.com/claude-code) がインストール済みであること

### 方法 1: GitHubから直接インストール（推奨）

Claude Code 内で以下を実行:

```
/plugin install --source https://github.com/dtakamiya/adflow.git
```

### 方法 2: ローカルインストール

```bash
git clone https://github.com/dtakamiya/adflow.git
```

Claude Code 内で以下を実行:

```
/plugin install --source ./adflow
```

## ワークフロー間のドキュメント引き継ぎ

各ステージの成果物は `docs/{NNNN-feature-name}/` ディレクトリにまとめて管理されます。コンテキストファイル（`.adflow-context.md`）がワークフローの状態を記録し、`/clear` してもドキュメントを引き継いで作業を続けられます。

### ディレクトリ構造

```
docs/
├── 0001-transfer-service/         ← 機能ごとのディレクトリ
│   ├── .adflow-context.md         ← ワークフロー状態管理
│   ├── 01-adr.md                  ← ADR
│   ├── 02-spec.md                 ← 仕様書
│   └── 03-plans.md                ← スタックPR計画
├── 0002-account-management/       ← 別の機能
│   ├── .adflow-context.md
│   ├── 01-adr.md
│   └── ...
```

### 使い方

```bash
# 1. ADRを作成
/adr 振込機能

# 2. /clear してもOK — コンテキストが保持される
/clear

# 3. 引数なしで次のステージを実行 — 自動的に前段の成果物を検出
/spec

# 4. さらに /clear しても続行可能
/clear
/stack-plan
```

各コマンドは引数なしで実行すると、`.adflow-context.md` をスキャンして次に進むべきワークフローを自動検出します。複数の機能が並行して進行中の場合は、一覧を提示してユーザーに選択してもらいます。

## ワークフローの特徴

### ADR駆動

すべてはアーキテクチャ決定の記録（ADR）から始まります。ADRで「なぜその設計にしたか」を明文化し、仕様書・スタックPR計画・テスト・レビューまで一貫した意思決定の連鎖を作ります。

### 承認ゲート

各ステージの完了時に承認ゲートを設けています。成果物をユーザーが確認・承認してから次のステージに進むため、手戻りを最小化します。

### TDD組み込み

実装計画にはTDDステップ（RED→GREEN→REFACTOR）が組み込まれており、テストファーストで実装を進めます。

### ドメイン品質チェック（ミッションクリティカルシステム向け）

TDDとコードレビューには、ミッションクリティカルシステム向けの品質チェックがビルトインされています:

- **金額計算**: 高精度小数型の使用（浮動小数点型禁止）
- **監査ログ**: すべての状態変更に監査イベントを記録
- **トランザクション**: 境界の明示的な設計
- **排他制御**: 楽観ロック / 悲観ロック
- **冪等性**: リトライ安全な設計
- **データ保護**: PII（個人識別情報）のマスキング・暗号化

### 対応ビルドシステム

プロジェクトのビルドシステムを自動検出して適切なコマンドを使用します:

| ビルドシステム | 検出ファイル | テストコマンド |
|-------------|-----------|-------------|
| Gradle | `build.gradle` / `build.gradle.kts` | `./gradlew test` |
| Maven | `pom.xml` | `./mvnw test` |
| Node.js | `package.json` | `npm test` |
| Python | `pyproject.toml` / `setup.py` | `pytest` |
| Rust | `Cargo.toml` | `cargo test` |
| Go | `go.mod` | `go test ./...` |
| .NET | `*.csproj` / `*.sln` | `dotnet test` |
| Make | `Makefile` | `make test` |

## プロジェクト構造

```
adflow/
├── .claude-plugin/
│   └── plugin.json            # プラグインマニフェスト
├── CLAUDE.md                  # プロジェクト指示書
├── skills/                    # スキル定義（ワークフローの中核ロジック）
│   ├── writing-adr/
│   │   └── SKILL.md
│   ├── specification/
│   │   └── SKILL.md
│   ├── stack-planning/
│   │   └── SKILL.md
│   ├── stack-pr-loop/
│   │   ├── SKILL.md
│   │   └── testing-patterns.md
│   └── workflow/
│       └── SKILL.md
├── agents/                    # 専門エージェント定義
│   ├── adr-author.md
│   ├── system-designer.md
│   ├── implementation-planner.md
│   ├── tdd-guide.md
│   ├── code-reviewer.md
│   └── security-reviewer.md
├── templates/                 # ドキュメントテンプレート
│   ├── adr-template.md
│   ├── adflow-context-template.md
│   ├── system-design-template.md
│   └── implementation-plan-template.md
├── references/                # 開発パターンのリファレンス
│   ├── transaction-patterns.md
│   ├── audit-logging-patterns.md
│   ├── exclusive-control-patterns.md
│   ├── idempotency-patterns.md
│   └── financial-security-checklist.md
└── hooks/                     # 自動リマインダー
    └── hooks.json
```

## ライセンス

MIT
