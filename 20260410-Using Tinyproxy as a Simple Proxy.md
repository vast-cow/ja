---
pubDatetime: 2026-04-10T16:26:01+09:00
title: "Tinyproxyをシンプルなプロキシとして使う方法"
description: "Tinyproxyのインストール、基本設定、Tinyproxyの起動を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

この記事では、Alpine Linux上で軽量なHTTP/HTTPSプロキシである**Tinyproxy**のセットアップと使用方法に限定して解説します。目的は、最小構成でプロキシを動作させ、別のマシンやクライアントから利用する方法を理解することです。

---

## 1. Tinyproxyのインストール

Alpine Linuxではインストールは簡単です：

```sh
apk add tinyproxy
```

これにより、Tinyproxyサービスとデフォルトの設定ファイルがインストールされます。

---

## 2. 基本設定

主な設定ファイルは通常以下にあります：

```
/etc/tinyproxy/tinyproxy.conf
```

最低限、以下を設定します：

```
Port 8080
Listen 0.0.0.0
```

### これらの設定の意味

* **Port 8080**
  Tinyproxyはポート8080でプロキシ接続を待ち受けます。

* **Listen 0.0.0.0**
  すべてのネットワークインターフェースからの接続を許可します（localhostのみではない）。
  別のマシンからアクセスする場合に必要です。

---

## 3. Tinyproxyの起動

用途に応じて2つの動作モードがあります。

### オプションA — フォアグラウンドで実行（デバッグモード）

```sh
tinyproxy -d -c /etc/tinyproxy/tinyproxy.conf
```

* `-d`: フォアグラウンドで実行（デバッグやログ確認に有用）
* `-c`: 設定ファイルを指定

テストやトラブルシューティング時に使用します。

---

### オプションB — バックグラウンドサービスとして実行

```sh
rc-service tinyproxy start
rc-update add tinyproxy
```

* Tinyproxyをデーモンとして起動
* 起動時に自動実行されるよう設定

---

## 4. Tinyproxyの使用方法

起動後、Tinyproxyは標準的なHTTPプロキシとして動作します。

### 例：クライアントからの利用

Tinyproxyサーバーが以下の場合：

```
172.27.59.40:8080
```

ツール側でプロキシとして設定できます。

#### `curl`を使用する場合

```sh
curl -x http://172.27.59.40:8080 http://example.com
```

#### 環境変数を使用する場合

```sh
export http_proxy="http://172.27.59.40:8080"
export https_proxy="http://172.27.59.40:8080"
```

これで多くのCLIツールがTinyproxy経由で通信するようになります。

---

## 5. 例：SSHでTinyproxyを使う（netcat経由）

TinyproxyはHTTP CONNECTメソッドを使ってTCP接続のプロキシも可能です。

SSH設定に以下を追加します：

```ssh
Host foomachine
    HostName 1.2.3.4
    ProxyCommand nc -X connect -x 172.27.59.40:8080 %h %p
```

### 説明

* `nc -X connect` → HTTP CONNECTプロキシモードを使用
* `-x 172.27.59.40:8080` → Tinyproxyサーバーを指定
* `%h %p` → 接続先ホストとポート

通常通り接続します：

```sh
ssh foomachine
```

接続はTinyproxy経由でルーティングされます。

---

## まとめ

Tinyproxyは以下を提供します：

* 最小構成で動作する軽量なHTTP/HTTPSプロキシ
* シンプルな設定（ポート + リッスンアドレス）
* 柔軟な利用方法：

  * CLIツール（`curl`、環境変数）
  * `nc`を使ったSSHトンネリング
  * 一般的なプロキシ用途

多くの場合、上記の最小構成だけで迅速に動作するプロキシを構築できます。
