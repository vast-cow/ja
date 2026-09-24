---
pubDatetime: 2026-07-28T15:48:17+09:00
title: "SquashFS化されたLive Linuxの改造手順"
description: "作業環境の準備、ISOを展開する、ISOレベルのファイルだけ変更する場合を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

SystemRescueのようなLive Linuxは、概ね次の構造です。

```text
ISO9660
├── EFI/、boot/、syslinux/、grub/   ← ブートローダー
├── vmlinuz                         ← カーネル
├── initramfs                       ← 初期RAMディスク
└── airootfs.sfs / filesystem.squashfs
    └── 実際のrootfs
```

SquashFSは読み取り専用なので、基本的な処理は次の流れになります。

```text
ISOを展開
  ↓
SquashFSを展開
  ↓
rootfsを編集またはchroot
  ↓
SquashFSを再生成
  ↓
ISO内のSquashFSを交換
  ↓
ブート可能なISOとして再生成
  ↓
BIOS・UEFIでテスト
```

ただし、**SystemRescueでは最初から`airootfs.sfs`を直接作り直すのではなく、次の優先順位で方法を選ぶ**のが安全です。

1. `sysrescue.d`のYAML設定
2. SRM（SystemRescueModule）によるオーバーレイ
3. `airootfs.sfs`の直接再構築
4. SystemRescueソースからの完全ビルド

SystemRescue公式も、ISOの変更には`sysrescue-customize`を推奨しています。SRMはSquashFS形式の追加レイヤーで、同じパスのファイルはベースrootfsよりSRM側が優先されます。([SystemRescue][1])

---

## 1. 作業環境の準備

Linux上で作業するのが簡単です。Debian／Ubuntu系なら次を導入します。

```bash
sudo apt update
sudo apt install squashfs-tools xorriso rsync file
```

テスト用にQEMUも入れておくと便利です。

```bash
sudo apt install qemu-system-x86 ovmf
```

SystemRescue公式のカスタマイズスクリプトも、主な依存関係として`xorriso`と`squashfs-tools`を要求します。WSL上でも実行できます。([SystemRescue][1])

作業ディレクトリを用意します。

```bash
mkdir -p ~/work/systemrescue
cd ~/work/systemrescue

cp /path/to/systemrescue.iso original.iso
```

最低でも、元ISOの数倍の空き容量を確保してください。SystemRescue自身で再構築する場合、公式はCopy-on-Write領域としてISOサイズのおよそ3倍が必要になる可能性を指摘しています。([SystemRescue][1])

---

# 方法A：SystemRescue公式の`sysrescue-customize`を使う

SystemRescueでは、この方法を第一選択にするのが適切です。

公式ページから`sysrescue-customize`を取得して、実行権限を付けます。

```bash
chmod +x sysrescue-customize
sudo install -m 755 sysrescue-customize /usr/local/bin/
```

## ISOを展開する

```bash
sysrescue-customize \
  --unpack \
  --source="$PWD/original.iso" \
  --dest="$PWD/iso-tree"
```

展開後は、例えば次のような構成になります。

```bash
find iso-tree -maxdepth 3 -type f | sort | less
```

SquashFSの場所はバージョンによって固定と決めつけず、検索します。

```bash
find iso-tree -type f \
  \( -name '*.sfs' -o -name '*.squashfs' -o -name '*.sqfs' \) \
  -print
```

SystemRescueでは通常、`airootfs.sfs`が対象です。

## ISOレベルのファイルだけ変更する場合

ブート設定、YAML設定、autorunスクリプトなど、rootfs内部でなくてもよいものは、そのままISOツリーへ追加します。

例：

```bash
sudo mkdir -p iso-tree/sysrescue.d

sudo tee iso-tree/sysrescue.d/500-local.yaml >/dev/null <<'EOF'
sysconfig:
  keyboard: jp
EOF
```

実際に利用可能なYAMLキーはバージョンごとの公式設定仕様に合わせてください。`sysrescue.d`のYAMLは辞書順にマージされ、ブートコマンドラインの設定が最終的に優先されます。([SystemRescue][2])

再構築します。

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-custom.iso"
```

既存ファイルを上書きする場合は次のようにします。

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-custom.iso" \
  --overwrite
```

