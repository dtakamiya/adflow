# /code-review スキル設計書

## 概要

adflow プラグインに汎用コードレビュー機能（`/code-review`）を追加する。ワーキングディレクトリの変更差分に対して4つの並列エージェントでレビューし、問題を自動修正する。simplify スキルの3観点（再利用・品質・効率）にセキュリティ・データ整合性を加えた包括的なレビューを提供する。

## 要件

- **対象**: ワーキングディレクトリの差分（`git diff --cached` または `git diff`、自動判定）
- **動作**: 問題検出 → 自動修正
- **位置**: 独立スキル + stack-loop 内 Step 5 の統一
- **カスタマイズ**: プロジェクト固有の `review-checklist.md` があれば追加適用

## スキル構造

### ファイル構成

```
skills/
  code-review/
    SKILL.md              # スキル本体（オーケストレーター）
    review-checklist.md   # 汎用レビューチェックリスト（stack-loop/review-checklist.md を統合移動）
```

既存の `skills/stack-loop/review-checklist.md` は `skills/code-review/review-checklist.md` に統合移動する。stack-loop の Step 5 は `/code-review` スキルの手順を参照するため、チェックリストの重複は発生しない。

### frontmatter

```yaml
name: code-review
description: ワーキングディレクトリの変更差分に対して汎用コードレビューを実行し、問題を自動修正する。「レビュー」「コードレビュー」「品質チェック」「レビューして」というキーワードに反応。
argument-hint: "[--staged | --all] - レビュー対象（省略時は自動判定）"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git *), Bash(./gradlew *), Bash(./mvnw *), Bash(npm *), Bash(npx *), Bash(pytest *), Bash(cargo *), Bash(go *), Bash(dotnet *), Bash(make *), Bash(find *)
```

**設計判断:**
- `context: fork` と `agent` は指定しない。スキル自体がオーケストレーターとして動作し、Phase 2 で4つの並列サブエージェントを Agent ツールで起動する。fork 内から更に fork を起動するネスト制限の問題を回避する。
- `plugin.json` の変更は不要。`skills` フィールドが `["./skills/"]` とディレクトリ指定になっているため、新しいスキルディレクトリは自動検出される。

## 実行フロー

### Phase 1: 差分取得と対象判定

1. `git diff --cached` を確認
2. ステージング済み変更があれば → それを対象
3. なければ `git diff` → 未ステージ変更を対象
4. どちらもなければ → 「レビュー対象の変更がありません」で終了

### Phase 2: 並列レビュー（4エージェント）

Agent ツールで4つのサブエージェントを**単一メッセージで並列起動**する。各エージェントに差分全体と `${CLAUDE_SKILL_DIR}/review-checklist.md` の内容を渡す。

| Agent | 観点 | 判定レベル | 詳細 |
|-------|------|-----------|------|
| Agent 1 | セキュリティ & データ整合性 | CRITICAL, HIGH | シークレット検出、インジェクション脆弱性、入力バリデーション、部分更新リスク、機密データ漏洩 |
| Agent 2 | コード再利用 | HIGH, MEDIUM | 既存ユーティリティの重複実装、インラインで書かれた汎用ロジック |
| Agent 3 | コード品質 + チェックリスト | HIGH, MEDIUM | hacky パターン（冗長な状態、コピペ、leaky abstractions、stringly-typed コード、不要なコメント）+ チェックリスト適用 |
| Agent 4 | 効率 | HIGH, MEDIUM | N+1 クエリ、並列化可能な逐次処理、ホットパスへの不要処理、メモリリーク |

各エージェントの出力フォーマット:
```
## [{ファイル名}:{行番号}] {タイトル}
- **重大度**: CRITICAL / HIGH / MEDIUM
- **問題**: {説明}
- **修正案**: {具体的な修正方法}
```

### Phase 3: 結果集約

全エージェントの結果を重大度順にソート: CRITICAL → HIGH → MEDIUM

偽陽性と判断したものはスキップし、理由を記録する。

### Phase 4: 自動修正

CRITICAL → HIGH → MEDIUM の順に修正を実行する。各修正後にテストを実行して（ビルドシステムが検出できた場合）、リグレッションがないことを確認する。

### Phase 5: サマリー出力

```
# コードレビュー結果

## サマリー
- CRITICAL: {N}件（修正済み: {N}）
- HIGH: {N}件（修正済み: {N}）
- MEDIUM: {N}件（修正済み: {N}）
- スキップ: {N}件

## 修正内容
### [{ファイル名}:{行番号}] {タイトル}（{重大度}）
- **問題**: {説明}
- **修正**: {何をどう変えたか}

## スキップした指摘
### {タイトル}
- **理由**: {偽陽性と判断した根拠}

## 良い点
- {良い実装や設計の指摘}
```

