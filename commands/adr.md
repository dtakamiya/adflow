---
name: adr
description: Architecture Decision Record（ADR）を銀行システム向けに作成する
arguments:
  - name: title
    description: ADRのタイトル（省略時はインタラクティブに確認）
    required: false
---

# /adr コマンド

ADR（Architecture Decision Record）を作成します。

## 実行手順

1. `writing-adr` スキルに従ってADRを作成する
2. `adr-author` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="opus",
  prompt="
    あなたはADR Authorエージェントです。agents/adr-author.md の指示に従ってください。

    タスク: ADRを作成する
    タイトル: {title または ユーザーに確認}

    手順:
    1. docs/adr/ ディレクトリをスキャンして次のADR番号を決定する
    2. templates/adr-template.md を読み込む
    3. ユーザーの要件に基づいてADRを作成する
    4. 銀行固有セクション（トランザクション影響、監査要件、データ分類、規制影響）を必ず含める
    5. docs/adr/{NUMBER}-{kebab-case-title}.md に保存する
    6. docs/adr/README.md のインデックスを更新する
  "
)
```

3. ADRの内容をユーザーに確認してもらう
4. 承認後、次のステージ（`/design`）の実行を提案する
