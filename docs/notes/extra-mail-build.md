# 番外編：メールサーバーを組む

最終更新：2026-11-05

[学習ロードマップ：OCI](../roadmaps/oci.md) の Step 8 で、mail VM にメールサーバーを組みながら分かったことの記録。回（8-2、8-3 …）ごとに追記していく。仕組みの基礎は [番外編：メールの仕組み](extra-mail-basics.md)、決めたことは [方式検討：OCI のメール基盤](../design/oci-mail.md)。

## 8-2. 受信：Postfix

### Postfix の役割

8-2 の時点の Postfix は「受け取る側の MTA」で、メールボックスへの保存も兼ねていた（8-3 で保存は Dovecot に移した）。閲覧（IMAP）は Dovecot、迷惑メールの判定は Rspamd の担当で、Postfix は持たない。8-4 で、送信の受付（MSA）と外への中継が加わる。

Postfix は1つのプログラムではなく、役割ごとの小さなデーモンの集まり。

```
Internet --25--> smtpd --> cleanup --> queue (qmgr) --+--> virtual : write Maildir   (8-2)
                 accept/reject                        +--> lmtp    : hand to Dovecot (8-3-)
                 per recipient                        +--> local   : Linux users (root@localhost)
                                                      +--> smtp    : other domains (8-4: via Email Delivery)
```

### アドレスクラス

理解の鍵は、**Postfix が宛先のドメインを「アドレスクラス」に分ける**こと。どのクラスに入るかで、受けるかどうかと、誰が配達するかが決まる。

| クラス | 設定 | 今の値 | 配達するもの |
| --- | --- | --- | --- |
| local | `mydestination` | `localhost` | `local`（Linux のユーザー） |
| virtual mailbox | `virtual_mailbox_domains` | `alphajinsei.com` | `virtual_transport`（8-2 は `virtual`、8-3 から Dovecot の LMTP） |
| それ以外 | — | gmail.com など | `smtp`（よそへ送る）。**中継を許された相手からしか受けない** |

「virtual」は、メールのユーザーを Linux のユーザーにしない方式。メールボックスは全員分を `vmail`（uid 5000）の1人が持つ。

### 期待する正常系の挙動と、対応する設定

| # | 挙動 | 設定 |
| --- | --- | --- |
| 1 | インターネットの誰からでも、25番で接続を受ける | `inet_interfaces = all`（と、NSG・iptables の 25番） |
| 2 | `alphajinsei.com` 宛てを、自分のドメイン宛てとして受ける | `virtual_mailbox_domains` |
| 3 | `alphajinsei@` 宛てを受ける | `virtual_mailbox_maps`（`vmailbox`：受けてよい宛先の一覧） |
| 4 | `postmaster@`・`abuse@`・`dmarc@` を `alphajinsei@` に届ける | `virtual_alias_maps`（`virtual`：別名 → 本当の宛先） |
| 5 | Maildir に、`vmail` の持ち物として書く | 8-2：`virtual_mailbox_base`、`virtual_uid_maps`・`virtual_gid_maps`。8-3 から Dovecot の `userdb` |
| 6 | 相手が対応していれば TLS で受ける | `smtpd_tls_security_level = may` と証明書 |
| 7 | `mail.alphajinsei.com` と名乗る（挨拶、`Received:` ヘッダー） | `myhostname`。差出人のドメインの補完は `myorigin` |
| 8 | VM の中のシステムのメール（cron のエラーなど）は、VM の中で処理する | `mydestination = localhost`、`alias_maps` |

### 禁止する挙動と、対応する設定

