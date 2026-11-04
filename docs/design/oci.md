# 個人サービス公開インフラ 方式検討：OCI（方式③）

最終更新：2026-11-04

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
|   host-based routing, 1 backend set/app      mail data in /srv/mail        |
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
  Vault: DB passwords                    Monitoring + Notifications: alarms -> email
  Budget: alert when cost > 0

Dev PC (WSL) -- Terraform ----------------------> OCI resources, Cloudflare DNS
Dev PC (WSL) -- Ansible (ssh via Bastion) ------> inside of VMs
Dev PC (WSL) -- npm run deploy -----------------> rsync source -> build on app VM
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

### OS：Ubuntu 24.04

- コンテナを載せる基盤として素直な方を選ぶ。Docker の公式のサポート対象で、コンテナ・メールサーバー・Arm の情報も、Ubuntu / Debian 前提のものが一番多い。
- 比べた Oracle Linux 9（業務の RHEL と同じ系統）は、Docker の公式のサポート対象外（動くが、Oracle が推すのは podman）。SELinux と firewalld が、Docker やメールサーバーと噛み合わない場面があり（ボリュームのラベル、firewalld の reload で Docker のルールが消える、など）、外側を NSG で守る今回の構成では、引っかかるだけ損になる。マイクロ（1GB）では dnf も重い。
- 再起動なしのカーネルの更新は、Ubuntu Pro の Livepatch（個人は5台まで無料）で試せる。

### コンピュート

| 枠 | 上限 | 割り振り |
| --- | --- | --- |
| A1（Arm、VM.Standard.A1.Flex） | 計2 OCPU / 12GB（以前は 4 / 24 だった） | app：1 OCPU / 8GB、mail：1 OCPU / 4GB |
| E2.1.Micro（AMD） | 2台。1台あたり 1/8 OCPU、1GB | ops：1台。もう1台は使わない |

- A1 とマイクロは別の枠で、同時に使える。
- Docker ホストに多めに配る。Keycloak や自作の認証基盤（Java などはメモリを 1〜2GB 使う）が載るため。メールは Postfix + Dovecot + Rspamd なら 4GB で足り、ClamAV を足しても収まる。
- A1 は東京などで「Out of host capacity」（空きがない）が出やすい。時間を置くか、別の AD で試す。

### ディスク（ブートボリュームとブロックボリュームの合計で 200GB、ブロックボリュームは2つまで・1つ 50GB 以上）

| VM | ブート | ブロック | 計 |
| --- | --- | --- | --- |
| ops | 47 | | 47 |
| app | 47 | | 47 |
| mail | 47 | | 47 |
| 合計 | | | **141GB（59GB 余る）** |

- ブートボリュームは 47GB より小さくできない。そのため、VM 4台に別のボリュームを足すと 200GB を超える。マイクロ2台に A1 2台（4台）は入るが、余りがなくなる。そこでマイクロは1台に減らした。
- コンテナイメージは1つ数百MB で、`docker image prune` で古いものを消せば、ブートボリュームで足りる。足りなくなったら、ブートボリュームを止めずに大きくできる。
- ブロックボリュームは使わない。最小が 50GB で枠を大きく食う一方、利点は「VM を作り直してもデータが残る」ことだけ。メールのデータはブートボリュームの `/srv/mail` に置き、ブートボリュームのバックアップと、Object Storage への定期的な退避で守る（Step 8）。
- あとでブロックボリュームに移すときは、ボリュームを付ける → メールを止める → `/srv/mail` を rsync でコピー → `/srv/mail` にマウントし直す、で済む（30分ほど）。そのために、メールのデータは最初から `/srv/mail` にまとめる。
- app のデータは HeatWave と Object Storage に置くので、作り直しても消えない。
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
- OCI の Ubuntu イメージは、VM の中の iptables でも 22 番以外を拒否するのが初期設定。NSG を開けても届かないので、Ansible で VM の中も開ける（mail のポート）。Docker で公開するポートは Docker が自分で足す。

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

