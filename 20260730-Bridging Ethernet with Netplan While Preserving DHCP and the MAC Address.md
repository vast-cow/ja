---
pubDatetime: 2026-07-30T13:29:13+09:00
title: "NetplanでEthernetをブリッジ化して、DHCPとMACアドレスを維持する"
description: "LinuxマシンをWi-Fiアクセスポイント（hostapd）として利用する場合、Wi-Fiクライアントを既存の有線LANと同じL2ネットワークに参加させたいことがあります。 そのような場合は、EthernetインターフェースをLinux Bridgeに参加させ、IPアドレスはBridge側に持たせ…"
---

LinuxマシンをWi-Fiアクセスポイント（hostapd）として利用する場合、Wi-Fiクライアントを既存の有線LANと同じL2ネットワークに参加させたいことがあります。

そのような場合は、EthernetインターフェースをLinux Bridgeに参加させ、IPアドレスはBridge側に持たせる構成が一般的です。

## Netplanの設定例

例えば、Ethernetインターフェースが `eth0` の場合は、以下のように設定します。

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false

  bridges:
    br0:
      interfaces:
        - eth0
      dhcp4: true
      dhcp6: false
      macaddress: xx:xx:xx:xx:xx:xx
```

この設定では次のようになります。

* `eth0` はBridgeのメンバーポートとなる
* IPアドレスは `br0` がDHCPで取得する
* `br0` のMACアドレスを元のEthernetインターフェースと同じ値に固定する

### なぜMACアドレスを固定するのか

Bridgeを作成すると、環境によっては `br0` に新しいMACアドレスが割り当てられることがあります。

MACアドレスが変わると、

* DHCPサーバから新しいクライアントとして認識される
* DHCPリースが変わる
* MACアドレスベースのアクセス制御に影響する
* ARPキャッシュが更新される

といった副作用が起こる可能性があります。

そこで、

```yaml
macaddress: xx:xx:xx:xx:xx:xx
```

のように、元のEthernetインターフェースと同じMACアドレスを設定しておくことで、既存ネットワークへの影響を最小限にできます。

## 安全に設定を試す

リモート（SSH）でネットワーク設定を変更する場合は、誤った設定によって接続できなくなるリスクがあります。

そのため、`netplan apply` をいきなり実行するのではなく、`netplan try` を使うことをおすすめします。

```bash
sudo netplan try --timeout 30
```

このコマンドは設定を一時的に適用し、30秒以内に確認操作を行えば設定を確定します。

もし接続できなくなったり、確認を行わなかった場合は、タイムアウト後に元の設定へ自動的に戻ります。

SSH越しで設定を変更する場合でも比較的安全に試すことができます。

## hostapdと組み合わせる

このBridgeを作成した後は、hostapdでBridgeを指定します。

```ini
interface=wlp2s0
bridge=br0
```

すると、Wi-Fiクライアントは `br0` を経由して有線LANと同じL2ネットワークに接続されます。

その結果、

* 有線端末
* Wi-Fi端末
* DHCPサーバ

がすべて同一セグメント上で動作し、Wi-Fiクライアントも既存のDHCPサーバから直接IPアドレスを取得できるようになります。

## まとめ

Linuxでhostapdを利用する場合は、Bridgeを利用する構成がシンプルで管理しやすい方法です。

特に次の3点を押さえておくと、既存ネットワークへの影響を抑えながら移行できます。

* IPアドレスは物理NICではなくBridgeに持たせる
* BridgeのMACアドレスを元のEthernetインターフェースと同じ値に設定する
* リモート作業では `netplan try` を利用し、安全に設定変更を行う
