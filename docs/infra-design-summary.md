# 個人サービス公開インフラ 方式検討サマリ

最終更新：2026-11-01

個人サービスは Cloudflare（Workers + D1 ほか）を標準基盤とし、1サービス＝1ディレクトリでアプリからインフラまで完結させる。

## 期待する要件

| # | 要件 | 優先度 | 内容 |
| --- | --- | --- | --- |
| R1 | 定常コストゼロ | 必須 | 常時動かすものは無料枠で運用する。学習のための一時的な課金は可 |
| R2 | 手軽なデプロイ | 必須 | git push でデプロイされる。サービスを今後いくつも追加できる |
| R3 | プロジェクト内完結 | 強く希望 | 1つのディレクトリにアプリとインフラのコードが揃い、サービスを全レイヤ分そこで説明しきれる |
| R4 | 構成管理 | できれば | Terraform / Ansible でコード化する（学習も兼ねる） |
| R5 | 対象構成 | 必須 | (a) 静的コンテンツ配信のみ、(b) アプリケーションサーバ + DB |
| R6 | 性能 | 考慮不要 | 個人サイト程度のアクセス |

前提：ドメイン（例：alphajinsei.com）はお名前.comで取得済みで、DNSはCloudflareで運用中。静的サイトは Cloudflare Pages + GitHub連携で運用実績あり（dia-logos）。保育園のページは Cloudflare Access で保護中。アプリの言語は未定（AIで生成する前提）。

## 推奨構成と理由

構成は4つの枠に分ける。新しいサービスはまず標準枠で作り、そこで実現できないときだけ他の枠を使う。

| 枠 | 構成 | 使う場面 |
| --- | --- | --- |
| ① 標準・静的 | Cloudflare Workers Static Assets（または Pages）+ GitHub連携 | 静的サイト、SPA |
| ② 標準・アプリ + DB | Cloudflare Workers + D1（必要に応じて R2 / Access / Cron）。TypeScript + Hono + Drizzle | DBを使うアプリ全般。既定の選択肢 |
| ③ 学習枠 | AWS Lambda + DynamoDB、GCP Cloud Run + Firestore、Azure Functions + Azure SQL など | 大手クラウドを触る目的で、1サービスずつ試す |
| ④ 汎用VM枠 | OCI Always Free VM + Docker + Dokploy/Coolify + Cloudflare Tunnel | Workersで動かないもの（常駐プロセス、任意の言語、PostgreSQLが必須なもの） |

標準を Cloudflare にする理由：

- R1 コスト：無料プランに請求が発生しない。上限を超えると止まるだけで、AWS等のような課金事故が起きない。
- R2 デプロイ：Pagesと同じ git push デプロイが Workers でも使える（Workers Builds）。サービス追加はリポジトリを増やすだけ。
- R3 完結性：共通基盤がアカウントとDNSゾーンだけで、他はすべてサービスの `wrangler.jsonc` と Terraform に書ける。
- R4 構成管理：Terraform の Cloudflare provider で DNS・Access・R2・D1 を管理できる。
- 既存資産：DNS・Pages・Access をすでに運用しており、管理画面も運用の流れも増えない。

制約：Workers の実行環境は V8 isolate（コンテナやVMではない）。そのため動くのは JS/TS が中心で、Python は制約付き、Rust/Go などは WebAssembly 経由となる。DB は SQLite（D1）。

```mermaid
flowchart LR
  user["利用者<br/>ブラウザ"] --> dns
  subgraph cf["Cloudflare（エッジ）"]
    dns["DNS / CDN<br/>名前解決・WAF"] --> access["Access<br/>ログイン（任意）"]
    access --> worker["Worker<br/>静的ファイル配信<br/>アプリ（Hono）"]
    cron["Cron Triggers"] --> worker
    worker -- "バインディング" --> d1["D1<br/>SQLite DB"]
    worker -- "バインディング" --> r2["R2<br/>ファイル"]
    worker -- "バインディング" --> kv["KV<br/>設定・キャッシュ"]
  end
  github["GitHub<br/>サービスのリポジトリ"] -- "push で自動デプロイ" --> worker
  style access stroke-dasharray: 5 5
```

