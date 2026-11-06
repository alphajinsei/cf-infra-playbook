# infra

個人サービスを公開するためのインフラの方針と、その学習の記録を置くリポジトリ。アプリのコードはここには置かない。

## やろうとしていること

- 個人サービスは Cloudflare（Workers + D1 ほか）を標準基盤にし、1サービス＝1ディレクトリでアプリからインフラまで完結させる。方針と理由は [docs/design/](docs/design/)（[全体](docs/design/overview.md)、方式ごとに [Cloudflare](docs/design/cloudflare.md)・[OCI](docs/design/oci.md)、OCI のメール基盤は別冊 [oci-mail](docs/design/oci-mail.md)）。
- まず標準構成のサンプル（メモ帳アプリ）を、ユーザー自身の手で作ってデプロイし、構成管理まで一通り経験する。手順と進捗は [docs/roadmaps/cloudflare.md](docs/roadmaps/cloudflare.md)（完了）。
- 次に方式③として、OCI Always Free で共通基盤（LB・Docker ホスト・MySQL HeatWave・メール用 VM など）を作り、その上にサンプルを載せる。手順と進捗は [docs/roadmaps/oci.md](docs/roadmaps/oci.md)。OCI は業務で触っているので粒度は粗め。コマンドはユーザーが打ち、コード生成は Claude が行う。
- 学習が終わったら、以降のサービスのデプロイは Claude に任せる。テンプレートは別に作らず、サンプル（GitHub の Template repository）をそのまま使う。
- サービスを作るときに読む資料は3段：[docs/service-guide.md](docs/service-guide.md)（全方式共通）、基盤ごとの資料（Cloudflare は [docs/platforms/cloudflare.md](docs/platforms/cloudflare.md)、OCI は `oci-platform` の README）、各サンプルの `README.md`（方式ごと。見出しは6つにそろえる）。新しい方式のサンプルを作ったら、ガイドの「方式の一覧」に1行足す。
- 会話の中で分かったことは [docs/notes/study-notes.md](docs/notes/study-notes.md)（勉強メモ）に短く追記していく。長くなる解説は `docs/notes/extra-*.md`（番外編）に分け、勉強メモからリンクする。

## ディレクトリ構成

```
~/claude-sandbox/
├── infra/                  このリポジトリ（方針・ロードマップ・番外編）
│   ├── docs/
│   │   ├── service-guide.md  サービスを作るときに読む（全方式共通）
│   │   ├── platforms/        基盤ごとの資料（cloudflare。OCI は oci-platform の README）
│   │   ├── design/           方式の検討（overview / cloudflare / oci / oci-mail）
│   │   ├── roadmaps/         学習ロードマップ（cloudflare / oci）
│   │   └── notes/            勉強メモと番外編（extra-*.md）
│   └── .secrets/           API トークンなどの置き場。git 管理外（.gitignore）
└── sample-<基盤>-<構成>/   方式ごとのサンプルアプリ。それぞれ独立した git リポジトリ
    例：sample-cf-workers-d1（方式②：アプリ + D1。ロードマップのサンプル）
        sample-cf-workers-static（方式①：Astro の静的サイト）
└── oci-platform/           方式③の共通基盤（Terraform + Ansible）。サンプルは sample-oci-vm-docker
```

- infra は GitHub では `alphajinsei/cf-infra-playbook`（公開）。手元のフォルダ名は infra のまま。公開リポジトリなので、秘密情報や個人のメールアドレスを書かない。コミットの作者は noreply アドレス（リポジトリ単位で `git config user.email` に設定済み）。
- サンプルは、フォルダ名＝GitHub のリポジトリ名＝Worker の名前。小文字とハイフンのみ。
- 作業の拠点は infra。サンプルのフォルダは `.claude/settings.json` の `additionalDirectories` に登録して、このセッションから読み書きする。サンプルを増やしたらここにも追記する。
- 開発環境は WSL2（Ubuntu）。Node.js は nvm で入れた LTS（24）。

## 進め方のルール

- ロードマップの作業は学習が目的なので、ユーザーが自分で実行する。Claude はコマンドと意味を説明し、実行は頼まれたときだけ行う。
- 新しく説明したことや、ユーザーが理解したことは、そのつど勉強メモの該当する節に追記する（ユーザーの依頼で始めたので、追記のたびに確認は取らなくてよい）。
- ユーザーに実行してもらうコマンドがロードマップにまだ書かれていなければ、該当する Step に追記する。終わった項目はチェックを付ける。
- ファイルの作成・変更・コピーなど、手元の状態が変わる作業は、実行する前に確認を取る。
- `.secrets/` の中身は読まない・表示しない（`.claude/settings.json` で Read を禁止している。Bash の cat なども使わない）。トークンが必要な作業は、ユーザーに `source` してもらうか、ユーザー自身に実行してもらう。
- Claude が使う Bash は非対話シェルで nvm が読み込まれず、`npx` や `npm` が Windows 側（`/mnt/c/...`）に解決される。Node.js のコマンドを Claude が実行するときは `bash -ic '...'` のように対話シェルで実行する。
