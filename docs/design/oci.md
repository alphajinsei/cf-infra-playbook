# 個人サービス公開インフラ 方式検討：OCI（方式③）

最終更新：2026-11-03

OCI の Always Free の範囲で、LB・サーバー・DB のそろった「いわゆる3層」の共通基盤を作り、その上にコンテナでアプリを載せる。メールサーバーのように VM 単体で動かしたいものも、同じ VCN に置く。要件と方式の一覧は [全体](overview.md)。

## 1. 位置づけ

- Workers（方式②）で動かないものの受け皿：常駐プロセス、WebSocket の常時接続、任意の言語、コンテナで配られている OSS、自前のメールサーバー。
- 方式②と違い、インフラ層を自分で持つ。そのインフラ層は **共通基盤（platform）** として1つだけ作り、アプリはその上に差分（コンテナ、LB の振り分け1つ、DB のスキーマ1つ）だけを足す。
- 共通基盤は、中身を知らなくても使えるように仕様（[9. 共通基盤とアプリの境界](#9-共通基盤とアプリの境界)）を外に見せる。アプリを作るときは、その仕様だけを読めばよい。
- 共通基盤は、自分で管理し続ける基盤でもある。OS の更新、脆弱性への対応、TLS や暗号の移行（PQC など）を、ここで回す。

## 2. 全体の構成

```
Users
  |  https://<app>.alphajinsei.com            mail: MX -> mail VM (25/465/587/993)
  v                                              |
Cloudflare (DNS / proxy / WAF)                   |
  |  HTTPS (Cloudflare Origin CA cert)           |
+-|----------------- VCN 10.0.0.0/16 ------------|---------------------------+
| |  [Internet GW]                               |                           |
| v                                              v                           |
| public-lb  10.0.0.0/24                   public-dmz  10.0.1.0/24           |
|   Flexible LB (10Mbps)                     mail (A1 1 OCPU / 4GB)          |
|   host-based routing, 1 backend set/app      + block volume 30GB           |
|     |  TCP 8000-8099                         outbound -> Email Delivery      |
|     v                                        (relay on 587)                |
| private-app  10.0.10.0/24                private-ops  10.0.30.0/24         |
|   app (A1 1 OCPU / 8GB)                    ops (E2.1.Micro)                |
|   Docker host, 1 container/app             workbench + batch (cron)        |
|     |  TCP 3306                              |  TCP 3306                   |
|     v                                        v                             |
| private-db  10.0.20.0/24                                                   |
|   MySQL HeatWave (50GB, automatic backups)                                 |
|                                                                            |
| [NAT GW]      private-app / private-ops -> internet (apt, docker, APIs)    |
| [Service GW]  -> OCIR, Object Storage, Vault (no internet)                 |
| [Bastion]     -> SSH 22 of app / ops / mail (sessions up to 3h)            |
+----------------------------------------------------------------------------+
  Object Storage: Terraform state, per-app buckets, backups
  Vault: DB passwords, registry token     Monitoring + Notifications: alarms -> email
  Budget: alert when cost > 0

Dev PC (WSL) -- Terraform ----------------------> OCI resources, Cloudflare DNS
Dev PC (WSL) -- Ansible (ssh via Bastion) ------> inside of VMs
Dev PC (WSL) -- npm run deploy -----------------> build arm64 image -> OCIR -> app VM
```

| 層 | 使うもの | 役割 |
| --- | --- | --- |
| 入口 | Cloudflare（DNS・プロキシ・WAF） | ①②と同じ `<名前>.alphajinsei.com`。Cloudflare から LB へは Origin 証明書（`*.alphajinsei.com`、最長15年）で HTTPS |
| 振り分け | OCI Flexible Load Balancer（L7、10Mbps、無料で1台） | TLS の終端、ホスト名でアプリごとのバックエンド・セットへ、アプリごとのヘルスチェック |
| アプリ | app VM（A1）+ Docker | アプリごとにコンテナ1つ。再起動は `restart: unless-stopped` |
| DB | MySQL HeatWave | VCN の中の非公開 IP。アプリごとにスキーマとユーザー |
| VM 単体のサービス | mail VM（A1） | 自前のメールサーバー。今後の「VM 単体で建てたいもの」もここ |
| 作業・バッチ | ops VM（マイクロ） | 勉強用に自由に使う。cron や systemd タイマーで軽い定時処理。DB の管理作業 |
| 運用 | Bastion、Vault、Monitoring、Notifications、Logging、Budget | SSH の入口、秘密情報、監視と通知、課金の見張り |

冗長化はしない。Docker ホストは1台で、単一障害点になることを受け入れる。そのかわり、Terraform + Ansible で「作り直せば元に戻る」状態を保つ。LB は残す（TLS の終端・振り分け・ヘルスチェックを LB が持つので、A1 の割り振りを変えて Docker ホストを2台に戻すときも、バックエンドを足すだけで済む）。

## 3. Always Free の枠と割り振り

2026-11-03 に [公式ドキュメント](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm) で確認した値。Always Free のリソースはホームリージョンに作る。アカウントは有料（Pay As You Go）に移行済みなので、使用率が低い VM の回収の対象にならない。そのかわり、枠を超えると課金されるので、予算アラートを必ず置く。

### コンピュート

| 枠 | 上限 | 割り振り |
| --- | --- | --- |
| A1（Arm、VM.Standard.A1.Flex） | 計2 OCPU / 12GB（以前は 4 / 24 だった） | app：1 OCPU / 8GB、mail：1 OCPU / 4GB |
| E2.1.Micro（AMD） | 2台。1台あたり 1/8 OCPU、1GB | ops：1台。もう1台は使わない |

- A1 とマイクロは別の枠で、同時に使える。
- Docker ホストに多めに配る。Keycloak や自作の認証基盤（Java などはメモリを 1〜2GB 使う）が載るため。メールは Postfix + Dovecot + Rspamd なら 4GB で足り、ClamAV を足しても収まる。
- A1 は東京などで「Out of host capacity」（空きがない）が出やすい。時間を置くか、別の AD で試す。

### ディスク（ブートボリュームとブロックボリュームの合計で 200GB、ブロックボリュームは2つまで）

| VM | ブート | ブロック | 計 |
| --- | --- | --- | --- |
| ops | 47 | | 47 |
| app | 47 | | 47 |
| mail | 47 | 30（メールボックス） | 77 |
| 合計 | | | **171GB（29GB 余る）** |

- ブートボリュームは 47GB より小さくできない。そのため、VM 4台に別のボリュームを足すと 200GB を超える。マイクロ2台に A1 2台（4台）は入るが、余りがなくなる。そこでマイクロは1台に減らした。
- コンテナイメージは1つ数百MB で、`docker image prune` で古いものを消せば、ブートボリュームで足りる。足りなくなったら、ブートボリュームを止めずに大きくできる。
- 別のブロックボリュームは「VM を作り直しても残したいデータ」にだけ使う（メールボックス）。app のデータは HeatWave と Object Storage に置くので、作り直しても消えない。
- バックアップ：ボリュームのバックアップは5つまで無料。

### そのほか使うもの

| もの | 上限 |
| --- | --- |
| Flexible Load Balancer | 1台、10Mbps。リスナー・バックエンド・セット・仮想ホスト名は16まで |
| Network Load Balancer | 1台（L4。HTTP 以外を受けたくなったとき用。今は使わない） |
| MySQL HeatWave | 1台。データとログ 50GB、バックアップ 50GB。HeatWave クラスタ（分析・ベクトル検索）1ノードも付く |
| Object Storage | 20GB（無料のみのアカウントの場合。有料アカウントは Standard / Infrequent / Archive が各 10GB）、月5万 API リクエスト |
| VCN | 2つまで |
| 外向きの転送 | 月 10TB |
| Vault | ソフトウェアの鍵は無料、シークレット150 |
| Certificates | CA 5つ、証明書150 |
| Bastion | 無料（数の記載なし） |
| Monitoring / Notifications | 取り込み5億データポイント／HTTPS 100万通・メール1,000通（月） |
| Email Delivery | 月3,000通 |

## 4. ネットワーク

### サブネット

VCN は1つ（`10.0.0.0/16`）。役割ごとにサブネットを分ける（どれもリージョン単位のサブネット）。

| サブネット | CIDR | 置くもの | 外への出口（ルート表） | 分ける理由 |
| --- | --- | --- | --- | --- |
| public-lb | 10.0.0.0/24 | LB | Internet GW | 外から受ける入口 |
| public-dmz | 10.0.1.0/24 | mail | Internet GW | 外から直接受ける VM の置き場（DMZ）。メールは HTTP ではないので LB を通さない |
| private-app | 10.0.10.0/24 | app | NAT GW + Service GW | 公開 IP を持たせない |
| private-db | 10.0.20.0/24 | HeatWave | なし | DB は外に出る必要がない |
| private-ops | 10.0.30.0/24 | ops | NAT GW + Service GW | 勉強で雑多なことをする VM を、本番のアプリから分ける |

- 1つのサブネットのデフォルトの出口は、Internet GW か NAT GW のどちらか一方だけ。外から直接受ける mail は、公開側のサブネットに置く。
- Service Gateway は、OCI のサービス（OCIR、Object Storage、Vault など）へ、インターネットを通らずに出る出口。Oracle が無料と明記している。
- 細かい通信の許可はサブネット（セキュリティ・リスト）ではなく、NSG で行う。サブネットを分ける主な理由は、出口の違いと見通しのよさ。

### NSG（ネットワーク・セキュリティ・グループ）

NSG は、VM や LB など、リソース単位にかけるファイアウォール。相手を IP ではなく「どの NSG に属するか」で指定できる。ルールはステートフル（行きを許せば、戻りは自動で通る）。

| NSG | 付ける先 | 受け付ける | 相手 |
| --- | --- | --- | --- |
| nsg-lb | LB | TCP 443 | Cloudflare の IP 範囲だけ |
| nsg-app | app | TCP 8000〜8099 | nsg-lb |
| | | TCP 22 | Bastion |
| nsg-mail | mail | TCP 25、465、587、993 | 全体 |
| | | TCP 22 | Bastion |
| | | Web メールの画面のポート | nsg-lb（Web メールを付けるときだけ、LB 経由で出す） |
| nsg-ops | ops | TCP 22 | Bastion |
| nsg-db | HeatWave | TCP 3306 | nsg-app、nsg-ops |

- アプリ用のポートは 8000〜8099 を一度に開けておく。アプリを足すたびに NSG を触らない。
- OCI の Ubuntu イメージは、VM の中の iptables で 22 番以外を拒否するのが初期設定。NSG を開けても届かないので、Ansible で VM の中も開ける。

### 要確認

- **NAT Gateway の料金を、公式で確認できていない。** 有料アカウントでは作れる。作った翌日に Cost Analysis で金額を見る。課金されていたら、app と ops を公開サブネットに移し、NSG で絞る形に変える（Terraform の変数1つで切り替えられるように作る）。

## 5. 公開と振り分け

### サブドメインで振り分ける（パスではなく）

| 観点 | サブドメイン（`aaa.alphajinsei.com`） | パス（`apps.alphajinsei.com/aaa`） |
| --- | --- | --- |
| アプリ側の作り | 何もしなくてよい。ルートが `/` のまま | アプリ全体に「`/aaa` の下で動く」設定が要る（base path、リンク、静的ファイル、リダイレクト）。漏れると壊れる |
| アプリどうしの分離 | オリジンが別なので、Cookie や localStorage が混ざらない | 同じオリジンを共有するので、あるアプリの Cookie が別のアプリから見える |
| 方式をまたいだ移動 | DNS を向け直すだけで、①②と③の間でアプリを動かせる | URL が変わる |
| 証明書 | Origin 証明書 `*.alphajinsei.com` 1枚で全部まかなえる | 1枚 |
| 足すときの作業 | DNS レコード1つと LB の設定（どちらも Terraform） | LB の設定だけ |

パス方式の利点は「DNS を触らなくてよい」くらいで、それも Terraform で書けば手間にならない。Cloudflare 側（①②）もサブドメインなので、方式としてもそろう。

### TLS：Cloudflare → LB も HTTPS（Full (strict)）

```
Client --HTTPS--> Cloudflare edge --HTTPS (Origin CA cert, verified)--> OCI LB :443 --HTTP--> app :8000-8099
```

- Cloudflare の暗号化モードは Full (strict)。Cloudflare → LB も HTTPS で、Origin 証明書を検証する。Flexible（Cloudflare → オリジンが HTTP）は使わない。
- TLS は LB で終わり、LB → アプリは VCN の中の HTTP。アプリは TLS を持たない。
- LB が受けるのは 443番だけ。http → https の転送は Cloudflare の「Always Use HTTPS」がエッジで行う。
- 暗号化モードはゾーン全体の設定。Workers と Pages はエッジで動くので影響を受けない。

### LB から先：アプリごとにポートとバックエンド・セット

- アプリごとに、Docker ホストのポートを1つ決める（`aaa` → `:8001`、`bbb` → `:8002`）。ポートは共通基盤の README に台帳を置く。
- LB には、アプリごとに「仮想ホスト名 + リスナー + バックエンド・セット（ヘルスチェック付き）」を作る。LB がアプリ単位で死活を見る。
- 比べた案：各 VM に Traefik などのリバースプロキシを置いて、Host ヘッダーでコンテナに振り分ける（Kubernetes の Ingress 風）。アプリを足すときに LB を触らずに済むが、LB のヘルスチェックはプロキシしか見ない。Docker ホストを2台に戻したとき、片方のアプリだけが死んでも LB が気付かない。LB をアプリの振り分けに使い切るほうを選んだ。
- Kubernetes の Deployment のような「台数の保証」はない。コンテナの再起動は Docker の `restart: unless-stopped` に任せ、止まったことは LB のヘルスチェックのアラームで知る。
- LB のリスナーは16までなので、載せられるアプリも16個まで（実装のときに確かめる）。

## 6. DB

| | **MySQL HeatWave** | Autonomous DB | NoSQL Database | VM 上の PostgreSQL（コンテナ） |
| --- | --- | --- | --- | --- |
| 容量 | 50GB（バックアップは別に 50GB） | 20GB、1 OCPU、同時に20セッション | 1表 25GB、3表まで | VM のディスク |
| 経路 | VCN の中の非公開 IP | 公開のエンドポイント（ACL + Wallet の mTLS）。無料版は非公開にできない | 公開の API（IAM 認証） | VM の中 |
| 止まる・回収される | 記載なし | 7日使わないと止まり、止まったまま90日で回収 | なし | なし |
| アプリからの使いやすさ | ◎ 普通の MySQL。Drizzle などの ORM がそのまま使える | △ Oracle SQL で、Node の ORM はほぼ非対応。APEX や JSON（Mongo 互換 API）は強い | △ 専用の SDK で使う KV / ドキュメント型。リレーションはない | ◎ 一番素直 |
| 運用 | 自動バックアップ付きのマネージド | マネージド | マネージド | バックアップも障害対応も自分で |
| 向く用途 | 共通基盤の標準の DB | Oracle 固有の機能を試したいアプリ、APEX | 単純な KV で量が多いもの | 採用しない |

- **結論：標準の DB は MySQL HeatWave。** ADB と NoSQL は「そのアプリに合うときだけ、アプリ側で足す」選択肢として残す。要件が変わったら考え直す。
- PostgreSQL は Always Free にない（OCI Database with PostgreSQL は有料）。
- アプリごとに、スキーマと DB ユーザーを1組作る（D1 の「アプリごとに DB」に当たる）。

## 7. SSH と構成管理

### 道具の分担

| 道具 | 管理するもの | 実行する場所 |
| --- | --- | --- |
| Terraform（OCI provider、Cloudflare provider） | OCI のリソースすべて（VCN、NSG、VM、ボリュームのアタッチ、LB、HeatWave、Bastion、Vault、監視、予算）と Cloudflare の DNS レコード | 手元の WSL |
| Ansible | VM の中：パッケージと OS の更新の設定、iptables、Docker、ブロックボリュームのフォーマットとマウント、メールサーバーの設定 | 手元の WSL（Bastion 経由の ssh） |
| デプロイのスクリプト | アプリのコンテナの入れ替え、マイグレーション | 手元の WSL（`npm run deploy`） |

- **state は OCI Object Storage に置く。** Terraform の公式の `oci` backend を使う（ロックあり。S3 互換の方法は非推奨）。バケットだけは最初に CLI で手作りし、バージョニングを有効にする（R2 のときと同じ「鶏と卵」）。キーは `<リポジトリ名>/terraform.tfstate`。
- **Ansible を使う。** VM が ops・app・mail の3種類あり、ボリュームのマウント、メールの設定、OS の更新、脆弱性への対応など、VM の中の作業を続けて回すため。cloud-init は使わず、VM の中の設定は Ansible に寄せる。
- ブロックボリュームは準仮想化（paravirtualized）でアタッチする（iSCSI のコマンドが要らない）。LVM は使わず、ボリュームに直接ファイルシステムを作る。容量を増やすときは、OCI 側で大きくしてから `resize2fs`。

### SSH は Bastion

- Bastion は OCI が持つ踏み台。「この鍵で、この VM の 22 番に、最長3時間」というセッションを作り、その間だけ通れる。VM に公開 IP も、外に開いたポートも要らない。
- 手間を減らすため、ラッパーのスクリプト（例：`bin/oci-ssh`）を作る。使えるセッションが残っていればそれを使い、なければ作って待ち（1分ほど）、ssh の設定を書き換える。`ssh oci-app` のように入れ、Ansible とデプロイのスクリプトも中でこれを使う。
- 非常口：OCI のコンソールからのコンソール接続（シリアルコンソール）。ネットワークを通らずに VM に入れる。
- 手で入る場面は少ない。デプロイ（毎回、ただしスクリプトが入る）、`docker logs` などの調べもの（たまに）、OS の更新（自動にする）。

## 8. デプロイ（CI/CD）

②と同じく「手元で確かめて、手元から1コマンドで出す。git は保管用」。

```
Dev PC (WSL)
  docker compose up        app + mysql:8.4 container (same version as HeatWave)
  npm run deploy
    1. docker buildx build --platform linux/arm64 -> push to OCIR (tag = git SHA)
    2. ssh app (via Bastion): pull -> migrate (docker compose run --rm app migrate)
                              -> up -d -> health check
  git push                 storage only
```

- **イメージの置き場は OCIR**（OCI Container Registry）。サービス自体は無料で、容量は Object Storage の料金で数えられる。Always Free に含まれるかは公式で確認できていないので、最初の push の後に Cost Analysis を見る。古いタグは保持ポリシーで自動で消す。課金が出たら GHCR（公開イメージは無料）に替える。
- **A1 は Arm（arm64）。** WSL は x86 なので、buildx でクロスビルドする（エミュレーションで遅め）。遅すぎたら app VM の上でビルドする形に変える。
- **マイグレーションは app VM の上で流す。** HeatWave は VCN の中にあり、手元から届かない。②の「マイグレーション + `wrangler deploy`」と同じく、デプロイのスクリプトに含める。アプリごとのスキーマと DB ユーザーは、初回だけ `npm run db:init` のようなスクリプトで作る。
- Docker ホストが1台なので、入れ替えの数秒はそのアプリが止まる。受け入れる。2台に戻したら、1台ずつ入れ替え、マイグレーションは「列を足すのが先、消すのは後」（expand / contract）で書く。
- GitHub Actions は使わない（②と同じ理由）。使うなら、公開リポジトリは Arm の runner が無料で使える。

## 9. 共通基盤とアプリの境界

リポジトリは2段に分ける。①②はインフラ層が完全にマネージドだったので、サンプルのリポジトリ1つで足りた。③はインフラ層そのものを作るので、共通基盤とアプリを分ける。

```
~/claude-sandbox/
├── oci-platform/            platform: VCN, NSG, LB, VMs, HeatWave, Bastion, Vault,
│                            monitoring, budget (Terraform) + Ansible + bin/oci-ssh
└── sample-oci-vm-docker/    app: code, Dockerfile, compose.yaml, deploy script,
                             small Terraform (LB listener + backend set, DNS record,
                             bucket) reading platform outputs via terraform_remote_state
```

アプリ側の Terraform にアプリ固有の LB 設定を持たせるので、「1サービス＝1ディレクトリ」は③でも残る。

### アプリが受け取るもの（共通基盤の仕様）

| もの | 内容 |
| --- | --- |
| ドメイン | `<名前>.alphajinsei.com`（HTTPS は LB と Cloudflare が持つ。アプリは HTTP で待つ） |
| ポート | 台帳から1つ（8000〜8099） |
| DB | HeatWave にスキーマ1つと専用のユーザー。接続情報は Vault から環境変数に入る |
| ファイル置き場 | Object Storage のバケット（任意）。VM のインスタンス・プリンシパル（VM 自身に持たせた IAM の権限）で読み書きする。FSS（NFS）は Always Free にないので使わない |
| メールの送信 | OCI Email Delivery（SMTP 587） |
| 監視 | LB のヘルスチェック（アプリは `/health` を返す）と、その異常のメール通知 |

### アプリが守ること

- **ステートレスに作る。** 状態は DB と Object Storage に置く（Workers と同じ。Docker ホストを2台に戻しても、そのまま動くように）。
- arm64 で動くイメージにする。
- 定時処理はアプリの中に持たず、ops の cron から起動する（2台に戻したとき、二重に実行されないように）。

## 10. メール基盤と認証基盤

### メール：受信は自前、送信は Email Delivery に中継

| 方向 | どうするか | 理由 |
| --- | --- | --- |
| 受信 | 外から mail VM の 25 番に直接届く。MX レコードは mail の予約パブリック IP に向ける | OCI が塞いでいるのは外へ出す 25 番だけで、受信の 25 番は塞がれていない |
| 送信 | 自前の Postfix から OCI Email Delivery に 587 番で中継する（スマートホスト） | 外への 25 番の解除の申請も、クラウドの IP の評判の問題も避けられ、Gmail などに届きやすい |
| 認証の DNS | SPF・DKIM・DMARC のレコードを Cloudflare に置く | |
| データ | メールボックスはブロックボリュームに置く | VM を作り直しても残す |

受信・メールボックス・IMAP は自分で持ち、配達だけをマネージドに任せる。メールは状態を持つサービスなので、Docker ホストではなく VM 単体で動かす。

要確認：予約パブリック IP が無料か。OCI の公開 IP に逆引き（PTR）を自分で設定できるか。

### 認証基盤

- Docker ホストのアプリとして載せる（例：`auth.alphajinsei.com`）。Keycloak は MySQL に対応している。Authentik と Zitadel は PostgreSQL が必須なので、このままでは載らない。
- 自作する場合も同じで、LB の振り分け1つと HeatWave のスキーマ1つを足すだけ。
- 自分で動かさない選択肢として、OCI IAM Identity Domains（無料のタイプがある）と Cloudflare Access がある。

### まだ載せていない処理の方式

| 処理の方式 | 今の構成で | 対応 |
| --- | --- | --- |
| 定時処理・バッチ | ops の cron で動かす。重いものは app のコンテナとして起動する | 9 の「アプリが守ること」 |
| PostgreSQL が必須の OSS、Redis、キュー | 標準の部品にない（OCI Cache は有料） | 必要になったら app にコンテナで置き、単一障害点として割り切る |
| 大きなファイルの配信 | LB の帯域が 10Mbps で詰まる | Object Storage の期限付き URL（PAR）か R2 から直接配る |
| HTTP 以外の通信（ゲーム、MQTT など） | Flexible LB では受けない | Network Load Balancer（無料で1台）か、public-dmz の VM |
| WebSocket | Flexible LB で通せる | LB のアイドルタイムアウトを延ばす |

## 11. 運用

| もの | 使い方 |
| --- | --- |
| Vault | DB のパスワード、OCIR のトークン。VM はインスタンス・プリンシパルで取りに行く（手元にも VM のディスクにも置かない） |
| Monitoring + Notifications | アラーム：LB の異常なバックエンドが1つ以上、VM の CPU・メモリ、HeatWave のストレージ。通知はメール |
| Budget | 予算アラート（金額が0円を超えたら通知）。枠を超えて課金されたことに気付くため |
| OS の更新 | unattended-upgrades で自動。設定は Ansible |
| バックアップ | HeatWave は自動バックアップ。メールのボリュームはボリュームのバックアップ。Terraform の state はバケットのバージョニング |

## 12. 要確認のまとめ

作ってから Cost Analysis で0円かを確かめるもの：

- NAT Gateway の料金
- 予約パブリック IP の料金
- ボリュームのバックアップ（容量の上限の記載がない）
- OCIR のイメージの容量

調べるもの：

- OCI の公開 IP の逆引き（PTR）を設定できるか
- LB のリスナーと仮想ホスト名で、アプリ16個を独立したリソースとして足せるか

## 13. 検討の経緯

1. **当初の案**（Cloudflare の検討のとき）：OCI の VM 1台 + Docker + Dokploy/Coolify + Cloudflare Tunnel。DB は VM 上の PostgreSQL。Workers で動かないものの受け皿として、最小の形を考えていた。
2. **3層の構成に広げた**：Always Free に LB、MySQL HeatWave、Bastion、Vault、監視などが含まれ、「LB があってサーバーがあって DB がある」教科書どおりの構成が無料で組めると分かった。学習にもなるので、Tunnel で直接公開する形から、Cloudflare → LB → VM → DB の形に変えた。Dokploy は入れず、デプロイは①②と同じく手元から1コマンドにした。
3. **DB**：PostgreSQL は Always Free にない。HeatWave（MySQL）・Autonomous DB・NoSQL・VM 上の PostgreSQL を比べ、非公開の経路、自動停止がないこと、ORM がそのまま使えることから HeatWave にした（6）。
4. **アプリ VM 2台 → 1台**：最初は A1 を2台に分けて冗長化する案だった。しかし A1 の枠が 4 OCPU / 24GB から 2 / 12 に減っていた。さらにディスクの 200GB はブートボリュームも込みで、作業用の VM とメール用の VM も欲しいことから、Docker ホストは1台にして冗長化をあきらめた。LB は残した（2）。
5. **SSH：Cloudflare Tunnel → Bastion**：Tunnel なら `ssh` が普通に通るが、Cloudflare 側の障害時に入れなくなるのが気になった。Tunnel の本来の用途は自宅のサーバーを公開することで、今回は枝葉の使い方になる。OCI の標準の踏み台である Bastion にし、手間はラッパーのスクリプトで減らす（7）。
6. **VM の中の設定：cloud-init → Ansible**：VM が1種類で Docker を入れるだけなら cloud-init で足りた。しかし ops・app・mail の3種類になり、ボリュームのマウントやメールの設定、継続的な OS の更新と脆弱性への対応が要るので、Ansible にした（7）。
7. **Terraform の state：R2 → OCI Object Storage**：OCI の中で閉じるほうが③には合う。公式の `oci` backend がロックにも対応している（7）。
8. **作業用の VM**：既存のマイクロ2台（2023年に勉強で作ったもの）は、ブートボリュームのバックアップを取ってから消す。残すものを調べる手間を、必要になったときに先送りできる。作業用の VM は、新しい VCN に作り直す（ops）。
