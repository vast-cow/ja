---
pubDatetime: 2026-07-28T14:54:05+09:00
title: "システムレスキュー用途で使えるLinux ISOを比較してみた"
description: "Anacondaを使わずにRocky Linuxをインストールしたい、rootfsをバックアップして別のファイルシステムへ移行したい、chrootしてdnfやaptを使いたいを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Linuxサーバーの保守やディスク移行では、インストールメディアとは別に「レスキュー用のLive ISO」を持っておくと便利です。

今回は代表的なレスキューISOを比較し、以下のような用途に適したものを整理しました。

* 障害時のデータ救出
* パーティション編集
* バックアップ・リストア
* `chroot` によるシステム修復
* Anacondaを使わないRocky Linuxの手動インストール
* ファイルシステム変更（ext4→XFSなど）

---

# 主なレスキューISO

| ISO              | ベース                 | 特徴              |
| ---------------- | ------------------- | --------------- |
| SystemRescue     | Arch Linux（旧Gentoo） | 最も多機能な総合レスキュー環境 |
| Rescuezilla      | Ubuntu              | GUIによるバックアップ・復元 |
| Clonezilla Live  | Debian（Ubuntu版あり）   | ディスクイメージ・クローン専用 |
| GParted Live     | Debian              | パーティション編集に特化    |
| Rescatux         | Debian              | ブート修復・GRUB修復    |
| Finnix           | Debian              | 軽量CLIレスキュー      |
| KNOPPIX          | Debian              | 汎用Live Linux    |
| Ultimate Boot CD | ツール集                | ハードウェア診断向け      |

---

# ベースディストリビューション

意外にも、多くのレスキューISOはDebian系です。

| ベース    | ISO                                             |
| ------ | ----------------------------------------------- |
| Arch   | SystemRescue                                    |
| Debian | Clonezilla、GParted Live、Rescatux、Finnix、KNOPPIX |
| Ubuntu | Rescuezilla、Clonezilla Ubuntu版                  |

SystemRescueは以前はGentooベースでしたが、現在はArch Linuxベースになっています。そのため比較的新しいカーネルやデバイスドライバを利用できます。

---

# ISOサイズ比較

現行版のおおよそのISOサイズを比較すると次のようになります。

| ISO              |      サイズ |
| ---------------- | -------: |
| Clonezilla Live  |  約573 MB |
| Finnix           | 約577 MiB |
| GParted Live     |  約649 MB |
| Rescatux         |  約690 MB |
| Ultimate Boot CD |  約803 MB |
| Rescuezilla      |    約1 GB |
| SystemRescue     |  約1.3 GB |
| KNOPPIX DVD      |  約4.4 GB |

サイズだけを見るとClonezillaやFinnixは非常にコンパクトですが、その分用途は限定されます。

---

# ユースケース別比較

## 1. Anacondaを使わずにRocky Linuxをインストールしたい

例えば以下のような手順です。

```text
partition
↓
mkfs.xfs
↓
mount
↓
dnf --installroot=/mnt
↓
fstab作成
↓
grub-install
↓
dracut
```

この用途では必要になります。

* GPT/MBR操作
* mkfs
* mount
* chroot
* grub-install
* efibootmgr
* rsync/tar

これらを一通り備えているのが **SystemRescue** です。

---

## 2. rootfsをバックアップして別のファイルシステムへ移行したい

例えば

* ext4 → XFS
* ext4 → Btrfs
* Btrfs → XFS

のような移行です。

SystemRescueには

* tar
* rsync
* fsarchiver
* ddrescue
* xfsprogs
* btrfs-progs
* e2fsprogs

などが収録されているため、

```bash
tar cpf backup.tar /
```

や

```bash
rsync -aAXH
```

でバックアップし、新しいファイルシステムへ復元できます。

---

## 3. chrootしてdnfやaptを使いたい

例えば

```bash
mount /dev/sda2 /mnt

mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

chroot /mnt
```

その後は通常通り

```bash
dnf update
grub2-install
dracut
passwd
```

などを実行できます。

この用途は

* SystemRescue
* Finnix

のどちらでも快適です。

---

# 各ISOの向き・不向き

| 用途               | おすすめ             |
| ---------------- | ---------------- |
| 総合レスキュー          | SystemRescue     |
| ディスクコピー          | Clonezilla       |
| GUIバックアップ        | Rescuezilla      |
| パーティション編集        | GParted Live     |
| CLI中心のサーバーメンテナンス | Finnix           |
| ハードウェア診断         | Ultimate Boot CD |

ClonezillaやRescuezillaはイメージバックアップには非常に優秀ですが、システム構築や`chroot`作業にはあまり向きません。

逆にSystemRescueは「Live Linux」として使えるため、通常のLinux環境に近い感覚で保守作業を行えます。

---

# 結論

私の用途では、

* Rocky Linuxを手動でインストールする
* rootfsを別ファイルシステムへ移行する
* `chroot`して`dnf`や`grub-install`を実行する

といった作業が多いため、**SystemRescueが最も適している**という結論になりました。

一方、

* とにかく軽量なCLI環境が欲しいなら **Finnix**
* SSD換装やディスクコピーなら **Clonezilla**
* GUIでバックアップしたいなら **Rescuezilla**

というように用途ごとに使い分けるのが良さそうです。

---

# 参考資料

* SystemRescue 公式: [SystemRescue](https://www.system-rescue.org/?utm_source=chatgpt.com)
* Clonezilla 公式: [Clonezilla Live](https://clonezilla.org/?utm_source=chatgpt.com)
* Rescuezilla 公式: [Rescuezilla](https://rescuezilla.com/?utm_source=chatgpt.com)
* GParted Live 公式: [GParted Live](https://gparted.org/livecd.php?utm_source=chatgpt.com)
* Finnix 公式: [Finnix](https://www.finnix.org/?utm_source=chatgpt.com)
* Rescatux / Rescapp: [Rescatux](https://www.supergrubdisk.org/rescatux/?utm_source=chatgpt.com)
* Ultimate Boot CD 公式: [Ultimate Boot CD](https://www.ultimatebootcd.com/?utm_source=chatgpt.com)
* KNOPPIX: [KNOPPIX](https://www.knopper.net/knoppix/?utm_source=chatgpt.com)
