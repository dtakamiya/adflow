---
name: workflow
description: ミッションクリティカルシステム開発の8段階ワークフロー（コンテキスト収集→ADR→仕様書→スタックPR計画→実装ループ）をオーケストレーションする。各ステージの成果物存在チェックと承認ゲート管理を行う。「新機能」「実装したい」「機能追加」「作りたい」というキーワードに反応。
argument-hint: "[feature] [--from=stage] - 機能名と開始ステージ (stage: adr, spec, stack-plan, stack-loop)"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git *), Bash(./gradlew *), Bash(./mvnw *), Bash(npm *), Bash(npx *), Bash(pytest *), Bash(cargo *), Bash(go *), Bash(dotnet *), Bash(make *), Bash(ls *), Bash(find *), Bash(mkdir *)
---

> "adflow の `/workflow` スキルを使用して、ADR駆動の一気通貫ワークフローを実行します。"

# ワークフロー オーケストレーションスキル

## アクティブなワークフロー
!`for d in docs/*/; do if [ -f "$d.adflow-context.md" ]; then echo "--- $(basename $d) ---"; cat "$d.adflow-context.md"; echo; fi; done 2>/dev/null || echo "アクティブなワークフローなし"`

## 既存成果物
!`echo "=== ADR ==="; ls docs/*/*-adr.md 2>/dev/null || echo "なし"; echo "=== 設計書 ==="; ls docs/*/*-spec.md 2>/dev/null || echo "なし"; echo "=== 計画 ==="; ls docs/*/*-plans.md 2>/dev/null || echo "なし"`

## ビルドシステム
!`if [ -f build.gradle ] || [ -f build.gradle.kts ]; then echo "Gradle"; elif [ -f pom.xml ]; then echo "Maven"; elif [ -f package.json ]; then echo "Node.js"; elif [ -f pyproject.toml ] || [ -f setup.py ]; then echo "Python"; elif [ -f Cargo.toml ]; then echo "Rust"; elif [ -f go.mod ]; then echo "Go"; elif ls *.csproj >/dev/null 2>&1 || [ -f *.sln ]; then echo ".NET"; elif [ -f Makefile ]; then echo "Make"; else echo "不明"; fi`

## 引数の処理

`$ARGUMENTS` をパースする:

1. `--from=` パラメータがある場合: 開始ステージを指定する
   - 有効な値: `adr`, `spec`, `stack-plan`, `stack-loop`
   - 無効な値の場合: 「有効なステージ: adr, spec, stack-plan, stack-loop」を提示して再入力を求める
2. `--from=` 以外の引数: 機能名として使用する
3. `$ARGUMENTS` が空の場合: ユーザーに機能名を確認する

**機能名ディレクトリ名の決定:**
- 機能名から英語のkebab-caseに変換する（例: 「振込機能」→ `transfer-service`）
- この機能名に4桁の連番プレフィックスがついたものを `{dir-name}` として以降のステップで使用する（例: `0001-transfer-service`）

**例:**
- `/workflow 振込機能` → 機能名「振込機能」、コンテキスト収集から開始
- `/workflow 振込機能 --from=stack-loop` → 機能名「振込機能」、Stage 8 (実装ループ) から開始
- `/workflow --from=spec` → 機能名をユーザーに確認、Stage 4 (仕様書) から開始

## 鉄則（絶対ルール）

1. **承認ゲートを絶対にスキップしない** — 各ステージの成果物はユーザーの承認を得てから次に進む。AIが勝手に「問題なし」と判断して先に進むことは禁止。
2. **前段の成果物なしに次ステージへ進まない** — ADRなしに仕様書を書かない。仕様書なしに計画を立てない。計画なしに実装を始めない。
3. **コンテキストファイルを必ず更新する** — 各ステージ完了時に `.adflow-context.md` を更新し、ワークフロー状態を正確に記録する。
4. **ユーザーの明示的な指示なく次ステージを自動実行しない** — `/workflow` 経由の場合のみ自動進行が許可される。

## Red Flags — よくある合理化

| 思考 | 現実 |
|------|------|
| 「ADRは明白だからスキップしよう」 | 明白に見える決定こそ記録が必要。後で「なぜ？」と聞かれる |
| 「仕様書は簡単な機能だから不要」 | 簡単な機能でも仕様の認識齟齬は起きる |
| 「承認待ちは非効率だ」 | 手戻りの方がはるかに非効率 |
| 「前のADRがあるから新しいのは不要」 | 別の機能には別のADRが必要 |
| 「テストは後で書こう」 | TDDはプロセスの一部。後回しは禁止 |

## ワークフロー概要

```
Stage 1-3: ADR       → docs/{dir-name}/01-adr.md
    ↓ [承認ゲート]
Stage 4-5: 仕様書     → docs/{dir-name}/02-spec.md
    ↓ [承認ゲート]
Stage 6-7: PR計画    → docs/{dir-name}/03-plans.md
    ↓ [承認ゲート]
Stage 8: 実装ループ   → ブランチ作成→TDD→レビュー→PR作成
```

## ワークフロー実行

### Step 1: 機能の特定と開始ステージの決定

1. 引数から機能名と開始ステージを取得する
2. 機能名をkebab-caseに変換して対応する `docs/{dir-name}/` ディレクトリを特定または作成する
3. `docs/{dir-name}/.adflow-context.md` を初期化する（まだ存在しない場合）
4. 上記の「アクティブなワークフロー」に基づき、最後の完了ステージを自動検出する
5. `--from=` 指定がない場合、既存成果物から最適な開始ステージを提案する

