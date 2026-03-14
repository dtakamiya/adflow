---
name: stack-pr-loop
description: スタックPR計画に基づいて、PRごとの実装ループ（ブランチ作成→TDD→ローカル検証→AI自己レビュー→コミット・PR作成）を反復実行する。計画承認後に実装開始する時に使用。
argument-hint: "[feature-name] - 対象機能名（例: transfer-service）"
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git *), Bash(gh *), Bash(./gradlew *), Bash(./mvnw *), Bash(npm *), Bash(npx *), Bash(pytest *), Bash(cargo *), Bash(go *), Bash(dotnet *), Bash(make *), Bash(ls *), Bash(find *)
context: fork
agent: tdd-guide
---

> "adflow の `/stack-loop` スキルを使用して、TDD駆動の実装ループを実行します。"

# スタックPR 実装ループスキル

## アクティブなワークフロー（stack-loop待ち）
!`for d in docs/*/; do f=${d}.adflow-context.md; if [ -f $f ] && grep -q stack-loop $f 2>/dev/null; then echo "--- $(basename $d) ---"; cat $f; echo; fi; done 2>/dev/null || echo "対象ワークフローなし"`

## 利用可能な実装計画書一覧
!`ls docs/*/*-plans.md 2>/dev/null || echo "計画書なし"`

## ドメイン品質テストパターン
!`cat ${CLAUDE_SKILL_DIR}/testing-patterns.md 2>/dev/null || echo "パターンファイルなし"`

## ビルドシステム
!`if [ -f build.gradle ] || [ -f build.gradle.kts ]; then echo "Gradle"; elif [ -f pom.xml ]; then echo "Maven"; elif [ -f package.json ]; then echo "Node.js"; elif [ -f pyproject.toml ] || [ -f setup.py ]; then echo "Python"; elif [ -f Cargo.toml ]; then echo "Rust"; elif [ -f go.mod ]; then echo "Go"; elif ls *.csproj >/dev/null 2>&1 || [ -f *.sln ]; then echo ".NET"; elif [ -f Makefile ]; then echo "Make"; else echo "不明"; fi`

## 鉄則（絶対ルール）

1. **RED（テスト失敗確認）をスキップしない** — テストを書いたら必ず実行して失敗を確認する。最初から成功するテストは意味がない。
2. **自己レビューで発見した問題を無視しない** — レビューで発見された問題は修正してから次に進む。「後で直す」は禁止。
3. **テストが通らない状態でコミットしない** — すべてのテストが成功してからコミットする。CI/CDの信頼性を守る。
4. **仕様書・ADRとの整合性を常に確認する** — 実装が仕様書やADRの決定事項から逸脱していないか、各PRで確認する。

## Red Flags — よくある合理化

| 思考 | 現実 |
|------|------|
| 「テストは後でまとめて書こう」 | TDDのREDフェーズは省略不可 |
| 「この修正は小さいからレビュー不要」 | 小さな変更こそバグの温床 |
| 「テストが1つ失敗してるが他は通ってる」 | 1つでも失敗はコミット禁止 |
| 「仕様書と少し違うが改善だ」 | 仕様との乖離はADR/仕様の更新が先 |
| 「リファクタリングは後で」 | GREEN後のREFACTORはTDDの一部 |

## 引数の処理

- `$ARGUMENTS` が提供された場合: 機能ディレクトリ名として使用し、`docs/{dir-name}/.adflow-context.md` を読み込む
- `$ARGUMENTS` が空の場合: 以下の自動検出ロジックを実行する

### 前段ドキュメント自動検出

1. `docs/*/` 配下の `.adflow-context.md` をスキャンする
2. 「現在のステージ」が `stack-loop` のコンテキストファイルを検索する
3. 該当が1件: そのワークフローの計画書を自動選択する
4. 該当が複数: 一覧を提示してユーザーに選択してもらう
5. 該当なし: 旧方式のフォールバック（`docs/*-plans.md` から最新を候補提示 or ユーザーに実装対象を確認）

## 前提条件

- 実装計画書が `docs/{feature-name}/03-plans.md` に存在すること（推奨）
- ビルドが通る状態であること

## ワークフロー