リクエストは DNS/CDN と（必要なら）Access を通って Worker に届き、Worker はバインディングで D1・R2・KV を使う。デプロイは GitHub への push だけで行われる。

## 検討した選択肢

大手クラウドは「常時稼働するVMやDB」が無期限無料にならず、課金の上限もないため学習枠に回す。数値は検討時点の記憶による概算で、採用前に公式ページでの確認が必要。

| 選択肢 | 種類 | 無期限の無料枠（概算） | 判断 |
| --- | --- | --- | --- |
| AWS | 大手クラウド | Lambda 月100万リクエスト、DynamoDB 25GB。EC2/RDS は新規アカウントのクレジットのみ | 学習枠 |
| GCP | 大手クラウド | Cloud Run 月200万リクエスト、Firestore 1GB、e2-micro 1台。Cloud SQL は無料なし | 学習枠 |
| Azure | 大手クラウド | Functions、Cosmos DB 無料枠、Azure SQL Database 無料オファー | 学習枠 |
| OCI | 大手クラウド | ARM VM 計4コア/24GB、Autonomous DB 2インスタンス（各約20GB） | 汎用VM枠。ADB は未使用で停止・回収、プライベートエンドポイント不可 |
| Neon | DBのみ（PostgreSQL） | 0.5GB/プロジェクト。未使用で自動停止し、次の接続に待ちが発生 | 保留。PostgreSQL必須なら Workers + Hyperdrive と組む |
| Supabase | DB + 認証/ストレージ/API | 500MB、2プロジェクト。1週間未使用で一時停止 | 保留 |
| Render | アプリ実行（Heroku型） | Webサービスは15分未使用でスリープ。無料のPostgreSQLは期限付き | 不採用 |
| Fly.io / Railway | アプリ実行 | 実質的に有料 | 不採用 |

## 設計上の論点

### 共通基盤とサービス固有の境界

共通基盤は小さく保ち、それ以外はすべてサービスのディレクトリに置く。

| 層 | 含むもの | 置き場所 |
| --- | --- | --- |
| 共通基盤 | Cloudflareアカウント、DNSゾーン、Terraform state 用の R2 バケット、（使うなら）VM と Tunnel | platform リポジトリ |
| サービス固有 | アプリコード、DBとスキーマ、サブドメイン、Cron、Access、CI/CD | 各サービスのリポジトリ |

標準枠のサービスのディレクトリ構成例：

```
my-service/
├── src/                 # アプリ（Hono 等）
├── migrations/          # D1 のスキーマ（SQL）
├── wrangler.jsonc       # Worker定義：D1/R2/Cron の紐付け、カスタムドメイン
├── infra/terraform/     # Access など wrangler で扱えないもの
└── .github/workflows/   # CI/CD
```

### アプリとDBの間の通信

標準枠では D1 を使うため、この懸念は発生しない。Worker から D1 へはバインディングで接続し、接続文字列もパスワードも不要で、DBがインターネットに公開されない。

外部のDBサービス（Neon 等）や無料版の OCI ADB では、通信がインターネットを経由する。その場合は「経路は信用せず、TLSと認証で守る」ゼロトラストの考え方で対応する。実務上は遅延の方が問題になりやすく、Hyperdrive で接続をプールして緩和する。個人サイトのため、個人情報などの機密データは扱わない前提とする。

### 汎用VM枠の Docker と冗長性

- Docker は冗長化のためではなく、依存関係の分離（1台のVMで複数サービスのミドルウェアが衝突しない）、再現性、`Dockerfile` でサービスのディレクトリ内に完結させるために使う。
- 冗長化はしない。VM 1台が単一障害点になることを受け入れ、Terraform + Ansible で「作り直せば元に戻る」状態を保つ。
- DB は日次で R2 にバックアップする。目標は RTO 数時間、RPO 1日。

