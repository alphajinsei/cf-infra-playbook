# 学習ロードマップ：Cloudflare 標準構成（Workers + D1）

最終更新：2026-11-02

## ゴール

標準構成のサンプルアプリ（メモ帳）を、自分の手で作ってデプロイし、構成管理まで一通り経験する。完了後は、各ステップで得た理解をもとに、以降のサービスのデプロイを Claude に任せられる状態にする。

- 公開先：`sample-cf-workers-d1.alphajinsei.com`
- アプリ：メモの一覧・追加・削除ができる画面と API
- 技術：TypeScript + Hono + D1（後半で Drizzle）、デプロイは wrangler（GitHub Actions での push デプロイも試した）、構成管理は Terraform

## 進め方

- 各ステップの「理解ポイント」に自分の言葉で答えられたら完了とする。
- 詰まったら、エラー全文と実行したコマンドを Claude に渡す。
- 途中で分かったことは [勉強メモ](study-notes.md) に積み上げる。
- 開発環境：WSL2（Ubuntu）。
- ディレクトリ構成：サンプルアプリは方式ごとに `~/claude-sandbox/sample-<基盤>-<構成>/` として infra の隣に作り、それぞれ独立した git リポジトリにする（フォルダ名＝GitHub のリポジトリ名＝Worker の名前。小文字とハイフンのみ）。このロードマップのサンプルは `sample-cf-workers-d1`。
- 作業の拠点は infra：VSCode は infra を開き、サンプルのフォルダを「フォルダーをワークスペースに追加」で並べる。Claude Code も infra で開き、サンプルのフォルダは `.claude/settings.json` の `additionalDirectories` に登録する（サンプルを増やしたら追記する）。
- Windows と WSL の使い分け：`~/claude-sandbox` 配下（Web サービス、Ansible / Docker / bash を使うもの）は WSL。Windows を自動操作するスクリプトと Chrome 拡張機能（chrome-extention-for-claude-enter-key）は Windows に置く。1つのリポジトリを両方から編集しない。

| Step | 内容 | 目安 | 主に学ぶこと |
| --- | --- | --- | --- |
| 0 | WSL のセットアップとログイン | 1〜2時間 | 開発環境、ツールと認証の関係 |
| 1 | Hello World を手動デプロイ | 1時間 | Worker とは何か、wrangler の基本 |
| 2 | カスタムドメインで公開 | 30分 | DNS と Worker の結び付き |
| 3 | 画面と API を作る | 1〜2時間 | Static Assets と Hono のルーティング |
| 4 | D1 でデータを保存する | 2時間 | バインディング、マイグレーション、local/remote |
| 5 | Drizzle を導入する | 1時間 | ORM とスキーマ管理 |
| 6 | デプロイの方式を決める | 1〜2時間 | GitHub 連携、GitHub Actions、CI/CD |
| 7 | Terraform で Access を管理 | 2〜3時間 | IaC、state、wrangler との役割分担 |
| 8 | 運用の仕組みを足す（見送り） | - | ログ、監視、バックアップ、定期実行 |
| 9 | 振り返りとテンプレート化 | Claude と | 次のサービスに展開する形にする |

---

## Step 0：WSL のセットアップとログイン

WSL2 の Ubuntu は導入済み。以下のコマンドは、特に断りがなければ Ubuntu（WSL）のターミナルで実行する。Windows 側に入れた gh / Terraform は使わないが、残しておいても害はない。

### 0-1. Ubuntu の基本設定

- [x] パッケージを更新し、基本ツールを入れる

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl unzip rsync build-essential
```

- [x] git の初期設定（2026-11-02 に Claude が設定済み：`user.name`、`user.email`、`core.autocrlf input`、Windows 側と同じグローバル除外設定 `~/.config/git/ignore`）
- [x] 残り：`git config --global init.defaultBranch main`

- [ ] VSCode（Windows 側）に拡張機能「WSL」（`ms-vscode-remote.remote-wsl`）を入れる

### 0-2. CLI の導入

- [x] Node.js：nvm（バージョン管理ツール）で LTS の 24 を入れる。用語は [番外編：Node.js まわりの用語](extra-nodejs-basics.md) を参照

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash
# ターミナルを開き直してから
nvm install 24 && nvm alias default 24
node -v && npm -v
which node npm   # 両方とも ~/.nvm/... を指していれば OK（/mnt/c/... なら Windows 側が使われている）
```

