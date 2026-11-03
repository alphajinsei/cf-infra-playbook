# サービス開発ガイド

最終更新：2026-11-03

個人サービスを作るときに、インフラについて知っておくことをまとめた資料。読むのは開発する自分と Claude。どの方式でも共通のことはここに書き、方式ごとのこと（構成・ビルド・デプロイ・新しいサービスの始め方）は、各サンプルの README に書く。

なぜこの構成にしたかは [方式検討サマリ](infra-design-summary.md)、どう学んだかは [学習ロードマップ](learning-roadmap.md) と [勉強メモ](study-notes.md) を見る。

## 1. 全体像

### リクエストの流れ

```
 Browser
   |  https://<service>.alphajinsei.com
   v
 Cloudflare edge (DNS / CDN / TLS)  -- anycast, no per-service IP
   |
   +-- Access (optional)  login check for protected paths (e.g. /admin)
   |
   v
 Worker  (picked by hostname = Custom Domain)
   |-- file exists in assets?  -> edge returns the file (no code runs)
   |-- otherwise               -> fetch() of the Worker code (Hono)
   |
   +-- binding --> D1 (SQLite)   / R2 (files) / KV (cache)

 Dev PC (WSL) --- npm run deploy (wrangler) ---> Worker, D1 migrations
 Dev PC (WSL) --- ./tf.sh apply (Terraform) ---> Access
 Dev PC (WSL) --- git push --------------------> GitHub (storage only)
```

- サービスごとに、サブドメイン1つ・Worker 1つ・GitHub のリポジトリ1つ・手元のフォルダ1つを持つ。
- Worker はサーバーではない。コードは世界中のエッジに配られ、リクエストが来たときだけ動く。起動・停止・台数・OS を考えなくてよい。

### 方式を選ぶ

新しいサービスは、上から順に当てはまる方式を使う。

| 作りたいもの | 方式 |
| --- | --- |
| 記事・ページを表示するだけ（Markdown のブログ、SPA） | ① 標準・静的 |
| データを保存・検索するアプリ（フォーム、メモ、API、Bot の Webhook） | ② 標準・アプリ + DB |
| Workers で動かないもの（常駐プロセス、WebSocket の常時接続、任意の言語、PostgreSQL が必須） | ③ 汎用VM（準備中） |
| 大手クラウドを触ってみたい | ④ 学習方式（サービスごとに試す） |

①で始めて、あとから API が欲しくなったら、同じ Worker に `main` を足せば②になる。

⓪（Pages + GitHub 連携）は、既存のサイトがそうなっているので一覧に載せているだけで、新しいサービスには使わない。

### 方式の一覧

方式を足すときは、ここに1行足し、サンプルの README を下の「README の見出し」にそろえて書く。

