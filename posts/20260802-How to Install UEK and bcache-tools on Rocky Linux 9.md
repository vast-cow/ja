---
title: "Rocky Linux 9へUEKを導入する手順（Oracleリポジトリ最小利用）"
description: "方針、UEK R7かR8か、事前確認を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-08-02T06:11:33.033Z
---

## 方針

Rocky Linux 9のBaseOS/AppStreamはそのまま維持し、Oracle Linux側からは次だけ取得します。

* UEK本体: `kernel-uek`と、その依存サブパッケージ
* `bcache-tools`
* Oracle Linuxの通常ユーザーランド、`oraclelinux-release-el9`、Oracle版`glibc`などは導入しない

Oracle Linux yum serverはRHEL互換ディストリビューションからの利用方法を公式に案内していますが、Rocky Linux上でUEKを動かす構成自体はOracle/Rocky双方の正式サポート対象とは考えないでください。([Oracle Linux Yum Server][1])

以下は **x86_64** 前提です。

## UEK R7かR8か

2026年8月時点では、Oracle Linux 9向けに以下があります。

| 系列     | カーネル系列 | 選択基準                  |
| ------ | -----: | --------------------- |
| UEK R7 |   5.15 | Rocky 9との混成構成では比較的保守的 |
| UEK R8 |   6.12 | 新しいハードウェア・機能を優先       |

Oracleの現行UEK R8リポジトリには、6.12系の`kernel-uek`、`kernel-uek-core`、各種modulesパッケージが収録されています。([Oracle Linux Yum Server][2])
UEK R7は5.15系です。([Oracle Linux Yum Server][3])

ここでは、混成リスクを少し抑えるため **UEK R7** を例にします。R8へ変更する場合はURL中の`UEKR7`を`UEKR8`へ置き換えます。

---

## 1. 事前確認

```bash
cat /etc/rocky-release
uname -m
findmnt /boot
findmnt /boot/efi 2>/dev/null || true
mokutil --sb-state 2>/dev/null || true
```

既存Rockyカーネルは削除しません。UEKが起動できない場合の復旧用です。

作業前に、Rocky側を通常の状態へ更新します。

```bash
sudo dnf upgrade --refresh
sudo reboot
```

再起動後:

```bash
uname -r
```

---

## 2. Oracle Linux 9署名鍵を登録

Oracle公式のOL9用鍵を配置します。

```bash
sudo curl -fsSL \
  https://yum.oracle.com/RPM-GPG-KEY-oracle-ol9 \
  -o /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
```

フィンガープリントを確認します。

```bash
gpg --show-keys --with-fingerprint \
  /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
```

少なくとも、Oracle公式掲載の次のフィンガープリントと一致することを確認します。

```text
3E6D 826D 3FBA B389 C2F3 8E34 BC4D 06A0 8D8B 756F
9822 3175 9C74 6706 5D0C E9B2 A7DD 0708 8B4E FBE6
```

Oracleが公開しているOL9鍵の取得先とフィンガープリントです。([Oracle Linux Yum Server][4])

RPMデータベースにも登録します。

```bash
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
```

---

## 3. 最小限に制限したOracleリポジトリを作成

`oraclelinux-release-el9`はインストールせず、自前のrepoファイルを2エントリだけ作ります。

```bash
sudo tee /etc/yum.repos.d/oracle-uek-minimal.repo >/dev/null <<'EOF'
[oracle-uek-r7-minimal]
name=Oracle Linux 9 UEK R7 - restricted
baseurl=https://yum.oracle.com/repo/OracleLinux/OL9/UEKR7/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
includepkgs=kernel-uek,kernel-uek-core,kernel-uek-modules,kernel-uek-modules-extra
metadata_expire=6h
skip_if_unavailable=0

[oracle-baseos-bcache-minimal]
name=Oracle Linux 9 BaseOS - bcache-tools only
baseurl=https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
includepkgs=bcache-tools
metadata_expire=6h
skip_if_unavailable=0
EOF
```

重要なのは次の2点です。

* `enabled=0`: 通常の`dnf upgrade`ではOracleリポジトリを使わない
* `includepkgs=`: 明示したパッケージ以外をOracleから取得できないようにする

Oracle Linux 9のBaseOS URLは、OracleがRHEL互換環境向けに案内しているものと同じです。([Oracle Linux Yum Server][1])

### UEK R8を使う場合

R7の代わりに、UEKエントリを次のようにします。

```ini
[oracle-uek-r8-minimal]
name=Oracle Linux 9 UEK R8 - restricted
baseurl=https://yum.oracle.com/repo/OracleLinux/OL9/UEKR8/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
includepkgs=kernel-uek,kernel-uek-core,kernel-uek-modules-core,kernel-uek-modules,kernel-uek-modules-extra
metadata_expire=6h
skip_if_unavailable=0
```

R8ではパッケージ分割がR7と異なり、`kernel-uek-modules-core`が存在します。([Oracle Linux Yum Server][2])

---

## 4. Oracleから見えるパッケージを確認

まずインストールせず、候補だけ確認します。

