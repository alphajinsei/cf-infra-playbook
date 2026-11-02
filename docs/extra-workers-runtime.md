# 番外編：Workers の実行環境（V8 isolate とリクエストの流れ）

最終更新：2026-11-02

[学習ロードマップ](learning-roadmap.md) の Step 1〜2 で出てきた、「Worker はどこでどう動いているのか」の整理。

## 全体像

Worker のコードは、`wrangler deploy` の時点で世界中のデータセンターに配られる。サーバーを起動して待ち受けるのではなく、リクエストが届いたデータセンターで、Cloudflare がホスト名から呼ぶべき Worker を判断し、その場で実行する。

- 事前に行うこと：コードと「ホスト名 → Worker」の対応を全世界に配る（デプロイ）
- リクエストのたびに行うこと：着いたサーバーで isolate を用意し、`fetch()` を呼ぶ（実行）

## V8 とは

V8 は、Google が Chrome のために作った JavaScript エンジン。JavaScript のコードを読み、機械語に変換して実行する部品で、ブラウザから切り離して別のプログラムに組み込める。

| 製品 | 中身 |
| --- | --- |
| Chrome | V8 ＋ 画面描画などのブラウザの機能 |
| Node.js | V8 ＋ ファイルやネットワークを扱う機能（`fs` など）。サーバー側で JavaScript を動かす |
| workerd（Cloudflare Workers） | V8 ＋ Web 標準の機能（`fetch`、`Request`、`Response`）＋ D1 などへの接続 |

Node.js と workerd は同じエンジンを積んだ兄弟で、Workers は Node.js の上で動いているわけではない。`wrangler dev` でローカルに起動しているのも workerd（Node.js が動かしているのは wrangler という CLI だけ）。

## V8 isolate とは：VM・コンテナとの比較

isolate は V8 自身が持っている仕組みで、1つのプロセスの中に、互いのメモリが見えない JavaScript の実行区画を複数作る。区画ごとにメモリ（ヒープ）と変数を持ち、ほかの区画のデータには触れない。

仮想化の観点では、どの層で区切るかが違う。

```
VM                     Container              V8 isolate
                       +-------+ +-------+
                       | App   | | App   |
                       | libs  | | libs  |
                       |       | |       |
                       +-------+ +-------+    +---------------------------+
+-------+ +-------+    +-----------------+    | workerd (one process)     |
| App   | | App   |    | Container       |    | +----+ +----+ +----+      |
| Guest | | Guest |    | runtime         |    | | W1 | | W2 | | W3 | ...  |
| OS    | | OS    |    +-----------------+    | +----+ +----+ +----+      |
+-------+ +-------+    +-----------------+    +---------------------------+
+-----------------+    | Host kernel     |    +---------------------------+
| Hypervisor      |    | (shared)        |    | Host OS (Linux)           |
+-----------------+    +-----------------+    +---------------------------+
+-----------------+    +-----------------+    +---------------------------+
| Hardware        |    | Hardware        |    | Hardware                  |
+-----------------+    +-----------------+    +---------------------------+
```

| | VM | コンテナ | V8 isolate |
| --- | --- | --- | --- |
| 何を仮想化するか | ハードウェア | OS（カーネルは共有） | 言語の実行環境（プロセスも共有） |
| 区切る仕組み | CPU の仮想化機能 | カーネルの namespace / cgroup | V8 のメモリ管理（JavaScript はメモリのアドレスを直接触れない） |
| 動かす土台 | ハイパーバイザ | コンテナランタイム（Docker、containerd） | workerd（V8 を組み込んだ Cloudflare の実行環境。オープンソース） |
| 起動時間 | 数十秒〜 | 数百 ms〜 | 数 ms |
| 1つあたりのメモリ | GB 単位 | 数十 MB〜 | 数 MB |
| 動かせるもの | 何でも | Linux で動くもの何でも | JavaScript と WebAssembly だけ |

ハイパーバイザやコンテナランタイムに当たる役割を果たすのが workerd というプロセス。1つの workerd の中に、世界中のユーザーの Worker が何千も同居している。

言語のレベルの区切りだけに頼らず、守りを重ねている。

