## Context

GitHub Issue Form（`.github/ISSUE_TEMPLATE/new_article.yml`）から新しい記事を作成する際、フォームの入力体験およびワークフローの挙動を改善する。

現在の課題:
- `title` フィールドに固定値が設定されており、入力時に手動で消去が必要。
- `emoji` フィールドが必須で、初期値が入力されていない（プレースホルダーのみ）。
- PR作成完了後にIssueが自動でクローズされない（マージ時ではなく作成完了時点でクローズしたい）。

## Goals / Non-Goals

**Goals:**
- Issue作成時のタイトル入力を空欄スタート（GitHub標準プレースホルダー）にする。
- 絵文字入力を任意（`required: false`）にし、フォーム初期値として `📝` を与える。
- ワークフロー側で絵文字が未入力の場合のフォールバック（`📝`）を保証する。
- PR作成完了直後に `gh issue close "$ISSUE_NUMBER" --reason "completed"` を実行してIssueを自動クローズする。

**Non-Goals:**
- GitHub Issue Formの標準UI（Assignee, Label, Project, Milestoneなど）の非表示化（GitHub仕様上非表示にできないため対象外）。

## Decisions

1. **Issue FormのTitle設定**
   - 決定: `new_article.yml` からトップレベルの `title:` を削除する。
   - 理由: GitHub Issue Form ではプレースホルダー文字列を指定する専用プロパティがなく、`title` に文字列を指定すると入力値として扱われるため、空にしてGitHub標準のプレースホルダー「Title」に任せる。

2. **絵文字フィールドの設定**
   - 決定: `validations.required: false` に変更し、`attributes.value: "📝"` を指定する。
   - 理由: ユーザーが特に指定しなくてもデフォルトで `📝` が入力され、変更したい場合のみ編集できる。未入力送信時もワークフロー側で `📝` にフォールバックする。

3. **Issue の自動クローズ**
   - 決定: PR作成ステップの直後に `gh issue close` を実行する。
   - 理由: `Resolves #xxx` はPRマージ時のクローズ用であるため、PR作成完了時点でクローズするには GitHub CLI で明示的にクローズする必要がある。

## Risks / Trade-offs

- [Issueが早期にクローズされる] → PR本文に元のIssue番号へのリンク、およびIssue側にPRへの参照コメントが残るため、追跡性は損なわれない。
