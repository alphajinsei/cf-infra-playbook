# 番外編：メールの流れを図にして分かったこと

最終更新：2026-11-07

[方式検討：OCI のメール基盤](../design/oci-mail.md) の構成図（[oci-mail.drawio.svg](../design/diagrams/oci-mail.drawio.svg)）を描き直す中で、Postfix の中身と、メールのプロトコル・ポート・認証を整理し直した記録。図には最終的に Postfix を1つの箱として描いたが、そこに至るまでに分かったことを残す。基礎は [番外編：メールの仕組み](extra-mail-basics.md)、組み立ての記録は [番外編：メールサーバーを組む](extra-mail-build.md)。

## 1. Postfix はルーターに近い

### 主なデーモン

Postfix は1つのプログラムではなく、役割ごとの小さなデーモンの集まり。master.cf の1行が1つのデーモンで、`master` が起動して見張る。メールは「受け取る → キュー → 配る」の3段を通る。

| 段 | デーモン | やること | この基盤での使われ方 |
| --- | --- | --- | --- |
| 受け取る | smtpd | SMTP のサーバー。つないできた相手と話し、受け取るか断るかを決める。パスワード認証（SMTP AUTH）と milter もここ | 口は3つで、master.cf の `smtp`（:25）・`submission`（:587）・`submissions`（:465）。プログラムは同じ smtpd で、`-o` で規則を変えている。パスワードの照合は Dovecot に、迷惑メールの判定は Rspamd に頼む |
| | cleanup | ヘッダーを整え、宛先を書き換え（`virtual_alias_maps`）、キューに書く | smtpd から渡される |
| キュー | qmgr | キューを見て、宛先ごとに配送のデーモンを呼ぶ。失敗したら後で再送 | 宛先から配り方を決めるのは trivial-rewrite（自分のドメイン → `virtual_transport`、それ以外 → `relayhost`） |
| 配る | lmtp | LMTP のクライアント | 自分宛てを Dovecot に渡す（`virtual_transport = lmtp:unix:private/dovecot-lmtp`） |
| | smtp | SMTP のクライアント | 外宛てを Email Delivery の :587 に中継する |
| 補助 | bounce・anvil・tlsmgr・scache など | バウンスの通知、接続数の制限、TLS のセッションのキャッシュ | 意識しなくてよい |

```
Outlook  --465--+                                +--> lmtp --> Dovecot   (own domain)
app      --25---+--> smtpd --> cleanup --> qmgr --+
Internet --25---+                                +--> smtp --587--> Email Delivery   (other domains)
```

- 受け取る口は smtpd だけで、3つのポートはどれも smtpd。
- Postfix が外へ :25 で送ることはない。外宛ては全部、smtp が Email Delivery の :587 に渡す（OCI は外向きの :25 を塞いでいる）。外の :25 に送るのは Email Delivery。
- Dovecot との線は2本あり、出どころが違う。パスワードの照合は smtpd から、メールの受け渡し（LMTP）は lmtp から。

### 送信も受信も、同じ形で処理する

Postfix は、受け取ったメールを「自分のユーザーが送ったものか、外から届いたものか」で分けて扱わない。全部いったんキューに入れ、qmgr が宛先のドメインだけを見て行き先を決める。

| ネットワーク | Postfix |
| --- | --- |
| ルート表（宛先のネットワーク → 次の行き先） | 宛先のドメイン → 配り方（`virtual_transport`、`transport_maps`） |
| 自分のネットワーク宛て（直接つながっている） | 自分のドメイン（`virtual_mailbox_domains`）→ lmtp で Dovecot へ |
| デフォルトゲートウェイ | `relayhost`（Email Delivery の :587） |
| ゲートウェイなしで直接届ける | `relayhost` を置かない場合。宛先の MX を DNS で引き、その :25 に直接送る |
| 転送する前のフィルタ（ACL） | `smtpd_relay_restrictions`。外宛て（中継）を受けるのは、SMTP AUTH が通った相手か `mynetworks` の相手だけ |

- 最後の行だけは、受け取る時点で宛先を見ている。だれでも外宛てを受け付けると、他人の迷惑メールを運ぶオープンリレーになるため。
- ルーターも、パケットが「自分の利用者が出したものか」を意識せず、宛先で行き先を決める。考え方が同じ。

## 2. ポートと暗号化は別の軸

ポートは「何の用途の受け口か」、暗号化は「TLS をどう始めるか」で、組み合わせられる。

| ポート | 用途 | TLS の始め方 | クライアントの認証 |
| --- | --- | --- | --- |
| 25 | サーバー同士の受け渡し（MTA → MTA） | 平文で始め、両方が対応していれば STARTTLS で切り替える（任意） | しない |
| 587 | メールソフトからの投稿（MSA。RFC 6409） | 平文で始め、STARTTLS で切り替える（実質必須） | パスワード（SMTP AUTH） |
| 465 | メールソフトからの投稿（MSA。RFC 8314） | つないだ瞬間から TLS | パスワード（SMTP AUTH） |

