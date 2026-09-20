---
title: "Supermicro X11DPI (ATEN IPMI) を Tailscale Serve で公開するなら、Nginx HTTP Proxy ではなく socat / Nginx stream を使う"
description: "やりたかったこと、最初に疑ったのはWebSocket、実際の原因を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-07-31T11:01:26.884Z
updatedDate: 2026-07-31T11:03:37.278Z
---

Supermicro X11世代のIPMI（ATENベース）を Tailscale Services (`tailscale serve --service`) で公開しようとしたところ、意外な落とし穴にはまりました。

結論から言うと、

> **ATEN IPMIを Tailscale Serve 経由で公開する場合は、HTTPリバースプロキシではなく TCPレベルで中継する構成（socat または Nginx stream）が第一選択です。**

## やりたかったこと

LAN内のIPMIを

```plaintext
https://x11dpi-ipmi.<tailnet>.ts.net/
```

のようなサービス名で公開したい。

Tailscale Services を使えば、

```bash
tailscale serve \
  --service=svc:x11dpi-ipmi \
  --https=443 \
  https+insecure://x.x.x.x
```

のような構成が作れます。

しかし、実際にはHTML5 KVMが正常に動作しませんでした。

## 最初に疑ったのはWebSocket

HTML5 KVMはWebSocketを使用しています。

そのため、

* Tailscale ServeがWebSocketに対応していないのでは？
* Upgradeヘッダーが落ちているのでは？

と考えました。

しかし、自作の `aiohttp` WebSocketサーバーでは正常に動作しました。

つまり、

* Tailscale
* WebSocket
* ブラウザ

の組み合わせ自体には問題がありません。

## 実際の原因

NginxをHTTPリバースプロキシとして挟いて調査すると、

```plaintext
upstream sent invalid header: "\x20..."
```

というエラーが出ました。

つまり、

```plaintext
502 Bad Gateway
```

になっています。

これは

```plaintext
ブラウザ
    ↓
Tailscale
    ↓
Nginx HTTP Proxy
    ↓
ATEN IPMI
```

という構成で、

**ATEN IPMIが返すHTTPレスポンスヘッダーをNginxが不正と判断して拒否している**

ことを意味します。

GETでは問題なくても、

```http
POST /cgi/ipmi.cgi
```

のようなCGIでは失敗します。

例えば、

```plaintext
op=UID_SUPPORT.XML
```

をPOSTすると502になります。

一方、

```http
GET /cgi/ipmi.cgi
```

は正常に返ります。

つまり、

HTTPレベルの互換性問題です。

## これはWebSocketの問題ではない

最初はHTML5 KVMだけが失敗するのでWebSocketを疑いました。

しかし実際には、

**CGIのPOSTレスポンスの時点でHTTPパーサが失敗していました。**

そのため、

HTML5 KVM以前にIPMIとのHTTP通信自体が成立していません。

## 解決策1: socat（おすすめ）

HTTPを一切解釈せず、

```plaintext
Tailscale
    ↓ TLS終端
TCP
    ↓
socat
    ↓ TLS
IPMI
```

という構成にします。

例えば

```bash
socat \
  TCP4-LISTEN:8082,bind=127.0.0.1,reuseaddr,fork \
  OPENSSL:x.x.x.x:443,verify=0
```

そして

```bash
tailscale serve \
  --service=svc:x11dpi-ipmi \
  --tls-terminated-tcp=443 \
  tcp://127.0.0.1:8082
```

とします。

この構成では、

* CGI
* Cookie
* WebSocket
* HTML5 KVM

すべて正常に動作しました。

HTTPヘッダーを解析しないため、ATEN独自の実装にも影響されません。

## 解決策2: Nginx stream

NginxでもHTTPではなくstreamモジュールを使えば同様です。

```nginx
stream {
    server {
        listen 127.0.0.1:8082;

        proxy_ssl on;
        proxy_ssl_verify off;

        proxy_pass x.x.x.x:443;
    }
}
```

streamはTCPプロキシなので、

HTTPヘッダーを解析しません。

そのためATEN IPMIとの相性問題を回避できます。

## Nginx HTTP Proxyはおすすめしない

一般的には

```nginx
proxy_pass https://x.x.x.x;
```

というHTTPリバースプロキシを書きたくなります。

しかしATEN IPMIでは、

```plaintext
upstream sent invalid header
```

になるケースがあります。

これは

* Hostヘッダー
* Origin
* WebSocket

以前に、

HTTPレスポンス自体がNginxに拒否されるためです。

バッファサイズや `proxy_buffering off` などでは改善しません。

## なぜsocatが動くのか

socatはHTTPを理解していません。

単なるTCP中継です。

そのため、

ATEN IPMIが多少変わったHTTPレスポンスを返しても、

そのままブラウザへ転送します。

結果として、

ブラウザは問題なく処理できます。

## 結論

ATENベースのSupermicro X11 IPMIをTailscale Servicesで公開する場合のおすすめ順位は次の通りです。

1. **socat + `tailscale serve --tls-terminated-tcp`**
2. **Nginx stream + `tailscale serve --tls-terminated-tcp`**
3. **Nginx HTTP Proxy（非推奨）**

一般的なWebアプリではHTTPリバースプロキシが第一選択ですが、ATEN IPMIでは事情が異なります。

HTTPを解析するプロキシよりも、TCPレベルでそのまま中継する構成の方が安定して動作します。

もし `upstream sent invalid header` や `502 Bad Gateway` に遭遇した場合は、WebSocketやTailscaleを疑う前に、HTTPプロキシを経由していないか確認してみてください。
