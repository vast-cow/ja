---
pubDatetime: 2026-07-28T21:16:54+09:00
title: "MacBookPro14,2 + Ubuntu 24.04でhostapdを設定する"
description: "nl80211、bridge、5 GHz固定を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

少し特殊な用途ですが、MacBook ProをLinuxルータ兼Wi-Fi APとして使う場合の記録です。

今回は **MacBookPro14,2 (2017, 13-inch)** に **Ubuntu 24.04** をインストールし、内蔵無線LANを `hostapd` でアクセスポイント化しました。

なお、本記事は **hostapdの設定** に焦点を当てています。`br0` の作成やIPアドレス設定などのブリッジ構成については扱いません。

## 環境

* MacBookPro14,2
* Ubuntu 24.04 LTS
* hostapd
* 無線IF: `wlp2s0`
* 有線IF: `enx68da73adde3c`
* ブリッジ: `br0`

`hostapd` は `nl80211` ドライバを使用します。

現在のLinuxでmac80211/cfg80211ベースの無線LANドライバを利用する場合、通常は `driver=nl80211` を指定します。

---

# hostapd.conf

現在使用している設定は次のようになりました。

```ini
driver=nl80211

country_code=JP
ieee80211d=1

# 5 GHz
hw_mode=a
channel=36

# 802.11n/ac
ieee80211n=1
ieee80211ac=1
wmm_enabled=1

# 40 MHz
ht_capab=[HT40+]

# 80 MHzは無効
# no_pri_sec_switch=1
# vht_oper_chwidth=1
# vht_oper_centr_freq_seg0_idx=42

# WPA2
wpa=2
# wpa_key_mgmt=WPA-PSK SAE
# wpa_key_mgmt=SAE
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP

# PMFは無効
# ieee80211w=1
# ieee80211w=2
ieee80211w=0

ignore_broadcast_ssid=1
disable_pmksa_caching=1

# SAE使用時の設定候補
# sae_pwe=2

# interface=wlx3476c5d38aef
interface=wlp2s0
bridge=br0

ssid=YOUR_SSID
wpa_passphrase=YOUR_PASSWORD
# wpa_psk=...
```

---

# 各設定の説明

## nl80211

```ini
driver=nl80211
```

Linuxの無線LANドライバとの通信に `nl80211` インターフェースを使用します。

mac80211/cfg80211ベースの一般的なLinux無線LANドライバでは、通常この設定を使用します。

---

## bridge

```ini
bridge=br0
```

無線クライアントを `br0` に接続し、有線LAN側と同一のL2セグメントへ参加させます。

既存LAN側にDHCPサーバが存在する構成であれば、Wi-Fiクライアントもブリッジを経由してそのDHCPサーバからアドレスを取得できます。

そのため、hostapd自身がDHCPサーバになる必要はありません。

---

## 5 GHz固定

```ini
hw_mode=a
channel=36
```

5 GHz帯のチャネル36を使用します。

今回はDFSによるレーダー検出やCACを必要としないW52のチャネルを選択しています。

---

## 日本のregulatory domain

```ini
country_code=JP
ieee80211d=1
```

`country_code=JP` を指定し、日本のregulatory domainに従って動作させます。

利用可能なチャネルや送信条件は、hostapdの設定だけではなく、Linuxのcfg80211/regulatory database、無線LANドライバ、デバイス側の制約にも影響されます。

`ieee80211d=1` は802.11dを有効化し、APがCountry Informationを通知できるようにします。

---

# 802.11n / 802.11ac

```ini
ieee80211n=1
ieee80211ac=1
wmm_enabled=1
```

802.11nおよび802.11acを有効にしています。

`wmm_enabled=1` はWMM（Wi-Fi Multimedia）を有効にする設定です。802.11nなどのHT動作ではQoS/WMMが前提になるため、有効にしています。

---

# 40 MHz幅（HT40）

以前の設定では `ht_capab` を無効にして20 MHz幅で運用していましたが、現在は

```ini
ht_capab=[HT40+]
```

を有効にしています。

`HT40+` は、プライマリチャネルより上側にセカンダリチャネルを配置する40 MHz HT動作を指定します。

チャネル36の場合、隣接する上側のチャネルを組み合わせることで40 MHz幅を構成します。

以前はこの指定を入れると

```text
Could not set channel for kernel driver
```

となりAPを起動できませんでしたが、現在の環境では `HT40+` を有効にした状態で動作させています。

したがって、以前の記事に記載していた

> HT40はドライバまたはファームウェア側で利用できない

という結論は、現在の環境には当てはまりません。

少なくとも現在は **5 GHz / channel 36 / HT40+** の構成で利用できています。

---

# 80 MHz（VHT80）は無効

一方、80 MHz幅については現在も明示的に有効化していません。

```ini
# no_pri_sec_switch=1
# vht_oper_chwidth=1
# vht_oper_centr_freq_seg0_idx=42
```

`vht_oper_chwidth=1` と `vht_oper_centr_freq_seg0_idx=42` は、チャネル36をプライマリチャネルとする5 GHzの80 MHz VHT構成で使用する設定です。

現在はこれらをコメントアウトし、**HT40までを使用する構成** としています。

重要なのは、

```ini
ieee80211ac=1
```

だけで「必ず80 MHz幅になる」わけではない点です。

802.11ac（VHT）を有効にすることと、実際に80 MHzチャネル幅を使用することは別の設定です。

---

# WPA2-Personalのみ使用

現在は