- Cloudflare の暗号化モードは Full (strict)。Cloudflare → LB も HTTPS で、Origin 証明書を検証する。
- TLS は LB で終わり、LB → アプリは VCN の中の HTTP。アプリは TLS を持たない。
- LB が受けるのは 443番だけ。Cloudflare は利用者と同じプロトコルでオリジンにつなぐ（http なら 80番）ので、http → https の転送は Cloudflare の「Always Use HTTPS」がエッジで行う。
- 暗号化モードはゾーン全体の設定。Workers と Pages はエッジで動き、Cloudflare → オリジンの区間がないので影響を受けない。ほかのホスト名と分けたくなったら、Configuration Rules でホスト名ごとに上書きできる（Free プランでも使える）。
- LB のリスナーは TLS 1.2 だけにした。OCI の LB の既定の暗号スイートが TLS 1.2 用で、1.3 を足すには暗号スイートの指定も替える必要がある。利用者 → Cloudflare の区間は Cloudflare が TLS 1.3 とハイブリッドの PQC 鍵交換を受け持つので、この区間を 1.3・PQC に上げるのは、基盤を回す中で試す題材として残す。

#### 案の比較

暗号化モードが決めるのは Cloudflare → オリジンの区間だけ。利用者 → Cloudflare は、どの案でも Cloudflare のエッジ証明書（自動で発行・更新）による HTTPS。

| 案 | Cloudflare → OCI | オリジンの証明書 | 残るリスク | 初期の手間 |
| --- | --- | --- | --- | --- |
| Flexible | HTTP（平文） | なし | インターネットの上を平文で流れる（認証基盤のパスワードや Cookie も）。アプリが http と見て https へ転送し、ループする罠がある | 小（LB に 80番） |
| Full + 自己署名 | HTTPS・相手を検証しない | 自己署名 | 経路で割り込まれても気付かない。DNS の IP の間違いや、LB を作り直して手放した IP が他人に渡ったときも、黙ってそこにつなぐ | 小（Origin 証明書と変わらない） |
| Full + Origin 証明書 | HTTPS・相手を検証しない | Origin 証明書 | 同上 | 小 |
| **Full (strict) + Origin 証明書** | HTTPS・相手を検証する | Origin 証明書 | 上の2つは、エラー（526）で止まる。証明書の秘密鍵が漏れると、なりすましに使える | 小（上 + モードの切り替え） |
| プロキシなし（DNS だけ） | （Cloudflare を通らない） | 公的に信頼される証明書（Let's Encrypt など） | LB が全世界に公開され、IP も知られる。WAF・DDoS 対策・キャッシュがない。更新が止まるとサイトが落ちる | 大（ACME の仕組み、LB の証明書の差し替えの自動化、80番の転送） |

- Full と Full (strict) は、手間がほぼ同じで、Workers・Pages への影響もないので、strict にした。
- プロキシの形そのものに残るリスク：Cloudflare は中身を平文で見る。NSG で Cloudflare の IP に絞っても、ほかの Cloudflare 利用者が自分のゾーンをこちらの IP に向ければ、Cloudflare 経由で届く。防ぐなら Authenticated Origin Pulls（Cloudflare → オリジンの mTLS）。後で試す題材として残す。
- プロキシにした理由：①②と入口がそろう（DNS の向け先を変えるだけで方式をまたいで移せる）、WAF・DDoS 対策・キャッシュが無料で付く、LB の IP を隠して NSG で絞れる、証明書の更新を考えなくてよい。

#### Origin 証明書とは

| | Cloudflare Origin 証明書 | 公的な証明書（Let's Encrypt、商用の CA） | OCI Certificates（プライベート CA） |
| --- | --- | --- | --- |
| 発行する CA | Cloudflare の Origin CA | ブラウザや OS が信頼する CA | 自分で OCI に作る CA |
| 信頼するのは | Cloudflare のエッジだけ | 誰でも（ブラウザ・OS・curl） | その CA を入れた相手だけ |
| 有効期限 | 7日〜15年から選ぶ | 業界のルール（CA/Browser Forum）で上限がある。2026年3月から200日、2027年3月から100日、2029年3月から47日。Let's Encrypt は90日 | 自分で決める。OCI が自動で更新できる |
| 料金 | 無料 | Let's Encrypt は無料、商用は有料 | Always Free で CA 5つ・証明書150 |
| Full (strict) で使えるか | 使える | 使える | 使えない（有料の機能でオリジンの信頼ストアを足さない限り） |

