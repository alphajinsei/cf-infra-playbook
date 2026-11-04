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
