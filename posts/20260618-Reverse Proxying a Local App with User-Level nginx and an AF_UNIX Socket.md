---
pubDatetime: 2026-06-18T16:09:29+09:00
title: "ユーザー権限 nginx + AF_UNIX socket でローカルアプリを reverse proxy する"
description: "方針、ディレクトリ構成、nginx の起動を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

複数のローカル Web アプリを手元で動かしていると、URL の prefix ごとに別アプリへ振り分けたいことがある。

たとえば、

```text
http://localhost:8080/app1/
```

へのアクセスを、別プロセスとして起動しているアプリに reverse proxy したい。

このとき、nginx を使うと簡単にできる。ただし、通常のシステム nginx としてではなく、今回は次の方針で運用する。

* nginx をユーザー権限で起動する
* アプリとの接続には TCP port ではなく AF_UNIX socket を使う
* nginx の設定ファイルを見るだけで、URL path とアプリの対応が分かるようにする

## 方針

構成としては、nginx を `localhost:8080` で待ち受けさせる。

そのうえで、たとえば `/app1/` 以下を、アプリが listen している Unix domain socket に転送する。

```text
browser
  |
  | http://localhost:8080/app1/
  v
nginx
  |
  | AF_UNIX socket
  v
/home/user/app1/sock.sock
```

TCP port をアプリごとに割り当ててもよいが、ローカル開発用の reverse proxy では Unix domain socket の方が見通しがよい場合がある。

設定ファイル上で、

```nginx
location /app1/ {
    proxy_pass http://unix:/home/user/app1/sock.sock:/;
}
```

のように書けるので、`/app1/` がどのアプリに対応しているかが直接分かる。

## ディレクトリ構成

nginx 用のディレクトリを適当に作る。

```text
nginx-local/
├── conf/
│   └── nginx.conf
├── logs/
└── temp/
    ├── client_body/
    ├── proxy/
    ├── fastcgi/
    ├── uwsgi/
    └── scgi/
```

作成例は以下。

```bash
mkdir -p ./conf
mkdir -p ./logs
mkdir -p ./temp/client_body
mkdir -p ./temp/proxy
mkdir -p ./temp/fastcgi
mkdir -p ./temp/uwsgi
mkdir -p ./temp/scgi
```

## nginx の起動

今回の nginx は foreground で起動する。

```bash
nginx -p . -c ./conf/nginx.conf -e ./logs/error.log -g 'daemon off;'
```

`-p .` によって nginx の prefix をカレントディレクトリにする。

設定ファイル中の相対パス、たとえば `./logs/error.log` や `./temp/proxy` は、この prefix を基準に解決される。

`-g 'daemon off;'` を付けているので、nginx は daemon 化せず foreground で動く。開発用や検証用にはこの方が扱いやすい。停止するときは `Ctrl-C` でよい。

## 設定ファイル

`conf/nginx.conf` は以下のようにした。

```nginx
worker_processes 1;

error_log ./logs/error.log;
error_log  /dev/stderr warn;
pid       ./nginx.pid;

events {
    worker_connections 1024;
}

http {
    access_log ./logs/access.log;
    access_log  /dev/stderr;

    client_body_temp_path ./temp/client_body;
    proxy_temp_path       ./temp/proxy;
    fastcgi_temp_path     ./temp/fastcgi;
    uwsgi_temp_path       ./temp/uwsgi;
    scgi_temp_path        ./temp/scgi;

    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        listen 8080;
        server_name localhost;

        location = /app1 {
            return 301 /app1/;
        }

        location /app1/ {
            proxy_pass http://unix:/home/user/app1/sock.sock:/;

            proxy_http_version 1.1;

            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_set_header Upgrade           $http_upgrade;
            proxy_set_header Connection        $connection_upgrade;

            proxy_buffering off;
            proxy_cache off;
            proxy_request_buffering off;

            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }
    }
}
```

## ユーザー権限で起動するためのポイント

nginx をユーザー権限で起動する場合、いくつか注意点がある。

まず、listen する port は 1024 以上にする。

