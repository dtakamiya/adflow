# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