| # | 禁止すること | 設定 | 理由 |
| --- | --- | --- | --- |
| 1 | **知らない宛先**（`nobody@alphajinsei.com`）を受ける | `virtual_mailbox_maps` に無い宛先は、会話の途中で 550 を返す（`smtpd_reject_unlisted_recipient` は既定で yes） | 一度受けてから「届きませんでした」と送り返すと、偽の差出人に大量のエラーメールを送る踏み台になる（**バックスキャッター**）。受け取る前に断るのが作法 |
| 2 | **よそのドメイン宛ての中継**（オープンリレー） | `smtpd_relay_restrictions = permit_mynetworks, reject_unauth_destination`、`mynetworks = 127.0.0.0/8` | 自分自身以外から gmail.com 宛てを頼まれたら 554 で断る |
| 3 | 宛先が実在するかの問い合わせ（VRFY） | `disable_vrfy_command = yes` | 実在するアドレスを集める手口に使われる |
| 4 | 挨拶（HELO/EHLO）なしの送信 | `smtpd_helo_required = yes` | 雑な迷惑メールの送信プログラムを弾く |
| 5 | 巨大なメール | `message_size_limit`（25MB） | ディスクを守る |
| 6 | ソフトの名前と版を見せる | `smtpd_banner = $myhostname ESMTP` | 既定の挨拶は `(Ubuntu)` まで出す |

- 例外として、**自分自身（localhost）からは中継を許す**。8-4 でログインしたユーザー、8-6 でアプリを、この許可のリストに足していく。
- まだ防げていないもの：迷惑メールそのもの、外部から `From: 〜@alphajinsei.com` を名乗るなりすまし（8-5 で Rspamd が DMARC で判定）、ログインの総当たり（ログインの口ができたあと、8-5 で fail2ban）。

### swaks で手で SMTP を話す

ops は Postfix から見ると「よその人」（`mynetworks` の外）なので、外から来る相手の代わりに試せる。swaks は会話を全部表示する（`<~` が相手、`~>` が自分）。

| 試したこと | 返事 | どこで断られるか |
| --- | --- | --- |
| `--to alphajinsei@alphajinsei.com` | 250 | 受け取る |
| `--to nobody@alphajinsei.com` | 550 | `RCPT TO` の段階（知らない宛先） |
| `--to someone@gmail.com` | 554 Relay access denied | `RCPT TO` の段階（よそ宛ての中継） |

断るのは本文（`DATA`）を受け取る前。

### 届いたメールのヘッダーを読む

ヘッダーは、受け取ったサーバーが**上に足していく**。

| ヘッダー | 付けた人 | 意味 |
| --- | --- | --- |
| `Return-Path:` | 自分の Postfix | 封筒の差出人（`MAIL FROM`）を、届いたときに書き残したもの |
| `X-Original-To:` / `Delivered-To:` | 自分の Postfix | 封筒の宛先（`RCPT TO`）。別名宛てだと、ここに元の宛先が残る |
| `Received:`（上） | 自分の Postfix | Google から受け取った記録。`using TLSv1.3`：自己署名の証明書でも、Gmail は暗号化して送ってきた（送る側が検証しないため） |
| `Received:`（下） | Google | Google の中での受け渡し。`Received:` は下から上へ読む |
| `DKIM-Signature: d=gmail.com` | Gmail | Gmail の署名。`From:` と同じ gmail.com なので、DMARC のアライメントも取れる |
| `ARC-Seal` など | Gmail | 転送されても「受け取った時点では正しかった」と示す ARC |

`Authentication-Results:` がまだ無いのは、自分のサーバーが SPF・DKIM・DMARC を確かめていないから（8-5 で Rspamd が付ける）。

### つまずいたこと

- **iptables の NOFLUSH**：OCI の Ubuntu のイメージは `netfilter-persistent` を NOFLUSH（今のルールを消さずに後ろに足す）にしている。reload で古い REJECT が先に残り、Step 3 で開けたつもりの 25番などがずっと閉じていた（`No route to host`）。読み直しは `iptables-restore < /etc/iptables/rules.v4`（フラッシュしてから）にした。NSG で落とされると応答がなくタイムアウトになり、VM の中の REJECT だと `No route to host` になる。症状で、どこで止まっているかが分かる。
- **vmail のグループ**：system ユーザーを作るとき、グループは空いている別の番号で作られる。Postfix は `static:5000` で書くので、ファイルのグループが「名前のない 5000」になった。グループを gid 5000 で先に作るようにした。

## 8-3. 閲覧：Dovecot と証明書

### 役割の分担

```
MUA --993 (IMAPS, login)--> Dovecot --reads--> /srv/mail/<domain>/<user>/Maildir
Postfix --LMTP--> Dovecot --writes--> (same Maildir)
```