## 登場サービスの解説

Cloudflare は「世界中のCDN拠点（エッジ）」の上に、実行環境・ストレージ・認証を載せたサービス群である。AWS のようにリージョンやVPCを選ぶ概念がなく、すべてがエッジで動く。

### Cloudflare

| サービス | 何をするものか | AWS で言うと | OCI で言うと | 無料枠（概算） |
| --- | --- | --- | --- | --- |
| DNS | ドメインの名前解決。プロキシを有効にするとCDN・WAF経由になる | Route 53 | OCI DNS | 無料 |
| Pages | 静的サイトのホスティング。Git連携でビルドと配信まで自動化 | Amplify Hosting（S3 + CloudFront） | Object Storage の静的ホスティング | サイト数無制限、月500ビルド |
| Workers | エッジで動くサーバレス関数。HTTPリクエストを受けてコードを実行する。アプリサーバの役割 | Lambda + API Gateway（配置は Lambda@Edge に近い） | OCI Functions + API Gateway | 1日10万リクエスト |
| Workers Static Assets | Worker に静的ファイルを同梱して配信する機能。Pages の後継的位置づけで、新規はこちらが推奨 | S3 + CloudFront + Lambda をまとめたもの | （直接の相当なし） | 静的ファイルの配信は無料 |
| D1 | サーバレスの SQLite。Worker からバインディングで使うリレーショナルDB | RDS（ただし SQLite でサーバレス） | Autonomous Database | 合計5GB、1日500万行の読取り・10万行の書込み |
| R2 | S3互換のオブジェクトストレージ。転送量（egress）の課金がない | S3 | Object Storage | 10GB |
| KV | 全拠点に複製されるキーバリューストア。読みが速く、反映は数十秒遅れる。設定やキャッシュ向き | DynamoDB / ElastiCache の用途 | NoSQL Database | 1日10万読取り・1,000書込み |
| Cron Triggers | Worker を定時実行する | EventBridge Scheduler | Resource Scheduler | 無料 |
| Access | アプリの手前でログインを求める認証プロキシ（ゼロトラスト）。アプリに認証機能が不要になる | Verified Access、または ALB + Cognito | IAM Identity Domains + WAF | 50ユーザ |
| Tunnel | サーバ側から Cloudflare へ外向きに接続し、受信ポートを開けずに公開する | 直接の相当なし（ALB + パブリックサブネットを不要にするもの） | 同左 | 無料 |
| Hyperdrive | Worker から外部 PostgreSQL/MySQL への接続プール兼キャッシュ | RDS Proxy | （直接の相当なし） | 要確認 |
| Turnstile | フォームのボット対策（CAPTCHA） | WAF CAPTCHA | （直接の相当なし） | 無料 |
| Email Routing | 独自ドメイン宛のメールを受信し、既存アドレスへ転送 | SES の受信ルール | Email Delivery（送信のみ） | 無料 |

開発では wrangler（Cloudflare 公式のCLI）を使う。ローカル実行、D1 のマイグレーション、デプロイを担い、設定は `wrangler.jsonc` に書く。

### アプリ層のライブラリ

標準枠②の「TypeScript + Hono + Drizzle」は、Worker の中で動くアプリのコードに使うライブラリの組み合わせである。インフラではなくアプリ側の選択で、どちらも Workers 上で問題なく動く軽量なものを選んでいる。

