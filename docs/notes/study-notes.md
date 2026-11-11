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

### Q. `wrangler.jsonc` の name はワーカーの名前、main はエントリポイント的な概念、compatibility_date はタグか

`name` と `main` はそのとおり。`name` は、workers.dev の URL やダッシュボードでの表示名にも使われる。

`compatibility_date` はタグ（自分のコードの版）ではない。**Cloudflare の実行環境の「どの日付時点の動き方に合わせるか」を固定する設定**。Cloudflare は実行環境を毎日のように更新していて、その中には動作が変わる修正もある。日付を固定しておけば、その日より後の動作変更は自分の Worker には適用されず、勝手に壊れることがない。新しい動作を使いたくなったら、自分で日付を進める。API のバージョン指定（「2026-10-23 版の API を使う」）に近い考え方。

### Q. `wrangler dev` は、デプロイ前の確認として、ローカルで Node.js を上げて確認しているだけか

ローカルで確認している、という点は合っている。ただ、**動いているのは Node.js ではない。**Cloudflare の本番と同じ実行環境の **workerd** を、手元で起動している。Node.js が動かしているのは wrangler という CLI ツールだけで、Worker のコード自体は workerd の中で動く。

workerd は Node.js とは別物なので、Node.js 専用の機能（ファイルを読む `fs` など）はそのままでは使えない。逆に、ローカルと本番で動きがほぼ一致する。

### Q. Worker には「起動しっぱなしのサーバー」がないとは、どうなっているのか。AWS Lambda のように、リクエストが来たときだけ動くのか

そのとおりで、Lambda と同じく**リクエストが来たときにだけ動く**。違うのは、動かす単位。

| | AWS Lambda | Cloudflare Workers |
| --- | --- | --- |
| 動かす単位 | 関数ごとに小さな VM（microVM）を起動する | 1つのプロセスの中に、多数の Worker が **V8 isolate**（ブラウザのタブのように仕切られた区画）として同居する |
| 起動の速さ | 初回は数百 ms〜数秒（コールドスタート） | 数 ms。ほぼ気にならない |
| 動く場所 | 選んだリージョン | 利用者に一番近い、世界中の Cloudflare のデータセンター |

リクエストが届くと、近くのデータセンターで isolate を用意し（直前に使ったものが残っていれば再利用し）、`fetch` 関数を呼ぶ。`export default` で関数を渡すだけで、`listen` のような待ち受けの処理がないのはこのため。

### Q. V8 isolate とはそもそも何か。VM ともコンテナとも違う仮想化技術と聞いた。動作環境（VM ならハイパーバイザ、コンテナならコンテナランタイム）は何か。リクエストはエッジに着いたあと、どこに飛ぶのか

3段に分けて答える。

**① V8 とは**

V8 は、Google が Chrome のために作った JavaScript エンジン。JavaScript のコードを読み、機械語に変換して実行する「部品」。「JavaScript はブラウザで動くもの」というイメージは歴史的には正しいが、V8 自体はブラウザから切り離して、別のプログラムに組み込める。

| 製品 | 中身 |
| --- | --- |
| Chrome | V8 ＋ 画面描画などのブラウザの機能 |
| Node.js | V8 ＋ ファイルやネットワークを扱う機能（`fs` など）。サーバー側で JavaScript を動かす |
| workerd（Cloudflare Workers） | V8 ＋ Web 標準の機能（`fetch`、`Request`、`Response`）＋ D1 などへの接続 |

つまり、**Node.js と workerd は兄弟**。同じエンジンを積んだ別の車で、Workers は Node.js の上で動いているわけではない。

**② isolate とは：VM・コンテナとの比較**

isolate は V8 自身が持っている仕組みで、「1つのプロセスの中に、互いのメモリが見えない JavaScript の実行区画を複数作る」機能。区画ごとにメモリ（ヒープ）と変数を持ち、ほかの区画のデータには触れない。仮想化の観点で並べると、**どの層で区切るか**が違う。

| | VM | コンテナ | V8 isolate |
| --- | --- | --- | --- |
| 何を仮想化するか | ハードウェア | OS（カーネルは共有） | 言語の実行環境（プロセスも共有） |
| 区切る仕組み | CPU の仮想化機能 | カーネルの namespace / cgroup | V8 のメモリ管理（JavaScript はメモリのアドレスを直接触れない） |
| 動かす土台 | ハイパーバイザ | コンテナランタイム（Docker、containerd） | **workerd**（V8 を組み込んだ Cloudflare の実行環境。オープンソース） |
| 起動時間 | 数十秒〜 | 数百 ms〜 | 数 ms |
| 1つあたりのメモリ | GB 単位 | 数十 MB〜 | 数 MB |
| 動かせるもの | 何でも | Linux で動くもの何でも | JavaScript と WebAssembly だけ |

「isolate の動作環境は何か」への答えは、**workerd というプロセス**。ハイパーバイザやコンテナランタイムに当たる役割を workerd が果たし、1つの workerd の中に、世界中のユーザーの Worker が何千も同居している。

区切りが「言語のレベル」だけだと心配に見えるが、Cloudflare は守りを何重にも重ねている。

- isolate の中では精密なタイマーやスレッドを使えなくして、CPU の脆弱性（Spectre）を突く攻撃をしにくくする
- workerd のプロセス自体も、OS のサンドボックスで閉じ込める
- 怪しい動きをする Worker は、別のプロセスに隔離する

そのぶん、VM やコンテナに比べて圧倒的に軽く、速く起動できる。デプロイのときに出ていた `Worker Startup Time: 1 ms` は、この isolate の起動時間。「JavaScript しか動かない」という Workers の制約も、ここから来ている。

**③ リクエストはエッジに着いてからどこへ行くか**

推察（isolate が動くサーバーが世界の各所に Cloudflare の持ち物としてあり、そのどれかにリクエストが飛ぶ）は**ほぼ正解**。1点だけ補正すると、**リクエストが着いたサーバーそのものが、isolate を動かすサーバー**。エッジの奥に「Worker 専用のサーバー群」が別にあるわけではない。

1. DNS：`sample-cf-workers-d1.alphajinsei.com` → Cloudflare の Anycast IP
2. その IP を世界中のデータセンターが名乗っている → ネットワーク的に一番近い所に届く（日本なら東京か大阪）
3. データセンターの負荷分散装置が、中の1台のサーバーに振る（どのサーバーも同じソフト一式が動いている）
4. そのサーバーが TLS を終端する（ホスト名に合う証明書で HTTPS をほどく）
5. ホスト名で設定を引く：「sample-cf-workers-d1.alphajinsei.com → Worker『sample-cf-workers-d1』」（この対応表を DNS の画面で見せているのが「Worker」レコード）
6. 同じサーバーの workerd が、その Worker の isolate を用意して `fetch()` を呼ぶ
7. レスポンスを同じ経路で返す

ポイントは2つ。

- **設定とコードは、デプロイの時点で全世界に配られている**。`wrangler deploy` すると、コードと「ホスト名 → Worker」の対応は、数秒で世界中のデータセンターに行き渡る。そのため、どのデータセンターのどのサーバーに着いても、その場で処理できる。
- **IP では振り分けていない**。IP は全員共通の入口にすぎない。どの Worker を動かすかは、5 でホスト名を見て決める。Cloudflare は、4 の TLS の最初のあいさつでホスト名が分かった時点で isolate の準備を始めるので、コールドスタートがほぼ見えない。

例外として、D1 はデータの本体が特定の場所に置かれる。Worker は東京で動いても、DB への問い合わせはデータのある場所まで飛ぶ。

→ [番外編：Workers の実行環境](extra-workers-runtime.md)（図は番外編に英数字だけで描いてある）

### Q. Worker は、あらかじめどこかにデプロイしておくものではなく、Cloudflare がリクエストを受け付けると、何が呼ばれたかを判断してその場で実行する技術、という世界観か

ほぼそのとおりだが、「あらかじめどこかにデプロイしておくものではない」は少し違う。正確には、**あらかじめ「世界中のすべての場所」にデプロイしておく**もの。

- **事前（デプロイ）**：コードと「ホスト名 → Worker」の対応を、全データセンターに配っておく
- **リクエストのたび（実行）**：着いたサーバーで isolate を用意して、その場で実行する

「サーバーを起動しておく」必要がないのは、実行のほうがリクエストのたびに一瞬で済むから。

### Q. `wrangler.jsonc` の `"assets": { "directory": "./public" }` と `"main": "src/index.ts"` は、どう違うのか

どちらも「何を返すか」の指定だが、**実行するものか、そのまま返すものか**が違う。

| | `main` | `assets` |
| --- | --- | --- |
| 中身 | **実行するコード**（`src/index.ts`） | **そのまま返すファイル**（`public/` の HTML、CSS、画像など） |
| デプロイで起きること | wrangler が TypeScript を JavaScript に変換し、1つのファイルにまとめて送る | ファイルをそのまま Cloudflare の配信用の置き場に送る |
| リクエストが来たとき | isolate で `fetch()` が呼ばれ、コードが応答を作る | ファイルがそのまま返る。Worker は起動しない |
| 料金・制限 | 無料枠のリクエスト数と CPU 時間を使う | 静的ファイルの配信は無料・無制限 |

振り分けのルールは、既定ではこうなっている。

- パスに合うファイルが `public/` にある → そのファイルを返す（Worker は呼ばれない）。例：`/` → `public/index.html`
- ない → `main` の `fetch()` を呼ぶ。例：`/api/health` → Hono が `{"ok":true}` を返す

これを判断しているのは **Cloudflare のエッジ**で、Worker のコードではない。開発者ツールで見た2つのリクエストのうち、HTML はファイルの置き場から、JSON は Worker から返っている。この挙動は設定で変えられる。

- `"run_worker_first"`：ファイルより先に Worker を通す（ログイン確認を挟みたいときなど）
- `"not_found_handling": "single-page-application"`：ファイルがないときに `index.html` を返す（React などの SPA 向け）

### Q. `wrangler dev` の実行中に Ctrl+V を押したら、トンネルや QR コード、remote preview などの表示が次々に出た。何だったのか

ログと本番のデプロイ履歴を確認した結果、**本番には影響していなかった**。開いていたものも、Ctrl+C で全部閉じていた。

**Ctrl+V でクリップボードの文字列が貼り付けられ、その中の文字が `wrangler dev` のキー操作として次々に実行された**もの。VSCode のターミナルでは Ctrl+V は貼り付けで、`wrangler dev` は起動中に1文字のキーを受け付ける（`b` でブラウザを開く、`c` で画面をクリア、`x` で終了など）。冒頭の「Resource location: local … Migrations to be applied:」は、貼り付けた文字列そのもの（直前にコピーした migrations apply の出力など）。主に2つの操作が動いていた。

**1. `t`：Cloudflare Tunnel で、手元の dev サーバーを一時的にインターネットに公開した**

- `cloudflared`（Cloudflare Tunnel のクライアント）を自動でダウンロードして起動し、`https://describing-tires-neck-prices.trycloudflare.com/` という一時的な URL を発行していた。QR コードは、その URL をスマホで開くためのもの。
- 「この URL を知っている人は誰でもアクセスできる」という警告が出ていたのはこのため。
- `t` が何度か押されたので、開く → 閉じる → また開く、を繰り返していた。Ctrl+C で wrangler ごと止めたので閉じた。この URL は使い捨てで、二度と使われない。

**2. ローカルとリモートの切り替え：「remote preview」に切り替わった**

- 「Starting remote preview」は、手元の workerd ではなく、**Cloudflare 上でプレビュー用に Worker を動かすモード**。コードとファイルがアップロードされたのはこのため（「Proceeding with deployment」はプレビューへの配置の意味で、本番へのデプロイではない）。
- このモードでは、D1 も本番の DB につながる。その間にトンネル経由でアクセスがあれば、本番の DB を触れる状態だった。ただ、開いていたのは数十秒程度で、URL も推測できない文字列なので、実害はまず考えられない。
- 本番のデプロイ履歴の最新は、この dev セッションが終わった直後の時刻で、テストのあとの `npx wrangler deploy` だった。プレビューは、本番の履歴には残らない。

後始末として、ダウンロードされた `cloudflared` は `~/.config/.wrangler/cloudflared/` に残っている（消しても、必要になればまた自動でダウンロードされる）。

Cloudflare Tunnel は、手元や VM のサーバーから Cloudflare に向けて外向きにつなぐので、受信のポートを開けずに公開できる技術。今回のは、アカウントなしで試せる使い捨て版（Quick Tunnel）。（方式③では検討したが、使わないことにした。）

**理解の確認**：「変な動作だった」ということで理解した。

## ドメインと DNS

### Q. アプリ群を alphajinsei.com のサブドメインとして公開していることが多い。これも wrangler で管理できるのか

できる。`wrangler.jsonc` に1行足して `npx wrangler deploy` するだけで、次の3つを Cloudflare がまとめてやってくれる（Custom Domain）。

```jsonc
"routes": [{ "pattern": "xxx.alphajinsei.com", "custom_domain": true }]
```

- DNS レコードの作成（ダッシュボードで手作業していた部分）
- SSL 証明書の発行
- そのホスト名へのアクセスを、この Worker に流す設定

`alphajinsei.com` の DNS が同じ Cloudflare アカウントにあるので、これが使える。設定をやめれば、DNS レコードも一緒に消える。注意点は2つ。

- **同じ名前の DNS レコードがすでにあると失敗する。**既存のサブドメイン（保育園のページなど）と重ならない名前にする。
- **サブドメインは1段にする。**無料の証明書（Universal SSL）が自動で守るのは `*.alphajinsei.com` の1段まで。`cf-workers-d1.sample.alphajinsei.com` のような2段の名前には、証明書が自動で付かない。

サブドメインの名前は、フォルダ名や Worker の名前とそろえて `sample-cf-workers-d1.alphajinsei.com` にした。

### Q. サブドメインではなく、ドメインは同じでパスベースルーティングで振り分ける構成も可能か

できる。それが **Route**。

```jsonc
"routes": [{ "pattern": "alphajinsei.com/memo/*", "zone_name": "alphajinsei.com" }]
```

| | Custom Domain | Route |
| --- | --- | --- |
| 単位 | ホスト名ごと Worker に渡す | ホスト名 + パスのパターンに合ったリクエストだけ Worker に渡す |
| DNS と証明書 | 自動で作られる | そのホスト名の DNS レコード（プロキシ有効）が別に必要 |
| 向いている使い方 | 1サービス＝1サブドメイン | 既存のサイトの一部のパスだけ Worker で処理する |

パスで振り分ける場合の注意点は2つ。

- Worker には `/memo/...` のように先頭のパスが付いたまま届く。アプリ側でその前提で作る必要がある（Hono には `basePath` という機能がある）。
- `alphajinsei.com` 本体はポートフォリオ（Pages）なので、Pages と Route がどう重なるかは、試す前に確認が必要。

サービスごとに独立させたいなら、サブドメインのほうが単純で、Access（ログイン保護）もサービス単位で設定しやすい。（このときは、フィージビリティを確かめたかっただけで、サブドメインで管理することにした。）

### Q. デプロイで出ていた warning は何だったのか。カスタムドメインを設定しているが、いつもの workers.dev の方でも今アクセスできてしまうよ、ということか

その理解で合っている。正確にはエラーではなく warning（注意）で、内容は「**`wrangler.jsonc` に `workers_dev` が書かれていないので、既定値（有効）で workers.dev の URL も公開したままにします**」というもの。

わざわざ警告するのは、設定ファイルに書いていない項目を wrangler が既定値で上書きするから。

- `workers_dev` を書かないと、デプロイのたびに「有効」で上書きされる。ダッシュボードで workers.dev を手動で無効にしていても、次のデプロイで有効に戻ってしまう。
- カスタムドメインを設定した人の多くは、workers.dev の URL を想定していない。そのため、「意図せず別の入口が開いたままですよ」と知らせている。

`"workers_dev": false` と明示したので、**設定ファイルが正**になった。workers.dev は閉じたまま、という意図が記録されている。Access でログインを付けるときには、これが大事になる。workers.dev の入口が開いていると、そこから Access を迂回できてしまうから。

### Q. DNS の画面で、レコードのタイプが「Worker」という謎のレコードになっていた。Cloudflare 独自の概念か。Worker が稼働する IP に自動的に振られるような感じか。Custom Domain と Route の違いは、パスベースルーティングかサブドメインか、という話か。DNS レコードを自分で作っていないのに名前解決できるのは、Cloudflare が Worker の IP（実態は V8 isolate なので IP の概念があるかも分からないが）に DNS をひもづけているからか

**Worker レコードの正体**：Cloudflare 独自の表示で、普通の DNS にあるレコードの種類ではない。また、**Worker に IP があるわけでもない。**

- `sample-cf-workers-d1.alphajinsei.com` を名前解決すると、**Cloudflare のエッジの IP アドレス**が返る。オレンジの雲（プロキシ有効）のレコードと同じ IP で、Cloudflare を通るサイトのほぼすべてがこの IP を共有している。同じ IP を世界中のデータセンターが一斉に名乗っていて（Anycast）、利用者に一番近いデータセンターに届く。
- 届いたリクエストのホスト名を見て、エッジが「このホスト名は `sample-cf-workers-d1` という Worker に渡す」と判断し、isolate で `fetch` を呼ぶ。
- 「Worker」というレコードは、この「ホスト名 → Worker」の対応を DNS の画面に表示したもの。Custom Domain が管理しているので、手では編集できない。

つまり、**振り分けは IP ではなく、エッジでホスト名を見て行っている。**「IP の概念があるかも分からない」は鋭くて、Worker 1つ1つに IP はない。

**Custom Domain と Route**：大筋は合っている。もう1点足すと、Custom Domain は Worker が**そのホスト名の終点（オリジン）そのもの**になる。Route は、オリジンがある既存のホスト名の**手前に割り込む**形。そのため、Route では Worker が処理しなかったリクエストを、後ろのサーバーに流すこともできる。

**DNS を自分で作っていないのに名前解決できる理由**：上のとおり。DNS（Cloudflare 自身）とエッジ（Cloudflare 自身）を同じ会社が持っているので、Custom Domain を設定すると、両方を一度に設定できる。

## デプロイの方式

### Q. git push で自動デプロイは必須ではなく、面倒な手順なしにデプロイ・更新できることが意図だった。手元の wrangler で反映できている時点で達成しているのでは。今 wrangler でデプロイしている Worker に、同名で GitHub 連携も設定すると、どんな挙動になるのか

**両方とも動き、後からデプロイしたほうが勝つ。**

- GitHub 連携（Workers Builds）をつなぐと、main に push するたびに、Cloudflare 側でビルドしてデプロイする。
- 手元の `wrangler deploy` も、これまでどおり使える。
- どちらも同じ Worker の新しい「版」を作り、最後に出したものが本番になる。デプロイ履歴の「Source」で、どちらから来たかは区別できる。

困るのは、**本番の中身がどちら由来か分からなくなる**こと。手元でコミット前のコードを `wrangler deploy` で出しても、次に push すると main の内容で上書きされる。逆の上書きも起きる。混ぜずに、どちらか一方に決めるのが普通。

「DB だけは管理できない」は GitHub 連携でも解決できる。Deploy command を `npx wrangler d1 migrations apply DB --remote && npx wrangler deploy` にすればよい。

手元の CLI にする方針自体には賛成で、要件の意図（面倒な手順なしでデプロイ・更新できる）は wrangler の時点でほぼ満たしている。ただし GitHub 連携には「**本番 ＝ git の main の内容**」が保証される利点があり、手元の CLI だと次の事故が起きうる。

- コミットしていない変更を、そのまま本番に出してしまう
- push を忘れて、GitHub と本番がずれる
- 本番がどのコミットなのか分からない（デプロイ履歴は「Source: Unknown」で、メッセージも空）
- マイグレーションの適用を忘れて、コードと DB がずれる

