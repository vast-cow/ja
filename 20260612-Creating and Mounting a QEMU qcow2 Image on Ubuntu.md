---
pubDatetime: 2026-06-12T12:16:32+09:00
title: "UbuntuでQemu qcow2のイメージをつくりマウントする"
description: "前提、注意点、必要パッケージを入れるを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## 前提

ここでは「QEMU COW イメージ」を、通常使われる **qcow2** 形式として扱います。QEMU は `raw` と `qcow2` を主要なディスクイメージ形式として扱い、`qcow2` は小さなイメージ、圧縮、複数スナップショットなどに対応する形式です。([QEMU][1])

Ubuntu では `qemu-img` と `qemu-nbd` を使うのが基本です。Ubuntu の manpage でも、`qemu-img` と `qemu-nbd` は `qemu-utils` パッケージに含まれるものとして示されています。([Ubuntu Manpages][2])

---

## 0. 注意点

既存 VM のディスクイメージをマウントする場合、**その VM は停止しておく**のが原則です。QEMU 公式ドキュメントは、実行中の VM や別プロセスが使用中のイメージを `qemu-img` で変更すると、イメージを破壊する可能性があると警告しています。([GitLab][3])

また、信頼できない qcow2 イメージをホスト OS に直接マウントするのは避けてください。QEMU の `qemu-nbd` ドキュメントも、悪意あるゲストイメージがパーティション検出やファイルシステムマウント時のカーネルバグを突く可能性を警告しています。([QEMU][4])

---

## 1. 必要パッケージを入れる

```bash
sudo apt update
sudo apt install -y qemu-utils parted e2fsprogs
```

主な役割は以下です。

| コマンド        | 用途                                    |
| ----------- | ------------------------------------- |
| `qemu-img`  | qcow2 イメージの作成、情報表示、変換、リサイズ            |
| `qemu-nbd`  | qcow2 を `/dev/nbd0` のようなブロックデバイスとして接続 |
| `parted`    | パーティション作成                             |
| `mkfs.ext4` | ext4 ファイルシステム作成                       |

`qemu-nbd` は QEMU ディスクイメージを NBD として公開し、Linux では `/dev/nbdX` ブロックデバイスに接続できます。([Ubuntu Manpages][5])

---

# 方法 A: パーティション付き qcow2 イメージを作成してマウントする

通常の仮想ディスクらしい構成にするなら、この方法が無難です。

## 2. qcow2 イメージを作成する

例として 20 GB の qcow2 ディスクを作ります。

```bash
mkdir -p ~/qemu-images
cd ~/qemu-images

qemu-img create -f qcow2 disk.qcow2 20G
qemu-img info disk.qcow2
```

QEMU のドキュメントでは、`qemu-img create` でディスクイメージを作成でき、サイズには `M` や `G` のサフィックスを使えると説明されています。([QEMU][1])

---

## 3. NBD カーネルモジュールを読み込む

```bash
sudo modprobe nbd max_part=16
```

`max_part=16` は、`/dev/nbd0p1`、`/dev/nbd0p2` のようなパーティションデバイスを扱うための指定です。

---

## 4. qcow2 を `/dev/nbd0` に接続する

```bash
sudo qemu-nbd --connect=/dev/nbd0 --format=qcow2 disk.qcow2
```

短縮形でも同じです。

```bash
sudo qemu-nbd -c /dev/nbd0 -f qcow2 disk.qcow2
```

QEMU の公式例でも、qcow2 ファイルを `/dev/nbd0` として公開し、パーティションがある場合は `/dev/nbd0p1` などが作られる場合があると説明されています。([QEMU][4])

確認します。

```bash
lsblk /dev/nbd0
```

---

## 5. パーティションを作る

ここでは GPT パーティションテーブルを作り、全体を ext4 用の 1 パーティションにします。

```bash
sudo parted -s /dev/nbd0 mklabel gpt
sudo parted -s -a optimal /dev/nbd0 mkpart primary ext4 1MiB 100%
sudo partprobe /dev/nbd0
```

確認します。

```bash
lsblk /dev/nbd0
```

期待される表示は概ね以下です。

```text
nbd0      43:0    0   20G  0 disk
└─nbd0p1  43:1    0   20G  0 part
```

`/dev/nbd0p1` が出ない場合は、一度切断して再接続します。

```bash
sudo qemu-nbd --disconnect /dev/nbd0
sudo qemu-nbd --connect=/dev/nbd0 --format=qcow2 disk.qcow2
lsblk /dev/nbd0
```

---

## 6. ファイルシステムを作る

```bash
sudo mkfs.ext4 -L qemu-data /dev/nbd0p1
```

既存データがあるデバイスに対して `mkfs` を実行すると中身は消えます。新規作成時だけ実行してください。

---

## 7. マウントする

```bash
sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0p1 /mnt/qcow2
```

確認します。

```bash
df -h /mnt/qcow2
mount | grep /mnt/qcow2
```