| ライブラリ | 何をするものか | 例えるなら | 選ぶ理由 |
| --- | --- | --- | --- |
| Hono | Web アプリのフレームワーク。URL ごとに処理を割り当てる（ルーティング）、リクエストの受け取り、レスポンスの返却、認証や CORS などの共通処理（ミドルウェア）を担う | Node.js の Express、Python の Flask、Java の Spring MVC の軽量版 | Workers 向けに作られた日本発の OSS。非常に小さく起動が速い。`app.get('/api/items', (c) => ...)` のように書け、`c.env.DB` で D1 にそのまま触れる |
| Drizzle（Drizzle ORM） | TypeScript 用の ORM。テーブル定義を TypeScript で書き、そこから型付きのクエリ実行と、マイグレーション用 SQL の生成を行う | Java の MyBatis / JPA、Python の SQLAlchemy | D1（SQLite）に公式対応。SQL に近い書き方で、列名の誤りや型の不一致をコンパイル時に検出できる。付属の drizzle-kit がスキーマの差分から `migrations/` の SQL を作り、それを wrangler で D1 に適用する |

処理の流れ：リクエスト → Hono がルートに応じて関数を呼ぶ → 関数内で Drizzle を通じて D1 を読み書き → Hono が JSON や HTML を返す。どちらも必須ではなく、素の Workers API と生の SQL でも書けるが、AI にコードを生成させる際に構造と型が揃い、誤りに気づきやすくなる。

### Cloudflare 以外

| サービス | 何をするものか |
| --- | --- |
| Neon | PostgreSQL だけのマネージドサービス。未使用時は自動停止し、DBをブランチで複製できる |
| Supabase | PostgreSQL に認証・ファイル保存・自動生成のREST APIを付けたもの。アプリサーバなしでも使える |
| Render | Heroku 型のアプリ実行サービス。`render.yaml` で構成をリポジトリに書ける |
| Dokploy / Coolify | 自分のVMに入れて使う Heroku 型のデプロイツール。git push で Docker コンテナをデプロイする |
| UptimeRobot | 外部からの死活監視。無料で50監視、5分間隔 |

## ユースケース別の構成パターン

どのユースケースも標準枠に部品を足すだけで組め、常駐接続が必要なものだけが汎用VM枠になる。

| ユースケース | 例 | 構成 | 使い方 |
| --- | --- | --- | --- |
| 静的サイト | ブログ、ポートフォリオ | Workers Static Assets / Pages | ビルド成果物を配信するだけ |
| アプリ + DB | メモや記録のアプリ | Workers + D1 | `wrangler.jsonc` で D1 を紐付け、コードから `env.DB` で使う |
| 定期実行 | 毎朝RSSを集めて要約し保存する | Workers Cron Triggers（重い処理は GitHub Actions の schedule） | `crons` を書き `scheduled()` を実装。普段のアプリと同じ Worker に同居できる |
| 自分だけが使うツール | 家計簿、管理画面 `/admin` | Access | ホスト名やパスに Access を設定し、Google/GitHub/メールOTPでログイン。保育園ページと同じ仕組み |
| ファイル置き場 | 画像アップロード、バックアップ | R2 | Worker にバインドするか、カスタムドメインで公開 |
| Bot・Webhook | Discordのスラッシュコマンド、LINE Bot | Workers | HTTPで呼ばれる型は Workers。WebSocketで常時接続する型は汎用VM枠 |
| 問い合わせフォーム | サイトのフォーム | Workers + Turnstile | Turnstile でスパムを弾き、内容を D1 に保存 |
| 独自ドメインのメール | contact@alphajinsei.com | Email Routing | 受信したメールを Gmail へ転送 |
| 死活監視 | サービスの停止に気づきたい | UptimeRobot + Workers Logs | 停止をメールやDiscordに通知。アプリのログは Workers Logs で見る |
| 任意の言語・常駐プロセス | Rails、PostgreSQL必須のアプリ | 汎用VM枠 | Docker コンテナを Dokploy でデプロイし、Tunnel で公開 |

## 次のステップ

- [ ] 標準枠（Workers + D1）でサンプルアプリを作り、サブドメインにデプロイする（[学習ロードマップ](learning-roadmap.md)）
- [ ] Terraform state 用の R2 バケットと API トークンを用意する
- [ ] サンプルをもとに、サービス開発側が使うインフラ設計資料（テンプレートと手順）を作る
- [ ] 無料枠の数値を各社の公式ページで確認する
