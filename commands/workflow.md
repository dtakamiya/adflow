---
name: workflow
description: 8段階ワークフローをオーケストレーションします。
arguments:
  - name: feature
    description: 開発する機能名（省略時は質問されます）
    required: false
  - name: from
    description: 特定のステージから開始する場合（`adr`, `spec`, `stack-plan`, `stack-loop`）— 途中から再開する場合
    required: false
---

# /workflow コマンド

ミッションクリティカルシステム開発の8段階ワークフローをオーケストレーションします。

## ワークフロー

```bash
/workflow [feature] [--from=stage]
```

- `feature`: 開発する機能名（省略時は質問されます）
- `--from=stage`: 特定のステージから開始する場合（`adr`, `spec`, `stack-plan`, `stack-loop`）— 途中から再開する場合

## 実行手順

このコマンドを実行すると、以下のステップが順番に案内されます：

1. **コンテキスト収集とADR作成** (`/adr`) - アーキテクチャ判断と自己レビュー
2. **仕様書作成** (`/spec`) - 詳細リサーチ、仕様策定、自己レビュー
3. **スタックPR計画** (`/stack-plan`) - タスク分割、PR依存関係計画、自己レビュー
4. **実装ループ** (`/stack-loop`) - ブランチ作成、TDD、自己テスト、PR作成の反復

## 途中再開

`--from` パラメータで途中のステージから再開可能:

```bash
/workflow transfer-service
```

途中から再開する場合：

```bash
/workflow transfer-service --from=stack-plan
```