### Q. GitHub 連携、手元の wrangler、GitHub Actions を、「既存方式との整合性」「初期設定の手間」「機能改修のときの取り回し」で比べるとどれがよいか。最終的に、アプリ開発に集中できる状態を達成できるか

| 観点 | A. GitHub 連携（Workers Builds） | B. 手元の wrangler（スクリプト） | C. GitHub Actions |
| --- | --- | --- | --- |
| 既存方式（dia-logos）との整合性 | ◎ 同じ（Cloudflare 側が push を拾う） | △ 体験が変わる（push とデプロイが別の操作になる） | ○ 体験は同じ（push ＝ デプロイ）。仕組みが違う |
| 初期設定の手間（サービスごと） | △ 毎回ダッシュボードで操作が要る。Claude は代わりにできない | ◎ ほぼゼロ（テンプレートにスクリプトを入れるだけ） | ○ ワークフローのファイルはテンプレートに入れておく。シークレットの登録は `gh secret set` で、Claude が CLI で全部できる |
| 改修のときの取り回し | ◎ push するだけ。どのパソコンからでもデプロイできる | ○ コマンド1つ。ただし、ログイン済みのこのパソコンからしかデプロイできない | ◎ A と同じ。手順がファイルなので、マイグレーションなども自由に足せる |
| 本番 ＝ git の main が保証されるか | ◎ | △ スクリプトで防ぐが、迂回できる | ◎ |

最終目標（新しいサービスを作るとき、インフラのことを知らなくてよい）から逆算すると、条件は2つ。

1. **立ち上げを Claude が全部 CLI でできる** → A は毎回ダッシュボードでの操作が残るので脱落。
2. **デプロイを意識しなくてよい** → B は「デプロイする」操作が毎回残り、push 忘れやマイグレーション忘れの心配も残る。ログイン済みのパソコンにも縛られる。

両方を満たすのは C だけ、として、このときは **GitHub Actions** を採用した（のちに方針転換した。下の問い）。

ブランチのプレビューは、A でも C でも**本番と同じ D1 につながる**。プレビューのつもりのコードが本番のデータを書き換えたり、ブランチで流したマイグレーションが本番の DB に適用されたりする。そのため、最初は main に push したときだけデプロイする。

### Q. GitHub Actions を利用するとは、具体的にどんな設定をするのか。GitHub が、push されるたびに Cloudflare にデプロイを流すイメージか。既存の Cloudflare Pages も同じ方式に乗り換えられるか

イメージはほぼそのとおり。正確には、**push のたびに GitHub が使い捨てのパソコン（Ubuntu）を1台用意し、そこで手元と同じ `wrangler` のコマンドを実行する**仕組み。Cloudflare 側は GitHub のことを何も知らず、普通に API を呼ばれてデプロイされるだけ。

| | GitHub Actions | Workers Builds / Pages |
| --- | --- | --- |
| 向き | GitHub が Cloudflare に**送り込む** | Cloudflare が GitHub から**取りに行く** |
| 認証情報を預ける先 | GitHub に Cloudflare の API トークンを預ける（Secrets） | Cloudflare に GitHub へのアクセス権（GitHub App）を渡す |

設定は3つ。

1. Cloudflare でデプロイ用の API トークンを作る（最初の1回だけ。全サービスで共用）。テンプレート「Edit Cloudflare Workers」に D1 の編集権限を足す。これが、手元の `wrangler login` の代わりになる鍵。
2. そのトークンを GitHub のリポジトリに預ける：`gh secret set CLOUDFLARE_API_TOKEN`（Secrets に入れた値は、登録した本人も後から読めない。ワークフローの実行中にだけ使われる）。
3. リポジトリにワークフローのファイル（`.github/workflows/deploy.yml`）を置く。

```yaml
name: Deploy
on:
  push:
    branches: [main]      # main に push したときだけ
  workflow_dispatch:      # 手動でも実行できる（gh workflow run Deploy）
concurrency: deploy       # 続けて push しても、デプロイは1つずつ順番に
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm
      - run: npm ci
      - run: npx wrangler d1 migrations apply DB --remote
      - run: npx wrangler deploy --message "${{ github.sha }}"
```

`steps` の中身は、手元でやってきたコマンドそのもの。違うのは、`wrangler login` の代わりに、環境変数の API トークンで認証している点だけ。`npm ci` は `package-lock.json` のとおりに依存パッケージを入れ、`npm install` と違ってロックファイルを書き換えない（CI 向け）。

**Pages の乗り換え**：できる。Pages にも `wrangler pages deploy <フォルダ> --project-name=dia-logos` という CLI があるので、ワークフローの最後の行をそれに変えればよい。ただし注意点が2つ。

- GitHub 連携で作った Pages のプロジェクトは、あとから CLI 方式（Direct Upload）に切り替えられない。新しいプロジェクトを作り、カスタムドメインを付け替えることになる。
- Cloudflare は新機能をおもに Workers 側に入れている。乗り換えるなら、Pages 同士ではなく、Workers の Static Assets に移すのが自然。

いま動いていて困っていないなら、乗り換える理由はない。

### Q. drizzle-kit の generate は、ローカルで実施する前提か。デプロイに含めるのとどちらがよいか

**手元で実行する前提。デプロイには含めないほうがよい。**

- generate が作る SQL は、**レビューしてから流すもの**。`updated_at` の件で見たとおり、自動で作られた SQL でもそのままでは流せないことがある。CI で generate すると、誰も読んでいない DDL がそのまま本番に流れる。
- 列名を変えたとき、drizzle-kit は「名前の変更か、削除して追加か」を**対話で聞いてくる**。CI では答えられない。
- 生成された `migrations/*.sql` をコミットしておけば、git の履歴が「本番にいつ何を流したか」の記録になる。

CI でやる価値があるのは、**generate し忘れの検知**。CI で generate を実行してみて、新しいファイルができたら（schema.ts を直したのに generate していなかったら）失敗させる。

### Q. ローカルでの確認は `wrangler dev` でするので、本番のデプロイだけ GitHub に任せることになり、どっちもどっち。決めて、そのやり方に従う整理が大事で、徹底できる仕組みを作ることが大事では

おっしゃるとおり。どちらの方式にも一長一短があり、大事なのは決めたことを守れる状態にすること。手段は2段ある。

- **文書で決める**：アプリ側向けの資料と、各サービスのリポジトリに置く CLAUDE.md。Claude が作業するなら、CLAUDE.md に「デプロイの方法」を書くのが一番効く。
- **仕組みで縛る**：generate し忘れの検知のように、CI でルール違反を落とす。さらに強くするなら、開発用のパソコンでは `wrangler logout` しておく手もある。`wrangler dev` はローカルの D1 で動くのでログインは不要で、手元からの本番デプロイが物理的にできなくなる（本番を触る作業だけ、`.secrets/` のトークンを読み込んで行う）。

### Q. デプロイは基本的に後勝ちか。ローカルの `wrangler deploy` を禁止することはできないが、Cloudflare からすると同一のリソースに対する同じデプロイ処理に見えて、違いはどこからコマンドを打つか（ローカルか GitHub Actions か）だけ、という理解でよいか

**その理解で合っている。**Cloudflare から見ると、どちらも「同じ Worker に新しい版を作って本番にする」という同じ API 呼び出し。違うのは2点だけ。

- 認証情報：手元は `wrangler login` の OAuth、Actions は API トークン
- コマンドを打った場所

実際にデプロイ履歴で確かめると、次のようになっていた。

| | 手元の deploy | Actions の deploy |
| --- | --- | --- |
| Author | 自分のメールアドレス | 自分のメールアドレス |
| Source | Unknown | Unknown |
| Message | （空） | コミットの ID |

API トークンも自分のアカウントで作ったものなので、Author も同じになる。**区別できるのは、Actions で付けたメッセージだけ**。逆に言えば、「メッセージが空のデプロイは、ルール外の手元からのデプロイ」と見分けられる。

**理解の確認**（Step 6 の理解ポイント）：「Cloudflare 目線では認証情報が違うだけで、実際にデプロイされる内容は同じ。後勝ちになる点に注意」「Workers Builds は Cloudflare が GitHub の認証情報を持ち、GitHub Actions は GitHub 側が Cloudflare の認証情報を持つ」で合っている。「どちらがどちらの鍵を持つか」が逆になっている。

### Q. やっぱり `npx wrangler deploy` を打つ方針に変えてよいか（git push の重みが増してしまった）

方針転換した。理由（ユーザーの判断）：

- git push で自動デプロイされるのは便利だが、push することの重みが増した（デプロイが失敗する push は原理的に受け入れられない）。
- GitHub Actions 方式は git と Cloudflare が一致するのは便利だが、個人開発なので、そこまでの統制を求めていない。`wrangler dev` → `wrangler deploy` して、git は適宜 push、のほうが手軽に開発できる。git はコードをインターネット上に保管しておくくらいの意味合い。
- 静的ファイルはデプロイの失敗のしようがないので、push で再デプロイされると便利。アプリは push の重みが増え、開発のときに考えることが増える。

結論は「アプリは手元の `npm run deploy`、git は保管用に適宜 push」。経緯（push デプロイを前提に始めた → 比較して GitHub Actions を採用し、実際に動かした → 使ってみて push が重くなった → 手元のデプロイに戻した）は、方式検討の資料にも残した。机上で決めたことでも、使ってみて合わなければ変えてよい。

`package.json` の `deploy` スクリプトは、マイグレーションの適用とデプロイを1つにまとめた。

```json
"deploy": "wrangler d1 migrations apply DB --remote && wrangler deploy --message \"$(git describe --always --dirty)\""
```

- マイグレーションの流し忘れを防ぐ。
- `--message` には、そのときのコミットが入る。コミット前の変更を含んでいると `4cd4220-dirty` のように付くので、統制はしないまま、あとから「本番がどの状態か」を追える（任意。最終的には `--message` なしのシンプルな形を選んだ）。

Actions で使っていた API トークンは、使い道がなくなったら削除（Roll ではなく Delete）しておくのがおすすめ。使わない編集権限付きのトークンは、漏れたときのリスクだけが残る。

## 静的サイト（方式①）

### Q. dia-logos ほかいくつかのサイトは、Markdown で書いて SSG でビルドするものだった。これまで Pages でやってきたのであまり意識しなかった。これまで見てきた方式とそろえるなら、GitHub Actions でも Workers Builds でもなく、wrangler 直が一番素直か。そもそも Workers ではなく Pages で実現すれば十分、という見方もある

**wrangler 直（Workers Static Assets）にそろえるのに賛成。**既存のサイトを見ると、`blog`（dia-logos）は Astro（Markdown → SSG）、`portfolio` は素の HTML だった。理由は3つ。

**1. 方式が1つにそろう**

アプリも静的サイトも、「**手元で確かめて `npm run deploy`、git は保管用**」の1つの型になる。違うのは `deploy` スクリプトの中身だけ。

| | アプリ（方式②） | 静的サイト（方式①、SSG） |
| --- | --- | --- |
| 手元で確かめる | `npx wrangler dev` | `npm run dev`（`astro dev`） |
| `npm run deploy` の中身 | マイグレーション → `wrangler deploy` | `astro build` → `wrangler deploy` |
| `wrangler.jsonc` | `main` ＋ `assets` ＋ D1 | `assets` だけ（`./dist`） |

Claude から見ても、どのサービスでも同じ手順でデプロイできる。

**2. SSG のビルドを手元でやるのは、むしろ自然**

Pages や Workers Builds では、ビルドが Cloudflare 側で走る。そのため、ビルドの失敗はデプロイの後で気づくことになる。手元でビルドすれば、`astro dev` で見ていたものと同じ成果物（`dist/`）がそのまま出る。

（その前に検討した段階では、「静的サイトはデプロイが失敗しても前の版が残るだけなので push でデプロイする」という案もあった。ただ SSG にはビルドがあるので、「失敗しようがない」わけではない。）

**3. Pages ではなく Workers にするのは、Cloudflare の向かう先だから**

「Pages で十分」というのも正しい見方で、静的サイトの配信だけなら機能はほぼ同じ（Pages にも `wrangler pages deploy dist` という CLI がある）。ただ、Cloudflare は新機能を Workers 側に入れていて、新しく作るなら Workers を勧めている。また、Workers なら、あとで問い合わせフォームの API を足したくなったとき、同じ Worker に `main` を足すだけで済む。

**受け入れることになるトレードオフ**：記事の公開が「このパソコンから `npm run deploy`」になる。スマホや GitHub の画面から記事を書いて push、という公開の仕方はできなくなる。そういう使い方をしたくなったら、そのときに GitHub Actions を足せば対応できる（Step 6 で書いたワークフローから、D1 の行をビルドの行に替えるだけ）。

既存の blog（Pages + GitHub 連携）は、困っていなければそのままでよい。移すかどうかは、サンプルで手応えを見てから決める。

### Q. Workers は基本的に Pages の上位互換だという理解でいいか。Pages にできて Workers にできないことはあるか。たとえば、Pages でしかサポートされていない動作環境があるとか

**ほぼ上位互換、という理解でよい。**Cloudflare 自身も、新しく作るなら Workers を勧めていて、Pages から Workers への移行の手順も公開している（ここに書くのは当時 Claude が知っていた範囲で、厳密には公式の比較表で確認する）。

**Pages にあって、Workers では形が違うもの**

- **`functions/` フォルダによるファイルベースのルーティング**（Pages Functions）：Workers にはこの仕組みがなく、Hono などで自分でルーティングを書く。
- **Git 連携・ブランチごとのプレビュー・ビルドの環境**：Workers でも Workers Builds とプレビュー URL でほぼ同じことができる。ただし、Pages のほうが歴史が長く、こなれている。
- **一部のフレームワークのアダプター**：昔は「Pages 向け」のものがあった（Next.js の `next-on-pages` など）。いまは Workers 向け（OpenNext など）に移っている。

**Workers にあって、Pages にないもの**

- Cron Triggers、Durable Objects、Queues の受信、Workers Logs（observability）、Smart Placement など、新しい機能のほとんど。

「Pages でしか動かない実行環境」は、知る限りない。Pages Functions も、中身は Workers と同じ実行環境（workerd）。静的サイトの配信に限れば、どちらでも同じことができる。

### Q. 完全な静的サイトでも、Astro の Cloudflare アダプターは要るのか

要らない。サンプル（`sample-cf-workers-static`）は完全な静的サイトなので、アダプターは使っていない。`wrangler.jsonc` には `main` を書かず、`assets`（`./dist`）とカスタムドメインと `"workers_dev": false` だけを書いた（コードのない Worker）。存在しないパスには、自前の `404.html` を返す（`not_found_handling: "404-page"`）。

| 内容 | 作り |
| --- | --- |
| 記事 | `src/pages/posts/*.md` に Markdown で2本。一覧ページが、記事を日付順に自動で並べる |
| `wrangler.jsonc` | `main` なしで、`assets`（`./dist`）とカスタムドメインと `"workers_dev": false` だけ |
| 404 | 存在しないパスには、自前の `404.html` を返す |
| `npm run deploy` | `astro build && wrangler deploy` |
| `npm run preview` | ビルドして、本番と同じ workerd で手元で確かめる |

### Q. 一覧から記事に飛ぶと、リダイレクトが1回挟まっていた。なぜか

Astro の既定の出力は `posts/hello/index.html` の形なので、`/posts/hello` へのアクセスが `/posts/hello/` に 307 でリダイレクトされていた。Astro の出力を `posts/hello.html` の形（`build.format: 'file'`）にして、リダイレクトなしで返るようにした。

### Q. 記事の日付が `2026-11-03T00:00:00.000Z` と出ていた。なぜか

frontmatter の `date: 2026-11-03` を、YAML が日付型として読むため。日付を `"2026-11-03"` と引用符で囲んで直した。

### Q. `npm install` で、インストールスクリプトを止めた旨の警告が出た。問題はないのか

新しい npm は、依存パッケージのインストールスクリプト（esbuild、workerd の postinstall）を既定で止める。止めたものは `npm install-scripts ls` で確認できる。止められていても、ビルド・ローカル実行（wrangler dev）・デプロイはどれも問題なく動いた。

## ステートレス

### Q. メモをメモリ上の配列に持って本番で試したら、たまにしか本番のメモを取得できなかった。V8 isolate の実行環境は回によってランダムで、たまたま以前アクセスしたところで再度実行された場合のみ過去のメモを追える、ということか。そもそも Worker をステートフルに作ってはいけない（アンチパターン）ということか

**なぜ「たまにしか」取れないか**

大筋はそのとおり。正確には「ランダム」ではなく、**メモがどの isolate に入ったか**で決まっている。`const memos = []` はその isolate のメモリにしかない。次の3つの条件が全部そろったときだけ、同じメモが見える。

1. 同じ**データセンター**に着く（Anycast で決まる。回線を変えると別の場所になりうる）
2. その中の同じ**サーバー**に振られる（負荷分散で決まる）
3. そのサーバーで、同じ **isolate** がまだ生きている（しばらく使われない、メモリが足りない、再デプロイした、などで捨てられる）

スマホから見えなかったり、放置や再デプロイのあとに消えたりしたのは、このどれかが外れたから。手元の `wrangler dev` で安定していたのは、workerd が1つ、isolate も1つしかないから。

**Worker をステートフルに作るのはアンチパターンか**

**そのとおり。Worker は「状態を持たない（ステートレス）」前提で作るのが原則。**状態は Worker の外の保存先に置き、バインディングで読み書きする。これが D1 を使う理由。

これは Workers に特有の話ではなく、ロードバランサーの後ろにアプリサーバーを何台も並べる構成と同じ考え方。

| 昔ながらの構成 | Workers |
| --- | --- |
| アプリサーバーのメモリにセッションを持つと、LB が別のサーバーに振ったときに消える | isolate のメモリに持つと、別の isolate に着いたときに消える |
| 対策：スティッキーセッション（同じ人を同じサーバーへ）や、セッションを DB に外出しする | 対策：外の保存先に置く。スティッキーにする手段はない |

Lambda も同じ。一般には「12-Factor App」の「プロセスはステートレスにする」という原則として知られている。Workers は、サーバーの台数が「世界中」、捨てられる頻度が「いつでも」と極端なので、この原則を守らないとすぐ壊れる。

ただし、グローバル変数がすべて悪いわけではない。

- **OK**：使い回せるもののキャッシュ。一度作った設定オブジェクトや、計算に時間のかかる値など、消えても作り直せばいいもの。
- **NG**：消えたら困る「正しいデータ」。今回のメモがこれ。

状態の置き場所は、用途で使い分ける。

| 置き場所 | 向いているもの |
| --- | --- |
| **D1** | 表形式のデータ（SQLite）。今回のメモ |
| **KV** | 設定やキャッシュ。読むのは速いが、書いた内容が世界中に行き渡るまで少し時間がかかる |
| **R2** | 画像などのファイル |
| **Durable Objects** | どうしても「状態を持つ処理」が必要なとき（チャット、同時編集など）。1つの ID に対して世界で1つだけ存在する、例外的な仕組み |

## D1

### Q. `wrangler d1 create` で聞かれた3つの質問は何だったのか

| 質問 | 意味 |
| --- | --- |
| add it on your behalf? → yes | 出力に出た設定を、wrangler が `wrangler.jsonc` に書き足す |
| binding name → `DB` | コードから DB を呼ぶときの名前。`c.env.DB` の `DB` になる。既定の `sample_cf_workers_d1_db` は長いので、短い名前にした |
| connect to the remote resource? → no | `wrangler dev` のときに本番の DB につなぐかどうか。no なので、手元の SQLite ファイル（`.wrangler/` の下）を使う。開発中に本番のデータを壊さずに済む |

`database_id` は git にコミットしてよい。DB を指す識別子にすぎず、触るには Cloudflare アカウントの認証が必要だから。パスワードのような秘密情報ではない。

