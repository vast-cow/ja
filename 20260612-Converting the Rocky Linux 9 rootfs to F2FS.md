---
pubDatetime: 2026-06-12T12:19:37+09:00
title: "Rocky Linux 9 の rootfs を F2FS に変更する手順"
description: "Rocky Linux 9 の root filesystem を F2FS に変更する場合、通常のインストーラだけで完結させるのは難しい。現実的には、いったん ext4 や XFS などで Rocky 9 を通常インストールし、その後 Live USB から rootfs をバックアップ、F2FS…"
---

Rocky Linux 9 の root filesystem を F2FS に変更する場合、通常のインストーラだけで完結させるのは難しい。現実的には、いったん ext4 や XFS などで Rocky 9 を通常インストールし、その後 Live USB から rootfs をバックアップ、F2FS で再作成、リストアする流れになる。

この記事では、次の前提で手順を整理する。

* `/boot` は rootfs とは別パーティションにする
* `/boot` は ext4 または XFS のままにする
* `/` のみを F2FS に変更する
* Rocky 9 標準カーネルではなく ELRepo の `kernel-lt` を使う
* `mkfs.f2fs` 時に元の rootfs と同じ UUID を指定する
* 起動しなかった場合は Ubuntu Live USB から chroot して復旧する

ELRepo は Enterprise Linux 向けに kernel や filesystem driver などを提供するリポジトリで、`kernel-lt` は長期サポート系カーネルとして提供されている。Rocky/RHEL 9 系で F2FS root を試す場合、標準カーネルではなく ELRepo kernel を前提にするのが実用的である。([Elrepo][1])

---

## 1. 基本方針

最終的な構成は次のようにする。

```text
/boot      ext4 または XFS
/boot/efi  vfat
/          f2fs
```

`/boot` を F2FS にしない理由は単純で、GRUB や early boot 周りの不確定要素を減らすためである。rootfs だけを F2FS にすれば、GRUB は通常通り `/boot` 上の kernel と initramfs を読み、initramfs 内の F2FS driver で rootfs を mount する構成になる。

---

## 2. Rocky 9 を通常インストールする

まず Rocky Linux 9 を普通にインストールする。

この時点では rootfs は ext4、XFS などでよい。

パーティション例：

```text
/dev/nvme0n1p1  /boot/efi
/dev/nvme0n1p2  /boot
/dev/nvme0n1p3  /
```

重要なのは、**`/boot` を `/` と分けておくこと**である。

---

## 3. SELinux を permissive にしておく

F2FS への変換後、SELinux label や extended attributes の扱いで詰まる可能性がある。そのため、作業前に SELinux は permissive にしておく。

```bash
sudo vi /etc/selinux/config
```

```ini
SELINUX=permissive
```

変更後に再起動する。

```bash
sudo reboot
```

起動後、確認する。

```bash
getenforce
```

```text
Permissive
```

---

## 4. ELRepo の kernel-lt を導入する

Rocky 9 の標準カーネルではなく、ELRepo の `kernel-lt` を使う。

```bash
sudo dnf install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm
sudo dnf --enablerepo=elrepo-kernel install kernel-lt
```

Secure Boot を有効にしている場合は注意が必要である。ELRepo の説明では、`kernel-ml` と `kernel-lt` は Secure Boot key で署名されていないとされている。Secure Boot 環境では無効化するか、自前で署名・MOK 登録する必要がある。([Elrepo][2])

再起動する。

```bash
sudo reboot
```

GRUB で ELRepo の `kernel-lt` を選び、起動後に確認する。

```bash
uname -r
```

`elrepo` を含む kernel version で起動していることを確認する。

さらに F2FS module が使えるか確認する。

```bash
sudo modprobe f2fs
grep f2fs /proc/filesystems
```

ここで失敗するなら、F2FS rootfs 化に進むべきではない。

---

## 5. dracut に F2FS driver を入れる設定を作る

rootfs が F2FS になると、initramfs の段階で F2FS driver が必要になる。そのため `dracut` 設定を永続化する。

```bash
sudo tee /etc/dracut.conf.d/90-f2fs-root.conf >/dev/null <<'EOF'
add_drivers+=" f2fs "
EOF
```

`dracut.conf` では `add_drivers` に指定した kernel module を initramfs に追加できる。module 名は `.ko` なしで指定する。([Ubuntu Manpages][3])

必要に応じて `force_drivers` を使ってもよい。

```bash
sudo tee /etc/dracut.conf.d/90-f2fs-root.conf >/dev/null <<'EOF'
force_drivers+=" f2fs "
EOF
```

