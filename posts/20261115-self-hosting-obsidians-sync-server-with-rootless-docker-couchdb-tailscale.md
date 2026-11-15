---
title: "Obsidianの同期サーバーをRootless Docker + CouchDB + Tailscaleでセルフホストする"
description: "CouchDBを同期サーバーとして使うSelf-hosted LiveSync構成。"
pubDatetime: 2026-11-15T20:50:00+09:00
---

Obsidianの公式同期サービス「Obsidian Sync」を利用せず、自分のサーバーでiPhoneやPC間のノートを同期する環境を構築する。

今回はCouchDBを同期サーバーとして使用し、Self-hosted LiveSyncプラグインでObsidianと接続する。

セキュリティと管理のしやすさを考慮し、次の構成を採用する。

- **Rootless Docker**でCouchDBを実行する
- Docker Composeは使わず、`docker run`で起動する
- Dockerのネットワークは**Host Network**を使用する
- CouchDBのデータと設定ファイルはホストのディレクトリからbind mountする
- Rootless DockerのUID/GIDマッピングを事前に計算してファイルの所有権を設定する
- TailscaleはホストOS上で実行し、**Tailscale ServeでHTTPSを終端する**
- CouchDB自体は`127.0.0.1:5984`で待ち受ける
- iPhoneを含む各端末からTailnet経由で同期する

## 1. システム構成

通信経路は次のようになる。

```text
iPhone / PC / Mac
    │
    │ Obsidian
    │ Self-hosted LiveSync
    │
    ▼
Tailscale Network
    │
    │ HTTPS :443
    ▼
Linux Host
    │
    ├── Tailscale Serve
    │       │
    │       │ HTTP
    │       ▼
    │   127.0.0.1:5984
    │       │
    │       ▼
    └── Rootless Docker
            │
            │ --network host
            ▼
        CouchDB
            │
            ├── local.ini
            └── data/
```

CouchDBはインターネットへ直接公開しない。

Tailscale Serveを使うことで、Tailnetに参加している端末だけがHTTPSでCouchDBにアクセスできる。

## 2. 前提条件

次の環境を想定する。

| 項目 | 条件 |
|---|---|
| OS | Linux（Ubuntu、Debianなど） |
| Docker | Rootless Docker |
| Docker Engine | 29.5以降 |
| 起動方法 | `docker run` |
| CouchDB | 公式イメージ `couchdb:3` |
| ネットワーク | `--network host` |
| データ保存 | ホストディレクトリへのbind mount |
| VPN | Tailscale |
| HTTPS | Tailscale Serve |
| クライアント | Obsidian + Self-hosted LiveSync |

**Docker Engine 29.5以降が必要な理由**

古いRootless Dockerでは、`--network host`を指定してもRootlessKit内部のネットワーク名前空間が使われ、実際のホストOSからコンテナのlocalhostへアクセスできないことがある。

Docker Engine 29.5以降ではこの制約が解消されている。

Dockerのバージョンを確認する。

```bash
docker version --format '{{.Server.Version}}'
```

Rootless Dockerで動作していることを確認する。

```bash
docker info --format '{{json .SecurityOptions}}'
```

出力に`rootless`が含まれていることを確認する。

複数のDockerコンテキストがある場合は、Rootless Dockerを選択する。

```bash
docker context use rootless
```

Dockerのコンテキスト名は環境によって異なる。

## 3. 作業ディレクトリを作成する

ホームディレクトリにCouchDB用のディレクトリを作成する。

```bash
mkdir -p ~/obsidian-livesync/data
cd ~/obsidian-livesync
```

最終的なディレクトリ構成は次のとおり。

```text
~/obsidian-livesync/
├── .env
├── local.ini
└── data/
```

各ファイルの役割は以下のとおり。

| ファイル | 用途 |
|---|---|
| `.env` | CouchDBの管理者認証情報 |
| `local.ini` | CouchDBの設定 |
| `data/` | データベースファイルの永続保存 |

Docker Named Volumeは使用しない。

## 4. Rootless DockerのUID/GIDマッピングを理解する

CouchDBの公式Dockerイメージは、コンテナ内でUID `5984`のユーザーを使用する。

しかし、Rootless Dockerではコンテナ内のUIDとホストOSのUIDが一致しない。

例えば、ホストの`/etc/subuid`が次のようになっているとする。

```text
user:100000:65536
```

標準的なRootless Dockerでは、次のようにマッピングされる。

| コンテナ内UID | ホストUID |
|---:|---:|
| 0 | Docker実行ユーザーのUID |
| 1 | 100000 |
| 2 | 100001 |
| 1000 | 100999 |
| 5984 | 105983 |

