# 番外編：Terraform の state と、置き場所

最終更新：2026-11-03

[学習ロードマップ](learning-roadmap.md) の Step 7 で、state を R2 に置く理由の整理。

## Terraform の仕組み

Terraform は、手元で動くクライアントのツール。3つのものを突き合わせて、差分を埋める。

```
 (1) Config (*.tf)          (2) State (terraform.tfstate)       (3) Real resources
 desired shape              "which real thing is which"         Cloudflare API
 ----------------------     -----------------------------       ------------------
 resource "access_app"  <-> address: cloudflare_..._app.admin <-> app id: 7f3a...
   "admin" { ... }          id: 7f3a...
                            last known attributes
                                                                 ^
 terraform plan:                                                 |
   1. read state -> know which real IDs belong to this config    |
   2. refresh: ask the API for the current values of those IDs --+
   3. compare config vs real -> show the diff
 terraform apply: call the API to close the diff, then update the state
```

state に入っているのは、主に次のもの。

- コードの名前（`cloudflare_zero_trust_access_application.admin`）と、実物の ID（`7f3a...`）の対応表
- 前回 apply したときの属性の値
- リソース同士の依存関係

### なぜ state が必要なのか

Cloudflare や AWS の API は、「このリソースはどのコードから作られたか」を覚えていない。名前が重複できるリソースや、名前のないリソースもある。そのため、Terraform は自分で対応表を持つ必要がある。

- コードから消したリソースを、apply で削除できるのは、state に「前回はあった」と記録されているから。state がないと、コードから消したリソースは削除されず、放置される。
- `plan` で「変更」と「新規作成」を区別できるのも、state の対応表があるから。

## state をローカルに置いたときの課題

| 課題 | 何が起きるか |
| --- | --- |
| 別の PC から実行できない | 別の PC には state がないので、Terraform は「何も作っていない」と判断する。`plan` は全部「新規作成」になり、apply すると重複して作られるか、名前の重複でエラーになる |
| PC を失うと対応表も失う | 実物は残っているのに、どれが自分の管理下か分からなくなる。`terraform import` で1つずつ対応を付け直すことになる |
| 複数の state が食い違う | 2台の PC でそれぞれ apply すると、state がそれぞれ別の内容になり、どちらも正しくない状態になる |
| 同時に実行すると壊れる | 2つの apply が同時に state を書き換えると、state が壊れる。ローカルのファイルでは、PC をまたいだロックができない |
| 秘密情報が平文で残る | state には、トークンなどの値がそのまま入ることがある。PC の中に平文で残り、バックアップなどにも混ざる |
| git に入れられない | 秘密情報が入るうえ、マージで衝突する。git で共有するのは避ける |

## バケット（R2）に置くと何が解消されるか

| 解消されること | 仕組み |
| --- | --- |
| どの PC からでも同じ state を使える | state の正本が1か所（バケット）になる。手元にはキャッシュしか置かない |
| PC を失っても困らない | state は PC の外にある。R2 のバケットに置いておけば消えにくい |
| 同時実行を防げる | backend のロック機能で、apply 中は他の apply を待たせる。S3 backend では、Terraform 1.10 から `use_lockfile = true` で、バケットの中のロック用ファイルを使ってロックできる。R2 でも使えるはずだが、念のため動作を確認する |
| 秘密情報を PC に残さない | バケットの読み書きは、API トークンを持つ人だけができる |

R2 は S3 互換の API を持っているので、Terraform の `s3` backend の接続先を R2 に向けて使う。

## state を自前で持つ理由と、CloudFormation のスタック

本質的には、**「どのリソースがこのコードの管理下か」を、サービス側が持ってくれないので、Terraform が自前で持っている**。

AWS の CloudFormation では、**スタック**がこれに当たる。

| | Terraform（Cloudflare などに対して） | CloudFormation |
| --- | --- | --- |
| 管理下のリソースの記録 | state ファイル（自分で置き場所を用意する） | スタック（AWS が持つ） |
| 差分の確認 | `terraform plan` | 変更セット（Change Set） |
| 同時実行の防止 | backend のロック（自分で用意する） | スタックの更新は1つずつ（AWS が管理） |
| 失敗時 | 途中までの変更が残る。state を見て直す | 自動でロールバック |
| 実物との差（drift）の検知 | `plan` のたびに refresh して検知 | ドリフト検出の機能 |

Terraform は、どのクラウドにも使える代わりに、スタックに当たるものを利用者が用意する。HCP Terraform（旧 Terraform Cloud）や Pulumi Cloud は、この state の置き場所とロックをサービスとして肩代わりするもの。

## Terraform のコードに書いてはいけないもの

- API トークンなどの秘密情報は、`.tf` ファイルに書かない。環境変数（`CLOUDFLARE_API_TOKEN` など）か、`.gitignore` に入れた `terraform.tfvars` で渡す。
- `terraform.tfstate`、`.terraform/` は git に入れない（state は秘密情報を含みうる。`.terraform/` はダウンロードしたプロバイダーの置き場）。
- `.terraform.lock.hcl`（プロバイダーの版を固定するファイル）は git に入れる。
