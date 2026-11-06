# 学習ロードマップ：OCI 共通基盤（方式③）

最終更新：2026-11-07

状態：**完了**（2026-11-07）。8-5 の後日の確認（fail2ban、DMARC のレポート、`p=quarantine` への昇格）は、ロードマップの完了とは切り離して、いつかやる。

## ゴール

[方式検討：OCI](../design/oci.md) の構成を、Terraform と Ansible で作って動かす。その上にサンプルアプリを載せ、③のサービスを作るときに読む資料（共通基盤とサンプルの README）までそろえる。

- 共通基盤：`oci-platform`
- サンプル：`sample-oci-vm-docker`。②と同じメモ帳を Node + Hono + Drizzle（MySQL）で作る（添付ファイルの Object Storage は見送った。要るアプリが出たら共通基盤に足す）
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
| 6 | サンプルアプリ | Docker、VM の上でのビルド、Vault からのパスワードの受け渡し、デプロイのスクリプト |
| 7 | 運用 | Monitoring（停止のアラーム）、Notifications、Logging（LB のログ） |
| 8 | メールサーバー | メールの仕組み、Postfix / Dovecot / Rspamd、Email Delivery への中継、SPF・DKIM・DMARC、アプリからの送信 |
| 9 | 仕上げ | 共通基盤の仕様、README、サービス開発ガイドの③の行 |

---

## Step 0：準備

WSL には Terraform（1.17）と Docker（29、buildx あり）が入っている。足りないのは OCI CLI、Ansible、arm64 のエミュレーション。

### 0-1. OCI CLI を入れ、API キーを作る

- [x] OCI CLI を入れる（公式のインストーラー。`~/bin/oci` に入り、`~/.bashrc` の PATH も足される）

```bash
sudo apt install -y python3-venv
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)" -- --accept-all-defaults
exec bash -l        # PATH を読み直す
oci --version
```

- [x] `oci setup config` で設定ファイルと API キーを作る。聞かれたら次のように答える
  - config の場所：既定（`~/.oci/config`）
  - user OCID：コンソール右上のプロフィール → My profile の OCID
  - tenancy OCID：プロフィール → Tenancy の OCID
  - region：ホームリージョン（例：`ap-tokyo-1`）
  - 新しい鍵を作るか：`Y`。置き場所：`/home/alphajinsei/claude-sandbox/infra/.secrets/oci`（先に作っておく）。パスフレーズ：`N/A`（なし）

```bash
mkdir -p ~/claude-sandbox/infra/.secrets/oci && chmod 700 ~/claude-sandbox/infra/.secrets/oci
oci setup config
```

- [x] 公開鍵をコンソールに登録する：My profile → API keys → Add API key → Paste a public key に、次の出力を貼る。表示されたフィンガープリントが `~/.oci/config` の `fingerprint` と同じことを確かめる

```bash
cat ~/claude-sandbox/infra/.secrets/oci/oci_api_key_public.pem
```

- [x] つながるか確かめる（Object Storage のネームスペースが出れば OK）

```bash
oci os ns get
```

### 0-2. コンパートメントを作る

- [x] 共通基盤用のコンパートメント `oci-platform` を、ルート（テナンシー）の下に作る。既存の雑多なリソースと分け、権限や費用をこの単位で見られるようにする

```bash
TENANCY=$(awk -F= '/^tenancy/{print $2}' ~/.oci/config)
oci iam compartment create --compartment-id "$TENANCY" --name oci-platform \
  --description "Personal platform (method 3)" --wait-for-state ACTIVE
COMP=$(oci iam compartment list --compartment-id "$TENANCY" --name oci-platform --query 'data[0].id' --raw-output)
echo "$COMP"        # あとで Terraform に渡す。秘密ではない
```

### 0-3. state のバケットを作る

- [x] state 用のバケット `tfstate` を作り、バージョニングを有効にする（公開はしない。既定で非公開）

```bash
oci os bucket create --compartment-id "$COMP" --name tfstate --versioning Enabled
oci os bucket get --bucket-name tfstate --query 'data.{name:name,versioning:versioning,public:"public-access-type"}'
```

### 0-4. 予算アラートを作る

- [x] 予算は対象（テナンシー）1つにつき1つまでで、以前に作った月1,000円の予算がある。新しくは作らず、それに「実績が100円を超えたら」のアラートを足す。今もカスタムイメージの保管料で毎月数円かかっているので、1円では毎月鳴ってしまう。100円なら、時間課金のもの（Always Free の外に出た NAT や LB など）が動き出したときに気付ける。予算はルートのコンパートメントにあるので、コンソールで見るときはルートを選ぶ

```bash
BUDGET=$(oci budgets budget budget list --compartment-id "$TENANCY" --query 'data[0].id' --raw-output)
oci budgets budget alert-rule list --budget-id "$BUDGET" \
  --query 'data[].{name:"display-name",type:type,threshold:threshold,ttype:"threshold-type"}' --output table
oci budgets budget alert-rule create --budget-id "$BUDGET" --type ACTUAL --threshold 100 \
  --threshold-type ABSOLUTE --recipients "<メールアドレス>" --display-name over-100yen
```

- 新しく作るときのコマンドは `oci budgets budget budget create`（`budget` が2回続く）。アラートは `oci budgets budget alert-rule create`。

### 0-5. Ansible と arm64 のエミュレーション

- [x] Ansible を pipx で入れる（Ansible の公式ドキュメントのやり方。apt のものより新しい）

```bash
sudo apt install -y pipx
pipx ensurepath && exec bash -l
pipx install --include-deps ansible
ansible --version
```