計算式は以下となる。

```text
ホストUID = subuid開始値 + コンテナ内UID - 1
```

今回の例では、

```text
100000 + 5984 - 1 = 105983
```

となる。

GIDも同様に`/etc/subgid`を使用して計算する。

ただし、subuidとsubgidの開始値は必ずしも同じではない。

また、この計算式はコンテナ内ID 0がホストの実行ユーザーに割り当てられ、ID 1以降が従属ID範囲に割り当てられる標準構成を前提としている。

### UID/GIDを自動計算する

以下のBashスクリプトで、ホスト側のUID/GIDを計算する。

```bash
#!/usr/bin/env bash
set -euo pipefail

CONTAINER_UID=5984
CONTAINER_GID=5984

USER_NAME=$(id -un)

read -r SUBUID_START SUBUID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subuid
)

read -r SUBGID_START SUBGID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subgid
)

if [[ -z "${SUBUID_START:-}" ||
      -z "${SUBGID_START:-}" ]]; then
  echo "Subordinate UID/GID not found" >&2
  exit 1
fi

if (( CONTAINER_UID > SUBUID_COUNT ||
      CONTAINER_GID > SUBGID_COUNT )); then
  echo "UID/GID is outside the mapped range" >&2
  exit 1
fi

HOST_UID=$((SUBUID_START + CONTAINER_UID - 1))
HOST_GID=$((SUBGID_START + CONTAINER_GID - 1))

printf 'Host UID: %s\n' "$HOST_UID"
printf 'Host GID: %s\n' "$HOST_GID"
```

例えば、subuidとsubgidの開始値が両方`100000`の場合、次の結果になる。

```text
Host UID: 105983
Host GID: 105983
```

この値は環境によって異なるため、固定値として扱わない。

## 5. CouchDBの設定ファイルを作成する

`local.ini`を作成する。

```bash
nano ~/obsidian-livesync/local.ini
```

内容は次のとおり。

```ini
[couchdb]
single_node = true
max_document_size = 50000000

[cluster]
n = 1

[chttpd]
bind_address = 127.0.0.1
port = 5984
require_valid_user = true
enable_cors = true
max_http_request_size = 4294967296

[chttpd_auth]
require_valid_user = true

[httpd]
WWW-Authenticate = Basic realm="couchdb"

[cors]
credentials = true
origins = app://obsidian.md,capacitor://localhost,http://localhost
headers = accept,authorization,content-type,origin,referer
methods = GET,PUT,POST,HEAD,DELETE
max_age = 3600

[log]
level = warning
```

主要な設定項目について説明する。

### single_node

```ini
[couchdb]
single_node = true
```

CouchDBをシングルノードで運用する。

この設定により、起動時に必要なシステムデータベースが自動作成される。

### bind_address

```ini
[chttpd]
bind_address = 127.0.0.1
```

CouchDBがホストのループバックアドレスでのみ待ち受けるようにする。

今回はHost Networkを使用するため、この設定は重要である。

`0.0.0.0`にすると、CouchDBがホストの外部ネットワークインターフェースでも待ち受ける可能性がある。

### require_valid_user

```ini
[chttpd]
require_valid_user = true

[chttpd_auth]
require_valid_user = true
```

認証されていないアクセスを拒否する。

### CORS

```ini
[cors]
credentials = true
origins = app://obsidian.md,capacitor://localhost,http://localhost
```

Obsidianのデスクトップアプリおよびモバイルアプリからアクセスするための設定である。

Self-hosted LiveSyncが必要とするCORS設定を指定している。

## 6. CouchDBの管理者認証情報を作成する

`.env`を作成する。

```bash
cd ~/obsidian-livesync

umask 077

printf 'COUCHDB_USER=admin\nCOUCHDB_PASSWORD=%s\n' \
  "$(openssl rand -hex 24)" > .env
```

内容は次の形式になる。

```dotenv
COUCHDB_USER=admin
COUCHDB_PASSWORD=ランダム生成されたパスワード
```

ファイルのパーミッションを設定する。

```bash
chmod 600 .env
```

このパスワードは後ほどObsidianのSelf-hosted LiveSyncでも使用する。

`.env`はGitなどにコミットしない。

## 7. bind mount用ディレクトリの所有権を設定する

Rootless DockerのUID/GIDマッピングに合わせて、`data/`と`local.ini`の所有権を調整する。

以下は標準的な単一subuid/subgid範囲を使用する環境向けの例である。

