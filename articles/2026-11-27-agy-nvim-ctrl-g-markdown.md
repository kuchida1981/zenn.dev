---
title: "Antigravity CLI の Ctrl+G で開くプロンプトを Neovim で Markdown として扱う"
emoji: "📝"
type: "tech"
topics: ["neovim", "antigravity", "claudecode", "markdown"]
published: true
---

# はじめに

Claude Code や Antigravity CLI（`agy`）では、プロンプト入力欄で `Ctrl+G` を押すと、環境変数 `$EDITOR` などに設定した外部エディタでプロンプトを編集できます。長文のプロンプトや構造化した指示を書きたいときに非常に便利な機能です。

普段エディタとして **Neovim** を使っているのですが、両者を併用しているとちょっとした違いに気づきました。

- **Claude Code**: 一時ファイルが Markdown（`.md`）として開かれる
- **Antigravity CLI**: 一時ファイルがテキストファイル（`.txt`）として開かれる

Antigravity CLI で開いたときもプロンプト自体は Markdown 記法（見出しや箇条書き、コードブロックなど）で書くことが多いのですが、拡張子が `.txt` のままだと Markdown のシンタックスハイライトや関連プラグインが効かず、少し不便に感じていました。

この記事では、Antigravity CLI 側での設定可否や世の中の要望の有無を調べた結果と、Neovim 側の設定で解決した方法をまとめます。

# 背景・現象

Antigravity CLI で `Ctrl+G` を押して Neovim を起動すると、以下のようなパスの一時ファイルが開かれます。

```text
/tmp/jetski-prompt-xxxxxxxx.txt
```

拡張子が `.txt` であるため、Neovim 上では `filetype=text`（あるいは未設定）となり、Markdown としてのハイライトや折りたたみ、入力支援などが有効になりません。

# Antigravity CLI 側の仕様と設定可否

「そもそも Antigravity CLI 側の設定で拡張子を `.md` に変えられないのか？」を確認してみました。

- **設定ファイル（`settings.json`）の全項目を確認**
  - Antigravity CLI の設定ファイル（`~/.gemini/antigravity-cli/settings.json`）用の公式 JSON Schema（`https://antigravity.google/schemas/v1/settings.json`）を確認したところ、外部エディタ関連の設定は `editor`（起動するコマンド、デフォルトは `"auto"` で `$EDITOR` を参照）と `editorMode`（CLI 上のキーバインドを `"default"` / `"vim"` に切り替え）のみで、**一時ファイルのファイル名や拡張子を指定する設定項目は存在しません**。
- **リポジトリの `CHANGELOG.md` とバイナリを確認**
  - 公式リポジトリ（`google-antigravity/antigravity-cli`）の `CHANGELOG.md` にも、外部エディタの一時ファイル拡張子を変更できる機能や設定についての記載はありませんでした。
  - 手元の `agy` バイナリ（`1.5.18`）内の文字列を確認したところ、`jetski-prompt-*.txt` というパターンがバイナリ内に直接埋め込まれていました。

そのため、現時点では Antigravity CLI 側の設定だけで `.md` として開かせることはできません。

# 同じ要望や提言はないのか？

「プロンプトを Markdown で書く人は多いはずなので、すでに Issue などで要望が出ているのでは？」と思い、[`google-antigravity/antigravity-cli`](https://github.com/google-antigravity/antigravity-cli/issues) の Issue（オープン・クローズ済み含む）を検索してみました。

外部エディタ機能（`Ctrl+G` や、類似 CLI の `Ctrl+X`）に関しては、以下のような議論や不具合報告は見られます。

- `$EDITOR` とは別に CLI 専用の外部エディタ設定キーが欲しい
- GUI エディタで `--wait` フラグがないと即座に閉じた扱いになる
- 端末エディタの終了時にカーソル位置や端末状態が崩れることがある

一方で、`filetype`、`".txt" editor`、`"ctrl+g" markdown` などのキーワードで検索した範囲では、**「一時ファイルの拡張子を `.txt` ではなく `.md` にしてほしい（あるいは変更できるようにしてほしい）」という要望 Issue は見つかりませんでした**。

考えられる理由としては、以下のような点が挙げられそうです。

- ツール側が「プロンプト＝プレーンテキスト」として素直に `.txt` を採用している（なお、Google の `gemini-cli` でも同様に `buffer.txt` が使われています）
- シンタックスハイライトがなくても実用上は困らない、あるいは今回のようにエディタ側の設定で簡単に回避できるため、Issue として上がりにくい

# Neovim 側の設定で解決する

CLI 側で変えられないのであれば、Neovim 側でファイル名のパターンを見て `filetype` を `markdown` として認識させれば解決します。

Neovim 0.8 以降では `vim.filetype.add` が使えるため、`init.lua`（または設定読み込み先）に以下のように記述するのがシンプルでモダンです（手元の Neovim 0.11.6 で動作確認済み）。

```lua
vim.filetype.add {
  pattern = {
    [".*/jetski%-prompt%-.*%.txt"] = "markdown",
  },
}
```

## この設定のポイント

- **最初から `markdown` として判定される**: `BufRead` / `BufNewFile` の `autocmd` で後から `vim.bo.filetype` を上書きする方法と違い、Neovim 標準の filetype 検出パイプラインに組み込まれるため、最初から `markdown` として扱われます。
- **影響範囲を限定できる**: `jetski-prompt-*.txt` にマッチしたときだけ `markdown` になるため、普段開く通常の `.txt` ファイル（`text`）には影響しません。
- **OS を問わずマッチする**: 先頭に `.*/` を付けているため、Linux の `/tmp/jetski-prompt-*.txt` だけでなく、macOS の `/var/folders/.../jetski-prompt-*.txt` のような深い一時ディレクトリ配下でもマッチします。
- **Lua パターンでのエスケープ**: `vim.filetype.add` の `pattern` は Vim の glob ではなく **Lua パターン** で評価されるため、`*` ではなく `.*` を使い、特殊文字の `-` と `.` は `%-`・`%.` のように `%` でエスケープする必要があります。

:::message
Neovim 0.7 以前の環境や、`autocmd` ベースで書きたい場合は以下のように記述することもできます。

```lua
vim.api.nvim_create_autocmd({ "BufRead", "BufNewFile" }, {
  pattern = "*/jetski-prompt-*.txt",
  callback = function()
    vim.bo.filetype = "markdown"
  end,
})
```

なお動作確認は未実施。
:::

# まとめ

- Antigravity CLI の `Ctrl+G` で開く一時ファイルは `jetski-prompt-*.txt` 固定になっており、現時点では CLI 側の設定で `.md` に変更することはできません。
- 拡張子を `.md` にしてほしいという要望は、調べた限りでは目立った Issue 等としては見つかりませんでした。
- Neovim 側で `vim.filetype.add` を使って `.*/jetski%-prompt%-.*%.txt` に `markdown` を割り当てることで、他の `.txt` ファイルに影響を与えずに Markdown として快適にプロンプトを編集できるようになりました。
