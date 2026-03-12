---
name: system-design
description: ADRを入力として、銀行システムのシステム設計書を作成する。Mermaid図（コンポーネント図・シーケンス図・ER図）、API仕様、データモデル、トランザクション設計、監査ログ設計を含む。
argument-hint: "[adr-number] - 対象ADR番号（例: 001）"
allowed-tools: Read, Write, Glob, Grep, Bash(ls *), Bash(find *)
---

# システム設計書作成スキル

## 利用可能なADR一覧
!`ls docs/adr/*.md 2>/dev/null || echo "ADRファイルなし"`

## 既存設計書一覧
!`ls docs/design/*.md 2>/dev/null || echo "設計書なし"`

## 引数の処理

- `$ARGUMENTS` が提供された場合: その内容を対象ADR番号として使用し、該当ADRファイルを読み込む
- `$ARGUMENTS` が空の場合: 上記の「利用可能なADR一覧」を提示し、ユーザーに対象ADRを選択してもらう

## 前提条件

- 関連するADRが `docs/adr/` に存在すること
- ADRが「承認済」ステータスであること（推奨）

## ワークフロー

### Step 1: ADR読込と分析

1. 該当するADRファイルを読み込む
2. ADRから以下を抽出する:
   - 決定内容と制約
   - トランザクション影響
   - 監査要件
   - データ分類

**エラーハンドリング:**
- 指定ADRが存在しない場合、利用可能なADR一覧を提示して選択を求める
- ADRが1つもない場合、`/adr` の実行を提案する

### Step 2: 既存コードベースの分析

1. プロジェクトの既存構造をスキャンする:
   - パッケージ構成
   - 既存のエンティティ・リポジトリ・サービス
   - 既存のAPIエンドポイント
2. 既存の設計パターンに合わせた設計を行う

### Step 3: コンポーネント設計

`templates/system-design-template.md` を使用して以下を設計:

1. **コンポーネント図**（Mermaid）: レイヤードアーキテクチャに基づく
2. **各コンポーネントの責務**: Controller / UseCase / Service / Repository

**エラーハンドリング:**
- テンプレートが見つからない場合、ユーザーに報告しビルトイン構造で代替する

### Step 4: シーケンス設計

1. **正常系シーケンス図**（Mermaid）: 主要フローを図示
2. **異常系シーケンス図**: 主要なエラーケース

### Step 5: API設計

1. エンドポイント一覧
2. リクエスト/レスポンスの JSON仕様
3. エラーレスポンスの標準化

### Step 6: データモデル設計

1. **ER図**（Mermaid）
2. **テーブル定義**: カラム名、型、制約
3. **インデックス設計**

### Step 7: 銀行固有設計

ADRの銀行固有セクションに基づいて:
1. **トランザクション設計**: 境界、伝播、分離レベル、ロールバック条件
2. **排他制御設計**: 楽観ロック/悲観ロックの選択と理由
3. **監査ログ設計**: イベント一覧、フォーマット、PII除外ルール
4. **冪等性設計**: 冪等キーの方式と有効期限
5. **エラーハンドリング**: エラー分類、リトライ可否

### Step 8: 設計書生成

1. `docs/design/{kebab-case-name}-design.md` にファイルを作成する
2. すべてのMermaid図が正しくレンダリングされることを確認する

**エラーハンドリング:**
- `docs/design/` が存在しない場合は `mkdir -p docs/design` で作成する

### Step 9: 承認ゲート

ユーザーに設計書の内容を確認してもらう:
- 修正が必要な場合は修正する
- 承認された場合、次のステージ（`/plan`）への進行を提案する

## リファレンス参照

設計時に以下のリファレンスを参照する:
- `references/transaction-patterns.md` — トランザクション設計の参考
- `references/audit-logging-patterns.md` — 監査ログ設計の参考
- `references/exclusive-control-patterns.md` — 排他制御設計の参考
- `references/idempotency-patterns.md` — 冪等性設計の参考
