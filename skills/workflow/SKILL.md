---
name: workflow
description: ミッションクリティカルシステム開発の8段階ワークフロー（コンテキスト収集→ADR→仕様書→スタックPR計画→実装ループ）をオーケストレーションする。各ステージの成果物存在チェックと承認ゲート管理を行う。
argument-hint: "[feature] [--from=stage] - 機能名と開始ステージ (stage: adr, spec, stack-plan, stack-loop)"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git *), Bash(./gradlew *), Bash(./mvnw *), Bash(npm *), Bash(npx *), Bash(pytest *), Bash(cargo *), Bash(go *), Bash(dotnet *), Bash(make *), Bash(ls *), Bash(find *), Bash(mkdir *)
---

# ワークフロー オーケストレーションスキル

## アクティブなワークフロー
!`for d in docs/*/; do if [ -f "$d.adflow-context.md" ]; then echo "--- $(basename $d) ---"; cat "$d.adflow-context.md"; echo; fi; done 2>/dev/null || echo "アクティブなワークフローなし"`

## 既存成果物
!`echo "=== ADR ==="; ls docs/*/*-adr.md 2>/dev/null || echo "なし"; echo "=== 設計書 ==="; ls docs/*/*-spec.md 2>/dev/null || echo "なし"; echo "=== 計画 ==="; ls docs/*/*-plans.md 2>/dev/null || echo "なし"`

## ビルドシステム
!`if [ -f build.gradle ] || [ -f build.gradle.kts ]; then echo "Gradle"; elif [ -f pom.xml ]; then echo "Maven"; elif [ -f package.json ]; then echo "Node.js"; elif [ -f pyproject.toml ] || [ -f setup.py ]; then echo "Python"; elif [ -f Cargo.toml ]; then echo "Rust"; elif [ -f go.mod ]; then echo "Go"; elif ls *.csproj >/dev/null 2>&1 || [ -f *.sln ]; then echo ".NET"; elif [ -f Makefile ]; then echo "Make"; else echo "不明"; fi`

## 引数の処理

`$ARGUMENTS` をパースする:

23. 1. `--from=` パラメータがある場合: 開始ステージを指定する
24.    - 有効な値: `adr`, `spec`, `stack-plan`, `stack-loop`
25.    - 無効な値の場合: 「有効なステージ: adr, spec, stack-plan, stack-loop」を提示して再入力を求める
26. 2. `--from=` 以外の引数: 機能名として使用する
27. 3. `$ARGUMENTS` が空の場合: ユーザーに機能名を確認する
28. 
29. **機能名ディレクトリ名の決定:**
30. - 機能名から英語のkebab-caseに変換する（例: 「振込機能」→ `transfer-service`）
31. - この機能名に4桁の連番プレフィックスがついたものを `{dir-name}` として以降のステップで使用する（例: `0001-transfer-service`）
32. 
33. **例:**
34. - `/workflow 振込機能` → 機能名「振込機能」、コンテキスト収集から開始
35. - `/workflow 振込機能 --from=stack-loop` → 機能名「振込機能」、Stage 8 (実装ループ) から開始
36. - `/workflow --from=spec` → 機能名をユーザーに確認、Stage 4 (仕様書) から開始

## ワークフロー概要

```
```
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
4. **承認ゲート**: ユーザーにADRの内容（人間によるバリデーション）を確認してもらう
   - 「承認」→ Stage 4 へ進む
   - 「修正」→ ADRを修正して再確認
   - 「スキップ」→ ADRなしで Stage 4 へ（非推奨の旨を伝える）
5. コンテキストファイルを更新（ADR完了、次ステージ: spec）

**エラーハンドリング:** ADR作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 4: Stage 4〜5 — 仕様書作成、自己レビュー

1. `/spec` スキルを呼び出す（機能名を引数として渡す）
2. `/spec` スキル内で、リサーチ・仕様書の定義・AI自己レビューが実行される
3. 仕様書が `docs/{dir-name}/02-spec.md` に生成されたことを確認する
4. **承認ゲート**: ユーザーに仕様書の内容を確認してもらう
5. コンテキストファイルを更新（仕様書完了、次ステージ: stack-plan）

**エラーハンドリング:** 仕様書作成に失敗した場合、エラー内容を表示し「リトライ / スキップ / 中止」を提案する

### Step 5: Stage 6〜7 — スタックPR計画作成、自己レビュー

1. `/stack-plan` スキルを呼び出す（機能名を引数として渡す）
2. `/stack-plan` スキル内で、独立した小さなPRのスタック計画（依存関係と実装順序）・AI自己レビューが実行される
3. 実装計画書が `docs/{dir-name}/03-plans.md` に生成されたことを確認する
4. **承認ゲート**: ユーザーにスタックPR計画の内容を確認してもらう
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