### Q. migration（`migrations create`）を実行しても、コンソール上は何も見えないのはなぜか

**まだ何も起きていない**から。`migrations create` は、手元に空の SQL ファイルを作っただけで、D1 には何も送っていない。

**マイグレーション**とは、DB の構造の変更（テーブルを作る、列を足す…）を、番号付きの SQL ファイルとして順に積み上げていく管理方法。

| コマンド | 何が起きるか |
| --- | --- |
| `migrations create` | 手元に SQL ファイルを作るだけ |
| `migrations apply --local` | 手元の SQLite ファイル（`.wrangler/` の下）に適用する。コンソールには出ない |
| `migrations apply --remote` | 本番の D1 に適用する。ここで初めてコンソールにテーブルが出る |

`--remote` で適用すると、コンソールに `memos` のほかに **`d1_migrations`** というテーブルも出てくる。「どのファイルを適用済みか」の記録。

### Q. D1 を使う API のコード（`prepare`、`bind`、`all` など）は、それぞれ何をしているのか

```ts
app.post("/api/memos", async (c) => {
  const { text } = await c.req.json<{ text: string }>();
  const memo = await c.env.DB
    .prepare("INSERT INTO memos (text) VALUES (?) RETURNING id, text, created_at AS createdAt")
    .bind(text)
    .first<Memo>();
  return c.json(memo, 201);
});
```

| コード | 意味 |
| --- | --- |
| `c.env.DB` | バインディングで渡された D1。接続文字列もパスワードも書いていない |
| `.prepare("... ? ...")` | SQL を用意する。`?` は後から値を入れる穴 |
| `.bind(text)` | `?` に値を入れる。文字列をつなげて SQL を組み立てないので、SQL インジェクションを防げる |
| `.all()` / `.first()` / `.run()` | 実行する。全行を返す／最初の1行だけ返す／結果の行は要らない（件数などだけ） |
| `created_at AS createdAt` | DB の列名（スネークケース）を、JavaScript の名前（キャメルケース）に合わせる |
| `RETURNING ...` | INSERT した行をその場で返してもらう。DB が振った id を受け取れる |
| `meta.changes` | DELETE で消えた行数。0 なら、その id は存在しなかった |

### Q. `wrangler dev` を止めていても、`d1 execute --local` の select の結果が返ってきた。D1 には起動停止の概念はないのか。表領域的な概念はあるか

**起動・停止の概念はない。**理由は、ローカルと本番で少し違う。

**ローカル：SQLite は「サーバー」ではなく「ファイルを読み書きするライブラリ」だから**

Oracle や PostgreSQL は、DB サーバーのプロセスが起動していて、クライアントはそこに接続して問い合わせる。SQLite にはそのプロセスがない。使う側のプログラムに組み込まれたライブラリが、**DB ファイルを直接開いて**読み書きする。

| | Oracle / PostgreSQL | SQLite |
| --- | --- | --- |
| 形 | サーバー（常駐プロセス＋共有メモリ）とクライアント | ライブラリ＋ファイル1つ |
| 起動・停止 | ある（インスタンスの起動・停止） | ない。ファイルがあれば読める |
| 接続 | ネットワーク越しにリスナーへ接続 | ファイルを開くだけ |

手元の実体は、`.wrangler/state/v3/d1/miniflare-D1DatabaseObject/` の下にある `.sqlite` のファイル。`d1 execute --local` は、`wrangler dev` を通さずにこのファイルを直接開いているので、dev を止めていても結果が返ってきた。横にある `-wal` と `-shm` は、SQLite が書き込み途中のデータを一時的に置くファイル。`.wrangler/` は `.gitignore` に入っているので、git には載らない。

**本番：Cloudflare が管理していて、使う側に「稼働中」という状態が見えないから**

本番の D1 も中身は SQLite。Cloudflare が裏で、問い合わせが来たら DB を動かし、使われなければ休ませている。料金も稼働時間ではなく、**読み書きした行数と、保存しているデータ量**で決まる。Worker と同じく、「止める」「起動しておく」を意識しない作り。

**表領域のような概念もない。**SQLite は **1つの DB ＝ 1つのファイル**で、表領域もスキーマ（Oracle でいうユーザーごとの名前空間）もない。ユーザーや権限（GRANT）もない。

| Oracle の概念 | D1 / SQLite での扱い |
| --- | --- |
| 表領域・データファイル | なし。DB 全体がファイル1つ |
| スキーマ | なし。テーブルは DB の中に直接並ぶ |
| ユーザー・GRANT | なし。**バインディングを持つ Worker が DB 全体を読み書きできる**。誰がアクセスできるかは「どの Worker にバインディングを付けるか」で決まる |
| 容量の管理 | DB ごとのサイズ上限がある（無料プランで 1 DB あたり 500 MB 程度、と記憶ベースで説明した。使う前に公式の料金ページで確認する） |

D1 は、大きな DB を1つ作って中を区切る使い方ではなく、**小さな DB をたくさん作る**使い方を想定している。サービスごと、場合によっては利用者ごとに DB を分ける。「1サービス＝1つの D1」は、その考え方にちょうど合っている。

### Q. ロックやトランザクションはどう管理されているのか。普通は DB 側で制御するが、クライアントが自らロックファイルを置いたりするのか

DB 側で制御している、という点は SQLite も同じ。ただ、「DB 側」の正体が違う。

**SQLite（ローカル）：OS のファイルロックを使う**

ロックファイルを別に置くのではなく、**DB ファイルそのものに OS のファイルロック**をかける。SQLite のライブラリが、ファイルを開いたプロセスの中でこれを自動でやる。`-wal` と `-shm` は、この仕組みの一部。

- `-wal`：書き込み途中のデータ。コミットまでは本体のファイルに書かない
- `-shm`：読み手と書き手が「いまどこまで書かれたか」を共有するための領域

Oracle との大きな違いは、**書き手は DB 全体で同時に1つだけ**という点。行ロックはない。読み手は何人でも同時に読める。個人サービス程度の書き込み量なら、これで十分。

**本番の D1：1つの DB を1か所で順番に処理する**

本番では、1つの DB を、APAC のどこか1か所にある1つのインスタンスが受け持つ。世界中の Worker からの書き込みは、すべてそこに集まって**順番に処理**される。同時に書き込む人がそもそも生まれないので、利用者がロックの競合を気にする場面はほとんどない。

トランザクションには1つ制約がある。

- `BEGIN` → 何往復かやりとり → `COMMIT`、という使い方は**できない**。
- 複数の SQL を「全部成功か全部失敗」にしたいときは、**`batch` でまとめて一度に送る**。

```ts
await c.env.DB.batch([
  c.env.DB.prepare("INSERT INTO memos (text) VALUES (?)").bind("1つ目"),
  c.env.DB.prepare("INSERT INTO memos (text) VALUES (?)").bind("2つ目"),
]);
```

途中で失敗すると、全部なかったことになる。「途中で Worker が止まって、ロックだけが残る」ことを避けるための設計。

### Q. バインディングとは、単にコード内で DB 接続する際になんと呼ぶか、という概念で、クラス名のようなものか

名前の付け方の話に見えるが、本質は「名前」より「**渡された窓口**」。**実行環境が、接続済み・許可済みの DB の窓口を `env` に入れて渡してくれる**ことが本質で、`DB` はその窓口を取り出すための名前にすぎない。

- Worker と DB の結び付けは、デプロイの時点で Cloudflare 側が済ませている → コードに認証情報が要らない
- 渡されていない DB には、コードから触れようがない

クラス名よりも、アプリサーバーの **JNDI データソース**に近い。接続先やパスワードはサーバー側で設定し、アプリは `jdbc/MyDS` のような名前で引くだけ、という仕組み。EC2 に付ける IAM ロール（コードに鍵を書かない）も同じ発想。

**理解の確認**：「アプリケーションには認証情報を書かず、Cloudflare が自動で渡してくれる。DB は窓口を取り出すための名前に過ぎない」で合っている。

### Q. マイグレーションの概念は面白い。個別に DDL の .sql を適用して、結局本番にどんなオブジェクトがあるか分からない、というのが実運用でよくある悩みだった。DDL 集を宣言的な記載として、apply や diff を取れる対象にする管理の仕方で、それが migration という表で管理されている、ということか

前半（本番に何があるか分からない悩みを解決する）は合っている。ただ、wrangler のマイグレーションは、実は**宣言的ではなく手続き的**。

| | 手続き的（wrangler のマイグレーション） | 宣言的 |
| --- | --- | --- |
| 書くもの | 変更の手順（CREATE、ALTER…）を番号順に | あるべき最終形 |
| 適用 | まだ流していない番号だけを順に流す | 現状と最終形の差分を計算して流す |
| 似ているもの | Flyway、番号付きの Ansible の手順 | Terraform |

`d1_migrations` が記録しているのは「どのファイルまで流したか」だけ。「本番にどんなオブジェクトがあるべきか」を知るには、ファイルを頭から全部たどる必要がある。誰かが直接 DDL を流せば、記録とずれる（drift）。

「宣言的に書いて、差分を取る」管理は、次の Drizzle がやってくれる。TypeScript で「あるべきテーブルの形」を書くと、前回との差分から、マイグレーションの SQL を自動で作る。宣言的な定義と、手続き的なマイグレーションのいいとこ取り。

## Drizzle

### Q. 結局 Drizzle とは何か。migration も宣言的な概念かと思ったが、そうではない。追記していく限りは追従できるが、削除や変更には耐えられない、ということか。素の管理、migration による管理、Drizzle による管理で、それぞれどんな世界観になり、どう改善されるのか。Drizzle はアプリのコードの面でも DB の構成管理の面でも有利だと思うが、既存のどんな課題がどう解消されるのか

**まず補正：マイグレーションも削除や変更はできる**

新しい番号のファイルに `ALTER` や `DROP` を書けば、削除も変更もできる。「耐えられない」わけではない。課題は別のところにある。

**3つの段階と、それぞれの世界観**

| | 素の管理 | マイグレーション | Drizzle |
| --- | --- | --- | --- |
| 書くもの | その場で流す DDL | 変更の**手順**を番号付きのファイルで | あるべき**最終形**（`schema.ts`） |
| あるべき姿 | どこにもない | ファイルを全部たどれば分かる | `schema.ts` を見れば分かる |
| 変更の SQL | 手で書く | 手で書く | 差分から自動で作られる |
| アプリのコード | 別管理 | 別管理 | 同じ定義から型が作られる |
| インフラでたとえると | ssh して手で設定 | 番号付きの手順書を1回ずつ流して記録 | Terraform に近い |

**素の管理 → マイグレーションで解決すること**（「結局本番に何があるか分からない」の解決）

- 各環境が、どこまで流したかを自分で知っている（`d1_migrations`）
- 新しい環境は、全ファイルを頭から流せば同じになる
- 変更が git の履歴とレビューに乗る

**マイグレーションに残る課題**

- あるべき最終形が1か所に書かれていない。履歴から頭の中で組み立てる。
- 変更の SQL を毎回手で書く。特に SQLite は `ALTER TABLE` でできることが少なく、列の型を変えるだけでも「新しいテーブルを作ってデータを移し、古いのを消す」手順を手で書くことになる。
- **アプリのコードと二重管理**になる。SQL でテーブルを定義したうえで、コード側にも `type Memo` を書き、`created_at AS createdAt` で手で合わせていた。列名を変えると、ずれは実行して初めて分かる。

**Drizzle で解決すること**

Drizzle は、`schema.ts` という1つの定義を、2つの面で使う。

- **構成管理の面（drizzle-kit）**：`schema.ts` を直して `generate` すると、前回との差分からマイグレーションの SQL が作られる。SQLite のテーブルの作り直しのような面倒な手順も自動で書かれる。出力は普通のマイグレーションなので、適用の記録や再現性はそのまま残る。**宣言的に書いて、手続き的に適用する**という形。
- **アプリのコードの面（drizzle-orm）**：クエリを `schema.ts` のオブジェクトで書くので、結果の型が自動で決まる。`type Memo` も `AS createdAt` も要らない。列名を変えると、使っているコードがすべてエディタで赤くなる。

推察のとおり、**構成管理とアプリのコードの両方に効く**。両方が同じ定義から作られるので、DB とコードのずれが起きにくくなる。

**Terraform との違い（大事な注意点）**

差分の比較相手は、実物の DB ではなく、Drizzle 自身の記録（`migrations/meta/` のスナップショット）。Terraform の `plan` のように実物を見に行かないので、本番に直接 DDL を流した drift は検出できない。Drizzle を入れたら、変更は必ず `schema.ts` から行う、という運用が前提になる。

→ [番外編：DB のスキーマ管理](extra-db-schema-management.md)

### Q. つまり、テーブル定義兼型定義のファイルで、アプリ側もインフラ側も一括管理しよう、という思想か。アプリ側と DB 側で2つの構造があると、テーブル名などの取り決めを両者がそれぞれ持つことになるが、1つのご本尊でどちらも管理できるようにする、両者の間を埋めるツール。加えて、それを DB 側には Terraform 的に適用できるようにした、という理解で合っているか

合っている。「1つのご本尊で、アプリと DB の間の取り決めを両方管理する」という言い方は、Drizzle の思想をよく表している。補足は1点だけ：Terraform 的に適用するとき、差分の比較相手は実物の DB ではなく、ご本尊の過去の版（スナップショット）。

### Q. 途中から Drizzle を入れるとき、何に気をつけるのか

Drizzle は「前回の形」を自分の記録（`migrations/meta/`）と比べて差分を作る。手書きの `0001` しかなく Drizzle の記録がない状態だと、最初の `generate` で「memos テーブルを作る」SQL がもう一度出てきて、既存のテーブルとぶつかる。途中から ORM を入れたときによく起きる問題（Flyway でいう「ベースライン」の問題）。

このときはテストデータしかなかったので、**手書きの 0001 を捨てて、Drizzle に最初から作らせた**（本番のテーブルも `DROP` して作り直した。本番データがあるときは絶対にやらない操作）。生成されたファイルの名前（`0000_concerned_smiling_tiger.sql` など）は、Drizzle がランダムに付ける。

### Q. 列を足すとき、生成された SQL に注意することはあるか

`updatedAt` は、`notNull` も `default` も付けない（空を許す）列にした。SQLite には、既存のテーブルに列を足すとき、`datetime('now')` のような「その時々で値が変わる既定値」は付けられない、という制約があるため。付けると、生成された SQL を流した時点でエラーになる。**自動で作られた SQL も、流す前に読む必要がある**、という例。

`generate` すると、`0001_xxx.sql` は `ALTER TABLE memos ADD updated_at text;` の1行になり、**前回との差分だけが SQL になっていた**。

### Q. 生 SQL を Drizzle に置き換えると、何が楽になり、何が隠れたのか。スキーマの正は schema.ts と migrations/*.sql のどちらか

| 生 SQL | Drizzle |
| --- | --- |
| `type Memo = { ... }` を手で書いた | なくなった。結果の型は `schema.ts` から自動で決まる |
| `"SELECT id, text, created_at AS createdAt ..."` | `db.select().from(memos)`。列名の対応は schema.ts に任せる |
| `.prepare("... ?").bind(text)` | `.values({ text })`。プレースホルダーは自動 |
| `ORDER BY id` / `WHERE id = ?` | `orderBy(asc(memos.id))` / `where(eq(memos.id, id))`。`eq` は「等しい」、`asc` は「昇順」 |
| `meta.changes === 0` | `.returning()` で消した行を返してもらい、0件かどうかを見る |
| — | `drizzle(c.env.DB)`：バインディングで渡された D1 の窓口を、Drizzle で包む |

VSCode で `rows` にマウスを乗せると、`{ id: number; text: string; createdAt: string; updatedAt: string | null }` のような型が出る。足した `updatedAt` も、何も書かずに型に入っている。`memos.id` を `memos.idx` に書き換えると、すぐ赤線が出る。

**理解の確認**：「型定義がコードに出てこなくなった。スキーマの正は schema.ts がご本尊」で合っている。`migrations/*.sql` は、schema.ts から作られる成果物という位置づけ。「何が隠れたか」を一言補足すると、**実際に流れる SQL がコードから見えなくなった**。遅いクエリを調べるときなどは、Drizzle が組み立てた SQL を確かめる必要がある。

## Terraform

### Q. ローカルに tfstate を置いておくと、別環境から terraform apply しようとしたときに困るんだったか。そもそもの Terraform の仕組みと、ローカルに tfstate を置いたときの課題、それがバケットに置かれることでどう解消されるかを整理してほしい。本質的には、サービス側で tfstate 相当のものを持ってほしいが、必ずしもそうではないので自前で用意している、ということだと理解した。CloudFormation ではスタックという概念がこれに相当するか

**Terraform の仕組み**

Terraform は**手元で動くクライアントのツール**で、3つのものを突き合わせて差分を埋める。

| | 中身 |
| --- | --- |
| コード（`*.tf`） | あるべき姿 |
| **state** | コードの名前（例：`..._application.admin`）と、実物の ID の**対応表**。前回 apply したときの値も入っている |
| 実物 | Cloudflare の API の向こうにあるリソース |

`plan` では、state を見て「このコードの管理下にある実物の ID」を知り、API でその実物の現在の値を取り直し（refresh）、コードと比べて差分を出す。

state が要るのは、**Cloudflare や AWS の API は「このリソースがどのコードから作られたか」を覚えていない**から。コードからリソースを消したとき apply で削除されるのも、state に「前回はあった」と記録されているから。

**ローカルに置いたときの課題**

指摘のとおり、別の環境から実行すると困る。

- **別の PC には state がない** → Terraform は「何も作っていない」と判断し、`plan` が全部「新規作成」になる。apply すると重複して作られるか、名前の重複でエラーになる。
- **PC を失うと対応表も失う** → 実物は残っているのに、どれが管理下か分からない。`terraform import` で1つずつ付け直すことになる。
- **同時に apply するとロックできない** → state が壊れる。
- **state には秘密情報が平文で入ることがある** → PC やバックアップに残る。git にも入れられない。

**バケットに置くと何が解消されるか**

state の正本が1か所になり、どの PC からも同じ対応表を使える。ロックもでき、PC を失っても困らない。R2 は S3 互換なので、Terraform の `s3` backend の接続先を R2 に向けて使う。ロックは、Terraform 1.10 から使える `use_lockfile = true`（バケットの中のロック用ファイルで排他する）を使う。

**理解の確認**：合っている。**本来はサービス側に持ってほしい「管理下のリソースの記録」を、Terraform が自前で持っている**、という捉え方は正確。CloudFormation の**スタック**がまさにそれで、AWS が記録・ロック・失敗時のロールバックまで持ってくれる。Terraform はどのクラウドにも使える代わりに、それを利用者が用意する。HCP Terraform（旧 Terraform Cloud）は、この部分をサービスとして肩代わりするもの。

→ [番外編：Terraform の state と、置き場所](extra-terraform-state.md)

### Q. もう一つ API キーの発行が必要なのか。Terraform には tfstate を更新する作業と、実際に Cloudflare Access をデプロイしてもらう作業の両方をやってもらう予定で、前者に必要な API キーが先ほど作ったもの、後者に必要な API キーが今から作るもの、ということで合っているか

その理解で合っている。Terraform は、**2つの別々の相手と話す**ので、鍵も2つ要る。

```
                 terraform plan / apply
                           |
          +----------------+-----------------+
          |                                  |
     backend ("s3")                    provider ("cloudflare")
     read/write the state              read/change real resources
          |                                  |
   R2 S3-compatible API               Cloudflare REST API
   key: AWS_ACCESS_KEY_ID /           key: CLOUDFLARE_API_TOKEN
        AWS_SECRET_ACCESS_KEY              (Access: Apps and Policies)
   (r2-tfstate.env)                   (terraform-cloudflare.env)
          |                                  |
   bucket "tfstate"                   Access application / policy
```

