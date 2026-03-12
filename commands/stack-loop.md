---
name: stack-loop
description: PRスタックの計画書に基づき、PRごとの実装ループ（ブランチ作成→TDD→ローカル検証→自己レビュー→PR作成）を反復実行します。
arguments:
  - name: feature-name
    description: 対象の機能名（省略時は現在のコンテキストから判別）
    required: false
---

- `feature-name`: 対象の機能名（省略時は現在のコンテキストから判別）
    required: false
---

# /stack-loop - スタックPR 実装ループ

## 概要

PRスタックの計画書に基づき、PRごとの実装ループ（ブランチ作成→TDD→ローカル検証→自己レビュー→PR作成）を反復実行します。

## 内部動作

このコマンドは `stack-pr-loop` スキルを呼び出します。

## 前提条件

- 実装計画書が `docs/{dir-name}/03-plans.md` に存在すること（推奨）
- ビルドが通る状態であること

## 実行手順

1. ```bash
/stack-loop transfer-service
```
2. `stack-pr-loop` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="sonnet",
  prompt="
    あなたはStack PR Loopエージェントです。agents/stack-pr-loop.md の指示に従ってください。

    タスク: 実装ループを実行する
    対象Task: {task または 自動検出}

    手順:
    1. 最初に docs/*/.adflow-context.md をスキャンして、アクティブなワークフローの前段成果物を自動検出してください
    2. 「現在のステージ」が stack-loop のコンテキストファイルを検索する
    3. 該当が1件ならその計画書を自動選択、複数なら一覧提示して選択してもらう。また該当する機能名ディレクトリ名（{dir-name}）を控える。
    4. コンテキストファイルから計画書と仕様書のファイルパスを取得し、両方を Read ツールで読み込む
    5. 実装計画書から対象Taskを特定する
    6. skills/stack-pr-loop/testing-patterns.md を参照する
    7. 各PRに対して、以下のループを自動的に回します：

    1. **ブランチ準備**: 前回のPRを親として新しいブランチを派生
    2. **RED**: 失敗するテスト（ドメイン品質アサーションを含む）の記述
    3. **GREEN**: テストを通す最小限の実装
    4. **REFACTOR**: コード整理
    5. **検証**: Lint、型チェック、テスト全体のローカル検証
    6. **自己レビュー**: AI自身によるドメイン要件と仕様のチェック
    7. **コミット・PR作成**: 作業内容のコミットとPR作成案内
    8. 次のPRへの移行確認を組み込む
  "
)
```

3. 各反復の結果を報告する
4. すべてのTaskが完了したら終了を提案する

3. 各TDDサイクルの結果を報告する
4. すべてのTaskが完了したら、次のステージ（`/review`）の実行を提案する