ファイルを書き込む例です。

```bash
echo "hello qcow2" | sudo tee /mnt/qcow2/hello.txt
ls -l /mnt/qcow2
```

---

## 8. アンマウントして切断する

作業後は必ずアンマウントしてから NBD を切断します。

```bash
sync
sudo umount /mnt/qcow2
sudo qemu-nbd --disconnect /dev/nbd0
```

短縮形でも切断できます。

```bash
sudo qemu-nbd -d /dev/nbd0
```

必要なら NBD モジュールを外します。

```bash
sudo rmmod nbd
```

---

# 方法 B: パーティションなしで qcow2 全体に ext4 を作る

単純なデータ置き場として使うだけなら、パーティションを作らず、qcow2 全体を ext4 にしても動きます。ただし、通常の VM 用ディスクとしてはパーティション付きの方法 A の方が自然です。

```bash
cd ~/qemu-images

qemu-img create -f qcow2 fs.qcow2 10G

sudo modprobe nbd max_part=8
sudo qemu-nbd -c /dev/nbd0 -f qcow2 fs.qcow2

sudo mkfs.ext4 -L qemu-fs /dev/nbd0

sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0 /mnt/qcow2
```

使い終わったら以下です。

```bash
sync
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

---

# 既存の qcow2 イメージをマウントする

既存 VM ディスクやクラウドイメージを読む場合は、まず読み取り専用で接続するのが安全です。

```bash
sudo modprobe nbd max_part=16
sudo qemu-nbd --read-only --connect=/dev/nbd0 --format=qcow2 existing.qcow2
```

短縮形です。

```bash
sudo qemu-nbd -r -c /dev/nbd0 -f qcow2 existing.qcow2
```

パーティションを確認します。

```bash
lsblk -f /dev/nbd0
sudo fdisk -l /dev/nbd0
```

例えば `/dev/nbd0p1` が ext4 なら、読み取り専用でマウントします。

```bash
sudo mkdir -p /mnt/qcow2
sudo mount -o ro /dev/nbd0p1 /mnt/qcow2
```

作業後です。

```bash
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

書き込みマウントしたい場合は `--read-only` / `-r` を外します。ただし、対象 VM が停止していること、同じイメージを他プロセスが使っていないことを確認してください。

---

# COW 差分イメージを作る場合

「COW イメージ」が **ベースイメージを変更せず、差分だけ別ファイルに保存するイメージ** という意味なら、backing file 付きの qcow2 を作ります。

例:

```bash
qemu-img create -f qcow2 -F qcow2 -b base.qcow2 overlay.qcow2
```

確認します。

```bash
qemu-img info --backing-chain overlay.qcow2
```

この場合、`overlay.qcow2` には `base.qcow2` との差分が記録されます。QEMU の `qemu-img create` ドキュメントでは、backing file を指定した場合、新しいイメージは backing file との差分を記録し、通常 backing file 自体は変更されないと説明されています。([GitLab][3])

差分イメージをマウントする場合は、**base ではなく overlay を接続**します。

```bash
sudo modprobe nbd max_part=16
sudo qemu-nbd -c /dev/nbd0 -f qcow2 overlay.qcow2

lsblk -f /dev/nbd0
sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0p1 /mnt/qcow2
```

作業後です。

```bash
sync
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

ベースイメージの形式が raw の場合は `-F raw` にします。

```bash
qemu-img create -f qcow2 -F raw -b base.raw overlay.qcow2
```

---

## 最小コマンドまとめ

新規 qcow2 を作って ext4 パーティションをマウントする最小例です。

```bash
sudo apt update
sudo apt install -y qemu-utils parted e2fsprogs

qemu-img create -f qcow2 disk.qcow2 20G

sudo modprobe nbd max_part=16
sudo qemu-nbd -c /dev/nbd0 -f qcow2 disk.qcow2

sudo parted -s /dev/nbd0 mklabel gpt
sudo parted -s -a optimal /dev/nbd0 mkpart primary ext4 1MiB 100%
sudo partprobe /dev/nbd0

sudo mkfs.ext4 /dev/nbd0p1

sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0p1 /mnt/qcow2

# 作業
echo test | sudo tee /mnt/qcow2/test.txt

# 後片付け
sync
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

[1]: https://www.qemu.org/docs/master/system/images.html "Disk Images — QEMU  documentation"
[2]: https://manpages.ubuntu.com/manpages/noble/man1/qemu-img.1.html "Ubuntu Manpage: qemu-img - QEMU disk image utility"
[3]: https://qemu-project.gitlab.io/qemu/tools/qemu-img.html "QEMU disk image utility — QEMU  documentation"
[4]: https://www.qemu.org/docs/master/tools/qemu-nbd.html "QEMU Disk Network Block Device Server — QEMU  documentation"
[5]: https://manpages.ubuntu.com/manpages/focal/man8/qemu-nbd.8.html "Ubuntu Manpage: qemu-nbd - QEMU Disk Network Block Device Server"
