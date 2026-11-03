# 勉強メモ

最終更新：2026-11-03

[学習ロードマップ](../roadmaps/) を進める中で分かったことを、短く積み上げていくメモ。長くなった話は番外編（`extra-*.md`）に分けて、ここからリンクする。

## 開発環境（WSL）

- WSL は既定で、Windows の PATH を Linux の PATH の後ろに足す。WSL に入れていないコマンドも Windows 側のものが動いてしまう。apt の nodejs には npm が含まれず、npm だけ Windows 側が使われていた。対策は、WSL の中に nvm で Node.js を入れること。→ [番外編：Node.js まわりの用語](extra-nodejs-basics.md)
- nvm と fnm は同じ役割。情報が多い（AI に聞いても当たりやすい）nvm を選んだ。
- apt で外部のソフトを入れるときは、「配布元の署名鍵（公開鍵）を置く → 配布元を `sources.list.d` に登録し、`signed-by=` で鍵と結び付ける → apt で入れる」の流れになる。向きは SSH の known_hosts に近い。→ [番外編：apt の署名鍵と GPG](extra-apt-signing.md)

## git

- `.gitattributes` の `* text=auto eol=lf` は、テキストファイルの改行を LF にそろえるリポジトリの設定。`core.autocrlf` は自分のパソコンの設定なので、リポジトリに書いておくほうが誰が clone しても確実に効く。CRLF が混ざると、シェルスクリプトが `bad interpreter: /bin/bash^M` で動かなくなる。
- `gh repo create <名前> --private --source . --push` で、手元の既存リポジトリから GitHub のリポジトリを作り、`origin` の登録と push まで一度にできる（ブラウザで作る必要がない）。
- コミットには作者のメールアドレスが必ず記録され、公開リポジトリでは誰でも `git log` で見られる。GitHub が用意する noreply アドレス（`<ID>+<ユーザー名>@users.noreply.github.com`）を使うと、本物のアドレスを隠したまま、アカウントのコミットとして扱われる。メールは届かない。infra（GitHub では `cf-infra-playbook`、公開）はリポジトリ単位で `git config user.email` に設定し、公開前のコミットも `git filter-branch --env-filter` で書き換えた。プッシュ前なら、履歴の書き換えは他に影響しない。
- GitHub の Template repository（リポジトリの Settings で有効にする）は、`gh repo create <新しい名前> --template <元のリポジトリ> --clone` で、履歴なしの丸ごとコピーを作れる機能。サンプルが薄いうちは、テンプレートを別に作ると中身がほぼ重複し、2つを同じ状態に保つ手間が増えるので、サンプルそのものをテンプレートにした。
- git リポジトリの中で別のプロジェクトを `git init` すると入れ子になる。親で `git add .` しても、子は中身のない参照としてしか記録されない。サンプルは infra の隣に `sample-<基盤>-<構成>/` として置くことにした。

## Claude Code

- メモリは、開いたプロジェクトのフォルダごとに分かれる（`~/.claude/projects/<パス>/memory/`）。別のフォルダで開き直すと、約束が引き継がれない。プロジェクトのルールは CLAUDE.md に書いておくほうが確実。
- 作業の拠点の外のフォルダは、`.claude/settings.json` の `permissions.additionalDirectories` に登録すれば、同じセッションから読み書きできる。
- Claude が使う Bash は非対話シェルなので `~/.bashrc` が読まれず、nvm が効かない。

## Node.js / npm

- `npm create foo` は npm の決まりで `npx create-foo` に読み替えられる。ツールを配る側が `create-` で始まる名前で公開している。`create-cloudflare`（C3）は、ひな形を書き出して `npm install` と `git init` までする使い捨てのツール。→ [番外編：Node.js まわりの用語](extra-nodejs-basics.md)

## TypeScript