```nginx
listen 8080;
```

`80` や `443` のような privileged port を一般ユーザーで直接 bind しようとすると失敗する。

次に、ログ、pid、temporary file の出力先をユーザーが書き込める場所にする。

```nginx
error_log ./logs/error.log;
pid       ./nginx.pid;
```

また、HTTP proxy 時に使われる temporary directory も明示しておく。

```nginx
client_body_temp_path ./temp/client_body;
proxy_temp_path       ./temp/proxy;
fastcgi_temp_path     ./temp/fastcgi;
uwsgi_temp_path       ./temp/uwsgi;
scgi_temp_path        ./temp/scgi;
```

これらを明示しないと、環境によっては `/var/log/nginx/` や `/var/lib/nginx/` など、ユーザー権限では書けない場所を使おうとして失敗することがある。

## AF_UNIX socket への reverse proxy

Unix domain socket へ proxy する場合、`proxy_pass` は次の形式で書ける。

```nginx
proxy_pass http://unix:/path/to/sock.sock:/;
```

今回の例では次のようにしている。

```nginx
proxy_pass http://unix:/home/user/app1/sock.sock:/;
```

この書き方では、`/app1/` の prefix は剥がされる。

つまり、

```text
http://localhost:8080/app1/foo
```

へのアクセスは、バックエンドアプリには

```text
/foo
```

として渡る。

アプリ側で `/app1/foo` をそのまま受けたい場合は、`proxy_pass` の末尾の扱いを変える必要がある。

今回の用途では、nginx 側で `/app1/` をルーティング用 prefix として使い、アプリ側には `/` 起点で渡す方針にした。

## WebSocket への対応

WebSocket を通すには、HTTP/1.1 と `Upgrade` / `Connection` header の設定が必要になる。

```nginx
proxy_http_version 1.1;

proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

ここで `$connection_upgrade` は nginx の組み込み変数ではないため、`http` block の中で `map` によって定義している。

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

`Connection "upgrade"` を常に指定してしまうと、通常の HTTP request や SSE に対しても `Connection: upgrade` を付けてしまう。そこで、`Upgrade` header がある場合だけ `upgrade` にし、それ以外では `close` にしている。

## Server-Sent Events / HTTP event stream への対応

SSE や HTTP event stream を使う場合、nginx の buffering があるとイベントが即時にクライアントへ流れないことがある。

そのため、proxy buffering を無効にする。

```nginx
proxy_buffering off;
proxy_cache off;
```

長時間接続が切れないように、timeout も長めにしている。

```nginx
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

これで、通常の HTTP、WebSocket、SSE のいずれも同じ reverse proxy 設定で扱える。

## 動作確認

設定ファイルの構文確認は以下。

```bash
nginx -p . -c ./conf/nginx.conf -e ./logs/error.log -t
```

起動は以下。

```bash
nginx -p . -c ./conf/nginx.conf -e ./logs/error.log -g 'daemon off;'
```

HTTP の確認。

```bash
curl -i http://localhost:8080/app1/
```

SSE の確認なら、curl の buffering を切る。

```bash
curl -N http://localhost:8080/app1/events
```

WebSocket の確認には `websocat` などを使う。

```bash
websocat ws://localhost:8080/app1/ws
```

## まとめ

この構成の利点は大きく2つある。

1つ目は、nginx をユーザー権限で起動できること。

システムの nginx 設定を変更せず、root 権限も使わずに、手元の作業ディレクトリだけで reverse proxy を立てられる。ローカル開発や個人用ツールの取り回しがよい。

2つ目は、AF_UNIX socket を使うことで、設定ファイル上の対応関係が明確になること。

```nginx
location /app1/ {
    proxy_pass http://unix:/home/user/app1/sock.sock:/;
}
```

このように書けるため、`/app1/` がどのアプリに対応しているかを nginx の設定ファイルだけで把握しやすい。

ローカルで複数の Web アプリをまとめて扱う用途では、ユーザー権限 nginx と AF_UNIX socket の組み合わせはかなり扱いやすい。