- Postfix は「受け取って、よいメールか判断して、Dovecot に渡す」まで（`virtual_transport = lmtp:unix:private/dovecot-lmtp`）。
- Dovecot は「Maildir に書く（LMTP）」と「メールソフトに見せる（IMAP）」と「ログインの確認」。書くのも Dovecot にすると、Dovecot が索引（既読、フォルダ）を自分で更新でき、あとで振り分けのルール（Sieve）も足せる。
- Postfix の `virtual_mailbox_maps` は「受けてよい宛先の一覧」としてだけ残る（保存先は Dovecot の `userdb` が決める）。

### Ansible（roles/mail）がやっていること

| 順 | やること | 目的 |
| --- | --- | --- |
| 1 | iptables に 80番（common） | Let's Encrypt が確かめに来られるように |
| 2 | oci CLI と `platform-secret`（oci_cli） | Vault からパスワードを読むため |
| 3 | `vmail` のグループとユーザー、`/srv/mail` | メールボックスの持ち主 |
| 4 | certbot を入れ、証明書を取る（初回だけ） | 下の ACME |
| 5 | 更新のあとに読み直すスクリプト（deploy フック）を置く | 下の自動更新 |
| 6 | Postfix の `main.cf`：証明書を Let's Encrypt に、保存を Dovecot に | 受け取りに専念させる |
| 7 | Dovecot を入れ、`dovecot.conf` を1つのファイルに全部書く | IMAP（993番のみ、TLS 必須）、LMTP、ログイン。パッケージの conf.d は使わず、Linux のユーザーでのログイン（PAM）も切る |
| 8 | Vault からパスワードを読み、SHA512-CRYPT のハッシュにして `/etc/dovecot/users` へ | ログインの照合。ソルトをユーザーごとに固定し、何度流してもファイルが変わらないようにした。平文はファイルにも Ansible の出力にも残さない |
| 9 | 変更があったものを読み直す（ハンドラー） | 新しい設定を効かせる |

パスワードの置き場は、DB と同じ「Terraform が `random_password` で作る → Vault → VM が自分の権限（インスタンス・プリンシパル）で取りに行く」形。この基盤のお決まりの形になった。

### 証明書は、誰の何を証明しているのか

**メールソフトに対して「今つないだ相手が本物の `mail.alphajinsei.com` だ」ということ**を証明する。

- IMAP では、ログインのたびにパスワードを送る。暗号化していても、相手が偽物なら、偽物にパスワードを渡してしまう（公衆の Wi-Fi で途中に割り込まれる場合など）。
- 証明書は「この公開鍵の持ち主は `mail.alphajinsei.com` だ」という、CA（Let's Encrypt）の署名付きの書類。メールソフトは、OS やスマホが信頼している CA の署名かどうかを確かめ、確かめられなければ警告を出して止まる。
- 自己署名でも暗号化はできるが、「本物か」が確かめられないので警告が毎回出る。サーバー同士の 25番の配送では、送る側が検証しないのが普通（8-2 で Gmail から届いたのはこのため）。
- 8-4 の送信（465・587番）でもパスワードを送るので、同じ証明書を使う。

### ACME：証明書の取得を自動化するプロトコル

ACME（RFC 8555）は、Let's Encrypt が作った、証明書の取得と更新を自動化するプロトコル。「この名前を持っているのは自分だ」という証明（チャレンジ）の仕方が複数ある。

| | 証明の仕方 | 外から来るもの |
| --- | --- | --- |
| HTTP-01 | `http://<名前>/.well-known/acme-challenge/<トークン>` に答えを置く | Let's Encrypt が 80番に取りに来る（こちらが受ける） |
| DNS-01 | `_acme-challenge.<名前>` の TXT レコードに答えを書く | なし。certbot が DNS の API に出ていって書き、Let's Encrypt は DNS を引くだけ |