| | backend（state の読み書き） | provider（実物の操作） |
| --- | --- | --- |
| 相手 | R2 の **S3 互換 API** | Cloudflare の **API** |
| 何をするか | state を取ってくる・書き戻す・ロックする | Access のアプリやポリシーを読む・作る・変える |
| 使う場面 | `init`、`plan`、`apply` のすべて | `plan`（refresh で実物を読む）と `apply` |
| 鍵 | アクセスキー ID ＋ シークレットアクセスキー（先に作ったもの） | API トークン（これから作るもの） |

鍵を分ける理由は2つ。

- **そもそも別の API なので、鍵の種類が違う**。R2 の S3 互換の鍵では、Access の API は呼べない。逆に、Cloudflare の API トークンでは、S3 互換の API は呼べない。
- **最小権限**。state の鍵は「tfstate バケットの読み書きだけ」、provider の鍵は「Access の設定だけ」にしておけば、どちらかが漏れても被害が限られる。

AWS で Terraform を使う場合は、同じ IAM ユーザーの鍵で両方を兼ねることが多いので、この区別を意識しにくい。ただ、考え方としては、AWS でも backend と provider は別物。

変数名を `AWS_...` にしているのは、Terraform の `s3` backend が、R2 に対してもこの名前の環境変数を読むから。

鍵の一覧は、この下の「認証情報の一覧」の節にまとめた。

### Q. R2 の API トークンを作ったら、画面に「トークンの値」「アクセスキー ID とシークレットアクセスキー」「エンドポイント」の3つが出た。どう違うのか

- **アクセスキー ID とシークレットアクセスキー**：S3 互換の API 用の鍵。Terraform の backend で使うのはこちら。
- **トークンの値**：Cloudflare 独自の API で R2 を操作するための鍵。今回は使わない。
- **エンドポイント**：バケットの住所。アカウント ID が入っているだけなので、秘密ではない。

アクセスキー ID も秘密ではないが、シークレットアクセスキーは秘密。どちらも `infra/.secrets/r2-tfstate.env` に置き、使うときは `set -a; source ...; set +a` で環境変数に読み込む。

### Q. Access のアプリとポリシーは、Terraform でどう書くのか。ネットで見つかる例と書き方が違う

**ポリシー**が「誰を通すか」、**アプリケーション**が「どこを守るか」。保育園のページでダッシュボードの画面で設定していた内容と同じものを、コードで書く。

```hcl
# 誰を通すか：自分のメールアドレスだけ
resource "cloudflare_zero_trust_access_policy" "owner_only" {
  account_id = local.account_id
  name       = "sample-cf-workers-d1: owner only"
  decision   = "allow"
  include = [{
    email = { email = local.owner_email }
  }]
}

# どこを守るか：/admin 以下
resource "cloudflare_zero_trust_access_application" "admin" {
  account_id       = local.account_id
  name             = "sample-cf-workers-d1 admin"
  type             = "self_hosted"
  domain           = "${local.hostname}/admin"
  session_duration = "24h"
  policies = [{
    id         = cloudflare_zero_trust_access_policy.owner_only.id
    precedence = 1
  }]
}
```

- アプリは `policies` でポリシーの ID を参照している。Terraform はこの参照から依存関係を読み取り、ポリシー → アプリの順に作る。
- `/admin` を守ると、`/admin/...` の下のパスもすべて守られる。

**provider の v5 は、v4 から書き方が大きく変わっている**（例：`cloudflare_access_application` → `cloudflare_zero_trust_access_application`。ポリシーはアカウント単位の `cloudflare_zero_trust_access_policy` を作って、アプリから ID で参照する）。ネットで見つかる例の多くは v4 の書き方なので、混ざらないように注意する。

backend の設定の `skip_...` の6行は、「S3 の backend を AWS 以外に向けるためのおまじない」。Cloudflare の公式の手順にも同じ設定が載っている。`key` はバケットの中での state のファイル名で、サービスごとにフォルダを分けておけば、1つのバケットを全サービスで共用できる。

### Q. なぜ `tf.sh`（鍵を読み込んで terraform を実行するラッパー）を挟むのか

```bash
#!/usr/bin/env bash
set -euo pipefail
set -a
source ~/claude-sandbox/infra/.secrets/r2-tfstate.env
source ~/claude-sandbox/infra/.secrets/terraform-cloudflare.env
set +a
exec terraform "$@"
```

ターミナルで直接 `source` すると、`CLOUDFLARE_API_TOKEN` がそのターミナルに残る。**wrangler は、この環境変数があると `wrangler login` より優先して使う**。そのため、同じターミナルで `npm run deploy` すると、Access 用のトークン（Workers の権限がない）で認証しようとして失敗する。スクリプトの中で読み込めば、鍵はその実行の間にしか存在しない。

`.terraform.lock.hcl`（provider の版を固定するファイル）は git に入れる。`.terraform/` と `*.tfstate` は `.gitignore` に入れる。

### Q. plan までできた（2 to add）。apply してしまって大丈夫か

**apply して大丈夫。**plan の読み方は次のとおり。

- **`+ create` が2つ、`0 to change, 0 to destroy`**：新しく作るだけで、既存のもの（保育園のページの Access など）には一切触れない。apply の前に一番確かめるべきはここ。
- **ポリシー**：`decision = "allow"`、`include` が自分のメールアドレスだけ。意図どおり。
- **アプリ**：`domain` が `/admin`、`type = "self_hosted"`、ポリシーを precedence 1 で参照している。ポリシーの `id` が `(known after apply)` なのは、ポリシーを作ってみるまで ID が決まらないから。Terraform はこの参照を見て、ポリシー → アプリの順に作る。
- **書いていないのに出ている値**（`app_launcher_visible = true`、`http_only_cookie_attribute = true`）は、provider の既定値。`app_launcher_visible` は、Access のアプリ一覧画面（App Launcher）に表示するかどうかで、害はない。

最後の Note は、「plan の結果を保存していないので、apply のときにもう一度計算し直す」という意味。1人で使うなら気にしなくてよい。apply を実行すると、もう一度同じ plan が表示され、`yes` と入力して初めて実行される。厳密にやるなら `./tf.sh plan -out=tfplan` → `./tf.sh apply tfplan` で、確認した plan をそのまま適用できる（`tfplan` は `.gitignore` に入れる）。

apply 後に確かめること：`/admin` でログイン画面が出る（ログイン後は、`/admin` のページがまだないので 404 で正常）、`/` や `/api/memos` はログインなしで開ける、R2 の `tfstate` バケットに `sample-cf-workers-d1/terraform.tfstate` ができている。

```bash
curl -sI https://sample-cf-workers-d1.alphajinsei.com/admin | head -3     # 302 で cloudflareaccess.com へ飛ばされる
curl -sI https://sample-cf-workers-d1.alphajinsei.com/api/memos | head -1 # 200 のまま
```

### Q. apply は完了したのに、割り込み画面（Access のログイン）が発生しない。なぜか（→ 少し後に見たら表示された）

**設定が世界中のエッジに行き渡るまで、数分かかった**のだと思われる。Worker のデプロイは数秒で反映されるが、Access の設定は少し遅い。

apply の直後は、`/admin`、`/admin/`、`/admin/x` のどれも **404** で、Access のログイン画面（`cloudflareaccess.com` への 302）に飛ばされていなかった。404 の本文は `text/plain` で Hono の「404 Not Found」だったので、リクエストが Access を素通りして Worker まで届いていたことになる。このときは、原因の候補として次の3つを挙げ、`./tf.sh state show` で実物の状態を見る・ダッシュボードで保育園のアプリと見比べる、という切り分けを用意していた。

1. Access のアプリが、想定と違う形で作られている（守る対象のホスト名・パスが入っていない、など）
2. ポリシーがアプリに付いていない
3. Workers の Custom Domain で受けているホスト名に対しては、Access の効き方が Pages と違う

切り分ける前に、数分後には `/admin` だけが `cloudflareaccess.com` のログイン画面に 302 で飛ばされ、`/api/memos` はログインなしの 200 のまま、という意図どおりの状態になった。

### Q. drift 検出 → 再度 apply まで実施した。差分が表示されるだけで、実機側で修正された！という表示が出るわけではないのか

Terraform が持っているのは、**「あるべき姿（コード）」と「今の姿（実物）」**だけ。誰がいつ変えたかという経緯は知らない。そのため、普通の `plan` は「今の姿をあるべき姿に戻すには、何をするか」を出す。`~ update in-place` は「これから戻す」という予告で、「手で変えられた」という報告ではない。

（このとき試したのは、ダッシュボードで Access のアプリの Session Duration を `24h` から `12h` に変える → `plan` で `session_duration = "12h" -> "24h"` の差分が出る → apply でコードに戻す、という流れ。手の変更を正としたいなら、逆に `main.tf` を `"12h"` に書き換えれば差分が消える。）

drift そのもの（Terraform の外で何が変わったか）を見たいときは、専用の見方がある。

```bash
./tf.sh plan -refresh-only
```

state（前回 apply したときの記録）と実物だけを比べて、「**Objects have changed outside of Terraform**」という見出しで、外で変わった部分を出してくれる。コードとの比較はしないので、純粋に「手で何が変わったか」が分かる。

運用で drift を見張りたい場合は、`./tf.sh plan -detailed-exitcode` が使える。差分があると終了コードが 2 になるので、これを定期的に実行して、2 なら通知する、という仕組みが作れる（たとえば GitHub Actions の定期実行）。個人サイトでは、そこまでは要らない。

Drizzle との違いもここに出る。Drizzle は比較相手がスナップショットなので、本番に直接流した変更は検出できなかった。Terraform は毎回実物を見に行くので、検出できる。

### Q. state 用の R2 バケットを、Terraform で作らずに手で（wrangler で）作ったのはなぜか

**理解の確認**（Step 7 の理解ポイント）：「tfstate 置き場だけは Terraform では作れない」で結論は合っている。理由を一言足すと、**state 用のバケットを Terraform で作ろうとすると、その作業自体の state を置く場所が、まだ存在しない**から（鶏と卵）。

どうしても Terraform で作りたい場合は、「置き場を作るだけの小さな設定」を別に用意し、その設定の state だけはローカルに置く。個人なら、wrangler で1回作るほうが簡単。

### Q. wrangler と Terraform の両方で同じものを管理すると、両方でドリフトが発生する。今回「Worker・D1・カスタムドメインは wrangler、Access などは Terraform」というすみわけにしている理由は何か。単に勉強目的か

勉強目的だけではない。**それぞれのツールが得意なもの**で分けている。

| | wrangler | Terraform |
| --- | --- | --- |
| Worker のコード | ◎ ビルドしてデプロイまで一度にやる | △ 作れるが、ビルド済みのコードを別に用意して渡す必要がある |
| D1 のマイグレーション | ◎ | × 扱えない |
| バインディング | ◎ `wrangler.jsonc` に書くだけ | △ 書けるが冗長 |
| カスタムドメイン | ◎ DNS レコードと証明書まで自動 | △ 個別に定義する |
| Access | × 扱えない | ◎ |

そもそも `wrangler.jsonc` 自体が、Worker まわりの宣言的な定義（IaC）になっている。そのため、Worker まわりは wrangler、wrangler で扱えないもの（Access など）は Terraform、と分けるのが自然。

両方から同じものを管理したときに起きるのは、指摘のとおり drift。もっと具体的に言うと、**互いに相手の変更を「ずれ」と見なして戻し合う**。たとえば、カスタムドメインの DNS レコードを Terraform でも定義すると、wrangler が自動で作ったレコードとぶつかる。Step 6 の「デプロイの経路を混ぜると後勝ちになる」話と同じ構造。

ただし、Terraform を入れた動機の一部が勉強目的なのも事実。個人サイトの Access が1つ2つなら、ダッシュボードで設定しても十分。サービスが増えて Access の設定が増えたとき、コードで管理しておく価値が出てくる。

## OCI（方式③）

### Q. OCI の Always Free で、LB があって、サーバがあって、DB があって、という構成を作れないか。すでに VM 2台（VM.Standard.E2.1.Micro が2つ）と HeatWave が立っている。Free Tier はマイクロだけだと昔は聞いたが、今はこれにプラスして Ampere A1 も別にあげられるのか。2 OCPU だと、2台上げようと思えば上げられるか

**はい、別枠。**マイクロ（E2.1.Micro）2台とは別に、A1 を計2 OCPU / 12GB まで使える（2026-11-03 に公式の Always Free の一覧で確認）。1 OCPU / 6GB を2台にする、といった分け方ができる。

公式の一覧を見て、先に気になった点が2つあった。

1. **PostgreSQL は Always Free に入っていない。**無料なのは **MySQL HeatWave**（1台、50GB）。立っている DB が「Database with PostgreSQL」の方だと、課金されている可能性がある（確認したら HeatWave MySQL だった）。
2. **Ampere A1 の上限が下がっていた。**以前は計4 OCPU / 24GB だったが、今の記載は**計2 OCPU / 12GB**（最大2台）。ほかに AMD のマイクロ VM（1/8 OCPU、1GB）を2台まで作れる。

**気をつけたいのはディスク**（次の問いを参照）。また、東京などのリージョンでは、A1 で「Out of host capacity」（空きがない）のエラーがよく出る。出たら時間を置くか、別の AD で試す。

### Q. 既存の1台目はブートボリュームが 47G、ブロックボリュームが 50G、2台目はブートボリュームが 47GB。ブロックボリュームにだけ Always Free の文言が書いてあった。公式ページの「Up to 2 block volumes, 200 GB total」を読むと、ブロックボリュームが 200G と読めるので、まだ余裕はあるのでは

**ブートボリュームとブロックボリュームを合わせて 200GB** という意味。公式ドキュメントにも「200 GB combined boot + block volume storage」とある。「2つまで」の数の制限はブロックボリュームにだけかかり、ブートボリュームも容量には数えられる。

| | 今 | A1 2台を足すと |
| --- | --- | --- |
| マイクロ1台目 | ブート 47 + ブロック 50 | 97 |
| マイクロ2台目 | ブート 47 | 47 |
| A1 ×2 | — | 47 × 2 = 94 |
| 合計 | 144GB | **238GB（38GB 超える）** |

ブートボリュームは 47GB より小さくできない（1台 47GB 以上）ので、**VM の台数はコアよりディスクで詰まる**。50GB のブロックボリュームを外せば 188GB で収まる。

中身の確認を省くなら、ブロックボリュームの**バックアップを取る**（Always Free でバックアップが5つまで無料）→ デタッチして削除する → 必要になったらバックアップから戻す、の順がよい。「何を残すか」の判断を後回しにでき、資産も失わない。

**理解の確認**：「ブートボリュームも込みで 200G」で合っている。

### Q. VM-A、B は、コンテナイメージがかさんでくると、ブートボリュームだけでは足りなくなってきたりしないか。最初からブロックボリュームをアタッチしておいた方がよくて、それだとさらに容量が必要、ということにならないか

まず大丈夫。

- イメージは1つ数百MB。アプリ10個 × 3世代を残しても 10GB ほど。デプロイのときに `docker image prune` で古いものを消す。
- 足りなくなっても、**ブートボリュームはあとから止めずに大きくできる**（47GB → 80GB など）。容量のために、別のブロックボリュームを最初から付けておく必要はない。
- 別のボリュームが向くのは、**VM を作り直しても残したいデータ**がある場合。今回だとメールボックスがそれに当たる。Docker ホストのデータは HeatWave と Object Storage に置くので、作り直しても消えない。

### Q. 作業用のサーバは1つ残したい、コンテナを浮かべる基盤が欲しい（最悪冗長性は捨てる）、メール・認証を今後実装したい。Always Free の範囲でどうやりくりするのがよいか。特に VCN とサブネットの設計も考えてほしい

**VM の割り振り**

| VM | シェイプ | 役割 | ディスク |
| --- | --- | --- | --- |
| **ops**（作業用） | マイクロ（1/8 OCPU、1GB） | 勉強用に自由に使う。DB に mysql クライアントでつなぐ管理作業にも使う | ブート 47 |
| **app**（Docker ホスト） | A1 1 OCPU / 8GB | コンテナの基盤。認証基盤もここに置く | ブート 47（必要になったら増やす） |
| **mail**（単体サービス用） | A1 1 OCPU / 4GB | メールサーバー。今後の「VM 単体で建てたいもの」もここに置く | ブート 47 |

- **冗長化はあきらめる。**ただし、LB は残す価値がある。TLS の終端、ホスト名での振り分け、ヘルスチェックを LB が引き受けるので、あとで A1 の割り振りを変えて Docker ホストを2台に戻すときも、バックエンドを足すだけで済む。
- **メモリは Docker ホストに多めにする。**Keycloak や、あとで作る自作の認証基盤が載るから。メールは Postfix + Dovecot + Rspamd なら 4GB で足りる。
- **既存のマイクロ2台は、中身を確かめずに片付けられる。**先にブートボリュームとブロックボリュームのバックアップを取り（5つまで無料）、そのあと全部消し、ops は新しい VCN に作り直す。

（最初の案では mail にメールのデータ用のブロックボリュームを付けていたが、後でやめた。下の「メールのブロックボリュームが大きすぎないか」を参照。）

**VCN とサブネット**

VCN は1つ（`10.0.0.0/16`）にして、役割ごとにサブネットを分ける。

| サブネット | 置くもの | 外への出口（ルート表） | 分ける理由 |
| --- | --- | --- | --- |
| public-lb（10.0.0.0/24） | LB | IGW | 外から受ける入口 |
| public-dmz（10.0.1.0/24） | mail | IGW | 外から直接受ける VM の置き場（DMZ）。メールは HTTP ではないので LB を通さない |
| private-app（10.0.10.0/24） | Docker ホスト | NAT + Service GW | 公開 IP を持たせない |
| private-ops（10.0.30.0/24） | 作業用 | NAT + Service GW | 勉強で雑多なことをする VM を、本番のアプリから分ける |
| private-db（10.0.20.0/24） | HeatWave | なし | DB は外に出る必要がない |

細かい通信の許可は NSG でかける。サブネットを分ける理由は、主に「出口（ルート表）が違うこと」と「見通しのよさ」。ひとつ制約があり、**IGW と NAT は同じサブネットのデフォルトの出口に同居できない**（1つのサブネットのデフォルトの出口は、どちらか一方）。そのため、外から直接受ける mail は公開側のサブネットに置く。

出口は3種類。

- **Internet GW**：公開サブネットの出入口。
- **NAT GW**：非公開サブネットの VM がインターネットへ出る口（apt、Docker、外の API）。
- **Service GW**：OCI のサービス（OCIR、Object Storage、Vault など）へ、インターネットを通らずに出る口。

入るのは Bastion から（app / ops / mail の 22 番）。

| NSG | 受け付ける | 相手 |
| --- | --- | --- |
| nsg-lb | 443 | Cloudflare の IP 範囲 |
| nsg-app | 8000〜8099 | nsg-lb |
| | 22 | Bastion |
| nsg-mail | 25、465、587、993 | 全体 |
| | 22 | Bastion |
| nsg-ops | 22 | Bastion |
| nsg-db | 3306 | nsg-app と nsg-ops |

### Q. ブロックボリュームのアタッチとか、LV を切るとか言った作業が発生する点が気になった。Ansible を導入するか。その方が見通しがよいように思う。また、将来バッチ処理をやりたくなった場合、ops から実行できるか