```bash
cd ~/obsidian-livesync

CONTAINER_UID=5984
CONTAINER_GID=5984

USER_NAME=$(id -un)

read -r SUBUID_START SUBUID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subuid
)

read -r SUBGID_START SUBGID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subgid
)

if [[ -z "${SUBUID_START:-}" ||
      -z "${SUBGID_START:-}" ]]; then
  echo "Subordinate UID/GID not found" >&2
  exit 1
fi

if (( CONTAINER_UID > SUBUID_COUNT ||
      CONTAINER_GID > SUBGID_COUNT )); then
  echo "UID/GID is outside the mapped range" >&2
  exit 1
fi

HOST_UID=$((SUBUID_START + CONTAINER_UID - 1))
HOST_GID=$((SUBGID_START + CONTAINER_GID - 1))

sudo chown -R "$HOST_UID:$HOST_GID" ./data
sudo chown "$HOST_UID:$HOST_GID" ./local.ini

sudo chmod 700 ./data
sudo chmod 644 ./local.ini
```

この方法で、コンテナ内UID/GID `5984:5984`が、対応するホスト側の所有権でファイルにアクセスできるようになる。

`local.ini`は読み取り専用でマウントするため、実際には読み取り権限さえあればよい。所有権の変更は必須ではないが、今回は両方とも同じCouchDBユーザーにそろえる。

なお、既存のデータがある場合は、バックアップを取得してから所有権を変更する。

**注意：** この方法は標準的なRootless Dockerのユーザー名前空間を前提としている。独自のIDマッピングや複数の従属ID範囲を使う場合は、実際の`uid_map`と`gid_map`を確認する必要がある。

## 8. Dockerイメージを取得する

CouchDBの公式Dockerイメージを取得する。

```bash
docker pull couchdb:3
```

ここではCouchDB 3系を使用する。

本番環境では、動作確認したバージョンのタグまたはダイジェストを固定しておくと、意図しないバージョン更新を防げる。

## 9. docker runでCouchDBを起動する

以下のコマンドを実行する。

```bash
cd ~/obsidian-livesync

docker run -d \
  --name obsidian-couchdb \
  --restart unless-stopped \
  --network host \
  --env-file "$PWD/.env" \
  --env ERL_EPMD_ADDRESS=127.0.0.1 \
  --env "ERL_FLAGS=-kernel inet_dist_use_interface '{127,0,0,1}'" \
  --mount type=bind,src="$PWD/data",dst=/opt/couchdb/data \
  --mount type=bind,src="$PWD/local.ini",dst=/opt/couchdb/etc/local.ini,readonly \
  couchdb:3
```

オプションの役割は次のとおり。

| オプション | 内容 |
|---|---|
| `-d` | バックグラウンドで実行 |
| `--name` | コンテナ名 |
| `--restart unless-stopped` | Docker起動時に自動復帰 |
| `--network host` | ホストネットワークを使用 |
| `--env-file` | 管理者認証情報を読み込む |
| `ERL_EPMD_ADDRESS` | Erlangのポートマッパーをループバックに制限 |
| `ERL_FLAGS` | Erlangノード間通信の待受インターフェースを制限 |
| `--mount` | データと設定ファイルをbind mount |

データディレクトリは書き込み可能、設定ファイルは読み取り専用としてマウントする。

Rootless Dockerでは、Dockerを操作する際に`sudo`を付けない。

### 補足：Erlang関連ポート

CouchDBはHTTPの5984番以外に、Erlangのノード管理・通信に関連するポートを使用する。

今回はHost Networkなので、これらのポートが外部公開されないことも確認する必要がある。

指定した環境変数に加え、後述の`ss`コマンドで実際の待受アドレスを確認する。

## 10. CouchDBの起動を確認する

コンテナが起動しているか確認する。

```bash
docker ps --filter name=obsidian-couchdb
```

ログを確認する。

```bash
docker logs --tail 100 obsidian-couchdb
```

CouchDBのヘルスチェックを実行する。

```bash
curl http://127.0.0.1:5984/_up
```

正常に起動していれば、以下のような応答が返る。

```json
{"status":"ok"}
```

管理者認証も確認する。

```bash
curl -u admin \
  http://127.0.0.1:5984/
```

パスワード入力後にCouchDBの情報が返れば、認証できている。

## 11. Obsidian用データベースを作成する

同期用データベースを作成する。

```bash
curl --fail-with-body \
  -u admin \
  -X PUT \
  http://127.0.0.1:5984/obsidiannotes
```

パスワードを入力すると、正常時は以下のような応答になる。

```json
{"ok":true}
```

既存のデータベースがある場合、作成済みであることを示すエラーが返る。

データベース一覧を確認する。

```bash
curl -u admin \
  http://127.0.0.1:5984/_all_dbs
```