- DNS-01 は「外から入られる穴」ではない。怖いのは、**DNS を書き換えられるトークンを、外に公開しているサーバーに置くこと**（乗っ取られるとトークンも盗まれ、ゾーン全体を書き換えられる）。外から入られる口があるのは、むしろ HTTP-01 の方（ふだんは何も聞いていない 80番）。
- DNS-01 が要るのは、HTTP-01 では取れない場面：**ワイルドカードの証明書**（DNS-01 でしか取れない）、**外から届かないサーバー**（社内、80番を開けられない環境）、**多数のサーバーの証明書を1か所でまとめて取る**とき（LB の後ろに何台もあると、HTTP-01 の確認がどのサーバーに届くか分からない）。
- その場合も、トークンを公開するサーバーには置かない。取得専用の場所に置いて証明書を配るか、`_acme-challenge` の名前だけを別のゾーンに CNAME で任せ、そのゾーンだけを書けるトークンにして被害を絞るのが定石。
- 今回は HTTP-01 にした。

### 初回の取得で起きていること

`certbot certonly --standalone -d mail.alphajinsei.com`：

```
certbot (mail VM)                                   Let's Encrypt
 1. create account key, register  ----------------> account created
 2. "I want a cert for mail.alphajinsei.com" -----> order
                                   <---------------- challenge: put token T at
                                                     http://mail.alphajinsei.com/.well-known/acme-challenge/T
 3. start a tiny web server on :80 (standalone),
    serve T + fingerprint of account key
 4. "ready" ----------------------------------------> 5. resolve mail.alphajinsei.com (DNS -> mail VM IP)
                                                        GET http://.../acme-challenge/T  (from several places)
                                   <---------------- matches -> "you control this name"
 6. stop the web server on :80
 7. make a new key pair for the cert, send CSR ----> sign it (90 days)
    (CSR = public key + name, signed by the key)
                                   <---------------- certificate + intermediate
 8. save under /etc/letsencrypt/
```

| 登場人物 | 役割 |
| --- | --- |
| certbot | mail VM の上の ACME クライアント |
| Let's Encrypt | ACME サーバー（CA） |
| アカウントの鍵 | certbot が初回に作る鍵ペア。以降の Let's Encrypt へのお願いはすべてこれで署名する（「前回と同じ人です」の証明） |
| 証明書の鍵 | `mail.alphajinsei.com` の秘密鍵。mail VM の中で作り、外へは公開鍵（CSR）しか出さない。Let's Encrypt も知らない |

- **5番が「名前を持っている」ことの証明。**Let's Encrypt が DNS を引き、その IP の 80番に取りに来る。答えを返せるのは、DNS がその名前を向けている先だけ。だから DNS（A レコード）と 80番（NSG と iptables）の両方が要る。
- **`--standalone` は、certbot が自分で一時的な Web サーバーを立てる方式。**Nginx などがなくても答えられる。数秒だけ 80番で待ち受け、終わったら閉じる。

### 保存されるもの

```
/etc/letsencrypt/
├── accounts/                     account key (who you are to Let's Encrypt)
├── archive/mail.alphajinsei.com/ real files, numbered: cert1.pem, privkey1.pem, ... cert2.pem ...
├── live/mail.alphajinsei.com/    symlinks to the newest numbers
│   ├── fullchain.pem -> ../../archive/.../fullchain2.pem
│   └── privkey.pem   -> ../../archive/.../privkey2.pem
└── renewal/mail.alphajinsei.com.conf   how it was obtained (standalone, domain, account)
```

Postfix と Dovecot の設定には `live/` のパスだけを書く。更新では数字が増えてリンクの向き先が変わるだけで、設定ファイルは書き換えない。

### 自動更新の仕組み

```
systemd timer (certbot.timer)  -- twice a day, random delay -->  certbot renew
                                                                    |
     for each file in /etc/letsencrypt/renewal/*.conf:              |
       expires in more than 30 days?  -> do nothing                 |
       expires within 30 days?        -> repeat steps 2-8 above,    |
                                         same method (standalone)   |
                                         new key, new cert -> live/ symlinks move
                                         then run renewal-hooks/deploy/*  --> reload-mail.sh
                                                                               systemctl reload postfix dovecot
```

| 部品 | 誰が用意したか | 役割 |
| --- | --- | --- |
| `certbot.timer` | certbot のパッケージ（apt で入れた時点で有効） | 1日2回 `certbot renew` を起こす。cron の systemd 版。`systemctl list-timers` で次の起動時刻が見える |
| `renewal/*.conf` | 初回の certbot | 「どの方式で、どの名前を取ったか」の記録。renew はこれを見て、初回と同じ手順を繰り返す |
| deploy フック | Ansible（`reload-mail.sh`） | 実際に更新されたときだけ動き、Postfix と Dovecot を読み直させる |

