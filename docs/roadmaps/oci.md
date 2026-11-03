# 学習ロードマップ：OCI 共通基盤（方式③）

最終更新：2026-11-03

## ゴール

[方式検討：OCI](../design/oci.md) の構成を、Terraform と Ansible で作って動かす。その上にサンプルアプリを載せ、③のサービスを作るときに読む資料（共通基盤とサンプルの README）までそろえる。

- 共通基盤：`oci-platform`
- サンプル：`sample-oci-vm-docker`。②と同じメモ帳を Node + Hono + Drizzle（MySQL）で作り、添付ファイルを Object Storage に置く
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

- [ ] 全コンパートメントの、残っているリソースを一覧にする（読むだけ）

```bash
oci search resource structured-search --query-text \
  "query all resources where lifecycleState != 'TERMINATED' && lifecycleState != 'DELETED'" \
  --query 'data.items[].{type:"resource-type",name:"display-name",state:"lifecycle-state",id:identifier}' \
  --output table --limit 1000
```

- `--all` はない。`--limit` を付けないと1ページで切れることがある。
- `OsmhSoftwareSource`（OS Management Hub が見せている Oracle Linux のパッケージ配布元。自分のリソースではなく、無料）が大量に出るので、`--query` で除く：`--query 'data.items[?"resource-type"!=`"OsmhSoftwareSource"`].{...}'`

- [ ] マイクロ2台を止めてから、2台のブートボリュームと 50GB のブロックボリュームのバックアップを取る（止めると、書き込みの途中のない状態で取れる。バックアップは5つまで無料）。ID は上の一覧の BootVolume と Volume の行から

```bash
oci compute instance action --action STOP --instance-id <インスタンスの ID> --wait-for-state STOPPED
oci bv boot-volume-backup create --boot-volume-id <ブートボリュームの ID> --type FULL \
  --display-name <VM 名>-boot-20261104 --wait-for-state AVAILABLE
oci bv backup create --volume-id <ブロックボリュームの ID> --type FULL \
  --display-name <VM 名>-data-20261104 --wait-for-state AVAILABLE
```

- [ ] カスタムイメージが要るか確かめる（毎月数円の保管料の元）。要らなければ消す
- [ ] マイクロ2台、ブロックボリューム、HeatWave（Always Free は1台なので作り直す）、古い VCN を消す。インスタンスを消しても、付いていたブロックボリュームは残るので、別に消す

```bash
oci compute instance terminate --instance-id <インスタンスの ID> --preserve-boot-volume false --wait-for-state TERMINATED
oci bv volume delete --volume-id <ブロックボリュームの ID> --wait-for-state TERMINATED
oci mysql db-system delete --db-system-id <HeatWave の ID> --wait-for-state DELETED
```

- [ ] もう一度一覧を出して、残っているのがバックアップ、コンパートメント、`tfstate` バケットなどだけであることを確かめる
- [ ] 翌日、Cost Analysis でバックアップの分の金額が増えていないかを確かめる（毎月数円のカスタムイメージ代は既にある）

## Step 2：ネットワーク

- [ ] VCN、サブネット5つ、Internet GW / NAT GW / Service GW、ルート表、NSG を Terraform で作る
- [ ] 翌日、Cost Analysis で NAT Gateway の分の金額が増えていないかを確かめる

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
- [ ] 翌日、Cost Analysis で OCIR の分の金額が増えていないかを確かめる

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
