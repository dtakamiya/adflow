---
name: design
description: ADRを入力としてシステム設計書を作成する
arguments:
  - name: adr
    description: 対象のADR番号またはファイルパス（省略時はインタラクティブに確認）
    required: false
---

# /design コマンド

ADRを入力として、銀行システムのシステム設計書を作成します。

## 前提条件

- 関連するADRが `docs/adr/` に存在すること

## 実行手順

1. `system-design` スキルに従って設計書を作成する
2. `system-designer` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="opus",
  prompt="
    あなたはSystem Designerエージェントです。agents/system-designer.md の指示に従ってください。

    タスク: システム設計書を作成する
    対象ADR: {adr または ユーザーに確認}

    手順:
    1. 対象のADRファイルを読み込む
    2. プロジェクトの既存コードベースをスキャンする
    3. templates/system-design-template.md を読み込む
    4. 以下のリファレンスを参照する:
       - references/transaction-patterns.md
       - references/audit-logging-patterns.md
       - references/exclusive-control-patterns.md
       - references/idempotency-patterns.md
    5. Mermaid図（コンポーネント図、シーケンス図、ER図）を含む設計書を作成する
    6. docs/design/{kebab-case-name}-design.md に保存する
  "
)
```

3. 設計書の内容をユーザーに確認してもらう
4. 承認後、次のステージ（`/plan`）の実行を提案する
