---
title: "Rocky Linux 9でNetworkManagerを使用してイーサネットブリッジを構築し、DHCPとMACアドレスを維持する"
description: "ネットワークトポロジー、既存のイーサネット接続を識別する、既存のMACアドレスを記録するを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-07-28T12:31:43.829Z
updatedDate: 2026-09-16T04:13:14.626Z
---

Ubuntuでは、イーサネットインターフェイスをLinuxブリッジに接続するためにNetplanを使用することが一般的です。しかし、Rocky Linux 9では、通常、ネットワークは**NetworkManager**によって管理されます。

Netplanの代わりに、Rocky Linuxは`nmcli`などのツールを使用してブリッジを作成し、物理ネットワークインターフェイスをブリッジに接続します。

この記事では、DHCPを維持し、元のイーサネットMACアドレスを維持し、NetworkManagerのチェックポイントを使用して、リモート接続の損失を防ぐことで、Rocky Linux 9でUbuntu Netplanブリッジ構成と同等のものを構築する方法について説明します。

## ネットワークトポロジー

たとえば、有線ネットワークインターフェイスが次のようになっている場合：

```text
eth0
```

最終的な構成は次のようになります。

```text
          DHCPサーバー
               │
        イーサネットスイッチ
               │
             eth0
               │
         Linuxブリッジ
             br0
               │
      IPアドレス（DHCP）
               │
         hostapd（AP）
               │
        Wi-Fiクライアント
```

この構成では：

* 物理NICはブリッジのポートになります。
* ブリッジ（`br0`）がIPアドレスを所有します。
* DHCPは`eth0`ではなく`br0`で実行されます。
* ブリッジは、以前`eth0`で使用されていたMACアドレスを使用します。
* `hostapd`はワイヤレスインターフェイスをブリッジに接続できます。
* Wi-Fiクライアントは、有線LANと同じレイヤー2ネットワークに参加します。

## 既存のイーサネット接続を識別する

変更する前に、現在`eth0`でアクティブになっているNetworkManagerの接続プロファイルがどれであるかを判断します。

```bash
nmcli device status
```

アクティブなプロファイルを直接取得することもできます。

```bash
nmcli -g GENERAL.CONNECTION device show eth0
```

たとえば、結果は次のようになります。

```text
Wired connection 1
```

この名前を後で使用するために保存します。以下のコマンドでは、次のように表されます。

```text
<existing-profile-name>
```

**まだ**このプロファイルを削除しないでください。これは、ブリッジの移行が失敗した場合に、NetworkManagerが復元できる既知の正常な構成を提供します。

## 既存のMACアドレスを記録する

`eth0`の現在の永続的なMACアドレスを調べます。

```bash
cat /sys/class/net/eth0/address
```

たとえば：

```text
00:11:22:33:44:55
```

この値はブリッジに割り当てられます。

必要に応じて、シェル変数に保存することもできます。

```bash
ETH_MAC=$(cat /sys/class/net/eth0/address)
```

## アクティブ化せずにブリッジを作成する

ブリッジプロファイルを作成します。

```bash
sudo nmcli connection add \
    type bridge \
    ifname br0 \
    con-name br0
```

構成をリモートで準備しているため、ステージング中に自動アクティブ化を無効にします。

```bash
sudo nmcli connection modify br0 \
    connection.autoconnect no
```

これにより、NetworkManagerが不完全なブリッジ構成を予期せずにアクティブ化することが防止されます。

## ブリッジのMACアドレスを維持する

新しく作成されたブリッジは、それ以外の場合、異なるMACアドレスを使用する可能性があります。

ブリッジが元のイーサネットインターフェイスと同じMACアドレスを使用するように構成します。

```bash
sudo nmcli connection modify br0 \
    bridge.mac-address "$ETH_MAC"
```

または、アドレスを直接指定します。

```bash
sudo nmcli connection modify br0 \
    bridge.mac-address 00:11:22:33:44:55
```

MACアドレスを維持することで、ホストの既存のネットワークIDを維持するのに役立ちます。

## ブリッジでDHCPを構成する

IPアドレスは物理NICではなく、ブリッジが取得する必要があります。

IPv4 DHCPを構成し、それが意図したネットワーク構成に一致する場合はIPv6を無効にします。

```bash
sudo nmcli connection modify br0 \
    ipv4.method auto \
    ipv6.method disabled
```