公式スクリプトは、展開時のISO構造を保持したまま再構築する前提なので、手作業でEl ToritoやUEFIブート設定を再現するより安全です。([SystemRescue][1])

---

# 方法B：SRMでrootfsにファイルを重ねる

設定ファイル、スクリプト、静的バイナリ、小規模な追加パッケージなどは、ベースの`airootfs.sfs`を編集せずSRMに入れる方法が適しています。

SRMもSquashFSですが、起動時にベースrootfs上へOverlayFSで重ねられます。ベースと同じパスのファイルを置くことで置換もできます。([SystemRescue][3])

## SRM用ディレクトリを作成する

ディレクトリ構造は、起動後のrootfsと同じにします。

```bash
mkdir -p srm-root/usr/local/bin
mkdir -p srm-root/etc/systemd/system
mkdir -p srm-root/root
```

例えば独自スクリプトを追加します。

```bash
cat >srm-root/usr/local/bin/local-rescue-tool <<'EOF'
#!/bin/sh
echo "Custom SystemRescue tool"
EOF

chmod 755 srm-root/usr/local/bin/local-rescue-tool
```

設定ファイルを上書きする場合も、同じパスに置きます。

```bash
mkdir -p srm-root/etc
cp my-config.conf srm-root/etc/my-config.conf
```

## SRMをISOに組み込む

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-srm.iso" \
  --srm-dir="$PWD/srm-root"
```

`--srm-dir`を指定すると、スクリプトはそのディレクトリをSRMとして圧縮し、SRMを有効化するための設定もISO側へ追加します。なお、この処理では`--source`で指定したISOツリー自体も変更されます。([SystemRescue][1])

## 所有者・パーミッションを指定する

例えばSSH公開鍵を入れる場合、モードを明示します。

```bash
cat >srm-root/.squashfs-pseudo <<'EOF'
/root/.ssh m 700 root root
/root/.ssh/authorized_keys m 600 root root
EOF
```

秘密鍵をISOへ埋め込むことは原則として避けてください。

## 圧縮オプションを指定する

SRM直下に`.squashfs-options`を作成できます。

```bash
cat >srm-root/.squashfs-options <<'EOF'
-comp zstd
-b 1M
EOF
```

このファイルの内容はシェルとして評価されるため、第三者から受け取ったレシピをそのまま実行してはいけません。公式も任意コマンド実行につながる点を警告しています。([SystemRescue][1])

---

# 方法C：`airootfs.sfs`を直接展開して変更する

ベースrootfsそのものを変更する必要がある場合の手順です。

以下では、すでに`iso-tree`へISOを展開したものとします。

## 1. 対象SquashFSを特定する

```bash
find iso-tree -type f \
  \( -name 'airootfs.sfs' \
     -o -name 'filesystem.squashfs' \
     -o -name '*.sqfs' \) \
  -print
```

変数へ入れます。

```bash
SFS="$(find iso-tree -type f -name 'airootfs.sfs' -print -quit)"

test -n "$SFS" || {
  echo "airootfs.sfsが見つかりません" >&2
  exit 1
}

printf '対象: %s\n' "$SFS"
```

## 2. 元のSquashFS情報を記録する

再生成時は、元の圧縮方式とブロックサイズをできるだけ合わせます。

```bash
unsquashfs -s "$SFS" | tee squashfs-original.txt
```

主に確認する項目は次です。

```text
Compression
Block size
Filesystem size
Xattrs
Fragments
Duplicates
```

SquashFSはgzip、xz、lzo、lz4、zstdなどを利用でき、ブロックサイズや圧縮方式が起動速度・メモリ使用量・ISOサイズに影響します。([Ubuntu Manpages][4])

## 3. SquashFSを展開する

既存ディレクトリがないことを確認します。

```bash
sudo rm -rf rootfs
sudo unsquashfs -d rootfs "$SFS"
```

確認します。

```bash
sudo ls -la rootfs
sudo cat rootfs/etc/os-release
```

ファイル所有者、デバイスノード、拡張属性などを保持するため、展開・再生成は原則としてroot権限で実行します。

---

## 4. 単純なファイル変更

単に設定ファイルやスクリプトを置くだけなら、chrootは不要です。

```bash
sudo install -Dm755 local-rescue-tool \
  rootfs/usr/local/bin/local-rescue-tool