- TypeScript は JavaScript に型の注釈を足したもの。型は実行前に取り除かれる。
- `export default` で、そのファイルの代表として外に渡すものを決める。Workers は、渡されたオブジェクトの `fetch` を、リクエストのたびに呼ぶ。
- `async` の関数は Promise（あとで結果が届く引換券）を返す。`await` で結果を待つ。
- `satisfies X` は「X の形に合っているか確かめて」という指示。合わないとエディタで赤線が出て、引数の型も X から自動で決まる。
- `(c) => ...` はアロー関数（関数の短い書き方）。`const` は再代入できず、`let` はできる。`type Memo = { ... }` で形を定義し、`Memo[]` は Memo の配列。

## Cloudflare Workers

- `wrangler.jsonc` の主な設定：
  - `name`：Worker の名前（workers.dev の URL やダッシュボードの表示名にも使われる）
  - `main`：入口のファイル
  - `compatibility_date`：Cloudflare の実行環境のどの日付時点の動き方に合わせるかの固定。自分のコードの版ではない。日付を進めない限り、後から入った動作変更は適用されない
- `wrangler dev` で動いているのは Node.js ではなく、本番と同じ実行環境の workerd。Node.js が動かしているのは wrangler だけ。
- Worker はサーバーを起動して待ち受けるのではない。コードは、デプロイ時に世界中のデータセンターに配られる。リクエストが着いたサーバーで isolate を用意し、`fetch()` を呼ぶ。→ [番外編：Workers の実行環境](extra-workers-runtime.md)
- `main` は実行するコード、`assets` はそのまま返すファイル。パスに合うファイルが `public/` にあれば、エッジがそれを返す（Worker は呼ばれない）。なければ Worker の `fetch()` が呼ばれる。振り分けているのは Cloudflare のエッジで、自分のコードではない。
- `wrangler dev` は起動中に1文字のキー操作を受け付ける（`b` ブラウザ、`t` トンネル、`x` 終了など）。VSCode のターミナルで Ctrl+V は貼り付けなので、貼り付けた文字がキー操作として次々に実行されることがある。実際に起きたのは次の2つで、どちらも Ctrl+C で止まり、本番のデプロイ履歴には何も残らなかった。
  - `t`：`cloudflared` を自動でダウンロードし、Quick Tunnel（`*.trycloudflare.com` の使い捨て URL）で手元の dev サーバーを一時的にインターネットに公開する。URL を知っている人は誰でもアクセスできる。Cloudflare Tunnel は、自宅のサーバーなどを、受信のポートを開けずに公開するための技術（方式③では検討したが使わないことにした）。
  - ローカル／リモートの切り替え：remote preview にすると、Cloudflare 上で Worker をプレビューとして動かし、D1 も本番の DB につながる。本番のデプロイとは別物。

## ドメインと DNS

- Custom Domain（`routes` に `custom_domain: true`）を設定すると、DNS レコード・証明書・Worker への振り分けが、`wrangler deploy` で一度にできる。DNS の画面には、種類が「Worker」の Cloudflare 独自のレコードが出る。
- Worker 1つ1つに IP はない。名前解決で返るのは Cloudflare 共通のエッジの IP（Anycast）で、どの Worker を動かすかは、エッジでホスト名を見て決める。
- Custom Domain はホスト名ごと Worker に渡す（Worker がオリジンになる）。Route はホスト名 + パスのパターンで既存のホスト名の手前に割り込む。パスベースの振り分けも可能。
- 無料の証明書（Universal SSL）が自動で守るのは `*.alphajinsei.com` の1段まで。サブドメインは1段にする。
- `workers_dev` を書かないと、デプロイのたびに既定値（有効）で上書きされ、workers.dev の URL が開いたままになる。設定ファイルに明示して、ファイルを正にする。Access を付けるときに迂回路にならないよう、閉じておく。

## デプロイの方式

- 同じ Worker に、手元の `wrangler deploy` と GitHub 連携（Workers Builds）の両方でデプロイできる。後から出したほうが本番になり、どちら由来か分かりにくくなるので、混ぜない。
- 比べた3案：
  - Workers Builds（Cloudflare が GitHub から取りに行く。Pages と同じ方式）：サービスごとにダッシュボードでの設定が要る。
  - 手元の wrangler：設定はほぼ不要だが、デプロイが毎回の操作として残り、ログイン済みのパソコンに縛られる。push 忘れやマイグレーション忘れが起きる。
  - GitHub Actions（GitHub が Cloudflare に送り込む）：ワークフローはファイルなのでテンプレートに入れられる。シークレットは `gh secret set` で登録でき、立ち上げを CLI だけで完結できる。
