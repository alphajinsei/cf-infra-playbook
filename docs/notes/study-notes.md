# 勉強メモ

最終更新：2026-11-12

[学習ロードマップ](../roadmaps/) を進める中で出てきた疑問と、その答えを積み上げていくメモ。1つの問いに1つの見出しを立てる。答えは、会話の中で説明した内容を、圧縮せずにそのまま残す（あとから読み返して「当時何を理解したのか」が分かるように）。とくに長い話は番外編（`extra-*.md`）に分けて、答えからリンクする。認証基盤の話は、auth リポジトリの [勉強メモ](https://github.com/alphajinsei/auth/blob/main/docs/notes/study-notes.md)。

## 開発環境（WSL）

### Q. node は WSL 上、npm は Windows 側、という状態になっている。これは自分だけではない、よく知られた事象に見える。ベストプラクティス的な解消法はないのか

WSL ではよく知られた事象。

**なぜ起きるか**

- WSL は、既定で Windows の PATH を Linux の PATH の**後ろ**に足す。そのため Linux 側から `code .` や `explorer.exe` が使える。
- `node` は Linux 側の `/usr/bin/node`（apt で入っていた v18）が先に見つかるので、そちらが使われる。
- Ubuntu では npm が `nodejs` とは別のパッケージになっている。Linux 側に npm がないので、PATH の後ろにある Windows 側の npm まで探しにいってしまう。

**ベストプラクティス**

Microsoft の公式ドキュメント（「Set up Node.js on WSL 2」）が勧めているのは、**WSL の中にバージョン管理ツール（nvm）で Node.js を入れる**方法。こうすると Linux 側の node と npm が PATH の前に来るので、Windows 側の npm は使われなくなる。

もっと強い方法として、`/etc/wsl.conf` に `appendWindowsPath = false` と書いて、Windows の PATH を丸ごと切り離すこともできる。ただし `code .` なども使えなくなり、必要なものを個別に PATH へ足し直すことになるので、最初はやらなくてよい。

**理解の確認**：「wsl 上にインストールしていなくても、なんか使えるな……という状況が起きうる。そうならないよう、wsl 上でも明示的に nodejs をインストールしよう」という理解で合っている。補足すると、困るのは「使えてしまう」ことそのものより、**Linux の道具を使っているつもりで、実は Windows の道具が動いている**こと。Windows 側の npm で WSL 側のファイルを扱うと、動作がとても遅くなったり、Windows 用にビルドされた部品が混ざって Linux で動かなくなったりする。

→ [番外編：Node.js まわりの用語](extra-nodejs-basics.md)

### Q. `sudo apt remove nodejs` は何だったのか。もともと古い Node.js が入っていたのを消したのか。apt より自分で取ってくる方が自然、という意味か

WSL にはもともと apt で入った Node.js v18 があり、それを消した（Ubuntu の初期状態には入っていないので、以前に何かのついでに入ったものと思われる）。

「自分で取ってくる方が自然」という意味ではない。違いは**誰が用意した Node.js を使うか**。

| | apt の nodejs | nvm |
| --- | --- | --- |
| 配っている人 | Ubuntu。OS のバージョンごとに固定されている | Node.js の公式（nodejs.org）がビルドしたもの |
| バージョン | 古い（今回は 18、サポート終了済み） | 好きなものを選べる |
| npm | 別パッケージ（今回は入っておらず、Windows 側が使われていた） | 一緒に入る |
| 置き場所 | `/usr/bin`（全ユーザー共通、sudo が要る） | `~/.nvm`（自分専用、sudo 不要） |

開発用の Node.js は nvm で入れるのが一般的。古い v18 が残っていると「どちらが動いているのか」が紛らわしくなるので消した。

### Q. gh と Terraform のインストールでは、何をやっているのか。署名鍵とは何か（相手の公開鍵、あるいはサーバー証明書のようなものか）。gpg とは何か。手元に gpg 鍵を登録するとは、具体的にどうやるのか

流れは「配布元の署名鍵を置く → 配布元を apt に登録する → apt で入れる」。Ubuntu 標準の apt には入っていない（か古い）ので、配布元を足している。

**署名鍵とは**

**配布元の公開鍵**。サーバー証明書と目的は似ている（相手が本物かを確かめる）が、確かめる対象が違う。

- サーバー証明書（TLS）：「いま通信している**サーバー**が本物か」を確かめる。認証局（CA）が保証する。
- 署名鍵（GPG）：「受け取った**中身（パッケージ）**を本物の配布元が作ったか」を確かめる。保証する第三者はいない。この鍵を信じると自分で決めて置く。

中身を確かめる仕組みなので、途中のミラーサーバーやキャッシュがすり替えられても検知できる。

**gpg と、検証の仕組み**

GPG（GNU Privacy Guard）は、OpenPGP という署名・暗号の規格を実装したツール。鍵の形式の名前でもある。apt の検証は次のように進む。

1. 配布元は、パッケージ一覧の目次（`Release` ファイル）を**秘密鍵で署名**して公開している。目次には、各パッケージ一覧のハッシュ値が載っている。
2. `apt update` で目次を取ってくると、手元の**公開鍵（署名鍵）**で署名を検証する。
3. 目次のハッシュ値でパッケージ一覧を確かめ、一覧に載っているハッシュ値で各 `.deb` ファイルを確かめる。

署名を1つ確かめれば、ハッシュ値の連鎖で全パッケージが本物だと分かる。yum で「GPG 鍵がなくて失敗する」のは、2 の時点で検証に使う公開鍵が手元にないから（yum は rpm ファイル1つ1つにも署名がある）。配布元が鍵を更新したときにもよく起きる。

**手元への登録の仕方**

置き場所は2か所に分かれる。`sources.list.d` に置くのは鍵ではなく、配布元の登録。

| ファイル | 中身 |
| --- | --- |
| `/etc/apt/keyrings/githubcli-archive-keyring.gpg` | 鍵そのもの |
| `/etc/apt/sources.list.d/github-cli.list` | 配布元の URL と「この配布元はあの鍵で検証せよ」という指定 |

```
deb [arch=amd64 signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main
```

`signed-by=` で、**この鍵はこの配布元にしか使わない**と結び付けている。昔は `apt-key add` で鍵を全体に登録する方式だったが、ある配布元の鍵で別の配布元のパッケージを偽装できてしまうので廃止された。Terraform の手順の `gpg --dearmor` は、テキスト形式の鍵（`-----BEGIN PGP PUBLIC KEY BLOCK-----`）を、apt が読むバイナリ形式に変換している。

→ [番外編：apt の署名鍵と GPG](extra-apt-signing.md)

### Q. apt の署名鍵は、SSH の authorized_keys や known_hosts のような概念か

いい対応づけ。向きでいうと **known_hosts のほうが近い**。

- known_hosts：クライアント（受け取る側）が、サーバー（相手）の公開鍵を置く
- authorized_keys：サーバーが、ログインしてくるユーザーの公開鍵を置く（向きが逆）
- apt の署名鍵：自分（受け取る側）が、配布元の公開鍵を置く → known_hosts と同じ向き

違うのは、SSH が「接続の相手」を確かめるのに対し、apt は「ファイルの中身」を確かめる点。最初の1回は信じるしかない、というところも同じ（SSH の初回接続で出る「この鍵を信じますか？」と同じ）。

## git

### Q. `.gitattributes` とは何か

**リポジトリの中のファイルを git がどう扱うかの設定**で、リポジトリと一緒にコミットされる。

```
* text=auto eol=lf
```

- `*`：すべてのファイルに対して
- `text=auto`：テキストファイルかどうかは git が自動で判定する（画像などのバイナリには何もしない）
- `eol=lf`：テキストファイルの改行コードは LF にそろえる

**なぜ要るか**：Windows の改行は CRLF、Linux の改行は LF。改行コードが混ざると、たとえばシェルスクリプトが `bad interpreter: /bin/bash^M` というエラーで動かなくなる。Windows で編集したファイルが紛れ込む可能性があるので、そろえておく。

**git config の `core.autocrlf` との違い**：`core.autocrlf` は**自分のパソコンの設定**。`.gitattributes` は**リポジトリの設定**なので、誰がどのパソコン（Windows 側でも）で clone しても同じルールが効く。リポジトリに書いておくほうが確実。`git ls-files --eol` で見ると、全ファイルが LF（`i/lf w/lf`）になっていた。

### Q. サンプルアプリは方式ごとに作るので、フォルダが増えていく。どう置くのがよいか。`npm create cloudflare` がサンプルの中でも `git init` するのはなぜか

**置き場所と名前**：`~/claude-sandbox/` の直下に、`sample-<基盤>-<構成>` という名前で並べる。

```
~/claude-sandbox/
├── infra/                    方針と学習の記録
├── sample-cf-workers-d1/     方式②
└── sample-oci-vm-docker/     方式③
```

- フォルダ名と GitHub のリポジトリ名を同じにできる。GitHub にはフォルダの階層がないので、`sample-` で始めておくと一覧で並んで見つけやすい。
- 基盤の名前（cf / oci など）を先頭に入れる。「worker-d1」だけだと、学習の枠が増えたときにどこの話か分かりにくい。
- 小文字とハイフンだけにする。フォルダ名がそのまま Worker の名前（`wrangler.jsonc` の `name`）になり、Worker の名前は小文字・数字・ハイフンしか使えない（`worker-D1` の大文字は通らない）。

**infra の中に置かなかった理由**：git リポジトリの中で別のプロジェクトを `git init` すると、**入れ子**になる。infra 側で `git add .` すると警告が出て、子のリポジトリは中身のない参照としてしか記録されない。サンプルは単独のリポジトリとして GitHub に push する予定だったので、infra の隣に置くことにした。infra は「方針と学習の記録」、サンプルは「サービスそのもの」とはっきり分かれ、設計方針の「1サービス＝1ディレクトリ」にも合う。

**git init する件**：そういうもの。`npm create cloudflare` のような、プロジェクトのひな形を作るツール（`create-next-app` など）の多くは、「git を使いますか？」と聞いて、Yes なら `git init` と最初のコミットまでやる。「プロジェクトを作ったら、たいていリポジトリにする」という前提で作られているから。No と答えれば git はやらない。

### Q. `gh repo create` は何をしているのか

手元の既存のリポジトリから、GitHub にリポジトリを作り、`origin` の登録と push まで一度にできる（ブラウザで作る必要がない）。

```bash
gh repo create sample-cf-workers-d1 --private --source . --push
```

| オプション | 意味 |
| --- | --- |
| `--private` | 非公開のリポジトリにする（`--public` なら公開） |
| `--source .` | いまのフォルダ（既存の git リポジトリ）を元にする |
| `--push` | 作ったリポジトリを `origin` として登録し、そのまま push する |

gh は、PR の作成（`gh pr create`）や Issue の操作もコマンドでできる。

### Q. noreply メールアドレスとは何か。公開リポジトリのコミットの作者をどうするか

noreply メールアドレスは、GitHub が各アカウントに1つずつ用意している、コミット専用のダミーのアドレス（`<ID>+<ユーザー名>@users.noreply.github.com`）。

- このアドレスには、メールは届かない。git のコミットに書くためだけのアドレス。
- GitHub は、このアドレスで作ったコミットをアカウントに結び付ける。コミットの横にアイコンが出て、草（コントリビューション）にも数えられる。中身のアドレスを隠したまま、本人のコミットとして扱われる。
- git のコミットには作者のメールアドレスが必ず記録され、公開リポジトリでは誰でも `git log` で見られるので、本物のアドレスを隠したいときに使う標準的な方法。

infra（GitHub では `cf-infra-playbook`、公開）は、リポジトリ単位で `git config user.email` に noreply を設定し、公開前の既存のコミットも `git filter-branch --env-filter` で作者を書き換えた。まだどこにも push していなかったので、履歴を書き換えても他に影響はなかった（コミットの ID は変わる）。GitHub の Settings → Emails の「Keep my email addresses private」と「Block command line pushes that expose my email」を有効にすると、うっかり本物のアドレスで push するのを GitHub 側で止められる。

その後、gmail のアドレスは秘匿性の高い情報ではないので、oci-platform 以降の新しいリポジトリはそのままでよいことにした。

### Q. 新しいサービスを始めるときのテンプレートは、サンプルとは別に作るのか

作らない。サンプルが薄いうちは、テンプレートを別に作るとほとんど重複し、2つを同じ状態に保ち続ける手間のほうが大きくなる。

そこで、**サンプルをそのままテンプレートとして使う**。サンプルのリポジトリを GitHub の Template repository に設定すると（リポジトリの Settings で有効にする）、次の1コマンドで、サンプルを丸ごとコピーした新しいリポジトリができる。履歴は引き継がれない。

```bash
gh repo create <新サービス名> --template alphajinsei/sample-cf-workers-d1 --public --clone
```

そのうえで、各サンプルの README に「コピーしたあと書き換える場所の一覧」（サービス名、ドメイン、DB の名前、Terraform の state のキーなど）を書いておく。特に Terraform の state のキーは、変え忘れるとサンプルの state を上書きしてしまう。サンプルが育てば、テンプレートも一緒に育つ。

## Claude Code

### Q. サンプルのフォルダを infra の外に置くと、VSCode で開いている infra とずれてしまう。どうするか

ずれるのは2つ。

- VSCode のエクスプローラー：サンプルのファイルが見えない
- Claude Code：作業ディレクトリが infra なので、外のフォルダの編集は毎回許可を求められる。さらに、**メモリはプロジェクトのフォルダごとに分かれている**（`~/.claude/projects/<パス>/memory/`）ので、サンプル側で Claude を開き直すと、それまでの約束が読み込まれない

そこで、作業の拠点は infra のままにして、サンプルを「追加のフォルダ」として登録した。

1. VSCode：「ファイル → フォルダーをワークスペースに追加」でサンプルを足す。1つのウィンドウに複数のフォルダが並び、git もそれぞれ別に表示される。
2. Claude Code：infra の `.claude/settings.json` の `permissions.additionalDirectories` にサンプルのフォルダを登録する。infra のセッションのまま、サンプルのファイルも普通に読み書きできる。サンプルが増えたら、この配列に足していく。

```json
{
  "permissions": {
    "additionalDirectories": ["/home/alphajinsei/claude-sandbox/sample-cf-workers-d1"]
  }
}
```

プロジェクトのルール（「コマンドをロードマップに残す」など）は、メモリではなく CLAUDE.md に書いておくほうが確実。CLAUDE.md はリポジトリにあるので、どのフォルダで開いても、そのリポジトリのルールが読まれる。

### Q. Claude が Node.js のコマンドを実行するとき、なぜ `bash -ic` を使うのか

Claude が使う Bash は nvm を読み込まないので、`npx` が Windows 側に解決されてしまう。作業の途中でこれに気づき、CLAUDE.md に「Claude が Node.js のコマンドを実行するときは `bash -ic` を使う」と書いた（`-i` で対話シェルとして起動すると、nvm の設定が読み込まれる）。ユーザー自身のターミナルには関係ない。

## Node.js / npm

### Q. node.js、npm、npx、nvm は、それぞれ何か。fnm や Node 22 とは何か。nvm をよく聞くが、あえて fnm を採用した理由はあるのか

**4つの関係**

| 名前 | 何者か | たとえ |
| --- | --- | --- |
| Node.js | JavaScript をパソコンやサーバーの上で動かすプログラム。もともと JavaScript はブラウザの中でしか動かなかった | エンジン |
| npm | Node.js 用のライブラリを、ネット上の置き場（npm レジストリ）から取ってきて入れる道具。Node.js に付いてくる | 部品の通販アプリ |
| npx | npm に付いてくる「入っている道具を、その場で実行する」コマンド。`npx wrangler login` は「このプロジェクトに入れた wrangler を動かす」という意味 | 買った工具をすぐ使うボタン |
| nvm | Node.js 自体のバージョンを入れたり切り替えたりする道具 | エンジンの載せ替え係 |

使う順番でいうと、nvm で Node.js を入れる → npm で wrangler や Hono を入れる → npx で wrangler を実行する。

**fnm と Node 22**

- fnm（Fast Node Manager）：nvm と同じ役割（Node.js のバージョン管理）をする別の道具。Rust で書かれていて速い。
- Node 22：Node.js のメジャーバージョン。偶数のバージョンが長期サポート版（LTS）で、数年間はバグや脆弱性の修正が続く。2026-11 の時点では 24 が現役の LTS で、22 はメンテナンス期間（2027-04 にサポート終了）。これから始めるなら 24。

**なぜ fnm にしていたか**：決定的な理由はなかった。fnm はターミナルを開くたびに少し時間がかかる nvm の問題がなく、プロジェクトのフォルダに入ると指定のバージョンに自動で切り替えられる。nvm は圧倒的に有名で、Microsoft の WSL 向け公式手順もこちらを使い、検索すると情報が多い。今回の使い方（Node は1バージョンだけ）ではどちらでも差はほぼない。

**nvm にした**：「情報がネットに多いということは、AI で検索した場合にヒットする情報も多いということ」なので、nvm を選んだ。

→ [番外編：Node.js まわりの用語](extra-nodejs-basics.md)

### Q. `npm create cloudflare@latest sample-cf-workers-d1` は何をしているのか。cloudflare というモジュールを npm で取り寄せて、インストールしているのか。このコマンドだけでデプロイまでされたように見えた

`npm create xxx` は「`create-xxx` というパッケージを取ってきて、その場で実行する」という決まった書き方。中身は `npx create-cloudflare@latest sample-cf-workers-d1` と同じ。

1. npm が `create-cloudflare`（通称 C3）というパッケージの最新版（`@latest`）をダウンロードし、一時的な場所で実行する。プロジェクトにはインストールされない。
2. C3 が質問をして、答えに合ったひな形のファイルを `sample-cf-workers-d1/` に書き出す。
3. C3 がそのフォルダで `npm install` を実行し、wrangler などの依存パッケージを `node_modules/` に入れる。
4. 「git を使う」と答えたので、`git init` して最初のコミットまでする。

C3 は**ひな形を作るためだけの使い捨てのツール**。できあがったプロジェクトの中で働くのは、3 で入った wrangler。

**デプロイはされていなかった**：Cloudflare 側に問い合わせると「This Worker does not exist on your account」と返った。最後に URL や「SUCCESS」のような表示が出たので、デプロイされたように見えただけ。デプロイは、このあとの `npx wrangler deploy` で初めて行う。

### Q. `npm create` という npm のオプションがあるように読めたが、実際には `npx create-cloudflare` なのか。npm が Cloudflare の記法に合わせている、というのがよく分からない

逆。**npm がルールを決めていて、Cloudflare がそのルールに合わせている。**

`npm create` は npm に最初からあるサブコマンドで、`npm init` の別名。npm は、こういう読み替えのルールを決めている。

| 打つコマンド | npm が実際に実行するもの |
| --- | --- |
| `npm create foo` | `npx create-foo` |
| `npm create @foo` | `npx @foo/create` |

npm 自身は Cloudflare のことも Vite のことも知らない。「`create-` を前に付けた名前のパッケージを探して実行する」だけ。ひな形のツールを配る側が、このルールに合わせてパッケージ名を `create-` で始めて公開している（`create-cloudflare`、`create-vite`、`create-next-app`）。

引数なしの `npm init` は、`package.json` を1つ作るだけ。こちらがちょうど `git init` に相当する。`npm init <名前>` にすると、ひな形のツールを呼び出す高機能版になる。

**理解の確認**：「npm の記載の規則に、各モジュールの作成者が従っている」で合っている。「git init の数段レベルが高い版」という理解も合っている。（「その場で実行する」という意図が見える `npx create-xxx` の方が分かりやすい、という感想だった。）

## TypeScript

### Q. TypeScript に全く触ったことがない。`src/index.ts` の中身を解説してほしい

TypeScript は、**JavaScript に「型」の注釈を足した言語**。型は書き間違いをエディタで見つけるためだけのもので、実行前に取り除かれて普通の JavaScript になる。

```ts
export default {
	async fetch(request, env, ctx): Promise<Response> {
		return new Response("Hello World!");
	},
} satisfies ExportedHandler<Env>;
```

- **`export default { ... }`**：このファイルの「代表として外に渡すもの」を決める。ここでは `{ }` で囲んだオブジェクト（いくつかの関数をまとめた入れ物）を渡している。Cloudflare の実行環境はこのファイルを読み込み、渡されたオブジェクトの中から決まった名前の関数を探して呼び出す。
- **`fetch(request, env, ctx)`**：その「決まった名前」の1つ。**HTTP リクエストが届くたびに、実行環境がこの関数を呼び出す。**ほかに `scheduled`（定期実行のときに呼ばれる）などがある。

  | 引数 | 中身 |
  | --- | --- |
  | `request` | 届いたリクエスト（URL、メソッド、ヘッダー、本文） |
  | `env` | `wrangler.jsonc` で設定した接続先や変数。D1 は `env.DB` として使う |
  | `ctx` | 応答を返したあとも処理を続けたいとき（ログの送信など）に使う道具 |

- **`async` と `Promise<Response>`**：`async` は「この関数は時間のかかる処理（DB への問い合わせなど）を待つことがある」という印。async の関数は、結果を **Promise**（「あとで結果が届く」という引換券）に包んで返す。`: Promise<Response>` は「最終的に Response が届く」という戻り値の型。
- **`return new Response("Hello World!")`**：「Hello World!」という本文の HTTP レスポンスを作って返す。ステータスは、指定しなければ 200。`Request` や `Response` は、ブラウザの JavaScript でも使われている Web 標準の部品。
- **`satisfies ExportedHandler<Env>`**：「このオブジェクトが、Cloudflare の決めた形（`ExportedHandler`）に合っているか確かめて」という指示。`fetch` の名前を書き間違えたり、Response 以外を返したりすると、エディタで赤線が出る。`request`、`env`、`ctx` に型を書いていないのに型が付くのも、このおかげ。`Env` は `worker-configuration.d.ts` に、`wrangler.jsonc` から自動生成されている。

この Worker には「サーバーを起動して待ち受ける」という処理がどこにもない（Workers の節の問いにつながる）。

### Q. Hono のコード（`app.get("/api/health", (c) => c.json({ ok: true }))`）は、それぞれ何を意味しているのか

```ts
import { Hono } from "hono";
const app = new Hono<{ Bindings: Env }>();
app.get("/api/health", (c) => c.json({ ok: true }));
export default app;
```

| コード | 意味 |
| --- | --- |
| `import { Hono } from "hono"` | 入れた hono パッケージから、`Hono` という部品を取り込む |
| `new Hono<{ Bindings: Env }>()` | アプリ本体を作る。`<{ Bindings: Env }>` は TypeScript の型の指定で、「`env` の中身は `Env` 型」と教えている。`c.env.DB` を書いたとき、エディタが補完してくれる |
| `app.get("/api/health", ...)` | 「GET で `/api/health` に来たら、この関数を実行する」という登録 |
| `(c) => c.json({ ok: true })` | `=>` は関数の短い書き方（アロー関数）。`c`（context）には、リクエストや `env` がまとめて入っている。`c.json(...)` で JSON の応答を返す |
| `export default app` | Hono のアプリ自体が `fetch` 関数を持っているので、そのまま渡せる。`export default { fetch(...) }` と同じ役割 |

### Q. メモの API（メモリ上の配列）のコードで、新しく出てくる書き方は何か

```ts
type Memo = { id: number; text: string; createdAt: string };

const memos: Memo[] = [];
let nextId = 1;

app.get("/api/memos", (c) => c.json(memos));

app.post("/api/memos", async (c) => {
  const { text } = await c.req.json<{ text: string }>();
  const memo: Memo = { id: nextId++, text, createdAt: new Date().toISOString() };
  memos.push(memo);
  return c.json(memo, 201);
});

app.delete("/api/memos/:id", (c) => {
  const id = Number(c.req.param("id"));
  const index = memos.findIndex((m) => m.id === id);
  if (index === -1) return c.json({ error: "not found" }, 404);
  memos.splice(index, 1);
  return c.body(null, 204);
});
```

| コード | 意味 |
| --- | --- |
| `type Memo = { ... }` | 「メモ」の形を TypeScript の型として定義する（id は数値、text は文字列…） |
| `const memos: Memo[] = []` | メモを入れる空の配列。`Memo[]` は「Memo の配列」という型 |
| `let nextId = 1` | 次に振る ID。`const` は再代入できない、`let` はできる |
| `async (c) => { ... await ... }` | `await` は「Promise の結果が届くまで待つ」。リクエストの本文を読むのは時間がかかる処理なので待つ |
| `c.req.json()` | リクエストの本文を JSON として読む |
| `c.json(memo, 201)` | JSON を返す。201 は「作成した」というステータス |
| `"/api/memos/:id"` と `c.req.param("id")` | URL の `:id` の部分を変数として受け取る。`/api/memos/3` なら `"3"` |
| `c.body(null, 204)` | 本文なしで返す。204 は「成功、返す中身はない」 |

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
- Bastion は、VM の 22 番に最長3時間だけ通れるセッションを作る、OCI の踏み台。VM に公開 IP が要らない。いざというときは、コンソール接続（シリアルコンソール）でネットワークを通らずに入れる。SSH は2段（Bastion へ、そのトンネルの中で VM へ）で、手でつなぐ手順は → [番外編：OCI Bastion で VM に SSH する](extra-oci-bastion.md)
- `terraform output` は、apply のときに state に保存された output の値を読んで表示するだけ（API には問い合わせない）。`-raw` は文字列を引用符なしで出す（シェルの変数に入れるとき）。アプリ側の Terraform は `terraform_remote_state` で同じ値を読む。
- Vault は金庫、Key は金庫に据え付けた暗号化の機械（外に出せず、中身も読めない。暗号化と復号を頼めるだけ）、Secret は金庫に預けた書類（Key で暗号化して保管され、権限があれば取り出せる）。Secret を読むと Vault が裏で Key で復号する。AWS で言うと Key が KMS、Secret が Secrets Manager。VM には `read secret-family`（読むだけ）を与え、Key を直接使う権限は渡していない。
- Bastion のセッションの鍵は RSA だけ。ed25519 を渡してもセッションは作れてしまうが、接続で `Permission denied (publickey)` になる。セッションの名前（display name）も英字で始める（数字で始まると InvalidParameter）。
- 無操作で数分たつと Bastion 経由の ssh が切れる。ssh_config に `ServerAliveInterval 60` を書くと、手元の ssh が60秒ごとに keepalive を送り、Bastion にも通信が流れるので切れなくなる。サーバ側の変更は要らない。`ssh_config` は Terraform の `local_file` がテンプレート（`ssh_config.tftpl`）から書き出す手元のファイル。apply しても OCI には何もせず、そのファイルを書き直すだけ。直接編集すると次の apply で消えるので、テンプレートを直す。
- OCI が塞いでいるのは外へ出す 25 番だけ。自前のメールサーバーは、受信は 25 番で直接受け、送信は Email Delivery に 587 番で中継すると、申請も IP の評判の問題も避けられる。
- Cloudflare の DNS でプロキシ（オレンジの雲）を有効にすると、名前解決で返るのは Cloudflare のエッジの IP になる。利用者がつなぐのは Cloudflare で、Cloudflare が改めてオリジン（OCI の LB）につなぎ直す。そのため LB に来るのは Cloudflare の IP だけで、NSG でそこに絞れば、LB の IP を直接たたく迂回を防げる。Origin 証明書はブラウザに信頼されず Cloudflare だけが信頼するので、プロキシを外すと HTTPS も通らなくなる。
- Cloudflare とオリジンの間の暗号化は、SSL/TLS の暗号化モードで決まる。Flexible は Cloudflare → オリジンが HTTP、Full は HTTPS だが証明書を検証しない、Full (strict) は HTTPS で証明書も検証する。③は Full (strict) で、LB が受けるのは 443番だけ。モードはゾーン全体の設定だが、Workers・Pages はエッジで動くので、①②では関係がなかった。
- CDN の用語では、利用者 → **エッジ**（利用者に近い Cloudflare の拠点）→ **オリジン**（コンテンツが生まれる大元。③では OCI の LB）と呼ぶ。HTTPS は2区間あり、証明書も別物。
  - エッジ証明書（利用者 → Cloudflare）：Cloudflare が公的な CA（Google Trust Services、Let's Encrypt など）から自動で取り、自動で更新する。ブラウザが内蔵するルートで検証できる。①②のサイトもこれを使っている。
  - Origin 証明書（Cloudflare → オリジン）：Cloudflare が自分の CA（Origin CA）で無料で発行し、オリジン側（OCI の LB）に置く。信頼するのは Cloudflare だけ。公的な CA のエコシステムの外なので、有効期限の短縮のルール（2026年3月から200日など）の外にあり、15年で作れる。ブラウザに直接見せると警告になる。→ [方式検討：OCI](../design/oci.md)
- Wrangler には任意の DNS レコードを作るコマンドがない（②でレコードができたのは、Worker の独自ドメインを登録した副作用）。CLI で DNS を触るなら、Cloudflare 公式の汎用 CLI `cf`（npm の `cf`、beta）の `cf dns records create`。中身は Cloudflare API そのもので、同じ JSON を curl で送っても同じことができる。
- DNS のワイルドカード（`*.example.com`）は、個別のレコードがある名前には効かない（個別のほうが優先）。正規表現は書けず、一番左のラベル全体が `*` の形だけ。Cloudflare の無料のエッジ証明書は1階層（`*.example.com`）までなので、`x.oci.example.com` のような2階層は有料（Advanced Certificate Manager）になる。
- Full と Full (strict) の差は「つないだ相手が本物か」を確かめるかどうかだけ。どちらも暗号化はされる。Full だと、DNS の IP が間違っていたり、LB を作り直して古い IP が他人に渡ったりしても、Cloudflare はそこへ黙ってつなぐ。strict ならエラー（526）で止まる。Origin 証明書を使えば、どちらのモードでもコードは同じ。
- 暗号化モードは、Configuration Rules でホスト名ごとに上書きできる（Free プランでも使える）。ゾーン全体を変えずに、一部のホスト名だけ strict にすることもできる。
- Full でも Full (strict) でも、Cloudflare は利用者と同じプロトコルでオリジンにつなぐ（http なら 80番）。LB が 443番しか受けないなら、「Always Use HTTPS」を有効にして、エッジで https に転送させる。
- プロキシの残存リスク：Cloudflare は中身を平文で見る。また、NSG で Cloudflare の IP に絞っても、ほかの Cloudflare 利用者が自分のゾーンからこちらの IP に向ければ、Cloudflare 経由で届いてしまう。本気で防ぐなら Authenticated Origin Pulls（Cloudflare → オリジンの mTLS）を使う。
- プロキシをやめて DNS だけにする場合、証明書はブラウザが信頼するもの（Let's Encrypt など）が要り、更新の仕組み（ACME と LB への差し替え）を自分で持つことになる。LB も世界中に公開することになり、WAF や DDoS 対策もなくなる。
- Always Free の MySQL HeatWave（`MySQL.Free`）は、作成時にバックアップの設定（`backup_policy`）を指定できない（`Cannot specify backup policy ... for an Always Free DB system`）。OCI の既定の自動バックアップになる。
- MySQL HeatWave は、サブネットへの VNIC の作成と NSG への参加を、DB システム自身の権限（リソース・プリンシパル）で行う。管理者の権限は使われないので、`Allow any-user to {VNIC_CREATE, ...} ... where request.principal.type='mysqldbsystem'` のポリシーが別に要る。ないと DB システムが FAILED になり、出るのは「AuthorizationFailed」だけ。原因は Audit のログで、自分以外（authType が resource）の 404 を探して分かった（`CreateInternalVnic`）。
- pipx は、Python 製のコマンドを1つずつ専用の venv に入れ、コマンドだけを `~/.local/bin` に置く道具。Ubuntu 24.04 はシステムの Python への `pip install` を禁止している（PEP 668）ので、apt にない Python 製の CLI（oci CLI など）は pipx で入れる。Ansible の `community.general.pipx` モジュールは pipx 1.7 以上が前提で、Ubuntu 24.04 の pipx（1.4）では動かない。`command: pipx install ...` に `creates:`（このファイルがあれば実行しない）を付けて、何度流しても同じ状態を保った。
- Terraform の state は、公式の `oci` backend で Object Storage に置ける（ロックあり。S3 互換の方法は非推奨）。
- A1 は Arm なので、x86 の WSL でビルドするイメージは `docker buildx --platform linux/arm64` でクロスビルドする。③では使わず、ソースを送って app の上でビルドする。手元で確かめたイメージ（x86）は、どのみち本番（arm64）にそのまま持っていけない。
- OCI LB のホスト名による振り分けは、「ホスト名」（仮想ホスト名の登録）と「リスナー」（`hostname_names` にそれを付ける）の2段。どのホスト名にも合わないリクエストは、ホスト名を持たないリスナーが受ける。コンソールでは LB の詳細の左のメニューの「ホスト名」。
- MySQL の公式イメージのヘルスチェックを `mysqladmin ping -h localhost` にすると、初期化中の仮のサーバー（ネットワークを閉じて、ソケットだけで動く）でも通ってしまい、アプリが早く起きてテーブルの作成とぶつかる。`-h 127.0.0.1`（TCP）で聞く。
- HeatWave はパスワードの規則（validate_password）が有効で、大文字・小文字・数字・記号をそれぞれ1文字以上含まないと `ERROR 1819` になる。Terraform の `random_password` は、`min_upper` などを付けないと記号が1つも入らないことがある。
- `ssh host bash -s < script.sh` でスクリプトを流し込むと、途中で標準入力を読むコマンド（`docker compose run`、`ssh` など）が、スクリプトの残りを食べてしまう。bash は入力が尽きたと見て、エラーなしで終わる。スクリプトは先にファイルで送って実行するか、そのコマンドに `-T` や `< /dev/null` を付ける。
- MySQL には `RETURNING` がない。Drizzle では、挿入した行の ID を `$returningId()` で受け取る。
- `docker compose` は `compose.override.yaml` を自動で重ねる。手元だけの MySQL や接続情報をそこに書き、本番には送らない。
- OCI のアラームは、メトリクスの系列（ディメンションの組み合わせ）ごとに判定する。`groupBy(backendSetName)` のように書けば、バックエンド・セット（＝アプリ）が増えてもアラーム1つで全部を見られ、`is_notifications_per_metric_dimension_enabled` で系列ごとに別のメールになる。
- 閾値のアラーム（CPU > 80 など）は、VM が止まるとデータが来なくなり、何も言わなくなる。止まったことは `absent()`（一定時間メトリクスが届かないと 1 になる）のアラームで知る。VM の CPU やメモリのメトリクスは、Ubuntu のイメージにも入っている Oracle Cloud Agent が送る。
- 監視は「打つ手があるもの」に絞る。Always Free では拡張の余力がないので、CPU の高騰は見ず、止まったこと（アプリと VM）だけを通知する。
- LB のアクセスログとエラーログは、既定では記録されない。Logging の「ログ」（ソースが LB のサービス・ログ）を作ると記録が始まる。コンソールの LB 作成ウィザードの「ロギングの管理」は、これを裏で作っているだけ。Terraform では `oci_logging_log` を自分で書く。エージェントは要らない。
- `docker logs <コンテナ名>` と `docker compose logs <サービス名>` は同じログを見る。compose で作ったコンテナの名前は `<プロジェクト>-<サービス>-1` で、`docker compose logs` は `compose.yaml` のあるディレクトリでサービス名から引き、複数のサービスを行頭に名前を付けて混ぜて出す。
- OCI の Ubuntu のイメージは `/etc/default/netfilter-persistent` で `IPTABLES_RESTORE_NOFLUSH=yes` にしている。`netfilter-persistent reload` は今のルールを消さずに後ろに足すので、古い `REJECT` が先に残り、足した ACCEPT が効かない（`No route to host` になる）。ファイルを直したら `iptables-restore < /etc/iptables/rules.v4`（フラッシュしてから読む）か再起動。`iptables -S` で実際に効いている順番を見る。
- メールの基礎（登場人物、封筒と便箋、ポート、SPF・DKIM・DMARC の全体像とアライメント）→ [番外編：メールの仕組み](extra-mail-basics.md)
- メールサーバーを組みながら分かったこと（Postfix の役割と設定、届いたメールのヘッダー、証明書の役割、ACME と自動更新、Dovecot との分担）→ [番外編：メールサーバーを組む](extra-mail-build.md)
- メールの流れを図にして分かったこと（Postfix はルーターに近い、ポートと暗号化は別の軸、認証の3種類、経路ごとのプロトコル・ポート・認証）→ [番外編：メールの流れを図にして分かったこと](extra-mail-diagram.md)
- サブドメインとパスの振り分けの比較：パスにすると、アプリ全体に base path の設定が要り、Cookie などのオリジンも共有になる。サブドメインなら Origin 証明書 `*.alphajinsei.com` 1枚で足り、方式をまたいで DNS の向け先を変えるだけで移せる。→ [方式検討：OCI](../design/oci.md)

## 認証・認可

- 分野は「あなたは誰か（認証）」「何をしてよいか（認可）」「誰がいるのか（ID 管理）」の3つの問いと、認証の結果を持ち回るセッションに分かれる。FIDO2 は認証の要素、OAuth 2.0 は委譲された認可（認証が欲しいなら、その上の OIDC）、LDAP はディレクトリ（名簿）で、地図の上では別々の場所にある。→ [番外編：認証・認可の全体像](https://github.com/alphajinsei/auth/blob/main/docs/notes/extra-auth-overview.md)（認証基盤の学習は auth リポジトリに移した。以後のメモもそちら）

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