- 有効期限の短縮は、ブラウザが信頼する公的な証明書のルール。Origin 証明書は公的な CA が出すものではなく、Cloudflare だけが信頼する私的な証明書なので、ルールの外にあり、15年で作れる。
- そのかわり、ブラウザに直接見せると警告が出る。プロキシ（オレンジの雲）を外すと、HTTPS も通らなくなる。
- 期限が長いぶん、秘密鍵が漏れたときの影響も長く続く。漏れたら Cloudflare のダッシュボードで失効させ、作り直す。
- 1枚で `alphajinsei.com` と `*.alphajinsei.com`（1階層）をまかなえる。
- 発行は手で行う（Cloudflare のダッシュボード → SSL/TLS → オリジンサーバー）。15年に一度の作業で、仕組みが見える方を選んだ。
- OCI には、ブラウザが信頼する証明書を発行する機能はない（AWS の ACM に当たるものがない）。OCI Certificates は、プライベート CA での発行と、外で作った証明書の取り込み（期限の管理だけ。自動更新はしない）を受け持つ。

### LB から先：アプリごとにポートとバックエンド・セット

- アプリごとに、Docker ホストのポートを1つ決める（`aaa` → `:8001`、`bbb` → `:8002`）。ポートは共通基盤の README に台帳を置く。8000 番は共通基盤の既定のページ（どのアプリにも当たらないホスト名の行き先で、LB のヘルスチェックの相手にもなる）。
- LB の 443 番には、ホスト名を持たない既定のリスナー（共通基盤）と、ホスト名付きのアプリのリスナー（各アプリのリポジトリの Terraform）が並ぶ。
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
- 作ってみて分かったこと：バックアップの設定は Always Free では指定できず、既定（有効、保存1日）になる。HeatWave は VNIC の作成と NSG への参加を DB システム自身の権限で行うので、そのためのポリシー（`request.principal.type='mysqldbsystem'`）が要る。作成時の版は 26.10.0（手元の開発用の MySQL コンテナはこれに合わせる）。
- 版は作ったときの既定のままにし、上げるのは OCI のメンテナンスに任せる（Terraform では版の差分を無視する）。データを持つので、削除保護を付け、`terraform destroy` や手違いで消えないようにする。
- アプリごとに、スキーマと DB ユーザーを1組作る（D1 の「アプリごとに DB」に当たる）。

## 7. SSH と構成管理

### 道具の分担

| 道具 | 管理するもの | 実行する場所 |
| --- | --- | --- |
| Terraform（OCI provider、Cloudflare provider） | OCI のリソースすべて（VCN、NSG、VM、ボリュームのアタッチ、LB、HeatWave、Bastion、Vault、監視、予算）と、自分が作った Cloudflare の DNS レコードだけ | 手元の WSL |
| 手（Cloudflare のダッシュボード） | Cloudflare のゾーンとアカウントの設定：暗号化モード、Always Use HTTPS、Origin 証明書の発行、API トークン、メールのレコード | — |
| Ansible | VM の中：パッケージと OS の更新の設定、iptables、Docker、swap、メールサーバーの設定。手で流す更新（`upgrade.yml`） | 手元の WSL（Bastion 経由の ssh） |
| デプロイのスクリプト | アプリのコンテナの入れ替え、マイグレーション | 手元の WSL（`npm run deploy`） |

