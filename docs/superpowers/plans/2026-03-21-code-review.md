# /code-review スキル実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** adflow プラグインに汎用コードレビュースキル（`/code-review`）を追加し、4並列エージェントによるレビュー＆自動修正を提供する

**Architecture:** スキル本体（SKILL.md）がオーケストレーターとして差分取得→4並列レビュー→結果集約→自動修正→サマリー出力の5フェーズを制御する。既存の `code-reviewer` エージェントを拡張し、stack-loop の Step 5 と手順を共通化する。

**Tech Stack:** Claude Code Plugin（skills + agents）、Git

**Spec:** `docs/superpowers/specs/2026-03-21-code-review-design.md`

**Note:** Task 1 と Task 2 は相互依存がないため並列実行可能。Task 3 は Task 1・2 の完了後に実行。Task 4・5 は Task 3 の完了後に実行（並列可能）。

---

### Task 1: review-checklist.md を code-review スキルディレクトリに移動・完全書き換え

**Files:**
- Create: `skills/code-review/review-checklist.md`
- Delete: `skills/stack-loop/review-checklist.md`

- [ ] **Step 1: skills/code-review/ ディレクトリを作成**

Run: `mkdir -p skills/code-review`

- [ ] **Step 2: 既存チェックリストを移動**

Run: `git mv skills/stack-loop/review-checklist.md skills/code-review/review-checklist.md`

- [ ] **Step 3: チェックリストを設計書のID体系に合わせて完全書き換え**

`skills/code-review/review-checklist.md` を以下の内容で完全に置き換える。設計書の最終形（C1-C5, H1-H5, M1-M5）に合わせつつ、simplify の3観点を統合する:

```markdown
# コードレビュー チェックリスト

`git diff` の差分に対して、以下のチェックリストを適用する。
該当しない項目はスキップし、該当する項目のみ検証する。

---

## CRITICAL — 1つでも違反があれば修正必須

### C1. セキュリティ — シークレット
- [ ] シークレット（APIキー、パスワード、トークン）がソースコードにハードコードされていない

### C2. セキュリティ — インジェクション
- [ ] SQLインジェクション対策（パラメータ化クエリの使用）
- [ ] XSS対策（出力エスケープ）
- [ ] コマンドインジェクション対策（ユーザー入力をシェルコマンドに直接渡していない）

### C3. セキュリティ — 入力バリデーション
- [ ] ユーザー入力がサニタイズ/バリデーションされている

### C4. データ整合性
- [ ] 複数リソースの更新時に部分更新が発生しない設計になっている
- [ ] エラー発生時のロールバック/リカバリが適切に実装されている
- [ ] 状態遷移が正しく管理されている

### C5. データ保護
- [ ] 機密データがログに平文で出力されていない
- [ ] エラーレスポンスに内部情報（スタックトレース等）が含まれていない

---

## HIGH — マージ前に修正すべき

### H1. エラーハンドリング
- [ ] 例外がキャッチされず漏洩していない
- [ ] 適切なHTTPステータスコードが使用されている
- [ ] ビジネス例外と技術的例外が区別されている
- [ ] エラーレスポンスのフォーマットが統一されている

### H2. ロジック
- [ ] 明白なロジックエラーがない
- [ ] 境界条件（null、空文字、境界値）が適切に処理されている

### H3. コード再利用
- [ ] 既存のユーティリティや共通関数と重複する実装がない
- [ ] インラインで書かれた汎用ロジック（文字列操作、パス処理、型ガード等）が既存ユーティリティで代替可能でないか確認

### H4. 効率
- [ ] N+1クエリ、重複API呼び出し、不要な全件取得がない
- [ ] 独立した処理が逐次実行されていない（並列化の検討）

### H5. テスト
- [ ] 追加・変更されたロジックに対するテストが存在する
- [ ] テストが独立して実行可能（テスト間の依存がない）
- [ ] エッジケース（null、空文字、境界値）がテストされている

---

## MEDIUM — 可能であれば対応

### M1. コード品質
- [ ] 関数が小さい（50行未満）
- [ ] ファイルが適切な範囲（800行未満）
- [ ] 深いネスト（4階層以上）がない
- [ ] ハードコードされた値がない（定数またはconfigを使用）

### M2. 品質パターン
- [ ] コピペの微修正（near-duplicate）がない — 共通抽象化すべき
- [ ] stringly-typed コード（既存の定数・enum・型で代替可能な生文字列）がない
- [ ] 不要なコメント（WHATの説明、変更履歴、呼び出し元の記述）がない — 非自明なWHYのみ残す
- [ ] 冗長な状態（既存状態の重複、導出可能なキャッシュ）がない
- [ ] leaky abstractions（内部詳細の露出、既存抽象化の破壊）がない

### M3. 効率（軽微）
- [ ] ホットパス（起動時・リクエスト処理等）に不要なブロッキング処理が追加されていない
- [ ] ループやイベントハンドラ内の無条件更新に変更検出ガードがあるか
- [ ] メモリリーク（未解放のイベントリスナー、無制限データ構造）がないか
- [ ] 不要な存在チェック（TOCTOU アンチパターン）がない

### M4. 保守性
- [ ] 命名が意図を明確に表現している
- [ ] SOLID原則、DRY原則に沿っている

### M5. 再利用（軽微）
- [ ] インラインの文字列操作、パス処理、型ガード等が既存ユーティリティで代替可能でないか
```

