---
pubDatetime: 2026-07-30T18:17:12+09:00
title: "PiKVMのセットアップと基本的な使い方"
description: "最初に理解しておきたい「読み取り専用」の仕組み、ISOファイルにchown kvmd:kvmdは必要か、仮想メディア利用時の注意を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

PiKVMは、管理対象のコンピューターにOSが起動していない状態でも、映像の確認、キーボード・マウス操作、仮想メディアからの起動などを行えるリモートKVMです。

本記事では、SDカードへのイメージ書き込み後から、初期設定、仮想メディアの配置、Tailscaleによるリモートアクセス、管理対象マシンの操作までを、実際に作業する順番に沿って解説します。

なお、画面構成や一部の機能はPiKVMのモデルやKVMDのバージョンによって異なります。本記事の仮想メディアに関する説明は、主にPiKVM V2以降を想定しています。

---

## 最初に理解しておきたい「読み取り専用」の仕組み

PiKVM OSは、突然の電源断によるファイルシステム破損やSDカードの消耗を抑えるため、通常はファイルシステムを読み取り専用で使用します。設定変更が終わったら、必ず読み取り専用へ戻すのが基本です。

ここで注意したいのが、PiKVMには用途の異なる2種類の書き込み切り替えコマンドがあることです。

### 書き込みモードを変更するコマンドの比較

| コマンド                                   | 対象                           | 主な用途                                    | 書き込み可能にする                       | 読み取り専用に戻す                       |
| -------------------------------------- | ---------------------------- | --------------------------------------- | ------------------------------- | ------------------------------- |
| `rw` / `ro`                            | PiKVM OSのルートファイルシステム         | パスワード変更、パッケージのインストール、`/etc`以下の編集、ホスト名変更 | `rw`                            | `ro`                            |
| `kvmd-helper-otgmsd-remount rw` / `ro` | `/var/lib/kvmd/msd`の仮想メディア領域 | ISO・IMGファイルの配置、削除、書き換え                  | `kvmd-helper-otgmsd-remount rw` | `kvmd-helper-otgmsd-remount ro` |

重要なのは、**`rw`を実行しても、仮想メディア領域が書き込み可能になるとは限らない**ことです。反対に、`kvmd-helper-otgmsd-remount rw`を実行しても、`/etc`などのシステム設定は編集できません。

作業対象に応じて使い分けます。

---

# 1. SDカードへイメージを書き込む

まず、使用するPiKVMモデルに対応したOSイメージをSDカードへ書き込みます。

書き込み完了後、PiKVMを起動する前にSDカードの先頭にあるFAT32パーティションを開きます。このパーティションには、初回起動時の設定に使用する`pikvm.txt`があります。

---

# 2. 初回起動前にWi-Fiを設定する

有線LANを使用できない環境では、初回起動前に`pikvm.txt`へWi-Fi情報を書いておくと、その後の接続が容易になります。

`pikvm.txt`へ次の行を追加します。

```text
WIFI_ESSID='Wi-FiのSSID'
WIFI_PASSWD='Wi-Fiのパスワード'
WIFI_REGDOM='JP'
```

`WIFI_REGDOM='JP'`は日本の無線規制ドメインを指定する設定です。

初回書き込み直後の`pikvm.txt`に次の行が存在する場合は、削除せず残してください。

```text
FIRST_BOOT=1
```

`FIRST_BOOT=1`は、初回起動時にSSHホスト鍵や証明書などを生成するための指定です。

また、Wi-Fiパスワードにバックスラッシュが含まれる場合は、次のように二重に記述する必要があります。

```text
WIFI_PASSWD='abc\\def'
```

`pikvm.txt`は設定の適用後に自動的に削除されます。Raspberry Pi Zero 2 Wを使用した構成では、5GHz Wi-Fiを利用できない点にも注意してください。

編集後はSDカードを安全に取り外し、PiKVMへ戻して起動します。

---

# 3. PiKVMのIPアドレスを確認する

PiKVMは通常、DHCPを使ってIPアドレスを取得します。

起動すると、PiKVMに接続されたディスプレイへ現在のIPアドレスが表示されるため、まずはその表示を確認するのが簡単です。

ディスプレイを接続していない場合は、ルーターの管理画面やDHCPリース一覧から、PiKVMへ割り当てられたIPアドレスを確認します。

以降の例では、PiKVMのIPアドレスを次のように表記します。

