---
pubDatetime: 2026-05-03T16:18:20+09:00
title: "bcacheのdiscardとcache_modeを確認・設定する方法"
description: "bcacheは、SSDなどの高速なデバイスをキャッシュとして使い、HDDなどの低速なストレージの読み書きを補助する仕組みです。 ここでは、bcacheで使われる主な設定として、discard と cachemode の確認方法と変更方法を簡単に説明します。 discard設定の確認と有効化 disc…"
---

bcacheは、SSDなどの高速なデバイスをキャッシュとして使い、HDDなどの低速なストレージの読み書きを補助する仕組みです。

ここでは、bcacheで使われる主な設定として、`discard` と `cache_mode` の確認方法と変更方法を簡単に説明します。

## discard設定の確認と有効化

`discard` は、不要になったブロックをストレージ側に通知するための設定です。SSDをキャッシュデバイスとして使っている場合、TRIMに関連する動作として理解できます。

現在の設定は、次のコマンドで確認できます。

```bash
cat /sys/block/sda/bcache/discard
```

`discard` を有効にするには、次のように `1` を書き込みます。

```bash
echo 1 | sudo tee /sys/block/sda/bcache/discard
```

この設定により、bcacheが不要になった領域をストレージに通知できるようになります。

## cache_modeの確認と変更

`cache_mode` は、bcacheがデータを書き込むときの動作を決める設定です。

現在のキャッシュモードは、次のコマンドで確認できます。

```bash
cat /sys/block/bcache0/bcache/cache_mode
```

キャッシュモードを `writethrough` に変更するには、次のコマンドを実行します。

```bash
echo writethrough | sudo tee /sys/block/bcache0/bcache/cache_mode
```

`writethrough` は、データをキャッシュと元のストレージの両方へ書き込む方式です。キャッシュだけに依存しないため、比較的安全な設定として使いやすいモードです。

## 使うときの注意点

これらの設定は、`/sys` 以下のbcache設定ファイルへ直接値を書き込む操作です。そのため、対象デバイス名を間違えないことが重要です。

たとえば、環境によっては `sda` や `bcache0` ではなく、別の名前になっている場合があります。実行前に、対象のデバイスが正しいか確認してください。

## まとめ

`discard` は不要な領域の通知を有効にする設定で、SSDを使う環境で役立ちます。

`cache_mode` はbcacheの書き込み方式を決める設定で、`writethrough` は安全性を重視した使いやすいモードです。

bcacheを運用するときは、現在の設定を確認してから、必要に応じて変更するのが基本です。