- **Ansible は入れる。**VM が ops・app・mail の3種類になり、mail はメールの設定や証明書の更新が要る。さらに「自分で管理する基盤を持ち、OS の更新や脆弱性対応で素養を身に付ける」という目的がある。こうなると、VM の中の設定をコードで持てる Ansible の方が見通しがよい（その前の案では、VM の設定は cloud-init だけにして Ansible は使わない、としていた）。分担はこうする。
  - Terraform：リソースを作る。ブロックボリュームのアタッチまで（準仮想化でアタッチすれば、iSCSI のコマンドは要らない）
  - Ansible：VM の中の設定。フォーマット、マウント、Docker、メール、OS の更新
  - LVM は不要。ボリューム1つに直接ファイルシステムを作り、容量を増やしたいときは OCI 側で大きくしてから `resize2fs` を流す。
- **ops は「作業用 兼 バッチサーバー」にできる。**DB にも届く位置に置いてある。マイクロ（1/8 OCPU、1GB）なので、cron や systemd のタイマーで回す軽い処理に向く。重い処理は app のコンテナとして動かし、ops からはそれを起動するだけにする。定時処理はアプリの中に持たず、ops の cron から起動する、という約束を共通基盤の仕様に入れた。Docker ホストを2台に戻したとき、二重に実行されないようにするため。

### Q. コンソールで ops には「Always Free」のラベルが付いているが、app と mail には付いていない。なぜか

これは正常。コンソールの「Always Free」のラベルは、**マイクロ（E2.1.Micro）のように、シェイプごと無料と決まっているものにだけ付く**。A1 は無料の仕組みが違うので、ラベルは付かない。

| | マイクロ（ops） | A1（app、mail） |
| --- | --- | --- |
| 無料の仕組み | シェイプそのものが無料 | 普通の有料のシェイプで、**毎月の使用量のうち一定量までが無料** |
| 無料の範囲 | 2台まで | 月 1,500 OCPU 時間・9,000 GB 時間まで（= 2 OCPU / 12GB を1か月動かし続ける分） |
| ラベル | 付く | 付かない |

今の割り振り（app 1/8GB、mail 1/4GB、合計 2 OCPU / 12GB）なら、31日の月でも 2 × 744 = 1,488 OCPU 時間、12 × 744 = 8,928 GB 時間。無料の範囲に収まる。ただ、ほぼ上限いっぱいなので、**A1 の OCPU やメモリーを足すと、その分は課金される。**

確かめ方：Cost Analysis でサービス別（または SKU 別）に見る。「Compute - Ampere A1」の行に使用量は出ていても、金額は0円になっているはず。

**理解の確認**：「課金体系が違う。flex タイプだと画一的に無料とできない」で合っている。

### Q. すでに有料アカウント（Pay As You Go）に移行していて、Free Tier の枠内を使いたい、という意図。使用率の低い VM が回収されるリスクはあるか

無料のみのアカウントだと、7日間の CPU・ネットワーク・メモリ（A1 のみ）の使用率（p95）が 20% を下回る VM は回収されることがある。小さいアプリはほぼ確実にこれに当たる。よくある対策は**アカウントを Pay As You Go に上げ、予算アラート（例：1円）を設定する**こと。無料枠の中なら請求は0円のままで、回収の対象からも外れる。

すでに有料アカウントなので、この懸念はない（実際、2023年から作っている VM 2台は回収されていない）。そのかわり、作り間違えて枠を超えると課金されるので、**予算アラート**（Cost Management の Budget。これも無料）を、何かを作る前に置いておく。

### Q. OCI LB が名前解決先になる。これはレイヤの高い LB か

**OCI Flexible LB は L7（HTTP/HTTPS）の LB。**Host ヘッダーやパスで振り分けられる。L4 の方は Network Load Balancer という別の製品（無料で1台）。HTTP 以外の通信（SMTP、ゲーム、MQTT など）を受けたいときは、Network Load Balancer か VM の公開 IP を使う。

### Q. アプリごとの振り分けはどうするか。アプリが開発されるたびに alphajinsei.com のサブドメインを切って、LB でサブドメインごとに振り分ける形か。パスベースルーティングの案と比べて、どちらが有利か。また、どちらで振り分けるにせよ、受け側をどう作るか。k8s なら Service や nginx ingress で自然に解決されるが、ここでは素で作るしかない。浮かべるコンテナの外部公開ポートが、サービスごとにかぶらないように設定するイメージか

**サブドメインがよい。**

| 観点 | サブドメイン（`aaa.alphajinsei.com`） | パス（`apps.alphajinsei.com/aaa`） |
| --- | --- | --- |
| アプリ側の作り | 何もしなくてよい。ルートが `/` のまま | アプリ全体に「`/aaa` の下で動く」設定が要る（フレームワークの base path、リンク、静的ファイル、リダイレクト）。漏れると壊れる |
| アプリどうしの分離 | オリジンが別なので、Cookie や localStorage が混ざらない | 同じオリジンを共有するので、あるアプリの Cookie が別のアプリから見える |
| 方式をまたいだ移動 | DNS を向け直すだけで、①②と③の間でアプリを動かせる | URL が変わる |
| 証明書 | Cloudflare の Origin 証明書 `*.alphajinsei.com`（最長15年）1枚で全部まかなえる | 1枚 |
| 足すときの作業 | DNS レコード1つと LB の設定。どちらも Terraform で書ける | LB の設定だけ |

パス方式の利点は「DNS を触らなくてよい」くらい。それも Terraform で消えるので、サブドメインの方が有利。

**受け側は、イメージどおりでよい。**`aaa` は VM の `:8081`、`bbb` は `:8082`、のようにアプリごとにポートを決め、LB に**アプリごとのバックエンド・セット**を作る。

もう一つの案として、各 VM に Traefik（リバースプロキシ）を置き、Host を見てコンテナに振り分ける ingress 風の形もある。ただこれだと、LB のヘルスチェックが見るのは Traefik だけになる。アプリ aaa だけが死んでも、LB は気付かずに送り続ける。アプリごとのバックエンド・セットなら、**LB がアプリ単位で死活を見る**。ポートは台帳で管理する（共通基盤の README に一覧の表を置く）。

→ [方式検討：OCI](../design/oci.md)

### Q. LB から VM への穴あけはどうやるのか

**NSG（ネットワーク・セキュリティ・グループ）**を使う。

- **セキュリティ・リスト**はサブネットにかけるルール、**NSG** は VM や LB などのリソース単位にかけるルール。
- NSG では、相手を IP ではなく「**どの NSG に属するか**」で指定できる（例：`nsg-app` は `nsg-lb` からの 8000〜8099 を受ける）。
- どちらもステートフル（行きの通信を許せば、戻りは自動で通る）。
- アプリ用のポートは **8000〜8099 の範囲で一度だけ開けておく**。そうすれば、アプリを足すたびに NSG を触らずに済む。

**OCI のハマりどころ**：OCI の VM は、VM の中のファイアウォールでも 22 番以外を拒否するのが初期設定（Ubuntu のイメージは iptables、Oracle Linux は firewalld）。NSG を開けても、VM の中も開けないと届かない。なお、Docker で公開したポートは、Docker が自分で iptables に穴を足す。そのため Ansible で iptables に足したのは mail のポート（25、465、587、993）だけで、app の 8000〜8099 は書いていない。

ほかに、VCN の既定のセキュリティ・リストは「SSH を 0.0.0.0/0 から許可」のルールを OCI が入れるので、これを消し、ICMP（経路の MTU の調整用）だけを残した。通信の許可は NSG だけで行う。

**理解の確認**：「NSG で理解した。アプリ用のポートは広めに開けておく」で合っている。

### Q. Terraform の state は、OCI なので Object Storage を使うのはどうか

賛成。Terraform には `oci` という公式の backend があり、ロックもできる。HashiCorp も Oracle もこれを推奨している（S3 互換の方法は非推奨）。バケットだけは最初に CLI で手作りし、バージョニングを有効にする（R2 のときと同じ「鶏と卵」の問題）。OCI の中で閉じるので、③にはこちらが合う。

### Q. SSH は、Cloudflare Tunnel はいざというときにログインできない気がして面倒。Tunnel も Bastion も使ったことがないが、いったん Bastion にする。最悪マイクロの VM から ssh する穴を通す、でどうか

**Bastion にする。**

- **仕組み**：Bastion は OCI が持つ踏み台。「この人がこの VM の 22 番に、最長3時間だけ入ってよい」という**セッション**を作り、その間だけ通れる。VM 側には公開 IP も、外に開いたポートも要らない。
- **いざというとき**：Bastion が使えなくても、**コンソール接続**（インスタンスのシリアルコンソール）がある。OCI のコンソールから、ネットワークを通らずに VM へ入れる仕組みで、非常口にはこれを使う。マイクロから SSH する穴は、マイクロがいる古い VCN と新しい VCN をつなぐ必要があり、手間が増えるので作らない。
- **手間を減らす工夫**：小さなスクリプト（`bin/bastion-proxy`）を ssh の ProxyCommand から呼び、使えるセッションが残っていれば使い回し、なければ作って待つ。これで `ssh oci-app` のように入れる。
- **Bastion を使っても NAT は要る**：Bastion は「入ってくる」道。VM が外へ出る道は別に必要。OCIR・Object Storage・Vault へは Service Gateway を通り、apt や Docker のインストールはインターネットに出るので NAT Gateway を通る。

（Claude の最初の案は Cloudflare Tunnel だった。Bastion はセッションの有効期限が最長3時間で、毎回 CLI でセッションを作ってから繋ぐ形になるので、スクリプトが毎回 ssh する用途には合いにくい、という理由。ユーザーは「Cloudflare Tunnel は自宅サーバをデータセンターのようにするための用途で、今回のようなユースケースは枝葉の使い方」と考えて Bastion を選んだ。）

### Q. 業務で使うのが RHEL なので、Oracle Linux にしたい。Oracle Linux だと困るところはあるか（→ 一度切り替えたあと）SELinux と firewalld が混在して変なことになったりしないか。Oracle Linux を起動すると SELinux だか firewalld に引っかかるのがお決まりのやつ。コンテナランタイムも podman になってしまうか。自分が慣れていないだけなので、自分の状況は外して検討するとどうなるか

**SELinux と firewalld は役割が別で、互いにはぶつからない。**

| | 何をするか | Ubuntu で当たるもの |
| --- | --- | --- |
| SELinux | プロセスが、どのファイルやポートに触れてよいかを決める（強制アクセス制御） | AppArmor（ずっと緩い） |
| firewalld | パケットを通すかどうかを決める | iptables / ufw |

「お決まりの、引っかかるやつ」は、それぞれが **Docker やメールサーバーとぶつかる**ことで起きる。

- **SELinux**：コンテナにホストのフォルダを渡すと、`:Z` を付けない限り読み書きできない。Postfix や Dovecot に標準外の場所（`/srv/mail`）を使わせるには、ラベルを付ける必要がある。どちらも、エラーの原因に気付くまでが長い。
- **firewalld と Docker**：firewalld を reload すると Docker が足したルールが消え、Docker の再起動が要る。nftables との相性の問題も、ときどき出る。

今回は外側を NSG で絞っているので、VM の中のファイアウォールは二重の守り。ここで引っかかるのは割に合わない。

**podman**：OL9 の OCI イメージに podman は初めから入っていないことが多く、入っていても外せば済む。ただ、**Docker が公式にサポートしている OS は RHEL・CentOS・Fedora で、Oracle Linux は入っていない。**実際には動くが、Oracle が推しているのは podman。

**中立に比べると、Ubuntu の方が見通しがよい。**

| 観点 | Ubuntu 24.04 | Oracle Linux 9 |
| --- | --- | --- |
| Docker | ◎ 公式サポート。手順も情報も一番多い | ○ 動くが公式サポートの外 |
| 情報の多さ（コンテナ、メールサーバー、Arm） | ◎ 例の多くが Ubuntu / Debian 前提 | ○ |
| 引っかかりやすさ | ◎ AppArmor は緩い（ただし Docker が ufw を素通りする癖はある。今回は ufw を使わないので関係ない） | △ SELinux と firewalld への対処が要る |
| マイクロ（1GB） | ○ apt は軽い | △ dnf が重い |
| カーネルを再起動なしで更新する仕組み | Livepatch（Ubuntu Pro。個人なら5台まで無料） | Ksplice |
| 業務との近さ | | ◎ |

一般論では、**コンテナを載せる基盤なら Ubuntu が素直**。「情報の多い主流の道具を選ぶ」という方針にも合う。RHEL 系を学ぶこと自体が目的なら OL に意味があるが、今回の目的（アプリ開発で基盤のことを考えない、基盤を自分で回す）なら Ubuntu に戻すのがおすすめ。→ Ubuntu 24.04 に戻した。

### Q. dnf-automatic（自動更新）はやるのか。Docker 上に浮かんでいたコンテナたちが、起動し直しになってしまわないか。ずっと起動しっぱなしの運用で、セキュリティ更新はたまに実施、と思っていた

先に、何が起きるかを整理する。

| 更新されるもの | コンテナへの影響 |
| --- | --- |
| 普通のパッケージ（openssl など） | なし。ホストのプロセスが再起動されるだけ |
| Docker 本体（docker-ce、containerd） | **Docker デーモンが再起動し、コンテナも再起動する**。`live-restore` を設定すれば、コンテナは動いたまま |
| カーネル | 再起動するまで効かない。**再起動すればコンテナも再起動する**（`restart: unless-stopped` で戻る。1〜2分止まる） |
| コンテナの中身 | **OS の更新では直らない。**イメージを作り直して入れ替える（デプロイ） |

「起動しっぱなしで、セキュリティ更新はたまに」という運用も成り立つ。ただし、mail は公開 IP で外に直接出ていて、更新を忘れたときの危険が一番大きい VM。

**おすすめは中間。**

- **自動にするもの**：セキュリティ更新だけ。**自動再起動はしない。**Docker 本体は自動更新の対象から外し、`live-restore` も設定する。これで、勝手にコンテナが止まることはなくなる。
- **手動にするもの**：再起動とカーネル・Docker の更新。`ansible-playbook upgrade.yml` を1本用意し、「全 VM を更新し、再起動が必要なものだけ1台ずつ再起動する」を1コマンドにする。月1回など、都合のよいときに流す。再起動が必要かどうかは、`/var/run/reboot-required` で分かる。

「基盤を自分で回して素養を身に付ける」という目的にも、手動で流すこの1コマンドが合う。

### Q. メールのブロックボリュームが 50G で大きすぎないか。Terraform のコメントを見ると 50G が最小らしい。あとどれだけ余力があるか。メールにそんなに大容量を使うとは思えず、あきらめてブートボリュームだけにする案もある。ブートボリューム自体を拡張することもできる。ただ、あとからメール領域をブロックボリュームに移すとなると面倒か

ブロックボリュームは**やめる**方がよい。ブロックボリュームは最小 50GB で、小さなデータのために付けると、Always Free のディスクの枠（200GB）を大きく食う。

- **今の余力**：47 × 3 + 50 = 191GB で、残りは **9GB だけ**。ブロックボリュームをやめると 141GB になり、**59GB 空く**。
- **ブロックボリュームの利点は「VM を作り直してもデータが残る」ことだけ**。その代わりに、ブートボリュームのバックアップ（5つまで無料）と、メールのデータを定期的に Object Storage へ退避する仕組みで守れる。
- **あとから移すのも難しくない。**メールのデータは最初から `/srv/mail` という1つのフォルダにまとめておく。移すときは、ボリュームを付ける → メールを止める → `rsync` でコピー → `/srv/mail` にマウントし直す、の順で、30分ほどの作業。
- 容量が足りなくなったら、ブートボリュームを止めずに大きくできる。

### Q. 自動更新を入れることで、夜間に OS 再起動 / Docker 再起動がスケジュールされるようになり、もともと上がっていたコンテナ群が再起動後に止まった状態になって、サービスが止まってしまうことを気にしていた。その対策として、再起動自体が発生しないようにする、ということを言っているのか

その理解で合っている。ただ、対策は2段構えにしている。

**1段目：自動更新では、再起動そのものを起こさない**

| 再起動 | 起きるきっかけ | 設定 |
| --- | --- | --- |
| OS の再起動 | カーネルなどを更新したあと、「再起動したら効く」状態になる | **自動で再起動しない**（`Automatic-Reboot "false"`）。更新は入るが、効くのは手で再起動したとき |
| Docker の再起動 | ① Docker 本体の更新<br>② ライブラリの更新後、needrestart が「古いライブラリを使っているサービス」を再起動する | ① **Docker 本体を自動更新から外す**<br>② **needrestart に Docker を再起動させない** |

この結果、夜中の自動更新でやることは「パッケージを入れ替えるだけ」になる。OS も Docker も再起動しないので、コンテナはそのまま動き続ける。万一 Docker のデーモンが再起動しても、`live-restore`（デーモンが再起動してもコンテナを動かしたままにする設定）でコンテナは止まらない。

**2段目：再起動しても、コンテナは自分で起き上がる**

再起動は、自動更新のほかにも起こる。手で流す `upgrade.yml`、OCI 側のメンテナンス（まれに事前通知のうえで再起動される）、障害など。そのときに「止まったまま」にならないよう、次の2つを使う。

- **Docker は OS の起動時に自動で起動する**（`systemctl enable docker`。Ansible で設定済み）。
- **各コンテナに再起動ポリシー `restart: unless-stopped` を付ける**（アプリの `compose.yaml` に書く）。Docker が起動すると、このポリシーのコンテナは自動で起動し直される。自分で `docker stop` したものだけは、止めたままになる。

再起動のときにサービスが止まるのは、OS が上がってコンテナが起動するまでの1〜2分だけ。

**まとめ**

- **自動更新**：再起動を起こさない。コンテナは止まらない。
- **再起動（手動・メンテナンス・障害）**：1〜2分止まるが、コンテナは自動で戻る。
- **手でやること**：月に1回などで `upgrade.yml` を流し、カーネルと Docker 本体の更新を効かせる（カーネルの更新は、手で再起動するまで効かない）。

### Q. Bastion 経由の ssh で、セッションの作成が InvalidParameter（invalid resource name (displayName)）になった。心当たりはあるか

原因は、セッションの名前が数字で始まっていたこと。名前を `10.0.10.10-22` にしていた。メッセージにある「使える記号」（`.` と `-`）しか使っていないので、残る候補は「先頭が数字」で、Bastion の名前は英字で始める必要がある、と見るのが自然。名前を `ssh-10-0-10-10-22` のように、英字で始まり、ドットを使わない形にしたら通った。

（最初はスクリプトが oci CLI のエラーの中身を握りつぶしていたので、失敗したときに CLI のメッセージをそのまま出すように直してから、原因が見えた。）

### Q. パラメタエラーは突破したが、今度は `Permission denied (publickey)` になった

原因は鍵の種類。**OCI の Bastion は、セッションの鍵に RSA しか受け付けない。**ed25519 の鍵でもセッション自体は作れてしまうが、Bastion への接続で `Permission denied (publickey)` になる。

VM へのログインは ed25519 のままで問題ないので、**Bastion 用にだけ RSA の鍵を足した。**

| 鍵 | 使う場面 |
| --- | --- |
| `~/.ssh/oci-platform`（ed25519） | VM へのログイン（VM に登録済み） |
| `~/.ssh/oci-platform-bastion`（RSA） | Bastion のセッションと、Bastion への接続（スクリプトが自動で使う） |

スクリプトも、既存のセッションは登録されている公開鍵が今の鍵と同じときだけ使い回す（ed25519 で作ったセッションを拾わないため）、ACTIVE になった直後は鍵の反映が間に合わないことがあるので数秒待つ、と直した。

### Q. 入れるようになったが、Claude 任せで結局何が起きたのか分からなかった。ラッパースクリプトでラップされていて、何が起きたのかよく分からない。そもそも手で接続する場合どんな手順になるのか。そのうち別環境からも接続したくなるので、Claude なしでどうやって接続するか自分でも理解しておきたい

**仕組み：SSH を2段で使っている**