シングルノード設定が正常に機能していれば、システムデータベースも含めて一覧が返る。

## 12. ネットワークの待受状態を確認する

Host Networkを使用しているため、CouchDBが外部インターフェースで待ち受けていないことを確認する。

```bash
sudo ss -lntp | grep -E ':(5984|4369|9100)\b'
```

5984番については、次のような待受状態が望ましい。

```text
127.0.0.1:5984
```

`0.0.0.0:5984`になっている場合は設定を確認する。

4369番や9100番などのErlang関連ポートについても、外部に公開されていないことを確認する。

## 13. ホストOS側でTailscale Serveを設定する

TailscaleはDockerコンテナ内ではなく、ホストOS上で動かす。

まずTailscaleがTailnetへ接続しているか確認する。

```bash
sudo tailscale status
```

続いてTailscale Serveを設定する。

```bash
sudo tailscale serve --bg \
  http://127.0.0.1:5984
```

これにより、TailscaleがHTTPS通信を受け付け、ホストのCouchDBへHTTPで転送する。

設定状態を確認する。

```bash
sudo tailscale serve status
```

例えば、次のようなHTTPSのURLが割り当てられる。

```text
https://livesync.example.ts.net
```

これは例示用のURLである。実際には自分のTailnetに割り当てられたURLを使用する。

Tailscale ServeのHTTPS証明書機能が未設定の場合は、Tailscale管理画面からMagicDNSとHTTPSを有効にする。

**Tailscale Funnelは使用しない。**

ServeはTailnet内からのアクセスに限定し、インターネットからの一般公開を避ける。

## 14. HTTPS経由の動作確認

Tailscaleに接続した別端末から確認する。

```bash
curl https://livesync.example.ts.net/_up
```

正常なら以下の応答が得られる。

```json
{"status":"ok"}
```

認証が必要なエンドポイントも確認する。

```bash
curl -u admin \
  https://livesync.example.ts.net/
```

さらにCORSヘッダーを確認する。

```bash
curl -i \
  -H "Origin: app://obsidian.md" \
  -u admin \
  https://livesync.example.ts.net/
```

レスポンスに以下のヘッダーが含まれることを確認する。

```text
Access-Control-Allow-Origin: app://obsidian.md
```

CORSに問題がある場合は`local.ini`の設定を見直す。

## 15. PCのObsidianを接続する

PCまたはMacにObsidianをインストールする。

続いて、コミュニティプラグインから**Self-hosted LiveSync**をインストールして有効化する。

設定画面でCouchDBを同期先として選択する。

| 設定項目 | 値 |
|---|---|
| Remote Type | CouchDB |
| URI | `https://livesync.example.ts.net` |
| Username | `admin` |
| Password | `.env`で生成したパスワード |
| Database name | `obsidiannotes` |
| End-to-End Encryption | 有効を推奨 |

E2EEのパスフレーズは、CouchDBの管理者パスワードとは別に設定する。

既存のVaultを同期する場合は、あらかじめバックアップを取得する。

新しい空のデータベースへ同期する場合と、すでにデータが存在するデータベースへ参加する場合では初期化操作が異なるため、プラグインのセットアップ画面を確認して選択する。

## 16. iPhoneのObsidianを接続する

iPhoneに以下のアプリをインストールする。

- Obsidian
- Tailscale

Tailscaleにログインし、PCと同じTailnetへ接続する。

Obsidianで新しい空のVaultを作成し、Self-hosted LiveSyncを有効にする。

PC側でプラグインのセットアップURIを発行し、iPhone側へ取り込む。

セットアップURIには接続情報が含まれるため、安全な方法で転送し、保護用パスフレーズを設定する。

iPhone側では既存の同期環境に参加する手順を選択し、サーバーからノートを取得する。

設定後、PCとiPhoneの両方でテスト用ノートを作成・編集し、双方向に同期されることを確認する。

iOSにはバックグラウンド実行の制限があるため、常時バックグラウンド同期が保証されるわけではない。確実に同期させたい場合はObsidianを起動して同期状況を確認する。

## 17. コンテナの管理

日常的な操作は`docker`コマンドで行う。

### 起動

```bash
docker start obsidian-couchdb
```

### 停止

```bash
docker stop obsidian-couchdb
```

### 再起動

```bash
docker restart obsidian-couchdb
```

### ログ確認

```bash
docker logs -f obsidian-couchdb
```

### コンテナ削除

```bash
docker rm -f obsidian-couchdb
```

コンテナを削除しても、bind mountした`~/obsidian-livesync/data/`は削除されない。

同じディレクトリをマウントしてコンテナを再作成すれば、データを引き継げる。