- **state は OCI Object Storage に置く。** Terraform の公式の `oci` backend を使う（ロックあり。S3 互換の方法は非推奨）。バケットだけは最初に CLI で手作りし、バージョニングを有効にする（R2 のときと同じ「鶏と卵」）。キーは `<リポジトリ名>/terraform.tfstate`。
- **Cloudflare は、ゾーンの設定は手、レコードはそれを使うアプリの道具が持つ。**
  - Terraform が Cloudflare で触るのは DNS レコードだけ。リポジトリの中でも `cloudflare.tf` に分け、OCI ではない場所を触っている箇所が見て分かるようにする。トークンの権限も Zone Read と DNS Edit だけにする（共通基盤とアプリで共用）。
  - 共通基盤は確認用の `oci-platform.alphajinsei.com` だけを持つ。アプリのレコード（`<名前>.alphajinsei.com`）は、そのアプリのリポジトリの Terraform が、LB のリスナーやバックエンド・セットと一緒に持つ。②で wrangler が Worker の独自ドメインのレコードを作るのと同じく、「アプリを出す道具が、そのアプリのレコードを持つ」。ゾーン全体を1つの Terraform で管理するのではない。
  - Terraform が作ったレコードには、コメントに `managed by Terraform: <リポジトリ名>` を入れ、ダッシュボードで持ち主が分かるようにする。
  - 比べた案：全部を手で持つと、アプリを足すたびに手作業が残り、デプロイを Claude に任せにくい。ワイルドカード（`*.alphajinsei.com` → LB）にすると追加の作業がなくなるが、レコードを作り忘れたホスト名が黙って OCI に流れ、行き先が暗黙に決まる。Cloudflare 公式の CLI `cf`（beta）でもレコードは作れるが、③のアプリはどのみち LB の設定を Terraform で書くので、同じ場所に置いた。Wrangler には任意の DNS レコードを作る機能がない。
- **Ansible を使う。** VM が ops・app・mail の3種類あり、ボリュームのマウント、メールの設定、OS の更新、脆弱性への対応など、VM の中の作業を続けて回すため。cloud-init は使わず、VM の中の設定は Ansible に寄せる。

### SSH は Bastion

- Bastion は OCI が持つ踏み台。「この鍵で、この VM の 22 番に、最長3時間」というセッションを作り、その間だけ通れる。VM に公開 IP も、外に開いたポートも要らない。
- 手間を減らすため、ラッパーのスクリプト `bin/bastion-proxy` を置く。Terraform が書き出す `ssh_config` の ProxyCommand から呼ばれ、使えるセッション（同じ宛先・同じ鍵で、残り15分以上）があればそれを使い、なければ作って ACTIVE を待つ（1分ほど）。最後に `ssh -W` で Bastion を通す。`ssh oci-app` のように入れ、Ansible とデプロイのスクリプトも同じ経路を通る。仕組みと手でつなぐ手順は [番外編：Bastion](../notes/extra-oci-bastion.md)。
- Bastion への接続に使う鍵は RSA にする（Bastion は RSA しか受け付けない。ed25519 でもセッションは作れてしまい、接続で拒否される）。VM へのログインの鍵は ed25519 のまま、2つを使い分ける。
- Bastion への接続元は絞らない（0.0.0.0/0）。セッションを作るには OCI の認証が要り、さらに Bastion の鍵と VM の鍵が要る、の3重で守る。自宅の IP に絞ると、IP が変わるたびに直す手間がかかり、外出先からも入れなくなる。
- 非常口：OCI のコンソールからのコンソール接続（シリアルコンソール）。ネットワークを通らずに VM に入れる。
- 手で入る場面は少ない。デプロイ（毎回、ただしスクリプトが入る）、`docker logs` などの調べもの（たまに）、OS の更新（自動にする）。

## 8. デプロイ（CI/CD）

②と同じく「手元で確かめて、手元から1コマンドで出す。git は保管用」。

```
Dev PC (WSL)
  docker compose up        app + mysql container (same version as HeatWave, 26.10)
  npm run deploy
    1. rsync source -> app:/opt/apps/<name>/ (via Bastion)
    2. on app: write .env (DB password from Vault, instance principal)
               -> tag current image :previous -> docker compose build (arm64, native)
               -> migrate (docker compose run --rm app) -> up -d -> health check
  git push                 storage only
```