- [ ] **Step 4: コミット**

```bash
git add skills/code-review/review-checklist.md
git commit -m "refactor: review-checklist.md を code-review スキルに移動し設計書のID体系で再構成"
```

---

### Task 2: code-reviewer エージェントを拡張

**Files:**
- Modify: `agents/code-reviewer.md`

- [ ] **Step 1: frontmatter を更新**

以下の変更を適用する:

```yaml
# 変更前
maxTurns: 20
disallowedTools:
  - Write
  - Edit
tools:
  - Read
  - Grep
  - Glob
  - Bash(git diff *)
  - Bash(git log *)
skills:
  - stack-loop

# 変更後
maxTurns: 30
tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash(git diff *)
  - Bash(git log *)
  - Bash(git status *)
  - Bash(./gradlew *)
  - Bash(./mvnw *)
  - Bash(npm *)
  - Bash(npx *)
  - Bash(pytest *)
  - Bash(cargo *)
  - Bash(go *)
  - Bash(dotnet *)
  - Bash(make *)
skills:
  - code-review
```

- `disallowedTools` セクションを完全に削除（Write, Edit を許可して自動修正対応）
- `maxTurns` を 20 → 30 に変更
- `tools` に Write, Edit, テスト実行系コマンドを追加
- `skills` を `stack-loop` → `code-review` に変更

- [ ] **Step 2: 「プロジェクト固有の品質チェック」のパス参照を更新**

「## レビュー手順」の「### 3. プロジェクト固有の品質チェック」セクションのパスを更新する:

```markdown
# 変更前:
`skills/stack-loop/review-checklist.md` が存在する場合は、そのチェックリストに基づいてレビューを実施する。

# 変更後:
`skills/code-review/review-checklist.md` のチェックリストに基づいてレビューを実施する。
```

- [ ] **Step 3: レビュー手順に simplify 観点を追加**

「## レビュー手順」セクションの「### 4. セキュリティチェック」の後に以下を追加:

```markdown
### 5. コード再利用チェック
- 既存のユーティリティ、ヘルパー関数との重複がないか検索する
- 新規関数が既存機能を再実装していないか確認する
- インラインロジック（文字列操作、パス処理、環境チェック、型ガード等）が既存ユーティリティで代替可能でないか確認する

### 6. 効率チェック
- 不要な計算の繰り返し、重複ファイル読み込み、重複API呼び出し、N+1パターンがないか確認する
- 独立した処理が逐次実行されている場合、並列化の可否を検討する
- 起動時やリクエスト処理等のホットパスに不要なブロッキング処理が追加されていないか確認する
- ループやイベントハンドラ内の無条件更新に変更検出ガードがあるか確認する
- メモリリーク（未解放リソース、無制限データ構造）がないか確認する
```

- [ ] **Step 4: 出力フォーマットを更新**

「## 出力フォーマット」セクション全体を以下に置き換える:

```markdown
## 出力フォーマット

\```markdown
# コードレビュー結果

## サマリー
- 変更ファイル数: {N}
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
\```
```

- [ ] **Step 5: コミット**

```bash
git add agents/code-reviewer.md
git commit -m "feat: code-reviewer エージェントに自動修正と simplify 観点を追加"
```

---

### Task 3: /code-review スキル本体（SKILL.md）を作成