### Step 2: 成果物存在チェック

以下のディレクトリをスキャンして既存の成果物を確認:

```
docs/{dir-name}/01-adr.md           → ADRファイルの有無
docs/{dir-name}/02-spec.md          → 仕様書ファイルの有無
docs/{dir-name}/03-plans.md         → 実装計画書ファイルの有無
テストディレクトリ                    → テストファイルの有無
ソースディレクトリ                    → 実装ファイルの有無
```

既存の成果物がある場合:
- ユーザーに現在のステージを提示する
- 途中のステージから再開するか確認する

### Step 3: Stage 1〜3 — コンテキスト収集、ADR作成、自己レビュー

1. `/adr` スキルを呼び出す（機能名を引数として渡す）
2. `/adr` スキル内で、コンテキスト収集・ブレインストーミング・ADR作成・AI自己レビューが実行される
3. ADRが `docs/{dir-name}/01-adr.md` に生成されたことを確認する
4. **承認ゲート（HARD-GATE）**: ユーザーにADRの内容を確認してもらう
   - ゲート条件チェックリスト:
     - [ ] ADRが `docs/{dir-name}/01-adr.md` に存在する
     - [ ] AI自己レビューで品質基準を満たしている
     - [ ] ユーザーに内容を提示済み
   - 「承認」→ Stage 4 へ進む
   - 「修正」→ ADRを修正して再確認
   - 「スキップ」→ ⚠️ **警告: ADRをスキップすると、以降のステージで設計根拠が不明確になり、手戻りリスクが大幅に増加します。本当にスキップしますか？**
5. コンテキストファイルを更新（ADR完了、次ステージ: spec）

**エラーハンドリング:** ADR作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 4: Stage 4〜5 — 仕様書作成、自己レビュー

1. `/spec` スキルを呼び出す（機能名を引数として渡す）
2. `/spec` スキル内で、リサーチ・仕様書の定義・AI自己レビューが実行される
3. 仕様書が `docs/{dir-name}/02-spec.md` に生成されたことを確認する
4. **承認ゲート（HARD-GATE）**: ユーザーに仕様書の内容を確認してもらう
   - ゲート条件チェックリスト:
     - [ ] 仕様書が `docs/{dir-name}/02-spec.md` に存在する
     - [ ] AI自己レビューで品質基準を満たしている
     - [ ] ユーザーに内容を提示済み
   - 「承認」→ Stage 6 へ進む
   - 「修正」→ 仕様書を修正して再確認
   - 「スキップ」→ ⚠️ **警告: 仕様書をスキップすると、実装の根拠が曖昧になり、品質保証が困難になります。本当にスキップしますか？**
5. コンテキストファイルを更新（仕様書完了、次ステージ: stack-plan）

**エラーハンドリング:** 仕様書作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 5: Stage 6〜7 — スタックPR計画作成、自己レビュー

1. `/stack-plan` スキルを呼び出す（機能名を引数として渡す）
2. `/stack-plan` スキル内で、独立した小さなPRのスタック計画（依存関係と実装順序）・AI自己レビューが実行される
3. 実装計画書が `docs/{dir-name}/03-plans.md` に生成されたことを確認する
4. **承認ゲート（HARD-GATE）**: ユーザーにスタックPR計画の内容を確認してもらう
   - ゲート条件チェックリスト:
     - [ ] 計画書が `docs/{dir-name}/03-plans.md` に存在する
     - [ ] AI自己レビューで品質基準を満たしている
     - [ ] ユーザーに内容を提示済み
   - 「承認」→ Stage 8 へ進む
   - 「修正」→ 計画書を修正して再確認
   - 「スキップ」→ ⚠️ **警告: 計画書をスキップすると、PRの分割やTDDステップが不明確になり、実装品質が低下します。本当にスキップしますか？**
5. コンテキストファイルを更新（計画書完了、次ステージ: stack-loop）

**エラーハンドリング:** 計画書作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 6: Stage 8 — スタックPR 実装ループ

1. `/stack-loop` スキルを呼び出す（機能名を引数として渡す）
2. 各PR（タスク）ごとに以下が反復実行される:
   - 前PRを親としたブランチ作成
   - TDD (Red -> Green -> Refactor)
   - ローカル検証 (Lint, 型チェック, テスト)
   - AI自己レビュー
   - コミット、Push、PR作成
   - 次ステップのコンテキスト保存
3. コンテキストファイルを更新（ループ完了、ステージ: completed）

### Step 7: 完了

1. 成果物の一覧を表示する:
   - ADR: `docs/{dir-name}/01-adr.md`
   - 仕様書: `docs/{dir-name}/02-spec.md`
   - 実装計画: `docs/{dir-name}/03-plans.md`
   - コンテキスト: `docs/{dir-name}/.adflow-context.md`
2. 今後の作業やマージの提案をする

## 途中再開

ワークフローは任意のステージから再開可能:

```
/workflow {feature} --from=spec        → Stage 4 から開始
/workflow {feature} --from=stack-plan  → Stage 6 から開始
/workflow {feature} --from=stack-loop  → Stage 8 から開始
```

既存のコンテキストファイル `docs/{dir-name}/.adflow-context.md` がある場合、前段の成果物を自動的に引き継ぐ。
