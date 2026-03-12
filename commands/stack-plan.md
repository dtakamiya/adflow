---
name: stack-plan
description: 仕様書を入力として、スタックPR（積み上げ型の小さなPR）の実装計画書を作成します。
arguments:
  - name: spec
    description: 対象の仕様書名または機能名（省略時は現在のコンテキストから判別）
    required: false
---

# /stack-plan - スタックPR計画作成

## 概要

仕様書を入力として、スタックPR（積み上げ型の小さなPR）の実装計画書を作成します。

## 内部動作

このコマンドは `stack-planning` スキルを呼び出します。

## 前提条件

- 関連する仕様書が `docs/{dir-name}/02-spec.md` に存在すること
- 関連するADRが `docs/{dir-name}/01-adr.md` に存在すること

## 実行手順

1. `stack-planning` スキルに従って実装計画書を作成する
2. `implementation-planner` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="opus",
  prompt="
    あなたはStack PR Plannerエージェントです。agents/stack-pr-planner.md の指示に従ってください。

    タスク: スタックPR実装計画書を作成する
    対象仕様書: {spec または 自動検出}

    手順:
    1. 最初に docs/*/.adflow-context.md をスキャンして、アクティブなワークフローの前段成果物を自動検出してください
    2. 「現在のステージ」が stack-plan のコンテキストファイルを検索する
    3. 該当が1件ならその仕様書を自動選択、複数なら一覧提示して選択してもらう。また該当する機能名ディレクトリ名（{dir-name}）を控える。
    4. コンテキストファイルから仕様書とADRのファイルパスを取得し、両方を Read ツールで全文読み込む
    5. プロジェクトのビルドシステムを特定する
    6. templates/stack-pr-plan-template.md を読み込む
    7. 仕様書のコンポーネント以下を含むスタックPR実装計画書を生成します：

- PRごとの論理的な分割と依存関係
- PR内のTDDステップ（RED/GREEN/REFACTOR）付きのTask定義
- ドメイン品質チェックポイント（監査ログ、トランザクション等の実装確認）
- ビルドシステムのテストコマンド
    10. docs/{dir-name}/03-plans.md に保存する
    11. 承認後、docs/{dir-name}/.adflow-context.md を更新する
        - 計画書ファイルパスを記録、ステータス「完了」、現在のステージを stack-loop に更新
    12. 「次のステージ /stack-loop を実行するか、/clear して後で再開できます」と案内して終了する。（※ /workflow の一環として実行されている場合を除き、自律的に次のコマンドを実行してはいけません。必ず人間の明示的なコマンド入力または承認を待つこと）
  "
)
```

3. 実装計画書の内容をユーザーに確認してもらう
4. 承認後、コンテキストファイルを更新し、次のステージ（`/stack-loop`）の実行を提案する
