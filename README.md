# zenn.dev

[Zenn](https://zenn.dev) と GitHub リポジトリを連携して記事（Articles）や本（Books）を管理・公開するためのリポジトリです。

---

## 📁 ディレクトリ構成

```text
.
├── articles/   # 記事マークダウンファイルを配置
├── books/      # 本のコンテンツを配置
└── README.md
```

---

## 🚀 基本的な使い方

Zenn CLI を使用して記事・本の作成やプレビューを行います。

### 1. 記事・本の新規作成

```bash
# 新しい記事を作成
npx zenn new:article

# 新しい本を作成
npx zenn new:book
```

### 2. ローカルプレビュー

```bash
# プレビューサーバーを起動 (デフォルト: http://localhost:8000)
npx zenn preview
```

### 3. 公開

`main` ブランチに push することで、Zenn に自動同期されます。
（記事・本ファイル内の `published: true` に設定されているものが公開されます）

---

## 📖 参考リンク

- [Zenn CLI の導入と使い方](https://zenn.dev/zenn/articles/install-zenn-cli)
- [GitHub リポジトリで Zenn のコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)
- [Zenn の Markdown 記法一覧](https://zenn.dev/zenn/articles/markdown-guide)