### Step 1: 実装タスクの特定と準備

1. コンテキストファイルから計画書・仕様書・ADRのファイルパスを取得する
2. 実装計画書を読み込み、**「未着手の最初のPR」** を特定する
3. PR内のタスク一覧を把握する

### Step 2: ブランチ作成（スタックルール）

1. 現在のGitステータスを確認し、未コミットの変更がないことを確認する
2. 対象PRの「親PR」に指定されているブランチがある場合:
   - その親ブランチをチェックアウトする: `git checkout {parent-branch}`
   - 最新状態をプルする: `git pull origin {parent-branch}`
3. 新しいPRブランチを作成する: `git checkout -b feature/{dir-name}-pr{number}-{short-desc}`

### Step 3: TDD（Red/Green/Refactor）の反復

PR内の各Taskに対して、以下のTDDサイクルを実行する:

**1. RED（テスト作成）**:
- テストファイルを作成し、Given/When/Then形式でテストを書く
- 「ドメイン品質テストパターン」を参照して必要なテスト（金額アサーション、ロールバック確認等）を追加
- テストを実行し、**失敗することを確認**する

**2. GREEN（最小実装）**:
- テストを通すための最小限のコードを実装する
- テストを実行し、**成功することを確認**する

**3. REFACTOR（リファクタリング）**:
- コードを整理し、SOLID原則と以下の品質基準を満たしているか確認:
  - 金銭計算（高精度小数）、トランザクション宣言、監査ログ
- テストを再実行し、**成功が維持されること**を確認する

（※プロジェクトのビルドシステムに応じたテストコマンドを使用する: Gradleなら `./gradlew test`、Nodeなら `npm test` 等）

### Step 4: ローカル検証（Lint, 型チェック, テスト全実行）

タスクがすべて完了したら、PR品質を担保するための全体チェックを行う:
1. 静的解析（Lint、フォーマットチェック）を実行し、問題があれば修正する
2. 型チェックを実行し、問題があれば修正する
3. 全体テストスイートを実行し、依存関係の破壊がないか確認する

### Step 5: AI自己レビュー（ドメイン品質・仕様整合性チェック）

エージェント自身で現在の差分（`git diff`）に対してコードレビューを行う:
1. `${CLAUDE_SKILL_DIR}/review-checklist.md` のドメイン要件を満たしているか？
2. `docs/{dir-name}/02-spec.md` の仕様書（API設計、データモデル）に記述通りか？
3. `docs/{dir-name}/01-adr.md` の決定事項や制約に反していないか？

問題を発見した場合は、**自律的にコードを修正し、Step 4 ローカル検証に戻る**。
指摘事項がクリアになれば次へ進む。

### Step 6: コミット作成、Push、PR作成

1. 変更をステージング: `git add .`
2. 適切なコミットメッセージでコミット: `git commit -m "feat({feature}): implement {pr-desc}"`
3. リモートへプッシュ: `git push -u origin {branch-name}`
4. （可能であれば）PRコマンドを出力、またはユーザーにPR作成を案内する

### Step 7: コンテキストの保存と次ステップの案内

1. 実装計画書（`docs/{dir-name}/03-plans.md`）の該当PR／Taskに「完了」チェックを入れる
2. 次のPRがある場合は、**「現在のPRが完了しました。続けて次のPR ({次のPR名}) に進みますか？」** とユーザーに尋ねる
3. すべてのPRが完了している場合は、コンテキストの現在のステージを「設定完了 (completed)」として終了を宣言する

## ドメイン品質テストの組み込みルール

以下の条件に該当する場合、自動的に対応するテストパターンを追加する:

| 実装内容 | 必須テスト | 参照 |
|---------|----------|------|
| 金額フィールドがある | 高精度小数型アサーション | testing-patterns.md §5 |
| トランザクション宣言がある | ロールバックテスト | testing-patterns.md §1 |
| 状態変更操作がある | 監査ログ検証テスト | testing-patterns.md §2 |
| バージョンフィールド（楽観ロック）がある | 並行アクセステスト | testing-patterns.md §3 |
| 冪等キーがある | 冪等性テスト | testing-patterns.md §4 |
