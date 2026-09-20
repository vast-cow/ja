---
pubDatetime: 2026-06-21T15:21:58+09:00
title: "bcache で cache device が無い backing device を強制的に認識させる方法"
description: "bcache を使っていた環境で cache device が故障・消失・未接続になると、backing device だけでは通常 /dev/bcache0 などの bcache デバイスが自動作成されないことがあります。 この場合、backing device 側を bcache に登録したうえ…"
---

bcache を使っていた環境で cache device が故障・消失・未接続になると、backing device だけでは通常 `/dev/bcache0` などの bcache デバイスが自動作成されないことがあります。

この場合、backing device 側を bcache に登録したうえで、`running` に `1` を書き込むことで、cache device が無い状態でも backing device を強制的に起動できます。

## 前提

この記事では、backing device が次のデバイスであるとします。

```bash
/dev/sdd4
```

環境に合わせて `/dev/sdd4` の部分は読み替えてください。

## 手順

### 1. bcache カーネルモジュールを読み込む

```bash
sudo modprobe bcache
```

### 2. backing device を bcache に登録する

```bash
echo /dev/sdd4 | sudo tee /sys/fs/bcache/register
```

この操作で、bcache に backing device を認識させます。

### 3. cache device が無くても強制起動する

```bash
echo 1 | sudo tee /sys/class/block/sdd/sdd4/bcache/running
```

これで `/dev/bcache0` などの bcache デバイスが作成されます。

確認します。

```bash
lsblk
ls -l /dev/bcache*
```

作成された bcache デバイスは、通常 `/dev/bcache0`、`/dev/bcache1` のような名前になります。

## mount するデバイス

mount するのは backing device そのものではなく、作成された bcache デバイスです。

例えば `/dev/bcache0` が作成された場合は、次のように mount します。

```bash
sudo mount /dev/bcache0 /mnt
```

`/dev/sdd4` を直接 mount するのではなく、必ず `/dev/bcache0` のような bcache デバイスを使います。

## sysfs のパスについて

partition を backing device として使っている場合、`running` は次のような場所にあります。

```bash
/sys/class/block/sdd/sdd4/bcache/running
```

今回の例では、ディスクが `/dev/sdd`、partition が `/dev/sdd4` なので、このパスになります。

NVMe の場合は、例えば backing device が `/dev/nvme0n1p3` なら、次のようなパスになります。

```bash
/sys/class/block/nvme0n1/nvme0n1p3/bcache/running
```

## まとめ

cache device が無い backing device を bcache として強制認識させる最小手順は以下です。

```bash
sudo modprobe bcache
echo /dev/sdd4 | sudo tee /sys/fs/bcache/register
echo 1 | sudo tee /sys/class/block/sdd/sdd4/bcache/running
```

その後、作成された `/dev/bcache0` などを mount します。

```bash
sudo mount /dev/bcache0 /mnt
```

## 注意点

writeback cache を使っていた場合、cache device 側に dirty data が残っている可能性があります。

その状態で cache device 無しに backing device を強制起動すると、ファイルシステムの不整合やデータ破損が起きる可能性があります。

特に以下のような場合は注意が必要です。

* cache mode が `writeback` だった
* cache device が突然故障した
* shutdown 前に dirty data を flush していない
* backing device と cache device の整合性が分からない

read-only で確認したい場合は、mount 時に `-o ro` を付けます。

```bash
sudo mount -o ro /dev/bcache0 /mnt
```

データ救出目的の場合は、まず read-only で mount し、必要なデータを退避するのが安全です。
