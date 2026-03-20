# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [2.0.0] - 2026-03-20

### Changed (BREAKING)
- **ドメイン非依存化**: スキル・エージェント・テンプレート・フックからミッションクリティカル/金融固有のロジックを除去し、あらゆるプロジェクトで使用可能な汎用ワークフローに変更
- ADRテンプレートの「ミッションクリティカル考慮事項」を「追加考慮事項」に変更 — プロジェクトのドメインに応じてカスタマイズ可能なオプショナルセクションに
- テストパターン(`testing-patterns.md`)を汎用パターン（エラーハンドリング、並行アクセス、冪等性、バリデーション、認証・認可）に書き換え
- レビューチェックリスト(`review-checklist.md`)をドメイン非依存の汎用チェックリストに書き換え
- `financial-security-checklist.md` を `security-checklist.md` にリネームし、金融固有の項目を汎用セキュリティ項目に変更
- hooks.jsonからドメイン固有チェック（金額計算、監査ログ、トランザクション等）を除去し、汎用的なコミット前チェックに変更
- PostToolUse (Write|Edit) フックを除去 — ファイル変更ごとのドメイン固有チェックは不要に
- PostToolUseFailure フックを除去

### Added
- カスタマイズガイドをCLAUDE.mdとREADMEに追加 — `references/`、テストパターン、レビューチェックリスト、ADRテンプレートのカスタマイズ方法を文書化
- スキル・エージェントに `references/` 自動参照ロジックを追加 — リファレンスが存在する場合のみ自動的にドメイン固有チェックが有効化
- テストパターンに認証・認可テスト（§6）を追加

## [1.3.0] - 2026-03-20

### Added
- ADRテンプレートをMADR 4.0形式に更新 — Y-Statement形式の決定要約、確認方法（フィットネス関数）、意思決定者フィールド、トレーサビリティセクションを追加
- Vibe ADRトレーサビリティ — コミットメッセージにADR番号を記載、PRにADR・仕様書リンクを追記するルールを追加
- TDAD（Test-Driven Agentic Development）パターン — TDDガイドエージェントに影響範囲分析・リグレッション防止ルール・テスト優先順位を追加
- TDDサイクルにBASELINE（影響範囲分析）ステップを追加 — 変更前のベースラインテスト実行を必須化
- フィットネス関数リファレンス (`references/fitness-functions.md`) — ADR決定事項の自動検証パターン（静的解析、依存関係、テストカバレッジ、セキュリティ、監査ログ）
- CI/CD統合ガイド (`references/ci-cd-integration.md`) — Decision GuardianパターンのGitHub Actions実装例、フィットネス関数パイプライン構成
- スタックPRサイズガイドライン — 200〜400行ルールの明示化、squash merge禁止の注意事項

### Changed
- ADRテンプレートの「結果」セクションをMADR 4.0の「決定の結果」+「確認方法」に変更
- 「銀行固有の考慮事項」を「ミッションクリティカル考慮事項」にリネーム — 金融以外のミッションクリティカルシステムにも対応
- ADR作成スキルの鉄則に「フィットネス関数定義」「Y-Statement形式」を追加
- ADR自己レビューチェックリストにMADR 4.0要素（Y-Statement、確認方法、意思決定者）を追加

## [1.2.0] - 2026-03-14

### Added
- 全エージェントに `color` フィールドを追加 — UI上でどのエージェントが実行中か一目で識別可能に (adr-author: blue, system-designer: purple, implementation-planner: green, tdd-guide: yellow, code-reviewer: cyan, security-reviewer: red)
- レビューエージェントに `background: true` を追加 — code-reviewer, security-reviewer がバックグラウンドで並行実行可能に
- レビューエージェントに `permissionMode: dontAsk` を追加 — 読み取り専用エージェントの権限プロンプトを省略
- `workflow` スキルに `disable-model-invocation: true` を追加 — ユーザーの明示的な `/workflow` 起動を必須化
- `stack-pr-loop` スキルに `disable-model-invocation: true` を追加 — ユーザーの明示的な `/stack-loop` 起動を必須化
- `SubagentStart` フックを追加 — サブエージェント開始時にワークフローコンテキストの読み込みを促す
- PreToolUse (Bash) フックに `statusMessage` を追加 — コミット前チェック中のスピナーテキストを表示

### Fixed
- 全スキルの動的コンテキストブロックで Bash 権限チェックエラーを修正 — `"$d.adflow-context.md"` のクォート付きハイフン文字列を変数代入方式に変更

### Changed
- Stop フックを `prompt` → `agent` タイプにアップグレード — 実際にツール (`git status` 等) を使って検証可能に (model: claude-haiku-4-6)
- TaskCompleted フックを `prompt` → `agent` タイプにアップグレード — 変更差分の実チェックによるドメイン品質検証 (model: claude-haiku-4-6)
- PostToolUse (Write|Edit) フックに `model: "claude-haiku-4-6"` を明示 — 軽量モデルで高速にドメイン品質チェック

## [1.1.0] - 2026-03-14

### Added
- スキルに `context: fork` + `agent` を追加 — 各スキルが専門エージェントのフォークコンテキストで実行されるように (writing-adr, specification, stack-planning, stack-pr-loop)
- レビューエージェントに `disallowedTools: [Write, Edit]` を追加 — code-reviewer, security-reviewer を読み取り専用に制限
- エージェントに `memory: project` を追加 — adr-author, system-designer, implementation-planner がプロジェクトメモリにアクセス可能に
- security-reviewer に `skills: [stack-pr-loop]` を追加 — 実装ループのコンテキストを参照可能に
- `using-adflow` スキルに `user-invocable: false` を設定 — 背景知識として自動的に使用
- `TaskCompleted` フックを追加 — タスク完了時にドメイン品質の最終検証を実行
- `SubagentStop` フックを追加 — サブエージェント完了時に成果物品質を確認
- SessionStart フックに `once: true` を追加 — セッション開始メッセージを1回のみ表示
- PostToolUse (Write|Edit) フックに `statusMessage` を追加 — ドメイン品質チェック中のスピナーテキストを表示

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