- GitHub Actions は、push のたびに使い捨ての Ubuntu を用意し、手元と同じ wrangler のコマンドを実行する。認証は `wrangler login` の代わりに、GitHub の Secrets に預けた API トークン（環境変数 `CLOUDFLARE_API_TOKEN`）を使う。
- Cloudflare から見ると、手元の `wrangler deploy` も Actions からのデプロイも、同じ Worker に対する同じ API 呼び出し。違いは認証情報（`wrangler login` の OAuth か API トークンか）と、どこでコマンドを打ったかだけで、後から出したほうが本番になる。実際にデプロイ履歴を見ると、どちらも Author は自分のメール、Source は「Unknown」で、区別できるのは Actions で付けたメッセージ（コミットの ID）だけだった。
- 手元からのデプロイは技術的には禁止できない。決めた方式を守るには、ルールを文書（アプリ側向けの資料、各リポジトリの CLAUDE.md）にし、必要なら仕組みで縛る（例：開発用のパソコンでは `wrangler logout` しておく。`wrangler dev` はローカルの D1 で動くのでログイン不要）。
- `drizzle-kit generate` は手元で実行し、生成された SQL を読んでからコミットする。CI で generate すると、レビューしていない DDL が本番に流れる（列名の変更か「削除して追加」かを対話で聞かれる場面もある）。CI でやるなら「generate し忘れの検知」（generate して差分が出たら失敗させる）にとどめる。
- ブランチのプレビューは、本番と同じ D1 につながる。プレビューのコードやマイグレーションが本番のデータに触れてしまうので、最初は main だけデプロイする。
- 方針転換：一度 GitHub Actions で push デプロイにしたが、使ってみると push が「本番に出す」操作になって重くなった。個人開発では git と本番の一致までの統制は要らないので、アプリは手元の `npm run deploy`（マイグレーション＋デプロイ）に戻し、git は保管用に適宜 push する。静的サイトは失敗しようがないので push デプロイの便利さを取る。机上で決めたことでも、使ってみて合わなければ変えてよい。
- `wrangler deploy --message "$(git describe --always --dirty)"` で、デプロイの履歴にそのときのコミットを残せる。コミット前の変更を含むと `-dirty` が付く。統制はしないが、あとから追えるようにする軽い仕組み。
- Pages も `wrangler pages deploy` で同じ方式にできる。ただし、GitHub 連携で作った Pages のプロジェクトは CLI 方式に切り替えられず、作り直しになる。乗り換えるなら Workers の Static Assets に移すのが自然。

## 静的サイト（方式①）

- Workers は、静的サイトの配信に限れば Pages のほぼ上位互換。Pages にしかない実行環境はない（Pages Functions も中身は workerd）。Pages 固有なのは `functions/` フォルダのファイルベースのルーティングくらいで、Git 連携やプレビューは Workers Builds でもできる。Cron、Durable Objects、Workers Logs など新しい機能は Workers にしかない。
- 静的サイトもアプリと同じく `npm run deploy`（`astro build && wrangler deploy`）にそろえた。SSG はビルドがあるので「失敗しようがない」わけではなく、手元でビルドすれば `astro dev` で見たものと同じ成果物が出る。
- 完全な静的サイトなら Astro の Cloudflare アダプターは不要。`wrangler.jsonc` に `main` を書かず、`assets.directory` を `./dist` にするだけ（コードのない Worker）。
- Astro の既定の出力（`posts/hello/index.html`）だと、`/posts/hello` へのアクセスが `/posts/hello/` に 307 でリダイレクトされる。`build.format: 'file'` で `posts/hello.html` にすると、リダイレクトなしで返る。
- frontmatter の `date: 2026-11-03` は YAML が日付型として読むので、表示が `2026-11-03T00:00:00.000Z` になる。`"2026-11-03"` と引用符で囲む。
- 新しい npm は、依存パッケージのインストールスクリプト（esbuild、workerd の postinstall）を既定で止める（`npm install-scripts ls` で確認）。止められていても、wrangler の dev と deploy は動いた。

