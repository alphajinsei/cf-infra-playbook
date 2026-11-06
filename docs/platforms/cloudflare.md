# 基盤ガイド：Cloudflare（方式⓪①②）

最終更新：2026-11-06

Cloudflare の上でサービスを作るときに、インフラについて知っておくこと。どの方式でも共通の決まりごとは [サービス開発ガイド](../service-guide.md)、方式ごとの作り方（構成・ビルド・デプロイ・新しいサービスの始め方）は各サンプルの README に書く。

Cloudflare の共通基盤はアカウントそのもの（DNS ゾーン、R2 の `tfstate`、Access のチーム）で、OCI の `oci-platform` のような専用のリポジトリはない。その分をこの資料に置く。なぜこの構成にしたかは [方式検討：Cloudflare](../design/cloudflare.md)。

| 方式 | サンプル | README |
| --- | --- | --- |
| ① 標準・静的 | `sample-cf-workers-static` | [README](https://github.com/alphajinsei/sample-cf-workers-static#readme) |
| ② 標準・アプリ + DB | `sample-cf-workers-d1` | [README](https://github.com/alphajinsei/sample-cf-workers-d1#readme) |

## 1. リクエストの流れ

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

- サービスごとに、サブドメイン1つ・Worker 1つ・GitHub のリポジトリ1つ・手元のフォルダ1つを持つ。Worker の名前はサービスの名前と同じにする。
- Worker はサーバーではない。コードは世界中のエッジに配られ、リクエストが来たときだけ動く。起動・停止・台数・OS を考えなくてよい。
- ①で始めて、あとから API が欲しくなったら、同じ Worker に `main` を足せば②になる。

## 2. 何をどの道具で管理するか

| 道具 | 管理するもの | 書く場所 |
| --- | --- | --- |
| wrangler | Worker のコード、静的ファイル、カスタムドメイン（DNS レコードと証明書を含む）、D1 の作成とマイグレーションの適用、バインディング | `wrangler.jsonc`、`package.json` の scripts |
| Drizzle | DB のテーブル定義（唯一の正）と、差分のマイグレーションの生成。アプリの型付きクエリ | `src/db/schema.ts`、`migrations/` |
| Terraform | wrangler で扱えないもの（Access など） | `infra/terraform/` |
| gh | GitHub のリポジトリ作成、テンプレートからのコピー | （コマンドだけ） |

- カスタムドメインの DNS レコードは wrangler が持つ。Terraform にも書くと取り合いになる。
- デプロイは `npm run deploy`（wrangler）だけ。GitHub 連携（Workers Builds）や GitHub Actions を同じ Worker に混ぜない。
- ブランチごとのプレビューは使わない。確かめるのは手元の `npm run dev`。

## 3. 開発するときに知っておくこと

### Worker の性質

- **ステートレスに作る。**グローバル変数やメモリ上の配列に、消えたら困るデータを置かない（別の isolate に当たると見えない）。状態は D1 / KV / R2 に置く。
- 動くのは JavaScript / TypeScript。Node.js そのものではなく workerd（V8 isolate）なので、Node.js 専用のパッケージは動かないことがある。`npm run dev` で動いたものは本番でも動く（同じ workerd）。
- 1リクエストの CPU 時間には上限がある（無料プランで 10ms 程度。待ち時間は含まない）。重い処理は向かない。

### D1（SQLite）

- コードからは `c.env.DB`（バインディング）で使う。接続文字列もパスワードもない。
- 開発中の `npm run dev` は手元の SQLite ファイル（`.wrangler/state/`）を使う。本番の DB には触れない。
- 表領域・ユーザー・GRANT はない。1サービス ＝ 1つの DB（用途ごとに小さく作る）。名前は `<サービス名>-db`。
- 書き込みは DB 全体で1つずつ。複数の SQL を「全部成功か全部失敗」にしたいときは `batch([...])` で送る（`BEGIN`〜`COMMIT` を何往復かに分けて使えない）。
- スキーマの変更は必ず `schema.ts` から：`schema.ts` を直す → `npx drizzle-kit generate` → できた SQL を読む → `npm run dev` で確かめる → `npm run deploy`。本番に直接 DDL を流さない。
- SQLite の `ALTER TABLE` でできることは少ない。既存のテーブルに、`datetime('now')` のような値が変わる既定値を持つ列は足せない。

### 秘密の値

- Worker の秘密の値は `wrangler secret put <名前>` で Cloudflare に預け、手元の開発用は `.dev.vars`（git 管理外）に置く。
- Terraform の鍵は `infra/.secrets/` に置き、サービスの `infra/terraform/tf.sh` が読み込む。ターミナルに `export CLOUDFLARE_API_TOKEN` しない（wrangler が `wrangler login` より優先して使い、権限の違うトークンで失敗する）。
- アカウント ID、D1 の `database_id` は識別子で、秘密ではない（git に入れてよい）。

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
| `npm run deploy` が認証で失敗する | ターミナルに `CLOUDFLARE_API_TOKEN` が残っている。`unset CLOUDFLARE_API_TOKEN` して `wrangler login` のほうを使う |
| `npm run deploy` が `code: 7403`（not authorized）で失敗する。`CLOUDFLARE_API_TOKEN` は入っていない | OAuth のアクセストークンの期限切れが疑わしい（2026-11-03 に1回発生。`npx wrangler whoami` を実行したあとは成功した）。`npx wrangler whoami` → 再実行。直らなければ `npx wrangler login` |
| `wrangler dev` 中に、勝手にトンネルやリモートに切り替わる | ターミナルへの貼り付け（Ctrl+V）が1文字のキー操作として実行された。Ctrl+C で止める |
| Access を apply したのに効かない | エッジに行き渡るまで数分かかる |
| workers.dev の URL が開いたまま | `wrangler.jsonc` に `"workers_dev": false` を書く（書かないとデプロイのたびに既定値の「有効」に戻る） |
| `drizzle-kit generate` が既存のテーブルを作る SQL を出す | 手書きのマイグレーションの後から Drizzle を入れた（ベースラインの問題）。[番外編](../notes/extra-db-schema-management.md) |
| `env.DB` の型が出ない | `npx wrangler types` で `worker-configuration.d.ts` を作り直す |
| Terraform のネットの例が動かない | Cloudflare provider の v4 の書き方。v5 では名前と書き方が違う（`cloudflare_zero_trust_access_*`） |

## 6. 共通基盤と認証情報

### 共通基盤（サービスをまたいで1つだけあるもの）

| もの | 内容 |
| --- | --- |
| Cloudflare アカウント | Workers / D1 / R2 / Access を持つ |
| DNS ゾーン | `alphajinsei.com`（ドメインはお名前.com、DNS は Cloudflare）。OCI の方式③やメールのレコードも同じゾーンにある |
| R2 バケット `tfstate` | Cloudflare の方式のサービスの Terraform の state（キーは `<サービス名>/terraform.tfstate`）。OCI の state は OCI の Object Storage |
| Access のチーム | `alphajinsei.cloudflareaccess.com`（ログインはメールのワンタイムコード） |

### 認証情報

| 認証情報 | 使う場面 | 置き場所 |
| --- | --- | --- |
| `wrangler login`（OAuth） | 手元の wrangler | `~/.config/.wrangler/` |
| `gh auth login` | 手元の gh | `~/.config/gh/` |
| R2 の S3 互換の鍵 | Terraform の backend | `infra/.secrets/r2-tfstate.env` |
| Terraform 用 API トークン（Access の編集） | Terraform の provider | `infra/.secrets/terraform-cloudflare.env` |
| デプロイ用 API トークン | GitHub Actions を使うとき（今は使っていない） | `infra/.secrets/cloudflare.env`、d1 のリポジトリの Secrets |

## 7. ⓪（Pages + GitHub 連携）について

- 仕組み：Pages のプロジェクトを GitHub のリポジトリにつなぐと、push のたびに Cloudflare がリポジトリを取りに行き、ダッシュボードに設定したビルドコマンドを実行して配信する。ブランチごとにプレビューの URL もできる。
- ①②との違い：push ＝ 本番へのリリース。デプロイの設定（ビルドコマンド、出力先、カスタムドメイン）はダッシュボードにあり、リポジトリのファイルには書かれていない。
- 運用：既存のサイト（`dia-logos`）は、このまま動かしてよい。手元から `wrangler deploy` や `wrangler pages deploy` を混ぜない（デプロイの経路を1つに絞る）。
- 移すなら：GitHub 連携の Pages は CLI 方式に切り替えられないので、①（Workers Static Assets）に作り直す。①のサンプルの README の5章どおりに作り、Pages のカスタムドメインを外してから `npm run deploy` する。
