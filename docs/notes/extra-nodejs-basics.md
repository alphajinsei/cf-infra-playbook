# 番外編：Node.js まわりの用語

最終更新：2026-11-02

[学習ロードマップ](../roadmaps/cloudflare.md) の Step 0-2 で出てくる用語の整理。

## 4つの道具の関係

| 名前 | 何者か | たとえ |
| --- | --- | --- |
| Node.js | JavaScript をパソコンやサーバーの上で動かすプログラム。もともと JavaScript はブラウザの中でしか動かなかった | エンジン |
| npm | Node.js 用のライブラリを、ネット上の置き場（npm レジストリ）から取ってきて入れる道具。Node.js に付いてくる | 部品の通販アプリ |
| npx | npm に付いてくる「入っている道具を、その場で実行する」コマンド。`npx wrangler login` は「このプロジェクトに入れた wrangler を動かす」という意味 | 買った工具をすぐ使うボタン |
| nvm | Node.js 自体のバージョンを入れたり切り替えたりする道具 | エンジンの載せ替え係 |

使う順番：nvm で Node.js を入れる → npm で wrangler や Hono を入れる → npx で wrangler を実行する。

## npm create（ひな形を作るコマンド）

`npm create` は npm に最初からあるサブコマンドで、`npm init` の別名。npm は次の読み替えのルールを決めている。

| 打つコマンド | npm が実際に実行するもの |
| --- | --- |
| `npm create foo` | `npx create-foo` |
| `npm create @foo` | `npx @foo/create` |

npm 自身は Cloudflare のことを知らない。ひな形を作るツールを配る側が、このルールに合わせてパッケージ名を `create-` で始めて公開している（`create-cloudflare`、`create-vite`、`create-next-app` など）。なので `npm create cloudflare@latest sample-cf-workers-d1` は `npx create-cloudflare@latest sample-cf-workers-d1` と同じ。

`create-cloudflare`（通称 C3）がやること：

1. 質問に答えた内容に合わせて、ひな形のファイルをフォルダに書き出す
2. そのフォルダで `npm install` を実行し、wrangler などを `node_modules/` に入れる
3. git を使うと答えたら、`git init` と最初のコミットまでする

C3 自体は一時的な場所で実行されるだけで、プロジェクトには入らない。できあがったプロジェクトで働くのは 2 で入った wrangler。

ちなみに引数なしの `npm init` は `package.json` を1つ作るだけで、こちらが `git init` に相当する。

## Node.js のバージョン

- 数字が大きいほど新しい。偶数のバージョンが長期サポート版（LTS）で、数年間はバグや脆弱性の修正が続く。
- 2026-11 時点では 24 が現役の LTS。22 はメンテナンス期間に入っていて、2027-04 にサポートが終わる。Ubuntu の apt で入る 18 は 2025-04 にサポートが終わっている。

## nvm と fnm

fnm（Fast Node Manager）は nvm と同じ役割の別の道具で、Rust で書かれていて速い。

- fnm の利点：ターミナルを開くたびに少し時間がかかる nvm の問題がない。プロジェクトのフォルダに入ると、指定のバージョンに自動で切り替えられる。
- nvm の利点：圧倒的に有名で、Microsoft の WSL 向け公式手順（「Set up Node.js on WSL 2」）もこちらを使っている。

採用：**nvm**。Node は 1 バージョンしか使わないので差はほぼない。情報が多いほうが、自分で調べるときにも AI に聞くときにも有利なので nvm にした。

## WSL で「node は Linux、npm は Windows」になる件

WSL ではよく知られた事象。

- WSL は既定で、Windows の PATH を Linux の PATH の後ろに足す。そのため Linux 側から `code .` や `explorer.exe` が使える。
- `node` は Linux 側の `/usr/bin/node`（apt で入ったもの）が先に見つかるので、そちらが使われる。
- Ubuntu では npm が `nodejs` とは別のパッケージになっている。Linux 側に npm がないので、PATH の後ろにある Windows 側の npm（`/mnt/c/Program Files/nodejs/npm`）まで探しにいってしまう。

困るのは「使えてしまう」ことそのものより、Linux の道具を使っているつもりで、実は Windows の道具が動いていること。Windows 側の npm で WSL 側のファイルを扱うと、動作がとても遅くなったり、Windows 用にビルドされた部品が混ざって Linux で動かなくなったりする。

対策（Microsoft 推奨）：WSL の中に nvm で Node.js を入れる。Linux 側の node と npm が PATH の前に来るので、Windows 側の npm は使われなくなる。`which node npm` で両方とも `~/.nvm/...` を指していることを確かめる。

もっと強い方法として、`/etc/wsl.conf` の `[interop]` に `appendWindowsPath = false` と書くと、Windows の PATH を丸ごと切り離せる。ただし `code .` なども使えなくなり、必要なものを個別に PATH へ足し直すことになるので、今回は採用しない。