## ステートレス

- Step 3 で、メモをメモリ上の配列に持つと、本番ではたまにしか取れなかった。配列は1つの isolate のメモリにしかなく、同じ「データセンター × サーバー × まだ生きている isolate」に当たったときだけ見える。
- Worker はステートレスに作るのが原則で、状態は外の保存先に置く。LB の後ろに並べたアプリサーバーがセッションをメモリに持てないのと同じ（12-Factor App の「プロセスはステートレス」）。
- グローバル変数は、消えても作り直せるもの（キャッシュ）ならよい。消えたら困るデータには使わない。
- 状態の置き場所：D1（表形式のデータ）／KV（設定・キャッシュ。書き込みが行き渡るまで少し時間がかかる）／R2（ファイル）／Durable Objects（状態を持つ処理が必要なときの例外的な仕組み）。

## D1

- バインディング（`wrangler.jsonc` の `d1_databases`）で Worker に DB を渡す。コードからは `c.env.DB`。接続文字列もパスワードも書かない。`database_id` は識別子なので git にコミットしてよい。
- バインディングは単なる呼び名ではなく、実行環境が「接続済み・許可済みの DB の窓口」を `env` に入れて渡してくれる仕組み。`DB` はその窓口を取り出すための名前。Worker と DB の結び付けはデプロイ時に Cloudflare 側で済んでいるので、認証情報が要らない。渡されていない DB には、コードからは触れようがない。アプリサーバーの JNDI データソース（接続情報はサーバー側で設定し、アプリは名前で引くだけ）や、EC2 に付ける IAM ロール（コードに鍵を書かない）に近い。
- `wrangler dev` は既定で、ローカルの SQLite ファイル（`.wrangler/state/v3/d1/` の下）を使う。本番の DB を壊さずに開発できる。
- マイグレーションは、DB の構造の変更を番号付きの SQL ファイルとして積み上げる管理方法。`migrations create` は手元にファイルを作るだけ。`apply --local` は手元のファイル、`apply --remote` で初めて本番に適用する。適用済みかどうかは、DB の中の `d1_migrations` テーブルに記録される。
- wrangler のマイグレーションは「宣言的」ではなく「手続き的」。あるべき姿を書いて差分を取るのではなく、変更手順を番号順に並べ、まだ流していないものだけを流す（Java の Flyway と同じ考え方）。本番のあるべき姿を知るには、ファイルを頭から全部たどる必要があり、手で直接 DDL を流すとずれる（drift）。宣言的な定義から差分の SQL を作るのは、Step 5 の Drizzle の役割。
- マイグレーションでも削除や変更はできる（新しいファイルに ALTER や DROP を書く）。課題は、あるべき姿が1か所にないこと、変更の SQL を毎回手で書くこと、アプリのコード（型や SQL の文字列）と二重管理になること。

## Drizzle

