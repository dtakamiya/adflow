---
name: plan
description: 設計書を入力としてTDD対応の実装計画書を作成する
arguments:
  - name: design
    description: 対象の設計書ファイルパス（省略時はインタラクティブに確認）
    required: false
---

# /plan コマンド

設計書を入力として、TDD対応の実装計画書を作成します。

## 前提条件

- 関連する設計書が `docs/design/` に存在すること
- 関連するADRが `docs/adr/` に存在すること

## 実行手順

1. `implementation-planning` スキルに従って実装計画書を作成する
2. `implementation-planner` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="opus",
  prompt="
    あなたはImplementation Plannerエージェントです。agents/implementation-planner.md の指示に従ってください。

    タスク: 実装計画書を作成する
    対象設計書: {design または ユーザーに確認}

    手順:
    1. 対象の設計書とADRを読み込む
    2. プロジェクトのビルドシステム（Gradle/Maven）を特定する
    3. templates/implementation-plan-template.md を読み込む
    4. 設計書のコンポーネントをPhaseに分割する
    5. 各PhaseをTDDステップ付きのTaskに分解する
    6. 銀行固有チェックポイントを挿入する
    7. docs/plans/{kebab-case-name}-plan.md に保存する
  "
)
```

3. 実装計画書の内容をユーザーに確認してもらう
4. 承認後、次のステージ（`/bank-tdd`）の実行を提案する
