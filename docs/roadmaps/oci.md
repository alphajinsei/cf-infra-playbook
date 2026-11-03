# 学習ロードマップ：OCI 共通基盤（方式③）

最終更新：2026-11-03

## ゴール

[方式検討：OCI](../design/oci.md) の構成を、Terraform と Ansible で作って動かす。その上にサンプルアプリを載せ、③のサービスを作るときに読む資料（共通基盤とサンプルの README）までそろえる。

- 共通基盤：`oci-platform`（名前は仮）
- サンプル：`sample-oci-vm-docker`（名前は仮）。②と同じメモ帳を Node + Hono + Drizzle（MySQL）で作り、添付ファイルを Object Storage に置く
- 公開先：`sample-oci-vm-docker.alphajinsei.com`

## 進め方

- OCI は業務で触っているので、Cloudflare のときより粗い粒度で進める。
- コマンドは自分で打つ。コード（Terraform、Ansible、スクリプト、アプリ）は Claude が書く。
- 各 Step のコマンドは、その Step に入るときに Claude がここに書き足す。終わった項目にチェックを付ける。
- 分かったことは [勉強メモ](../notes/study-notes.md) の「OCI」の節に積み上げる。

| Step | 内容 | 主に学ぶこと |
| --- | --- | --- |
| 0 | 準備 | OCI の API キー、コンパートメント、state のバケット、予算 |
| 1 | 既存リソースの片付け | ボリュームのバックアップ、Always Free の枠の空け方 |
| 2 | ネットワーク | VCN、サブネット、ゲートウェイ、ルート表、NSG |
| 3 | VM と Bastion と Ansible | A1 / マイクロ、Bastion のセッション、Ansible の基本 |
| 4 | DB と Vault | HeatWave、Vault、インスタンス・プリンシパル |
| 5 | LB と公開 | Flexible LB、Origin 証明書、Cloudflare の DNS を Terraform で |
| 6 | サンプルアプリ | Docker、arm64 のビルド、OCIR、デプロイのスクリプト |
| 7 | 運用 | Monitoring、Notifications、Logging、OS の自動更新 |
| 8 | メールサーバー | Postfix / Dovecot、Email Delivery への中継、SPF・DKIM・DMARC |
| 9 | 仕上げ | 共通基盤の仕様、README、サービス開発ガイドの③の行 |

---

## Step 0：準備

- [ ] OCI CLI を WSL に入れ、API キーを作る（鍵は `infra/.secrets/` に置き、Claude は読まない）
- [ ] 共通基盤用のコンパートメントを作る（既存の雑多なリソースと分ける）
- [ ] Terraform の state 用のバケットを CLI で作り、バージョニングを有効にする
- [ ] 予算アラートを作る（0円を超えたらメール）。何かを作る前に置いておく
- [ ] Ansible と、Docker（buildx を含む）を WSL に入れる
- [ ] `oci-platform` のリポジトリを作る（`additionalDirectories` と CLAUDE.md に足す）

理解ポイント：

- OCI の API キー認証で、何が手元にあり、何が OCI に登録されているか。
- state のバケットを Terraform で作らないのはなぜか（Cloudflare のときと同じ）。

## Step 1：既存リソースの片付け

- [ ] マイクロ2台のブートボリュームと、50GB のブロックボリュームのバックアップを取る（5つまで無料）
- [ ] マイクロ2台、ブロックボリューム、HeatWave（Always Free は1台なので作り直す）、古い VCN を消す
- [ ] 翌日、Cost Analysis でバックアップが0円かを確かめる

## Step 2：ネットワーク

- [ ] VCN、サブネット5つ、Internet GW / NAT GW / Service GW、ルート表、NSG を Terraform で作る
- [ ] 翌日、Cost Analysis で NAT Gateway が0円かを確かめる

理解ポイント：

- サブネットのセキュリティ・リストと NSG の違い。
- mail を private ではなく public-dmz に置く理由。

## Step 3：VM と Bastion と Ansible

- [ ] ops（マイクロ）、app（A1 1/8GB）、mail（A1 1/4GB + ブロック 30GB）を Terraform で作る
- [ ] Bastion と、ラッパーのスクリプト `bin/oci-ssh` を作り、3台に ssh で入る
- [ ] Ansible で共通の設定を流す（OS の自動更新、iptables、Docker、mail のボリュームのマウント）
- [ ] コンソール接続（シリアルコンソール）で入れることも確かめておく

理解ポイント：

- Bastion のセッションは何を許可しているか。期限が切れたら何が起きるか。
- Terraform と Ansible の境目はどこか。

## Step 4：DB と Vault

- [ ] HeatWave を private-db に作る
- [ ] Vault を作り、DB の管理者パスワードを入れる
- [ ] ops から mysql クライアントでつなぐ

## Step 5：LB と公開

- [ ] Cloudflare の Origin 証明書（`*.alphajinsei.com`）を作り、OCI の LB に入れる
- [ ] LB を作る。nsg-lb で Cloudflare の IP 範囲だけを受ける
- [ ] 仮のコンテナ（nginx など）を1つ置き、サブドメインから見えるところまで

理解ポイント：

- Cloudflare → LB → app の各区間で、TLS はどこで終わり、どこからまた始まるか。

## Step 6：サンプルアプリ

- [ ] `sample-oci-vm-docker` を作る。手元で `docker compose up`（アプリ + MySQL のコンテナ）
- [ ] アプリ側の Terraform（LB の振り分け、DNS、バケット）を作る
- [ ] `npm run db:init`（スキーマとユーザー）と `npm run deploy`（arm64 のビルド → OCIR → app で入れ替え）
- [ ] 翌日、Cost Analysis で OCIR が0円かを確かめる

理解ポイント：

- ②の `npm run deploy` と比べて、何が増えたか。

## Step 7：運用

- [ ] アラーム（LB の異常なバックエンド、CPU・メモリ、HeatWave のストレージ）とメール通知
- [ ] コンテナのログの見方を決める（`docker logs` か、OCI Logging に送るか）
- [ ] app のコンテナを止めて、通知が来ることを確かめる

## Step 8：メールサーバー

- [ ] 予約パブリック IP の料金と、逆引き（PTR）の設定可否を調べる
- [ ] Postfix / Dovecot を Ansible で入れ、Email Delivery に中継する
- [ ] MX、SPF、DKIM、DMARC を Cloudflare に置き、Gmail と送受信できることを確かめる

## Step 9：仕上げ

- [ ] 共通基盤の README に、仕様（アプリが受け取るもの・守ること）とポートの台帳を書く
- [ ] サンプルの README を、[サービス開発ガイド](../service-guide.md) の6つの見出しで書く
- [ ] サービス開発ガイドの「方式の一覧」の③の行を埋める