- [x] 不要になった apt の Node.js（v18、サポート終了済み）を消す：`sudo apt remove -y nodejs`

- [x] GitHub CLI：[公式の手順（Debian/Ubuntu）](https://github.com/cli/cli/blob/trunk/docs/install_linux.md)で apt リポジトリを追加して入れる

```bash
(type -p wget >/dev/null || (sudo apt update && sudo apt install wget -y)) \
	&& sudo mkdir -p -m 755 /etc/apt/keyrings \
	&& out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
	&& cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
	&& sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
	&& sudo mkdir -p -m 755 /etc/apt/sources.list.d \
	&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
	&& sudo apt update \
	&& sudo apt install gh -y
```

- [x] Terraform：[公式の手順（Linux → Ubuntu/Debian）](https://developer.hashicorp.com/terraform/install)で apt リポジトリを追加して入れる

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
```

補足：どちらも「署名鍵を置く → 配布元（apt リポジトリ）を登録する → apt で入れる」という同じ流れ。Ubuntu 標準の apt には入っていないか古いので、配布元を追加する。以後は `sudo apt upgrade` で一緒に更新される。仕組みは [番外編：apt の署名鍵と GPG](extra-apt-signing.md) を参照。
- [x] `node -v`、`gh --version`、`terraform -version` がすべて通ることを確認する

補足：wrangler はグローバルに入れず、プロジェクトごとに `npm install -D wrangler` してバージョンを固定する（`npx wrangler ...` で呼ぶ）。

### 0-3. 作業ディレクトリを WSL に移す

- [x] claude-sandbox 一式（Chrome 拡張機能を除く）を `~/claude-sandbox` に移動済み（2026-11-02、Claude が実施。`node_modules` / `.venv` は除外したので、各プロジェクトで必要になったら `npm install` などをやり直す）
- [x] infra の git 管理を始める

```bash
cd ~/claude-sandbox/infra
printf '* text=auto eol=lf\n' > .gitattributes
git init && git add . && git commit -m "Add infra design docs"
```

- [x] `code .` で VSCode を開き、左下に「WSL: Ubuntu」と表示されることを確認する
- [x] Windows 側に残っている `claude-sandbox\infra` と空の `claude-sandbox\todo` を削除する（以降は WSL 側だけを編集する）

### 0-4. Claude Code を WSL 側で使えるようにする

- [x] WSL に接続した VSCode で Claude Code を開き、ログインする（WSL 側の設定は `~/.claude` で、Windows 側とは別になる）
- [x] Windows 側の自作スキルを WSL 側にコピーする

```bash
mkdir -p ~/.claude/skills
cp -r /mnt/c/Users/peinn/.claude/skills/about-me /mnt/c/Users/peinn/.claude/skills/browser-check ~/.claude/skills/
```

- [x] 新しいセッションで `docs/learning-roadmap.md` を読ませ、続きから進める

### 0-5. ログイン

- [x] `gh auth login` で GitHub にログイン（GitHub.com → HTTPS → ブラウザでログイン）
- [x] `npx wrangler login` で Cloudflare にログイン（Windows のブラウザで認可画面が開く）
- [x] `gh auth status` と `npx wrangler whoami` で、ログインしたアカウントが表示されることを確認する

理解ポイント：

- `/mnt/c/...`（Windows 側）と `~/`（Linux 側）は何が違うか。プロジェクトを `~/` に置くのはなぜか。
- `gh` と `wrangler` はそれぞれどこに認証情報を保存し、何の権限を得ているか（後の Step 6/7 で使う API トークンとの違い）。

## Step 1：Hello World を手動デプロイ

やること：

- [x] `cd ~/claude-sandbox && npm create cloudflare@latest sample-cf-workers-d1` を実行し、「Hello World example」→「Worker only」→「TypeScript」を選ぶ（git は Yes、デプロイは No）
- [x] 生成された `wrangler.jsonc`、`src/index.ts`、`package.json` を読む
- [x] `npx wrangler dev` でローカル起動し、ブラウザで表示を確認
- [x] `npx wrangler deploy` でデプロイし、`*.workers.dev` の URL で表示を確認
- [x] ダッシュボード（Workers & Pages）で、デプロイしたものを確認
- [x] `npx wrangler tail` を実行しながらアクセスし、ログが流れることを確認

理解ポイント：

- `wrangler.jsonc` の `name`、`main`、`compatibility_date` はそれぞれ何を決めているか。
- `wrangler dev` で動いているのは何か（ローカルの workerd という実行環境）。
- Worker には「起動しっぱなしのサーバ」がない。では、リクエストが来たときに何が起きているか。

参考：[Workers Get started](https://developers.cloudflare.com/workers/get-started/guide/)、[番外編：Workers の実行環境](extra-workers-runtime.md)

## Step 2：カスタムドメインで公開

やること：

- [x] `wrangler.jsonc` に以下を追加して `npx wrangler deploy`

```jsonc
"routes": [{ "pattern": "sample-cf-workers-d1.alphajinsei.com", "custom_domain": true }]
```

- [x] `https://sample-cf-workers-d1.alphajinsei.com` で表示を確認
- [x] ダッシュボードの DNS で、自動作成されたレコードを確認
- [x] `"workers_dev": false` を追加して workers.dev の URL を無効化してみる

理解ポイント：

- Custom Domain と Route（`example.com/api/*` のような指定）の違い。
- DNS レコードを自分で作っていないのに名前解決できる理由。

参考：[Custom Domains](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)

## Step 3：画面と API を作る

やること：

- [x] `npm install hono` で Hono を入れ、`src/index.ts` を Hono で書き直す
- [x] `GET /api/health` が `{"ok": true}` を返す API を作る
- [x] `public/index.html` を作り、`wrangler.jsonc` に Static Assets を設定する

```jsonc
"assets": { "directory": "./public" }
```

- [x] 画面から `fetch('/api/health')` を呼び、結果を表示する
- [x] メモの一覧・追加・削除 API（`GET/POST /api/memos`、`DELETE /api/memos/:id`）を、まずはメモリ上の配列で作る。`wrangler dev` を起動したまま curl で確かめる

```bash
curl http://localhost:8787/api/memos
curl -X POST http://localhost:8787/api/memos -H "Content-Type: application/json" -d '{"text":"はじめてのメモ"}'
curl -X DELETE http://localhost:8787/api/memos/1 -i
```
- [x] デプロイし、何度かアクセスしてメモが消えたり残ったりする様子を観察する

```bash
npx wrangler deploy
URL=https://sample-cf-workers-d1.alphajinsei.com
curl -X POST $URL/api/memos -H "Content-Type: application/json" -d '{"text":"本番のメモ"}'
for i in $(seq 1 10); do curl -s $URL/api/memos; echo; done   # 10回続けて一覧を取る
```

観察のしかた：直後に何度か取る／数分〜数十分放置してから取る／スマホのモバイル回線から開く／もう一度デプロイしてから取る

理解ポイント：

- `/` は静的ファイル、`/api/*` は Worker のコードが応答する。その振り分けは誰がどう決めているか。
- メモリ上の配列でデータが安定して残らないのはなぜか（→ Step 4 の動機）。

参考：[Static Assets](https://developers.cloudflare.com/workers/static-assets/)、[Hono on Cloudflare Workers](https://hono.dev/docs/getting-started/cloudflare-workers)

## Step 4：D1 でデータを保存する

やること：

- [x] `npx wrangler d1 create sample-cf-workers-d1-db` で DB を作り、出力された設定を `wrangler.jsonc` の `d1_databases` に追加する（binding 名は `DB`）
- [x] `npx wrangler types` で型定義を生成し、コードから `c.env.DB` が補完されることを確認する
- [x] `npx wrangler d1 migrations create DB create_memos` で SQL ファイルを作り、`CREATE TABLE` を書く
- [x] `npx wrangler d1 migrations apply DB --local` でローカルに適用し、API を D1 に書き換えて `wrangler dev` で確認する（`c.env.DB.prepare(...).bind(...).all()`）

```bash
npx wrangler d1 migrations list DB --local     # 未適用のマイグレーションを確認
npx wrangler d1 migrations apply DB --local
npx wrangler d1 execute DB --local --command "SELECT * FROM memos"   # ローカルの中身を見る
```
- [x] `npx wrangler d1 migrations apply DB --remote` で本番に適用し、デプロイする
- [x] `npx wrangler d1 execute DB --remote --command "SELECT * FROM memos"` で中身を見る

理解ポイント：

- バインディングとは何か。接続文字列やパスワードが不要なのはなぜか。
- `--local` と `--remote` で、それぞれどこの DB を触っているか（ローカルの実体は `.wrangler/` 配下の SQLite ファイル）。
- マイグレーションを「ファイルとして管理し順番に適用する」意味（適用済みかどうかは D1 のどこに記録されるか）。

参考：[D1 Get started](https://developers.cloudflare.com/d1/get-started/)

## Step 5：Drizzle を導入する

やること：

- [x] `npm install drizzle-orm` と `npm install -D drizzle-kit`
- [x] `src/db/schema.ts` に memos テーブルを TypeScript で定義する
- [x] `drizzle.config.ts` を作り、出力先を wrangler のマイグレーションディレクトリに合わせる（`wrangler.jsonc` の `migrations_dir` と一致させる）
- [x] 手書きの `0001_create_memos.sql` を捨てて、Drizzle に最初から作らせる（テストデータしかないので DB を作り直す。理由は [番外編：DB のスキーマ管理](extra-db-schema-management.md) の「ベースライン」）

```bash
rm migrations/0001_create_memos.sql
npx drizzle-kit generate                   # migrations/0000_xxx.sql と migrations/meta/ ができる
rm -rf .wrangler/state/v3/d1               # ローカルの DB を消す
npx wrangler d1 migrations apply DB --local
npx wrangler d1 execute DB --remote --command "DROP TABLE memos; DROP TABLE d1_migrations;"   # 本番の DB を空にする
npx wrangler d1 migrations apply DB --remote
```

- [x] 列を1つ追加（例：`updated_at`）し、`npx drizzle-kit generate` で SQL が生成されることを確認する
- [x] 生 SQL のクエリを Drizzle に置き換える

理解ポイント：

- Step 4 の生 SQL と比べて、何が楽になり何が隠れたか。
- スキーマの「正」は schema.ts と migrations/*.sql のどちらか。

参考：[Drizzle with Cloudflare D1](https://orm.drizzle.team/docs/connect-cloudflare-d1)、[番外編：DB のスキーマ管理](extra-db-schema-management.md)

## Step 6：デプロイの方式を決める（GitHub と CI/CD）

GitHub Actions で「main への push ＝ デプロイ」を作って試したうえで、アプリは手元の `npm run deploy` でデプロイし、git は保管用に適宜 push する方式にした。経緯は [方式検討サマリの「CI/CD」](infra-design-summary.md#cicd) を参照。

やること：

- [x] `.gitignore` に `node_modules/`、`.wrangler/`、`.dev.vars` が入っていることを確認する
- [x] `gh repo create sample-cf-workers-d1 --private --source . --push` で GitHub にリポジトリを作る
- [x] （試行）ダッシュボード（My Profile → API Tokens → Create Token）で、テンプレート「Edit Cloudflare Workers」に D1 の編集権限を足した API トークンを作る。手元では `infra/.secrets/cloudflare.env`（git 管理外）に控える
- [x] （試行）トークンとアカウント ID を、リポジトリの Secrets に登録する（`gh secret set CLOUDFLARE_API_TOKEN`、`gh secret set CLOUDFLARE_ACCOUNT_ID --body "<アカウント ID>"`）
- [x] （試行）`.github/workflows/deploy.yml` を書き、push で Actions が動いて本番に反映されることを確認する（`gh run watch`、`npx wrangler deployments list`）
- [x] （試行）`schema.ts` に列を足して `npx drizzle-kit generate` → push し、マイグレーションも Actions で本番に適用されることを確認する
- [x] 方針転換の後片付け：ワークフローを消す（GitHub の Secrets は、静的サイトなどで使うかもしれないので残した）

```bash
cd ~/claude-sandbox/sample-cf-workers-d1
git rm .github/workflows/deploy.yml
gh secret delete CLOUDFLARE_API_TOKEN
gh secret delete CLOUDFLARE_ACCOUNT_ID
```

- [x] `package.json` の `deploy` スクリプトを、マイグレーションの適用とデプロイをまとめたものに書き換え、`npm run deploy` でデプロイする

```json
"deploy": "wrangler d1 migrations apply DB --remote && wrangler deploy"
```

- （任意）デプロイの履歴にそのときのコミットを残したければ、`wrangler deploy` に `--message "$(git describe --always --dirty)"` を付ける（コミット前の変更を含むと `-dirty` が付く）。今回はシンプルさを優先して付けていない

理解ポイント：

- Cloudflare から見て、手元の `wrangler deploy` と Actions からのデプロイは何が違い、何が同じか。混ぜると何が困るか。
- Workers Builds（Cloudflare が GitHub から取りに行く）と GitHub Actions（GitHub が Cloudflare に送り込む）では、認証情報をどちらに預けているか。
- アプリでは push デプロイをやめ、静的サイトでは使うことにした理由。

参考：[GitHub Actions でデプロイする（Cloudflare）](https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/)、[Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/)

## Step 7：Terraform で Access を管理

題材：`sample-cf-workers-d1.alphajinsei.com/admin` を Access で保護する（保育園ページで手動でやっていることをコード化する）。

やること：

- [x] ダッシュボードで Terraform 用の API トークンを作る（Account / Access: Apps and Policies / Edit など、必要最小限の権限）。`infra/.secrets/terraform-cloudflare.env` に控える
- [x] state 置き場として R2 バケット（例：`tfstate`）を手動で作り、R2 用の API トークンを発行する

```bash
# ダッシュボードで R2 を有効にした（支払い方法を登録した）あとで
npx wrangler r2 bucket create tfstate --location apac
npx wrangler r2 bucket list
```

R2 用の API トークン（S3 互換の Access Key ID と Secret Access Key）は、ダッシュボードの R2 →「Manage API tokens」で、`tfstate` バケットだけに「Object Read & Write」を付けて作る。手元では `infra/.secrets/` に控える。
- [x] `sample-cf-workers-d1/infra/terraform/` に provider と backend を書く（backend は s3 互換設定で R2 を指す）
- [x] 鍵を読み込んで terraform を実行するラッパー `tf.sh` を置き（ターミナルに `CLOUDFLARE_API_TOKEN` を残さないため。残すと wrangler がそれを優先して使い、`npm run deploy` が失敗する）、`.gitignore` に `.terraform/`、`*.tfstate`、`*.tfstate.*` を足す
- [x] `./tf.sh init` → `./tf.sh validate` → `./tf.sh plan` → `./tf.sh apply`

```bash
cd ~/claude-sandbox/sample-cf-workers-d1/infra/terraform
./tf.sh init
./tf.sh validate
./tf.sh plan
./tf.sh apply
```
- [x] `/admin` にアクセスし、ログインを求められることを確認する（ログイン後は、`/admin` のページがないので 404 になる）。`/` や `/api/memos` はログインなしのまま。state が R2 の `tfstate` バケットにできていることも確認する

```bash
curl -sI https://sample-cf-workers-d1.alphajinsei.com/admin | head -3      # 302 で cloudflareaccess.com へ
curl -sI https://sample-cf-workers-d1.alphajinsei.com/api/memos | head -1  # 200 のまま
```
- [x] ダッシュボードで設定を手で変えてから `terraform plan` を実行し、差分（drift）が検出されることを確認する

```bash
./tf.sh plan                 # 「~ update in-place」として手の変更が出る
./tf.sh apply                # コードの状態に戻す（コードを正とする場合）
./tf.sh plan -refresh-only    # 「Terraform の外で変更されたもの」だけを見る（drift の確認用）
```

理解ポイント：

- state とは何か。なぜローカルではなく R2 に置くのか。
- state 用の R2 バケットを Terraform で作らず手動で作るのはなぜか（鶏と卵の問題）。
- wrangler と Terraform の役割分担：同じリソースを両方から管理すると何が起きるか。このプロジェクトでは「Worker・D1・カスタムドメインは wrangler、Access などは Terraform」とする。
- トークンや秘密情報をリポジトリに入れない方法（環境変数、`terraform.tfvars` を `.gitignore` に入れる）。

参考：[Cloudflare Terraform provider](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs)、[Cloudflare と Terraform](https://developers.cloudflare.com/terraform/)、[番外編：Terraform の state と、置き場所](extra-terraform-state.md)

## Step 8：運用の仕組みを足す（見送り）

個人サイトには過剰なので見送った。ログ（`observability`）はひな形で最初から有効になっている。必要になったら、以下を個別に足す。

やること：

- [ ] `wrangler.jsonc` で `"observability": { "enabled": true }` を設定し、ダッシュボードでログを検索する
- [ ] Cron Triggers を設定し、1日1回、古いメモを消す（または件数をログに出す）`scheduled()` を書く
- [ ] UptimeRobot で `https://sample-cf-workers-d1.alphajinsei.com/api/health` を監視し、通知を受け取れるようにする
- [ ] `npx wrangler d1 export DB --remote --output backup.sql` でバックアップを取ってみる。D1 の Time Travel（過去の時点への復元）の仕組みも調べる

理解ポイント：

- 障害時に「何が起きたか」を知る手段はどこにあるか。
- データを誤って消したとき、どこまで戻せるか（RPO）。

## Step 9：振り返りとテンプレート化（Claude と一緒に）

- [x] 各ステップでつまずいた点と、理解ポイントへの答えを `docs/` にメモする（[勉強メモ](study-notes.md) と番外編で済んだ）
- [x] 方式①（静的サイト）のサンプル `sample-cf-workers-static` を作る（Claude が作成。Astro + Workers Static Assets、デプロイは `npm run deploy`）。公開先は https://sample-cf-workers-static.alphajinsei.com

```bash
cd ~/claude-sandbox
npm create astro@latest sample-cf-workers-static -- --template minimal --install --git --yes --skip-houston
cd sample-cf-workers-static
npm install -D wrangler
# wrangler.jsonc（assets だけ・カスタムドメイン・workers_dev: false）と、
# package.json の "deploy": "astro build && wrangler deploy" を書く
npm run deploy
gh repo create sample-cf-workers-static --private --source . --push
```
- [x] 2つのサンプルを public にする（コミットの作者とd1 の `main.tf` に gmail が入っているが、そのまま公開すると判断した。infra は `cf-infra-playbook` として public、作者は noreply）

```bash
gh repo edit alphajinsei/sample-cf-workers-d1 --visibility public --accept-visibility-change-consequences
gh repo edit alphajinsei/sample-cf-workers-static --visibility public --accept-visibility-change-consequences
```
- [x] サービス用テンプレート：別には作らず、2つのサンプルを GitHub の Template repository にした（設定は画面から）。コピーしたあとに書き換える場所は、各サンプルの README の5章

```bash
gh repo create <name> --template alphajinsei/sample-cf-workers-d1 --public --clone
```
- [x] サービス開発側に渡すインフラ設計資料をまとめる：全方式共通は [サービス開発ガイド](service-guide.md)、方式ごとは各サンプルの `README.md`（見出しは6つにそろえる）
- [ ] 以降のサービスのデプロイは Claude に任せる（実際のサービスを作るときに）

次：方式③（OCI）のサンプル。別のセッションで始める。引き継ぎは [handoff-method3-oci.md](handoff-method3-oci.md)