- `schema.ts` を唯一の正にして、構成管理（drizzle-kit が差分のマイグレーションを作る）とアプリのコード（drizzle-orm で型付きのクエリを書く）の両方に使う。「宣言的に書いて、手続き的に適用する」。→ [番外編：DB のスキーマ管理](extra-db-schema-management.md)
- 自分の言葉で：アプリと DB の2つの構造があると、テーブル名や列名の取り決めを両者がそれぞれ持つことになる。Drizzle は、`schema.ts`（テーブル定義 兼 型定義）を1つの「ご本尊」にして、両方をそこから管理する、両者の間を埋める道具。さらに DB 側には、その定義を Terraform のように差分で適用できるようにしている。
- 差分の比較相手は実物の DB ではなく、`migrations/meta/` のスナップショット。本番に直接流した変更（drift）は検出できない。
- 途中から入れると、最初の `generate` で既存のテーブルを作る SQL が再び作られてぶつかる（ベースラインの問題）。
- 使ってみて：`type Memo` や `AS createdAt` がコードから消え、列を足すと何も書かずに結果の型にも入った。列名を書き間違えるとすぐ赤線が出る。隠れたのは、実際に流れる SQL（必要なら生成された SQL を確認する）。
- SQLite では、既存のテーブルに列を足すとき、`datetime('now')` のような値が変わる既定値は付けられない。drizzle-kit が作った SQL でも、流すまでエラーに気づかないことがあるので、生成された SQL は読んでから流す。
- SQL は `prepare("... ?").bind(値)` で組み立てる（文字列をつなげないので SQL インジェクションを防げる）。`.all()` は全行、`.first()` は1行、`.run()` は結果の行が不要なとき。
- SQLite はサーバーではなく、ファイルを直接開いて読み書きするライブラリ。起動・停止の概念がない。`d1 execute --local` は dev を止めていても動く。本番の D1 も、使う側に稼働状態は見えない。料金は読み書きした行数と保存量で決まる。
- 表領域・スキーマ・ユーザー・GRANT はない。1つの DB ＝ 1つのファイル。誰がアクセスできるかは、どの Worker にバインディングを付けるかで決まる。大きな DB を区切るのではなく、用途ごとに小さな DB を作る。
- ロックとトランザクション：
  - SQLite のロックは、ロックファイルを置くのではなく、OS のファイルロックを使う。WAL モードでは `-wal`（書き込み途中のデータ）と `-shm`（読み手と書き手の調整用の共有メモリ）を併用する。読み手は同時に何人でもよいが、書き手は DB 全体で同時に1つだけ（行ロックはない）。
  - 本番の D1 は、1つの DB を1か所の1つのインスタンスが受け持ち、書き込みをそこで順番に処理する。そのため、利用者がロックの競合を意識する場面はほとんどない。
  - D1 では、`BEGIN` 〜 `COMMIT` を何往復かに分けて使うトランザクションは使えない。複数の SQL をまとめて「全部成功か全部失敗」にしたいときは、`c.env.DB.batch([...])` で一度に送る。

## Terraform

- Terraform は、コード（あるべき姿）・state（コードの名前と実物の ID の対応表）・実物（API）の3つを突き合わせて差分を埋める。API はどのリソースがどのコードから作られたかを覚えていないので、Terraform が自前で state を持つ。→ [番外編：Terraform の state と、置き場所](extra-terraform-state.md)
- state をローカルに置くと、別の PC では全部「新規作成」に見えて重複して作られる、PC を失うと対応表も失う、同時実行でロックできない、秘密情報が平文で残る、といった問題がある。バケット（R2）に置けば、正本が1か所になり、ロックもできる。
- 自分の言葉で：本来はサービス側に持ってほしい「管理下のリソースの記録」を、自前で用意しているのが state。CloudFormation ではスタックがこれに当たる（AWS が記録・ロック・ロールバックまで持つ）。
- Terraform は2つの相手と話すので、鍵も2つ要る。backend（state の読み書き）は R2 の S3 互換 API で、アクセスキー ID とシークレットアクセスキー（`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`）を使う。provider（実物の操作）は Cloudflare の API で、API トークン（`CLOUDFLARE_API_TOKEN`）を使う。別の API なので鍵の種類が違い、最小権限のためにも分ける。AWS では同じ IAM ユーザーで兼ねることが多いので意識しにくい。

- Access のアプリを apply した直後は、まだ効いていなかった（`/admin` が Worker まで素通りして 404）。数分後には `cloudflareaccess.com` のログイン画面に 302 で飛ばされるようになった。設定が世界中のエッジに行き渡るまで、少し時間がかかる。
- Cloudflare provider の v5 は v4 から書き方が大きく変わった（例：`cloudflare_access_application` → `cloudflare_zero_trust_access_application`、ポリシーはアカウント単位の `cloudflare_zero_trust_access_policy` を作ってアプリから ID で参照する）。ネットの例は v4 が多いので混ぜない。
- `tf.sh`（鍵を読み込んで terraform を実行するラッパー）を挟むのは、ターミナルに `CLOUDFLARE_API_TOKEN` を残さないため。残すと wrangler が `wrangler login` より優先して使い、権限の違うトークンで失敗する。