**依存:** Task 1, Task 2 の完了後

**Files:**
- Create: `skills/code-review/SKILL.md`

- [ ] **Step 1: SKILL.md を作成**

以下の内容で `skills/code-review/SKILL.md` を作成する:

```markdown
---
name: code-review
description: ワーキングディレクトリの変更差分に対して汎用コードレビューを実行し、問題を自動修正する。「レビュー」「コードレビュー」「品質チェック」「レビューして」というキーワードに反応。
argument-hint: "[--staged | --all] - レビュー対象（省略時は自動判定）"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git *), Bash(./gradlew *), Bash(./mvnw *), Bash(npm *), Bash(npx *), Bash(pytest *), Bash(cargo *), Bash(go *), Bash(dotnet *), Bash(make *), Bash(find *)
---

> "adflow の `/code-review` スキルを使用して、コードレビュー＆自動修正を実行します。"

# コードレビュー＆自動修正スキル

## レビューチェックリスト
!read `${CLAUDE_SKILL_DIR}/review-checklist.md`

## ビルドシステム
!`find . -maxdepth 1 -name 'build.gradle' -o -name 'build.gradle.kts' -o -name 'pom.xml' -o -name 'package.json' -o -name 'pyproject.toml' -o -name 'setup.py' -o -name 'Cargo.toml' -o -name 'go.mod' -o -name 'Makefile' -o -name '*.csproj' -o -name '*.sln' 2>/dev/null`

## 鉄則（絶対ルール）

1. **偽陽性を報告しない** — 確信のない指摘は報告しない。偽陽性は信頼を損なう。
2. **修正後にテストを実行する** — ビルドシステムが検出できた場合、各修正後にテストを実行してリグレッションがないことを確認する。
3. **CRITICAL は必ず修正する** — CRITICAL レベルの問題をスキップすることは許可されない。

## 引数の処理

`$ARGUMENTS` をパースする:
- `--staged`: `git diff --cached` を対象とする
- `--all`: `git diff` を対象とする
- 引数なし: 自動判定（Phase 1 参照）

## Phase 1: 差分取得と対象判定

1. `git diff --cached --stat` を実行する
2. ステージング済み変更がある場合:
   - `git diff --cached` を対象とする
   - 「ステージング済みの変更をレビューします」と報告する
3. ステージング済み変更がない場合:
   - `git diff --stat` を実行する
   - 未ステージ変更がある場合: `git diff` を対象とする
   - 「未ステージの変更をレビューします」と報告する
4. どちらもない場合:
   - 「レビュー対象の変更がありません」と報告して終了する

差分全体を取得し、変数 `DIFF` として以降のフェーズで使用する。

## Phase 2: 並列レビュー（4エージェント）

Agent ツールで以下の4つのサブエージェントを **単一メッセージで並列起動** する。各エージェントに `DIFF` の全体と、上記で読み込んだレビューチェックリストの内容を渡す。

### Agent 1: セキュリティ & データ整合性レビュー

サブエージェントタイプ: `adflow:code-reviewer`

プロンプト:
> 以下の差分に対してセキュリティとデータ整合性の観点でレビューしてください。
> チェック項目: C1〜C5（CRITICAL）、H1（HIGH: エラーハンドリング）
> 出力フォーマット: `## [{ファイル名}:{行番号}] {タイトル}` + 重大度 + 問題 + 修正案
> 確信のない指摘は報告しないでください。

### Agent 2: コード再利用レビュー

サブエージェントタイプ: `general-purpose`

> Agent 2 と Agent 4 は `general-purpose` を使用する。理由: コードベース全体を Grep/Glob で検索して既存実装との重複や効率問題を発見する必要があり、`code-reviewer` のペルソナ（セキュリティ重視）とは異なる広範な探索が必要なため。

プロンプト:
> 以下の差分に対してコード再利用の観点でレビューしてください。リサーチのみ、コード変更なし。
> 1. 差分内の新規関数・ロジックに対して、コードベース内に既存の類似実装がないか Grep/Glob で検索する
> 2. インラインロジック（文字列操作、パス処理、型ガード等）が既存ユーティリティで代替可能でないか確認する
> 3. 重複を発見した場合: H3（HIGH）または M5（MEDIUM）として報告する
> 出力フォーマット: `## [{ファイル名}:{行番号}] {タイトル}` + 重大度 + 問題 + 修正案

