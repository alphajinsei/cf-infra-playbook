# 番外編：メールの仕組み

最終更新：2026-11-05

[学習ロードマップ：OCI](../roadmaps/oci.md) の Step 8（メールサーバー）の教材。1通のメールが届くまでの登場人物から始めて、SPF・DKIM・DMARC が「何を確かめているのか」の全体像までを整理する。決めたこと（構成や方式）は [方式検討：OCI のメール基盤](../design/oci-mail.md)。

## 1. 登場人物

| 役割 | 略称 | やること | 代表的なソフト |
| --- | --- | --- | --- |
| メールソフト | MUA（Mail User Agent） | 書く・読む | Thunderbird、iPhone のメール、Gmail のアプリ |
| 送信の受付 | MSA（Mail Submission Agent） | 自分のユーザーからの送信を、認証してから受け付ける | Postfix（587・465番） |
| 配送 | MTA（Mail Transfer Agent） | ドメインの間でメールを運ぶ。宛先のドメインの MX を DNS で引き、そこへ渡す | Postfix（25番） |
| 保存 | MDA（Mail Delivery Agent） | 受け取ったメールをメールボックスに書き込む | Dovecot（LMTP） |
| 読み出し | IMAP サーバー | メールボックスをメールソフトに見せる。メールはサーバーに残り、既読やフォルダも共有される | Dovecot（993番） |

MSA と MTA は、たいてい同じソフト（Postfix）が、ポートを変えて兼ねている。

## 2. 1通のメールの旅

`alphajinsei@alphajinsei.com` から `someone@gmail.com` に送り、返事が来るまで。

```
Sending
  MUA --(587 or 465, login)--> MSA/MTA of alphajinsei.com
                                  |  1. DNS: MX of gmail.com? -> gmail-smtp-in.l.google.com
                                  |  2. connect to it on port 25, hand over the mail
                                  v
                               MTA of gmail.com --> mailbox of someone

Receiving (the reply)
  MTA of gmail.com
     |  1. DNS: MX of alphajinsei.com? -> mail.alphajinsei.com
     |  2. DNS: A of mail.alphajinsei.com? -> public IP of the mail VM
     |  3. connect on port 25, hand over the mail
     v
  MTA (Postfix) --> MDA (Dovecot) --> /srv/mail/... --(993, login)--> MUA
```

- **MX レコード**は「このドメイン宛てのメールは、このサーバーへ」という DNS のレコード。Web の A レコードとは別に持つので、Web は Cloudflare、メールは自前のサーバー、のように分けられる。
- 受け取る側のサーバーが止まっていても、送る側の MTA はメールを自分のキュー（送信待ちの列）に置き、数日間（Postfix の既定は5日）再送を続ける。メールは、Web より停止に寛容。
- **サーバー同士の配送は 25番、メールソフトからの送信は 587・465番。**ポートが分かれているのは役割が違うから。25番は「よそのドメインから自分のドメイン宛てのメールを、認証なしで受ける」口。587・465番は「自分のユーザーが、ログインしてから、どこ宛てにでも送る」口。

## 3. 封筒と便箋

メールには、郵便と同じく**封筒（envelope）**と**便箋（header と本文）**がある。SMTP の会話を見ると分かる。

```
S: 220 gmail-smtp-in.l.google.com ESMTP
C: EHLO mail.alphajinsei.com
S: 250-STARTTLS ...
C: MAIL FROM:<bounce@alphajinsei.com>        <- envelope: sender (where errors go back)
C: RCPT TO:<someone@gmail.com>               <- envelope: recipient (where to deliver)
C: DATA
C: From: Alpha <alphajinsei@alphajinsei.com>  <- header: what the reader sees
C: To: Someone <someone@gmail.com>
C: Subject: hello
C:
C: body ...
C: .
S: 250 OK
```

- 配送に使うのは**封筒だけ**。便箋の `To:` や `From:` は、配送には使われない（BCC が成り立つのはこのため。BCC の相手は封筒の `RCPT TO` にだけいる）。
- 封筒の差出人（envelope from）は、届かなかったときのエラー（バウンス）の戻り先。届いたメールでは `Return-Path:` ヘッダーとして残る。
- 便箋の `From:` は、書く側が何でも書ける。**何もしなければ、誰でも `From: ceo@yourcompany.com` と名乗れる。**これを防ぐのが、次の SPF・DKIM・DMARC。

