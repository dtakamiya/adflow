# モデル選択ガイド

adflow のエージェント、スキル、フック定義で使用するモデル選択の指針。

---

## モデルファミリー（Claude 4.5/4.6 世代）

| モデル | モデルID | 特性 | 推奨用途 | コスト |
|--------|----------|------|----------|--------|
| **Opus 4.6** | `claude-opus-4-6` | 最も深い推論能力、1Mコンテキスト | アーキテクチャ設計、ADR作成、セキュリティレビュー | 高 |
| **Sonnet 4.6** | `claude-sonnet-4-6` | コーディング性能と速度のバランス | TDD実装、コードレビュー、仕様書作成 | 中 |
| **Haiku 4.5** | `claude-haiku-4-5-20251001` | 軽量・高速・低コスト | フック検証、定型チェック、ステータス確認 | 低 |

> **Note**: エージェント定義の `model:` フィールドでは短縮名（`opus`, `sonnet`, `haiku`）を使用する。これらは自動的に最新のモデルIDにマッピングされる。

## adflow エージェントのモデル割り当て

| エージェント | モデル | 理由 |
|-------------|--------|------|
| `adr-author` | opus | 設計判断は深い推論が必要 |
| `system-designer` | opus | コンポーネント設計・API設計の複雑な判断 |
| `implementation-planner` | sonnet | タスク分解は構造的だがコスト効率が重要 |
| `tdd-guide` | sonnet | 実装量が多くコスト効率が重要（maxTurns=50） |
| `code-reviewer` | sonnet | パターン認識中心、コスト効率重視 |
| `security-reviewer` | opus | セキュリティは見逃しのコストが高い |

## スキルの effort レベル

スキルの `effort` フロントマターは、推論の深さとコストのバランスを制御する:

| スキル | effort | 理由 |
|--------|--------|------|
| `/adr` | high | 複数の選択肢比較と深いトレードオフ分析が必要 |
| `/spec` | high | コンポーネント設計・API設計の複雑な判断 |
| `/stack-plan` | medium | 構造的なタスク分解、パターンベース |
| `/stack-loop` | high | TDDサイクルの厳密な実行と品質判断 |
| `/workflow` | high | 全フェーズのオーケストレーション |
| `/systematic-debugging` | high | 根本原因分析に深い推論が必要 |
| `/verification-before-completion` | medium | 定型的な検証ステップ |
| `/using-git-worktrees` | low | 手順が明確な定型タスク |
| `/dispatching-parallel-agents` | medium | タスク分析と並列化判断 |

## フック（自動実行）のモデル選択

フックは頻繁に実行されるため、**Haiku を標準とする**:

- `Stop` フック: Haiku（git status 確認などの定型処理）
- `TaskCompleted` フック: Haiku（差分確認などの定型処理）

## コスト最適化のヒント

### コンテキストウィンドウ管理

- 大規模リファクタリングや複数ファイル変更時は、コンテキストウィンドウの残り20%に入る前にタスクを区切る
- `PreCompact` フックで状態を保存し、コンパクション後に復元可能にする

### エージェントの使い分け

- **バックグラウンドエージェント**（`code-reviewer`, `security-reviewer`）は `permissionMode: dontAsk` で非同期実行し、メインフローをブロックしない
- **複数の独立したタスク**は並列エージェントで実行する（例: セキュリティレビューとコードレビューの同時実行）
- **Worktree 分離**: `isolation: worktree` でサブエージェントに独立したgitワーキングコピーを割り当て、ファイルコンフリクトを防止する

### エージェントチーム（実験的機能）

大規模な並列作業には Agent Teams を検討する:
- 各チームメイトが独自のコンテキストウィンドウとセッションを持つ
- 共有タスクリストで協調作業が可能
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` で有効化

### Extended Thinking の活用

- 複雑なアーキテクチャ判断や、複数のトレードオフを比較する場面では Extended Thinking を有効にする
- 定型的なコーディングタスクでは無効にしてスループットを優先する

### エージェントメモリの活用

adflow の全エージェントは `memory: project` を使用し、セッション間で学習を蓄積する:
- 発見したパターンやビルドコマンドを自動保存
- プロジェクト固有のコーディング規約を学習
- `.claude/agent-memory/<agent-name>/` に保存され、gitで共有可能