```

設定ファイルを追加します。

```bash
sudo install -Dm644 my-config.conf \
  rootfs/etc/my-config.conf
```

systemdユニットを追加する例：

```bash
sudo install -Dm644 my-service.service \
  rootfs/etc/systemd/system/my-service.service
```

有効化はシンボリックリンクで行えます。

```bash
sudo mkdir -p rootfs/etc/systemd/system/multi-user.target.wants

sudo ln -sfn ../my-service.service \
  rootfs/etc/systemd/system/multi-user.target.wants/my-service.service
```

ただしユニットの`WantedBy=`や実際のtarget構成に合わせてください。

---

## 5. chroot環境を準備する

パッケージの追加、ユーザー作成、コマンド実行などにはchrootを使用します。

### 仮想ファイルシステムをマウントする

```bash
sudo mount --rbind /dev rootfs/dev
sudo mount --make-rslave rootfs/dev

sudo mount -t proc proc rootfs/proc
sudo mount --rbind /sys rootfs/sys
sudo mount --make-rslave rootfs/sys

sudo mount --rbind /run rootfs/run
sudo mount --make-rslave rootfs/run
```

DNS解決が必要なら、`resolv.conf`を一時的に置き換えます。元がシンボリックリンクの場合があるので、先に状態を記録してください。

```bash
sudo ls -l rootfs/etc/resolv.conf
sudo cp -a rootfs/etc/resolv.conf \
  rootfs/etc/resolv.conf.before-chroot 2>/dev/null || true

sudo cp -L /etc/resolv.conf rootfs/etc/resolv.conf
```

### chrootへ入る

```bash
sudo chroot rootfs /bin/bash
```

chroot内部で：

```bash
export HOME=/root
export LC_ALL=C

