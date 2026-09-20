---
pubDatetime: 2026-06-25T19:00:49+09:00
title: "Weston 内で Waydroid を実行する"
description: "目的、使用方法、Waydroid の停止を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Waydroid は、Wayland バックエンドで Weston を起動し、Waydroid セッションを開始してから、Waydroid の完全なユーザーインターフェースを開くことで、Weston セッション内で起動できます。

このセットアップは、メインのデスクトップセッション上で直接実行するのではなく、制御された Wayland 環境で Waydroid を実行したい場合に便利です。

## 目的

このセットアップの目的は、軽量な Wayland コンポジターである Weston 内で Waydroid を実行することです。Weston は、Waydroid が Android インターフェースを表示できる独立したグラフィカル環境を提供します。

これは、テスト、キオスク形式の環境、または Waydroid を専用のディスプレイセッションで実行したいシステムで役立ちます。

## 使用方法

### 1. Weston を起動する

まず、Wayland バックエンドと kiosk shell を使用して Weston を起動します。

```bash
weston --backend=wayland-backend.so --shell=kiosk-shell.so --idle-time=0 
```

これにより、Weston がネストされた Wayland コンポジターとして起動します。kiosk shell はシンプルな全画面形式の環境を提供し、`--idle-time=0` によって Weston がアイドル状態に入るのを防ぎます。

### 2. Waydroid セッションを開始する

次に、Weston の Wayland ディスプレイを使用して Waydroid セッションを開始します。

```bash
WAYLAND_DISPLAY=wayland-1 waydroid session start 
```

`WAYLAND_DISPLAY=wayland-1` の設定は、Weston によって作成された Wayland ディスプレイを使用するよう Waydroid に指示します。

### 3. Waydroid の完全な UI を表示する

Waydroid セッションが開始されたら、Waydroid の完全なインターフェースを開きます。

```bash
waydroid show-full-ui 
```

これにより、Android インターフェースが Weston 環境内に表示されます。

## Waydroid の停止

Waydroid の使用が終わったら、まず Android 環境をシャットダウンしてから Waydroid セッションを停止する方が適切です。

### 1. Waydroid 内の Android をシャットダウンする

次のコマンドを実行します。

```bash
waydroid shell -- svc power shutdown 
```

これにより、Waydroid 内で実行されている Android システムにシャットダウンコマンドが送信されます。これは Android デバイスの電源を切る操作に似ています。

### 2. Waydroid セッションを停止する

Android がシャットダウンされた後、Waydroid セッションを停止します。

```bash
waydroid session stop 
```

これにより、ホストシステム上のアクティブな Waydroid セッションが停止します。

## 注意事項

通常、コマンドは順番に実行する必要があります。Waydroid はインターフェースを表示するために Weston の Wayland ディスプレイを必要とするため、Waydroid セッションを開始する前に Weston が実行されている必要があります。

Waydroid を停止する際は、まず `waydroid shell -- svc power shutdown` で Android をシャットダウンし、その後 `waydroid session stop` でセッションを停止します。

システムによっては、Wayland ディスプレイ名が `wayland-1` と異なる場合があります。コマンドが動作しない場合は、利用可能な Wayland ディスプレイソケットを確認し、それに応じて `WAYLAND_DISPLAY` を調整してください。

## まとめ

Weston 内で Waydroid を実行することは、専用の Wayland 環境で Android を起動する簡単な方法です。まず Weston を起動し、Waydroid が Weston のディスプレイを使用するよう指定してから、Waydroid の完全な UI を開きます。

クリーンに停止するには、まず Waydroid 内の Android をシャットダウンし、その後 Waydroid セッションを停止します。