ネットワークでIPv6が必要な場合は、無効にするのではなく、適切に構成してください。

## イーサネットブリッジポートを作成する

`eth0`を`br0`に接続する別のNetworkManagerプロファイルを作成します。

```bash
sudo nmcli connection add \
    type bridge-slave \
    ifname eth0 \
    master br0 \
    con-name br0-port-eth0
```

新しい構成をステージング中に、このプロファイルの自動接続を無効にします。

```bash
sudo nmcli connection modify br0-port-eth0 \
    connection.autoconnect no
```

この時点で、新しいブリッジ構成はディスクに存在しますが、まだアクティブなイーサネット接続を置き換えているわけではありません。

プロファイルを検証します。

```bash
nmcli connection show
```

次のようなエントリが表示されます。

```text
NAME                 TYPE      DEVICE
Wired connection 1   ethernet  eth0
br0                  bridge    --
br0-port-eth0        ethernet  --
```

正確な出力は、NetworkManagerのバージョンと既存の構成によって異なります。

## NetworkManagerチェックポイントで移行をテストする

移行の最も破壊的な部分は、`eth0`をブリッジポートとしてアクティブ化することです。

SSH経由でこれを行う場合は、NetworkManagerチェックポイントを使用します。

```bash
sudo nmcli device checkpoint --timeout 120 -- \
    nmcli connection up br0-port-eth0
```

ブリッジポートをアクティブにすると、NetworkManagerは必要に応じてブリッジコントローラーをアクティブにします。

`eth0`は1つのNetworkManager接続プロファイルしか使用できないため、`br0-port-eth0`をアクティブにすると、`eth0`上の現在のアクティブなスタンドアロンイーサネットプロファイルが置き換えられます。

この操作中は、SSH接続が一時的に中断される可能性があります。

NetworkManagerは、変更を保持するかどうかを尋ねます。

新しい構成が機能し、SSH接続が利用可能な場合は、次のように答えます。

```text
Yes
```

ネットワーク構成が壊れてSSH接続が失われた場合、チェックポイントを確認することはできません。タイムアウトが期限切れになると、NetworkManagerはコマンドの実行前に存在していたネットワーク状態を復元しようとします。

デフォルトのチェックポイントのタイムアウトは短いため、DHCPが完了するまで十分な時間を与えるために、120秒などのより長いタイムアウトを指定すると便利です。

## 確認する前にブリッジを検証する

「はい」と答える前に、可能であれば別のSSHセッションから構成を確認します。

ブリッジアドレスを確認します。

```bash
ip addr show br0
```

ブリッジポートを確認します。

```bash
bridge link
```

NetworkManagerを確認します。

```bash
nmcli device status
```

デフォルトルートを確認します。

```bash
ip route
```

期待される状態は次のとおりです。

* `eth0`は`br0`のポートです。
* `br0`がIPv4アドレスを所有します。
* DHCPが`br0`にアドレスを割り当てました。
* デフォルトルートは`br0`を使用します。
* ブリッジは、元のイーサネットMACアドレスを持っています。
* リモートSSH接続は引き続き機能します。

次のコマンドでブリッジのMACアドレスを確認できます。

```bash
ip link show br0
```

そして、次のコマンドと比較します。

```bash
cat /sys/class/net/eth0/address
```

これらのチェックが成功したら、「はい」とチェックポイントのプロンプトに答えます。

## 動作する構成を永続化する

テストプロファイルは、意図的に自動接続が無効になっている状態で構成されました。

チェックポイントが成功し、接続が確認されたら、ブリッジとそのイーサネットポートで自動接続を有効にします。

```bash
sudo nmcli connection modify br0 \
    connection.autoconnect yes

sudo nmcli connection modify br0-port-eth0 \
    connection.autoconnect yes
```

次に、古いスタンドアロンのイーサネットプロファイルの自動アクティブ化を無効にします。

```bash
sudo nmcli connection modify "<existing-profile-name>" \
    connection.autoconnect no
```

古いプロファイルを一時的に残しておく方が安全です。

結果の設定を確認します。

```bash
nmcli -f NAME,TYPE,AUTOCONNECT connection show
```

次のように表示されるはずです。

```text
br0                  bridge    yes
br0-port-eth0        ethernet  yes
<existing-profile>   ethernet  no
```