- **1日2回も起こすのは、空振りが前提だから。**ふだんは何もしない。期限の30日前から取り直し、失敗しても（Let's Encrypt の障害、80番の閉め忘れなど）30日のあいだ1日2回やり直すので、期限切れまでに気付いて直す余裕がある。
- **deploy フックが要るのは、Postfix と Dovecot が証明書を起動時に読み込んでメモリに持つから。**読み直させないと古い証明書を出し続ける（「更新は成功していたのに、期限切れの証明書を出していた」という、よくある事故）。
- `certbot renew --dry-run` は、この流れを Let's Encrypt の試験用の環境で試す。本物の証明書は替わらない。

### なぜ今「自動更新」なのか

- 公的な証明書の有効期限は短くなる一方。Let's Encrypt は最初から90日で、業界のルール（CA/B Forum）でも2029年に47日まで縮む。Let's Encrypt もさらに短くする計画を出している。
- 期限が短いほど、鍵が漏れたときの被害の期間は短くなるが、人が手で更新するのは無理になる。「人は最初に仕組みを作るだけで、更新は機械が回す」前提になり、その標準のプロトコルが ACME。
- Cloudflare の Origin 証明書（15年、手で発行。Step 5）は、公的な CA のルールの外にあるので、例外として長く持てている。

## 8-4. 送信：465・587番と Email Delivery への中継

### SPF・DKIM・DMARC は、それぞれ何を証明するのか

| 仕組み | 証明すること | 確かめる相手 | 証明しないこと |
| --- | --- | --- | --- |
| SPF | 「この IP のサーバーは、このドメインの名で送ってよいと、ドメインの持ち主が認めている」 | 封筒の差出人のドメイン（と送ってきた IP） | `From:`、本文。転送されると IP が変わって落ちる |
| DKIM | 「このメール（本文と主なヘッダー）は、ドメイン `d=` の鍵を持つ者が署名し、途中で書き換えられていない」 | 署名の `d=` のドメイン | `From:`（`d=` は誰のドメインでもよい）。送った経路 |
| DMARC | 「SPF か DKIM で確かめられたドメインが、人が見る `From:` と一致する」＋持ち主の方針（none / quarantine / reject）と報告先 | `From:` のドメイン | ―（SPF と DKIM の結果を `From:` に結び付ける役） |

- SPF は「経路（どのサーバーから来たか）」、DKIM は「中身（誰が責任を持ち、改ざんされていないか）」を証明する。
- どちらも `From:` を縛らない。迷惑メールの送り手が自分のドメインで SPF・DKIM を通し、`From:` だけ銀行を名乗ることができる。その穴を埋めるのが DMARC で、だから「アライメント」が中心になる。
- 実装はこの順に読める：SPF は「許す IP の一覧」を DNS に置く、DKIM は「公開鍵」を DNS に置いて送る側が秘密鍵で署名する、DMARC は「一致しなかったときの扱い」を DNS に置く。

### DNS に置いたレコードの意味

```
alphajinsei.com.                             TXT   "v=spf1 include:ap.rp.oracleemaildelivery.com -all"
oci-nrt-20261105._domainkey.alphajinsei.com. CNAME (public key TXT on the Email Delivery side)
bounce.alphajinsei.com.                      CNAME (SPF and MX for bounces on the Email Delivery side)
_dmarc.alphajinsei.com.                      TXT   "v=DMARC1; p=none; rua=mailto:dmarc@alphajinsei.com"
```

SPF：

| 部分 | 意味 |
| --- | --- |
| `v=spf1` | SPF のレコードの印（同じ名前の TXT が何本あっても、これで見分ける） |
| `include:ap.rp.oracleemaildelivery.com` | Oracle が公開しているアジア太平洋の送信サーバーの一覧（その名前の SPF）を取り込む。Oracle がサーバーを増やしても、こちらは変えなくてよい |
| `-all` | それ以外の IP は fail（はっきり違反）。`~all` だと softfail（怪しいが受け取ってよい） |

