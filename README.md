# bank-aidd

銀行システム開発現場で使える AI 駆動開発プラグイン for Claude Code。

金融特有の要件（トランザクション安全性、監査ログ、排他制御、冪等性、BigDecimal による金額計算）を開発ワークフローの各段階に組み込み、品質の高い銀行システムを効率的に構築します。

## ワークフロー

5 段階のワークフローを提供します。各段階はスラッシュコマンドで個別に実行でき、`/bank-workflow` で一気通貫の実行も可能です。

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
| `/bank-tdd [task-id]` | 銀行特化TDD（RED→GREEN→REFACTOR） | `src/test/java/` + `src/main/java/` |
| `/bank-review [branch]` | 金融セキュリティ対応コードレビュー | レビュー結果 + 修正 |
| `/bank-workflow [feature]` | 全5段階を順番に実行 | 上記すべて |

`/bank-workflow` は `--from=` パラメータで途中のステージから再開できます:

```
/bank-workflow 振込機能 --from=tdd    # Stage 4 (TDD) から再開
```

## インストール

### 前提条件

- [Claude Code](https://claude.com/claude-code) がインストール済みであること
- Java 17+ / Spring Boot 3.x プロジェクトであること（対象プロジェクト）

### 方法 1: プラグインマーケットプレイスから（推奨）

Claude Code 内で以下を実行:

```
/plugin marketplace add dtakamiya/adflow
/plugin install bank-aidd
```

### 方法 2: ローカルインストール

```bash
git clone https://github.com/dtakamiya/adflow.git
```

Claude Code 内で以下を実行:

```
/plugin install --source ./bank-aidd
```

## プロジェクト構造

```
bank-aidd/
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
├── references/                # 銀行開発パターンのリファレンス
│   ├── transaction-patterns.md
│   ├── audit-logging-patterns.md
│   ├── exclusive-control-patterns.md
│   ├── idempotency-patterns.md
│   └── financial-security-checklist.md
└── hooks/                     # 自動リマインダー
    └── hooks.json
```

## 銀行システム開発の必須原則

このプラグインは以下の原則をワークフロー全体に組み込んでいます:

- **金額計算**: 必ず `BigDecimal` を使用（`double`/`float` 禁止）
- **監査ログ**: すべての状態変更に監査イベントを記録
- **トランザクション**: 境界を明示的に設計（`@Transactional` の適切な使用）
- **排他制御**: 楽観ロック（`@Version`）/ 悲観ロック（`SELECT FOR UPDATE`）を考慮
- **冪等性**: リトライ安全な設計を確保
- **データ保護**: PII（個人識別情報）のマスキング・暗号化

## 対象技術スタック

- Java 17+
- Spring Boot 3.x
- Spring Data JPA / MyBatis
- Gradle / Maven
- JUnit 5 + Mockito + AssertJ

## フック（自動リマインダー）

`.java` ファイルの作成・編集時に、銀行システム固有のチェックリマインダーが自動表示されます:

- `@Transactional` の設定確認
- `BigDecimal` 使用の確認
- 監査ログ記録の確認
- 排他制御の考慮確認
- 冪等性の確保確認

## ライセンス

MIT
