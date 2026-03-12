---
name: spec
description: ADRの内容を入力として、銀行システムの要件を満たす仕様書を作成します。
arguments:
  - name: adr
    description: 対象のADR番号、機能名、またはファイルパス（省略時は自動検出）
    required: false
---

# /spec - 仕様書作成コマンド

ADRの内容を入力として、銀行システムの要件を満たす仕様書を作成します。

## 前提条件

- 関連するADRが `docs/{dir-name}/01-adr.md` に存在すること

## 実行手順

1. このコマンドは `specification` スキルに従って仕様書を作成する
2. `specification-agent` エージェントを起動する:

```
Agent(
  subagent_type="general-purpose",
  model="opus",
  prompt="
    あなたはSpecificationエージェントです。

    タスク: 仕様書を作成する
    対象ADR: {adr または 自動検出}

    手順:
    1. 最初に docs/*/.adflow-context.md をスキャンして、アクティブなワークフローの前段成果物を自動検出してください
    2. 「現在のステージ」が spec のコンテキストファイルを検索する
    3. 該当が1件ならそのADRを自動選択、複数なら一覧提示して選択してもらう。また該当する機能名ディレクトリ名（{dir-name}）を控える。
    4. コンテキストファイルからADRのファイルパスを取得し、Read ツールで全文読み込む
    5. プロジェクトの既存コードベースをスキャンする
    6. templates/system-design-template.md を適宜読み込む
    7. 以下のリファレンスを参照する:
       - references/transaction-patterns.md
       - references/audit-logging-patterns.md
       - references/exclusive-control-patterns.md
       - references/idempotency-patterns.md
    8. Mermaid図（コンポーネント図、シーケンス図、ER図）を含む仕様書を作成する
    9. docs/{dir-name}/02-spec.md に保存する
    10. 承認後、docs/{dir-name}/.adflow-context.md を更新する
        - 仕様書ファイルパスを記録、ステータス「完了」、現在のステージを stack-plan に更新
    11. 「次のステージ /stack-plan を実行するか、/clear して後で再開できます」と案内して終了する。（※ /workflow の一環として実行されている場合を除き、自律的に次のコマンドを実行してはいけません。必ず人間の明示的なコマンド入力または承認を待つこと）
  "
)
```

3. 以下を含む仕様書（Markdown）を生成し、AI自己レビュー後にユーザーに確認を求めます：
4. 承認後、コンテキストファイルを更新し、次のステージ（`/stack-plan`）の実行を提案する
