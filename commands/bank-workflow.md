---
name: bank-workflow
description: 5段階ワークフロー（ADR→設計書→実装計画→TDD→レビュー）を順番に実行する
arguments:
  - name: feature
    description: 実装する機能名
    required: false
  - name: from
    description: 開始ステージ（adr/design/plan/tdd/review）— 途中から再開する場合
    required: false
---

# /bank-workflow コマンド

銀行システム開発の5段階ワークフローを順番に実行します。

## ワークフロー

```
/adr → /design → /plan → /bank-tdd → /bank-review
  ↓        ↓         ↓          ↓            ↓
 ADR   設計書    実装計画    テスト+実装    レビュー
```

## 実行手順

1. `bank-workflow` スキルに従ってオーケストレーションを実行する

2. 機能名の確認:
   - ユーザーに実装する機能名と概要を確認する

3. 既存成果物のチェック:
   - `docs/adr/` — ADRの有無
   - `docs/design/` — 設計書の有無
   - `docs/plans/` — 実装計画書の有無

4. 各ステージを順番に実行:

   **Stage 1: ADR** (`/adr`)
   → 承認ゲート → 次へ

   **Stage 2: 設計書** (`/design`)
   → 承認ゲート → 次へ

   **Stage 3: 実装計画** (`/plan`)
   → 承認ゲート → 次へ

   **Stage 4: TDD** (`/bank-tdd`)
   → テスト全通過 → 次へ

   **Stage 5: レビュー** (`/bank-review`)
   → 指摘解消 → 完了

5. 完了時に成果物一覧を表示する

## 途中再開

`--from` パラメータで途中のステージから再開可能:
- `/bank-workflow --from=design` → Stage 2 から
- `/bank-workflow --from=plan` → Stage 3 から
- `/bank-workflow --from=tdd` → Stage 4 から
- `/bank-workflow --from=review` → Stage 5 から