`force_drivers` は `add_drivers` と同様に module を含めるが、より早い段階で `modprobe` されることを期待する設定である。([Ubuntu Manpages][3])

initramfs を作り直す。

```bash
sudo dracut -f /boot/initramfs-$(uname -r).img $(uname -r)
```

中身を確認する。

```bash
lsinitrd /boot/initramfs-$(uname -r).img | grep f2fs
```

`f2fs.ko` が含まれていればよい。

---

## 6. Live USB で起動する

ここからは Ubuntu Live USB などで起動して作業する。

まずディスク構成を確認する。

```bash
sudo lsblk -f
sudo blkid
```

ここでは例として次のように仮定する。

```text
/dev/nvme0n1p1  /boot/efi
/dev/nvme0n1p2  /boot
/dev/nvme0n1p3  /
```

rootfs は `/dev/nvme0n1p3` とする。

---

## 7. rootfs の UUID を記録する

今回の方針では、`mkfs.f2fs` の際に元の rootfs と同じ UUID を指定する。

まず現在の UUID を保存する。

```bash
OLD_UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p3)
echo "$OLD_UUID"
```

この UUID を F2FS 作成時にも使う。

---

## 8. 変換前 rootfs をバックアップする

変換前 rootfs を mount する。

```bash
sudo mkdir -p /mnt/src
sudo mount /dev/nvme0n1p3 /mnt/src
```

バックアップ先を mount しておく。例：

```bash
sudo mkdir -p /mnt/backup
sudo mount /dev/sdX1 /mnt/backup
```

`tar` でバックアップする場合、SELinux context、xattrs、ACL、所有者情報を落とさないようにする。

```bash
sudo tar \
  --one-file-system \
  --xattrs --xattrs-include='*' \
  --acls \
  --selinux \
  --numeric-owner \
  --sparse \
  -cpf /mnt/backup/rocky-root.tar \
  -C /mnt/src .
```

このバックアップは rootfs の論理バックアップである。

さらに保険として `partclone` などで変換前パーティションを丸ごと退避しておくのもよい。`partclone` バックアップは「失敗したら元の ext4/XFS rootfs に戻す」ための保険になる。

バックアップ後、unmount する。

```bash
sudo umount /mnt/src
```

---

## 9. rootfs を F2FS で作成する

ここで rootfs パーティションを F2FS に作り直す。

`mkfs.f2fs` が UUID 指定に対応している版なら、元の UUID を指定する。

```bash
sudo mkfs.f2fs -f -l rocky-root -U "$OLD_UUID" /dev/nvme0n1p3
```

作成後、UUID が同じになっているか確認する。

```bash
sudo blkid /dev/nvme0n1p3
```

期待する状態：

```text
UUID="<OLD_UUID>" TYPE="f2fs"
```

`mkfs.f2fs -U` が使えるかは `f2fs-tools` のバージョンに依存する可能性があるため、事前に確認しておく。

```bash
mkfs.f2fs -h
```

同じ UUID にできるなら、BLS/GRUB の `root=UUID=...` を大きく変更せずに済む。ただし、**fstab の filesystem type は必ず F2FS に変更する必要がある**。

---

## 10. rootfs をリストアする

F2FS 化した rootfs を mount する。

```bash
sudo mkdir -p /mnt/dst
sudo mount -t f2fs /dev/nvme0n1p3 /mnt/dst
```

tar からリストアする。

```bash
sudo tar \
  --xattrs --xattrs-include='*' \
  --acls \
  --selinux \
  --numeric-owner \
  --same-owner \
  --sparse \
  -xpf /mnt/backup/rocky-root.tar \
  -C /mnt/dst
```

同期する。

```bash
sudo sync
```

---

## 11. `/etc/fstab` を修正する

リストア先の fstab を編集する。

```bash
sudo vi /mnt/dst/etc/fstab
```

rootfs の行を F2FS に変更する。

例：

```fstab
UUID=<OLD_UUID>  /  f2fs  defaults,noatime  0  0
```

今回、UUID は元と同じにしているので、UUID自体は変えなくてよい。

ただし、ここは必ず変更する。

```text
ext4 または xfs → f2fs
```

復旧性を優先するなら、まず最後の pass number は `0` にしておく。

```fstab
UUID=<OLD_UUID>  /  f2fs  defaults,noatime  0  0
```

`fsck.f2fs` を initramfs に確実に入れて運用できるなら、後で見直せばよい。

---

## 12. `/boot` を mount して BLS entry を確認する

`/boot` を別パーティションにしているので、必ず mount する。

```bash
sudo mount /dev/nvme0n1p2 /mnt/dst/boot
```

UEFI 環境なら EFI System Partition も mount する。