## 再起動テスト

ライブ移行が成功したとしても、システムが意図した構成で起動するかどうかは証明されません。

リモートアクセスが重要な場合は、最初に再起動する前に、必ずアウトオブバンドコンソール（IPMI、iDRAC、iLO、またはハイパーバイザーコンソールなど）が利用可能になっていることを確認してください。

次に、再起動します。

```bash
sudo reboot
```

システムが再起動したら、次を確認します。

```bash
nmcli device status
```

```bash
ip addr show br0
```

```bash
bridge link
```

```bash
ip route
```

期待される構成は次のとおりです。

* `br0`がアクティブです。
* `eth0`が`br0`に接続されています。
* `br0`にDHCPアドレスがあります。
* デフォルトルートは`br0`を使用します。
* 古いスタンドアロンのイーサネットプロファイルは非アクティブのままです。

## 古いイーサネットプロファイルを削除する

ブリッジが再起動に耐え、完全に検証された場合にのみ、古いスタンドアロンのイーサネットプロファイルを削除します（必要な場合）。

```bash
sudo nmcli connection delete "<existing-profile-name>"
```

削除することはオプションです。`connection.autoconnect no`で残しておくこともできます。これは、リカバリ構成として役立ちます。

## MACアドレスを維持する理由

ブリッジを作成すると、元のイーサネットインターフェイスとは異なるMACアドレスを使用する可能性があります。

MACアドレスが変更されると：

* DHCPサーバーは、システムを別のデバイスとして認識する可能性があります。
* 新しいDHCPリースが割り当てられる可能性があります。
* 古いMACアドレスに関連付けられたDHCP予約は機能しなくなる可能性があります。
* MACアドレスベースのアクセス制御が一致しなくなる可能性があります。
* 隣接するシステムは、ARPまたはネイバーキャッシュエントリを更新する必要がある場合があります。

既存のネットワークへの変更を最小限に抑えるために、ブリッジが元のイーサネットインターフェイスで使用していたMACアドレスを使用するように構成します。

```bash
bridge.mac-address 00:11:22:33:44:55
```

レイヤー3のIDは`eth0`から`br0`に移動しますが、レイヤー2のアドレスは同じままです。

## hostapdとのブリッジの使用

ブリッジを作成して検証した後、`hostapd.conf`でそれを指定します。

```ini
interface=wlp2s0
bridge=br0
```

これにより、次のことが可能になります。

* 有線LANデバイス
* Wi-Fiクライアント
* 上流のDHCPサーバー

が同じレイヤー2ネットワークで動作します。

Wi-Fiクライアントは、既存のDHCPサーバーから直接IPアドレスを取得できるため、別のルーティングまたはNATネットワークは必要ありません。

## 構成は再起動後も維持される

`nmcli`を使用して作成されたブリッジは、単なる一時的なカーネル構成ではありません。

NetworkManagerは、次の場所に永続的な接続プロファイルを保存します。

```text
/etc/NetworkManager/system-connections/
```

Rocky Linux 9では、新しく作成された接続に対してNetworkManagerキーファイルプロファイルが使用されます。

たとえば：

```bash
sudo nmcli connection add \
    type bridge \
    ifname br0 \
    con-name br0
```

通常、NetworkManager接続プロファイルは次の場所に保存されます。

```text
/etc/NetworkManager/system-connections/
```

ブリッジポートプロファイルも永続的に保存されます。

対照的に、次のコマンドでブリッジを直接作成すると：

```bash
ip link add br0 type bridge
```

カーネルネットワークインターフェイスのみが作成されます。別の構成メカニズムが起動時にそれを再作成しない場合、そのインターフェイスは再起動後に消えます。

## SSH経由での再構成に関する注意事項

インターフェイスを変更すると、SSHセッションが確立されている場合、リスクがあります。

NetworkManagerは、破壊的なネットワーク変更をより安全に行うための`device checkpoint`コマンドを提供します。

一般的な形式は次のとおりです。

```bash
nmcli device checkpoint [--timeout SECONDS] [DEVICE...] -- COMMAND
```

NetworkManagerはチェックポイントを取得し、指定されたコマンドを実行し、結果のネットワーク状態を保持するかどうかを尋ねます。

確認を受け取らない場合、タイムアウトの前にNetworkManagerは前の状態を復元しようとします。