```bash
sudo dnf clean metadata

sudo dnf \
  --disablerepo='oracle-*' \
  --enablerepo=oracle-uek-r7-minimal \
  repoquery --available 'kernel-uek*'
```

`bcache-tools`も確認します。

```bash
sudo dnf \
  --disablerepo='oracle-*' \
  --enablerepo=oracle-baseos-bcache-minimal \
  repoquery --available --info bcache-tools
```

Oracle Linux 9では`bcache-tools`がOracleによるBaseOS追加パッケージとして提供されています。([Oracle Docs][5])

### 取得元を確認

```bash
sudo dnf repoquery \
  --available \
  --qf '%{name}-%{evr}.%{arch} <- %{repoid}' \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

---

## 5. トランザクションを事前確認

最初は`--assumeno`を付けます。

```bash
sudo dnf install --assumeno \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

確認すべき点:

* Oracleから入るものが`kernel-uek*`と`bcache-tools`だけ
* `glibc`、`systemd`、`dracut`、`grub2`などがOracle版へ置換されない
* RockyのBaseOS/AppStreamパッケージが削除されない
* `--allowerasing`を要求されない

想定外のOracleパッケージが表示された場合は中止します。

より厳密に確認するには:

```bash
sudo dnf install --assumeno -v \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

---

## 6. UEKとbcache-toolsをインストール

事前確認に問題がなければ実行します。

```bash
sudo dnf install \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

`kernel-uek`メタパッケージが、対応する`kernel-uek-core`とmodulesを依存関係として導入します。UEK R7リポジトリにはこれらが同一バージョンで収録されています。([Oracle Linux Yum Server][3])

確認:

```bash
rpm -qa | grep -E '^(kernel-uek|bcache-tools)' | sort
```

ベンダーも確認します。

```bash
rpm -q \
  --qf '%{NAME} %{VERSION}-%{RELEASE} | %{VENDOR}\n' \
  kernel-uek bcache-tools
```

Oracle由来パッケージ全体の確認:

```bash
rpm -qa \
  --qf '%{NAME} %{VERSION}-%{RELEASE} | %{VENDOR}\n' |
grep -i oracle |
sort
```

ここで、意図しないOracle版ユーザーランドがないことを確認します。

---

## 7. initramfsとbcacheモジュールを確認

UEKのインストールで通常はinitramfsが生成されます。

インストール済みUEK一覧:

```bash
rpm -q kernel-uek-core
ls -lh /boot/vmlinuz-*uek /boot/initramfs-*uek.img
```

UEKカーネル内にbcacheモジュールがあるか確認します。

```bash
UEK_VER="$(rpm -q --qf '%{VERSION}-%{RELEASE}.%{ARCH}\n' \
  kernel-uek-core | sort -V | tail -1)"

echo "$UEK_VER"
modinfo -k "$UEK_VER" bcache
```

`modinfo`が情報を返せば、そのUEKにbcacheモジュールがあります。

initramfsへ明示的に入れる場合:

```bash
sudo dracut --force \
  --add-drivers bcache \
  "/boot/initramfs-${UEK_VER}.img" \
  "$UEK_VER"
```

ただし、ルートファイルシステムをbcache上に置かないのであれば、通常は起動時initramfsへの強制追加は不要です。起動後に`modprobe bcache`できます。

---

## 8. GRUBへ登録されているか確認

```bash
sudo grubby --info=ALL |
grep -E '^(index|kernel|title)='
```

UEKエントリを特定します。

```bash
sudo grubby --info=ALL |
grep -B2 -A3 'el9uek'
```

最初からデフォルトにはせず、まず一度だけGRUBメニューからUEKを選択して起動するのが安全です。

GRUBメニューを表示しやすくする場合:

```bash
sudo grub2-editenv - unset menu_auto_hide
```

---

## 9. UEKでテスト起動

再起動後、GRUBから`el9uek`を含むエントリを選択します。

```bash
sudo reboot
```

起動後:

```bash
uname -r
```

想定例:

```text
5.15.0-...el9uek.x86_64
```

bcacheを確認:

```bash
sudo modprobe bcache
lsmod | grep '^bcache'
```

ツール確認:

```bash
make-bcache --version
bcache-super-show --help
```

カーネルログも確認します。

```bash
sudo journalctl -b -k -p warning
sudo dmesg -T | grep -iE 'bcache|error|failed|firmware'
```

ネットワーク、ストレージ、コンソール、SELinuxも確認します。

```bash
ip addr
findmnt
getenforce
systemctl --failed
```

---

## 10. 問題なければUEKをデフォルトにする

現在起動中のUEKをデフォルトにするなら:

```bash
sudo grubby --set-default "/boot/vmlinuz-$(uname -r)"
sudo grubby --default-kernel
```

または、インストール済み最新UEKを指定:

```bash
UEK_KERNEL="$(ls -1 /boot/vmlinuz-*el9uek* | sort -V | tail -1)"
sudo grubby --set-default "$UEK_KERNEL"
sudo grubby --default-kernel
```

Rocky標準カーネルは残してください。