- isolate の中では精密なタイマーやスレッドを使えなくして、CPU の脆弱性（Spectre）を突く攻撃をしにくくする
- workerd のプロセス自体も、OS のサンドボックスで閉じ込める
- 怪しい動きをする Worker は、別のプロセスに隔離する

そのぶん VM やコンテナより圧倒的に軽く、速く起動できる。デプロイ時に出る `Worker Startup Time: 1 ms` は isolate の起動時間。「JavaScript しか動かない」という Workers の制約もここから来ている。

## リクエストの流れ

リクエストが着いたサーバーそのものが、isolate を動かすサーバー。エッジの奥に「Worker 専用のサーバー群」が別にあるわけではない。

```
Browser
  | (1) DNS: sample-cf-workers-d1.alphajinsei.com -> Cloudflare anycast IP
  | (2) The same IP is announced worldwide -> reaches the nearest data center
  v
Cloudflare data center (e.g. Tokyo)
  | (3) Load balancer picks one server
  v
Edge server (every server runs the same full software stack)
  | (4) TLS termination (certificate chosen by hostname / SNI)
  | (5) Look up hostname -> Worker "sample-cf-workers-d1"
  |     (this mapping is what the DNS page shows as a "Worker" record)
  | (6) workerd on the same server prepares the isolate and calls fetch()
  v
Response goes back the same way
```

1. 名前解決すると、Cloudflare のエッジの IP アドレスが返る。オレンジの雲（プロキシ有効）のレコードと同じ IP で、Cloudflare を通るサイトのほぼすべてがこの IP を共有している。
2. 同じ IP を世界中のデータセンターが一斉に名乗っている（Anycast）ので、ネットワーク的に一番近いデータセンターに届く。日本からなら東京か大阪。
3. データセンターの中で、負荷分散装置が1台のサーバーに振る。どのサーバーにも同じソフト一式が入っている。
4. ホスト名に合う証明書で HTTPS をほどく。
5. ホスト名で設定を引き、どの Worker を動かすかを決める。
6. 同じサーバーの workerd が isolate を用意して `fetch()` を呼ぶ。

ポイント：

- 振り分けは IP ではなく、エッジでホスト名を見て行う。IP は全員共通の入口にすぎず、Worker 1つ1つに IP はない。
- デプロイすると、コードと「ホスト名 → Worker」の対応は数秒で世界中のデータセンターに行き渡る。なので、どこに着いてもその場で処理できる。
- Cloudflare は、TLS の最初のあいさつでホスト名が分かった時点で isolate の準備を始める。そのため、コールドスタートがほぼ見えない。
- 例外：D1 のデータの本体は特定の場所に置かれる。Worker は東京で動いても、DB への問い合わせはデータのある場所まで飛ぶ（Step 4）。

## DNS の画面の「Worker」レコード

Custom Domain を設定すると、DNS の画面に種類が「Worker」のレコードが現れる。Cloudflare 独自の表示で、普通の DNS にあるレコードの種類（A、CNAME など）ではない。

- 実体は、上の (5) の「ホスト名 → Worker」の対応を DNS の画面に表示したもの。外から名前解決すると、Cloudflare のエッジの IP が返る。
- Custom Domain が管理しているので、手では編集できない。`routes` から消してデプロイすると、レコードも消える。
- DNS（名前解決）とエッジ（実行）を同じ Cloudflare が持っているので、Custom Domain の設定ひとつで両方が済む。DNS レコードを自分で作らなくても名前解決できるのはこのため。

## Custom Domain と Route

| | Custom Domain | Route |
| --- | --- | --- |
| 単位 | ホスト名ごと Worker に渡す。Worker がそのホスト名の終点（オリジン）になる | ホスト名 + パスのパターン（例：`alphajinsei.com/memo/*`）に合ったリクエストだけ Worker に渡す。既存のホスト名の手前に割り込む |
| DNS と証明書 | 自動で作られる | そのホスト名の DNS レコード（プロキシ有効）が別に必要 |
| 向いている使い方 | 1サービス＝1サブドメイン（このプロジェクトの方針） | 既存のサイトの一部のパスだけ Worker で処理する |

Route の場合、Worker には `/memo/...` のように先頭のパスが付いたまま届くので、アプリ側でその前提にする（Hono の `basePath` など）。