**既存 `code-reviewer` エージェントの出力フォーマットからの変更点:**
- 「修正案」（コードブロック提示）→ 「修正」（実際に修正した内容の報告）に変更
- 「スキップした指摘」セクションを追加
- 「良い点」セクションは維持

## 汎用レビューチェックリスト

### CRITICAL（1つでも違反があれば修正必須）

| ID | カテゴリ | チェック項目 |
|----|---------|-------------|
| C1 | セキュリティ | シークレット（APIキー、パスワード、トークン）のハードコード |
| C2 | セキュリティ | SQLインジェクション、XSS、コマンドインジェクションの脆弱性 |
| C3 | セキュリティ | ユーザー入力の未サニタイズ・未バリデーション |
| C4 | データ整合性 | 部分更新が発生しうる複数リソース操作（ロールバック欠如） |
| C5 | データ保護 | 機密データのログ出力・エラーレスポンスへの漏洩 |

### HIGH（マージ前に修正すべき）

| ID | カテゴリ | チェック項目 |
|----|---------|-------------|
| H1 | エラーハンドリング | 未処理の例外、エラーの握りつぶし |
| H2 | ロジック | 明白なロジックエラー、境界条件の未処理 |
| H3 | コード再利用 | 既存ユーティリティの重複実装 |
| H4 | 効率 | N+1クエリ、重複API呼び出し、不要な全件取得 |
| H5 | テスト | 追加・変更ロジックに対するテストの欠如 |

### MEDIUM（可能であれば対応）

| ID | カテゴリ | チェック項目 |
|----|---------|-------------|
| M1 | 品質 | 関数50行超、ファイル800行超、ネスト4層超 |
| M2 | 品質 | コピペの微修正、stringly-typed コード |
| M3 | 効率 | 並列化可能な逐次処理、ホットパスへの不要な処理追加 |
| M4 | 保守性 | 命名の不明瞭さ、不要なコメント、ハードコード値 |
| M5 | 再利用 | インラインで書かれた汎用ロジック（既存ユーティリティで代替可能） |

## エージェント変更

### `code-reviewer` エージェントの変更点

| 項目 | 現在 | 変更後 |
|------|------|--------|
| `disallowedTools` | Write, Edit | 削除（自動修正のため） |
| `tools` | Read, Grep, Glob, Bash(git diff/log) | + Write, Edit, Bash(テスト実行系) |
| `maxTurns` | 20 | 30（修正ループのため増加） |
| `skills` | stack-loop | code-review |
| 出力フォーマット | 修正案（コードブロック提示） | 修正報告 + スキップ理由 |

ペルソナ（シニア AppSec/QA エンジニア）と重大度分類（CRITICAL/HIGH/MEDIUM）は維持。simplify の3観点（再利用・品質・効率）をレビュー手順に追加。

**注意:** `/code-review` スキルは `agent` を frontmatter で指定しない。スキル自体がオーケストレーターとなり、Phase 2 で Agent ツールを使って4つのサブエージェントを並列起動する。`code-reviewer` エージェントはこれらのサブエージェントの1つ（Agent 1: セキュリティ担当）として使用される。

## stack-loop との統合

### Step 5 の書き換え

**現在:** tdd-guide エージェントが直接 `git diff` を見て自己レビュー + 自律修正

**変更後:** stack-loop の Step 5 に `/code-review` スキルの手順（Phase 1〜5）を埋め込む。スキル呼び出し（スキルからスキル）ではなく、同じ手順をテキストとして Step 5 に記述し、tdd-guide エージェントが実行する。

追加コンテキストとして仕様書・ADRとの整合性チェックを含める:
```
Phase 2 の各エージェントに以下も渡す:
- docs/{dir-name}/02-spec.md の API設計・データモデルとの整合性
- docs/{dir-name}/01-adr.md の決定事項・制約への違反
```

問題発見・修正後、Step 4（ローカル検証）に戻る。

### チェックリストの統合

`skills/stack-loop/review-checklist.md` を `skills/code-review/review-checklist.md` に移動統合する。stack-loop の Step 5 では `!read` ディレクティブで code-review スキルのチェックリストを参照する。

## CLAUDE.md の変更

コマンド自動選択テーブルに追加:

```
| 「レビュー」「コードレビュー」「品質チェック」「レビューして」 | `/code-review` | コードレビュー＆自動修正 |
```

## 参考資料

- [The Ultimate 2025 Code Review Checklist: 8 Pillars](https://www.docuwriter.ai/posts/code-review-checklist)
- [The Ultimate Code Review Checklist - Qodo](https://www.qodo.ai/blog/code-review-checklist/)
- [Code Review Checklist for AI-Generated Code - ClackyAI](https://clacky.ai/blog/code-review-checklist-ai-generated-code)
- [Anthropic Code Review Plugin](https://github.com/anthropics/claude-code/blob/main/plugins/code-review/commands/code-review.md)
