---
pubDatetime: 2026-06-11T20:38:33+09:00
title: "GrubbyでELRepoカーネル設定"
description: "ELRepo kernel を確認、最新の ELRepo kernel をデフォルトに設定、設定確認を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

以下で設定できます。`kernel-ml` / `kernel-lt` のどちらでも、ELRepo カーネルは通常 `/boot/vmlinuz-*-elrepo.*` として見えます。

## 1. ELRepo kernel を確認

```bash
sudo grubby --info=ALL | awk 'BEGIN{RS=""} /elrepo/ {print $0 "\n"}'
```

または簡易確認:

```bash
ls -1 /boot/vmlinuz-*elrepo*
```

## 2. 最新の ELRepo kernel をデフォルトに設定

```bash
ELREPO_KERNEL=$(ls -1 /boot/vmlinuz-*elrepo* | sort -V | tail -n 1)

sudo grubby --set-default "$ELREPO_KERNEL"
```

Red Hat 公式ドキュメントでも、`grubby --set-default /boot/vmlinuz-...` によりデフォルトカーネルを永続変更する手順が示されています。([Red Hat Documentation][1])

## 3. 設定確認

```bash
grubby --default-kernel
grubby --default-index
```

期待値は、`--default-kernel` が ELRepo カーネルを指すことです。

例:

```bash
/boot/vmlinuz-6.x.x-x.el9.elrepo.x86_64
```

ELRepo では `kernel-ml` が mainline stable 系、`kernel-lt` が long-term support 系として提供されています。([elrepo.org][2])

## 4. 再起動後に確認

```bash
sudo reboot
```

起動後:

```bash
uname -r
```

`elrepo` を含むカーネルなら成功です。

```bash
6.x.x-x.el9.elrepo.x86_64
```

## `kernel-ml` だけを選びたい場合

```bash
ELREPO_KERNEL=$(rpm -q kernel-ml --qf '/boot/vmlinuz-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort -V | tail -n 1)

sudo grubby --set-default "$ELREPO_KERNEL"
grubby --default-kernel
```

## `kernel-lt` だけを選びたい場合

```bash
ELREPO_KERNEL=$(rpm -q kernel-lt --qf '/boot/vmlinuz-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort -V | tail -n 1)

sudo grubby --set-default "$ELREPO_KERNEL"
grubby --default-kernel
```

[1]: https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/setting-a-kernel-as-default_assembly_the-linux-kernel "1.7. カーネルのデフォルトとしての設定"
[2]: https://elrepo.org/wiki/doku.php?id=kernel-ml "Kernel-ml"