- mail VM の IP は書かない（外へは直接送らず、全部 Email Delivery を通す）。
- カスタム Return-Path が効いていれば、SPF で確かめられるのは `bounce.alphajinsei.com`（CNAME の先の Oracle 側の SPF）。ルートのこのレコードは、封筒の差出人が `@alphajinsei.com` のまま届いたときの保険と、「ほかの誰もこのドメインで送れない」という宣言。

DKIM：本来は `<セレクター>._domainkey.<domain>` に公開鍵そのものを TXT で置く。今回は CNAME で Oracle 側の TXT を指し、秘密鍵は Oracle が持つ（Oracle が鍵を替えても DNS はそのまま）。

DMARC：

| 部分 | 意味 |
| --- | --- |
| `v=DMARC1` | DMARC のレコードの印（置き場所も `_dmarc.` と決まっている） |
| `p=none` | 不合格でも何もしない（様子見）。`quarantine` は迷惑メールへ、`reject` は受け取り拒否 |
| `rua=mailto:dmarc@...` | 集計レポートの送り先。Gmail などが1日ごとに、どの IP から何通来て SPF・DKIM がどうだったかを XML で送ってくる（`dmarc@` は alphajinsei への別名） |

書いていないものは既定値：`adkim` と `aspf` は relaxed（親ドメインが同じなら一致）、`pct=100`、`sp`（サブドメインの方針）は `p` と同じ。レポートで正しいメールが全部 pass しているのを見てから、8-5 で `quarantine` に上げる。

### 465 と 587：違うのは暗号化を始めるタイミングだけ

| ポート | 暗号化の始め方 | 経緯 |
| --- | --- | --- |
| 587（submission） | 平文でつないでから `STARTTLS` で TLS に切り替える | 1998年に「メールソフトからの投稿用」として 25番から分けられた（RFC 6409） |
| 465（submissions） | つないだ瞬間から TLS（HTTPS と同じ形） | 一度は廃止扱いになったが、2018年の RFC 8314 で正式に戻り、今はこちらが推奨 |

- STARTTLS は、途中の誰かが `STARTTLS` の案内を消して平文のまま話させる、といった攻撃の隙がありうる。
- どちらかにしか対応しないソフトや機器があるので、両方開けるのが普通。587番も `smtpd_tls_auth_only=yes` で TLS 前のログインを受けないので、安全さは同じ。Outlook には 465 を使う。
- 25番はサーバー同士の受け渡し用で、ログインはしない。

### 送信の流れと、3つの差出人の移り変わり

```
Outlook --465 (TLS, login as alphajinsei@alphajinsei.com)--> Postfix (submissions)
   envelope from: alphajinsei@alphajinsei.com            |  login checked by Dovecot (SASL)
   From:          alphajinsei@alphajinsei.com            |  sender must match the login (sender_login)
                                                         v
                     Postfix (smtp client) --587 (TLS, cert verified, login as IAM user mail-relay)-->
                                                         Email Delivery
                                                         |  From: must be an approved sender
                                                         |  envelope from -> bounce.alphajinsei.com
                                                         |  DKIM sign: d=alphajinsei.com, s=oci-nrt-20261105
                                                         v
                                                       Gmail MX
```

| 差出人 | Outlook → Postfix | Email Delivery → Gmail | Gmail の判定 |
| --- | --- | --- | --- |
| ① 封筒の差出人 | `alphajinsei@alphajinsei.com` | **`...@bounce.alphajinsei.com` に書き換え**（カスタム Return-Path） | SPF：`bounce.alphajinsei.com` の SPF（CNAME で Email Delivery 側）に送信元の IP が載っているか |
| ② 署名のドメイン | なし | `d=alphajinsei.com`（Email Delivery が署名） | DKIM：`oci-nrt-20261105._domainkey.alphajinsei.com`（CNAME で Email Delivery 側の公開鍵）で検証 |
| ③ `From:` | `alphajinsei@alphajinsei.com` | そのまま | DMARC：① は親ドメインが同じ（relaxed）、② は同じ。どちらでもアライメントが取れる |

