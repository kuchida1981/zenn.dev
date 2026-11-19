---
title: "Ubuntu で Antigravity CLI の Terminal Sandbox が出ない原因と AppArmor での有効化"
emoji: "🛡️"
type: "tech"
topics: ["ubuntu", "apparmor", "antigravity", "linux", "sandbox"]
published: false
---

# はじめに

Arch Linux 上にインストールした Antigravity CLI（`agy`）では `/config` に Terminal Sandbox や Auto モードの項目が表示されるのに、**Ubuntu 環境では同じ項目が一切表示されない**という現象に遭遇しました。

システム全体のセキュリティ制限（`kernel.apparmor_restrict_unprivileged_userns = 1`）を緩めることなく、原因の特定から専用 AppArmor プロファイルによる安全な有効化までを行えたため、調査手順と対処方法をまとめます。

# 環境

- OS: Ubuntu 26.04.1 LTS (Resolute Raccoon)
- カーネル: `7.0.0-43-generic`
- Antigravity CLI: `agy 1.5.12`（`~/.local/bin/agy`）
- 比較対象: Arch Linux 上の Antigravity CLI（`/config` に Sandbox ON/OFF 項目が表示される）

# 現象

- Ubuntu 上の `agy 1.5.12` で `/config` を開いても、Sandbox Mode や Auto モードの選択肢が表示されない。
- `agy update` を実行しても最新バージョンと表示される（＝バージョンが古いことが原因ではない）。

# 原因調査

## 1. `agy` の起動ログ（`cli.log`）の確認

`~/.gemini/antigravity-cli/cli.log` を確認したところ、起動直後に以下のログが出力されていました。

```text
W1119 14:18:06.947043     288 store.go:31] Terminal sandbox unavailable: AppArmor restricts unprivileged user namespaces (kernel.apparmor_restrict_unprivileged_userns = 1)
I1119 14:18:06.947051     288 manager.go:1740] auto mode is unavailable because the terminal sandbox is unavailable, falling back to default
```

つまり、`agy` 自体が起動時に Ubuntu の AppArmor による非特権 user namespace 制限を検知し、**Terminal Sandbox を利用不可と判定して関連設定を非表示（フォールバック）にしていた**ことが直接の原因でした。

## 2. `sysctl` と `unshare` / `bwrap` の挙動の違い

システム側の設定と、user namespace を扱う代表的なコマンドの挙動を確認しました。

```bash
$ sysctl kernel.apparmor_restrict_unprivileged_userns kernel.unprivileged_userns_clone user.max_user_namespaces
kernel.apparmor_restrict_unprivileged_userns = 1
kernel.unprivileged_userns_clone = 1
user.max_user_namespaces = 248257
```

- `kernel.unprivileged_userns_clone` は `1` なので、非特権 user namespace の作成自体はカーネル側で許可されています。
- `user.max_user_namespaces` は十分な数があり、単純な上限ゼロではありません。
- 一方で、コマンドごとの挙動は以下のように分かれました。

```bash
# UID マッピングなしの unshare は成功する
$ unshare --user id
uid=65534(nobody) gid=65534(nogroup) ...

# --map-root-user を付けると失敗する
$ unshare --user --map-root-user true
unshare: 書き込みに失敗しました /proc/self/uid_map: 許可されていない操作です

# bubblewrap (bwrap) は成功する
$ bwrap --ro-bind / / true
# (exit 0)
```

## 3. AppArmor プロファイルの仕組み

`unshare` が失敗し `bwrap` が成功する理由は、`/etc/apparmor.d/` にあるプロファイルの違いにあります。

- `/etc/apparmor.d/unprivileged_userns`
  - `unconfined` なプロセスが非特権 user namespace を作成すると、自動的に `unprivileged_userns` プロファイルへ遷移します。
  - このプロファイルには `audit deny capability,` が設定されているため、user namespace 自体は作れても capability（`CAP_SETUID` 等）が剥奪され、`/proc/self/uid_map` への書き込みなどで `EPERM` になります。
- `/etc/apparmor.d/bwrap-userns-restrict`
  - `/usr/bin/bwrap` 専用のプロファイルが用意されており、`allow capability,` と `allow userns,` が明示的に許可されているため、`bwrap` 単体は動作します。

