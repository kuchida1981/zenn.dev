## Why

IssueからZenn記事の下書きPRを自動作成するワークフローにおいて、以下のユーザビリティ上の課題があります。
1. Issueフォームのタイトル欄に固定文字列が初期入力されてしまっており、入力時に手動で消す手間が発生している。
2. アイキャッチ絵文字が必須になっており、デフォルト値（📝）の自動補完や任意入力に対応していない。
3. PR作成後に元のIssueがオープンのまま残り、手動クローズが必要になっている（PR作成時点でIssueの目的は達成されているため自動でクローズしたい）。

これらを改善し、最小限の入力でスムーズに記事作成PRが立ち上がり、Issueが適切にクローズされるフローを実現します。

## What Changes

- **Issueテンプレートの改善**:
  - `title` の初期値を削除し、GitHub標準の空のタイトル入力欄（プレースホルダー表示）にする。
  - アイキャッチ絵文字入力欄の必須属性（`required: true`）を解除し、初期値（`value: "📝"`）を設定する。
- **GitHub Actionsワークフローの改善**:
  - タイトル抽出ロジックをシンプルにし、入力されたIssueタイトルをそのまま利用する。
  - 絵文字が未入力または空文字の場合でも、デフォルト値 `📝` を使用するフォールバックを維持する。
  - PR作成成功後に GitHub CLI (`gh issue close`) を用いて元のIssueを即時クローズする。

## Capabilities

### New Capabilities
None

### Modified Capabilities
- `article-creation-workflow`: Issueフォームのタイトル・絵文字フィールドの要件変更（絵文字の任意化・デフォルト値提供）、およびPR作成後のIssue自動クローズ要件の追加。

## Impact

- `.github/ISSUE_TEMPLATE/new_article.yml`
- `.github/workflows/create_article.yml`