- カスタム Return-Path を置かないと、① は Oracle のドメインになり、SPF は通っても DMARC に効かない（業務でよくある「SPF は pass なのに DMARC は fail」の形）。
- `alphajinsei.com` そのものの SPF（`include:ap.rp.oracleemaildelivery.com -all`）は、① が `@alphajinsei.com` のまま届く場合の保険。mail VM の IP からは外へ送らないので載せない。

### 2つのログイン（SASL）

| | Postfix の立場 | 相手 | 確かめる人 | 設定 |
| --- | --- | --- | --- | --- |
| メールソフトからの送信 | サーバー | Outlook | Dovecot（`private/auth` の口。IMAP と同じユーザーとパスワード） | `smtpd_sasl_*`、master.cf の `-o smtpd_sasl_auth_enable=yes` |
| Email Delivery への中継 | クライアント | Email Delivery | Email Delivery（IAM のユーザー `mail-relay` の SMTP の資格情報） | `relayhost`、`smtp_sasl_*`、`/etc/postfix/sasl_passwd` |

`smtpd_` は受ける側、`smtp_` は送る側の設定、という命名の規則で見分ける。

### master.cf：口ごとに規則を変える

`main.cf` は全体の既定、`master.cf` はデーモン（口）の一覧で、`-o` でその口だけ設定を上書きできる。

| 口 | ログイン | 中継の許可 | 用途 |
| --- | --- | --- | --- |
| 25番（smtp） | させない | 自分のドメイン宛てだけ（`reject_unauth_destination`） | よそのサーバーからの配送 |
| 587番（submission） | STARTTLS のあと必須（`smtpd_tls_auth_only`） | ログインした人だけ（`permit_sasl_authenticated,reject`） | メールソフトからの送信 |
| 465番（submissions） | 最初から TLS（`smtpd_tls_wrappermode`）、必須 | 同上 | 同上 |

- 25番でログインを受けないのは、ログインの総当たりの口を増やさないため。メールソフトは 465・587番だけを使う。
- `reject_sender_login_mismatch`：ログインした人が、`sender_login` にない封筒の差出人を名乗るのを拒否する（他人を名乗らせない）。`From:` は Postfix では見ず、Email Delivery の承認済みの差出人が見る。
- chroot（デーモンを `/var/spool/postfix` に閉じ込める）は使わない（n）。Ubuntu の既定は y だが、ファイルの置き場が分かりにくくなるので切った。

### 中継先の証明書も確かめる

`smtp_tls_security_level = secure`：Email Delivery の証明書が、`smtp.email.ap-tokyo-1.oci.oraclecloud.com` に対して公的な CA から出たものかを確かめてからパスワードを渡す。8-3 の「証明書は、つないだ相手が本物かを確かめるもの」の、Postfix がクライアントになる側。`may`（相手が対応していれば暗号化）や `encrypt`（暗号化は必須、相手は確かめない）より強い。

### Email Delivery を Terraform で持つ

| リソース | 役割 |
| --- | --- |
| `oci_email_email_domain` | 送信するドメイン（`alphajinsei.com`）。DKIM と Return-Path がぶら下がる |
| `oci_email_dkim` | DKIM の鍵（Email Delivery が作って持つ）。出力の CNAME を Cloudflare に置く |
| `oci_email_email_return_path` | カスタム Return-Path（`bounce.alphajinsei.com`）。出力の CNAME を Cloudflare に置く |
| `oci_email_sender` | 承認済みの差出人（`From:` に使ってよいアドレス） |
| `oci_identity_user` / `group` / `policy` | 中継専用の IAM のユーザー（Postfix が Email Delivery の SMTP にログインするときの身元）。権限は `use approved-senders` だけ |
| `oci_identity_smtp_credential` | SMTP のユーザー名とパスワード。パスワードは作ったときしか返らないので Vault へ |

DKIM の鍵は自分で作って渡すこともできる（`oci_email_dkim` の `private_key`）。今回は Email Delivery に作らせた。

### 届かなくなるもの

Postfix 自身が作るエラーのメール（`MAILER-DAEMON@alphajinsei.com` から）は承認済みの差出人ではないので、Email Delivery が受け取らない。自分が送ったメールが届かなかったときの知らせは、Email Delivery が `bounce.alphajinsei.com` で受けて処理する（コンソールの抑止リスト）。