- drift を手で起こして plan すると、出るのは「これから戻す」差分（`~ update in-place`）で、「誰かが手で変えた」とは言ってくれない。Terraform にあるのは「あるべき姿」と「今の姿」だけで、変更の経緯は知らない。drift そのものを見たいときは `terraform plan -refresh-only`（state と実物の差だけを「Objects have changed outside of Terraform」として出す）。定期的に見張るなら `plan -detailed-exitcode`（差分があると終了コード 2）を定期実行する。

- state 用のバケットを Terraform で作ろうとすると、その Terraform の state を置く場所がまだない（鶏と卵）。なので、state 置き場だけは手で（wrangler で）作る。Terraform で作りたければ、置き場を作る用の小さな設定を別に用意し、その state だけはローカルに置く。
- wrangler と Terraform の役割分担は、勉強目的だけでなく「それぞれが得意なもの」で分けている。wrangler はコードのビルド、D1 のマイグレーション、バインディング、カスタムドメイン（DNS レコードと証明書まで）を一度に扱える。Terraform でも Worker を作れるが、ビルド済みのコードを渡す必要があり、マイグレーションも扱えない。逆に Access は wrangler では扱えない。同じものを両方から管理すると、互いに相手の変更を drift として戻し合う（例：カスタムドメインの DNS レコードを Terraform でも定義すると、wrangler が作ったレコードとぶつかる）。

## OCI（方式③）

