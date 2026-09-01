## Context

Zenn の記事管理リポジトリにおいて、新しい記事を作成する際の手順（ブランチ作成、`zenn-cli` での初期ファイル生成、PR 作成）を自動化し、GitHub Issue をトリガーとしてシームレスに執筆を開始できるようにする。

## Goals / Non-Goals

**Goals:**
- GitHub Issue Form を使用して、記事のタイトル・タイプ (`tech`/`idea`)・絵文字 (`emoji`) を入力可能にする。
- Issue 起票をトリガーに GitHub Actions を起動し、`npx zenn new:article` を実行して記事雛形を生成する。
- 記事用の新ブランチを作成・コミット・プッシュし、PR を自動作成する。
- PR と Issue を自動リンク (`Resolves #<issue_number>`) する。

**Non-Goals:**
- 本（Books）作成の自動化（今回は記事: Articles のみ対象）。
- スラッグ（Slug）のカスタム指定（Zenn CLI による自動生成に任せる）。
- 記事の自動公開（PR マージおよび Zenn の自動同期で行う）。

## Decisions

### 1. Issue Form (`.github/ISSUE_TEMPLATE/new_article.yml`) の採用
- **Rationale**: Markdown 形式の自由記述テンプレートではなく、YAML 形式の Issue Form を採用することで、入力必須チェックやプルダウン選択 (`tech`/`idea`) を強制し、Actions 側でのパースを容易・堅牢にする。

### 2. スラッグの自動生成
- **Rationale**: ユーザーにスラッグの命名規則（文字数・形式）を意識させず、Zenn CLI の自動生成機能（ランダム英数字14文字）に任せることで入力の手間とバリデーションエラーを防ぐ。

### 3. GitHub CLI (`gh`) による Pull Request 作成
- **Rationale**: サードパーティの GitHub Action に依存せず、GitHub Actions 標準環境に組み込まれている `gh pr create` を使用することで、セキュリティと保守性を高める。
- **Permissions**: ワークフローに `contents: write`, `pull-requests: write`, `issues: write` を付与する。

### 4. Issue Body からのデータ抽出方法
- **Rationale**: GitHub Actions 内で Issue Form の Markdown 出力をパースするために、正規表現またはスクリプト (Bash / Node.js 等) を用いてセクションヘッダー下の値を取得する。

## Risks / Trade-offs

- **[Risk] Issue Form 以外の Issue からワークフローが誤爆する** → **Mitigation**: ワークフローの `if:` 条件で、特定のラベル（例: `article-draft`）が付与されている場合のみ動作するようにフィルタリングする。
- **[Risk] 絵文字入力で複数文字や不正文字が指定される** → **Mitigation**: Issue Form 側で文字数指定の説明を明記し、Actions 側でも先頭1文字または適切なフォールバックを行う。