- **イメージは運ばず、app VM の上でビルドする。** 手元で確かめたイメージをそのまま運べないのは、手元（x86）と app（Arm、arm64）で CPU が違うため。OCIR に置く形でも、手元で arm64 向けに別にビルドし直す（エミュレーションで遅い）ことになり、確かめたイメージそのものを運ぶわけではない。それなら、同じソース・同じ `package-lock.json`・同じベースイメージから、app の上でそのままビルドするほうが速く、レジストリの認証トークンもいらない。
- 戻すときのために、入れ替える前のイメージを `:previous` のタグで VM に残す。イメージの履歴はこの1つだけ。
- 比べた案：**OCIR**（OCI Container Registry）。標準的な形でイメージの履歴も残るが、認証トークンが手元と app の両方に要り、容量の課金も公式で確かめられていない。レジストリの運用を学ぶ題材として、後から足せる。
- **DB のパスワードは手元を通らない。** アプリの Terraform がパスワードを作って共通基盤の Vault に入れ、デプロイのとき app VM が `platform-secret`（インスタンス・プリンシパル）で取り出して `.env`（権限 600）に書く。コンテナの環境変数は Docker が設定ファイルに保存するので、VM のディスクには残る。そこは割り切る。
- **マイグレーションは app VM の上で流す。** HeatWave は VCN の中にあり、手元から届かない。②の「マイグレーション + `wrangler deploy`」と同じく、デプロイのスクリプトに含める。アプリごとのスキーマと DB ユーザーは、初回だけ `npm run db:init` で作る（ops から。ops には mysql のクライアントと、Vault から管理者のパスワードを読む権限がある）。
- Docker ホストが1台なので、入れ替えの数秒はそのアプリが止まる。受け入れる。2台に戻したら、1台ずつ入れ替え、マイグレーションは「列を足すのが先、消すのは後」（expand / contract）で書く。
- GitHub Actions は使わない（②と同じ理由）。使うなら、公開リポジトリは Arm の runner が無料で使える。

## 9. 共通基盤とアプリの境界

リポジトリは2段に分ける。①②はインフラ層が完全にマネージドだったので、サンプルのリポジトリ1つで足りた。③はインフラ層そのものを作るので、共通基盤とアプリを分ける。

```
~/claude-sandbox/
├── oci-platform/            platform: VCN, NSG, LB, VMs, HeatWave, Bastion, Vault,
│                            monitoring, budget (Terraform) + Ansible + bin/bastion-proxy
└── sample-oci-vm-docker/    app: code, Dockerfile, compose.yaml, deploy script,
                             small Terraform (LB listener + backend set, DNS record,
                             bucket) reading platform outputs via terraform_remote_state
```

アプリ側の Terraform にアプリ固有の LB 設定を持たせるので、「1サービス＝1ディレクトリ」は③でも残る。

### 誰が何を持つか

共通基盤は一度作れば終わり（更新はするが、アプリを足しても触らない）。アプリを足すたびに増えるのは右の列だけで、すべてアプリのリポジトリから行う。