### Agent 3: コード品質レビュー

サブエージェントタイプ: `adflow:code-reviewer`

プロンプト:
> 以下の差分に対してコード品質の観点でレビューしてください。
> チェック項目: H2（ロジック）、H5（テスト品質）、M1（コード品質）、M2（品質パターン）
> 追加チェック:
> - 冗長な状態（既存状態の重複、導出可能なキャッシュ）
> - パラメータの肥大化（新パラメータの追加ではなく一般化・再構成すべき）
> - leaky abstractions（内部詳細の露出、既存抽象化の破壊）
> 出力フォーマット: `## [{ファイル名}:{行番号}] {タイトル}` + 重大度 + 問題 + 修正案
> 確信のない指摘は報告しないでください。

### Agent 4: 効率レビュー

サブエージェントタイプ: `general-purpose`

プロンプト:
> 以下の差分に対して効率の観点でレビューしてください。リサーチのみ、コード変更なし。
> チェック項目: H4（効率）、M3（効率・軽微）
> 追加チェック:
> - 不要な存在チェック（TOCTOU アンチパターン）— 直接操作してエラーハンドリング
> - 無制限データ構造、クリーンアップ漏れ、イベントリスナーリーク
> - ファイル全体の読み込み（一部のみ必要な場合）
> 出力フォーマット: `## [{ファイル名}:{行番号}] {タイトル}` + 重大度 + 問題 + 修正案

## Phase 3: 結果集約

全エージェントの結果を受け取り、以下の順序でソートする:
1. **CRITICAL** — 必ず修正
2. **HIGH** — 修正すべき
3. **MEDIUM** — 可能であれば対応

偽陽性と判断した指摘はスキップし、理由を記録する。判断基準:
- 変更差分に含まれないコードへの指摘 → スキップ
- 既存コードの問題（pre-existing issue）→ スキップ
- 主観的なスタイルの好み → スキップ

## Phase 4: 自動修正

CRITICAL → HIGH → MEDIUM の順に修正を実行する。

各修正について:
1. Read ツールで対象ファイルを読み込む
2. Edit ツールで修正を適用する
3. ビルドシステムが検出できている場合、テストを実行する
4. テストが失敗した場合、修正をリバートして「修正不可」として記録する

## Phase 5: サマリー出力

以下のフォーマットで結果を出力する:

\```
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
\```
```

- [ ] **Step 2: コミット**

```bash
git add skills/code-review/SKILL.md
git commit -m "feat: /code-review スキルを追加（4並列レビュー＆自動修正）"
```

---

### Task 4: stack-loop の Step 5 を更新

**依存:** Task 3 の完了後

**Files:**
- Modify: `skills/stack-loop/SKILL.md`

- [ ] **Step 1: Step 5 のレビュー手順を書き換え**

`skills/stack-loop/SKILL.md` の `### Step 5: AI自己レビュー（品質・仕様整合性チェック）` セクション全体（現在の内容: エージェント自身で git diff に対してレビュー〜指摘事項がクリアになれば次へ進む）を以下に置き換える:

```markdown
### Step 5: AI自己レビュー（品質・仕様整合性チェック）

`/code-review` スキルと同じ手順でコードレビューを実行する。

**Phase 1**: 差分取得 — `git diff` で現在の変更を取得する

**Phase 2**: 4つの並列サブエージェントでレビューを実行する（Agent ツールで単一メッセージで起動）。各エージェントに差分全体と `skills/code-review/review-checklist.md` の内容を渡す:

| Agent | 観点 | サブエージェント型 |
|-------|------|-------------------|
| Agent 1 | セキュリティ & データ整合性 | `adflow:code-reviewer` |
| Agent 2 | コード再利用 | `general-purpose` |
| Agent 3 | コード品質 + チェックリスト | `adflow:code-reviewer` |
| Agent 4 | 効率 | `general-purpose` |

各エージェントに差分全体を渡す。追加で以下のコンテキストも渡す:
- `docs/{dir-name}/02-spec.md` の API設計・データモデルとの整合性チェック
- `docs/{dir-name}/01-adr.md` の決定事項・制約への違反チェック

**Phase 3**: 結果集約 — CRITICAL → HIGH → MEDIUM 順にソート。偽陽性はスキップ。

**Phase 4**: 自動修正 — 問題を修正し、修正後にテストを実行する。

**Phase 5**: サマリー出力

問題を発見・修正した場合は、**Step 4 ローカル検証に戻る**。
指摘事項がすべてクリアになれば次へ進む。
```

