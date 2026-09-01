## Why

GitHub Issue から記事の作成をトリガーできるようにすることで、記事執筆の開始（ブランチ作成、Zenn CLIでの雛形生成、PR作成）を自動化し、執筆者がスムーズに執筆作業へ移行できるようにする。

## What Changes

- GitHub Issue Form テンプレート (`.github/ISSUE_TEMPLATE/new_article.yml`) を追加し、記事作成に必要なパラメータ（タイトル、記事タイプ、アイキャッチ絵文字）を入力可能にする。
- GitHub Actions ワークフロー (`.github/workflows/create_article.yml`) を追加し、新規 Issue 起票時に自動で `npx zenn new:article` を実行して下書き記事を作成、ブランチのプッシュおよび PR 作成を行う。

## Capabilities

### New Capabilities
- `article-creation-workflow`: GitHub Issue起票をトリガーとしたZenn記事テンプレート作成・ブランチプッシュ・Pull Request作成の自動化ワークフロー。

### Modified Capabilities
<!-- None -->

## Impact

- 追加されるファイル:
  - `.github/ISSUE_TEMPLATE/new_article.yml`
  - `.github/workflows/create_article.yml`
- 必要な権限: GitHub Actions 実行における `contents: write`, `pull-requests: write`, `issues: write` 権限