```bash
sudo mount /dev/nvme0n1p1 /mnt/dst/boot/efi
```

BLS entry を確認する。

```bash
grep -R "root=" /mnt/dst/boot/loader/entries/*.conf
```

UUID を元と同じにしているので、`root=UUID=...` は基本的にそのままでよい。

ただし、`rootfstype=f2fs` は追加しておくと明示的でよい。

chroot して `grubby` で追加する。

```bash
sudo mount --rbind /dev /mnt/dst/dev
sudo mount --make-rslave /mnt/dst/dev
sudo mount -t proc proc /mnt/dst/proc
sudo mount -t sysfs sysfs /mnt/dst/sys
sudo mount -t tmpfs tmpfs /mnt/dst/run

sudo chroot /mnt/dst /bin/bash
```

chroot 内で実行する。

```bash
grubby --update-kernel=ALL --args="rootfstype=f2fs"
```

RHEL 9 系では `grubby` で boot entry の kernel command line を変更できる。Red Hat のドキュメントでも、すべての boot entry に kernel parameter を追加する方法として `grubby --update-kernel=ALL --args=...` が示されている。([Red Hat Documentation][4])

確認する。

```bash
grep -R "root=" /boot/loader/entries/*.conf
grep -R "rootfstype" /boot/loader/entries/*.conf
```

---

## 13. chroot 内で initramfs を再生成する

Live USB から chroot している場合、**`uname -r` は使わない**。

`uname -r` は Live USB 側の kernel version を返すためである。

Rocky 側の kernel version は `/lib/modules` で確認する。

```bash
ls /lib/modules
```

ELRepo kernel を対象にする。

```bash
KVER=$(ls /lib/modules | grep elrepo | tail -n 1)
echo "$KVER"
```

復旧性を優先して、まずは広めの initramfs を作る。

```bash
dracut -f \
  --no-hostonly \
  --force-drivers "f2fs" \
  "/boot/initramfs-${KVER}.img" \
  "${KVER}"
```

dracut は initramfs image を作成するツールで、installed system から必要な tools/files を集めて initramfs を構築する。([GitHub][5])

確認する。

```bash
lsinitrd "/boot/initramfs-${KVER}.img" | grep f2fs
```

`f2fs.ko` が入っていることを確認する。

---

## 14. SELinux relabel を仕込む

tar で xattrs と SELinux context を保存していても、初回は relabel を仕込んでおく方が安全である。

chroot 内で：

```bash
touch /.autorelabel
```

初回起動後に relabel が走る。

---

## 15. chroot を抜けて再起動する

chroot から抜ける。

```bash
exit
```

unmount する。

```bash
sudo sync
sudo umount -R /mnt/dst
```

再起動する。

```bash
sudo reboot
```

GRUB では ELRepo の `kernel-lt` を選ぶ。

---

## 16. 起動後の確認