| もの | 共通基盤（一度だけ） | アプリごと（足すたび） |
| --- | --- | --- |
| Cloudflare のゾーン | 手：暗号化モード Full (strict)、Always Use HTTPS、Origin 証明書の発行、API トークン | なし |
| DNS レコード | Terraform（`oci-platform/terraform/cloudflare.tf`）：確認用の `oci-platform` | Terraform（`<name>/infra/terraform/cloudflare.tf`）：`<name>.alphajinsei.com` → LB |
| LB | Terraform（`lb.tf`）：LB、証明書、既定のリスナーとバックエンド・セット（8000） | Terraform（`main.tf`）：ホスト名、リスナー（443、ホスト名付き）、バックエンド・セット（`/health`）、バックエンド（app:ポート） |
| ポート | NSG で 8000〜8099 をまとめて開ける | 手：共通基盤の README の台帳に1行足す |
| app VM | Ansible：OS、Docker、iptables、oci CLI、`platform-secret`、`/opt/apps`、既定のページ | `npm run deploy`：`/opt/apps/<name>/` のソース、`.env`、イメージ、コンテナ |
| HeatWave | Terraform：DB システム、管理者のユーザー | `npm run db:init`（初回だけ、ops から）：スキーマと専用のユーザー。マイグレーションは毎回の `npm run deploy` |
| Vault | Terraform：Vault、鍵、管理者のパスワード、VM が読むためのポリシー | Terraform（`main.tf`）：`<name>-db-password` |
| state | 手（Step 0）：バケット `tfstate`。キーは `oci-platform/terraform.tfstate` | キー `<name>/terraform.tfstate`（`versions.tf`） |
| SSH（Bastion、`ssh_config`） | Terraform + `bin/bastion-proxy` | なし（スクリプトが `ssh oci-app` / `ssh oci-ops` を使う） |

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
| データ | メールボックスは `/srv/mail`（ブートボリューム）に置き、Object Storage に定期的に退避する | ブロックボリュームは枠を食うので使わない（3） |

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
| Vault | DB のパスワード（管理者とアプリごと）。VM はインスタンス・プリンシパルで取りに行く（`platform-secret <名前>`。手元には置かない）。Vault は DEFAULT 型（無料）、鍵はソフトウェアの鍵（無料）にした。専用の区画を持つ仮想プライベート Vault は有料。Vault は消すと復元の猶予の後に戻せないので、Terraform で `prevent_destroy` を付ける。HeatWave の管理者のパスワードは Terraform が作って Vault に入れるので、値は state にも残る（state は非公開のバケットにある） |
| Monitoring + Notifications | アラーム：LB の異常なバックエンドが1つ以上、VM の CPU・メモリ、HeatWave のストレージ。通知はメール |
| Budget | 予算アラート（金額が0円を超えたら通知）。枠を超えて課金されたことに気付くため |
| OS の更新（自動） | unattended-upgrades でセキュリティの更新だけ毎日入れる。**自動で再起動しない。Docker 本体は対象外**。needrestart にも Docker を再起動させない。Docker は `live-restore` で、デーモンが再起動してもコンテナを止めない。これで、勝手にコンテナが止まることはない |
| OS の更新（手動） | 月に1回などで `ansible-playbook upgrade.yml`。カーネルと Docker も含めて全部更新し、再起動が要る VM だけ1台ずつ再起動する（その間、その VM のコンテナは1〜2分止まる）。コンテナの中身は OS の更新では直らないので、イメージの作り直し（デプロイ）で更新する |
| バックアップ | HeatWave は OCI の既定の自動バックアップ（Always Free では設定を指定できない）。メールのデータ（`/srv/mail`）は Object Storage に定期的に退避する（Step 8）。VM はブートボリュームのバックアップ（5つまで無料）。Terraform の state はバケットのバージョニング |

## 12. 要確認のまとめ

作ってから Cost Analysis で0円かを確かめるもの：

- NAT Gateway の料金
- 予約パブリック IP の料金
- ボリュームのバックアップ（容量の上限の記載がない）

調べるもの：

- OCI の公開 IP の逆引き（PTR）を設定できるか
- LB のリスナーと仮想ホスト名で、アプリ16個を独立したリソースとして足せるか
- 予約パブリック IP の料金。今は LB の公開 IP を予約せず、作ってから DNS に設定する。LB を作り直すと IP が変わるので、そのときは DNS を直す（Full (strict) なので、直し忘れても他人の IP に黙ってつなぐことはない）

## 13. 検討の経緯

