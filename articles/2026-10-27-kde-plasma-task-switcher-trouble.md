---
title: "KDE Plasma でタスクスイッチャーでウィンドウのプレビューが表示されなくなった"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["KDE", "KDEPlasma", "ArchLinux"]
published: false
---

ノートパソコンにインストールしているArchLinux + KDE Plasma で、Alt+Tab で表示されるタスクスイッチャーにウィンドウプレビューが表示されなくなってしまった。

画面イメージとしては[このような感じ](https://www.reddit.com/r/kde/comments/1kf6q66/no_window_previews_in_task_switcher_on_arch_kde/) (スクショがうまくとれなかったので、 Reddit で見つけた記事を引用).

同じ問題が、タッチパッドで4本指で上方向へすワイプしたときに表示されるオーバービューでも起きる。

# 原因

いろいろ切り分けを行ったが (後述)、結論としては `~/.config/kwinrc` という設定ファイルあった次の設定に原因があった。

```
[QtQuickRendererSettings]
SceneGraphBackend=software
```

これを削除し解決。

AIちゃんいわくですが、

> `kdeglobals` の `SceneGraphBackend=software` が、**Qt Quick (QML) の描画エンジンを強制的にソフトウェアレンダラーに切り替えていた**ことが根本原因。
> 
> この設定はQtの自己保護機能で、過去に何らかのGPUドライバ関連のクラッシュが起きた際、以降の安全のため自動的に書き込まれたものと推測される。

とのことだった。

# 切り分けのために行ったこと

当初はパッケージ更新に伴う問題だと思いこんでいたため、AIと相談しつつ、これらのことを行いながら検証・切り分けを行っていました.

1. kwin, plasma-workspace, libplasma, kdeplasma-addons を 6.8.1 から6.8.0 へダウングレード

2. metaを 26.2.5 から 26.2.4 へダウングレード

3. kwayland, kwindousystem を更新前バージョンにダウングレード

4. qt6-base, qt6-declarative を 6.12.0 から 6.11.2 へダウングレード

    * SDDMのログイン画面が壊れてログイン不可になり、 Qt のロールバックはあきらめた

5. ディスプレイスケールを125%から100%へ

6. `~/.cache/mesa_shader_cache`, `~/.cache/kwin/qmlcache` を削除し再生成

7. `kwinrc` を一時的に初期化

8. **検証用ユーザーを作成し、ユーザープロファイル関連のファイルを調べたところ `kwinrc` にある決定的な違いを発見し解決**

# 所感

Claude に付き合ってもらったためか、普段ならためらいそうになるパッケージバージョンのロールバックから試してしまい、「すわ上流のバグでは？」と結論づけそうになった。

ふと自分がどうやってこういうケースで切り分けしていたかなと考えて、ダミーのユーザーを作成して試し、解決に至った。ユーザープロファイル系の問題なのか、パッケージに起因する問題なのかは、こういう切り分け方が有効であることを思い出した。