```text
192.168.1.100
```


---

# 4. SSHでPiKVMへ接続する

初期状態では、Linuxの管理者アカウントは次の設定です。

| 項目    | 初期値    |
| ----- | ------ |
| ユーザー名 | `root` |
| パスワード | `root` |

SSHクライアントから接続します。

```bash
ssh root@192.168.1.100
```

初回接続時にはSSHホスト鍵の確認が表示されるため、接続先が正しいことを確認して承認します。

PiKVMには、SSHなどで使うLinuxの`root`アカウントとは別に、Web UI用の`admin`アカウントがあります。初期状態では、Web UI側もユーザー名とパスワードがともに`admin`です。両者は独立したアカウントなので、両方のパスワードを変更する必要があります。

---

# 5. 最初に2種類のパスワードを変更する

初期パスワードのまま運用するのは危険です。ネットワーク設定や機能追加よりも先に変更します。

まず、ルートファイルシステムを書き込み可能にします。

```bash
rw
```

Linuxの`root`パスワードを変更します。

```bash
passwd root
```

続いて、Web UIで使用する`admin`のパスワードを変更します。

```bash
kvmd-htpasswd set admin
```

両方の変更が完了したら、ファイルシステムを読み取り専用に戻します。

```bash
ro
```

まとめると、実行順は次のとおりです。

```bash
rw
passwd root
kvmd-htpasswd set admin
ro
```

パスワード変更時に`rw`を実行し忘れると、設定を保存できません。公式ドキュメントでも、`rw`と`ro`の間で両方のパスワードを変更する手順が案内されています。

---

# 6. ホスト名を変更する

複数のPiKVMを管理する場合は、設置場所や管理対象が分かるホスト名へ変更しておくと便利です。

例えば、サーバールームの1号機を管理するPiKVMであれば、次のような名前にします。

```text
pikvm-server01
```

変更手順は次のとおりです。

```bash
rw
systemctl restart systemd-hostnamed
hostnamectl set-hostname pikvm-server01
ro
reboot
```

再起動後、SSHのプロンプトやネットワーク上のホスト名が変更されます。公式FAQでも、`systemd-hostnamed`を再起動してから`hostnamectl set-hostname`を実行し、PiKVMを再起動する方法が案内されています。

Tailscaleを導入する予定がある場合も、先にホスト名を設定しておくと管理画面上で機器を識別しやすくなります。

---

# 7. PiKVM OSをアップデートする

初期設定が終わったら、PiKVM OSを更新します。

```bash
pikvm-update
```

通常、手動で`rw`や`ro`を実行する必要はありません。`pikvm-update`を使用するのがPiKVMで推奨される更新方法です。

ただし、更新には失敗の可能性がまったくないわけではありません。公式ドキュメントでは、SDカードの再書き込みが必要になる場合に備え、可能であればPiKVMへ物理的にアクセスできる状態で更新することが推奨されています。VPN経由でしか接続できないPiKVMを安易に更新するのは避けたほうが安全です。

古いイメージで`pikvm-update`が存在しない場合は、次の手順でアップデーターを導入します。

```bash
rw
pacman -Syy
pacman -S pikvm-os-updater
pikvm-update
```

更新後は、必要に応じて再起動します。

---

# 8. TailscaleをPiKVMへ導入する

インターネット経由でPiKVMを利用する場合、Web UIやSSHを直接ポート開放するよりも、Tailscaleなどのプライベートネットワークを利用する方法が管理しやすくなります。

PiKVMには、PiKVM向けのTailscaleパッケージとして`tailscale-pikvm`が用意されています。

先に`pikvm-update`を実行したうえで、次の順番で導入します。

```bash
rw
pacman -S tailscale-pikvm
systemctl enable --now tailscaled
tailscale up --qr
ro
```

`tailscale up --qr`を実行すると、認証用URLに対応するQRコードがターミナルへ表示されます。スマートフォンで読み取って認証できるため、URLをコピーしにくいリモートコンソール環境で便利です。`--qr`は現在のTailscale CLIで正式に用意されているオプションです。

### Exit Nodeを利用する場合

PiKVMをExit Nodeとして利用する場合は、TailscaleでExit Nodeを有効にします。

```bash
rw

tailscale set \
  --advertise-exit-node \
  --advertise-routes=192.168.0.0/24

ro
```

