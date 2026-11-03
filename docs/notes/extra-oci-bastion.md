# 番外編：OCI Bastion で VM に SSH する

最終更新：2026-11-04

[学習ロードマップ：OCI](../roadmaps/oci.md) の Step 3 で、公開 IP のない VM に Bastion 経由で入れるようにしたときの整理。普段は `ssh oci-app` だけで入れるが、裏で何が起きているかと、Claude やラッパーなしで（別の PC からでも）つなぐ手順を残す。

## 仕組み：SSH を2段で使う

```
PC ==(1) ssh as <session OCID>, RSA key==> Bastion (host.bastion.ap-tokyo-1.oci.oraclecloud.com)
                                             |  forwards TCP only to 10.0.10.10:22
PC ==(2) ssh as ubuntu, ed25519 key ====================================> app VM (10.0.10.10)
         (2) runs inside the tunnel made by (1)
```

- Bastion は OCI が持つ踏み台。VCN の中（private-app サブネット）に窓口を置き、インターネットからの SSH を受ける。
- 入るには **セッション** が要る。「この公開鍵の持ち主は、10.0.10.10 の 22 番に、最長3時間つないでよい」という許可で、OCI の API（oci CLI かコンソール）で作る。作るには OCI の認証（API キー）が要る。
- (1) Bastion への SSH：ユーザー名は **セッションの OCID**、鍵はセッションに登録した **RSA** の鍵。Bastion は、許された宛先へ TCP をつなぐだけ。
- (2) VM への SSH：(1) のトンネルの中で、普通に VM へ SSH する。ユーザーは `ubuntu`、鍵は VM に登録した ed25519。暗号は PC と VM の間で直接かかるので、Bastion からは中身が見えない。
- 認証は3つ重なっている：OCI の認証（セッションを作れるか）、Bastion の鍵、VM の鍵。

## 手でつなぐ手順

準備（1回だけ）：

- oci CLI が使える（`~/.oci/config` と API キー）
- 鍵が2つある：Bastion 用の RSA（`~/.ssh/oci-platform-bastion`）と、VM 用の ed25519（`~/.ssh/oci-platform`。公開鍵は VM に登録済み）
- Bastion の OCID：`terraform -chdir=$HOME/claude-sandbox/oci-platform/terraform output -raw bastion_id`、またはコンソールの Identity & Security → Bastion

### 1. セッションを作る（3時間有効）

```bash
BASTION=$(terraform -chdir=$HOME/claude-sandbox/oci-platform/terraform output -raw bastion_id)
SESSION=$(oci bastion session create-port-forwarding --bastion-id "$BASTION" \
  --ssh-public-key-file ~/.ssh/oci-platform-bastion.pub \
  --target-private-ip 10.0.10.10 --target-port 22 \
  --session-ttl 10800 --display-name ssh-app --query 'data.id' --raw-output)

# ACTIVE になるまで待つ（1分ほど）
oci bastion session get --session-id "$SESSION" --query 'data."lifecycle-state"' --raw-output
```

コンソールでも作れる：Bastion → Create session → Port forwarding → IP とポート → 公開鍵を貼る。できたセッションの「⋮ → Copy SSH command」で、次の A のコマンドが手に入る。

### 2. つなぐ（どちらか）

A. トンネルを張ってからつなぐ（仕組みが見えやすい）：

```bash
# ターミナル1：手元の 2222 番を、Bastion 経由で VM の 22 番につなぐ（-N はコマンドを実行しない。開いたままにする）
ssh -i ~/.ssh/oci-platform-bastion -N -L 2222:10.0.10.10:22 -p 22 \
  "$SESSION"@host.bastion.ap-tokyo-1.oci.oraclecloud.com

# ターミナル2
ssh -i ~/.ssh/oci-platform -p 2222 ubuntu@localhost
```

B. 1行でつなぐ（ProxyCommand。ラッパーがやっているのはこれ）：

```bash
ssh -i ~/.ssh/oci-platform \
  -o ProxyCommand="ssh -i ~/.ssh/oci-platform-bastion -W %h:%p -p 22 $SESSION@host.bastion.ap-tokyo-1.oci.oraclecloud.com" \
  ubuntu@10.0.10.10
```

`-W %h:%p` は「Bastion 経由で宛先の host:port につなぎ、標準入出力をそのまま流す」という指定。外側の ssh がそれをトンネルとして使う。

## ラッパー（`oci-platform/bin/bastion-proxy`）がやっていること

| 手でやること | ラッパー |
| --- | --- |
| 1. セッションを作って ACTIVE を待つ | 同じ宛先・同じ鍵で残り15分以上のセッションがあれば使い回す。なければ作って待つ |
| 2-B. ProxyCommand でつなぐ | `oci-platform/ssh_config`（Terraform が書き出す）の ProxyCommand から呼ばれ、最後に `ssh -W` を実行する |

`~/.ssh/config` の先頭で `Include ~/claude-sandbox/oci-platform/ssh_config` しているので、`ssh oci-app` も Ansible も、この経路を通る。

## はまったこと

| 症状 | 原因 | 対処 |
| --- | --- | --- |
| セッションの作成が `InvalidParameter`（displayName） | セッションの名前が数字で始まっていた（`10.0.10.10-22`） | 英字で始める（`ssh-10-0-10-10-22`） |
| Bastion への接続が `Permission denied (publickey)` | Bastion は RSA の鍵しか受け付けない。ed25519 でもセッションは作れてしまう | Bastion 用に RSA の鍵を別に作る。VM 用は ed25519 のまま |
| スクリプトのエラーが `CalledProcessError` だけで原因が分からない | oci CLI のエラー出力を捨てていた | 失敗したら CLI のメッセージをそのまま出すようにした |

## 別の環境から入るには

- oci CLI の認証：その PC 用の API キーを作り、コンソールに登録する（1ユーザーに3つまで）。
- 鍵：今の2つを安全にコピーするか、新しく作る。Bastion 用はセッションを作るときに登録するだけなので、新しい鍵でもすぐ使える。VM 用は、VM の `~/.ssh/authorized_keys` に公開鍵を足す必要がある（今の PC から入って足す。Terraform の `metadata` は作成時だけ使い、以降の変更は無視している）。
- 手軽な代わり：OCI Cloud Shell（コンソール右上から開くブラウザのシェル）。oci CLI が認証済みなので、手順1と2がそのまま使える（鍵は Cloud Shell に置く）。
- 非常口：コンソールのコンソール接続（シリアルコンソール）。ネットワークを通らずに入れる（パスワードがないので、実際に入るには別の手順が要る）。

## ほかの案との比較

| 案 | 手間 | 守り | 判断 |
| --- | --- | --- | --- |
| **Bastion**（今の形） | セッションを作る分だけ手間。ラッパーで `ssh oci-app` に縮めた | VM に公開 IP がいらない。OCI の認証と鍵の3重 | 採用。OCI の標準の踏み台 |
| VM の SSH を直接インターネットに公開 | 一番楽 | 鍵認証だけにし、NSG で自宅の IP に絞れば現実的。ポートを 10022 などに変えるだけでは、守りとしてはあまり効かない（スキャンですぐ見つかる） | 面倒さが勝つようなら見直す。自宅の IP が変わると NSG を直す手間がある |
| Cloudflare Tunnel（`cloudflared access ssh`） | 設定後は `ssh` が普通に通る | 公開 IP もポートも不要。Cloudflare Access で認証 | Cloudflare 側の障害のときに入れないのが気になり、見送った。本来の用途は自宅のサーバーの公開 |
