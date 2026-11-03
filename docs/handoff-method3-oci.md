# 引き継ぎ：方式③（OCI）のサンプル

最終更新：2026-11-03

方式③（汎用VM：OCI）のサンプルを、別のセッションで始めるための引き継ぎ。始めたら、決まったことは [学習ロードマップ](learning-roadmap.md) の新しい Step と [方式検討サマリ](infra-design-summary.md) に移し、このファイルは最後に消す。

## ここまでに決まっていること

- 方式の番号：⓪ 既存・Pages、① 標準・静的、② 標準・アプリ + DB、③ 汎用VM（OCI）、④ 学習方式（AWS / GCP / Azure。後回し）。
- 方式③の役割：Workers で動かないもの（常駐プロセス、WebSocket の常時接続、任意の言語、PostgreSQL が必須なもの）の受け皿。
- 方式検討サマリに書いてある当初の案：OCI Always Free の VM + Docker + Dokploy/Coolify + Cloudflare Tunnel。DB は VM 上の PostgreSQL（コンテナ）。
  - Docker は冗長化ではなく、依存関係の分離・再現性・サービスのフォルダ内で完結させるため。
  - 冗長化はしない。VM 1台が単一障害点になるのを受け入れ、Terraform + Ansible で「作り直せば元に戻る」状態を保つ。
  - DB は日次で R2 にバックアップ（RTO 数時間、RPO 1日）。
  - 公開は Cloudflare Tunnel（VM のポートを開けない）。サブドメインは①②と同じく `<名前>.alphajinsei.com`。
- 要件 R1（定常コストゼロ）は守る。Always Free の範囲に収める。

## ユーザーの希望（2026-11-03）

- OCI の Always Free は VM 以外にもいろいろあり、勉強になる構成が作れそう、と考えている。VM + Docker だけに絞らず、Always Free のサービスを組み合わせた構成も検討したい。
- 参考：https://www.oracle.com/jp/cloud/free/ （このセッションからは 403 で取得できなかった。最新の Always Free の一覧と上限は、公式ページかドキュメントでユーザーと確認する）
- 記憶による候補（要確認）：Ampere A1（計4 OCPU / 24GB）と AMD のマイクロ VM 2台、ブロック・ボリューム、Object Storage、Autonomous Database 2つ、NoSQL Database、MySQL HeatWave、ロードバランサー、VCN、Vault、Monitoring / Logging / Notifications、Bastion、Email Delivery など。

## 手元にあるもの

- OCI の VM と Autonomous Database は、ユーザーが **2023年10月** に作成した。今も残っているかは未確認。
  - 無料版の ADB は、7日使わないと止まり、止まったまま90日で回収される。消えている可能性が高い。
  - Always Free の VM も、使用率が低いと回収されることがある（無料のみのアカウントの場合）。
  - **最初に、OCI のコンソールで両方がまだあるかを、ユーザーに確認してもらう。**
- 接続の資料は Windows 側の `C:\Users\peinn\sandbox\OCI`（WSL からは `/mnt/c/Users/peinn/sandbox/OCI`）にある。ファイル名だけ確認済みで、中身はまだ読んでいない。

| ファイル | 中身の見当 | 扱い |
| --- | --- | --- |
| `ssh-key-2023-10-09.key` | VM の SSH 秘密鍵 | **Claude は読まない**（`.claude/settings.json` で Read を禁止済み）。使うときは、ユーザーに `~/.ssh/` へコピーして `chmod 600` してもらう（Windows 側は権限が 777 で、ssh が受け付けない） |
| `ssh-key-2023-10-09.key.pub` | 公開鍵 | 読んでよい |
| `Wallet_myautonomousDB.zip` | ADB の接続用の鍵と証明書 | **Claude は読まない**（同上） |
| `OCI_memo.txt`、`OCI-FSS.md`、`OCI-FSS図.pptx` | 接続の手順などのメモ | 秘密情報が書かれていないか、読む前にユーザーに確認する |
| `oracle-instantclient*.rpm` | sqlplus など | ADB を使う場合だけ |

## 次のセッションで決めること

1. 進め方：学習として、ユーザー自身がコマンドを実行するか（ロードマップに Step を書く）、Claude に任せるか。
2. 範囲：VM + Docker + PostgreSQL + Tunnel に絞るか、Always Free の他のサービス（ADB、Object Storage、ロードバランサーなど）も組み合わせるか。
3. DB：VM 上の PostgreSQL（コンテナ）、Autonomous Database、MySQL HeatWave のどれにするか。ADB は自動停止・回収と、無料版でプライベートエンドポイントが使えない（インターネット経由になる）点を伝えたうえで決める。
4. デプロイ：Dokploy / Coolify を入れるか、`docker compose` を手元から ssh で流すだけにするか。①②の「手元から1コマンド」にそろえるかどうか。
5. 構成管理：OCI の Terraform provider の認証（API キーを `infra/.secrets/` に置く）。state は R2 の `tfstate` バケットに `<サンプル名>/terraform.tfstate` で置く（①②と同じ）。VM の中の設定は Ansible か cloud-init か。
6. サンプルの名前：`sample-oci-<構成>`（例：`sample-oci-vm-docker`）。

## 決まりごと（CLAUDE.md とサービス開発ガイドにあるもの）

- サンプルは `~/claude-sandbox/` の直下に、独立した git リポジトリとして作る。作ったら `.claude/settings.json` の `additionalDirectories` と CLAUDE.md のディレクトリ構成に足す。
- サンプルの README は、[サービス開発ガイド](service-guide.md) の付録「README の見出し」の6つにそろえる。ガイドの「方式の一覧」の③の行を埋める。
- 新しく分かったことは [勉強メモ](study-notes.md) に追記し、長い解説は `docs/extra-*.md` に分ける。
- 鍵・パスワードは `infra/.secrets/` に置き、Claude は中身を読まない。リポジトリは public にする前提で書く。
- 手元の状態が変わる作業は、実行前にユーザーに確認する。
