---
pubDatetime: 2026-04-16T21:54:25+09:00
title: "Singularity（Apptainer）で non-root / 非特権ポートの Samba サーバーを立てて `$HOME` を共有する"
description: "コンテナ環境（Singularity / Apptainer）上で、root 権限なし・非特権ポートで Samba を動かし、ユーザーのホームディレクトリを共有する構成をまとめる。 HPC や制限環境でも成立する実用構成。 --- 要件と設計方針 smbd は non-root で実行 ポートは 1…"
---

コンテナ環境（Singularity / Apptainer）上で、**root 権限なし**・**非特権ポート**で Samba を動かし、ユーザーのホームディレクトリを共有する構成をまとめる。
HPC や制限環境でも成立する実用構成。

---

## 要件と設計方針

* `smbd` は **non-root** で実行
* ポートは **1445 など（1024以上）**
* `nmbd` は使わず **SMB over TCP のみ**
* 共有は `[homes]` を利用（ユーザーごとの `$HOME`）
* 書き込みが必要なパスはすべて **ユーザーの home 配下へ退避**
* Singularity の **bind mount** を前提

---

## 1. コンテナイメージの作成

### 定義ファイル `samba.def`

```def
Bootstrap: docker
From: debian

%post
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y samba smbclient
    apt-get clean
    rm -rf /var/lib/apt/lists/*
```

### ビルド

```bash
singularity build --fakeroot samba.sif samba.def
```

---

## 2. ホスト側のディレクトリ準備

```bash
mkdir -p ~/samba/{etc,log,lock,run,cache,private}
chmod 700 ~/samba/private
mkdir -p ~/samba/run/ncalrpc
```

---

## 3. `smb.conf`

`~/samba/etc/smb.conf`

```ini
[global]
   server role = standalone server
   workgroup = WORKGROUP
   netbios name = MYSMB

   security = user
   map to guest = never

   smb ports = 1445
   disable netbios = yes

   lock directory = /hosthome/USER/samba/lock
   pid directory = /hosthome/USER/samba/run
   state directory = /hosthome/USER/samba/cache
   cache directory = /hosthome/USER/samba/cache
   private dir = /hosthome/USER/samba/private

   log file = /hosthome/USER/samba/log/log.%m
   max log size = 1000

   ncalrpc dir = /hosthome/USER/samba/run/ncalrpc

   load printers = no
   printing = bsd
   printcap name = /dev/null
   disable spoolss = yes

[homes]
   browseable = no
   read only = no
   valid users = %S
   create mask = 0600
   directory mask = 0700
```

ユーザー名を置換:

```bash
sed -i "s|USER|$USER|g" ~/samba/etc/smb.conf
```

---

## 4. bind mount 設定

ホストの `$HOME` をコンテナ内 `/hosthome/$USER` にマウント:

```bash
export SMB_BIND="$HOME:/hosthome/$USER"
```

---

## 5. Samba パスワード設定（fakeroot）

```bash
singularity exec --fakeroot \
  --bind "$SMB_BIND" \
  samba.sif \
  smbpasswd -c /hosthome/$USER/samba/etc/smb.conf -a root
```

※今回の構成では **root ユーザーに対してパスワード設定**

---

## 6. 起動（重要ポイントあり）

### ログ用ディレクトリを bind

```bash
mkdir -p ~/samba/varlog
```

### 起動コマンド

```bash
singularity exec --fakeroot \
  --bind "$SMB_BIND" \
  --bind "$HOME/samba/varlog:/var/log/samba" \
  samba.sif \
  smbd --foreground --no-process-group --debug-stdout \
    -s /hosthome/$USER/samba/etc/smb.conf \
    -p 1445
```

---

## 7. 動作確認

```bash
ss -ltnp | grep 1445
```

```bash
smbclient -L //127.0.0.1 -p 1445 -U root
```

---

## ハマりポイント（重要）

### 1. `/var/log/samba` が read-only

```
Unable to open new log file '/var/log/samba/log.smbd'
```

→ 対策:

```bash
--bind "$HOME/samba/varlog:/var/log/samba"
```

---

### 2. `/run/samba/ncalrpc` が作れない

```
Failed to create pipe directory /run/samba/ncalrpc
```

→ 対策:

* `smb.conf` に追加

```ini
ncalrpc dir = /hosthome/$USER/samba/run/ncalrpc
```

* ディレクトリ作成

```bash
mkdir -p ~/samba/run/ncalrpc
```

---

### 3. `-S` オプションは使えない

```
Invalid option -S
```

→ 正しい起動方法:

```bash
smbd --foreground --no-process-group
```

---

### 4. 非特権ポート必須

* 445 / 139 は使用不可（root が必要）
* → `1445` などを使用

---

## まとめ

この構成で実現できること:

* root 権限なしで Samba 起動
* Singularity コンテナ内で完結
* `$HOME` をそのまま共有
* HPC / 制限環境でも動作

重要なのは以下の3点:

1. **すべての writable path を `$HOME` 配下に逃がす**
2. **コンテナ内のデフォルトパス（/var/log, /run）を潰す**
3. **ポートは必ず非特権にする**

---

## 補足

この構成は「Samba をコンテナに閉じ込める」というよりは、

> **Samba の実体はユーザー領域に置き、コンテナは実行環境として使う**

という考え方に近い。
