---
name: using-git-worktrees
description: Git worktree を使用して隔離された開発環境を作成する。メインのワーキングツリーを汚さずに並行開発やプロトタイプを行いたい場合に使用。「worktree」「並行開発」「隔離」「別ブランチで同時に」「プロトタイプ」というキーワードに反応。
argument-hint: "[branch-name] - worktreeで作成するブランチ名（例: feature/new-api）"
allowed-tools: Read, Glob, Grep, Bash(git *), Bash(ls *), Bash(mkdir *), Bash(npm *), Bash(pip *), Bash(cargo *), Bash(go *), Bash(dotnet *), Bash(./gradlew *), Bash(./mvnw *), Bash(make *)
---

> "adflow の `/using-git-worktrees` スキルを使用して、隔離された開発環境を作成します。"

# Git Worktree スキル

## 鉄則（絶対ルール）

1. **プロジェクトローカルのworktreeディレクトリは必ず .gitignore に含める** — worktreeの中身がリポジトリにコミットされることを防ぐ。
2. **worktree作成後は必ずベースラインテストを実行する** — 新しいworktreeが正常な状態であることを確認してから作業を開始する。
3. **不要になったworktreeは速やかに削除する** — 放置されたworktreeはディスクスペースを消費し、混乱の原因になる。
4. **worktree内で `git checkout` しない** — worktreeのブランチ変更は `git worktree` コマンドで行う。

## Red Flags — よくある合理化

| 思考 | 現実 |
|------|------|
| 「.gitignore はあとで設定しよう」 | 忘れてworktreeの中身をコミットするリスクがある。先に設定する |
| 「テストは本体で通ってるから大丈夫」 | worktreeは独立した環境。依存関係のインストールが必要な場合がある |
| 「このworktree、あとで使うかも」 | 使わないなら削除する。必要になったらまた作れる |

## ワークフロー

### Step 1: worktree ディレクトリの決定

以下の優先順位でworktreeの配置先を決定する:

1. **既存のworktreeディレクトリを確認**:
   ```
   ls -d .worktrees/ worktrees/ 2>/dev/null
   ```
2. **プロジェクトの慣習を確認**: CLAUDE.md やドキュメントにworktreeの配置先が記載されている場合はそれに従う
3. **デフォルト**: `.worktrees/` をプロジェクトルートに作成する

### Step 2: .gitignore の確認

プロジェクトローカルにworktreeを作成する場合、`.gitignore` に含まれていることを確認する:

```bash
# .gitignore にworktreeディレクトリが含まれているか確認
grep -q '.worktrees' .gitignore 2>/dev/null
```

含まれていない場合は追加する:

```bash
echo '.worktrees/' >> .gitignore
```

**グローバルディレクトリ（プロジェクト外）の場合、この手順はスキップする。**

### Step 3: worktree の作成

```bash
# プロジェクト名を取得
project_name=$(basename $(git rev-parse --show-toplevel))

# worktreeを作成（新しいブランチ）
git worktree add .worktrees/{branch-name} -b {branch-name}

# または既存ブランチをチェックアウト
git worktree add .worktrees/{branch-name} {branch-name}
```

### Step 4: 依存関係のインストール

worktree内で依存関係をインストールする（ビルドシステムに応じて自動検出）:

| ビルドシステム | コマンド |
|-------------|---------|
| Node.js | `cd .worktrees/{branch} && npm install` |
| Python | `cd .worktrees/{branch} && pip install -e .` |
| Rust | `cd .worktrees/{branch} && cargo build` |
| Go | `cd .worktrees/{branch} && go mod download` |
| Gradle | `cd .worktrees/{branch} && ./gradlew dependencies` |
| Maven | `cd .worktrees/{branch} && ./mvnw dependency:resolve` |

### Step 5: ベースラインテストの実行

```bash
cd .worktrees/{branch-name}
# ビルドシステムに応じたテストコマンドを実行
# 例: npm test, ./gradlew test, pytest, cargo test, go test ./...
```

テストが失敗する場合は、worktreeの状態に問題がある可能性がある。メインのワーキングツリーで同じテストが通るか確認する。

### Step 6: 完了報告

以下の情報を報告する:

```markdown
# Worktree 作成完了

- **場所**: .worktrees/{branch-name}
- **ブランチ**: {branch-name}
- **ベースラインテスト**: PASS / FAIL
- **依存関係**: インストール済み / スキップ

作業開始準備が完了しました。
```

## worktree の削除

作業完了後、またはworktreeが不要になった場合:

```bash
# worktreeを削除
git worktree remove .worktrees/{branch-name}

# 強制削除（未コミットの変更がある場合 — ユーザーの確認を取ること）
git worktree remove --force .worktrees/{branch-name}
```

## worktree 一覧の確認

```bash
git worktree list
```

## stack-loop との連携

`/stack-loop` で並行してPRを開発する場合、各PRをworktreeで隔離することで安全に作業できる:

1. `/using-git-worktrees feature/pr1-基盤` でworktreeを作成
2. worktree内で `/stack-loop` を実行
3. PR完了後にworktreeを削除
4. 次のPRのworktreeを作成して繰り返す
