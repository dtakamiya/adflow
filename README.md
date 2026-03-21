# adflow

![Version](https://img.shields.io/badge/version-2.0.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)

ADR駆動のAI駆動開発ワークフロープラグイン for Claude Code。

アーキテクチャ決定（ADR）を起点に、仕様書→スタックPR計画→実装ループ（TDD・自動レビュー・PR作成）までを一気通貫で進めるワークフローを提供します。各ステージの成果物が次のステージの入力となり、設計と実装の一貫性を保ちます。

あらゆるドメインのプロジェクトに対応。`references/` やテンプレートをカスタマイズすることで、プロジェクト固有の品質チェックを組み込めます。

### v2.0.0 の新機能

- **ドメイン非依存化**: スキル・エージェント・テンプレートを汎用化。あらゆるプロジェクトで使用可能に
- **カスタマイズポイントの明確化**: `references/`、テストパターン、レビューチェックリスト、ADRテンプレートの追加考慮事項セクションでプロジェクト固有のカスタマイズが可能
- **リファレンスの自動参照**: `references/` 配下にファイルが存在する場合、スキルとエージェントが自動的に参照

## クイックスタート（3分で体験）

```bash
# 1. マーケットプレースを追加してプラグインをインストール
/plugin marketplace add dtakamiya/adflow
/plugin install adflow@dtakamiya/adflow

# 2. ワークフローを開始（例: ユーザー認証機能）
/workflow ユーザー認証

# 3. あとはadflowの案内に従うだけ！
#    ADR作成 → 仕様書作成 → PR計画 → TDD実装
#    各ステージで承認ゲートがあり、確認してから次に進みます
```

## ワークフロー全体像

```
┌─────────┐    ┌─────────┐    ┌──────────────┐    ┌─────────────┐
│  /adr   │───→│  /spec  │───→│ /stack-plan  │───→│ /stack-loop │
│         │    │         │    │              │    │             │
│  ADR    │    │  仕様書  │    │  PR計画書     │    │ TDD実装     │
│  作成   │    │  作成    │    │  作成         │    │ ループ      │
└─────────┘    └─────────┘    └──────────────┘    └─────────────┘
     ↑              ↑               ↑                   ↑
  承認ゲート      承認ゲート       承認ゲート          自己レビュー
```

統合コマンド `/workflow` で全フェーズを一気通貫実行。`--from=` で途中再開も可能。

## ワークフロー

各段階はスラッシュコマンドで個別に実行でき、`/workflow` で一気通貫の実行も可能です。

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
| `/workflow [feature]` | 全フェーズを順番に実行 | 上記すべて |

`/workflow` は `--from=` パラメータで途中のステージから再開できます:

```
/workflow ユーザー認証 --from=stack-loop    # 実装ループから再開
```

## インストール

### 前提条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) がインストール済みであること

### 方法 1: マーケットプレースからインストール（推奨）

Claude Code 内で以下を実行:

```bash
# マーケットプレースを追加
/plugin marketplace add dtakamiya/adflow

# プラグインをインストール
/plugin install adflow@dtakamiya/adflow
```

スコープを指定してインストールすることもできます:

```bash
/plugin install adflow@dtakamiya/adflow --scope project   # プロジェクトスコープ（チーム共有）
/plugin install adflow@dtakamiya/adflow --scope local      # ローカルスコープ（gitignored）
```

### 方法 2: ローカル開発・テスト用

```bash
git clone https://github.com/dtakamiya/adflow.git
claude --plugin-dir ./adflow
```

### 方法 3: settings.json でチームマーケットプレースを設定

`.claude/settings.json`（プロジェクトスコープ）に追加してチーム全体で共有:

```json
{
  "extraKnownMarketplaces": {
    "dtakamiya/adflow": {
      "source": {
        "source": "github",
        "repo": "dtakamiya/adflow"
      }
    }
  },
  "enabledPlugins": [
    "adflow@dtakamiya/adflow"
  ]
}
```

## カスタマイズ

adflow はドメイン非依存の汎用ワークフローです。プロジェクトのドメインに合わせて以下をカスタマイズできます:

### リファレンスの追加

`references/` ディレクトリにプロジェクト固有のリファレンスを追加すると、スキルとエージェントが自動的に参照します。同梱のリファレンス例:

- `transaction-patterns.md` — トランザクション設計パターン
- `audit-logging-patterns.md` — 監査ログ設計パターン
- `exclusive-control-patterns.md` — 排他制御パターン
- `idempotency-patterns.md` — 冪等性パターン
- `security-checklist.md` — セキュリティチェックリスト

### テストパターン・レビューチェックリスト

- `skills/stack-loop/testing-patterns.md` — TDDで参照されるテストパターン
- `skills/stack-loop/review-checklist.md` — AI自己レビューで使用するチェックリスト

### ADRテンプレート

`templates/adr-template.md` の「追加考慮事項」セクションをドメインに合わせて編集できます。

## 開発・テスト

### ローカルテスト

プラグインの開発中は `--plugin-dir` オプションでローカルディレクトリを指定してテストできます:

```bash
claude --plugin-dir /path/to/adflow
```

### プラグインバリデーション

```bash
claude plugin validate .
```

### プラグイン再読込

```
/reload-plugins
```

## ワークフロー間のドキュメント引き継ぎ

各ステージの成果物は `docs/{NNNN-feature-name}/` ディレクトリにまとめて管理されます。コンテキストファイル（`.adflow-context.md`）がワークフローの状態を記録し、`/clear` してもドキュメントを引き継いで作業を続けられます。

### ディレクトリ構造

```
docs/
├── 0001-user-authentication/       ← 機能ごとのディレクトリ
│   ├── .adflow-context.md           ← ワークフロー状態管理
│   ├── 01-adr.md                    ← ADR
│   ├── 02-spec.md                   ← 仕様書
│   └── 03-plans.md                  ← スタックPR計画
├── 0002-notification-service/       ← 別の機能
│   ├── .adflow-context.md
│   ├── 01-adr.md
│   └── ...
```

## ワークフローの特徴

### ADR駆動（Vibe ADR）

すべてはアーキテクチャ決定の記録（ADR）から始まります。MADR 4.0形式で「なぜその設計にしたか」を明文化し、Y-Statement形式で決定を要約します。各ADRにはフィットネス関数（自動検証方法）を定義し、コミット・PRからADRへの双方向リンクで設計意図のトレーサビリティを確保します。

### 承認ゲート

各ステージの完了時に承認ゲートを設けています。成果物をユーザーが確認・承認してから次のステージに進むため、手戻りを最小化します。

### TDD組み込み（TDAD対応）

実装計画にはTDDステップ（BASELINE→RED→GREEN→REFACTOR）が組み込まれています。TDAD（Test-Driven Agentic Development）パターンにより、AIエージェントが変更前にベースラインテストを実行し、影響範囲の依存テストを特定してからTDDサイクルに入ります。

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
│   ├── adr/
│   │   └── SKILL.md
│   ├── spec/
│   │   └── SKILL.md
│   ├── stack-plan/
│   │   └── SKILL.md
│   ├── stack-loop/
│   │   ├── SKILL.md
│   │   ├── testing-patterns.md  ← カスタマイズ可能
│   │   └── review-checklist.md  ← カスタマイズ可能
│   ├── using-adflow/
│   │   └── SKILL.md
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
│   ├── adr-template.md          ← カスタマイズ可能
│   ├── adflow-context-template.md
│   ├── system-design-template.md
│   └── implementation-plan-template.md
├── references/                # 開発パターンのリファレンス ← 追加・編集可能
│   ├── transaction-patterns.md
│   ├── audit-logging-patterns.md
│   ├── exclusive-control-patterns.md
│   ├── idempotency-patterns.md
│   ├── security-checklist.md
│   ├── fitness-functions.md
│   └── ci-cd-integration.md
└── hooks/                     # 自動リマインダー
    └── hooks.json
```

## 対応環境

- **Claude Code**: 必須（プラグインシステムを使用）
- **Git**: 必須（ブランチ管理・PR作成に使用）
- **GitHub CLI (`gh`)**: 推奨（PR作成の自動化に使用）
- **OS**: macOS, Linux, Windows (WSL)

## FAQ

### Q: 途中で `/clear` しても大丈夫？
A: はい。各ステージの成果物は `docs/{機能名}/` に保存され、コンテキストファイル（`.adflow-context.md`）がワークフロー状態を管理します。次回のコマンド実行時に自動的に前回の状態を引き継ぎます。

### Q: ADRなしで仕様書を作れる？
A: `/spec` は引数でADRを指定するか、自動検出します。ADRがない場合は `/adr` の実行を提案します。ワークフローの一貫性のため、ADRの作成を推奨します。

### Q: 既存プロジェクトにも使える？
A: はい。ビルドシステム（Gradle, Maven, npm, Python, Rust, Go, .NET, Make）を自動検出し、既存のプロジェクト構造に合わせて動作します。

### Q: プロジェクト固有の品質チェックを追加するには？
A: `references/` にリファレンスファイルを追加し、`testing-patterns.md` や `review-checklist.md` を編集してください。スキルとエージェントが自動的に参照します。

### Q: 複数の機能を並行して開発できる？
A: はい。各機能は独立したディレクトリ（`docs/{NNNN-機能名}/`）で管理されるため、複数のワークフローを並行して進められます。

## ライセンス

MIT