## 18. Rootless Dockerの自動起動

ホスト再起動後にもRootless Dockerを起動したい場合は、systemdのユーザーサービスを有効にする。

```bash
sudo loginctl enable-linger "$USER"
```

続いて以下を実行する。

```bash
systemctl --user enable --now docker
```

`docker run`時に`--restart unless-stopped`を指定しているため、Dockerデーモン起動後にコンテナの自動復帰が期待できる。

## 19. バックアップ

CouchDBのデータはホストのディレクトリに保存される。

```text
~/obsidian-livesync/data/
```

通常のファイルバックアップを取得する場合は、データベースファイルの整合性を確保するため、CouchDBを停止してからコピーするのが簡単である。

```bash
docker stop obsidian-couchdb
```

バックアップを取得する。

```bash
sudo tar -czf \
  "$HOME/obsidian-couchdb-backup.tar.gz" \
  -C "$HOME/obsidian-livesync" \
  data
```

完了したら再起動する。

```bash
docker start obsidian-couchdb
```

管理者認証情報と設定ファイルも別途バックアップする。

なお、CouchDBの同期データベースだけでなく、PCにあるObsidianのVaultそのものもバックアップしておくとよい。

同期はバックアップの代わりにはならない。

## 20. トラブルシューティング

### Permission deniedが発生する

Rootless DockerのUID/GIDマッピングを確認する。

```bash
cat /etc/subuid
cat /etc/subgid
```

実際のコンテナのマッピングは次の方法で確認できる。

```bash
PID=$(docker inspect -f '{{.State.Pid}}' obsidian-couchdb)

sudo cat "/proc/$PID/uid_map"
sudo cat "/proc/$PID/gid_map"
```

マッピングが想定と異なる場合、`data/`と`local.ini`の所有権を再設定する。

### CouchDBへ接続できない

ホスト側から確認する。

```bash
curl http://127.0.0.1:5984/_up
```

これが失敗する場合、まずCouchDBの起動状態と`local.ini`を確認する。

また、Rootless DockerのバージョンとHost Networkの動作を確認する。

### PCでは接続できるがiPhoneでは接続できない

以下を確認する。

- iPhoneのTailscaleが接続されているか
- Tailscale ServeのHTTPS URLが正しいか
- CouchDBのCORS設定が正しいか
- Self-hosted LiveSyncの接続情報が一致しているか

### 同期時に認証エラーが出る

`.env`の管理者認証情報と、Obsidian側に登録した認証情報を比較する。

また、CouchDBの設定変更が正しく反映されているか確認する。

## 21. 最終構成まとめ

| 項目 | 構成 |
|---|---|
| アプリケーション | Obsidian |
| 同期プラグイン | Self-hosted LiveSync |
| 同期サーバー | Apache CouchDB |
| コンテナ実行環境 | Rootless Docker |
| コンテナ起動 | `docker run` |
| ネットワーク | Host Network |
| CouchDBポート | `127.0.0.1:5984` |
| HTTPS終端 | ホストOSのTailscale Serve |
| VPN | Tailscale |
| データ保存 | ホストディレクトリへのbind mount |
| 設定ファイル | `local.ini` |
| UID/GID | `/etc/subuid`・`/etc/subgid`から計算 |
| Docker Compose | 不使用 |
| Docker Named Volume | 不使用 |
| iOS対応 | あり |
| ソフトウェア利用料 | 無料 |

この構成では、Obsidianの同期データを自分のサーバーで管理しながら、iPhone・PC・MacからHTTPS経由で利用できる。

CouchDBをRootless Dockerで稼働させ、ネットワークの公開範囲をTailscaleに限定することで、一般公開のリバースプロキシやDocker内のVPNコンテナを用意する必要もなくなる。

ただし、Rootless Dockerのユーザー名前空間、Host Networkの挙動、CouchDBの権限設定には環境差がある。初回起動時には必ず待受アドレス・所有権・同期動作を検証すること。

---

## 参考資料

- [Docker公式：Rootless mode](https://docs.docker.com/engine/security/rootless/)
- [Docker公式：Rootless modeの制限とトラブルシューティング](https://docs.docker.com/engine/security/rootless/troubleshoot/)
- [Apache CouchDB公式：Docker Installation](https://docs.couchdb.org/en/stable/install/docker.html)
- [Apache CouchDB公式：Single Node Setup](https://docs.couchdb.org/en/stable/setup/single-node.html)
- [Self-hosted LiveSync公式リポジトリ](https://github.com/vrtmrz/obsidian-livesync)
- [Tailscale公式：Serve](https://tailscale.com/docs/features/tailscale-serve)