`--advertise-routes`には、PiKVMから到達可能なLANのネットワークアドレスを指定します。例えば、管理対象ネットワークが`192.168.0.0/24`であれば上記のようになります。

設定後、Tailscale管理画面で対象ノードを開き、**Exit Node**および**Subnet routes**を承認します。

その後、クライアント側でPiKVMをExit Nodeとして選択すると、インターネット通信だけでなく、PiKVMが接続されているLAN上の機器へもアクセスできるようになります。

### Raspberry PiのUDP GRO設定

Raspberry PiをExit Nodeとして利用する場合は、LinuxカーネルのUDP GRO設定を有効にすることが推奨されています。

PiKVMでは次のsystemdサービスを作成すると、起動時に自動で設定が適用されます。

```bash
rw

cat > /etc/systemd/system/apply-udp-gro.service <<'EOF'
[Unit]
Description=Configure UDP GRO forwarding
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now apply-udp-gro.service

ro
```

このサービスは、PiKVMの起動時に

```bash
ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```

を実行し、再起動後もUDP GROの設定を維持します。

設定後は、次のコマンドで状態を確認できます。

```bash
ethtool -k eth0 | grep -E 'rx-udp-gro-forwarding|rx-gro-list'
```

期待する出力は次のとおりです。

```text
rx-gro-list: off
rx-udp-gro-forwarding: on
```

なお、`eth0`以外のインターフェースを利用している場合は、環境に合わせて読み替えてください。

---

# 9. 仮想メディアをPiKVMへ配置する

PiKVMは、ISOやIMGファイルを管理対象マシンへ仮想CD/DVDまたは仮想USBメモリーとして接続できます。

イメージファイルは、PiKVM上の次のディレクトリに保存されます。

```text
/var/lib/kvmd/msd
```

通常はWeb UIの`Drive`メニューからアップロードする方法が簡単です。手動でSCPやrsyncを使用する場合は、仮想メディア専用領域を書き込み可能にします。

PiKVM上で次を実行します。

```bash
kvmd-helper-otgmsd-remount rw
```

次に、手元のPCからISOファイルを転送します。

```bash
scp installer.iso root@192.168.1.100:/var/lib/kvmd/msd/
```

転送が完了したら、PiKVM上で読み取り専用へ戻します。

```bash
kvmd-helper-otgmsd-remount ro
```

公式ドキュメントでも、手動アップロードは次の順番になっています。

1. `kvmd-helper-otgmsd-remount rw`
2. `/var/lib/kvmd/msd`へファイルを転送
3. `kvmd-helper-otgmsd-remount ro`

仮想メディア用パーティションを通常は読み取り専用にすることで、突然の電源断によるデータ破損を防いでいます。

---

## ISOファイルに`chown kvmd:kvmd`は必要か

SCPで転送したファイルは、通常は`root`所有になります。

```bash
ls -l /var/lib/kvmd/msd/
```

例えば、次のように全ユーザーから読み取り可能になっていれば、読み取り専用ISOとして使用するだけなら、必ずしも所有者を変更する必要はありません。

```text
-rw-r--r-- 1 root root ... installer.iso
```

つまり、次のコマンドをすべてのISOへ機械的に実行する必要はありません。

```bash
chown kvmd:kvmd installer.iso
```

重要なのは、`kvmd`ユーザーがファイルを読み取れることです。必要なら、次のように読み取り権限を設定します。

```bash
kvmd-helper-otgmsd-remount rw
chmod 644 /var/lib/kvmd/msd/installer.iso
kvmd-helper-otgmsd-remount ro
```

一方、管理対象マシンから書き込み可能な仮想Flashイメージとして使用する場合は、書き込み権限も必要です。公式ドキュメントの書き込み可能なFlashイメージ作成例では、次のように`chmod 666`を設定しています。

```bash
chmod 666 /var/lib/kvmd/msd/flash.img
```

運用方針として所有者を統一したい場合は、次のようにしても構いません。

```bash
kvmd-helper-otgmsd-remount rw
chown kvmd:kvmd /var/lib/kvmd/msd/installer.iso
chmod 644 /var/lib/kvmd/msd/installer.iso
kvmd-helper-otgmsd-remount ro
```

ただし、読み取り専用ISOについては「`kvmd`から読み取れること」が本質であり、`chown`自体が必須条件ではありません。

---

# 10. 仮想メディアから管理対象マシンを起動する