```bash
rpm -q kernel-core
ls -1 /boot/vmlinuz-*
```

---

## 11. 更新方法

Oracleリポジトリは無効のままなので、通常の更新はRockyだけが対象です。

```bash
sudo dnf upgrade
```

UEKと`bcache-tools`を更新するときだけ明示的に有効化します。

```bash
sudo dnf upgrade \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  'kernel-uek*' bcache-tools
```

より慎重にするなら、毎回事前確認します。

```bash
sudo dnf upgrade --assumeno \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  'kernel-uek*' bcache-tools
```

### `dnf upgrade --enablerepo=oracle-...`だけは避ける

次のような無指定更新は実行しない方が安全です。

```bash
# 非推奨
sudo dnf upgrade --enablerepo=oracle-uek-r7-minimal
```

現状は`includepkgs`で制限されていますが、更新対象を明示する方が事故を防げます。

---

## 12. Secure Bootの注意

Secure Bootが有効な場合、最大の問題はRPM署名ではなく、**Rockyのshim/ファームウェアがOracleのカーネル署名を信頼するか**です。

確認:

```bash
mokutil --sb-state
```

`SecureBoot enabled`の場合、UEKが次のようなエラーで起動できない可能性があります。

```text
Verification failed
Security Violation
Bad shim signature
```

この混成構成では、まず以下のどちらかで検証するのが現実的です。

1. 検証段階ではSecure Bootを無効化する
2. Oracleカーネルの署名証明書をMOKへ適切に登録する

後者は証明書の取得・検証・MOK登録が必要で、単純にOracle RPM GPG鍵を登録するだけでは解決しません。RPMパッケージ署名鍵とUEFI Secure Boot用カーネル署名証明書は別物です。

---

## 13. bcacheを使い始める前の最低限の注意

`make-bcache`は対象デバイスの既存データを破壊します。デバイス名を十分に確認してください。

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE,FSTYPE,MOUNTPOINTS
```

例として:

```bash
# SSDキャッシュ側
sudo make-bcache --cache /dev/nvme0n1p1

# HDDバックエンド側
sudo make-bcache --bdev /dev/sdb
```

この操作は指定デバイスへbcacheスーパーブロックを書き込みます。実デバイス名へ読み替える前に、バックアップとコンソールアクセスを確保してください。

Linuxカーネル文書によると、bcacheはwritethroughとwritebackをサポートし、writebackは既定で無効です。([Linuxカーネルドキュメント][6])

初期検証では、書き込み損失リスクを抑えるためwritethroughのまま使う方が妥当です。

---

## ロールバック

UEKで起動できない場合は、GRUBからRocky標準カーネルを選択します。

Rockyカーネルをデフォルトへ戻す:

```bash
ROCKY_KERNEL="$(ls -1 /boot/vmlinuz-*el9_* 2>/dev/null |
  grep -v el9uek |
  sort -V |
  tail -1)"

sudo grubby --set-default "$ROCKY_KERNEL"
sudo grubby --default-kernel
```

UEKを削除:

```bash
sudo dnf remove 'kernel-uek*'
```

`bcache-tools`も不要なら:

```bash
sudo dnf remove bcache-tools
```

repoファイルを無効化または削除:

```bash
sudo mv \
  /etc/yum.repos.d/oracle-uek-minimal.repo \
  /etc/yum.repos.d/oracle-uek-minimal.repo.disabled
```

## 推奨する最終構成

* Rocky BaseOS/AppStream: 常時有効
* Oracle UEK repo: `enabled=0`
* Oracle BaseOS repo: `enabled=0`
* Oracle側の`includepkgs`:

  * `kernel-uek`
  * `kernel-uek-core`
  * UEK系列に必要なmodulesパッケージ
  * `bcache-tools`
* Rocky標準カーネル: 最低1世代を常に保持
* UEK更新: パッケージ名を指定して手動実行
* Secure Boot: 事前検証必須
* DKMS/kmod製品: UEK ABI対応を個別確認

この方法なら、Oracle由来パッケージをカーネル一式と`bcache-tools`へ実質的に限定し、RockyユーザーランドのOracle化を防げます。

[1]: https://yum.oracle.com/getting-started.html "Oracle Linux Yum Server"
[2]: https://yum.oracle.com/repo/OracleLinux/OL9/UEKR8/x86_64/index.html "Oracle Linux 9 (x86_64) UEK Release 8 | Oracle, Software. Hardware. Complete."
[3]: https://yum.oracle.com/repo/OracleLinux/OL9/UEKR7/x86_64/index.html "Oracle Linux 9 (x86_64) UEK Release 7 | Oracle, Software. Hardware. Complete."
[4]: https://yum.oracle.com/faq.html "Frequently Asked Questions about the Oracle Linux Yum Server | Oracle, Software. Hardware. Complete."
[5]: https://docs.oracle.com/en/operating-systems/oracle-linux/9/relnotes9.1/ol-PackageChangesfromtheUpstreamRelease.html "6 Package Changes From the Upstream Release"
[6]: https://docs.kernel.org/admin-guide/bcache.html "A block layer cache (bcache)"