### つまずいたこと

- **Return-Path が作れない**（`should have an active DKIM or Domain Verification should be completed`）：Return-Path を作るには、DKIM が Active でなければならない。Active になるのは、Cloudflare に CNAME ができ、Oracle がそれを DNS で見て確かめたあと。Terraform が知っている順番は参照と `depends_on` だけで、「Oracle が DNS を確かめ終わる」という外の出来事は待てない。`depends_on = [cloudflare_dns_record.dkim]` で「CNAME を作ってから」までは保証できるが、確認が済むまでは保証できないので、一から作るときは apply が2回になる。待つ処理（`time_sleep`）は、DNS の確認にかかる時間が読めないので入れなかった。
- **IAM のユーザーが作れない**（`primaryEmailNotSpecified`）：テナンシーが Identity Domains の形で、ユーザーは人のアカウントとして扱われ、主メールアドレスが必須。昔の IAM の形では要らなかった。コンソールにログインしない「サービス・ユーザー」もあるが、別のリソース（`oci_identity_domains_user`）で SCIM の属性を書くことになり重いので、普通のユーザーに `postmaster@alphajinsei.com` を付けた。
- そもそも IAM のユーザーが要る理由：Email Delivery の SMTP の口は、IAM のユーザーに発行した SMTP の資格情報（ユーザー名とパスワード）でしかログインできない。VM のインスタンス・プリンシパル（Vault を読むときの身元）は SMTP では使えない。
- **Outlook から送れない**（`Server 'mail.alphajinsei.com' Port '25' ... 503 5.5.1 Error: authentication not enabled`）：Outlook の送信のポートが既定の 25 のままだった。25番はサーバー同士の口でログインを受けない設計なので、拒否されたのは正しい挙動（「25番でログインさせない」が効いている証拠）。送信を 465・SSL/TLS に直す。

### bounce.alphajinsei.com の CNAME：バウンスを受けるのは Oracle

送り先が存在しないなどで届けられないとき、受け取る側（Gmail など）は、エラーのメール（バウンス）を**封筒の差出人**に送り返す。封筒の差出人は Email Delivery が `...@bounce.alphajinsei.com` に書き換えているので、バウンスは `bounce.alphajinsei.com` の MX に向かう。

```
Gmail --bounce--> MX of bounce.alphajinsei.com
                  = (CNAME) -> Email Delivery side name -> Oracle's bounce processor
                                                           -> suppression list (stop sending to that address)
```

- CNAME は「この名前の情報は全部あちらを見て」という意味なので、`bounce.alphajinsei.com` の MX（バウンスの受け口）も TXT（SPF）も、Oracle 側のものになる。こちらは CNAME 1本で済み、Oracle が中身を変えても追従する。
- oci-mail で受けないのは、バウンスを知る必要があるのが Email Delivery だから。届かない宛先に送り続けると送信元の評判が下がるので、Email Delivery はバウンスを受けて抑止リスト（コンソールの Suppressions）に入れ、以後その宛先には送らない。書き換えた封筒の差出人には、どの送信のバウンスかを見分ける印が入っている。
- 確かめ方：`dig +short MX bounce.alphajinsei.com`（Oracle のサーバーが出る）、`dig +short TXT bounce.alphajinsei.com`（Oracle の SPF が出る）。

### postconf で、効いている設定を見る

- `postconf -M`：master.cf の口の一覧（Ansible が置いたものが読まれているか）。列は、サービス名、種類（`inet` は TCP で待ち受け）、private、unpriv、chroot、wakeup、maxproc、コマンドと `-o` の上書き。`smtp`（25）、`submission`（587）、`submissions`（465）の順に並び、587・465 だけに `smtpd_sasl_auth_enable=yes` と `permit_sasl_authenticated,reject` が付いていれば、口ごとの規則が効いている。chroot の列は n。
- `postconf relayhost`：main.cf のその設定の今の値。`postconf -n` は既定から変えた設定の全部、`postconf -d` は既定値。ファイルを読むより確かで、書き間違えた設定名は警告が出る。

