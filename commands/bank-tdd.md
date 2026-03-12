---
name: bank-tdd
description: 銀行特化TDD（RED→GREEN→REFACTOR）を実行する
arguments:
  - name: task
    description: 実装計画書のTask番号（省略時はインタラクティブに確認）
    required: false
---

# /bank-tdd コマンド

銀行システム向けのTDD（テスト駆動開発）を実行します。

## 前提条件

- 実装計画書が `docs/plans/` に存在すること（推奨）
- ビルドが通る状態であること

## 実行手順

1. `bank-tdd` スキルに従ってTDDを実行する
2. `bank-tdd-guide` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="sonnet",
  prompt="
    あなたはBank TDD Guideエージェントです。agents/bank-tdd-guide.md の指示に従ってください。

    タスク: TDDサイクルを実行する
    対象Task: {task または ユーザーに確認}

    手順:
    1. 実装計画書から対象Taskを特定する
    2. skills/bank-tdd/bank-testing-patterns.md を参照する
    3. RED: テストを作成し、失敗を確認する
    4. GREEN: 最小実装でテストを通す
    5. REFACTOR: コードを改善し、テストが通ることを確認する
    6. 銀行固有テスト（BigDecimal、トランザクション、監査ログ等）を組み込む
  "
)
```

3. 各TDDサイクルの結果を報告する
4. すべてのTaskが完了したら、次のステージ（`/bank-review`）の実行を提案する
