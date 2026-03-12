# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

adflow - ADR駆動のAI駆動開発ワークフロープラグイン

ADR（Architecture Decision Record）を起点に、設計から実装・検証まで一気通貫で進める5段階ワークフローを提供:
1. `/workflow` (オーケストレーション) - コンテキスト収集と要件分析
2. `/adr` - ブレインストーミングとADR作成・自己レビュー
3. `/spec` - 仕様書の作成と自己レビュー
4. `/stack-plan` - スタックPR計画の作成と自己レビュー
5. `/stack-loop` - スタックPR 実装ループ (ブランチ作成, TDD, 自己レビュー, PR作成)

統合コマンド: `/workflow` で全フェーズを順番に実行

## ドメイン品質チェック（ミッションクリティカルシステム向け）

TDDとコードレビューに以下の品質チェックをビルトイン:

- 金額計算は必ず高精度小数型を使用（浮動小数点型禁止）
- すべての状態変更に監査ログを記録
- トランザクション境界を明示的に設計
- 排他制御を考慮（楽観ロック / 悲観ロック）
- 冪等性を確保（リトライ安全な設計）
- PII（個人識別情報）のマスキング・暗号化

## プラグイン構造

- `skills/` - 各ワークフローステージのスキル定義
- `agents/` - 専門エージェント定義
- `commands/` - スラッシュコマンド定義
- `templates/` - ドキュメントテンプレート
- `references/` - 開発パターンのリファレンス
- `hooks/` - 自動リマインダー設定