- [ ] **Step 2: コミット**

```bash
git add skills/stack-loop/SKILL.md
git commit -m "refactor: stack-loop Step 5 を /code-review と同じ4並列レビュー手順に統一"
```

---

### Task 5: CLAUDE.md を更新

**依存:** Task 1 の完了後（チェックリストの移動によるパス変更を反映）

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: コマンド自動選択テーブルに /code-review を追加**

`CLAUDE.md` の「## コマンド自動選択テーブル」セクションのテーブルに以下の行を追加する:

```markdown
| 「レビュー」「コードレビュー」「品質チェック」「レビューして」 | `/code-review` | コードレビュー＆自動修正 |
```

- [ ] **Step 2: カスタマイズセクションのパスを更新**

「## カスタマイズ」セクションにある以下のパスを更新する:

```markdown
# 変更前:
- `skills/stack-loop/review-checklist.md` にプロジェクト固有のレビュー項目を追加する

# 変更後:
- `skills/code-review/review-checklist.md` にプロジェクト固有のレビュー項目を追加する
```

- [ ] **Step 3: コミット**

```bash
git add CLAUDE.md
git commit -m "docs: CLAUDE.md にコードレビューコマンドを追加しチェックリストパスを更新"
```

---

### Task 6: 動作検証

**Files:**
- 変更なし（検証のみ）

- [ ] **Step 1: スキルが認識されることを確認**

`skills/code-review/SKILL.md` の frontmatter を確認:
- `name: code-review` が設定されている
- `allowed-tools` に必要なツールが含まれている
- `context: fork` や `agent` が指定されていないことを確認

- [ ] **Step 2: review-checklist.md の移動を確認**

```bash
# 旧ファイルが存在しないことを確認
find skills/stack-loop -name "review-checklist.md" 2>/dev/null | grep -c . && echo "ERROR: old file still exists" || echo "OK: old file removed"

# 新ファイルが存在することを確認
find skills/code-review -name "review-checklist.md" 2>/dev/null | grep -c . && echo "OK: new file exists" || echo "ERROR: new file missing"
```

- [ ] **Step 3: code-reviewer エージェントの変更を確認**

```bash
# disallowedTools が削除されていることを確認
grep -c "disallowedTools" agents/code-reviewer.md && echo "ERROR: disallowedTools still exists" || echo "OK: disallowedTools removed"

# Write, Edit が tools に含まれていることを確認
grep "Write" agents/code-reviewer.md && echo "OK: Write tool added"
grep "Edit" agents/code-reviewer.md && echo "OK: Edit tool added"

# skills が code-review に変更されていることを確認
grep "code-review" agents/code-reviewer.md && echo "OK: skills updated"

# 旧パス参照が残っていないことを確認
grep "stack-loop/review-checklist" agents/code-reviewer.md && echo "ERROR: old path reference remains" || echo "OK: no old path references"
```

- [ ] **Step 4: stack-loop の Step 5 が更新されていることを確認**

```bash
# 旧 review-checklist.md 参照が削除されていることを確認
grep "CLAUDE_SKILL_DIR.*review-checklist" skills/stack-loop/SKILL.md && echo "WARNING: old CLAUDE_SKILL_DIR reference may remain" || echo "OK: old reference removed"

# 新しい4並列レビュー手順が含まれていることを確認
grep "adflow:code-reviewer" skills/stack-loop/SKILL.md && echo "OK: new review procedure added"

# チェックリストの参照パスが明示されていることを確認
grep "skills/code-review/review-checklist.md" skills/stack-loop/SKILL.md && echo "OK: checklist path explicit"
```

- [ ] **Step 5: CLAUDE.md のパス更新を確認**

```bash
# 旧パスが残っていないことを確認
grep "stack-loop/review-checklist" CLAUDE.md && echo "ERROR: old path in CLAUDE.md" || echo "OK: path updated"

# 新パスが存在することを確認
grep "code-review/review-checklist" CLAUDE.md && echo "OK: new path in CLAUDE.md"
```

- [ ] **Step 6: コミット（必要な場合のみ）**

検証で問題が見つかった場合は修正してコミットする。問題がなければスキップ。
