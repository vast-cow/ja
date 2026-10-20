---
title: "Rootless Dockerの`--network host`が、いつの間にか本当に機能するようになっていた"
description: "Rootless Dockerで`--network host`が実ホストのネットワーク名前空間を共有できるようになったのは、Docker Engine 29.5.0からである。"
pubDatetime: 2026-10-20T13:01:00+09:00
---

## はじめに

Rootless Dockerで`--network host`を使ったことがある人なら、次のような挙動に遭遇したことがあるかもしれません。

```bash
docker run --rm --network host \
  curlimages/curl http://127.0.0.1:8080
```

ホスト側で8080番ポートのサービスが動いているにもかかわらず、コンテナから接続できない。

通常のDocker（rootfulモード）では接続できるのに、rootlessモードでは接続できないという挙動です。

これは従来のRootless Dockerの制約でした。

ところが調べてみると、**Docker Engine 29.5.0から、この制約が解消されていました。**

以前は`--network host`を指定しても実ホストのネットワーク名前空間を共有できませんでしたが、現在は通常のDockerに近い形で動作するようになっています。

この記事では、従来の動作、変更された仕組み、バージョン境界、動作確認方法を整理します。

## 従来のRootless Dockerでは`--network host`が期待どおりに動かなかった

Dockerの`--network host`は、コンテナがホストのネットワーク名前空間を共有するためのオプションです。

通常のDockerでは、次のような挙動になります。

- コンテナ内の`127.0.0.1`がホストのループバックインターフェースを指す
- コンテナ内で待ち受けたポートをホストから直接利用できる
- NATやポートフォワーディングを介さず通信できる
- `-p`によるポート公開は不要になる

しかし、従来のRootless Dockerでは事情が異なりました。

Rootless Dockerは、root権限を使用せずにDockerデーモンやコンテナを動作させるため、RootlessKitを利用します。

従来の構造を簡略化すると次のようになります。

```text
実ホストのNetwork Namespace
│
└── RootlessKitのNetwork Namespace
    │
    ├── dockerd
    │
    └── コンテナ（--network host）
```

この構造では、`--network host`を指定しても、コンテナが共有するのはRootlessKit側のネットワーク名前空間です。

**実ホストのネットワーク名前空間ではありません。**

そのため、コンテナ内で`127.0.0.1:8080`に接続しても、実ホストの`127.0.0.1:8080`には接続できませんでした。

また、コンテナ内でポートを待ち受けても、そのポートが実ホストに直接公開されるわけではありません。

実ホストにサービスを公開したい場合は、通常、`--network host`ではなく`-p`によるポートフォワーディングを使う必要がありました。

```bash
docker run --rm -p 8080:80 nginx:alpine
```

つまり、Rootless Dockerでは`--network host`という指定自体は受け付けられるものの、rootfulモードとは意味が異なっていたわけです。

## Docker Engine 29.5.0で仕様が変更された

この挙動が変更されたのは、**2026年5月14日にリリースされたDocker Engine 29.5.0**です。

公式リリースノートのNetworkingセクションには、次の変更が記載されています。

> Rootless: Properly support `--net=host` and localhost registries.

