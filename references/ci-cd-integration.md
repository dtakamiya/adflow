# CI/CD統合ガイド

adflowワークフローをCI/CDパイプラインに統合するためのガイド。

## Decision Guardianパターン

PRの変更差分に関連するADRを自動的にサーフェシングし、レビュアーに提示する。

### 仕組み

1. PRの変更ファイル一覧を取得する
2. 変更されたディレクトリ・モジュールに関連するADRを `docs/` から検索する
3. 関連ADRの決定事項をPRコメントとして投稿する
4. レビュアーがADRの文脈を理解した上でレビューできる

### GitHub Actions での実装例

```yaml
name: ADR Decision Guardian
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  adr-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Find related ADRs
        run: |
          # 変更ファイルの一覧を取得
          CHANGED_FILES=$(git diff --name-only ${{ github.event.pull_request.base.sha }} HEAD)

          # 変更ファイルに関連するADRを検索
          for adr in docs/*/01-adr.md; do
            dir=$(dirname "$adr")
            feature=$(basename "$dir")
            # ADR内のコンポーネント名・モジュール名を検索
            if echo "$CHANGED_FILES" | grep -qi "${feature}"; then
              echo "## 関連ADR: $adr" >> adr-comments.md
              echo "" >> adr-comments.md
              # 決定セクションを抽出
              sed -n '/^## 決定/,/^## /p' "$adr" | head -20 >> adr-comments.md
              echo "" >> adr-comments.md
            fi
          done

      - name: Post ADR context as PR comment
        if: -f adr-comments.md
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = fs.readFileSync('adr-comments.md', 'utf8');
            if (body.trim()) {
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: '### ADR Decision Guardian\n\n' + body
              });
            }
```

## フィットネス関数パイプライン

ADRの決定事項を自動検証するCIステージ。

### パイプライン構成

```yaml
name: ADR Fitness Functions
on: [push, pull_request]

jobs:
  fitness-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check decimal usage for monetary fields
        run: |
          # 金額フィールドにdouble/floatが使われていないか
          VIOLATIONS=$(grep -rn -E '(amount|price|balance|fee|total|cost).*\b(double|float|Double|Float)\b' \
            --include='*.java' --include='*.kt' --include='*.ts' --include='*.py' \
            src/ app/ lib/ || true)
          if [ -n "$VIOLATIONS" ]; then
            echo "::error::ADR違反: 金額計算にdouble/floatが使用されています"
            echo "$VIOLATIONS"
            exit 1
          fi

      - name: Check audit log coverage
        run: |
          # 状態変更操作に監査ログがあるか
          # プロジェクト固有のパターンに応じてカスタマイズ
          echo "監査ログカバレッジチェック完了"

      - name: Check transaction declarations
        run: |
          # データベース書き込み操作にトランザクション宣言があるか
          echo "トランザクション宣言チェック完了"

      - name: Check PII in logs
        run: |
          # ログ出力にPIIフィールドが含まれていないか
          VIOLATIONS=$(grep -rn -E 'log\.(info|debug|warn|error).*\b(ssn|password|creditCard|email)\b' \
            --include='*.java' --include='*.kt' --include='*.ts' --include='*.py' \
            src/ app/ lib/ || true)
          if [ -n "$VIOLATIONS" ]; then
            echo "::error::ADR違反: ログにPIIが含まれている可能性があります"
            echo "$VIOLATIONS"
            exit 1
          fi
```

## ADRステータス管理

### ADRライフサイクル

```
提案中 → 承認済 → [実装中] → [実装完了]
                ↓
              廃止 / 代替済（by ADR-XXXX）
```

### 定期レビュー

- 6〜12ヶ月ごとにADRを見直す
- 技術環境の変化に応じてADRを更新または廃止する
- `docs/*/01-adr.md` のステータスフィールドで管理する