- Always Free の枠（2026-11-03 に公式で確認）：A1（Arm）は計2 OCPU / 12GB（以前の 4 / 24 から減った）で、AMD のマイクロ VM 2台とは別枠。ディスクは**ブートボリュームとブロックボリュームを合わせて 200GB**（ブートは1台 47GB 以上）なので、VM の台数はコアよりディスクで詰まる。PostgreSQL は Always Free にない（無料なのは MySQL HeatWave）。
- コンソールの「Always Free」のラベルは、シェイプごと無料のもの（E2.1.Micro）にだけ付く。A1 は普通の有料のシェイプで、毎月の使用量のうち 1,500 OCPU 時間・9,000 GB 時間（2 OCPU / 12GB を1か月動かし続ける分）までが無料になる仕組みなので、ラベルは付かない。無料かどうかは Cost Analysis の金額で確かめる。
- 有料アカウント（Pay As You Go）でも Always Free の枠はそのまま無料で、使用率の低い VM の回収の対象から外れる。そのかわり枠を超えると課金されるので、予算アラートを置く。
- Flexible LB は L7（HTTP/HTTPS。Host ヘッダーやパスで振り分けられる）、Network Load Balancer は L4。
- セキュリティ・リストはサブネットにかけるルール、NSG は VM や LB などのリソースにかけるルール。NSG は相手を「どの NSG に属するか」で指定できる。どちらもステートフル。
- 1つのサブネットのデフォルトの出口は、Internet GW か NAT GW のどちらか一方。外から直接受ける VM（メール）は公開サブネットに置く。Service Gateway は、OCI のサービス（Object Storage、OCIR など）へインターネットを通らずに出る出口。
- OCI の VM は、NSG に加えて VM の中のファイアウォールも 22 番以外を拒否している（Ubuntu は iptables、Oracle Linux は firewalld）。NSG を開けても、VM の中も開けないと届かない。Docker で公開したポートは、Docker が自分でファイアウォールに穴を足す。
- SELinux（プロセスがどのファイルやポートに触れてよいか）と firewalld（パケットを通すか）は役割が別で、互いにはぶつからない。引っかかるのは、それぞれが Docker やメールサーバーとぶつかるとき（ボリュームの SELinux のラベル、firewalld の reload で Docker のルールが消える、など）。一度 Oracle Linux 9 にしかけたが、Docker の公式サポートの外でもあり、Ubuntu に戻した。
- ブロックボリュームは最小 50GB。小さなデータのために付けると、Always Free のディスクの枠（200GB）を大きく食う。
- OS の自動更新でコンテナが止まる原因は、OS の再起動と Docker 本体の更新の2つ。自動再起動を切り、Docker 本体を自動更新から外し、needrestart にも Docker を再起動させず、Docker の `live-restore`（デーモンが再起動してもコンテナを動かしたままにする）を入れれば、自動更新でコンテナは止まらない。カーネルの更新は、手で再起動するまで効かない。
- 対策は2段構え。①自動更新では再起動を起こさない（上の設定。夜中にやるのはパッケージの入れ替えだけ）。②それでも再起動したとき（手で流す更新、OCI のメンテナンス、障害）は、Docker が OS の起動時に自動で起動し（`systemctl enable docker`）、`restart: unless-stopped` を付けたコンテナを自動で起動し直す。止まるのは OS が上がるまでの1〜2分だけ。自分で `docker stop` したコンテナだけは止めたままになる。
- OS を更新しても、コンテナの中身（ベースイメージのライブラリ）は直らない。イメージを作り直して入れ替える。
- Bastion は、VM の 22 番に最長3時間だけ通れるセッションを作る、OCI の踏み台。VM に公開 IP が要らない。いざというときは、コンソール接続（シリアルコンソール）でネットワークを通らずに入れる。
- Bastion のセッションの鍵は RSA だけ。ed25519 を渡してもセッションは作れてしまうが、接続で `Permission denied (publickey)` になる。セッションの名前（display name）も英字で始める（数字で始まると InvalidParameter）。
- OCI が塞いでいるのは外へ出す 25 番だけ。自前のメールサーバーは、受信は 25 番で直接受け、送信は Email Delivery に 587 番で中継すると、申請も IP の評判の問題も避けられる。
- Cloudflare の DNS でプロキシ（オレンジの雲）を有効にすると、名前解決で返るのは Cloudflare のエッジの IP になる。利用者がつなぐのは Cloudflare で、Cloudflare が改めてオリジン（OCI の LB）につなぎ直す。そのため LB に来るのは Cloudflare の IP だけで、NSG でそこに絞れば、LB の IP を直接たたく迂回を防げる。Origin 証明書はブラウザに信頼されず Cloudflare だけが信頼するので、プロキシを外すと HTTPS も通らなくなる。
- Cloudflare とオリジンの間の暗号化は、SSL/TLS の暗号化モードで決まる。Flexible は Cloudflare → オリジンが HTTP、Full は HTTPS だが証明書を検証しない、Full (strict) は HTTPS で証明書も検証する。③は Full (strict) で、LB が受けるのは 443番だけ。モードはゾーン全体の設定だが、Workers・Pages はエッジで動くので、①②では関係がなかった。
- Terraform の state は、公式の `oci` backend で Object Storage に置ける（ロックあり。S3 互換の方法は非推奨）。
- A1 は Arm なので、x86 の WSL でビルドするイメージは `docker buildx --platform linux/arm64` でクロスビルドする。
- サブドメインとパスの振り分けの比較：パスにすると、アプリ全体に base path の設定が要り、Cookie などのオリジンも共有になる。サブドメインなら Origin 証明書 `*.alphajinsei.com` 1枚で足り、方式をまたいで DNS の向け先を変えるだけで移せる。→ [方式検討：OCI](../design/oci.md)

## 認証情報の一覧

| 認証情報 | 使う場面 | 権限 | 置き場所 |
| --- | --- | --- | --- |
| `wrangler login`（OAuth） | 手元の wrangler（dev、deploy、d1、r2…） | アカウントのほぼ全部 | `~/.config/.wrangler/` |
| `gh auth login` | 手元の gh（リポジトリ作成、Secrets の登録…） | GitHub のリポジトリ操作 | `~/.config/gh/` |
| デプロイ用 API トークン | GitHub Actions（いまは使っていない） | Workers の編集 ＋ D1 の編集 | GitHub の Secrets、`infra/.secrets/cloudflare.env` |
| R2 の S3 互換の鍵 | Terraform の backend | tfstate バケットの読み書き | `infra/.secrets/r2-tfstate.env` |
| Terraform 用 API トークン | Terraform の provider | Access の設定 | `infra/.secrets/terraform-cloudflare.env` |

## ドキュメントの書き方

- コードブロック内の図に日本語を入れると、全角の幅が環境によって英数字2文字分ぴったりにならず、罫線が崩れる。図の中は英数字だけにする。