関連する実装は、MobyのPull Request [#47103](https://github.com/moby/moby/pull/47103)です。

変更前後を整理すると、次のようになります。

| 項目 | 29.4.3以前 | 29.5.0以降 |
|---|---|---|
| `--network host`の指定 | 可能 | 可能 |
| 共有されるネットワーク名前空間 | RootlessKit側 | 実ホスト側 |
| ホストの`127.0.0.1`へのアクセス | 原則不可 | 可能 |
| コンテナの待受ポート | 実ホストとは分離 | 実ホストと共有 |
| `-p`によるポート公開 | 無効 | 無効 |

ポイントは、`--network host`が新たに追加されたわけではなく、**既存のオプションが本来期待される動作をするように修正された**という点です。

## 何が変わったのか

実装上の重要な変更の一つが、RootlessKitの`--detach-netns`を利用するようになったことです。

Docker Engine 29.5.0の`dockerd-rootless.sh`には、次の設定があります。

```bash
: "${DOCKERD_ROOTLESS_ROOTLESSKIT_DETACH_NETNS:=true}"
```

`DOCKERD_ROOTLESS_ROOTLESSKIT_DETACH_NETNS`のデフォルトが`true`になっています。

この設定により、DockerデーモンをRootlessKitのネットワーク名前空間から分離し、実ホストのネットワーク名前空間で動作させられます。

変更後の構造を簡略化すると、次のようになります。

```text
実ホストのNetwork Namespace
│
├── dockerd（rootless）
│
├── コンテナ（--network host）
│
└── ホスト上のサービス
```

一方で、通常のbridgeネットワークを利用するコンテナについては、引き続きrootlessのネットワーク機構が使用されます。

つまり、rootlessモードそのものが廃止されたわけでも、すべてのコンテナがホストネットワークを共有するようになったわけでもありません。

`--network host`を明示的に指定した場合に、実ホストのネットワーク名前空間を利用できるようになった、という変更です。

### 従来の動作に戻すこともできる

Docker Engine 29.5.0以降でも、次の環境変数によって新しい動作を無効化できます。

```bash
DOCKERD_ROOTLESS_ROOTLESSKIT_DETACH_NETNS=false
```

そのため、29.5.0以降であっても、起動設定によっては従来の挙動になる可能性があります。

## 実際に動作を確認する

Linux上のRootless Dockerで、ホストのlocalhostに接続できるか確認してみます。

### 1. Dockerのバージョンを確認

```bash
docker version --format '{{.Server.Version}}'
```

Docker Engine 29.5.0以降であることを確認します。

続いてrootlessモードかどうかを確認します。

```bash
docker info --format '{{json .SecurityOptions}}'
```

出力に`rootless`が含まれていれば、rootlessモードで動作しています。

### 2. ホスト側でHTTPサーバーを起動

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

ホストのループバックインターフェースでHTTPサーバーを起動します。

### 3. コンテナから接続する

別のターミナルから次のコマンドを実行します。

```bash
docker run --rm --network host \
  curlimages/curl http://127.0.0.1:8080
```

Docker Engine 29.5.0以降の標準設定であれば、ホスト側のHTTPサーバーに接続できるはずです。

従来のRootless Dockerでは、この接続は通常失敗していました。

### 4. ネットワーク名前空間を比較する

さらに、ホストとコンテナが同じネットワーク名前空間を利用しているか確認できます。

ホスト側：

```bash
readlink /proc/self/ns/net
```

コンテナ側：

```bash
docker run --rm --network host alpine \
  readlink /proc/self/ns/net
```

両者で同じネットワーク名前空間のinode番号が表示されれば、ネットワーク名前空間が共有されていることを確認できます。

## セキュリティ上の注意点

`--network host`がrootfulモードに近い動作になったからといって、コンテナがホストのroot権限を取得するわけではありません。

Rootless Dockerのユーザー名前空間による権限制限は引き続き存在します。

ただし、ネットワークの分離はなくなります。

例えば、次の点には注意が必要です。

- ホストのループバックアドレスで待ち受けるサービスにアクセスできる
- ホスト上のサービスと待受ポートが競合する
- ネットワーク名前空間による隔離がなくなる
- privileged portへのbindなどは引き続き権限の制約を受ける

特に、`127.0.0.1`で待ち受けているサービスを「ホスト上のプロセスからしかアクセスできない」という前提で運用している場合は、`--network host`を指定したコンテナからもアクセスできるようになる点に注意が必要です。

## まとめ

Docker Engine 29.5.0で、Rootless Dockerの`--network host`の挙動が変更されました。

従来はRootlessKit内部のネットワーク名前空間を共有するだけだったため、実ホストのlocalhostに直接アクセスできませんでした。

しかし、29.5.0以降では、RootlessKitの`--detach-netns`を利用することで、実ホストのネットワーク名前空間を共有できるようになっています。

個人的には、Rootless Dockerの`--network host`は実ホストとは別物、という認識のままだったので、この変更は意外でした。

Rootless Dockerでホストのlocalhostへ接続するために回避策を使っている場合は、Docker Engineのバージョンを確認してみる価値がありそうです。

## 参考資料

- [Docker Engine v29 Release Notes（29.5.0）](https://docs.docker.com/engine/release-notes/29/#2950)
- [Moby PR #47103 — rootless: properly support --net=host and localhost registries](https://github.com/moby/moby/pull/47103)
- [Docker Engine 29.4.3のdockerd-rootless.sh](https://github.com/moby/moby/blob/docker-v29.4.3/contrib/dockerd-rootless.sh)
- [Docker Engine 29.5.0のdockerd-rootless.sh](https://github.com/moby/moby/blob/docker-v29.5.0/contrib/dockerd-rootless.sh)
- [Docker公式ドキュメント — Rootless mode](https://docs.docker.com/engine/security/rootless/)