なお、`agy` 自身の拒否ログは `journalctl -k -b` では確認できませんでした。他プロセスの大量の監査ログにより `audit_backlog_limit=64` を超えてログが抑制・欠落（`audit_lost` / `kauditd_printk_skb: ... callbacks suppressed`）していましたが、ログが溢れる前の起動直後に、`agy` とは別のプロセス（`ubuntu-insights`）で、同じ仕組みによる以下の遷移が記録されていました。

```text
apparmor="AUDIT" operation="userns_create" class="namespace" info="Userns create - transitioning profile" profile="unconfined" pid=15190 comm="(ubuntu-insigh)" requested="userns_create" target="unprivileged_userns" execpath="/usr/lib/systemd/systemd-executor"
apparmor="DENIED" operation="capable" class="cap" profile="unprivileged_userns" pid=15190 comm="(ubuntu-insigh)" capability=21  capname="sys_admin"
```

# 解決方法（システム全体の制限を維持して有効化する）

`sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0` としてシステム全体の制限を解除する方法もありますが、すべての非特権プロセスに対して user namespace 内での capability が開放され、カーネルの攻撃対象面が広がるため避けたいところです。

そこで、Ubuntu 24.04 以降で Chrome や Electron アプリ等でも採用されている **「対象バイナリ専用の AppArmor プロファイルを作成し、`userns` だけを許可する」** 方法をとりました。

## 1. `/etc/apparmor.d/agy` の作成

`~/.local/bin/agy` 向けのプロファイルを作成します（root 権限が必要）。AppArmor のプロファイルでは `~` が使えないため、`@{HOME}` でホームディレクトリを指定します。

```apparmor
abi <abi/5.0>,
include <tunables/global>

profile agy @{HOME}/.local/bin/agy flags=(unconfined) {
  userns,

  include if exists <local/agy>
}
```

:::message
`abi <abi/5.0>,` は Ubuntu 26.04 の環境（`/etc/apparmor.d/unprivileged_userns` 等）に合わせた指定です。Ubuntu 24.04 等の環境では `abi <abi/4.0>,` になる場合があります。
:::

### このプロファイルの意味

- `flags=(unconfined)`: `agy` 自体は通常どおり（`unconfined` と同等に）動かします。
- `userns,`: user namespace の作成を明示的に許可します。これにより、`unconfined` プロセスが user namespace を作ると `unprivileged_userns`（`audit deny capability,`）へ遷移して capability を剥奪される挙動が起きなくなり、`/proc/self/uid_map` への書き込みも通ります。
- 許可の対象は `agy` のバイナリパスに限定されます。他のプロセスには従来どおり `kernel.apparmor_restrict_unprivileged_userns = 1` の制限がかかります。

:::message alert
トレードオフとして、`agy` とその子プロセスは user namespace 内で capability を使えるようになります。信頼できるバイナリにだけ適用してください。
:::

## 2. プロファイルの読み込みと `agy` の再起動

```bash
sudo apparmor_parser -r /etc/apparmor.d/agy
```

- OS や AppArmor サービス全体の再起動は不要で、`apparmor_parser -r` 実行時点でカーネルに即時反映されます。`/etc/apparmor.d/` に置いたプロファイルは、再起動後も AppArmor サービスにより自動で読み込まれます。
- ただし、AppArmor のプロファイル適用はプロセス起動（`execve`）時に行われるため、**すでに起動中の `agy` は一度終了して再起動する必要があります**。

# 動作確認結果

`agy` を再起動したあと、Terminal Sandbox 内で実行されるコマンドから状態を確認しました。

- **分離された PID 名前空間での動作**
  - サンドボックス内から `ps -eo pid,comm,label` を実行すると、`PID 1` が `exe` となっており、PID 名前空間が分離されていることが確認できました。
- **AppArmor ラベルの遷移**
  - `ps` の `label` 列が `unconfined` ではなく `agy (unconfined)` になっていました。
- **ファイルシステム分離**
  - サンドボックス内のシェルからは `~/.gemini/antigravity-cli/log/` へのアクセスが遮断されており、保護境界が機能していることが確認できました。

# まとめ

- Ubuntu で Antigravity CLI の Terminal Sandbox が無効化されていた原因は、`kernel.apparmor_restrict_unprivileged_userns = 1` による非特権 user namespace の capability 制限でした。
- システム全体の `kernel.apparmor_restrict_unprivileged_userns` を `0` に下げるのではなく、`/etc/apparmor.d/agy` に `flags=(unconfined)` + `userns,` のプロファイルを追加することで、システム全体の防御を維持したまま安全に Terminal Sandbox を有効化できました。
