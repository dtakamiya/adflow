# adflow

ADR駆動のAI駆動開発ワークフロープラグイン for Claude Code。

アーキテクチャ決定（ADR）を起点に、設計書→実装計画→TDD→コードレビューまでを一気通貫で進める5段階ワークフローを提供します。各ステージの成果物が次のステージの入力となり、設計と実装の一貫性を保ちます。

銀行・金融ドメイン向けの品質チェック（トランザクション安全性、監査ログ、排他制御、冪等性、BigDecimal）をビルトインで提供しています。

## ワークフロー

5段階のワークフローを提供します。各段階はスラッシュコマンドで個別に実行でき、`/bank-workflow` で一気通貫の実行も可能です。

```
/adr → /design → /plan → /bank-tdd → /bank-review
 ↓        ↓         ↓          ↓            ↓
ADR    設計書    実装計画   テスト+実装    レビュー
```

### コマンド一覧

| コマンド | 説明 | 成果物 |
|---------|------|--------|
| `/adr [title]` | Architecture Decision Record 作成 | `docs/adr/NNN-title.md` |
| `/design [adr-number]` | システム設計書作成（Mermaid図・API仕様・データモデル） | `docs/design/name-design.md` |
| `/plan [design-name]` | TDD対応の実装計画書作成（Phase分割・Task定義） | `docs/plans/name-plan.md` |
| `/bank-tdd [task-id]` | TDD実装（RED→GREEN→REFACTOR） | `src/test/java/` + `src/main/java/` |
| `/bank-review [branch]` | コードレビュー（金融セキュリティチェック付き） | レビュー結果 + 修正 |
| `/bank-workflow [feature]` | 全5段階を順番に実行 | 上記すべて |

`/bank-workflow` は `--from=` パラメータで途中のステージから再開できます:

```
/bank-workflow 振込機能 --from=tdd    # Stage 4 (TDD) から再開
```

## インストール

### 前提条件

- [Claude Code](https://claude.com/claude-code) がインストール済みであること

### 方法 1: プラグインマーケットプレイスから（推奨）

Claude Code 内で以下を実行:

```
/plugin marketplace add dtakamiya/adflow
/plugin install adflow
```

### 方法 2: ローカルインストール

```bash
git clone https://github.com/dtakamiya/adflow.git
```

Claude Code 内で以下を実行:

```
/plugin install --source ./adflow
```

## ワークフローの特徴

### ADR駆動

すべてはアーキテクチャ決定の記録（ADR）から始まります。ADRで「なぜその設計にしたか」を明文化し、設計書・実装計画・テスト・レビューまで一貫した意思決定の連鎖を作ります。

### 承認ゲート

各ステージの完了時に承認ゲートを設けています。成果物をユーザーが確認・承認してから次のステージに進むため、手戻りを最小化します。

### TDD組み込み

実装計画にはTDDステップ（RED→GREEN→REFACTOR）が組み込まれており、テストファーストで実装を進めます。

### ドメイン特化チェック（銀行・金融）

TDDとコードレビューには、金融ドメイン向けの品質チェックがビルトインされています:

- **金額計算**: `BigDecimal` 使用の強制（`double`/`float` 禁止）
- **監査ログ**: すべての状態変更に監査イベントを記録
- **トランザクション**: 境界の明示的な設計（`@Transactional`）
- **排他制御**: 楽観ロック（`@Version`）/ 悲観ロック（`SELECT FOR UPDATE`）
- **冪等性**: リトライ安全な設計
- **データ保護**: PII（個人識別情報）のマスキング・暗号化

## プロジェクト構造

```
adflow/
├── .claude-plugin/
│   ├── plugin.json            # プラグインマニフェスト
│   └── marketplace.json       # マーケットプレイス定義
├── CLAUDE.md                  # プロジェクト指示書
├── skills/                    # スキル定義（ワークフローの中核ロジック）
│   ├── writing-adr/
│   │   └── SKILL.md
│   ├── system-design/
│   │   └── SKILL.md
│   ├── implementation-planning/
│   │   └── SKILL.md
│   ├── bank-tdd/
│   │   ├── SKILL.md
│   │   └── bank-testing-patterns.md
│   ├── bank-code-review/
│   │   ├── SKILL.md
│   │   └── bank-review-checklist.md
│   └── bank-workflow/
│       └── SKILL.md
├── commands/                  # スラッシュコマンド定義
│   ├── adr.md
│   ├── design.md
│   ├── plan.md
│   ├── bank-tdd.md
│   ├── bank-review.md
│   └── bank-workflow.md
├── agents/                    # 専門エージェント定義
│   ├── adr-author.md
│   ├── system-designer.md
│   ├── implementation-planner.md
│   ├── bank-tdd-guide.md
│   ├── bank-code-reviewer.md
│   └── bank-security-reviewer.md
├── templates/                 # ドキュメントテンプレート
│   ├── adr-template.md
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

## 対象技術スタック

現在、以下の技術スタックに対応しています:

- Java 17+
- Spring Boot 3.x
- Spring Data JPA / MyBatis
- Gradle / Maven
- JUnit 5 + Mockito + AssertJ

## フック（自動リマインダー）

`.java` ファイルの作成・編集時に、ドメイン固有のチェックリマインダーが自動表示されます。

## ライセンス

MIT
