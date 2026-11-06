# サービス開発ガイド

最終更新：2026-11-06

個人サービスを作るときに、インフラについて知っておくことの入口。読むのは開発する自分と Claude。資料は3段に分けてある。

| 段 | 資料 | 書いてあること |
| --- | --- | --- |
| 全方式に共通 | この資料 | 方式の選び方、どの方式でも守る決まりごと |
| 基盤ごと | Cloudflare：[基盤ガイド：Cloudflare](platforms/cloudflare.md)<br>OCI：[共通基盤 oci-platform の README](https://github.com/alphajinsei/oci-platform#readme) | その基盤の性質と制約、道具の分担、運用、困ったとき、共通の資源と認証情報 |
| 方式ごと | 各サンプルの README | 構成、手元での開発、ビルドとデプロイ、新しいサービスの始め方 |

読む順番：この資料で方式を決める → その基盤の資料 → サンプルの README（サンプルをコピーして始める）。

OCI だけ基盤の資料が別のリポジトリにあるのは、OCI には共通基盤のリポジトリ（`oci-platform`）があり、Cloudflare の共通基盤はアカウントそのもの（リポジトリがない）だから。

なぜこの構成にしたかは [方式検討](design/overview.md)（方式ごとに [Cloudflare](design/cloudflare.md)・[OCI](design/oci.md)）、どう学んだかは [学習ロードマップ](roadmaps/) と [勉強メモ](notes/study-notes.md) を見る。

## 1. 方式を選ぶ

```
 Browser
   |  https://<service>.alphajinsei.com
   v
 Cloudflare (DNS / proxy / TLS)  -- the entrance for every method
   |
   +-- (1)(2) Worker on the Cloudflare edge  [+ D1 / R2 / Access]
   |
   +-- (3)    OCI LB --> container on the app VM --> MySQL HeatWave
                                    |
                                    +--> mail VM --> Email Delivery (sending mail)
```

新しいサービスは、上から順に当てはまる方式を使う。

| 作りたいもの | 方式 |
| --- | --- |
| 記事・ページを表示するだけ（Markdown のブログ、SPA） | ① 標準・静的 |
| データを保存・検索するアプリ（フォーム、メモ、API、Bot の Webhook） | ② 標準・アプリ + DB |
| Workers で動かないもの（常駐プロセス、WebSocket の常時接続、Node.js 専用のライブラリ、任意の言語、コンテナで動かしたいもの、メールを送るもの） | ③ 汎用VM |
| 大手クラウドを触ってみたい | ④ 学習方式（サービスごとに試す） |

⓪（Pages + GitHub 連携）は、既存のサイトがそうなっているので一覧に載せているだけで、新しいサービスには使わない。

### 方式の一覧

方式を足すときは、ここに1行足し、サンプルの README を下の「README の見出し」にそろえて書く。基盤が新しければ、基盤の資料も作る。

| 方式 | サンプル（手元 = GitHub のリポジトリ名） | 構成 | 基盤の資料 | README |
| --- | --- | --- | --- | --- |
| ⓪ 既存・Pages（新規には使わない） | なし（既存の例：`dia-logos`） | Cloudflare Pages + GitHub 連携。main に push すると Cloudflare がビルド・デプロイする | [Cloudflare](platforms/cloudflare.md)（7章） | なし |
| ① 標準・静的 | `sample-cf-workers-static` | Astro（SSG）+ Workers Static Assets | [Cloudflare](platforms/cloudflare.md) | [README](https://github.com/alphajinsei/sample-cf-workers-static#readme) |
| ② 標準・アプリ + DB | `sample-cf-workers-d1` | Workers + Hono + D1 + Drizzle、Access は Terraform | [Cloudflare](platforms/cloudflare.md) | [README](https://github.com/alphajinsei/sample-cf-workers-d1#readme) |
| ③ 汎用VM | `sample-oci-vm-docker` | OCI Always Free：Cloudflare → Flexible LB → Docker ホスト（A1）→ MySQL HeatWave。メールは mail VM から送る | [oci-platform](https://github.com/alphajinsei/oci-platform#readme) | [README](https://github.com/alphajinsei/sample-oci-vm-docker#readme) |
| ④ 学習方式 | 未作成 | | | |

手元では `~/claude-sandbox/<サンプル名>/README.md`、`~/claude-sandbox/oci-platform/README.md`。

## 2. 決まりごと

### 名前

- 手元のフォルダ名 ＝ GitHub のリポジトリ名 ＝ サービスの名前 ＝ サブドメイン（`<名前>.alphajinsei.com`）。小文字とハイフンだけ。
- 基盤の中のものも、この名前を使う（Worker の名前、コンテナのイメージ、LB のバックエンド・セット、DB のスキーマなど）。関連するリソースは、名前を頭に付ける（D1 なら `<名前>-db`、Vault なら `<名前>-db-password`、Terraform の state なら `<名前>/terraform.tfstate`）。
- サブドメインは1段にする（`a.b.alphajinsei.com` は Cloudflare の無料の証明書の対象外）。

### 置き場所

```
~/claude-sandbox/
├── infra/                  this repo (GitHub: cf-infra-playbook, public)
│   └── .secrets/           API keys for Terraform (not in git)
├── oci-platform/           common platform for method (3) (Terraform + Ansible)
├── sample-*/               samples = templates (GitHub template repositories)
└── <service>/              each service: app + deploy settings + infra/terraform
```

- 1サービス ＝ 1フォルダ ＝ 1リポジトリで、アプリからインフラの設定まで、そのフォルダで完結させる。③だけは、共通基盤（`oci-platform`）とアプリの2段になる。
- サービスのフォルダを infra の中に作らない（git が入れ子になる）。
- Claude Code を infra で開いて作業するときは、サービスのフォルダを `infra/.claude/settings.json` の `additionalDirectories` に足す。

### デプロイ

- デプロイは手元の PC から `npm run deploy` の1コマンド。中身は方式ごとに違う（各 README）。⓪（既存の Pages）だけは push でデプロイされる。
- git は保管用。push してもデプロイされない。区切りのいいところで push する。
- デプロイの経路は1つに絞る。CI など別の経路を同じサービスに混ぜない（後から出したほうが本番になり、区別がつかない）。

### 構成管理

- 同じものを2つの道具で管理しない。どの道具が何を持つかは、基盤の資料の表に従う。
- ダッシュボードで手で変えたものは、ファイルに書き戻す。ファイルを正にする。手で持つと決めたもの（Cloudflare の暗号化モードなど）は、基盤の資料に書いておく。
- DB のテーブルは `src/db/schema.ts`（Drizzle）を唯一の正にし、マイグレーションはデプロイで流す。本番に直接 DDL を流さない。

### アプリの作り

- **ステートレスに作る。**消えたら困るデータを、メモリやコンテナの中のファイルに置かない。状態は基盤の DB やストレージに置く。
- ログインが要るページは、アプリで作らず基盤の仕組みを使う（Cloudflare は Access。③は未整備）。

### 秘密情報と公開リポジトリ

- リポジトリは public にする前提で書く。API トークン・鍵・パスワードはファイルに書かない。置き場所は基盤ごとに決まっている（Cloudflare は `wrangler secret`、OCI は Vault）。
- 手元で Terraform などが使う鍵は `infra/.secrets/` に置き、各リポジトリの `tf.sh` が読み込む。
- アカウント ID、OCID、DB の ID などは識別子で、秘密ではない（git に入れてよい）。
- コミットの作者のメールアドレスは公開される。infra は noreply アドレスにした。サービスごとにどちらにするか決める（`git config user.email`）。

## 3. 困ったとき（どの方式でも）

基盤ごとのものは、基盤の資料にある。

| 症状 | 原因と対処 |
| --- | --- |
| `npm` や `npx` が `/mnt/c/...` の Windows 側を指す | WSL の PATH に Windows の PATH が足されている。nvm で WSL 側に Node.js を入れる。Claude の Bash は非対話シェルで nvm が読まれないので `bash -ic '...'` で実行する |

## 4. README の見出し（各サンプル・各サービス共通）

1. この方式の用途（向いているもの、向いていないもの）
2. 構成と主なファイル
3. 手元での開発
4. ビルドとデプロイ
5. このサンプルから新しいサービスを始める（書き換える場所の一覧）
6. この方式の注意点