- [x] arm64 のイメージをビルドできるように、QEMU のエミュレーションを登録する。WSL を再起動すると消えるので、ビルドの前に毎回確かめる（デプロイのスクリプトにも入れる）

```bash
docker run --privileged --rm tonistiigi/binfmt --install arm64
docker run --rm --platform linux/arm64 alpine uname -m     # aarch64 と出れば OK
```

### 0-6. リポジトリの置き場を作る

- [x] `oci-platform` のフォルダを作って `git init` する。GitHub のリポジトリは、最初のコードができたとき（Step 2）に作る

```bash
mkdir ~/claude-sandbox/oci-platform && cd ~/claude-sandbox/oci-platform && git init -b main
```

- [x] Claude が、`.claude/settings.json` の `additionalDirectories` と CLAUDE.md のディレクトリ構成に `oci-platform` を足す

理解ポイント：

- OCI の API キー認証で、何が手元にあり、何が OCI に登録されているか（秘密鍵は手元だけ。OCI には公開鍵とそのフィンガープリント）。
- `~/.oci/config` の中身は秘密か（OCID とフィンガープリントと鍵の場所だけで、秘密そのものは `.secrets/` の鍵）。
- state のバケットを Terraform で作らないのはなぜか（Cloudflare のときと同じ）。

## Step 1：既存リソースの片付け

2023年に作ったものを、中身を確かめずにバックアップしてから消す。消すのはコンソールでも CLI でもよい（VCN はコンソールの「Delete」が、サブネットやゲートウェイも一度に消してくれて楽）。

- [x] 全コンパートメントの、残っているリソースを一覧にする（読むだけ）

```bash
oci search resource structured-search --query-text \
  "query all resources where lifecycleState != 'TERMINATED' && lifecycleState != 'DELETED'" \
  --query 'data.items[].{type:"resource-type",name:"display-name",state:"lifecycle-state",id:identifier}' \
  --output table --limit 1000
```

- `--all` はない。`--limit` を付けないと1ページで切れることがある。
- `OsmhSoftwareSource`（OS Management Hub が見せている Oracle Linux のパッケージ配布元。自分のリソースではなく、無料）が大量に出るので、`--query` で除く：`--query 'data.items[?"resource-type"!=`"OsmhSoftwareSource"`].{...}'`

- [x] マイクロ2台を止めてから、2台のブートボリュームと 50GB のブロックボリュームのバックアップを取る（止めると、書き込みの途中のない状態で取れる。バックアップは5つまで無料）。ID は上の一覧の BootVolume と Volume の行から。実際は、ブロックボリュームは何にも使っていなかったので、バックアップを取らずに消した

```bash
oci compute instance action --action STOP --instance-id <インスタンスの ID> --wait-for-state STOPPED
oci bv boot-volume-backup create --boot-volume-id <ブートボリュームの ID> --type FULL \
  --display-name <VM 名>-boot-20261104 --wait-for-state AVAILABLE
oci bv backup create --volume-id <ブロックボリュームの ID> --type FULL \
  --display-name <VM 名>-data-20261104 --wait-for-state AVAILABLE
```

- [x] カスタムイメージが要るか確かめる（毎月数円の保管料の元）。要らなければ消す
- [x] マイクロ2台、ブロックボリューム、HeatWave（Always Free は1台なので作り直す）、古い VCN を消す。インスタンスを消しても、付いていたブロックボリュームは残るので、別に消す

```bash
oci compute instance terminate --instance-id <インスタンスの ID> --preserve-boot-volume false --wait-for-state TERMINATED
oci bv volume delete --volume-id <ブロックボリュームの ID> --wait-for-state TERMINATED
oci mysql db-system delete --db-system-id <HeatWave の ID> --wait-for-state DELETED
```

- [x] もう一度一覧を出して、残っているのがバックアップ、コンパートメント、`tfstate` バケットなどだけであることを確かめる（Oracle が作ったもの（`Tenant Admin Policy`、`PSM-*`、`ManagedCompartmentForPaaS`、`Oracle-Tags`、`OsmhProfile`）と予算は残す。古い VCN `vcn-20231009-1034` と DRG が1つずつ残っているが、新しい VCN と合わせて2つで、上限内）
- [ ] 翌日、Cost Analysis でバックアップの分の金額が増えていないかを確かめる（毎月数円のカスタムイメージ代は既にある）

## Step 2：ネットワーク