| 方式 | サンプル（手元 = GitHub = Worker の名前） | 構成 | README |
| --- | --- | --- | --- |
| ⓪ 既存・Pages（新規には使わない） | なし（既存の例：`dia-logos`） | Cloudflare Pages + GitHub 連携。main に push すると Cloudflare がリポジトリを取りに行ってビルド・デプロイする | なし（下の「⓪ について」） |
| ① 標準・静的 | `sample-cf-workers-static` | Astro（SSG）+ Workers Static Assets | [README](https://github.com/alphajinsei/sample-cf-workers-static#readme) |
| ② 標準・アプリ + DB | `sample-cf-workers-d1` | Workers + Hono + D1 + Drizzle、Access は Terraform | [README](https://github.com/alphajinsei/sample-cf-workers-d1#readme) |
| ③ 汎用VM | 未作成（OCI の VM と ADB は作成済み） | OCI VM + Docker + PostgreSQL + Cloudflare Tunnel（予定） | |
| ④ 学習方式 | 未作成 | | |

手元では `~/claude-sandbox/<サンプル名>/README.md`。

#### ⓪ について

- 仕組み：Pages のプロジェクトを GitHub のリポジトリにつなぐと、push のたびに Cloudflare がリポジトリを取りに行き、ダッシュボードに設定したビルドコマンドを実行して配信する。ブランチごとにプレビューの URL もできる。
- ①②との違い：push ＝ 本番へのリリース。デプロイの設定（ビルドコマンド、出力先、カスタムドメイン）はダッシュボードにあり、リポジトリのファイルには書かれていない。
- 運用：既存のサイトは、このまま動かしてよい。手元から `wrangler deploy` や `wrangler pages deploy` を混ぜない（デプロイの経路を1つに絞る）。
- 移すなら：GitHub 連携の Pages は CLI 方式に切り替えられないので、①（Workers Static Assets）に作り直す。①のサンプルの README の5章どおりに作り、Pages のカスタムドメインを外してから `npm run deploy` する。

## 2. 決まりごと

### 名前

- 手元のフォルダ名 ＝ GitHub のリポジトリ名 ＝ Worker の名前 ＝ サブドメイン（`<名前>.alphajinsei.com`）。小文字とハイフンだけ。
- サブドメインは1段にする（`a.b.alphajinsei.com` は無料の証明書の対象外）。
- 関連するリソースは、サービス名を頭に付ける（D1 なら `<名前>-db`、Terraform の state なら `<名前>/terraform.tfstate`）。

### 置き場所

```
~/claude-sandbox/
├── infra/                  this repo (GitHub: cf-infra-playbook, public)
│   └── .secrets/           API keys for Terraform (not in git)
├── sample-cf-workers-*/    samples = templates (GitHub template repositories)
└── <service>/              each service: app + wrangler.jsonc + infra/terraform
```

- 1サービス ＝ 1フォルダ ＝ 1リポジトリで、アプリからインフラの設定まで、そのフォルダで完結させる。
- サービスのフォルダを infra の中に作らない（git が入れ子になる）。
- Claude Code を infra で開いて作業するときは、サービスのフォルダを `infra/.claude/settings.json` の `additionalDirectories` に足す。

### デプロイ

- デプロイは手元の PC から `npm run deploy` の1コマンド。中身は方式ごとに違う（各 README）。⓪（既存の Pages）だけは push でデプロイされる。
- git は保管用。push してもデプロイされない。区切りのいいところで push する。
- デプロイの経路は1つに絞る。GitHub 連携（Workers Builds）や GitHub Actions を同じ Worker に混ぜない（後から出したほうが本番になり、区別がつかない）。
- ブランチごとのプレビューは使わない。確かめるのは手元の `npm run dev`。

### 何をどの道具で管理するか

| 道具 | 管理するもの | 書く場所 |
| --- | --- | --- |
| wrangler | Worker のコード、静的ファイル、カスタムドメイン（DNS レコードと証明書を含む）、D1 の作成とマイグレーションの適用、バインディング | `wrangler.jsonc`、`package.json` の scripts |
| Drizzle | DB のテーブル定義（唯一の正）と、差分のマイグレーションの生成。アプリの型付きクエリ | `src/db/schema.ts`、`migrations/` |
| Terraform | wrangler で扱えないもの（Access など） | `infra/terraform/` |
| gh | GitHub のリポジトリ作成、テンプレートからのコピー | （コマンドだけ） |

- 同じものを2つの道具で管理しない（例：カスタムドメインの DNS レコードを Terraform にも書くと、wrangler と取り合いになる）。
- ダッシュボードで手で変えたものは、ファイルに書き戻す。ファイルを正にする。

### 秘密情報と公開リポジトリ

- リポジトリは public にする前提で書く。API トークン・鍵・パスワードはファイルに書かない。
- Worker の秘密の値は `wrangler secret put <名前>` で Cloudflare に預け、手元の開発用は `.dev.vars`（git 管理外）に置く。
- Terraform の鍵は `infra/.secrets/` に置き、サービスの `infra/terraform/tf.sh` が読み込む。ターミナルに `export CLOUDFLARE_API_TOKEN` しない（wrangler が `wrangler login` より優先して使い、権限の違うトークンで失敗する）。
- アカウント ID、D1 の `database_id` は識別子で、秘密ではない（git に入れてよい）。
- コミットの作者のメールアドレスは公開される。infra は noreply アドレスにした。サービスごとにどちらにするか決める（`git config user.email`）。

## 3. 開発するときに知っておくこと

### Worker の性質

- **ステートレスに作る。**グローバル変数やメモリ上の配列に、消えたら困るデータを置かない（別の isolate に当たると見えない）。状態は D1 / KV / R2 に置く。
- 動くのは JavaScript / TypeScript。Node.js そのものではなく workerd（V8 isolate）なので、Node.js 専用のパッケージは動かないことがある。`npm run dev` で動いたものは本番でも動く（同じ workerd）。
- 1リクエストの CPU 時間には上限がある（無料プランで 10ms 程度。待ち時間は含まない）。重い処理は向かない。

### D1（SQLite）

- コードからは `c.env.DB`（バインディング）で使う。接続文字列もパスワードもない。
- 開発中の `npm run dev` は手元の SQLite ファイル（`.wrangler/state/`）を使う。本番の DB には触れない。
- 表領域・ユーザー・GRANT はない。1サービス ＝ 1つの DB（用途ごとに小さく作る）。
- 書き込みは DB 全体で1つずつ。複数の SQL を「全部成功か全部失敗」にしたいときは `batch([...])` で送る（`BEGIN`〜`COMMIT` を何往復かに分けて使えない）。
- スキーマの変更は必ず `schema.ts` から：`schema.ts` を直す → `npx drizzle-kit generate` → できた SQL を読む → `npm run dev` で確かめる → `npm run deploy`。本番に直接 DDL を流さない。
- SQLite の `ALTER TABLE` でできることは少ない。既存のテーブルに、`datetime('now')` のような値が変わる既定値を持つ列は足せない。

### 無料枠（目安。超えると課金ではなく止まる）

| サービス | 無料枠 |
| --- | --- |
| Workers | 1日10万リクエスト |
| D1 | 合計5GB、1日500万行の読取り・10万行の書込み |
| R2 | 10GB |
| Access | 50ユーザーまで |

## 4. 日々の運用

| やりたいこと | どうするか |
| --- | --- |
| 本番に出す | `npm run deploy` |
| 本番のログを見る | `npx wrangler tail`（リアルタイム）、またはダッシュボードの Workers → Observability（`observability.enabled: true` のとき） |
| 本番の DB を覗く | `npx wrangler d1 execute <DB 名> --remote --command "SELECT ..."` |
| 一部のパスにログインを付ける | `infra/terraform/` に Access のアプリとポリシーを書いて `./tf.sh apply`（数分で効く） |
| 手で変えた設定との差を見る | `./tf.sh plan -refresh-only` |
| 前のバージョンに戻す | `npx wrangler rollback`（コードだけ戻る。DB のマイグレーションは戻らない） |
| サービスをやめる | `npx wrangler delete`、`npx wrangler d1 delete <DB 名>`、`./tf.sh destroy`、GitHub のリポジトリを archive |

## 5. 困ったとき

| 症状 | 原因と対処 |
| --- | --- |
| `npm` や `npx` が `/mnt/c/...` の Windows 側を指す | WSL の PATH に Windows の PATH が足されている。nvm で WSL 側に Node.js を入れる。Claude の Bash は非対話シェルで nvm が読まれないので `bash -ic '...'` で実行する |
| `npm run deploy` が認証で失敗する | ターミナルに `CLOUDFLARE_API_TOKEN` が残っている。`unset CLOUDFLARE_API_TOKEN` して `wrangler login` のほうを使う |
| `wrangler dev` 中に、勝手にトンネルやリモートに切り替わる | ターミナルへの貼り付け（Ctrl+V）が1文字のキー操作として実行された。Ctrl+C で止める |
| Access を apply したのに効かない | エッジに行き渡るまで数分かかる |
| workers.dev の URL が開いたまま | `wrangler.jsonc` に `"workers_dev": false` を書く（書かないとデプロイのたびに既定値の「有効」に戻る） |
| `drizzle-kit generate` が既存のテーブルを作る SQL を出す | 手書きのマイグレーションの後から Drizzle を入れた（ベースラインの問題）。[番外編](extra-db-schema-management.md) |
| `env.DB` の型が出ない | `npx wrangler types` で `worker-configuration.d.ts` を作り直す |
| Terraform のネットの例が動かない | Cloudflare provider の v4 の書き方。v5 では名前と書き方が違う（`cloudflare_zero_trust_access_*`） |

## 6. 付録

### 共通基盤（サービスをまたいで1つだけあるもの）

| もの | 内容 |
| --- | --- |
| Cloudflare アカウント | Workers / D1 / R2 / Access を持つ |
| DNS ゾーン | `alphajinsei.com`（ドメインはお名前.com、DNS は Cloudflare） |
| R2 バケット `tfstate` | すべてのサービスの Terraform の state（キーは `<サービス名>/terraform.tfstate`） |
| Access のチーム | `alphajinsei.cloudflareaccess.com`（ログインはメールのワンタイムコード） |

### 認証情報

| 認証情報 | 使う場面 | 置き場所 |
| --- | --- | --- |
| `wrangler login`（OAuth） | 手元の wrangler | `~/.config/.wrangler/` |
| `gh auth login` | 手元の gh | `~/.config/gh/` |
| R2 の S3 互換の鍵 | Terraform の backend | `infra/.secrets/r2-tfstate.env` |
| Terraform 用 API トークン（Access の編集） | Terraform の provider | `infra/.secrets/terraform-cloudflare.env` |
| デプロイ用 API トークン | GitHub Actions を使うとき（今は使っていない） | `infra/.secrets/cloudflare.env`、d1 のリポジトリの Secrets |

### README の見出し（各サンプル・各サービス共通）

1. この方式の用途（向いているもの、向いていないもの）
2. 構成と主なファイル
3. 手元での開発
4. ビルドとデプロイ
5. このサンプルから新しいサービスを始める（書き換える場所の一覧）
6. この方式の注意点