ISOを配置したら、PiKVMのWeb UIを開き、`Drive`メニューから対象イメージを選択します。

基本的な流れは次のとおりです。

1. `Drive`メニューを開く
2. ISOまたはIMGファイルを選択する
3. メディア種別を選択する
4. イメージを接続する
5. 管理対象マシンを再起動する
6. BIOSまたはUEFIのブートメニューから仮想メディアを選択する

PiKVMでは、仮想メディアを次のような形式で接続できます。

| モード    | 管理対象マシンからの見え方 | 主な用途                          |
| ------ | ------------- | ----------------------------- |
| CD/DVD | 光学ドライブ        | 一般的なOSインストールISO、レスキューISO      |
| Flash  | USBメモリー       | UEFIとの相性対策、書き込み可能なファイル交換用イメージ |

通常はISOをCD/DVDとして接続します。

ただし、管理対象マシンのUEFI実装によっては、CD/DVDとして接続したイメージをブートデバイスとして認識できない場合があります。その場合は、同じイメージを`Flash`として接続すると認識されることがあります。

これはすべての機種で必要な設定ではなく、仮想CD/DVDから起動できない場合の互換性対策です。

PiKVMの仮想メディアはBIOS・UEFIから利用でき、Web UI上でCD/DVDまたはFlashとしての接続方式を変更できます。

---

## 仮想メディア利用時の注意

イメージのアップロード中や、書き込み可能な状態で管理対象マシンへ接続している間は、PiKVMの電源を切らないでください。

アップロード中や書き込み中に電源を切ると、イメージファイルやファイルシステムが破損する可能性があります。

---

# 11. Ctrl＋Alt＋Delを送信する

ブラウザーが`Ctrl`や`Alt`などのキー操作をローカル側で処理してしまうことがあるため、特殊なショートカットはPiKVMのWeb UIから送信します。

`Ctrl＋Alt＋Del`を送信する場合は、Web UI上部の`Shortcuts`メニューを開き、`Ctrl+Alt+Del`を選択します。

`Shortcuts`メニューに目的の操作がない場合は、PiKVMのショートカット用マジックキー機能を使う方法もあります。現在の公式ドキュメントでは、マジックキーを押して離した後、`Ctrl`、`Alt`、`Del`を順番に押して離す操作が案内されています。

---

# 12. 管理対象マシンへTailscaleを設定する

PiKVMから操作している管理対象マシンにもTailscaleを導入する場合、Linuxでは次のコマンドが便利です。

```bash
tailscale up --qr
```

画面に表示されたQRコードをスマートフォンで読み取り、Tailscaleの認証を完了します。

PiKVMの画面上で長い認証URLを手入力したり、クリップボードを経由したりする必要がないため、OSインストール直後やGUIを利用できない環境に適しています。

---

# 作業全体の推奨順序

初回セットアップは、次の順番で進めると手戻りを減らせます。

1. SDカードへPiKVM OSを書き込む
2. 起動前に`pikvm.txt`へWi-Fi情報を設定する
3. PiKVMを起動してIPアドレスを確認する
4. SSHで`root`として接続する
5. Linuxの`root`パスワードを変更する
6. Web UIの`admin`パスワードを変更する
7. ホスト名を変更する
8. 物理アクセス可能な状態で`pikvm-update`を実行する
9. 必要に応じてTailscaleを導入する
10. ISO・IMGファイルを仮想メディア領域へ配置する
11. Web UIから管理対象マシンへ接続する
12. 仮想メディアを接続してOSのインストールや復旧作業を行う

---

# まとめ

PiKVMを扱ううえで最も混乱しやすいのは、書き込みモードの切り替えです。

システム設定を変更する場合は、次の組み合わせを使用します。

```bash
rw
# 設定変更
ro
```

ISOやIMGを配置する場合は、次の組み合わせです。

```bash
kvmd-helper-otgmsd-remount rw
# ISO・IMGの配置や削除
kvmd-helper-otgmsd-remount ro
```

この2種類を区別し、変更後は読み取り専用へ戻すことが、安全にPiKVMを運用するための基本です。

また、初回起動後は必ずLinux側とWeb UI側の両方のパスワードを変更してください。外部からアクセスする場合は、Web UIを直接公開するのではなく、Tailscaleなどのプライベートネットワークを組み合わせると、接続経路を管理しやすくなります。