コードは `~/claude-sandbox/oci-platform/terraform/`（`network.tf` と `nsg.tf`）。中身と理由は [方式検討：OCI の 4](../design/oci.md#4-ネットワーク)。

- [x] VCN、サブネット5つ、Internet GW / NAT GW / Service GW、ルート表、NSG を Terraform で作る（48 リソース）。plan では、作られるもの（VCN 1、サブネット 5、ゲートウェイ 3、ルート表 3、NSG 5、NSG のルール 30前後）を見てから apply する

```bash
cd ~/claude-sandbox/oci-platform/terraform
terraform init        # provider のダウンロードと、backend（tfstate バケット）への接続
terraform plan
terraform apply
```

- [ ] GitHub にリポジトリを作って push する（`.terraform.lock.hcl` も入れる。provider の版を固定するファイル）

```bash
cd ~/claude-sandbox/oci-platform
git add -A && git commit -m "Add network (VCN, subnets, gateways, NSGs)"
gh repo create oci-platform --public --source . --push
```

- [ ] 翌日、Cost Analysis で NAT Gateway の分の金額が増えていないかを確かめる

理解ポイント：

- サブネットのセキュリティ・リストと NSG の違い。
- mail を private ではなく public-dmz に置く理由。

## Step 3：VM と Bastion と Ansible

コードは `oci-platform/terraform/compute.tf`（VM 3台、Bastion、ssh の設定の書き出し）、`oci-platform/bin/bastion-proxy`（Bastion のセッションを用意して ssh をつなぐ）、`oci-platform/ansible/`（VM の中の設定）。

- [x] SSH の鍵を2つ作る。VM へのログイン用（ed25519）と、Bastion のセッション用（RSA。Bastion は RSA しか受け付けず、ed25519 だとセッションは作れるが接続で `Permission denied (publickey)` になる）。パスフレーズを付けるなら ssh-agent に入れておく

```bash
ssh-keygen -t ed25519 -f ~/.ssh/oci-platform -C oci-platform
ssh-keygen -t rsa -b 4096 -f ~/.ssh/oci-platform-bastion -C oci-platform-bastion
```

- [x] ops（マイクロ）、app（A1 1/8GB）、mail（A1 1/4GB）と Bastion を作る（コンソールで「Always Free」のラベルが付くのはマイクロだけ。A1 は使用量のうち一定量までが無料になる仕組みなので付かない。翌日、Cost Analysis で A1 の金額が0円かを確かめる）。A1 で `Out of host capacity` が出たら、時間を置いてもう一度 apply する

```bash
cd ~/claude-sandbox/oci-platform/terraform
terraform init        # local provider が増えたので
terraform plan
terraform apply
```

- [x] Terraform が書き出した `oci-platform/ssh_config` を、`~/.ssh/config` の先頭から読み込む（Include は Host の行より前に置く）

```bash
{ echo 'Include ~/claude-sandbox/oci-platform/ssh_config'; echo; cat ~/.ssh/config 2>/dev/null; } > ~/.ssh/config.new
mv ~/.ssh/config.new ~/.ssh/config && chmod 600 ~/.ssh/config
```

- [x] 3台に ssh で入る。初回はセッションを作るので1分ほど待つ。2回目からは、3時間以内なら待たない

```bash
ssh oci-app
ssh oci-mail
ssh oci-ops
oci bastion session list --bastion-id "$(terraform -chdir=$HOME/claude-sandbox/oci-platform/terraform output -raw bastion_id)" \
  --session-lifecycle-state ACTIVE --query 'data[].{name:"display-name",created:"time-created"}' --output table
```

- [x] Ansible で VM の中の設定を流す（タイムゾーン、セキュリティの自動更新（自動再起動なし・Docker 本体は対象外）、iptables、app と ops に Docker（`live-restore`）、ops に swap、mail に `/srv/mail`）

```bash
cd ~/claude-sandbox/oci-platform/ansible
ansible all -m ping
ansible-playbook site.yml
ansible-playbook site.yml     # 2回目は changed=0 になる（何度流しても同じ状態）
```

- [x] 手で流す更新を一度試す（全部更新し、再起動が要る VM だけ1台ずつ再起動する）

```bash
ansible-playbook upgrade.yml
```

- [x] コンソール接続を確かめる：コンソールで ops を開き、Resources → Console connection → Launch Cloud Shell connection。ログインの画面が出れば OK（ubuntu にはパスワードがないので、実際に入るときは別の手順が要る）

理解ポイント：

- Bastion のセッションは何を許可しているか。期限が切れたら何が起きるか。
- Terraform と Ansible の境目はどこか。

## Step 4：DB と Vault

コードは `oci-platform/terraform/database.tf`（HeatWave）と `vault.tf`（Vault、管理者のパスワード、インスタンス・プリンシパルの権限）、`ansible/roles/ops_tools`（ops に mysql のクライアントと oci CLI）。

- [x] HeatWave（MySQL.Free、private-db、削除保護あり）と Vault を作る。Always Free ではバックアップの設定（backup_policy）を作成時に指定できない（`InvalidParameter`）ので書かず、OCI の既定の自動バックアップにする。中身はコンソールで確かめる（既定は「有効、保存1日」だった）。HeatWave が VNIC を作って NSG に入るには、DB システム自身の権限のポリシー（`platform-mysql`）が要る（ないと FAILED。`database.tf` に入れた）。できた MySQL のバージョンは 26.10.0。管理者のパスワードは Terraform が作って Vault に入れる。HeatWave の作成には 15〜20 分かかる

```bash
cd ~/claude-sandbox/oci-platform/terraform
terraform init        # random provider が増えたので
terraform plan
terraform apply
terraform output mysql
terraform output -raw vault_id
```

- [x] ops に mysql のクライアントと oci CLI を入れる（oci CLI の認証は、鍵を置かずにインスタンス・プリンシパルで行う）

```bash
cd ~/claude-sandbox/oci-platform/ansible
ansible-playbook site.yml
```

- [x] ops から、Vault のパスワードを取り出して HeatWave につなぐ（IAM のポリシーが効くまで1分ほどかかることがある）

```bash
ssh oci-ops
VAULT=<terraform output -raw vault_id の値>
PW=$(oci secrets secret-bundle get-secret-bundle-by-name --auth instance_principal \
  --vault-id "$VAULT" --secret-name mysql-admin-password \
  --query 'data."secret-bundle-content".content' --raw-output | base64 -d)
mysql -h mysql.db.platform.oraclevcn.com -u admin -p"$PW" -e 'SELECT VERSION();'
```

理解ポイント：

- インスタンス・プリンシパルでは、何が「鍵」の代わりになっているか（動的グループに入っている VM であること）。
- パスワードは、どこに・どの形で残っているか（Vault、Terraform の state）。

## Step 5：LB と公開

コードは `oci-platform/terraform/lb.tf`（LB、Origin 証明書の読み込み、既定のバックエンド・セットとリスナー）、`terraform/cloudflare.tf`（確認用の DNS レコード。Cloudflare を触るのはこのファイルだけ）、`ansible/roles/default_site`（app の 8000 番に、どのアプリにも当たらないホスト名のための小さなページ）。Cloudflare のゾーンとアカウントの設定は手で行い、Terraform は DNS レコードだけを持つ。この Step から、terraform は `./tf.sh` から実行する（Cloudflare の API トークンを読み込むため）。

- [x] Always Use HTTPS を有効にする（SSL/TLS → エッジ証明書）。LB は 443番しか受けないので、http はエッジで https に転送させる
- [x] 暗号化モードを **Full (strict)** にする（SSL/TLS → 概要）。今は Full。DNS の一覧でプロキシしているのは Workers と Pages だけで、どちらも影響しないことは確認済み
- [x] Origin 証明書を発行する（SSL/TLS → オリジンサーバー → 証明書を作成）。「Cloudflare で秘密キーと CSR を生成する」、キーの種類は RSA (2048)、ホスト名は `*.alphajinsei.com` と `alphajinsei.com`（既定のまま）、有効期間は15年、形式は PEM。**秘密鍵はこの画面でしか表示されない**ので、閉じる前に `.secrets` に置く（Claude は読まない）

```bash
cat > ~/claude-sandbox/infra/.secrets/oci-platform-origin.pem   # 「オリジン証明書」を貼り付けて Ctrl+D
cat > ~/claude-sandbox/infra/.secrets/oci-platform-origin.key   # 「秘密キー」を貼り付けて Ctrl+D
chmod 600 ~/claude-sandbox/infra/.secrets/oci-platform-origin.*
openssl x509 -in ~/claude-sandbox/infra/.secrets/oci-platform-origin.pem -noout -subject -issuer -enddate   # 発行者が Cloudflare、期限が15年後
```

- [x] Terraform 用の Cloudflare の API トークンを作る（My Profile → API Tokens → Create Token → Custom token）。名前は `terraform-oci-dns`、権限は「Zone / Zone / Read」「Zone / DNS / Edit」、対象のゾーンは alphajinsei.com だけ。共通基盤とアプリの Terraform で共用する

```bash
cat > ~/claude-sandbox/infra/.secrets/terraform-oci-dns.env   # 貼り付けて Ctrl+D
# 中身は1行：CLOUDFLARE_API_TOKEN=<トークン>
chmod 600 ~/claude-sandbox/infra/.secrets/terraform-oci-dns.env
```

- [x] app に既定のページ（nginx、8000番）を置き、VM の中で見えることを確かめる

```bash
cd ~/claude-sandbox/oci-platform/ansible
ansible-playbook site.yml
ssh oci-app curl -s localhost:8000
```

- [x] LB・証明書・確認用の DNS レコード（`oci-platform.alphajinsei.com`）を作る

```bash
cd ~/claude-sandbox/oci-platform/terraform
./tf.sh init          # cloudflare の provider が増えたので
./tf.sh plan
./tf.sh apply         # LB の作成に数分かかる
```

- [x] 外から見えることと、LB を直接たたけないことを確かめる。コンソールの LB の画面で、バックエンド・セット `default` のヘルスが OK になっていることも見る

```bash
curl -s https://oci-platform.alphajinsei.com            # 既定のページが返る
curl -sI http://oci-platform.alphajinsei.com | head -3  # 301 で https に転送される
LB_IP=$(./tf.sh output -json lb | jq -r .public_ip)
curl -sk --max-time 5 https://$LB_IP || echo "blocked"  # Cloudflare 以外からは NSG で落とされる
```

理解ポイント：

- Cloudflare → LB → app の各区間で、TLS はどこで終わり、どこからまた始まるか。どの区間で、どの証明書を使っているか。
- LB の IP を直接たたくと、なぜ届かないか。

## Step 6：サンプルアプリ

コードは `~/claude-sandbox/sample-oci-vm-docker`（②のメモ帳の API を、Node + MySQL に載せ替えたもの。中身と手順は、その README）。共通基盤の側は、app にも oci CLI と `platform-secret`（Vault からシークレットを取り出す）を入れ、Terraform の出力に `vault_key_id` を足した。イメージは運ばず、ソースを送って app の上でビルドする（[方式検討 8](../design/oci.md#8-デプロイcicd)）。

- [x] 共通基盤の変更を流す（app に oci CLI と `platform-secret`、`/opt/apps`。Terraform は出力が1つ増えるだけ）

```bash
cd ~/claude-sandbox/oci-platform/terraform && ./tf.sh apply
cd ../ansible && ansible-playbook site.yml
ssh oci-app platform-secret mysql-admin-password | wc -c   # 文字数だけ出す（値は表示しない）。24 なら読めている
```

- [x] GitHub にリポジトリを作る（Template repository にする）

```bash
cd ~/claude-sandbox/sample-oci-vm-docker
gh repo create sample-oci-vm-docker --public --source . --push
gh repo edit --template
```

- [x] 手元で動かす

```bash
npm install
docker compose up -d mysql
npm run db:migrate:local
npm run dev                                    # 別のターミナルで curl（README の 3）
docker compose up -d --build                   # 本番と同じコンテナでも確かめる（:8001）
docker compose run --rm app node dist/migrate.js
curl -s localhost:8001/api/memos
docker compose down
```

- [x] アプリの分を共通基盤に足す（LB のホスト名・リスナー・バックエンド・セット、DB のパスワード、DNS レコード）。LB に同じ 443 番のリスナーを、ホスト名を付けて並べられるかを、ここで確かめる

```bash
cd infra/terraform
./tf.sh init
./tf.sh plan
./tf.sh apply
cd ../..
```

- [x] スキーマとユーザーを作り（初回だけ）、デプロイする

```bash
npm run db:init       # ops から HeatWave に
npm run deploy        # 初回はベースイメージの取得と npm ci で数分かかる
curl -s https://sample-oci-vm-docker.alphajinsei.com/api/health
curl -s -X POST https://sample-oci-vm-docker.alphajinsei.com/api/memos -H "Content-Type: application/json" -d '{"text":"OCI から"}'
curl -s https://sample-oci-vm-docker.alphajinsei.com/api/memos
curl -s https://oci-platform.alphajinsei.com       # 共通基盤の既定のページも、引き続き返る
```

- [x] コンソールの LB の画面で、バックエンド・セット `sample-oci-vm-docker` のヘルスが OK になっていることを見る

理解ポイント：

- ②の `npm run deploy` と比べて、何が増えたか（ソースの転送、VM の上でのビルド、マイグレーションを流す場所、パスワードの受け渡し）。
- DB のパスワードは、どこで作られ、どこを通って、どこに残るか。

## Step 7：運用

アラームは「止まった」ことの2つだけ（アプリと VM）。ログは LB のものだけ OCI Logging に入れ、コンテナのログは `docker compose logs` で見る（[方式検討の 11](../design/oci.md#11-運用)）。OS の自動更新は Step 3 で入れ済み。

- [x] 通知のトピックとメールの購読、アラーム2つ、LB のログを作る（`terraform/monitoring.tf`）

  ```bash
  cd ~/claude-sandbox/oci-platform/terraform
  ./tf.sh apply
  ```

- [x] 届いた確認メール（件名に「OCI Notifications Service Subscription Confirmation」）のリンクを押す。押すまで通知は届かない
- [x] app のコンテナを止めて、3〜4分でアラームのメールが来ることを確かめる。戻して、OK のメールが来ることも確かめる

  ```bash
  ssh oci-app 'cd /opt/apps/sample-oci-vm-docker && docker compose stop app'
  # メールが来たら
  ssh oci-app 'cd /opt/apps/sample-oci-vm-docker && docker compose start app'
  ```

- [x] LB のログを見る：コンソールの「監視およびマネジメント → ロギング → ログ」で `lb-access` を開き、さっきの 502 と、その後の 200 が残っていることを確かめる
- [x] コンテナのログを見る：`ssh oci-app 'cd /opt/apps/sample-oci-vm-docker && docker compose logs --tail 50 app'`

VM の停止のアラームは、VM を止めて確かめることもできる（ops を止めると10分ほどで来る）。止めるのは任意。

## Step 8：メールサーバー

`alphajinsei@alphajinsei.com` で送受信できるようにし、アプリからも送れるようにする。方式は部品を自分で組む形（Postfix・Dovecot・Rspamd、送信は Email Delivery に中継）。決めたことは [方式検討：OCI のメール基盤](../design/oci-mail.md)、基礎は [番外編：メールの仕組み](../notes/extra-mail-basics.md)、組みながら分かったことは [番外編：メールサーバーを組む](../notes/extra-mail-build.md)。大きいので6つに分け、各回のコマンドはその回に入るときに書き足す。

### 8-1. 基礎

- [x] [番外編：メールの仕組み](../notes/extra-mail-basics.md) を読む
- [x] DNS を引いてみる：MX（配送先）、SPF、DMARC がどう書かれているか

  ```bash
  dig +short MX gmail.com
  dig +short TXT gmail.com | grep spf
  dig +short TXT _dmarc.gmail.com
  dig +short MX alphajinsei.com       # まだ何も返らない
  ```

- [x] Gmail で受け取ったメール（会社や通販などのもの）の「メッセージのソースを表示」を開き、3つの差出人（`Return-Path:`、`DKIM-Signature:` の `d=`、`From:`）と `Authentication-Results:` を見比べる。どれが一致して DMARC が通っているか

理解ポイント：

- サーバー同士の 25番と、メールソフトからの 465・587番の違い。
- SPF・DKIM・DMARC が、それぞれどの差出人を確かめているか。DMARC のアライメントとは何か。

### 8-2. 受信

Postfix で `alphajinsei.com` 宛てだけを受け、`/srv/mail/alphajinsei.com/alphajinsei/Maildir/` に1通1ファイルで保存する。メールのユーザーは Linux のユーザーにしない（virtual。持ち主は `vmail` の1人）。TLS は仮に自己署名（snakeoil）で、8-3 で Let's Encrypt に替える。外へはまだ送れない。

- [x] DNS（`mail.alphajinsei.com` の A と、`alphajinsei.com` の MX）を置く。どちらもプロキシしない

  ```bash
  cd ~/claude-sandbox/oci-platform/terraform
  ./tf.sh apply
  dig +short MX alphajinsei.com
  dig +short A mail.alphajinsei.com
  ```

- [x] Postfix を入れる（mail に `roles/mail`、ops に swaks）

  ```bash
  cd ~/claude-sandbox/oci-platform/ansible
  ansible-playbook site.yml --limit mail,ops
  ssh oci-mail 'sudo postconf -n'      # 効いている設定（既定から変えたものだけ）
  ```

- [x] ops から mail に、手で SMTP を話す（swaks は会話を全部表示する。`<~` が相手、`~>` が自分）。ops は Postfix から見ると「よその人」

  ```bash
  ssh oci-ops
  swaks --server 10.0.1.10 --to alphajinsei@alphajinsei.com --from test@example.com   # 250：受け取る
  swaks --server 10.0.1.10 --to nobody@alphajinsei.com --from test@example.com        # 550：知らない宛先は拒否
  swaks --server 10.0.1.10 --to someone@gmail.com --from test@example.com             # 454/554 Relay access denied：よそ宛ては中継しない
  ```

- [x] Gmail から `alphajinsei@alphajinsei.com` に送り、mail VM の上で読む

  ```bash
  ssh oci-mail
  sudo tail -f /var/log/mail.log                                   # 受け取りのログ（from=、to=、status=sent）
  sudo ls /srv/mail/alphajinsei.com/alphajinsei/Maildir/new/       # 1通1ファイル
  sudo cat /srv/mail/alphajinsei.com/alphajinsei/Maildir/new/<ファイル名>
  ```

  ヘッダーの `Received:`（どのサーバーを通ったか。上が新しい）と、Gmail が付けた `DKIM-Signature:` を見る。

理解ポイント：

- `mydestination`（Linux のユーザー宛て）と `virtual_mailbox_domains`（メールだけのユーザー宛て）の違い。
- オープンリレーにしない仕組み（`smtpd_relay_restrictions`）。ops からの 3つ目が拒否される理由。
- MX と A の関係。Gmail は `alphajinsei.com` の MX を引き、その名前の A を引いて、25番につなぐ。

### 8-3. 閲覧

Let's Encrypt の証明書を ACME の HTTP-01 で取り（自動更新）、Dovecot で IMAP（993番）を開ける。保存は Postfix から Dovecot に LMTP で渡す形に替える。パスワードは Terraform が作って Vault に入れ、mail VM が読んでハッシュにする（DB のパスワードと同じ形）。

- [x] 80番（HTTP-01）を NSG で開け、メールのパスワードを Vault に作る

  ```bash
  cd ~/claude-sandbox/oci-platform/terraform
  ./tf.sh apply
  ```

- [x] mail に oci CLI、certbot、Dovecot を入れ、Postfix の証明書と保存先を替える

  ```bash
  cd ../ansible
  ansible-playbook site.yml --limit mail
  ```

- [x] 証明書と自動更新を確かめる

  ```bash
  ssh oci-mail 'sudo certbot certificates'                 # 期限（90日）と鍵の種類
  ssh oci-mail 'systemctl list-timers certbot.timer'       # 更新のタイマー
  ssh oci-mail 'sudo certbot renew --dry-run'              # 更新を本番に影響なく試す（Let's Encrypt の試験用の環境）
  openssl s_client -connect mail.alphajinsei.com:993 -servername mail.alphajinsei.com </dev/null 2>/dev/null | head -12
  #   手元から。issuer が Let's Encrypt、Verify return code: 0 (ok) なら、メールソフトも警告なしでつながる
  ```

- [x] パスワードを手元で一度だけ取り出し、パスワード管理のツールに入れる（画面に表示されるので、人に見られない場所で）

  ```bash
  oci secrets secret-bundle get-secret-bundle-by-name \
    --vault-id "$(terraform -chdir=../terraform output -raw vault_id)" \
    --secret-name mail-alphajinsei-password \
    --query 'data."secret-bundle-content".content' --raw-output | base64 -d; echo
  ```

- [x] Thunderbird とスマホでつなぎ、8-2 のメールが見えることを確かめる。設定は手動で：
  - 受信（IMAP）：`mail.alphajinsei.com`、993、SSL/TLS、通常のパスワード、ユーザー名 `alphajinsei@alphajinsei.com`
  - 送信（SMTP）：`mail.alphajinsei.com`、465、SSL/TLS（8-4 までは送れない。Thunderbird で確認が通らなければ、受信だけ先に作る）
- [x] Gmail からもう1通送り、メールソフトに届くことを確かめる。`/var/log/mail.log` で、受け渡しが `relay=...[private/dovecot-lmtp]`（Postfix → Dovecot）になっていることを見る

理解ポイント：

- 証明書は「つないだ相手が本物の `mail.alphajinsei.com` か」をメールソフトが確かめるためのもの。確かめられないと、パスワードを偽物に渡しうる。
- ACME の HTTP-01 の流れ（certbot が 80番で答え、Let's Encrypt が外から確かめに来る）と、更新が自動で回る仕組み（タイマーと deploy フック）。
- Postfix（受け取り）と Dovecot（保存と閲覧）の役割の分担。

### 8-4. 送信

メールソフトから 465・587番でログインして送り、Postfix が全部を Email Delivery に中継する。Email Delivery が DKIM（`d=alphajinsei.com`）で署名し、封筒の差出人を `bounce.alphajinsei.com`（カスタム Return-Path）にして送るので、SPF・DKIM の両方で DMARC のアライメントが取れる。Email Delivery の設定（ドメイン、DKIM、Return-Path、承認済みの差出人、中継用の IAM のユーザーと SMTP の資格情報）と DNS（SPF、DKIM、Return-Path、DMARC）は Terraform で作る。

- [x] Email Delivery と DNS を作る。1回目の apply は Return-Path で失敗する（DKIM が Active でないと作れない）ので、DKIM が Active になってからもう一度 apply する

  ```bash
  cd ~/claude-sandbox/oci-platform/terraform
  ./tf.sh apply          # 1回目：Return-Path だけ失敗する
  # コンソールで DKIM が Active になるのを待つ（下の項目）
  ./tf.sh apply          # 2回目：Return-Path と、その CNAME・SMTP の資格情報ができる
  terraform output mail_relay
  ```

- [x] コンソールの「開発者サービス → アプリケーション統合 → 電子メール配信 → 電子メール・ドメイン → alphajinsei.com」で、DKIM と Return-Path が「Active」（DNS が確かめられた）になり、承認済みの差出人に SPF の印が付くのを待つ（数分〜数十分）

  ```bash
  dig +short TXT alphajinsei.com
  dig +short TXT _dmarc.alphajinsei.com
  dig +short CNAME oci-nrt-20261105._domainkey.alphajinsei.com
  dig +short CNAME bounce.alphajinsei.com
  ```

- [x] Postfix に送信の受付（465・587番）と中継を足す

  ```bash
  cd ../ansible
  ansible-playbook site.yml --limit mail
  ssh oci-mail 'sudo postconf -M | head -3; sudo postconf relayhost'
  ```

- [x] ops から、ログインしないと送れないことを確かめる

  ```bash
  ssh oci-ops
  swaks --server 10.0.1.10 --port 587 --tls --to alphajinsei@gmail.com --from alphajinsei@alphajinsei.com
  #   ログインなし：554 Relay access denied などで拒否される
  ```

- [x] Outlook に送信（SMTP）を設定する：`mail.alphajinsei.com`、465、SSL/TLS、ユーザー名 `alphajinsei@alphajinsei.com`、パスワードは受信と同じ（Outlook は送信のポートが既定で 25 になる。25 のままだと `503 5.5.1 Error: authentication not enabled` で送れない）
- [x] `alphajinsei@gmail.com` に送る。mail VM の `/var/log/mail.log` で、`relay=smtp.email.ap-tokyo-1.oci.oraclecloud.com[...]:587 ... status=sent (250 ...)` を見る
- [x] Gmail で「メッセージのソースを表示」を開き、`SPF: PASS`、`DKIM: PASS`、`DMARC: PASS` と、3つの差出人（`Return-Path:` が `bounce.alphajinsei.com`、`DKIM-Signature:` の `d=alphajinsei.com`、`From:`）を確かめる
- [x] Gmail から返信し、Outlook に届くことを確かめる

理解ポイント：

- 封筒の差出人を誰が決めているか（Outlook → Postfix では `alphajinsei@alphajinsei.com`、Email Delivery が `bounce.alphajinsei.com` に書き換える）。SPF と DKIM が、それぞれどのドメインで通ったか。
- 中継先へのログイン（Postfix がクライアント）と、メールソフトからのログイン（Postfix がサーバー、確認は Dovecot）の2つの SASL。
- オープンリレーにしないための、465・587番の `permit_sasl_authenticated,reject`。

### 8-5. 守り

受け取ったメールを Rspamd に点数付けさせ（6点以上は Junk へ、15点以上は拒否）、ログインの失敗が続く IP を fail2ban で締め出し、`/srv/mail` を週に1回 Object Storage に退避する（2世代）。最後に、DMARC のレポートを見て `p=quarantine` に上げる。

- [x] バックアップの置き場（バケット `mail-backup`、14日で消すライフサイクル、mail VM だけが置ける・読めるポリシー）を作る

  ```bash
  cd ~/claude-sandbox/oci-platform/terraform
  ./tf.sh apply
  ```

- [x] Rspamd・Sieve・fail2ban・バックアップのタイマーを入れる

  ```bash
  cd ../ansible
  ansible-playbook site.yml --limit mail
  ssh oci-mail 'sudo rspamadm configtest; sudo postconf smtpd_milters; sudo doveconf -n | grep sieve; sudo fail2ban-client status; systemctl list-timers mail-backup.timer'
  ```

- [x] Rspamd：Gmail から `alphajinsei@alphajinsei.com` に送り、Outlook で「ソースを表示」して、`X-Spamd-Result`（点数と当たった規則）、`X-Spam-Status: No`、Rspamd の `Authentication-Results`（`spf=pass`、`dkim=pass`、`dmarc=pass`）を見る
- [x] Rspamd：ops から、迷惑メールの試験用の文字列（GTUBE）を含むメールが拒否されることを確かめる

  ```bash
  ssh oci-ops
  swaks --server 10.0.1.10 --to alphajinsei@alphajinsei.com --from test@example.com \
    --body 'XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X'
  #   554 5.7.1 Gtube pattern などで拒否される
  ```

- [x] Rspamd の画面を SSH のトンネルで開き、「History」で今のメールの点数と規則を見る（手元の 11334番を mail VM の 11334番につなぐ）

  ```bash
  ssh -N -L 11334:localhost:11334 oci-mail
  #   ブラウザで http://localhost:11334（mail VM の中からの接続になるので、パスワードは要らない）。Ctrl+C で閉じる
  ```

- [x] fail2ban：ops から間違ったパスワードでログインし、そのログの行を fail2ban の規則が拾うことを確かめる（ops は締め出さない設定なので、数は増えない。数えるのは外からの失敗だけ）

  ```bash
  ssh oci-ops
  swaks --server 10.0.1.10 --port 587 --tls --auth LOGIN \
    --auth-user alphajinsei@alphajinsei.com --auth-password wrong \
    --to alphajinsei@gmail.com --from alphajinsei@alphajinsei.com
  #   535 5.7.8 Error: authentication failed
  exit
  ssh oci-mail 'sudo fail2ban-regex /var/log/mail.log "postfix[mode=auth]" | grep -A2 "^Failregex"'
  ```

- [ ] 1日ほどたってから、外からの失敗で締め出されている IP を見る（後日）

  ```bash
  ssh oci-mail 'sudo fail2ban-client status postfix-sasl; sudo fail2ban-client status dovecot'
  ```

- [x] バックアップ：タイマーを待たずに1回動かし、バケットに置かれたことを確かめる

  ```bash
  ssh oci-mail 'sudo systemctl start mail-backup.service; sudo journalctl -u mail-backup.service -n 5 --no-pager'
  oci os object list --bucket-name mail-backup --query 'data[].{name:name,size:size}' --output table
  ```

- [x] 戻せることを確かめる（mail VM の上で、/tmp に取り出して中を見るだけ。`/srv/mail` は触らない）

  ```bash
  ssh oci-mail
  oci os object get --auth instance_principal --bucket-name mail-backup \
    --name <上の一覧の名前> --file /tmp/restore.tar.gz
  tar -tzf /tmp/restore.tar.gz | head
  rm /tmp/restore.tar.gz
  #   消せないことも確かめる：oci os object delete --auth instance_principal --bucket-name mail-backup --name <名前> → 404 NotAuthorizedOrNotFound
  ```

- [ ] DMARC のレポート（後日。最初の1通は 2026-11-07 に読んだ。全部 pass）：1週間ほど、`dmarc@` に届くレポート（Google などから、zip か gz の XML）を開き、自分のメールが全部 `dkim` と `spf` で pass、`disposition` が `none` になっていることを見る
- [ ] 全部 pass なら、`cloudflare.tf` の DMARC を `p=quarantine` に変えて apply する（変更は Claude に頼む）（後日）

理解ポイント：

- milter：Postfix が受け取りの途中で、メールを Rspamd に見せて判定をもらう仕組み。Rspamd が止まっていたら、判定なしで受け取る（`milter_default_action = accept`）。
- 点数と扱い（拒否・ヘッダーの追加）は Rspamd、Junk への振り分けは Dovecot の Sieve。役割が分かれている。
- fail2ban は、ログを読んで iptables に締め出しのルールを足すだけ。common の iptables の読み直しで消えるので、再起動して足し直させる。
- バックアップの権限は「置く・読む」だけ。mail VM が乗っ取られても、過去のバックアップは消されない。

### 8-6. アプリから

アプリのコンテナは、mail VM の 25番にログインなしで送る（Postfix の `mynetworks` に app VM のサブネットを入れる。自分のシステムの中は信じる）。差出人は全アプリ共通の `noreply@alphajinsei.com` で、表示名でアプリを区別する。共通基盤は、送り先を出力 `smtp` で渡す。

- [x] 認証の方式を決める：送信元の IP で信頼する（アプリごとのパスワードは、仕組みに見合わない）
- [x] 共通基盤：承認済みの差出人 `noreply@` と出力 `smtp` を作り、Postfix に app VM のサブネットを信頼させる

  ```bash
  cd ~/claude-sandbox/oci-platform/terraform
  ./tf.sh apply
  terraform output smtp
  cd ../ansible
  ansible-playbook site.yml --limit mail
  ssh oci-mail 'sudo postconf mynetworks'
  ```

- [x] サンプル：nodemailer を入れ、手元の Mailpit（外へ出さないメールの受け口）で送れることを確かめる

  ```bash
  cd ~/claude-sandbox/sample-oci-vm-docker
  npm install nodemailer
  npm install -D @types/nodemailer
  docker compose up -d mailpit
  npx tsx --env-file=local.env src/send-test-mail.ts you@example.com
  #   http://localhost:8025 に届いている
  ```

- [x] サンプル：出力 `smtp` を足してデプロイし、app VM のコンテナから Gmail に送る

  ```bash
  cd infra/terraform && ./tf.sh apply && cd ../..   # 出力 smtp が増えるだけ
  npm run deploy
  ssh oci-app 'cd /opt/apps/sample-oci-vm-docker && docker compose exec -T app node dist/send-test-mail.js alphajinsei@gmail.com'
  #   sent: <...> 250 2.0.0 Ok: queued as ...
  ssh oci-mail 'sudo grep -E "client=.*10\.0\.10\.|relay=smtp.email" /var/log/mail.log | tail -4'
  ```

- [x] Gmail で、差出人が「sample-oci-vm-docker <noreply@alphajinsei.com>」で、SPF・DKIM・DMARC が pass になっていることを見る

理解ポイント：

- `mynetworks`（信頼するネットワーク）からは、25番でもよそのドメイン宛てを中継する（`permit_mynetworks`）。外から 25番に来たものは、今までどおり自分のドメイン宛てだけ。
- 外へ出られる差出人は、Email Delivery の承認済みの差出人（`alphajinsei@` と `noreply@`）で縛られる。Postfix では縛らない。
- VCN の中の名前（`mail.dmz.platform.oraclevcn.com`）でつなぎ、証明書は `mail.alphajinsei.com` を指定して確かめる。公開の名前でつなぐと公開 IP 宛てになり、送信元が NAT の IP になって信頼されない。
- 手元の Mailpit で確かめられるのはアプリのコードまで（設定・差出人・ビルド）。経路・中継・TLS・SPF などは本番で確かめる。

## Step 9：仕上げ

- [x] 共通基盤の README に、仕様（アプリが受け取るもの・守ること）とポートの台帳を書く
- [x] サンプルの README を、[サービス開発ガイド](../service-guide.md) の6つの見出しで書く（Step 6〜8 で書いてきたものを見直した）
- [x] サービス開発ガイドの「方式の一覧」の③の行を埋める
- [x] サービス開発ガイドを3段に分ける：全方式共通（`service-guide.md`）、基盤ごと（Cloudflare は `docs/platforms/cloudflare.md`、OCI は `oci-platform` の README）、方式ごと（サンプルの README）。ガイドが Cloudflare の話に寄っていたため