## 4. ポート

| ポート | 用途 | 暗号化 |
| --- | --- | --- |
| 25 | サーバー同士の配送（MTA → MTA） | STARTTLS（平文で始めて、途中から TLS に切り替える。相手が対応していなければ平文のまま） |
| 587 | メールソフトからの送信（submission） | STARTTLS（ログインの前に必須にする） |
| 465 | メールソフトからの送信（submissions） | 最初から TLS（implicit TLS）。今はこちらが推奨（RFC 8314） |
| 993 | IMAP | 最初から TLS |
| 143 / 110 | 暗号化なしの IMAP / POP3 | 使わない |

クラウドや家庭の回線は、外へ出る 25番を塞いでいることが多い（迷惑メールの送信の踏み台にされるため）。OCI も塞いでいる。受ける 25番は塞がれていない。

## 5. 届かない、迷惑メールにされる理由

受け取る側（Gmail など）は、知らない送信元を基本的に疑う。見ているのは大きく3つ。

1. **IP の評判**：その IP から過去に迷惑メールが来ていないか。クラウドの IP は、前の持ち主の履歴を引き継ぎ、ブロックリストに載っていることが多い。
2. **逆引き（PTR）**：送信元の IP を逆引きした名前と、その名前を正引きした IP が一致するか（FCrDNS）。家庭の回線やクラウドの初期設定の名前だと減点。
3. **ドメインの認証**：SPF・DKIM・DMARC（次の節）。

自前の IP で 1 と 2 を整えるのは手間がかかり、運も絡む。そこで、評判のよい送信サービスに中継してもらう（**スマートホスト**、リレー）のが定番。今回は OCI Email Delivery に中継するので、1 と 2 は Email Delivery の IP の話になり、自分で持つのは 3 だけになる。

### オープンリレー

認証なしで、よそのドメイン宛てのメールまで中継してしまうサーバーを**オープンリレー**と呼ぶ。見つかると数時間で迷惑メールの踏み台になり、IP もドメインもブロックリストに載る。25番では「自分のドメイン宛て」だけを受け、よそ宛ては「ログインした自分のユーザー」からだけ受ける、が絶対の決まり。

## 6. SPF・DKIM・DMARC の全体像

全体像がつかみにくい一番の理由は、**メールには「差出人」が3つある**こと。それぞれの仕組みは、別々の差出人を確かめている。

| 差出人 | どこにあるか | 確かめる仕組み | 確かめること |
| --- | --- | --- | --- |
| ① 封筒の差出人（envelope from） | SMTP の `MAIL FROM`（届いたメールでは `Return-Path:`） | **SPF** | このドメインの DNS が、送ってきた IP に送信を許しているか |
| ② 署名したドメイン | `DKIM-Signature:` ヘッダーの `d=` | **DKIM** | このドメインの DNS にある公開鍵で、署名が正しいか（途中で改ざんされていないか） |
| ③ 便箋の差出人（header from） | `From:` ヘッダー。**人が見るのはこれだけ** | **DMARC** | SPF か DKIM のどちらかが通り、**しかもそのドメインが ③ と一致（アライメント）しているか** |

```
                     checks                 must match the From: domain (alignment)
SPF   --> (1) envelope from domain  ---------+
DKIM  --> (2) d= of the signature   ---------+--> DMARC: pass if (SPF pass AND aligned)
                                                         OR (DKIM pass AND aligned)
          (3) From: header  <---- what the reader sees, protected only by DMARC
```

- **SPF も DKIM も、それだけでは `From:` を守らない。**迷惑メールの送り主も、自分で取ったドメイン `evil.example` で SPF と DKIM を完璧に通したうえで、`From: support@yourbank.com` と書ける。SPF・DKIM が確かめたのは `evil.example` だけ。
- **DMARC が、SPF・DKIM の結果と `From:` を結び付ける。**`From:` のドメイン（`yourbank.com`）の DMARC のレコードを引き、「SPF か DKIM で通ったドメインが `yourbank.com` と一致しているか」を見る。一致の判定は、既定では親ドメインが同じならよい（relaxed。`bounce.alphajinsei.com` と `alphajinsei.com` は一致とみなす）。
- 業務でよくある「SPF は通っているのに DMARC で落ちる」は、たいていこのアライメントが原因。外部の配信サービス（SendGrid、Email Delivery など）は、封筒の差出人を**自分のドメイン**にして送る（バウンスを自分で受けるため）。すると SPF は通るが、通ったのは配信サービスのドメインで、`From:` と一致しない。そこで、DKIM を自分のドメインで署名してもらうか、封筒の差出人を自分のサブドメインにしてもらう（カスタム Return-Path）。

