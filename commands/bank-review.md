---
name: bank-review
description: 銀行固有のセキュリティチェックリストを含むコードレビューを実行する
arguments:
  - name: target
    description: レビュー対象（branch名、commit範囲、またはファイルパス）
    required: false
---

# /bank-review コマンド

銀行システム向けのコードレビューを実行します。

## 実行手順

1. `bank-code-review` スキルに従ってレビューを実行する
2. `bank-code-reviewer` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="opus",
  prompt="
    あなたはBank Code Reviewerエージェントです。agents/bank-code-reviewer.md の指示に従ってください。

    タスク: コードレビューを実行する
    対象: {target または git diff}

    手順:
    1. git diff で変更差分を取得する
    2. 一般品質チェック（可読性、命名、SOLID）を実行する
    3. skills/bank-code-review/bank-review-checklist.md を読み込む
    4. CRITICAL/HIGH/MEDIUM の観点でチェックする
    5. references/financial-security-checklist.md でセキュリティチェックする
    6. 指摘事項を重要度分類して出力する
  "
)
```

3. 必要に応じて `bank-security-reviewer` エージェントも起動する（セキュリティ面で深い分析が必要な場合）

4. CRITICALおよびHIGHの指摘がある場合、修正案を提示する
5. 修正後にテストを再実行する
