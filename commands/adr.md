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
    1. 最初に docs/*/.adflow-context.md をスキャンして、アクティブなワークフローの前段成果物を自動検出してください
    2. タイトルからkebab-caseの機能名ディレクトリ名を決定する
    3. docs/ ディレクトリをスキャンして次の機能連番（NNNN）を決定する
    4. mkdir -p docs/{NNNN}-{feature-name} でディレクトリを作成する
    5. templates/adr-template.md を読み込む
    6. ユーザーの要件に基づいてADRを作成する
    7. 銀行固有セクション（トランザクション影響、監査要件、データ分類、規制影響）を必ず含める
    8. docs/{NNNN}-{feature-name}/01-adr.md に保存する
    9. 承認後、templates/adflow-context-template.md を使って docs/{NNNN}-{feature-name}/.adflow-context.md を作成する
        - 機能名、ADRファイルパス、ステータス「完了」、現在のステージ「spec」を記録する
    11. 「次のステージ /spec を実行するか、/clear して後で再開できます」と案内して終了する。（※ /workflow の一環として実行されている場合を除き、自律的に次のコマンドを実行してはいけません。必ず人間の明示的なコマンド入力または承認を待つこと）
  "
)
```

3. ADRの内容をユーザーに確認してもらう
4. 承認後、コンテキストファイルを作成し、次のステージ（`/spec`）の実行を提案する
