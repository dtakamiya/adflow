# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] - 2026-03-14

### Added
- ADR駆動の5段階ワークフロー (`/workflow`, `/adr`, `/spec`, `/stack-plan`, `/stack-loop`)
- 6つの専門エージェント (adr-author, system-designer, implementation-planner, tdd-guide, code-reviewer, security-reviewer)
- ミッションクリティカルシステム向けドメイン品質チェック (金額計算・監査ログ・トランザクション・排他制御・冪等性・PII保護)
- セッション開始時の自動リマインダー (hooks)
- コミット前・ファイル変更時の品質チェックフック
- ワークフローコンテキスト引き継ぎ (.adflow-context.md)
- ドキュメントテンプレート (ADR, 仕様書, スタックPR計画)
- 開発パターンリファレンス
- MIT LICENSE
- marketplace.json によるプラグイン配布対応
