---
pubDatetime: 2026-07-03T17:32:35+09:00
title: "インターネット未接続環境でWindowsのモバイルホットスポットを使う方法"
description: "目的、何をしているか、主な使い方を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## 目的

このプログラムは、Windowsが行うインターネット接続確認を常に成功させるためのものです。

Windowsでは、ネットワークに接続したときに「本当にインターネットへ接続できるか」を確認する仕組みがあります。この確認に失敗すると、Windowsはそのネットワークを「インターネットなし」と判断します。

その結果、インターネットに接続していない環境では、モバイルホットスポットなど一部の機能が使いにくくなる場合があります。

このプログラムを使うと、Windowsの接続確認をローカルPC上で成功させることができます。これにより、実際には外部インターネットに接続していない環境でも、Windows側では「インターネットに接続できる」と判断され、モバイルホットスポットを利用しやすくなります。

## 何をしているか

このプログラムは、主に次の2つのことを行います。

### 1. Windowsの接続確認先をlocalhostに変更する

Windowsのインターネット接続確認では、通常 `www.msftconnecttest.com` が使われます。

このプログラムは、Windowsのレジストリ設定を一時的に変更し、接続確認先を `localhost` にします。

`localhost` は自分自身のPCを指します。そのため、外部のインターネットにアクセスしなくても、同じPC上で接続確認に応答できます。

### 2. 接続確認用のWebサーバーを起動する

プログラムはPC上で簡単なWebサーバーを起動します。

Windowsが `/connecttest.txt` にアクセスすると、プログラムは次の内容を返します。

```text
Microsoft Connect Test
```

これはWindowsが期待する応答です。そのため、Windowsはインターネット接続確認に成功したと判断します。

## 主な使い方

## 1. 管理者権限で実行する

このプログラムはWindowsのレジストリを変更するため、管理者権限が必要です。

コマンドプロンプトやPowerShellを管理者として起動し、Pythonスクリプトを実行します。

## 2. ポート80を使える状態にする

このプログラムは、Webサーバーを `port 80` で起動します。

すでに別のWebサーバーやアプリケーションがポート80を使用している場合は、起動に失敗する可能性があります。その場合は、ポート80を使用しているアプリケーションを停止してから実行します。

## 3. モバイルホットスポットを有効にする

プログラムを起動した状態で、Windowsのモバイルホットスポットを有効にします。

Windowsがインターネット接続確認に成功した状態になるため、インターネットがないネットワーク環境でもモバイルホットスポットを利用できる可能性があります。

## 終了時の動作

プログラムを終了すると、変更したレジストリ値を元に戻そうとします。

起動時には接続確認先を次の値に変更します。

```text
localhost
```

終了時には元の確認先として、次の値へ戻します。

```text
www.msftconnecttest.com
```

通常終了だけでなく、Ctrl+Cなどで終了した場合にも、可能な限り元に戻す処理が実行されます。

## 注意点

このプログラムは、Windowsに対して「インターネット接続確認が成功した」と見せるためのものです。

実際に外部のWebサイトやオンラインサービスへ接続できるようになるわけではありません。あくまで、Windowsの判定を通過させるための仕組みです。

また、レジストリを変更するため、実行には注意が必要です。プログラムが異常終了した場合などには、設定が元に戻らない可能性があります。その場合は、`ActiveWebProbeHost` の値を手動で `www.msftconnecttest.com` に戻す必要があります。

## まとめ

このプログラムを使うと、Windowsのインターネット疎通テストをローカルPC上で成功させることができます。

その結果、実際にはインターネットに接続していない環境でも、Windowsに「インターネット接続あり」と認識させることができます。

主な用途は、インターネットがない環境でモバイルホットスポットを使いたい場合です。技術的には単純な仕組みですが、Windowsの接続判定に依存する機能を利用したい場面で役立ちます。


```
#!/usr/bin/env python3
import atexit
import signal
import sys
import winreg
from aiohttp import web

CONNECT_TEST_BODY = b"Microsoft Connect Test"
NOT_FOUND_BODY = b"Page not found"

REDIRECT_LOCATION = "http://go.microsoft.com/fwlink/?LinkID=219472&clcid=0x409"

REG_PATH = r"SYSTEM\CurrentControlSet\Services\NlaSvc\Parameters\Internet"
REG_VALUE_NAME = "ActiveWebProbeHost"

STARTUP_PROBE_HOST = "localhost"
SHUTDOWN_PROBE_HOST = "www.msftconnecttest.com"


def set_active_web_probe_host(value: str) -> None:
    with winreg.OpenKey(
        winreg.HKEY_LOCAL_MACHINE,
        REG_PATH,
        0,
        winreg.KEY_SET_VALUE,
    ) as key:
        winreg.SetValueEx(
            key,
            REG_VALUE_NAME,
            0,
            winreg.REG_SZ,
            value,
        )


def restore_active_web_probe_host() -> None:
    try:
        set_active_web_probe_host(SHUTDOWN_PROBE_HOST)
        print(f"Restored {REG_VALUE_NAME} = {SHUTDOWN_PROBE_HOST}")
    except Exception as e:
        print(f"Failed to restore registry value: {e}", file=sys.stderr)


def setup_registry() -> None:
    set_active_web_probe_host(STARTUP_PROBE_HOST)
    print(f"Set {REG_VALUE_NAME} = {STARTUP_PROBE_HOST}")

    # 通常終了時の復元
    atexit.register(restore_active_web_probe_host)


def handle_exit_signal(signum, frame) -> None:
    restore_active_web_probe_host()
    sys.exit(0)


async def connecttest(request: web.Request) -> web.Response:
    return web.Response(
        status=200,
        body=CONNECT_TEST_BODY,
        headers={
            "Content-Type": "text/plain",
            "Cache-Control": "max-age=30, must-revalidate",
            "Connection": "keep-alive",
        },
    )


async def redirect(request: web.Request) -> web.Response:
    return web.Response(
        status=302,
        reason="Moved Temporarily",
        body=b"",
        headers={
            "Server": "AkamaiGHost",
            "Content-Length": "0",
            "Location": REDIRECT_LOCATION,
            "Connection": "keep-alive",
        },
    )


async def not_found(request: web.Request) -> web.Response:
    return web.Response(
        status=404,
        body=NOT_FOUND_BODY,
        headers={
            "Content-Type": "text/plain",
            "Connection": "keep-alive",
        },
    )


def create_app() -> web.Application:
    app = web.Application()

    app.router.add_get("/connecttest.txt", connecttest)
    app.router.add_get("/redirect", redirect)

    # 他のすべての path
    app.router.add_route("*", "/{tail:.*}", not_found)

    return app


if __name__ == "__main__":
    setup_registry()

    # Ctrl+C / kill などで可能な限り復元
    signal.signal(signal.SIGINT, handle_exit_signal)
    signal.signal(signal.SIGTERM, handle_exit_signal)

    web.run_app(
        create_app(),
        host="0.0.0.0",
        port=80,
        access_log=None,
    )
```
