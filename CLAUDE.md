# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

adflow - ADR駆動のAI駆動開発ワークフロープラグイン

ADR（Architecture Decision Record）を起点に、設計から実装・検証まで一気通貫で進める5段階ワークフローを提供:
1. `/adr` - Architecture Decision Record 作成
2. `/design` - システム設計書作成
3. `/plan` - 実装計画書作成
4. `/bank-tdd` - TDD実装（RED→GREEN→REFACTOR）
5. `/bank-review` - コードレビュー（金融セキュリティチェック付き）

統合コマンド: `/bank-workflow` で全5段階を順番に実行

## ドメイン特化チェック（銀行・金融）

TDDとコードレビューに以下の品質チェックをビルトイン:

- 金額計算は必ず `BigDecimal` を使用（`double`/`float` 禁止）
- すべての状態変更に監査ログを記録
- トランザクション境界を明示的に設計（`@Transactional` の適切な使用）
- 排他制御を考慮（楽観ロック `@Version` / 悲観ロック `SELECT FOR UPDATE`）
- 冪等性を確保（リトライ安全な設計）
- PII（個人識別情報）のマスキング・暗号化

## プラグイン構造

- `skills/` - 各ワークフローステージのスキル定義
- `agents/` - 専門エージェント定義
- `commands/` - スラッシュコマンド定義
- `templates/` - ドキュメントテンプレート
- `references/` - 開発パターンのリファレンス
- `hooks/` - 自動リマインダー設定

## 技術スタック（対象プロジェクト）

- Java 17+
- Spring Boot 3.x
- Spring Data JPA / MyBatis
- Gradle / Maven
- JUnit 5 + Mockito + AssertJ
