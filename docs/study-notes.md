# 勉強メモ

最終更新：2026-11-02

[学習ロードマップ](learning-roadmap.md) を進める中で分かったことを、短く積み上げていくメモ。長くなった話は番外編（`extra-*.md`）に分けて、ここからリンクする。

## 開発環境（WSL）

- WSL は既定で、Windows の PATH を Linux の PATH の後ろに足す。WSL に入れていないコマンドも Windows 側のものが動いてしまう。apt の nodejs には npm が含まれず、npm だけ Windows 側が使われていた。対策は、WSL の中に nvm で Node.js を入れること。→ [番外編：Node.js まわりの用語](extra-nodejs-basics.md)
- nvm と fnm は同じ役割。情報が多い（AI に聞いても当たりやすい）nvm を選んだ。
- apt で外部のソフトを入れるときは、「配布元の署名鍵（公開鍵）を置く → 配布元を `sources.list.d` に登録し、`signed-by=` で鍵と結び付ける → apt で入れる」の流れになる。向きは SSH の known_hosts に近い。→ [番外編：apt の署名鍵と GPG](extra-apt-signing.md)

## git

- `.gitattributes` の `* text=auto eol=lf` は、テキストファイルの改行を LF にそろえるリポジトリの設定。`core.autocrlf` は自分のパソコンの設定なので、リポジトリに書いておくほうが誰が clone しても確実に効く。CRLF が混ざると、シェルスクリプトが `bad interpreter: /bin/bash^M` で動かなくなる。
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

## ドメインと DNS

- Custom Domain（`routes` に `custom_domain: true`）を設定すると、DNS レコード・証明書・Worker への振り分けが、`wrangler deploy` で一度にできる。DNS の画面には、種類が「Worker」の Cloudflare 独自のレコードが出る。
- Worker 1つ1つに IP はない。名前解決で返るのは Cloudflare 共通のエッジの IP（Anycast）で、どの Worker を動かすかは、エッジでホスト名を見て決める。
- Custom Domain はホスト名ごと Worker に渡す（Worker がオリジンになる）。Route はホスト名 + パスのパターンで既存のホスト名の手前に割り込む。パスベースの振り分けも可能。
- 無料の証明書（Universal SSL）が自動で守るのは `*.alphajinsei.com` の1段まで。サブドメインは1段にする。
- `workers_dev` を書かないと、デプロイのたびに既定値（有効）で上書きされ、workers.dev の URL が開いたままになる。設定ファイルに明示して、ファイルを正にする。Access を付けるときに迂回路にならないよう、閉じておく。

## ステートレス

- Step 3 で、メモをメモリ上の配列に持つと、本番ではたまにしか取れなかった。配列は1つの isolate のメモリにしかなく、同じ「データセンター × サーバー × まだ生きている isolate」に当たったときだけ見える。
- Worker はステートレスに作るのが原則で、状態は外の保存先に置く。LB の後ろに並べたアプリサーバーがセッションをメモリに持てないのと同じ（12-Factor App の「プロセスはステートレス」）。
- グローバル変数は、消えても作り直せるもの（キャッシュ）ならよい。消えたら困るデータには使わない。
- 状態の置き場所：D1（表形式のデータ）／KV（設定・キャッシュ。書き込みが行き渡るまで少し時間がかかる）／R2（ファイル）／Durable Objects（状態を持つ処理が必要なときの例外的な仕組み）。

## D1

- バインディング（`wrangler.jsonc` の `d1_databases`）で Worker に DB を渡す。コードからは `c.env.DB`。接続文字列もパスワードも書かない。`database_id` は識別子なので git にコミットしてよい。
- `wrangler dev` は既定で、ローカルの SQLite ファイル（`.wrangler/state/v3/d1/` の下）を使う。本番の DB を壊さずに開発できる。
- マイグレーションは、DB の構造の変更を番号付きの SQL ファイルとして積み上げる管理方法。`migrations create` は手元にファイルを作るだけ。`apply --local` は手元のファイル、`apply --remote` で初めて本番に適用する。適用済みかどうかは、DB の中の `d1_migrations` テーブルに記録される。
- SQL は `prepare("... ?").bind(値)` で組み立てる（文字列をつなげないので SQL インジェクションを防げる）。`.all()` は全行、`.first()` は1行、`.run()` は結果の行が不要なとき。
- SQLite はサーバーではなく、ファイルを直接開いて読み書きするライブラリ。起動・停止の概念がない。`d1 execute --local` は dev を止めていても動く。本番の D1 も、使う側に稼働状態は見えない。料金は読み書きした行数と保存量で決まる。
- 表領域・スキーマ・ユーザー・GRANT はない。1つの DB ＝ 1つのファイル。誰がアクセスできるかは、どの Worker にバインディングを付けるかで決まる。大きな DB を区切るのではなく、用途ごとに小さな DB を作る。
- ロックとトランザクション：
  - SQLite のロックは、ロックファイルを置くのではなく、OS のファイルロックを使う。WAL モードでは `-wal`（書き込み途中のデータ）と `-shm`（読み手と書き手の調整用の共有メモリ）を併用する。読み手は同時に何人でもよいが、書き手は DB 全体で同時に1つだけ（行ロックはない）。
  - 本番の D1 は、1つの DB を1か所の1つのインスタンスが受け持ち、書き込みをそこで順番に処理する。そのため、利用者がロックの競合を意識する場面はほとんどない。
  - D1 では、`BEGIN` 〜 `COMMIT` を何往復かに分けて使うトランザクションは使えない。複数の SQL をまとめて「全部成功か全部失敗」にしたいときは、`c.env.DB.batch([...])` で一度に送る。

## ドキュメントの書き方

- コードブロック内の図に日本語を入れると、全角の幅が環境によって英数字2文字分ぴったりにならず、罫線が崩れる。図の中は英数字だけにする。