このブリッジ移行の場合：

```bash
sudo nmcli device checkpoint --timeout 120 -- \
    nmcli connection up br0-port-eth0
```

は、Ubuntuの次のコマンドと同様の動作を提供します。

```bash
sudo netplan try
```

いくつかの重要な注意事項があります。

* ブリッジをテストする前に、元のイーサネットプロファイルを削除しないでください。
* 新しいプロファイルを、`connection.autoconnect no`でステージングします。
* DHCPが完了するまで、十分なチェックポイントのタイムアウトを使用します。
* `br0`にアドレス、ルート、および動作する接続があることを確認してから、チェックポイントを受け入れます。
* ブリッジが正常にテストされるまで、新しいプロファイルの自動接続を有効にします。
* 古いイーサネットプロファイルを最初に無効にし、新しい構成が再起動に耐えた後にのみ削除します。
* 可能な場合は、アウトオブバンドコンソール（IPMI、iDRAC、iLO、またはハイパーバイザーコンソールなど）を使用します。
* `tmux`または`screen`は、他の管理作業には依然として役立ちますが、壊れたネットワーク構成から保護するものではありません。NetworkManagerチェックポイントは、自動ネットワーク状態のロールバックを提供するものです。

## Rocky Linux 9のバージョンによる違い

最近のRHEL 9およびRocky Linux 9リリースでは、「コントローラー」と「ポート」という用語が、ブリッジなどの関係に使用されます。

たとえば、新しい構文では次のようになります。

```bash
sudo nmcli connection add \
    type ethernet \
    ifname eth0 \
    con-name br0-port-eth0 \
    port-type bridge \
    controller br0
```

古いRocky Linux 9リリースでは、同等の`bridge-slave` / `master`用語が使用されます。

```bash
sudo nmcli connection add \
    type bridge-slave \
    ifname eth0 \
    master br0 \
    con-name br0-port-eth0
```

後者の形式は、古いRocky Linux 9のインストールにも対応する必要がある場合に役立ちます。

## NetplanとNetworkManagerの比較

| Netplan         | NetworkManager（`nmcli`）             |
| --------------- | ------------------------------------ |
| `bridges:`      | `type bridge`                        |
| `interfaces:`   | ブリッジポート/ `bridge-slave`         |
| `dhcp4: true`   | `ipv4.method auto`                   |
| `dhcp6: false`  | `ipv6.method disabled`               |
| `macaddress:`   | `bridge.mac-address`                 |
| `netplan apply` | `nmcli connection up ...`            |
| `netplan try`   | `nmcli device checkpoint -- COMMAND` |

正確な実装は異なりますが、どちらも、潜在的に破壊的なネットワーク変更をロールバック保護でテストする方法を提供します。

## 概要

Rocky Linux 9では、NetworkManagerの`nmcli`を使用して、UbuntuでNetplanを使用して一般的に構成されるものと同じ種類のイーサネットブリッジを作成できます。

重要な点は次のとおりです。

* IP構成を物理NICではなく、ブリッジに割り当てます。
* ネットワークIDを維持するために、ブリッジがイーサネットNICによって以前に使用されていたMACアドレスを使用するように構成します。
* `eth0`を`br0`に接続する別のブリッジポートプロファイルを作成します。
* ブリッジをテストする前に、既知の良好なイーサネットプロファイルを削除しないでください。
* 新しいプロファイルを、`connection.autoconnect no`でステージングします。
* SSH経由でこれを行う場合は、`nmcli device checkpoint`を使用します。
* `br0`にアドレス、ルート、および動作する接続があることを確認してから、チェックポイントを受け入れます。
* ブリッジが正常にテストされるまで、新しいプロファイルの自動接続を有効にします。
* 古いイーサネットプロファイルを最初に無効にし、新しい構成が再起動に耐えた後にのみ削除します。
* `hostapd`が`br0`を指すようにし、Wi-Fiクライアントが有線LANと同じレイヤー2ネットワークに参加できるようにします。
* `nmcli`で作成されたNetworkManagerプロファイルは、再起動後も永続化されます。

この設定により、有線LANデバイスとWi-Fiクライアントが同じレイヤー2ネットワークを共有し、NetworkManagerのチェックポイントメカニズムにより、移行中にリモート接続が永続的に失われるリスクが大幅に軽減されます。