- **STARTTLS は 587 専用ではない。**どのポートの SMTP でも使える。今のサーバー同士の SMTP は、ほとんどが「:25 で平文で始めて STARTTLS に切り替える」形で暗号化されている。「25＝平文、587＝STARTTLS、465＝TLS」という1対1の覚え方は正しくない。:25 は TLS が任意、:587 は必須、という違い。
- プロトコルはどれも SMTP。`submission`（587）・`submissions`（465）はプロトコルの名前ではなく、役割（投稿）の名前で、/etc/services や master.cf のサービス名。
- HTTP と HTTPS がポートと1対1に見えるのは、HTTPS が :465 と同じ「つないだ瞬間から TLS」の形だけを使っているから。HTTP にも STARTTLS にあたる仕組み（RFC 2817 の Upgrade）はあるが、ほとんど使われていない。

## 3. 認証は3種類ある

「認証」と言うとき、別々のものが混ざりやすい。

| | 確かめること | 確かめる人 | 手段 |
| --- | --- | --- | --- |
| ① サーバーの認証 | つないだ先が本物の `mail.alphajinsei.com` か | クライアント | TLS の証明書（Let's Encrypt） |
| ② クライアントの認証 | つないできた相手がだれか | サーバー | パスワード（SMTP AUTH、IMAP のログイン）、送信元の IP、何もしない |
| ③ 差出人のドメインの認証 | 届いたメールが、名乗っているドメインから本当に来たか | 受け取る側 | SPF・DKIM・DMARC（DNS を引く） |

- ① と ② は「つなぎ方」の話で、③ は「メールそのもの」の話。
- 暗号化（TLS）と ① は同じ TLS の中で起きるが、② は TLS とは別の話。TLS は「途中で読まれない」「相手のサーバーが本物」までを保証し、「つないできたのがだれか」は ② で別に確かめる。
- 「ログイン」と呼んでいたのは ② のうちの **SMTP AUTH**（RFC 4954）。SMTP の中で SASL の仕組みを使い、ユーザー名とパスワードを送る。方式は PLAIN・LOGIN で、どちらもパスワードを平文で送るので、TLS の中でだけ許す（`smtpd_tls_auth_only=yes`、465 は最初から TLS）。

## 4. 経路ごとの整理

| 経路 | プロトコル :ポート | 暗号化 | ① サーバーの認証 | ② クライアントの認証 |
| --- | --- | --- | --- | --- |
| Outlook → Postfix | SMTP :465 | TLS（つないだ瞬間から） | Outlook が証明書を確かめる | SMTP AUTH（パスワード）。照合は Dovecot（passwd-file の SHA512-CRYPT のハッシュと比べる） |
| アプリ → Postfix | SMTP :25 | STARTTLS（アプリが切り替える） | アプリが証明書を `mail.alphajinsei.com` として確かめる | なし。送信元の IP（`mynetworks` の 10.0.10.0/24）で許す |
| よそのサーバー → Postfix | SMTP :25 | STARTTLS（相手が対応していれば。Gmail は使う） | ふつうは確かめない（任意の TLS では、送る側は証明書を検証しないことが多い） | なし。そのかわり自分宛てしか受けない。差出人は ③ を Rspamd が判定する |
| Postfix → Dovecot（パスワードの照合） | Dovecot の認証の口 | なし（UNIX ソケット、VM の中） | なし | ソケットのファイルの権限 |
| Postfix → Dovecot（受け渡し） | LMTP | なし（UNIX ソケット、VM の中） | なし | ソケットのファイルの権限 |
| Postfix → Email Delivery | SMTP :587 | STARTTLS（必須。`smtp_tls_security_level = secure`） | Postfix が証明書を確かめる | SMTP AUTH（IAM ユーザー `mail-relay` の SMTP の資格情報） |
| Email Delivery → よそのサーバー | SMTP :25 | STARTTLS（相手が対応していれば。未確認） | Oracle の運用しだい | なし。受け取る側は ③ で判定する |
| Outlook → Dovecot | IMAP :993 | TLS（つないだ瞬間から） | Outlook が証明書を確かめる | IMAP のログイン（パスワード） |
| Let's Encrypt → certbot | HTTP :80 | なし | なし | なし（ACME のトークンを置いて、ドメインの持ち主であることを示す） |

構成図の線のラベルは、この表から「プロトコル :ポート（暗号化・クライアントの認証）」の形で書いている。

## 5. 図に描かなかったこと

- Postfix の中（smtpd・qmgr・lmtp・smtp）を描いた版も作った。送信と受信がどちらも一度キューに降りてから左右に分かれ、ルーターのように振る舞うことは見えたが、送信の段・受信の段という見通しが崩れた。構成図は全体をつかむためのものなので、Postfix は1つの箱にし、中身はこの番外編に残した。
- Postfix の箱に受け付けるポートを並べるのもやめた。ほかの箱は役割しか書いておらず粒度が合わないうえ、ポートと暗号化の組み合わせ（2 の節）は線のラベルで足りる。