1. **当初の案**（Cloudflare の検討のとき）：OCI の VM 1台 + Docker + Dokploy/Coolify + Cloudflare Tunnel。DB は VM 上の PostgreSQL。Workers で動かないものの受け皿として、最小の形を考えていた。
2. **3層の構成に広げた**：Always Free に LB、MySQL HeatWave、Bastion、Vault、監視などが含まれ、「LB があってサーバーがあって DB がある」教科書どおりの構成が無料で組めると分かった。学習にもなるので、Tunnel で直接公開する形から、Cloudflare → LB → VM → DB の形に変えた。Dokploy は入れず、デプロイは①②と同じく手元から1コマンドにした。
3. **DB**：PostgreSQL は Always Free にない。HeatWave（MySQL）・Autonomous DB・NoSQL・VM 上の PostgreSQL を比べ、非公開の経路、自動停止がないこと、ORM がそのまま使えることから HeatWave にした（6）。
4. **SSH：Cloudflare Tunnel → Bastion**：Tunnel なら `ssh` が普通に通るが、Cloudflare 側の障害時に入れなくなるのが気になった。Tunnel の本来の用途は自宅のサーバーを公開することで、今回は枝葉の使い方になる。OCI の標準の踏み台である Bastion にし、手間はラッパーのスクリプトで減らす（7）。
5. **VM の中の設定：cloud-init → Ansible**：VM が1種類で Docker を入れるだけなら cloud-init で足りた。しかし ops・app・mail の3種類になり、ボリュームのマウントやメールの設定、継続的な OS の更新と脆弱性への対応が要るので、Ansible にした（7）。
6. **Terraform の state：R2 → OCI Object Storage**：OCI の中で閉じるほうが③には合う。公式の `oci` backend がロックにも対応している（7）。
7. **アプリ VM 2台 → 1台**：最初は A1 を2台に分けて冗長化する案だった。しかし A1 の枠が 4 OCPU / 24GB から 2 / 12 に減っていた。さらにディスクの 200GB はブートボリュームも込みで、作業用の VM とメール用の VM も欲しいことから、Docker ホストは1台にして冗長化をあきらめた。LB は残した（2）。
8. **OS：Ubuntu → Oracle Linux 9 → Ubuntu**：業務で RHEL を使っているので、一度は同じ系統の Oracle Linux 9 に書き換えた。しかし SELinux・firewalld と Docker の噛み合わせ、Docker の公式サポートの外であることを考え、慣れを外して中立に比べると Ubuntu の方が見通しがよいので戻した（3）。
9. **mail のデータ：ブロックボリューム → ブートボリューム**：ブロックボリュームは最小 50GB で、ディスクの枠の残りが 9GB になる。メールにそこまで要らず、あとから移すのも難しくないので、ブートボリュームの `/srv/mail` に置き、枠を 59GB 残した（3）。
10. **自動更新**：コンテナが勝手に止まるのを避けたい。自動はセキュリティの更新だけにし、再起動と Docker 本体の更新は手で流す1コマンド（`upgrade.yml`）にした（11）。
11. **作業用の VM**：既存のマイクロ2台（2023年に勉強で作ったもの）は、ブートボリュームのバックアップを取ってから消す。残すものを調べる手間を、必要になったときに先送りできる。作業用の VM は、新しい VCN に作り直す（ops）。
12. **Cloudflare → LB の暗号化：Full (strict)**：個人サイトなので、HTTP（Flexible）や自己署名（Full）でも許せると考えた。しかし Full (strict) にしても手間はほぼ変わらず、Workers・Pages への影響もない。Full では、IP の間違いや、手放した IP が他人に渡ったときにも、黙ってそこにつないでしまう。そこで strict にした。プロキシをやめて Let's Encrypt を LB に置く案は、更新の仕組みを自分で持つ手間と、LB を全世界に公開することから見送った（5）。
13. **Cloudflare の設定の持ち方：Terraform で全部 → ゾーンの設定は手、レコードはアプリの Terraform**：最初は Origin 証明書の発行も DNS も Terraform で持つつもりだった。しかし DNS の大部分は手で作ってきており、②では wrangler もレコードを作るので、持ち主が3つに分かれて見通しが悪い。ゾーンとアカウントの設定（証明書を含む）は手にし、Terraform は自分のレコードだけを `cloudflare.tf` で持つ形にした。ワイルドカードと `cf` も比べた（7）。
14. **イメージの運び方：OCIR → app VM の上でビルド**：手元（x86）と app（arm64）で CPU が違い、OCIR を使っても手元で確かめたイメージそのものは運べない。レジストリの認証トークンを増やさず、0円が確実な、app の上でのビルドにした（8）。