```
PC ==(1) ssh as <session OCID>, RSA key==> Bastion (host.bastion.ap-tokyo-1.oci.oraclecloud.com)
                                             |  forwards TCP only to 10.0.10.10:22
PC ==(2) ssh as ubuntu, ed25519 key ====================================> app VM (10.0.10.10)
         (2) runs inside the tunnel made by (1)
```

- **(1) Bastion への SSH**：ユーザー名は「セッションの OCID」で、鍵はセッションを作るときに登録した RSA の鍵。Bastion は、そのセッションで許された宛先（10.0.10.10 の 22番）へ、TCP をつなぐだけ。
- **(2) VM への SSH**：(1) のトンネルの中で、普通に VM へ SSH する。ユーザーは `ubuntu`、鍵は VM に登録した ed25519。
- (2) の暗号は PC と VM の間で直接かかるので、Bastion から中身は見えない。認証も2回で、Bastion を通る許可と VM へのログインは別々。

**手順1：セッションを作る（3時間有効）**

```bash
BASTION=$(terraform -chdir=$HOME/claude-sandbox/oci-platform/terraform output -raw bastion_id)
SESSION=$(oci bastion session create-port-forwarding --bastion-id "$BASTION" \
  --ssh-public-key-file ~/.ssh/oci-platform-bastion.pub \
  --target-private-ip 10.0.10.10 --target-port 22 \
  --session-ttl 10800 --display-name ssh-app --query 'data.id' --raw-output)
# ACTIVE になるまで待つ（1分ほど）
oci bastion session get --session-id "$SESSION" --query 'data."lifecycle-state"' --raw-output
```

コンソールでも作れる（Bastion → Create session → 「Port forwarding」→ IP とポートを指定 → 公開鍵を貼る。できたセッションの「Copy SSH command」で接続のコマンドが手に入る）。

**手順2：つなぐ**

A. トンネルを張ってからつなぐ（仕組みが見えやすい）

```bash
# terminal 1: -N = run no command, keep the tunnel open
ssh -i ~/.ssh/oci-platform-bastion -N -L 2222:10.0.10.10:22 -p 22 \
  "$SESSION"@host.bastion.ap-tokyo-1.oci.oraclecloud.com
# terminal 2: local 2222 is connected to the VM's 22
ssh -i ~/.ssh/oci-platform -p 2222 ubuntu@localhost
```

B. 1行でつなぐ（ProxyCommand。スクリプトが最後にやっているのはこれ）

```bash
ssh -i ~/.ssh/oci-platform \
  -o ProxyCommand="ssh -i ~/.ssh/oci-platform-bastion -W %h:%p -p 22 $SESSION@host.bastion.ap-tokyo-1.oci.oraclecloud.com" \
  ubuntu@10.0.10.10
```

`-W %h:%p` は、「Bastion 経由で宛先の host:port につなぎ、標準入出力をそのまま流す」という指定。外側の ssh が、それをトンネルとして使う。

**ラッパー（`bin/bastion-proxy`）が自動でやっていること**：手順1（同じ宛先・同じ鍵で残り15分以上のセッションがあれば使い回し、なければ作って待つ）と、手順2-B（`ssh_config` の ProxyCommand から呼ばれ、最後に `ssh -W` を実行する）。さっきのエラー2つも、手順1（名前が数字で始まっていた）と手順2の (1)（Bastion が ed25519 の鍵を受け付けなかった）で起きていた。

**別の環境から接続するには**

- **oci CLI の認証**：新しい PC 用に API キーを作って、コンソールに登録する（1ユーザーに3つまで登録できる）。
- **SSH の鍵**：今の2つの鍵を安全にコピーするか、新しい鍵を作る。新しい鍵の場合、Bastion 用は手順1で登録するだけなので、そのまま使える。VM 用は、VM の `~/.ssh/authorized_keys` に公開鍵を足す必要がある。
- **手軽な代わり：OCI Cloud Shell**。コンソールの右上から開けるブラウザ上のシェルで、oci CLI が認証済みで入っている。手順1と2をそのまま実行できる（鍵は Cloud Shell に置く必要がある）。

→ [番外編：OCI Bastion で VM に SSH する](extra-oci-bastion.md)

### Q. 結構面倒。正直、直接 ssh を 10022 とかでインターネットに公開したほうが楽な気がした

直接の SSH 公開も、実際よく使われる選択肢。「鍵認証だけにして、NSG で自宅の IP に絞る」なら十分に現実的。ただ、ポートを 10022 に変えるだけでは、守りとしてはあまり効かない。今は Bastion で進めて、面倒さが勝つようなら後で見直す。

### Q. `ansible-playbook site.yml` って何をやっているのか

`site.yml` は「どの VM に、どの役割（role）を当てるか」の一覧。Ansible は次の流れで動く。

1. `inventory.ini` から、対象の VM（`oci-app`、`oci-mail`、`oci-ops`）を読む。
2. それぞれに **ssh で入る**。`~/.ssh/config` を使うので、Bastion 経由で `bastion-proxy` が呼ばれ、セッションを作るか使い回す。
3. **sudo（`become: true`）で**、タスクを上から順に実行する。
4. 各タスクは「あるべき状態」を書いたもので、**すでにそうなっていれば何もしない**。結果は、何もしなかったものが `ok`、変更したものが `changed` と表示される。2回目に流すと全部 `ok` になる。

| 対象 | role | やること | 変わる場所 |
| --- | --- | --- | --- |
| 3台全部 | common | タイムゾーンを Asia/Tokyo にする | `/etc/localtime` |
| | | `unattended-upgrades`、`needrestart`、`iptables-persistent`、`jq` を入れる | パッケージ |
| | | 自動更新の設定：セキュリティの更新のみ、自動再起動なし、Docker 本体は対象外 | `/etc/apt/apt.conf.d/52unattended-upgrades-local` |
| | | needrestart に Docker を再起動させない | `/etc/needrestart/conf.d/docker.conf` |
| | | iptables でポートを開ける（**mail だけ**：25、465、587、993）。変えたら iptables を読み直す | `/etc/iptables/rules.v4` |
| app と ops | docker | Docker の公式の apt リポジトリを登録して、Docker を入れる | `/etc/apt/sources.list.d/docker.sources`、パッケージ |
| | | ログの設定（local ドライバー）と `live-restore` | `/etc/docker/daemon.json` |
| | | `ubuntu` を docker グループに入れる（sudo なしで `docker` が使える） | `/etc/group` |
| | | Docker を起動し、OS の起動時にも自動で起動する | systemd |
| ops | swap | 2GB の swap ファイルを作って有効にし、起動時にも有効にする。swap を使いにくくする（`vm.swappiness=10`） | `/swapfile`、`/etc/fstab`、sysctl |
| mail | （直接書いたタスク） | `/srv/mail` を作る | `/srv/mail` |

流す前に、何が変わるかだけ見る方法：

```bash
cd ~/claude-sandbox/oci-platform/ansible
ansible all -m ping                       # 3台に入れるかだけ確かめる（pong が返れば OK）
ansible-playbook site.yml --check --diff  # 実際には変えずに、変わる予定のものと差分を表示する
ansible-playbook site.yml                 # 本番
```

`--check` には注意が一つある。Docker のように「パッケージを入れてから、その設定をする」タスクは、入れる前の段階で止まるため、途中でエラーに見えることがある。全体の雰囲気をつかむためのものと考える。

### Q. `ssh oci-mail` のように Bastion 経由でログインするスクリプトを書いてもらったが、数分無操作だと connection closed になって不便。サーバ側の設定を変えるのか、クライアント側で keepalive のオプションを足すとかで対策できないか

**クライアント側に keepalive を足せば直る。サーバ側の設定変更は要らない。**

今の接続は二重になっている（外側の ssh が VM の sshd へ、その ProxyCommand の内側の `ssh -W` が Bastion 経由で VM の 22 番へ）。何も打たないでいると、この経路には1バイトも流れない。Bastion などの途中の機器は、通信のない TCP 接続を一定時間で切ることが多いので、数分後に `Connection closed` になる（切れるまでの時間の短さから見て、Bastion 側の無通信タイムアウトだと推測した）。

対策は、外側の ssh に `ServerAliveInterval` を足すこと。外側の ssh が60秒ごとに暗号化された keepalive を VM の sshd へ送るようになり、このパケットは Bastion と内側の ssh も通るので、経路全体が「通信あり」の状態に保たれる。

```
  ServerAliveInterval 60    # send a keepalive every 60s
  ServerAliveCountMax 3     # give up after 3 missed replies
```

- **クライアント側でやる理由**：設定するのは `ssh_config` 1か所だけで、mail・app・ops すべてのホストに効く。VM の sshd に `ClientAliveInterval` を入れても同じ効果はあるが、Ansible で全 VM に配る手間がかかる。
- `ServerAliveCountMax` を付けておくと、本当に切れたときに3分ほどで ssh が終わる。ターミナルが固まったまま待たされることはない。
- **3時間の上限は別の話**：Bastion のセッションの TTL（3時間）は keepalive では延びない。
- まず `ssh -o ServerAliveInterval=60 oci-mail` で、ファイルを変えずに試せる（試したら切れなくなった）。

### Q. ローカルの ssh 用のスクリプトを修正するのかと思っていたが、Terraform と見えた。サーバ側の設定を修正するのか

直すのはローカルの ssh の設定だけ。サーバ側（OCI の VM や Bastion）は何も変わらない。Terraform が出てきたのは、**手元の ssh 設定ファイルを Terraform が作っている**から。

- `compute.tf` の `local_file` リソースが、テンプレート `ssh_config.tftpl` から手元のファイル `oci-platform/ssh_config` を書き出している。`~/.ssh/config` はそれを Include している。
- テンプレートを使うのは、Bastion の OCID と VM のプライベート IP を Terraform の値から埋め込むため。
- `local_file` は WSL の中にファイルを書くだけで、OCI の API は呼ばない。apply しても、plan に出る変更はこの1件だけ（`local_file.ssh_config` の置き換え）。
- 生成された `ssh_config` を直接書き換えても動くが、次に apply したときテンプレートの内容で上書きされて消える。なので元のテンプレートを直す。

スクリプト（`bin/bastion-proxy`）に足すこともできるが、あれが起動するのは Bastion までの内側の ssh。ここに足すと Bastion との区間しか守れない。

### Q. その apply で Cloudflare の 403（Missing X-Auth-Email header）が出た。Terraform の中で Cloudflare の IP 帯を取得して LB の許可 IP に入れる設計があったと思うが、その IP 帯の取得でエラーになっているのでは

違う。エラーになっているのは、ゾーンを引く `data "cloudflare_zone" "main"` のほう。原因は、Cloudflare の API トークンが環境変数に入っていないこと。

| | IP 帯の取得 | ゾーンの取得（今回のエラー） |
| --- | --- | --- |
| 定義 | `nsg.tf` の `data "http" "cloudflare_ipv4"` | `cloudflare.tf` の `data "cloudflare_zone" "main"` |
| 取りに行く先 | `https://www.cloudflare.com/ips-v4`（公開ページ） | `api.cloudflare.com`（API） |
| 認証 | 要らない | API トークンが要る |

トークン（`CLOUDFLARE_API_TOKEN`）が見つからないと、Cloudflare プロバイダーは古い方式（メールアドレス＋グローバル API キー）で認証しようとする。そちらのヘッダーもないので、このメッセージになる。`terraform` を直接ではなく、ラッパーの `./tf.sh` から実行すれば、`.secrets` からトークンを読み込んで起動する。トークンをターミナルに残さないために、わざとこの形にしてある。

### Q. `terraform output mysql` や `terraform output -raw vault_id` は何をやっているのか。outputs.tf に出力された内容を出しているのか

そのとおり。`outputs.tf` に書いた `output` ブロックの値を表示している。

- 値は `apply` のときに計算され、**state に保存される**。`terraform output` はその state を読んで表示するだけで、OCI の API には問い合わせない（そのため一瞬で返る）。
- `terraform output mysql` は、オブジェクト（host、ip、port、version の組）を HCL の形で表示する。
- `-raw` は、文字列を引用符なしでそのまま出す指定。`VAULT=$(terraform output -raw vault_id)` のように、シェルの変数に入れるときに使う。文字列の output にしか使えない。
- 何も付けない `terraform output` は、全部の output を一覧にする。
- アプリ側の Terraform からは、`terraform_remote_state` で同じ output を読む。共通基盤からアプリへ値を渡す口が、この output。

### Q. Vault の思想について教えてほしい。vault、key、secret の3つを作っている、で合っているか。key と secret の違いが分かっていないが、どういう関係にあるのか

3つを作っている、で合っている。

```
Vault "platform"          the safe (container + endpoints, permissions are set here)
 ├─ Key "platform-secrets" the encryption machine inside the safe (never leaves it; you cannot read it)
 └─ Secret "mysql-admin-password"
       the document stored in the safe, encrypted with the Key
       versions: v1, v2, ... (read the content as a "secret bundle")
```

| | 何か | 中身を取り出せるか | 例えるなら |
| --- | --- | --- | --- |
| **Vault** | 鍵とシークレットを入れる入れ物。管理用の窓口（エンドポイント）を持つ | — | 金庫 |
| **Key**（マスター暗号鍵） | 暗号化と復号をする鍵。**Vault の外に出ない** | **取り出せない**。「これを暗号化して」「これを復号して」と頼めるだけ | 金庫に据え付けた暗号化の機械 |
| **Secret** | 預けたい秘密の値（パスワード、API トークンなど）。Key で暗号化して保管される | **取り出せる**（権限があれば）。版を重ねられる | 金庫に預けた書類 |

- **関係**：Secret を作るときは、どの Key で暗号化するかを指定する（`vault.tf` の `key_id`）。Secret を読むと、Vault が裏で Key を使って復号し、中身を返す。読む側は Key のことを意識しない。
- **分けている理由**：Key は「暗号化の道具」で、中身を誰にも見せないことが価値。Secret は「取り出して使う値」。役割が逆なので、権限も別になっている。今回 VM に与えた権限は `read secret-family` で、シークレットを読めるだけ。Key を直接使う権限は渡していない。
- **Key のほかの使い道**：ブロックボリュームや Object Storage を「自分で管理する鍵」で暗号化するときにも使う（既定は Oracle 管理の鍵）。今回は Secret の暗号化にだけ使っている。
- **AWS で言うと**：Key が KMS、Secret が Secrets Manager。OCI では、この2つが Vault という1つのサービスにまとまっている。

### Q. MySQL の作成で `Cannot specify backup policy with 'CreateDbSystem' operation for an Always Free DB system` になった。Always Free の場合、backup ポリシー自体が設定不可のようだ

そのとおり。Always Free の MySQL HeatWave（`MySQL.Free`）は、作成時にバックアップの設定（`backup_policy`）を指定できない。OCI が決めた既定の自動バックアップになる（作成のログでは「有効、保存1日」だった）。

- `backup_policy` を削除した。
- `ignore_changes` に `backup_policy` を足した。OCI 側が決めた値を、次の plan で差分として扱わないようにするため。

### Q. 今度は MySQL の作成が FAILED になった。コンソールのログは「Authorization failed or requested resource not found」（AuthorizationFailed）。何が原因として考えられるか

**原因**：HeatWave の作成の途中で、**DB システム自身**（自分ではなく、`authType: resource`）が `CreateInternalVnic`（private-db サブネットへの VNIC の作成）を実行し、404 で拒否されていた。

HeatWave は、サブネットへの VNIC の作成と NSG への参加を、**DB システム自身の権限（リソース・プリンシパル）**で行う。管理者である自分の権限は、ここでは使われない。そのため、「この DB システムは VNIC を作り、NSG に入ってよい」という、次の形のポリシーが別に要る（公式の Mandatory Policies and Permissions の「NSG と VNIC」の節）。今回は NSG を指定したので、これが必要になった。

```
Allow any-user to {VNIC_CREATE, ...} ... where request.principal.type='mysqldbsystem'
```

条件は「このコンパートメントの MySQL の DB システムだけ」に絞り、DB システムに `depends_on` を付けて、ポリシーができてから作るようにした。

**原因の探し方**：エラーのメッセージ（AuthorizationFailed）からは、どのリソースかまでは分からない。管理者のユーザーで操作している限り、普通は IAM のポリシーを足さなくても作れるので、推測で直すより中身を見るほうが早い。

1. コンソールの **Audit**（監査）で、失敗の時刻の前後を、ステータスが 401 / 403 / 404 のイベントで絞り込む。
2. 最初に見つけた 404（`ListWorkRequestLogs`）は、自分がコンソールで作業リクエストのログを開いたときの読み込みで、原因ではなかった。見るべきは、失敗より前の操作で、**自分の名前以外が主体になっているもの**（MySQL のサービスが何かを読もう・作ろうとして拒否されたイベント）。
3. 直近15分の Audit のログを出して探すと、`CreateInternalVnic` の 404 が見つかった。

Audit で見つからなければ、コンソールから同じ設定で作ってみる（コンソールでも失敗すればアカウント側、成功すれば Terraform の指定の問題）、という切り分けも用意していた。

### Q. ops に oci CLI を入れるタスクで `The pipx tool must be at least at version 1.7.0` になった。何を修正したのか詳しく教えてほしい。oci CLI を入れるのに pipx が必要だが、Ubuntu に入っている pipx のバージョンが古すぎた、という話か

惜しいが、少し違う。**pipx 自体は 1.4.3 でも oci CLI を入れられる。古すぎたのは、「Ansible の pipx 用モジュールから見て」**。

```
Ansible task (community.general.pipx module)
   └─ calls pipx with options that exist only in pipx >= 1.7   <- stopped here
        └─ pipx 1.4.3 (Ubuntu 24.04 apt) installs oci-cli        <- this part itself is fine
```

- 最初のコードは、`community.general.pipx` というモジュール（「pipx でこれを入れた状態にして」と宣言するもの）を使っていた。
- このモジュールは、内部で pipx の新しい機能（入っているものを JSON で一覧にする、など）を使って状態を調べる。そのため、**pipx 1.7.0 以上を前提**にしている。
- Ubuntu 24.04 の apt で入る pipx は 1.4.3 なので、モジュールが「この pipx には対応できない」と言って止まった。oci CLI のインストールまでは、たどり着いていない。

**どう直したか**

```yaml
- name: Install oci CLI for ubuntu
  ansible.builtin.command: pipx install oci-cli
  args:
    creates: /home/ubuntu/.local/bin/oci
  become: true
  become_user: ubuntu
```

- **モジュールを通さず**、`pipx install oci-cli` をコマンドとして直接実行する。pipx 1.4.3 で問題なく動くコマンド。
- `command` は、そのままだと毎回実行されてしまう。そこで **`creates`** を付けた。「このファイルがあれば、もう入っているので実行しない」という指定。何度流しても同じ状態になる、という Ansible の性質を、これで保っている。
- `become_user: ubuntu` は、oci CLI を root ではなく ubuntu ユーザーの持ち物として入れるため。
- **引き換えに失ったもの**：モジュールなら「最新版にする」「消す」も宣言で書けたが、今の形は「無ければ入れる」だけ。oci CLI を新しくしたいときは、ops で `pipx upgrade oci-cli` を手で流す。

**そもそも、なぜ pipx を使うのか**

- oci CLI は Python 製のツールだが、**Ubuntu の apt にはない**。
- Ubuntu 24.04 では、システムの Python に `pip install` で直接入れることが禁止されている（PEP 668。OS のパッケージを壊さないための仕組み）。
- **pipx は、Python 製のコマンドを1つずつ専用の環境（venv）に入れ、コマンドだけを `~/.local/bin` に置く道具**。OS を汚さずに入れられるので、Python 製の CLI を入れるときの定番。
- 手元の WSL では、Oracle の公式のインストーラーを使った。あれも中では専用の venv を作っていて、考え方は同じ。

### Q. なぜ nsg-lb を Cloudflare の IP 範囲に絞るのか