source /etc/os-release
printf '%s %s\n' "$ID" "$VERSION_ID"
```

---

## 6. パッケージを追加する

### SystemRescue／Arch Linux系

SystemRescueはArch Linuxベースで、追加パッケージには`pacman`を利用できます。([SystemRescue][5])

例えば：

```bash
pacman -Sy --needed tmux
```

ただし、次のような全面アップグレードは避けるべきです。

```bash
# 原則として実行しない
pacman -Syu
```

理由は、通常カーネルやinitramfsがSquashFSの外側にあるためです。

```text
ISO内のvmlinuz             古いまま
ISO内のinitramfs           古いまま
airootfs.sfs内のmodules    新しくなる
airootfs.sfs内のuserspace  新しくなる
```

この状態では、次のような不整合が発生します。

* `/usr/lib/modules/<kernel-version>`が一致しない
* カーネルモジュールがロードできない
* initramfs内のツールとrootfs内ライブラリが一致しない
* glibcやsystemdだけ更新されて起動不能になる
* pacmanのスナップショット設定を壊す

SystemRescue公式も、異なるバージョンで作ったSRMや、コアライブラリを置換するSRMは不安定化や起動不能の原因になると警告しています。([SystemRescue][3])

### Debian／Ubuntu系Live ISO

```bash
apt-get update
apt-get install --no-install-recommends tmux
```

不要データを削除します。

```bash
apt-get clean
rm -rf /var/lib/apt/lists/*
```

こちらも、カーネルパッケージを更新する場合は外側の`vmlinuz`とinitramfsも更新する必要があります。

---

## 7. chroot内の後処理

一時ファイルやログを削除します。

```bash
rm -rf /tmp/*
rm -rf /var/tmp/*
rm -f /root/.bash_history
```

`machine-id`は元イメージの方針を確認します。

```bash
ls -l /etc/machine-id
cat /etc/machine-id
```

元が空ファイルなら空に戻します。

```bash
: >/etc/machine-id
```

終了します。

```bash
exit
```

---

## 8. マウントを解除する

逆順に解除します。

```bash
sudo umount -R rootfs/run 2>/dev/null || true
sudo umount -R rootfs/sys 2>/dev/null || true
sudo umount -R rootfs/proc 2>/dev/null || true
sudo umount -R rootfs/dev 2>/dev/null || true
```

残っていないか確認します。

```bash
findmnt -R "$(realpath rootfs)"
```

何も表示されなければ解除済みです。

`resolv.conf`を一時変更した場合は戻します。

```bash
if sudo test -e rootfs/etc/resolv.conf.before-chroot ||
   sudo test -L rootfs/etc/resolv.conf.before-chroot; then
  sudo rm -f rootfs/etc/resolv.conf
  sudo mv rootfs/etc/resolv.conf.before-chroot \
    rootfs/etc/resolv.conf
fi
```

---

## 9. SquashFSを再生成する

元の`unsquashfs -s`結果に合わせます。

例えば元が次だったとします。

```text
Compression zstd
Block size 1048576
```

再生成：

```bash
sudo rm -f airootfs.sfs.new

sudo mksquashfs rootfs airootfs.sfs.new \
  -noappend \
  -comp zstd \
  -b 1M
```

xzの場合：

```bash
sudo mksquashfs rootfs airootfs.sfs.new \
  -noappend \
  -comp xz \
  -b 1M
```

再生成したイメージを確認します。

```bash
unsquashfs -s airootfs.sfs.new
```

一部を展開して確認します。

```bash
rm -rf verify-root
unsquashfs -d verify-root airootfs.sfs.new \
  usr/local/bin/local-rescue-tool

ls -l verify-root/usr/local/bin/local-rescue-tool
```

## よくある再圧縮時の誤り

### `-all-root`を安易に使う

```bash
mksquashfs rootfs output.sfs -all-root
```

これを使うと、通常ユーザー所有のファイルまで全部root所有になります。Live環境によってはユーザーのホームディレクトリやサービス用ファイルが壊れます。

### 元と異なる圧縮方式を使う

initramfs側のSquashFS実装が対応していない圧縮方式を使うと、マウントできません。特に古いカーネル向けISOでは注意してください。

### 拡張属性を落とす

Linux capabilitiesやSELinux属性を利用している環境では、xattrが失われるとコマンドが正常動作しない可能性があります。`mksquashfs`では通常xattr保存がデフォルトですが、`-no-xattrs`は指定しないでください。

---

## 10. 元のSquashFSと交換する

バックアップを残します。

```bash
sudo mv "$SFS" "${SFS}.original"
sudo install -m 644 airootfs.sfs.new "$SFS"
```

比較します。

```bash
ls -lh "$SFS" "${SFS}.original"
```

---

# ISOを再生成する

## SystemRescueの場合

公式スクリプトを使います。

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-custom-rootfs.iso"
```

これは最も安全な方法です。

---

## 一般的なISOでSquashFSだけ置換する場合

ISO全体をゼロから`mkisofs`で再構成するより、元ISOを`xorriso`で読み込み、対象ファイルだけ差し替える方がブート情報を保持しやすくなります。

ISO内部でのSquashFSのパスを求めます。

```bash
ISO_SFS_PATH="/${SFS#iso-tree/}"

printf '%s\n' "$ISO_SFS_PATH"
```

例えば次のようになります。

```text
/sysresccd/x86_64/airootfs.sfs
```

差し替えて新ISOを生成します。

```bash
rm -f custom.iso

xorriso \
  -indev original.iso \
  -outdev custom.iso \
  -map airootfs.sfs.new "$ISO_SFS_PATH" \
  -boot_image any replay
```

追加ファイルも同時に入れる場合：

```bash
xorriso \
  -indev original.iso \
  -outdev custom.iso \
  -map airootfs.sfs.new "$ISO_SFS_PATH" \
  -map local.cfg /config/local.cfg \
  -boot_image any replay
```

ファイルを削除する例：

```bash
xorriso \
  -indev original.iso \
  -outdev custom.iso \
  -rm /path/in/iso/unneeded-file \
  -map airootfs.sfs.new "$ISO_SFS_PATH" \
  -boot_image any replay
```

複雑なHybrid ISO、GRUB2 MBR、追加パーティション付きISOでは、ディストリビューション固有のビルドツールを優先してください。`xorriso`のboot replayは元ISOのブート設定を再利用する仕組みですが、ブートイメージ自体を交換する場合などにはバージョン依存の問題も報告されています。([Scdbackup][6])

---

# チェックサムの扱い

Live ISOによっては次のような整合性情報を持っています。

```text
md5sum.txt
SHA256SUMS
embedded MD5
copytoram時のchecksum
```

rootfsを変更すると、それらは当然一致しなくなります。

SystemRescueでは過去のリリースからISO内にチェックサムが埋め込まれており、13.01では`copytoram`時の整合性確認用`checksum`ブートオプションも追加されています。公式再構築ツールを使う理由の一つです。([SystemRescue][7])

一般的なISOでは、例えば`md5sum.txt`があるなら再生成します。

```bash
cd iso-tree

find . -type f \
  ! -name md5sum.txt \
  -print0 |
  sort -z |
  xargs -0 md5sum |
  sudo tee md5sum.txt >/dev/null

cd ..
```

ただし、チェックサム形式はディストリビューションごとに違うので、既存ファイルの書式を確認してください。

---

# 動作確認

## 1. ISO構造を確認

```bash
file custom.iso
xorriso -indev custom.iso -toc
xorriso -indev custom.iso -report_el_torito plain
```

## 2. ISO内のSquashFSを確認

一時的に取り出します。

```bash
rm -f verify.sfs

xorriso \
  -osirrox on \
  -indev custom.iso \
  -extract "$ISO_SFS_PATH" verify.sfs
```

確認：

```bash
unsquashfs -s verify.sfs
```

目的のファイルが入っているか確認します。

```bash
rm -rf verify-root
unsquashfs -d verify-root verify.sfs \
  usr/local/bin/local-rescue-tool

ls -l verify-root/usr/local/bin/local-rescue-tool
```

## 3. BIOS起動をQEMUで確認

```bash
qemu-system-x86_64 \
  -m 4096 \
  -smp 2 \
  -cdrom custom.iso \
  -boot d
```

KVMが使える場合：

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -m 4096 \
  -smp 2 \
  -cdrom custom.iso \
  -boot d
```

## 4. UEFI起動を確認

OVMFファイルのパスはディストリビューションにより異なります。

```bash
find /usr/share -iname 'OVMF_CODE*.fd' -o -iname 'OVMF_VARS*.fd'
```

例：

```bash
cp /usr/share/OVMF/OVMF_VARS_4M.fd OVMF_VARS.fd

qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 \
  -smp 2 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd \
  -drive if=pflash,format=raw,file="$PWD/OVMF_VARS.fd" \
  -cdrom custom.iso
```

最低限、次をテストします。

* Legacy BIOSで起動するか
* UEFIで起動するか
* SquashFSがマウントされるか
* systemdが起動完了するか
* ネットワークが使えるか
* 追加したコマンドが実行できるか
* 既存のストレージ・暗号化・RAIDツールが壊れていないか
* `copytoram`でも起動するか

---

# 自動化用レシピ

SystemRescueの`sysrescue-customize --auto`では、次の構成で変更をコード化できます。

```text
recipe/
├── iso_delete/
├── iso_add/
├── iso_patch_and_script/
└── build_into_srm/
```

* `iso_delete`：ISO内のファイルを削除
* `iso_add`：ISOへ追加・上書き
* `iso_patch_and_script`：パッチ適用、任意スクリプト実行
* `build_into_srm`：SRMにするrootfsオーバーレイ

公式スクリプトは、この順序で処理します。スクリプトはISOツリーのルートをカレントディレクトリとして実行されるため、レシピ内で`airootfs.sfs`を展開・再生成することもできます。([SystemRescue][1])

例：

```text
recipe/
├── iso_add/
│   └── sysrescue.d/
│       └── 500-local.yaml
└── build_into_srm/
    └── usr/
        └── local/
            └── bin/
                └── local-rescue-tool
```

実行：

```bash
sysrescue-customize \
  --auto \
  --source="$PWD/original.iso" \
  --dest="$PWD/custom.iso" \
  --recipe-dir="$PWD/recipe" \
  --work-dir="$PWD/auto-work"
```

レシピをGit管理しておけば、新しいSystemRescueへ変更を再適用しやすくなります。公式もautoモードを、異なるバージョンへ同じ変更を適用するための仕組みとして説明しています。([SystemRescue][1])

---

# カーネルやinitramfsも変更する場合

次の変更は、SquashFSだけを再生成する方法には向きません。

* カーネル更新
* カーネルコンフィグ変更
* ドライバー追加
* initramfsフック追加
* GRUB／Syslinuxの大規模変更
* 大量のパッケージ入れ替え
* glibc、systemd、pacmanなど基盤パッケージの更新

この場合は、SystemRescueのソースビルドを利用します。

SystemRescueはArch Linuxベースで、パッチを加えたArchisoによって公式ISOを構築しています。ソース、ビルドツール、ドキュメントも公開されています。([SystemRescue][8])

Archisoプロファイルでは概ね次を管理します。

```text
profile/
├── airootfs/
├── efiboot/
├── syslinux/
├── grub/
├── packages.x86_64
├── pacman.conf
└── profiledef.sh
```

`packages.x86_64`でパッケージを選択し、`airootfs/`でrootfsへ追加するファイルを管理し、`profiledef.sh`でSquashFSの圧縮方式やブートモードを指定します。([GitHub][9])

継続的に独自版を作るなら、既成ISOを毎回分解するより、こちらの方法が適しています。

---

# Secure Bootについて

一般に、署名済みEFIバイナリ、カーネル、Unified Kernel Imageなどを変更すると署名は無効になります。独自鍵で再署名し、ファームウェア側へ鍵を登録する工程が必要です。

SystemRescueについては、2026年1月22日の公式フォーラム管理者回答ではSecure Bootは標準対応していないと説明されています。2026年6月6日公開の13.01変更履歴にもSecure Boot対応追加の記載はないため、標準状態でのSecure Boot起動を前提にしない方がよいでしょう。([System-Rescue][10])

---

# 方法の選択基準

| 変更内容                   | 推奨方法                         |
| ---------------------- | ---------------------------- |
| キーボード、ネットワーク、autorun設定 | `sysrescue.d` YAML           |
| スクリプトや静的バイナリの追加        | SRM                          |
| 少数の追加パッケージ             | SRMまたは`cowpacman2srm`        |
| `/etc`のファイル置換          | SRM                          |
| ベースrootfsからファイルを完全削除   | SquashFS直接再構築                |
| 大量のパッケージ変更             | ソース／Archisoビルド               |
| カーネル・モジュール変更           | ソース／Archisoビルド               |
| initramfs変更            | ソース／Archisoビルド               |
| 一回だけの実験                | SquashFS直接再構築                |
| 新版へ繰り返し適用              | `sysrescue-customize --auto` |

実務上は、**設定はYAML、追加物はSRM、カーネルを含む大改造はソースビルド**、という分離が最も保守しやすい構成です。

[1]: https://www.system-rescue.org/scripts/sysrescue-customize/ "SystemRescue - sysrescue-customize: customize SystemRescue ISO-images"
[2]: https://www.system-rescue.org/manual/Configuring_SystemRescue/ "SystemRescue - Configuring SystemRescue"
[3]: https://www.system-rescue.org/Modules/ "SystemRescue - Custom SystemRescue Modules"
[4]: https://manpages.ubuntu.com/manpages/stonking/man1/unsquashfs.1.html?utm_source=chatgpt.com "unsquashfs - tool to uncompress, extract and list squashfs ..."
[5]: https://www.system-rescue.org/manual/Installing_packages_with_pacman/?utm_source=chatgpt.com "Installing additional software packages with pacman"
[6]: https://scdbackup.sourceforge.net/xorriso_eng.html?utm_source=chatgpt.com "GNU xorriso - GNU Project - Free Software Foundation"
[7]: https://www.system-rescue.org/Changes-x86/ "SystemRescue - ChangeLog"
[8]: https://www.system-rescue.org/manual/Building_SystemRescue_from_Source/ "SystemRescue - Building SystemRescue from Source"
[9]: https://github.com/archlinux/archiso/blob/master/docs/README.profile.rst "archiso/docs/README.profile.rst at master · archlinux/archiso · GitHub"
[10]: https://forums.system-rescue.org/t/cannot-boot-into-systemrescue-with-secure-boot-enabled/42?utm_source=chatgpt.com "Cannot boot into SystemRescue with Secure boot enabled"