### それぞれの DNS レコード

```
; SPF: who may send with envelope from @alphajinsei.com
alphajinsei.com.                     TXT  "v=spf1 include:<relay's SPF> -all"

; DKIM: public key for signatures with d=alphajinsei.com, s=<selector>
<selector>._domainkey.alphajinsei.com. TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
                                       (or a CNAME to the relay's key)

; DMARC: policy for mail whose From: is @alphajinsei.com, and where to send reports
_dmarc.alphajinsei.com.              TXT  "v=DMARC1; p=none; rua=mailto:dmarc@alphajinsei.com"
```

- SPF の末尾：`-all`（それ以外は拒否）、`~all`（それ以外は疑わしい）。
- DKIM のセレクター（`s=`）は、鍵を複数持つための名前。鍵を替えるときは、新しいセレクターで公開鍵を足し、署名を切り替えてから古いものを消す。
- DMARC のポリシー（`p=`）：`none`（何もしない。レポートだけ受け取る）→ `quarantine`（迷惑メールのフォルダへ）→ `reject`（受け取り拒否）の順に強める。最初は `none` でレポートを見て、正しいメールが全部通ることを確かめてから強める。
- DMARC のレポート（`rua=`）は、受け取る側（Gmail など）が1日1回送ってくる集計（XML）。「あなたのドメインを名乗るメールが、どの IP から何通来て、SPF・DKIM・DMARC がどうだったか」が分かる。なりすましに気付くための仕組み。

### 受け取ったメールでの見え方

Gmail で「メッセージのソースを表示」を開くと、受け取った側の判定が `Authentication-Results:` ヘッダーに残っている。

```
Authentication-Results: mx.google.com;
       dkim=pass header.i=@alphajinsei.com header.s=sel1 ...;
       spf=pass (google.com: domain of bounce@bounce.alphajinsei.com designates 192.0.2.10 as permitted sender) smtp.mailfrom=bounce.alphajinsei.com;
       dmarc=pass (p=NONE sp=NONE dis=NONE) header.from=alphajinsei.com
```

`smtp.mailfrom`（①）、`header.i` / `d=`（②）、`header.from`（③）の3つを見比べると、どれが一致して DMARC が通ったかが分かる。

### 転送で壊れるもの

メールの転送（受け取ったメールを、別のアドレスへそのまま送り直す）では、送り直すサーバーの IP が元のドメインの SPF に載っていないので、**SPF は必ず落ちる**。DKIM は本文とヘッダーが変わらなければ通る。DMARC を `reject` にするなら、DKIM の署名を必ず付けておく（SPF だけに頼らない）。メーリングリストのように件名や本文を書き換えると DKIM も落ちる。その対策が ARC（転送したサーバーが「受け取った時点では通っていた」と署名する仕組み）。

## 7. Gmail の送信者のガイドライン（2024年から）

Gmail は2024年2月から、送る側に次を求めている。

| 対象 | 求められること |
| --- | --- |
| すべての送信者 | SPF か DKIM。送信元の IP の正引きと逆引きの一致。TLS での送信。迷惑メールの報告率 0.3% 未満 |
| 1日5,000通以上 | SPF と DKIM の両方。DMARC（`p=none` でよい）と、そのアライメント。ワンクリックの配信停止 |

個人のメールは前者だけで足りるが、今回は SPF・DKIM・DMARC を全部そろえる。正引きと逆引きの一致は、中継する Email Delivery の IP の話になる。

## 8. 受け取る側として

自分のサーバーも、受け取る側として同じことをする。迷惑メールのフィルター（Rspamd）が、届いたメールの SPF・DKIM・DMARC を確かめ、IP のブロックリスト（RBL）や中身の怪しさと合わせて点数を付け、点数で「受け取る・迷惑メールのフォルダへ・拒否」を決める。