Cloudflare が担うのは DNS だけ、ではない。①②と同じく **プロキシ（オレンジの雲）を有効にする**前提。

- 名前解決で返るのは、LB の IP ではなく Cloudflare のエッジの IP。
- 利用者がつなぐ先は Cloudflare で、Cloudflare が改めて LB（オリジン）につなぎ直す。WAF やキャッシュは、この経路の途中で効く。
- そのため、LB にやって来るのは Cloudflare の IP だけ。

`nsg-lb` を Cloudflare の IP に絞るのは、**LB の公開 IP を直接たたいて、Cloudflare を迂回する通信を防ぐ**ため。もう一つ、Cloudflare と LB の間で使う Origin 証明書はブラウザに信頼されないので、プロキシを外すと HTTPS も通らない。どちらの意味でも、プロキシは必須。

DNS だけ（灰色の雲）にすると、利用者が LB に直接来る。その場合は、ブラウザに信頼される証明書（Let's Encrypt など）を LB に入れる必要があり、構成が変わる。

**理解の確認**：「証明書を Cloudflare 任せにするために、前段に必ず Cloudflare を通るようにしている」で合っている。

### Q. この場合、クライアント → Cloudflare → OCI LB の経路のプロトコルは何になるのか。Cloudflare で HTTPS が終端されるとすると、Cloudflare → OCI LB の間は HTTP になるのかと一瞬思ったが、そうではなく、server SSL 的な概念で暗号化されるのか。LB のポートに何を開けておけばよいか

**Cloudflare → LB の区間も HTTPS。**Cloudflare には、オリジン（ここでは OCI の LB）との間をどう暗号化するかの設定（SSL/TLS の暗号化モード）がある。

| モード | 利用者 → Cloudflare | Cloudflare → オリジン | 証明書の検証 |
| --- | --- | --- | --- |
| Flexible | HTTPS | **HTTP（80番）** | なし |
| Full | HTTPS | HTTPS（443番） | しない（自己署名でも通る） |
| **Full (strict)** | HTTPS | HTTPS（443番） | **する**（Origin 証明書か、公的な証明書が要る） |

使うのは **Full (strict)**。

```
Client --HTTPS--> Cloudflare edge --HTTPS (Origin CA cert, verified)--> OCI LB :443 --HTTP--> app :8000-8099
        (edge cert: Universal SSL)                                        (TLS ends here)  (inside VCN)
```

- **LB で開けるのは 443番だけ。**`nsg-lb` もそうなっている。80番（http から https への転送）は、Cloudflare の「Always Use HTTPS」がエッジで行うので、LB まで来ない。
- **TLS は LB で終わる。**LB からアプリまでは VCN の中なので HTTP。アプリは TLS を気にしなくて済む。
- **暗号化モードは、ゾーン（`alphajinsei.com`）全体の設定。**今が Flexible だと、Cloudflare が LB の 80番に HTTP でつなぎに行き、失敗する。Workers や Pages はエッジで動くので、このモードの影響を受けない。そのため、①②では気にする場面がなかった。

### Q. 今の設定は Full だった。Full でも Cloudflare と配信元の間を HTTPS 化してくれるので、十分な気がする。Full (strict) はなりすましとかを考えると良いのかもしれないが、違いがよく分かっていない。Full と Full (strict) について、暗号化の状態・証明書・残存リスク・初期工数・既存への影響を整理してほしい。そもそもなぜ Cloudflare にプロキシさせる案にしたのか。DNS の名前解決だけ任せて、向き先を OCI LB にする案もある（証明書は Let's Encrypt などで OCI 側に持つ）。正直、個人サイトなので http も止むなし、自己署名でも許容範囲かと思っている

**暗号化モードが決めるのは、Cloudflare → 配信元（OCI）の区間だけ。**クライアント → Cloudflare は、どのモードでも Cloudflare のエッジ証明書による HTTPS（Cloudflare が自動で発行・更新する）。

```
Client --(1) HTTPS: Cloudflare edge cert (auto)--> Cloudflare --(2) depends on mode--> OCI LB --(3) HTTP in VCN--> app
```

- **Full と Full (strict) の差**：どちらも (2) は暗号化される。違いは、**つないだ相手が本物かを確かめるか**だけ。Full は鍵のかかった電話をかけるが、相手の名前を確認しない。strict は相手の名前も確認する。
- **既存の Worker への影響**：Worker や Pages はエッジで動くので (2) の区間がない。モードを変えても影響しない。

| | A. Flexible | B. Full + 自己署名 | C. Full + Origin 証明書 | D. Full (strict) + Origin 証明書 | E. プロキシなし（DNS だけ）+ Let's Encrypt |
|---|---|---|---|---|---|
| クライアント → CF | HTTPS | HTTPS | HTTPS | HTTPS | （CF は通らない）クライアント → LB を直接 HTTPS |
| CF → OCI | **HTTP（平文）** | HTTPS・検証なし | HTTPS・検証なし | HTTPS・検証あり | — |
| 証明書 | エッジのみ | エッジ + 自己署名 | エッジ + Origin 証明書（15年） | 同左 | Let's Encrypt（90日。今後もっと短くなる予定） |
| 初期工数 | 小（LB に 80番リスナー） | 小（C とほぼ同じ） | 済み（当時のコードと設定） | 済み + スイッチ1つ | 中〜大 |
| 既存への影響 | ゾーン全体を変えるのは危険。ホスト名ごとのルールで限定する | なし | なし | DNS の「プロキシ済み」が Worker・Pages だけならなし | なし |

**残存リスク**

- **A（Flexible）**：インターネット上の Cloudflare → OCI の区間を平文で流れる。いずれ載せる認証基盤のパスワードや Cookie も平文になるので、ここが一番の問題。さらに「アプリは http で受けたと思って https に転送 → ループ」というよくある罠がある。
- **B・C（Full）**：経路の途中でなりすまされても気付かない。ただ、実際にできるのは Cloudflare と OCI の間の通信に割り込める攻撃者だけなので、個人サイトとしては低いリスク。**現実的にありうるのは設定ミスの方。**例えば、LB を作り直して古い公開 IP が他人に渡ったのに、DNS の更新が漏れていたとする。Full だと、Cloudflare はそこへ黙って利用者の通信を流す。strict ならエラー 526 で止まる。自己署名にしても工数は減らない（Terraform で作るのはどちらも同じ）ので、C を選んでおけば、D に上げるときに何も作り直さずに済む。
- **D（strict）**：上のリスクは消える。
- **A〜D に共通（プロキシ方式そのもの）**：Cloudflare は中身を平文で見る。また、NSG で Cloudflare の IP に絞っても、ほかの Cloudflare 利用者が自分のゾーンをこちらの IP に向ければ、Cloudflare 経由で届いてしまう。完全に防ぐなら Authenticated Origin Pulls（Cloudflare → オリジンの mTLS）を使う。
- **E（プロキシなし）**：LB を世界に公開し、IP も知られる。WAF・DDoS 対策・キャッシュがなくなり、10Mbps の LB に直接負荷がかかる。証明書の更新が止まると、サイトが落ちる。

**E の工数**：ブラウザが信頼する証明書が要るので、Origin 証明書は使えない。代わりに、ops に ACME クライアント（certbot や lego）を置き、DNS-01 用の Cloudflare トークンを VM に持たせる。定期的に証明書を取り直し、oci CLI で LB の証明書を差し替えて、リスナーを付け替える。NSG を 0.0.0.0/0 に開け、80番の転送用リスナーも作る。運用の勉強にはなるが、最初にやることではない。

**なぜプロキシにしたのか**：①②と入口をそろえる（`<名前>.alphajinsei.com` で、DNS の向け先を変えるだけで方式をまたいで移せる）。WAF・DDoS 対策・キャッシュが無料で付く。LB の IP を隠し、NSG で Cloudflare からだけに絞れる。証明書の更新を考えなくてよい（エッジは自動、Origin 証明書は15年）。

**ほかに確かめること**

- **ゾーン全体を触りたくなければ**、Configuration Rules でホスト名ごとに暗号化モードを上書きできる（一部のホスト名だけ strict にする）。
- **Always Use HTTPS**：Full でも strict でも、http で来たアクセスには、Cloudflare は利用者と同じプロトコルでオリジンの 80番へつなぎにいく。LB が 443番しか受けないなら、無効のままだと http のアクセスはエラー 522 になる。「Always Use HTTPS」を有効にして、エッジで https に転送させる（確認したら無効だったので、有効にした）。
- 「http もやむなし」とのことだったが、A は工数がほとんど減らない。それに、後で載せる認証基盤とは相性が悪いので、選ばない方がよい。

結論：C でも D でも手間はあまり変わらず、DNS の「プロキシ済み」のレコードが Pages と Workers だけなら影響もないので、D（Full (strict)）にした。

### Q. そもそも Origin 証明書とは何か。普通の商用証明書とは何か違うのか。最近は証明書の有効期限がどんどん短くなっているが、これは15年で作れてしまうのか。Cloudflare が無料で発行してくれるということでいいか。OCI にこうした証明書の発行機能はないのか

- **何か**：Cloudflare が自分の CA（Origin CA）で無料で発行する証明書。信頼するのは Cloudflare のエッジだけで、ブラウザや OS は信頼しない。Cloudflare 専用の通行証のようなもの。
- **15年で作れる理由**：有効期限が短くなっているのは、ブラウザが信頼する公的な証明書の業界ルール（CA/Browser Forum）。上限は今 200日で、2027年3月から 100日、2029年3月から 47日になる。Origin 証明書は公的な CA が出すものではないので、このルールの外にある。代わりに、ブラウザに直接見せると警告が出る。期限が長いぶん、鍵が漏れたら自分でダッシュボードから失効させる必要がある。
- **OCI の証明書**：OCI に、ブラウザが信頼する証明書を出す機能はない（AWS の ACM に当たるものがない）。OCI Certificates というサービスはあるが、できるのは、自分で作るプライベート CA での発行（ただし Full (strict) では受け付けられない）と、外で作った証明書の取り込みの2つ。

→ [方式検討：OCI](../design/oci.md)

### Q. 2か所の HTTPS（① クライアント → Cloudflare、② Cloudflare → OCI LB）があると思うが、それぞれ何の証明書を使うのか。どちらにも同じ Origin 証明書を設定するのか。①は気にしていなかっただけで Cloudflare がよしなに発行してくれていて、②を Origin 証明書として今回作る、という理解で合っているか。Origin 証明書という名前の由来は何か

その理解で合っている。ただし用語が1つずれていて、CDN の用語では、Cloudflare 側を「**エッジ**」（利用者に近い Cloudflare の拠点）、OCI LB 側を「**オリジン**」（コンテンツが生まれる大元。③では OCI の LB）と呼ぶ。HTTPS は2区間あり、証明書も別物。

| 区間 | 証明書を出す側 | 証明書 | 検証する側が持つルート |
|---|---|---|---|
| ① クライアント → Cloudflare | Cloudflare のエッジ | **エッジ証明書（Universal SSL）**。Cloudflare が公的な CA（Google Trust Services、Let's Encrypt など）から自動で取り、自動で更新する。`alphajinsei.com` と `*.alphajinsei.com` を含み、①②のサイトもすでにこれを使っている | ブラウザ・OS の信頼ストア |
| ② Cloudflare → OCI LB | OCI LB | **Origin 証明書**（今回作るもの） | Cloudflare の Origin CA のルート |

- 同じ証明書を両方に置くわけではない。①は Cloudflare がすでに持っていて、②を今回作る。現在の①は、ダッシュボードの SSL/TLS → エッジ証明書 で見られる。
- **名前の由来**：CDN の用語で、コンテンツが生まれる（originate）大元のサーバーを「オリジン・サーバー」と呼ぶ。エッジの反対側。「オリジン・サーバーに置くための証明書」なので Origin 証明書で、それを発行する CA が Origin CA。

**理解の確認**：「クライアント → Cloudflare（エッジ）→ OCI（オリジン）と呼ばれ、それでエッジ証明書とオリジン証明書と呼ばれる。エッジ証明書は Cloudflare が自動で取ってくれていて、一般的な認証局が署名した、ブラウザ同梱の上位証明書で検証可能な証明書が送られてくる。一方、オリジン証明書は Cloudflare から OCI への通信の暗号化に使われるもので、Cloudflare が独自に発行して、オリジン側（OCI 側）に設定するもの」で合っている。

### Q. OCI ではそもそも LB の public IP の予約ができるのか。予約するのにお金がかかったりしないか。作ってからその IP を DNS に設定する、でもいいかと思っている

- **予約はできる。**予約パブリック IP という仕組みがあり、LB には作るときだけ指定できる。
- **料金は確認できなかった。**公式の料金表では裏付けられず、「どこにも付けていない間は月 $7.30」という非公式の記載だけ見つかった。
- **作ってから DNS に設定する形で十分。**LB を作り直すと IP は変わるが、Full (strict) なので、DNS を直し忘れても、他人に渡った IP に黙ってつなぐことはない（エラーで止まる）。

### Q. アプリが増えるたびに DNS エントリを追加する手順を省くために、`*.alphajinsei.com` を全部 OCI LB へ流すワイルドカードの案について。Worker・Pages 向けの名前解決には影響しないか。Cloudflare でコンテナを起動したくなったときや、AWS で個人サービスをホストしたくなったとき、通信が OCI LB に飛んでしまわないか。`xxx.oci.alphajinsei.com` や `xxx-oci.alphajinsei.com` のように、正規表現に一致したものだけ OCI LB に飛ばすことはできるか（証明書が2階層まで効かない話や、DNS が正規表現を受け入れるのかなど、ハードルがあると思うが）

| 懸念 | 答え |
|---|---|
| Workers・Pages | 影響しない。個別のレコードがワイルドカードより優先される |
| Cloudflare でコンテナを動かす | Cloudflare のコンテナは Worker から呼ぶ形なので、Worker の独自ドメインのレコードが優先され、影響しない |
| AWS などに置く | AWS 向けのレコード（CNAME など）を個別に作れば、それが優先される |
| `*-oci` のような正規表現 | DNS にはない。ワイルドカードは「一番左のラベル全体が `*`」の形だけ |
| `*.oci.alphajinsei.com` | DNS としては書ける。ただし無料のエッジ証明書は1階層（`*.alphajinsei.com`）までなので、①で有料の Advanced Certificate Manager（月 $10）が要る |

つまり、技術的に壊れるものはない（DNS のワイルドカードは、個別のレコードがある名前には効かない）。それでも、懸念の本質は正しい。レコードを作り忘れたり消したりしても、エラーにならずに黙って OCI に流れるので、どこへ行くかが暗黙に決まってしまう。環境名をドメインに入れないという判断にも賛成。**ワイルドカードはやめた。**

### Q. アプリごとの DNS は、毎回 wrangler で個別に設定する、でどうか（手でやるのを wrangler で代替する意図）

**wrangler には、任意の DNS レコードを作る機能がない。**②でレコードができたのは、Worker の独自ドメインを登録した副作用（Worker に向いたレコードしか作れない）。OCI LB に向けた A レコードは、wrangler では作れない。

代わりの案は、アプリ側の Terraform にそのアプリのレコード1つだけを書く、小さなスクリプト（curl で Cloudflare API を叩く）を共通基盤に置く、手で作る、の3つ。

### Q. Terraform 以外で、CLI で Cloudflare の DNS を操作する手段はないのか。あるならそれにしたい

ある。2025年に出た Cloudflare 公式の汎用 CLI **`cf`**（npm の `cf`、beta）が使える。Wrangler が Workers のプロジェクト用なのに対し、`cf` は Cloudflare のリソース全般を操作する別の CLI で、DNS も扱える。

```bash
npm install --global cf          # Node 22.18 以上（手元は 24 なので OK）
cf dns records list   --zone alphajinsei.com
cf dns records create --zone alphajinsei.com \
  --body '{"type":"A","name":"aaa","content":"<LB の IP>","proxied":true,"comment":"oci: aaa"}'
cf dns records delete <レコード ID> --zone alphajinsei.com
```

- **まだ beta。**公式に「stable になるまでに、コマンドや設定が変わりうる」と書かれている。
- **認証は Wrangler とは別。**手で使うときは `cf auth login`。
- **宣言的ではない。**「作る」「消す」を命令するだけなので、同じ名前のレコードがすでにあるかは、先に `list` で見る。
- **中身は Cloudflare API そのもの。**`--body` の JSON は API とまったく同じなので、`cf` が使えなくなっても、同じ JSON を curl で API に送れば済む。

| 手段 | 状態 | 判断 |
|---|---|---|
| `cf`（公式） | beta | 「手の代わりに CLI で」という意図に一番合う |
| curl で Cloudflare API を直接呼ぶ | 安定 | `cf` が使えないときの逃げ道 |
| flarectl（cloudflare-go に付いていた CLI） | Go の SDK の新しい版から外れた | 使わない |
| サードパーティの CLI（cfdns など） | 非公式 | 使わない（主流の道具を選ぶ方針どおり） |
| Terraform（アプリ側） | 安定 | 宣言的にしたくなったら |
| Wrangler | DNS を作るコマンドがない | Worker の独自ドメインの副作用でしかレコードを作れない |

### Q. 仮に Terraform を採用したとして、これからアプリ開発を行う際に、アプリ開発者がほかに Terraform を触る箇所はあるか。ないなら cf、あるなら Terraform にしたい

ある。**OCI に載せるアプリは、どのみち自分のリポジトリで Terraform を触る。**なので Terraform にした。

| もの | 必ず要るか | 中身 |
|---|---|---|
| LB のバックエンド・セット | 必ず | app VM の自分のポート（8001〜）と、`/health` のヘルスチェック |
| LB のホスト名とリスナー（443番） | 必ず | `aaa.alphajinsei.com` を自分のバックエンド・セットへ振り分ける |
| Object Storage のバケット | 任意 | ファイルを置くアプリだけ |
| **Cloudflare の DNS レコード** | 必ず | ここに1つ足すだけ |

- 共通基盤の値（LB の ID・IP、VM の IP など）は、`terraform_remote_state` で共通基盤の出力から読む。
- LB の設定を書くついでに DNS レコードも同じファイルに書けば、`terraform apply` 1回でアプリの入口がそろう。片付けるときも `terraform destroy` でまとめて消える。②でアプリのデプロイ道具（wrangler）が自分のレコードを持っていたのと同じ考え方。

| 方式 | アプリの DNS を作る道具 |
|---|---|
| ①② Cloudflare | wrangler（独自ドメインの登録で自動で作られる） |
| ③ OCI | アプリのリポジトリの Terraform（自分のレコードだけを持つ） |
| ゾーン全体の設定、メールのレコード、そのほか既存のもの | 手 |

ユーザーの希望で、Cloudflare を触る部分は `cloudflare.tf` という別ファイルに分け、OCI ではない箇所を触っていることが見た目で分かるようにした。

### Q. app に既定のページ（8000番）を置く手順は必須か。不要なサービスを立てても仕方ない。ヘルスチェックの観点で必須、などがあるのか

厳密には必須ではない。OCI の LB で必須なのは「リスナーには既定のバックエンド・セットが1つ要る」ことだけで、バックエンド・セットは中身が空でも作れる。ただ、置いておくことをおすすめした。

| | 置く | 置かない（空のバックエンド・セット） |
|---|---|---|
| 未登録のホスト名へのアクセス | 「アプリがありません」のページを返す | 502 エラー |
| Step 5 での確認 | LB → Docker のコンテナまで通るかを、今確かめられる | 経路の確認はサンプルアプリまで持ち越す |
| 監視 | LB → app の経路の死活の見張り役になる | 空のバックエンド・セットのヘルスが異常に見えるので、アラームから外す工夫が要る |
| コスト | nginx:alpine で、メモリは 10MB ほど | なし |