```ini
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

としています。

以前は

```ini
wpa_key_mgmt=WPA-PSK SAE
```

としてWPA2/WPA3移行モードを使用していましたが、現在はSAEを無効にして **WPA2-Personal（WPA-PSK）のみ** としています。

`wpa=2` はRSN、一般にWPA2として呼ばれる方式を使用する設定です。

`wpa_key_mgmt=WPA-PSK` によりPSK認証を使用します。

---

# AES（CCMP）のみ

```ini
rsn_pairwise=CCMP
```

ペアワイズ暗号としてCCMPを使用します。

TKIPは使用しません。

一般的なWPA2-Personal + AESの構成です。

---

# WPA3 / SAEは現在無効

以前使用していたWPA3-Personal（SAE）は現在無効にしています。

設定ファイルには検証用として、

```ini
# wpa_key_mgmt=WPA-PSK SAE
# wpa_key_mgmt=SAE
# sae_pwe=2
```

を残していますが、すべてコメントアウトされています。

したがって現在のAPはWPA3-Personalを提供していません。

`sae_pwe` もSAEを使用していない現在の構成では適用されません。

---

# PMFも無効

現在は

```ini
ieee80211w=0
```

としています。

`ieee80211w` はProtected Management Frames（PMF / IEEE 802.11w）の設定です。

概ね、

```text
0 = 無効
1 = 任意（optional）
2 = 必須（required）
```

という指定になります。

以前はWPA2/WPA3移行モードに合わせてPMFを有効にしていましたが、WPA3/SAEを使用しない現在の構成では `ieee80211w=0` としてPMFを無効化しています。

設定ファイルには

```ini
# ieee80211w=1
# ieee80211w=2
ieee80211w=0
```

として、切り替え候補を残しています。

なお、SAEによるWPA3-Personalを再び有効にする場合は、PMFの要件についても合わせて設定を見直す必要があります。

---

# PMKSAキャッシュを無効化

現在はさらに

```ini
disable_pmksa_caching=1
```

を指定しています。

PMKSA（Pairwise Master Key Security Association）キャッシュは、過去の認証結果を利用して再接続時の認証処理を効率化する仕組みです。

`disable_pmksa_caching=1` を指定すると、このPMKSAキャッシュを無効化します。

通常のAPでは必ずしも無効にする必要はありませんが、今回は接続互換性や挙動を切り分けるため、キャッシュを利用しない設定にしています。

---

# SSIDを隠す

```ini
ignore_broadcast_ssid=1
```

SSIDを通常の方法ではビーコンに含めない、いわゆる「ステルスSSID」として動作させます。

ただし、SSIDを隠すこと自体をセキュリティ対策と考えるべきではありません。

また、

* クライアント側でSSIDの手動設定が必要になる
* クライアントによっては接続性に影響する
* トラブルシューティングが複雑になる

といったデメリットがあります。

セキュリティ上必要だから設定しているというより、今回の用途上の設定です。

---

# interface

```ini
interface=wlp2s0
```

hostapdがAPとして使用する無線インターフェースを指定します。

以前検証したUSB無線LANインターフェース

```ini
# interface=wlx3476c5d38aef
```

も設定ファイルには残していますが、現在使用しているのは

```ini
interface=wlp2s0
```

です。

---

# 起動

デバッグログを表示して起動する場合は、

```bash
sudo hostapd -dd /etc/hostapd/hostapd.conf
```

とします。

正常にAPモードへ移行すると、

```text
wlp2s0: AP-ENABLED
```

のようなログが表示されます。

チャネル幅やドライバの問題を調査するときは、systemd経由で起動する前に `-dd` を付けて直接起動すると、hostapdとnl80211間の処理を確認しやすくなります。

---

# HT40 / VHT80について

当初は40 MHz / 80 MHz動作を試した際、

```text
Could not set channel for kernel driver
```

や

```text
80/80+80 MHz: no second channel offset
```

といったエラーが発生しました。

そのため以前は `ht_capab` とVHT80関連設定をすべて外し、20 MHz幅で運用していました。

しかし、その後設定や環境を見直した結果、現在は

```ini
hw_mode=a
channel=36

ieee80211n=1
ieee80211ac=1

ht_capab=[HT40+]
```

という構成で動作しています。

つまり現在の状況は、

* 20 MHzのみ → **以前の構成**
* HT40 → **現在は利用可能**
* VHT80 → **現在は使用していない**

となります。

`iw list` にHT/VHT capabilityが表示されることと、特定のチャネル幅・チャネル構成でAPを実際に起動できることは同義ではありません。

実際の可否は、無線チップのcapabilityだけでなく、ドライバ、ファームウェア、regulatory domain、チャネル構成などにも依存します。

---

# 最終的な構成

現在の構成をまとめると、

* 5 GHz
* チャネル36
* **40 MHz幅（HT40+）**
* 802.11n有効
* 802.11ac有効
* **WPA2-Personal（WPA-PSK）のみ**
* AES（CCMP）のみ
* **WPA3/SAEは無効**
* **PMFは無効**
* PMKSA cachingは無効
* `br0` へのブリッジ接続
* ステルスSSID

という構成です。

以前はHT40/VHT80でAPを起動できなかったため20 MHz幅に落とし、WPA2/WPA3移行モード + PMFという構成で使用していました。

現在は逆に、**無線側はHT40を有効化して40 MHz幅とし、認証側はWPA2-PSKのみに単純化**しています。

MacBookPro14,2 + Ubuntu 24.04 + hostapdというやや特殊な構成では、`iw list` に表示されるcapabilityだけで判断せず、実際に `hostapd -dd` で起動してnl80211/ドライバが受け付ける設定を確認しながら調整するのが確実です。
