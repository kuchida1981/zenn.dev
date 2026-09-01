---
title: "個人開発でここまでやるか?と思いつつ作った、家族用Vaultwardenの自宅ならぬ自クラウド運用基盤"
emoji: "🔐"
type: "tech"
topics: ["terraform", "gcp", "tailscale", "githubactions", "vaultwarden"]
published: false
---

## はじめに

家族で使うパスワードマネージャーとして [Vaultwarden](https://github.com/dani-garcia/vaultwarden)(Bitwarden互換のOSSサーバー実装)をGCP上にセルフホストしています。この記事では、その運用基盤をTerraform + GitHub Actions + Tailscaleでどう組んだかを紹介します。

個人利用とはいえ「マスターパスワードの入り口」を預かるシステムなので、以下を最低限のラインとして設計しました。

- 管理画面や22番ポートをインターネットに一切晒さない
- バージョンアップは自動反映せず、人間の承認を挟む
- インフラの変更もPull Requestとレビューを経る(自分しかいなくても)
- バックアップは自宅NASへ、世代管理付きで

リポジトリはパブリックです: https://github.com/kuchida1981/vaultwarden-ops

## 全体構成

```mermaid
flowchart TB
    internet["インターネット(誰でも)"]
    subgraph vm["GCE VM (e2-micro, asia-northeast1, Debian13)"]
        caddy["Caddy (TLS終端)<br/>/ → Vaultwarden<br/>/admin → tailnetのみ"]
        disk[("データディスク: VMとは独立したPersistent Disk")]
    end
    admin["管理者は tailscale ssh のみ<br/>(22番ポートは公開しない)"]
    ci["vaultwarden-deploy.yml<br/>(承認ゲート付き、GCP IAPトンネル経由)"]

    internet -->|"443のみ"| vm
    vm -->|"Tailscale (WireGuard)"| admin
    vm -->|"GCP IAPトンネル (tcp:22)"| ci
```

構成要素はシンプルです。

- **GCE e2-micro 1台**(東京リージョン)に Caddy + Vaultwarden を Docker Compose で同居させる
- 公開しているのは443番ポートだけ。22番はファイアウォールで塞ぎ、運用アクセスは**Tailscale経由の `tailscale ssh` のみ**
- `/admin` パネルは公開ドメインからは常に403を返し、`tailscale serve` でtailnet内だけに別途公開
- CI/CDからのSSHだけは例外で、GCPのIAPトンネル経由で承認ゲート付きの自動化に許可している
- データはVM本体とライフサイクルを分離した専用Persistent Diskに置き、毎日自宅Synology NASへrsyncでバックアップ

この「人間は完全にTailscale経由、CIだけはIAP経由で特別扱い」という非対称な設計が今回のポイントの一つです。

## Terraformをbootstrapとmainの2段に分けた理由

`terraform/`配下は `bootstrap` と `main` の2ディレクトリに分かれています。

- `terraform/bootstrap`: **手動apply、一度きり**。GCSのstateバケット、GitHub Actions用のWorkload Identity Federation(WIF)プール、Terraform実行用のCIサービスアカウントそのものを作る
- `terraform/main`: **GitHub Actionsが継続的にapply**。VM、ファイアウォール、ディスク、Secret Manager、Tailscale ACL/認証キーなど実際のインフラ

なぜ分けるかというと、「CIサービスアカウント自身の権限を、そのCIサービスアカウント自身に管理させない」ためです。もし `bootstrap` もCIでapplyしてしまうと、CIの実行主体が自分自身に権限を追加できてしまい、権限昇格の温床になります。そのため `bootstrap` は意図的にCI化せず、`terraform-plan.yml` でPRごとに `terraform plan` の結果だけコメントし、実際の `apply` は人間がローカルで叩く運用にしています。

一方 `main` はDependabotが週次でプロバイダのバージョンを上げるPRを送ってくるので、`terraform-plan.yml` → レビュー → mainマージ → `terraform-apply.yml` が起動し、GitHub Environmentの`production`承認ゲートで一時停止 → 承認、という流れを自動化しています。

## Vaultwardenのバージョンアップも「マージ」と「デプロイ」を分離

Vaultwardenのイメージタグは `vaultwarden/docker-compose.yml` に固定値で書いてあり、Dependabotが新バージョンを検知するとPRを出してくれます。ここでのポイントは、**PRをマージしただけでは本番に何も反映されない**という設計です。

1. **バージョンを受け入れる**: PRをレビューしてmainにマージ(この時点でVM側は無変更)
2. **今デプロイする**: マージをトリガに `vaultwarden-deploy.yml` が起動し、`production` Environmentの承認待ちで一時停止。承認すると、CIランナーがGCP IAPトンネル経由でVMにSSHし、`git pull --ff-only && docker compose pull && docker compose up -d` を実行

「マージ」と「本番反映」を明確に分けることで、深夜にDependabotのPRを機械的にマージしても勝手に本番が更新されることはなく、実際のロールアウトタイミングは常に自分の意思で選べます。VM自体は再起動しないので、Caddyが持っているTLS証明書やconfigのボリュームにも影響が出ません。

## Tailscaleでネットワーク境界を作る

このリポジトリでのTailscaleの使い所は3つあります。

1. **運用SSHの唯一の経路**: 22番ポートはGCPファイアウォールで公開せず、`tailscale ssh` だけを許可
2. **管理画面の隔離公開**: `/admin` は `tailscale serve` でtailnet内のMagicDNSホスト名(`https://vaultwarden.<tailnet名>.ts.net/admin`)にのみ公開し、パブリックドメイン経由では常に403
3. **NASバックアップの転送経路**: VMから自宅NASへのrsyncも同じtailnet内で完結させ、インターネットに一切出さない

ACLやTailscale側の認証キー発行も `terraform/main/tailscale.tf` でTerraform管理下に置いているので、「誰が admin タグを持てるか」も含めてコードでレビューできます。

## バックアップ: Btrfsスナップショットに世代管理を丸投げする

Vaultwarden側のバックアップスクリプトは、DBのconsistentなスナップショット(`sqlite3 .backup`)・添付ファイル・Send・署名鍵・設定ファイルをまとめて、毎日Tailscale経由でSynology NASのrsyncデーモンへpushします。

工夫したのは、**世代管理(何日分残すか)をVM側では一切持たず、NAS側のBtrfsスナップショット機能に丸投げした**点です。rsyncは常に「最新の状態を上書きする」だけのシンプルな役割に留め、7日分・4週分・3ヶ月分といった世代管理はNASのSmart Retentionルールに任せています。認証もrsyncdのパスワードのみで、SSH鍵ペアの管理をしていません。これは通信がTailscaleのWireGuardトンネル内に閉じているため許容している判断で、`openspec/changes/archive/2026-07-12-add-nas-backup/design.md` に判断理由を残しています。

## OpenSpecでインフラ変更もスペック駆動にした

このリポジトリは実装をすべて[Claude Code](https://claude.com/claude-code)と一緒に進めていますが、単に「変更を頼んで終わり」にはせず、[OpenSpec](https://github.com/Fission-AI/OpenSpec)というスペック駆動開発のワークフローに乗せています。`openspec/changes/archive/` を見ると、これまでの変更が全部proposal(なぜやるか・設計・タスク一覧)として残っていることが分かります。

```
2026-07-07-add-vaultwarden-hosting
2026-07-08-add-smtp-support
2026-07-12-add-nas-backup
2026-07-12-hash-admin-token
2026-07-12-route-admin-via-tailscale-serve
2026-07-24-enable-bootstrap-ci-plan
2026-07-24-migrate-bootstrap-state-to-gcs
2026-07-29-add-vaultwarden-deploy-pipeline
2026-08-02-disable-email-duo-2fa
2026-08-04-fix-vaultwarden-ip-header
```

一人で回している個人インフラであっても、「なぜこの設計にしたか」をproposalとして先に固め、実装後にspecへアーカイブする、というサイクルを踏むことで、数ヶ月後の自分が読んでも意思決定の背景が追える状態を維持できています。実際、上記のような小粒な変更(管理者トークンのハッシュ化、admin導線をTailscale serve経由に変更、IPヘッダの修正など)も、思いつきで直接VMを触るのではなく、必ずこのサイクルを通しています。

## まとめ

- 家族用の小さなVaultwardenサーバーでも、「公開領域は最小限」「人間の承認を挟む」「変更はレビュー可能な形で残す」という原則は個人開発でも十分効果がある
- Terraformのbootstrap/main分離は、CI用サービスアカウントに自己権限昇格をさせないための設計として汎用的に使える
- Tailscaleを「管理アクセスの唯一の入口」として扱うと、パブリックな攻撃対象を大きく減らせる
- インフラの意思決定もOpenSpec + Claude Codeでスペック駆動にしておくと、後から読み返せる資産になる

まだ「マスターパスワードを完全に忘れた場合の復旧手段(Organization Account Recovery / Emergency Access)」など未対応の部分もあり、Roadmapとして残しています。興味があればリポジトリも覗いてみてください。

https://github.com/kuchida1981/vaultwarden-ops