### Q. 未登録のホスト名を宛先とした通信が OCI に届くことは、はたしてありうるのか。仮にあったとして、フォールバックする設定をどうやって入れるのか。もしかして、OCI LB の「バックアップ」の宛先として 8000番を登録する、ということか

指摘のとおり、**普段はほとんど起きない。**それでも届くのは、次のような場面。

| 場面 | 何が起きているか |
|---|---|
| DNS と LB の設定がずれたとき | アプリの片付けで LB のリスナーだけ先に消えた、作るときに DNS が先にできた、`hostname_names` の綴りを間違えた、など |
| ほかの Cloudflare 利用者が、自分のゾーンからこちらの LB の IP に向けたとき | NSG は Cloudflare の IP なので通ってしまう。Host ヘッダーが他人のドメインになる |
| 確認用の `oci-platform.alphajinsei.com` | アプリのリスナーがないので、既定に落ちる |

「未登録のホスト名の行き先」という説明は役割を大きく言いすぎていて、実際の価値は「経路の確認」と「基盤の死活の見張り役」の方が大きい。

**フォールバックの仕組みは「バックアップ」ではない。**バックエンドの「バックアップ」の設定（ほかのバックエンドが全部落ちたときだけ使う宛先）とは別物で、使っているのは**ホスト名を持たないリスナー**。

```
request to :443
  Host: aaa.alphajinsei.com -> listener hostname_names=[aaa...] -> backend set aaa (:8001)
  Host: bbb.alphajinsei.com -> listener hostname_names=[bbb...] -> backend set bbb (:8002)
  no match                  -> listener without hostname (https-default) -> default (:8000)
```

OCI LB は、Host ヘッダー（と SNI）でリスナーの仮想ホスト名を突き合わせ、どれにも合わなければホスト名のないリスナーが受ける。

### Q. サブドメインを見分けて、別ポートのコンテナに振り分ける処理は、どの部分が受け持つのか。サブドメインが何であれ同じ OCI LB にたどり着くので、LB 側で設定できるのか。コンソールを見たが、どこで設定できるのか分からない

振り分けは2段で、**サブドメインを見分けるのは LB**、ポートからコンテナへ渡すのは Docker。`Host: aaa.alphajinsei.com` のリクエストは、次の順に渡る。

1. **ホスト名**（仮想ホスト名）：OCI LB に `aaa.alphajinsei.com` という名前を登録する。
2. **リスナー**（443、`hostname_names` = 1 の名前）：Host が 1 と一致したら、このリスナーが受ける。
3. **バックエンド・セット**：2 の行き先。ヘルスチェック（`/health`）もここに付く。
4. **バックエンド**：`10.0.10.10:8001`（app VM の IP とポート）。
5. **app VM の Docker のポートの公開**：`compose.yaml` の `"8001:3000"` で、VM の 8001 番をコンテナの 3000 番へ渡す。

- 1〜4 はアプリごとに1組作る。アプリのリポジトリの Terraform が、`oci_load_balancer_hostname`、`oci_load_balancer_listener`（`hostname_names` 付き）、`oci_load_balancer_backend_set`、`oci_load_balancer_backend` の4つを足す。
- 5 はアプリの compose.yaml の役目。台帳で割り当てたポートを、VM の側のポートに使う。

**コンソールで見る場所**：ロード・バランサの詳細の左の「リソース」メニュー。

| メニュー | 対応するもの |
|---|---|
| **ホスト名** | 1。アプリを足すとここに並ぶ |
| **リスナー** | 2。詳細の「ホスト名」の欄が空のリスナー（`https-default`）が、既定のリスナーである印 |
| **バックエンド・セット** | 3。開くと、中のバックエンド（4）とヘルスの状態が見える |
| 証明書 | 443 で出す証明書 |
| ルール・セット、ルーティング・ポリシー | パスやヘッダーで細かく振り分ける別の仕組み。使わない |

1 のホスト名に名前を登録し、2 のリスナーでそれを選ぶ、という2段階なので、リスナーの画面だけを見ても分かりにくい。

**理解の確認**：「ホスト名、という設定がある」で理解できた。

### Q. イメージの運び方について。実際に開発するときは手元でビルドして確認する。確認できたイメージをあえて搬送せず、イメージの元ネタを運んで向こうで再度ビルドすることにする、ということでいいか。若干遠回りだが、環境制約で仕方ないという感じか

その理解で合っている。ただ「環境の制約で仕方なく遠回り」というより、**どの案でも、手元で確かめたイメージそのものは本番に運べない。**

- **運べない理由**：手元は x86、app は Arm（A1、arm64）で、CPU が違う。OCIR を使う案でも、手元で `docker buildx --platform linux/arm64` で改めてクロスビルドするので、確かめたものとは別のイメージになる。
- **同じものになる理由**：「同じソース・同じ `package-lock.json`・同じベースイメージ」から作るので、手元と本番で中身がそろう。
- **それなら、app の上でビルドするほうがよい**と判断した。エミュレーションがなくて速く、レジストリのトークンも要らない（ソースを rsync で送り、app で `docker compose build`）。

| 案 | 中身 | 良い点 | 気になる点 |
|---|---|---|---|
| A. OCIR | 手元で arm64 にクロスビルド → OCIR に push → app で pull | 標準的な形。イメージの履歴が残る | OCIR の認証トークンが手元と app の両方に要る。x86 からの arm64 のビルドはエミュレーションで遅い。容量の課金も要確認 |
| **B. app VM の上でビルド**（採用） | ソースを rsync で送り、app で `docker compose build` | トークンもレジストリも要らず、arm64 のビルドがそのまま速い。0円が確実 | ビルドに app の CPU（1 OCPU）を使う。戻すときは、VM に残した前のイメージを使う |

### Q. `npm run db:init` で `ERROR 1819 (HY000): Your password does not satisfy the current policy requirements` になった。PW が短いということはないと思うので、禁則文字が使われているとかか

禁則文字ではなく、**記号が1文字も入っていなかった**のが原因。

- HeatWave は、パスワードの規則（validate_password）が有効で、大文字・小文字・数字・記号をそれぞれ1文字以上含まないと `ERROR 1819` になる。
- アプリの Terraform の `random_password` には「各種類を最低1文字」の指定がなかったので、32文字が英数字だけになることがあった（共通基盤の管理者のパスワードには付けていたのに、アプリの側で漏らしていた）。
- `min_upper` / `min_lower` / `min_numeric` / `min_special = 1` を足した。

### Q. デプロイ後、なぜか疎通できない。コンテナも起動していないように見える（作ったはずなのに存在が見えず、落ちたわけでもない）。一瞬起動してすぐ止まるような設定になっていないか

**コンテナが起動してすぐ止まったのではなく、`up -d` まで処理がたどり着いていなかった。**出力が「migrations applied」で終わっていて、その後の `up -d` もヘルスチェックの出力も出ていないのが手がかり。

- deploy.sh は、VM で動かすスクリプトを `ssh oci-app bash -s < remote-deploy.sh` で、**標準入力として**流し込んでいた。
- 途中の `docker compose run`（マイグレーション）は、既定で標準入力をコンテナにつなぐ。
- そのため、スクリプトの残りの行（`up -d` 以降）をコンテナが読んで捨ててしまった。
- bash から見ると入力が途中で終わったので、エラーも出さずに終了した。手元で確かめたときは、端末から1行ずつ打ったので起きなかった。

**直し方**：VM で動かすスクリプトは rsync で一緒に送られているので、それを**ファイルとして実行**する。念のため、`docker compose run` には `-T`（標準入力をつながない）も付ける。標準入力を読むコマンド（`ssh` など）全般に同じことが起きるので、`-T` や `< /dev/null` で防ぐ。

502 になっていたのは、8001 番で何も動いていないので、LB がバックエンドを異常と見ていたから。

### Q. MySQL の中身を見たい。oci-ops に入って確認だと思うが、PW は結局どこに書いてあるのか。ローカルの terraform の output とかにあるか

ops に入って見る。パスワードは **Vault にあり**、ops の上で `platform-secret` を使うと取り出せる。

```bash
ssh oci-ops
# as the app user (sees only its own schema)
MYSQL_PWD=$(platform-secret sample-oci-vm-docker-db-password) \
  mysql -h mysql.db.platform.oraclevcn.com -u sample_oci_vm_docker sample_oci_vm_docker
# as admin (sees everything)
MYSQL_PWD=$(platform-secret mysql-admin-password) \
  mysql -h mysql.db.platform.oraclevcn.com -u admin
```

`MYSQL_PWD` は、パスワードをコマンドの引数に出さないための環境変数（`-p<パスワード>` で渡すと、`ps` や履歴に残る）。`__drizzle_migrations` が、流したマイグレーションの記録。

| 場所 | 中身 | 見てよいか |
|---|---|---|
| **Vault** | 正本。管理者のパスワードとアプリのパスワード | ◎ 普段はここから（`platform-secret`） |
| Terraform の state（Object Storage） | `random_password` の値 | 見られるが、見に行くものではない |
| app VM の `.env` とコンテナの設定 | アプリのパスワード | デプロイが書いたもの。調べもののときだけ |
| `terraform output` | **出していない**（出力は名前とシークレットの名前だけ） | — |

手元の PC にはパスワードを置いていないので、ops（または app）に入って Vault から取り出すのが正規の道。

### Q. 手元の MySQL コンテナで、アプリが早く起きてテーブルの作成とぶつかった。なぜか

MySQL の公式イメージのヘルスチェックを `mysqladmin ping -h localhost` にすると、初期化中の仮のサーバー（ネットワークを閉じて、ソケットだけで動く）でも通ってしまい、アプリが早く起きてテーブルの作成とぶつかる。`-h 127.0.0.1`（TCP）で聞く。

（会話では「MySQL の初期化が終わる前にアプリがつなぎに行ってしまう問題が出たので、ヘルスチェックを TCP で聞く形に直し、ゼロから起動し直して問題が出ないことを確かめた」とだけ報告している。）

### Q. MySQL で、挿入した行の ID はどう受け取るのか

MySQL には `RETURNING` がない。Drizzle では、挿入した行の ID を `$returningId()` で受け取る。

### Q. 手元だけの MySQL や接続情報は、どこに書くのか

`docker compose` は `compose.override.yaml` を自動で重ねる。手元だけの MySQL や接続情報をそこに書き、本番には送らない（rsync の対象から外す）。

### Q. アラームは何を見ればよいか。CPU の高騰とかを受けてもできることがない（拡張余力がない）ので、Docker の停止と VM の停止だけで十分ではないか

**スコープを絞る方針に賛成。手を打てない警報は、鳴っても困るだけ。**監視は「打つ手があるもの」に絞り、止まったこと（アプリと VM）だけを通知する。

- **コンテナ（アプリ）の停止**：LB の `unhealthyBackendServers > 0`。
- **VM の停止**：`CpuUtilization` が届かなくなったら発報（`absent()`）。
- 外したもの：CPU、メモリ、DB の容量、DB のバックアップの失敗。`/health` は DB を見ないので、HeatWave が落ちても通知は来ない。HeatWave は OCI のマネージドなので任せる、という割り切り。

**アラームの判定の単位**：OCI のアラームは、メトリクスの系列（ディメンションの組み合わせ）ごとに判定される。そのため「全部の VM」や「全部のバックエンド・セット」に対するアラームを1つ書けば、あとから足したアプリにも自動で効く。アプリの Terraform には何も足さない。実際のクエリは次のとおり。

- `platform-backend-down`：`unhealthyBackendServers[1m].groupBy(backendSetName).max() > 0` が3分続いたら発報。`is_notifications_per_metric_dimension_enabled` で、メールはバックエンド・セット（アプリ）ごとに別々に届く。
- `platform-vm-down`：`CpuUtilization[1m].groupBy(resourceDisplayName).absent(10m)`。VM ごとに判定する。

**`absent()` が要る理由**：閾値のアラーム（CPU > 80 など）は、VM が止まるとデータが来なくなり、何も言わなくなる。止まったことは、一定時間メトリクスが届かないと発報する `absent()` のアラームで知る。VM の CPU やメモリのメトリクスは、Ubuntu のイメージにも入っている Oracle Cloud Agent が送る（メトリクス・エクスプローラで、届いていることを確かめた）。

**誤報について**：「構築中に誤作動的に鳴るのでは」という心配はそのとおり。デプロイの入れ替え（数秒）は3分の待ちで吸収できる。ただ、新しいアプリを apply してから最初にデプロイするまでの間は鳴る。

### Q. ログは、普通に oci-app に ssh で入って docker logs で見るということか。`docker compose logs` というコマンドを初めて聞いたが、`docker logs` と何が違うのか

その理解で合っている。oci-app に ssh で入って見る。違うのは、何を指定するかだけ。

| | 指定するもの | 例 |
|---|---|---|
| `docker logs` | コンテナの名前（1つ） | `docker logs -f sample-oci-vm-docker-app-1` |
| `docker compose logs` | `compose.yaml` のサービス名（省略すると全部） | `cd /opt/apps/sample-oci-vm-docker && docker compose logs -f app` |

中身は同じログ。compose で作ったコンテナは `<プロジェクト名>-<サービス名>-1` という名前になる。`docker compose logs` は `compose.yaml` のあるディレクトリでその名前を自分で解決してくれ、複数のサービスのログを、行頭にサービス名を付けて混ぜて出す。アプリのディレクトリにいるなら compose の方が楽で、どこからでも打てるのは `docker logs`。

Docker のログの形式は `local` にしてあり、ローテーションも既定で効いている（1コンテナあたり 20MB×5ファイル）。コンテナのログは OCI Logging には送らない（送るには VM にエージェントの設定が要るので、必要になったら足す）。

### Q. LB のアクセスログとエラーログは、普通にコンソールから LB 作成時に同時に作成されるログ群で、特別に転送設定を入れるのではなく、デフォルトでそうだという理解で合っているか。Terraform から作成だと、明示的に作成の指示が必要ということか

半分合っている。**ログは既定では有効にならない。**どちらの場合も、Logging サービスに「ログ」（とその入れ物の「ロググループ」）というリソースを作って初めて記録が始まる。

- **コンソール**：LB の作成ウィザードの最後に「ロギングの管理」がある。そこでエラーログ（とアクセスログ）を有効にすると、ウィザードが裏でロググループとログを作る。作成時に同時にできたように見えるのは、このため。
- **Terraform**：ウィザードがないので、`oci_logging_log_group` と `oci_logging_log`（ソースは LB、種類は `access` か `error`）を自分で書く。LB のリソースに「ログを出す」という設定の項目はない。

「特別な転送の設定」ではなく、「ログのリソースを作ると、LB のログがそこに入る」という仕組み（ソースが LB のサービス・ログ）。エージェントも要らない。全アプリへのリクエストの記録（ステータスコード、応答時間、どのホスト名か）を1か所で見られるので、障害のときに docker logs より先に見る場所になる。Logging の無料枠は月10GB。

### Q. メール基盤は、今の共通構成の上のアプリとして実現できるか。あるいは別に VM として作る方が自然か

**「送るだけ」なら乗る。「メールサーバーを自分で持つ」なら別扱い。**

| やりたいこと | どうするか | 今の構成に乗るか |
| --- | --- | --- |
| アプリから送る（登録の確認、パスワードの再設定など） | OCI Email Delivery に、SMTP の 587 番か HTTPS の API で送る。SPF・DKIM のレコードは Cloudflare の DNS に置く | ◎ 乗る |
| 受け取る | Cloudflare Email Routing（無料）で転送するか、Worker で処理する | ◎ OCI 側は何もしない |
| メールサーバー（MTA）を自分で動かす | 25 番の受信と送信が要る | △ 別扱い |

自分のメールサーバーを別扱いにする理由：送信の 25 番は OCI が初期設定で塞いでいる（解除には申請が要る）。受信には外から 25 番に届く道が要る（Flexible LB は HTTP 用）。クラウドの IP はスパム判定されやすい。メールボックスはディスク上の状態で、ステートレスの前提に合わない。→ メールサーバーは別の VM（mail）に建てることにした。

**メールサーバーの肝：送信は Email Delivery に中継させる**

| 方向 | どうするか | 理由 |
| --- | --- | --- |
| 受信 | 外から mail の **25 番に直接**届く。MX レコードは mail の公開 IP に向ける | **OCI が塞いでいるのは、外へ出す 25 番だけ** |
| 送信 | 自前の Postfix から、**OCI Email Delivery に 587 番で中継**する | 外への 25 番の解除の申請も、クラウドの IP の評判の問題も避けられる |
| 認証の DNS | SPF・DKIM・DMARC のレコードを Cloudflare に置く | |

受信・メールボックス・IMAP は自分で持ち、配達の部分だけをマネージドに任せる形。

### Q. ops から mail へ swaks を流したら `No route to host` になった。ops サーバから mail サーバへ 25 の穴をまだあけていないからではないか

原因は NSG ではなく、**mail VM の中の iptables**。`No route to host` は、NSG で落とされたとき（応答がなくタイムアウトする）とは症状が違い、相手の iptables が REJECT（icmp-host-prohibited）したときに出る形。

mail VM の iptables を見ると、ルールが2組並んでいた。

```
-A INPUT ... --dport 22 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited   <- old set; everything is rejected here
-A INPUT ... --dport 22 -j ACCEPT
-A INPUT ... --dport 993/587/465/25 -j ACCEPT            <- new set; never reached
-A INPUT -j REJECT ...
```

- ファイル（`/etc/iptables/rules.v4`）は正しく書けていた。
- OCI の Ubuntu のイメージは `/etc/default/netfilter-persistent` に `IPTABLES_RESTORE_NOFLUSH=yes` を入れている。そのため `netfilter-persistent reload` は、今のルールを消さずに**後ろに足す**。
- 結果として、古い REJECT が先に効き、足した ACCEPT が効いていなかった。Step 3 で開けたつもりだった mail の4つのポートは、実はずっと閉じていたことになる。

**直したもの**：Ansible の common のハンドラーを、フラッシュしてから読み直す形（`iptables-restore < /etc/iptables/rules.v4`）に替えた。Docker が足したルールも消えるので、Docker を再起動するところは今までどおり。今の mail VM は、ファイルが変わらないのでハンドラーが動かないので、一度だけ手で読み直す（または再起動）。

```bash
ssh oci-mail "sudo sh -c 'iptables-restore < /etc/iptables/rules.v4' && sudo iptables -S INPUT"
```

`sudo sh -c` にしているのは、`<` のリダイレクトを root で開くため（`rules.v4` は root しか読めない）。`iptables -S` で、実際に効いている順番を見る。REJECT が最後に1つだけになっていれば成功。

### Q. メールについて分かったことは、どこにまとめたか

メールの話は大きいので、ユーザーの希望で番外編に分けて、そのつど詳しく書いている。

- メールの基礎（登場人物、封筒と便箋、ポート、SPF・DKIM・DMARC の全体像とアライメント）→ [番外編：メールの仕組み](extra-mail-basics.md)
- メールサーバーを組みながら分かったこと（Postfix の役割と設定、届いたメールのヘッダー、証明書の役割、ACME と自動更新、Dovecot との分担）→ [番外編：メールサーバーを組む](extra-mail-build.md)
- メールの流れを図にして分かったこと（Postfix はルーターに近い、ポートと暗号化は別の軸、認証の3種類、経路ごとのプロトコル・ポート・認証）→ [番外編：メールの流れを図にして分かったこと](extra-mail-diagram.md)

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

### Q. 図が崩れてしまっている。日本語が入っているせいか。もし日本語が原因だとすると、図の中だけ英語にしてしまおう

崩れた原因は、推察のとおり日本語。日本語は1文字が英数字の約2文字分の幅で表示されるが、その比率が環境によってぴったりではないため、罫線がずれる。

そのため、**コードブロック内の図はすべて英数字だけ**にする。説明は図の外に日本語で書く。罫線もそろうように、比較図のような込み入った図はスクリプトで生成する。