起動できたら、rootfs が F2FS になっているか確認する。

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /
```

期待する出力：

```text
/dev/nvme0n1p3 f2fs ...
```

kernel も確認する。

```bash
uname -r
```

ELRepo kernel であることを確認する。

```bash
grep f2fs /proc/filesystems
```

SELinux はまだ permissive のままでよい。

```bash
getenforce
```

問題なければ、必要に応じて enforcing に戻す。

```bash
sudo vi /etc/selinux/config
```

```ini
SELINUX=enforcing
```

再起動後に確認する。

```bash
getenforce
```

---

# 起動しなかった場合の復旧手順

F2FS 化後に起動しなかった場合は、Ubuntu Live USB から chroot して修復する。

## 1. Live USB で起動して確認

```bash
sudo lsblk -f
sudo blkid
```

rootfs が F2FS として認識されるか確認する。

```bash
sudo file -s /dev/nvme0n1p3
```

---

## 2. rootfs、boot、EFI を mount する

```bash
sudo mkdir -p /mnt/sysroot
sudo mount -t f2fs /dev/nvme0n1p3 /mnt/sysroot
```

`/boot` を mount する。

```bash
sudo mount /dev/nvme0n1p2 /mnt/sysroot/boot
```

UEFI の場合：

```bash
sudo mount /dev/nvme0n1p1 /mnt/sysroot/boot/efi
```

ここが重要である。

`/boot` を mount せずに `dracut` すると、実際の `/boot` ではなく rootfs 内の空ディレクトリに initramfs を作ってしまう。

---

## 3. chroot する

```bash
sudo mount --rbind /dev /mnt/sysroot/dev
sudo mount --make-rslave /mnt/sysroot/dev
sudo mount -t proc proc /mnt/sysroot/proc
sudo mount -t sysfs sysfs /mnt/sysroot/sys
sudo mount -t tmpfs tmpfs /mnt/sysroot/run
```

```bash
sudo chroot /mnt/sysroot /bin/bash
```

---

## 4. fstab を確認する

```bash
cat /etc/fstab
```

rootfs が F2FS になっていることを確認する。

```fstab
UUID=<OLD_UUID>  /  f2fs  defaults,noatime  0  0
```

---

## 5. BLS entry を確認する

```bash
grep -R "root=" /boot/loader/entries/*.conf
grep -R "rootfstype" /boot/loader/entries/*.conf
```

UUID を元と同じにしている場合、`root=UUID=...` は基本的にそのままでよい。

ただし `rootfstype=f2fs` がなければ追加する。

```bash
grubby --update-kernel=ALL --args="rootfstype=f2fs"
```

---

## 6. 対象 kernel を明示して dracut を再実行する

ここでも `uname -r` は使わない。

```bash
ls /lib/modules
```

ELRepo kernel を選ぶ。

```bash
KVER=$(ls /lib/modules | grep elrepo | tail -n 1)
echo "$KVER"
```

initramfs を作り直す。

```bash
dracut -f \
  --no-hostonly \
  --force-drivers "f2fs" \
  "/boot/initramfs-${KVER}.img" \
  "${KVER}"
```

確認する。

```bash
lsinitrd "/boot/initramfs-${KVER}.img" | grep f2fs
```

---

## 7. SELinux relabel を仕込む

```bash
touch /.autorelabel
```

---

## 8. chroot を抜けて再起動する

```bash
exit
```

```bash
sudo sync
sudo umount -R /mnt/sysroot
sudo reboot
```

GRUB で ELRepo `kernel-lt` を選ぶ。

---

# よくある失敗原因

| 症状                               | 原因                                                    |
| -------------------------------- | ----------------------------------------------------- |
| `unknown filesystem type 'f2fs'` | initramfs に F2FS module が入っていない                       |
| `VFS: Unable to mount root fs`   | rootfs 指定、initramfs、kernel module のいずれかが不正            |
| `UUID=... does not exist`        | UUID が変わった、または指定ミス                                    |
| emergency shell に落ちる             | fstab、root指定、initramfs、SELinux のいずれか                  |
| 標準kernelでは起動しない                  | Rocky 9 標準kernel側にF2FS root用moduleがない、またはinitramfsにない |
| chrootでdracutしたのに直らない            | `/boot` をmountせずにinitramfsを作った                        |
| `dracut -f $(uname -r)` で失敗      | Live USB側のkernel versionを使っている                        |

---

# まとめ

Rocky Linux 9 の rootfs を F2FS に変更する手順は、次の形にすると安全性が高い。

```text
1. /boot を別パーティションにして Rocky 9 を通常インストール
2. SELinux を permissive にする
3. ELRepo kernel-lt を導入
4. kernel-lt で起動して f2fs module を確認
5. dracut に f2fs を追加する設定を永続化
6. Ubuntu Live USB で起動
7. rootfs を tar で xattrs/ACL/SELinux込みでバックアップ
8. 元の rootfs UUID を控える
9. mkfs.f2fs 時に元の UUID を指定する
10. rootfs をリストア
11. /etc/fstab の rootfs type を f2fs に変更
12. BLS entry に rootfstype=f2fs を追加
13. chroot 内で対象 kernel を明示して dracut 再生成
14. /.autorelabel を作成
15. ELRepo kernel-lt で起動
```

特に重要なのは次の4点である。

```text
- /boot は必ず分ける
- initramfs に f2fs.ko を入れる
- mkfs.f2fs 後も UUID が元と同じであることを確認する
- 起動しなければ Live USB + chroot + dracut で復旧する
```

UUIDを元と同じにしておけば、`root=UUID=...` の変更を避けられるため、作業はかなり単純になる。ただし、`/etc/fstab` の filesystem type と initramfs の F2FS module だけは必ず正しく設定する必要がある。

[1]: https://elrepo.org/wiki/doku.php?id=kernel-lt "kernel-lt [ELRepo Wiki]"
[2]: https://elrepo.org/wiki/doku.php?id=secureboot "secureboot [ELRepo Wiki]"
[3]: https://manpages.ubuntu.com/manpages/xenial/man5/dracut.conf.5.html "dracut.conf - configuration file(s) for dracut"
[4]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/configuring-kernel-command-line-parameters_managing-monitoring-and-updating-the-kernel "Chapter 4. Configuring kernel command-line parameters"
[5]: https://github.com/dracutdevs/dracut "dracut the event driven initramfs infrastructure"
