---
pubDatetime: 2026-01-19T07:51:00
title: 予備実験：オンデマンド（Suspend + WoL）アクセスシステムのための Floating IP/MAC ハンドオフ
description: "目的、想定設計（意図した仕様）、なぜ自明ではないのか（成立させるための前提条件）を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## 目的

アイドル時には対象マシンを **Suspend** 状態に保ち、クライアントからのアクセス要求があったときに **Wake on LAN (WoL)** によって **オンデマンドで起動**させるシステムを構築したい。

クライアント視点では、常に**安定した宛先**に対してアクセスでき、再設定を必要とせず、かつ起動までの待ち時間（ウェイクアップギャップ）を可能な限り自然に吸収できることが望ましい。

この実現可能性を検証するため、固定された **サービス用 MAC アドレス（mac0）** と **サービス用 IP アドレス（ip0）** を、あるノードから別のノードへ「引き渡す（ハンドオフする）」ことで、**TCP の接続試行が正しく完了するか**を確認する事前実験を行った。

本記事では、想定している仕様、実験構成、そして得られた知見を説明する。



## 想定設計（意図した仕様）

### 役割

* **常時稼働フロント（プロキシ／スタンバイ）**
  常に起動しており、WoL を発行できる。対象マシンがスリープ中の間、一時的にサービスの識別情報（MAC/IP）を保持する。
* **対象マシン（省電力）**
  通常はサスペンド状態。起動されたらサービス識別情報を引き継ぎ、リクエストの処理を開始する。
* **クライアント**
  常に固定された宛先 `ip0:PORT` に接続する。対象マシンがスリープ中か起動中かを意識しない。

### 中核となるアイデア

専用の「サービス識別情報」を用意する。

* **サービス IP**: `ip0`（例: `10.200.0.100`）
* **サービス MAC**: `mac0`（例: `02:00:00:00:00:64`）

そして、この IP/MAC を **スタンバイ側と実サーバ側の間で移動（ハンドオフ）**させる。

これが成立すれば、クライアントは常に `ip0:PORT` に接続するだけでよく、バックエンドは現在トラフィックを処理すべきマシンへ動的に識別情報を移すことができる。



## なぜ自明ではないのか（成立させるための前提条件）

この方式は、以下の挙動が確実であることに依存する。

1. **一意性**: 任意の時点で `ip0/mac0` を保持するノードは必ず 1 台だけであること。
2. **起動中の TCP 挙動**: 対象マシンが起動中の間、クライアントは即座に失敗するのではなく、理想的には「待つ」こと。
3. **L2/L3 の収束**: スイッチ／ブリッジおよびクライアントが、「mac0 がどこにあるか（FDB 学習）」や「ip0 に対応する MAC は何か（ARP キャッシュ）」を正しく更新すること。

本事前実験は、これらの制約を制御された環境で検証することを目的としている。



## 事前実験：Linux ネットワーク名前空間によるネットワークのシミュレーション

観測可能かつ再現性のある形で実験するため、Linux の `network namespace` を使って 3 台のホストをシミュレートした。

* `A`, `B`, `C` はそれぞれ独立した名前空間（別マシンのように振る舞う）
* `veth` ペアで各名前空間を共有 Linux ブリッジ `br-abc`（単一の L2 セグメント）に接続
* 「サービス識別情報」（`mac0` + `ip0`）は `macvlan` を介して付与
* クライアント名前空間 `C` が `ip0:8080` への TCP 接続を開始
* その途中で、サービス識別情報を `A` から `B` に移動
* `B` はハンドオフ完了後にサーバ（`nc -l`）を起動して接続を受け入れる

概念図：

```
          (root namespace)
               br-abc (bridge)
           /        |        \
      vethA-br  vethB-br  vethC-br
         |         |         |
       [A]       [B]       [C]
       vethA     vethB     vethC

サービス識別情報 ip0/mac0 は macvlan（macv0）で付与される。
最初は A が保持し、その後 B に移動する。
```



## 実験スクリプト（要点）

以下は実験で使用したスクリプト（原文のまま）。流れは次の通り。

1. 名前空間、veth ペア、Linux ブリッジを作成
2. `A`、`B`、`C` に固定 IP を割り当て
3. ARP フラックスを避けるための sysctl 設定
4. `A` に `macvlan` を作成し、`mac0` と `ip0` を設定
5. `A` で `ip0:8080` への TCP を DROP（応答しない）
6. `C` から `ip0:8080` への TCP 接続試行を開始
7. `B` でサーバを起動
8. 少し待機
9. `A` の `macvlan` を削除
10. `B` に同じ `mac0` と `ip0` を持つ `macvlan` を作成
11. クライアントの接続試行が完了することを確認

（スクリプト本体は省略せず、原文どおり掲載）

```bash
#!/usr/bin/env bash
set -euo pipefail
set -x

BR=br-abc
MAC0="02:00:00:00:00:64"
IP0_ADDR="10.200.0.100"
IP0_CIDR="10.200.0.100/24"
PORT=8080

# cleanup
ip netns del A 2>/dev/null || true
ip netns del B 2>/dev/null || true
ip netns del C 2>/dev/null || true
ip link del $BR 2>/dev/null || true

ip netns add A
ip netns add B
ip netns add C

ip link add $BR type bridge
ip link set $BR up

ip link add vethA type veth peer name vethA-br
ip link set vethA netns A
ip link set vethA-br master $BR
ip link set vethA-br up

ip link add vethB type veth peer name vethB-br
ip link set vethB netns B
ip link set vethB-br master $BR
ip link set vethB-br up

ip link add vethC type veth peer name vethC-br
ip link set vethC netns C
ip link set vethC-br master $BR
ip link set vethC-br up

ip netns exec A ip link set lo up
ip netns exec A ip link set vethA up
ip netns exec A ip addr add 10.200.0.11/24 dev vethA

ip netns exec B ip link set lo up
ip netns exec B ip link set vethB up
ip netns exec B ip addr add 10.200.0.12/24 dev vethB

ip netns exec C ip link set lo up
ip netns exec C ip link set vethC up
ip netns exec C ip addr add 10.200.0.13/24 dev vethC

for ns in A B; do
  ip netns exec $ns sysctl -w \
    net.ipv4.conf.all.arp_ignore=1 \
    net.ipv4.conf.default.arp_ignore=1 \
    net.ipv4.conf.all.arp_announce=2 \
    net.ipv4.conf.default.arp_announce=2 >/dev/null
done

# A: macvlan + ip0/mac0
ip netns exec A ip link add macv0 link vethA type macvlan mode bridge
ip netns exec A ip link set macv0 address $MAC0
ip netns exec A ip addr add $IP0_CIDR dev macv0
ip netns exec A ip link set macv0 up

# A: drop ip0:8080 (no SYN-ACK / no RST)
if ip netns exec A iptables -V >/dev/null 2>&1; then
  ip netns exec A iptables -I INPUT -d $IP0_ADDR -p tcp --dport $PORT -j DROP
else
  ip netns exec A nft -f - <<NFT
table inet filter {
  chain input {
    type filter hook input priority 0;
    policy accept;
    ip daddr $IP0_ADDR tcp dport $PORT drop
  }
}
NFT
fi

# C: start curl
ip netns exec C nc -v -z $IP0_ADDR $PORT &
# ip netns exec C bash -c "curl -vvv --retry 10 $IP0_ADDR:$PORT" &
CURL_PID=$!

# B: start server
ip netns exec B nc -v -l -p $PORT  &
# ip netns exec B sh -c "python -m http.server $PORT"  &
SERVER_PID=$!

sleep 10

# A: delete macvlan
ip netns exec A ip link del macv0

# B: macvlan + ip0/mac0
ip netns exec B ip link add macv0 link vethB type macvlan mode bridge
ip netns exec B ip link set macv0 address $MAC0
ip netns exec B ip addr add $IP0_CIDR dev macv0
ip netns exec B ip link set macv0 up

# trigger FDB relearn (optional; commented out)
# ip netns exec B ping -c 1 -I macv0 10.200.0.13 >/dev/null 2>&1 || true
# ip netns exec B sh -c "arping -A -c 1 -I macv0 $IP0_ADDR || true" &

wait $CURL_PID
echo "DONE (server log: /tmp/httpserver-b.log)"

kill -s INT $SERVER_PID
```



## この実験が示していること

### 1) 「Floating identity」（ip0/mac0）は L2/L3 で成立する

`ip0/mac0` を `A` から削除して `B` に付与すると、ネットワークは収束し、`ip0` 宛のトラフィックが `B` に到達する。これはオンデマンド起動型アーキテクチャの基盤となる挙動である。

### 2) DROP を使うことでクライアントの即時失敗を防げる

ホストが IP を持っているが、その `PORT` で待ち受けていない場合、通常は SYN に対して **RST** が返り、「Connection refused」となりやすい。

本実験では、`A` が意図的に `ip0:8080` への TCP を **DROP** しているため、

* SYN-ACK も返らない
* RST も返らない

という状態になる。その結果、クライアント側は（TCP の再送、あるいは `curl --retry` のようなアプリケーションレベルの再試行によって）`B` が引き継いでサービスを開始するまで待ち続けることができる。

これは「起動中は即失敗させず、準備が整うまで待たせる」という本番環境で求められる挙動に合致する。

### 3) ARP フラックスの制御が必須

以下の sysctl 設定：

* `arp_ignore=1`
* `arp_announce=2`

は、複数インターフェースが存在する場合の誤った ARP 応答や送信元 IP の誤通知を抑制する。Floating IP/MAC 設計では特に重要であり、ベースライン要件として扱うのが妥当である。



## この設計から導かれる運用要件

本実験から、実運用レベルの設計では以下を明示的な要件とすべきことが分かる。

### 要件 A: ip0/mac0 の単一所有

任意の時点で、サービス識別情報を所有するノードは 1 台のみでなければならない。引き継ぎ手順は必ず次の順序とする。

1. 旧オーナーが識別情報を解放
2. 新オーナーが識別情報を取得

これを守らないと ARP 競合や非決定的な配送が発生する。

### 要件 B: 起動中の挙動を定義する（DROP か即失敗か）

「準備ができるまで待つ」セマンティクスを実現したい場合、スタンバイ側は RST を生成すべきではない。実装上は次のようになる。

* 起動ウィンドウ中は `ip0:PORT` への TCP を DROP する、または
* クライアントの挙動に整合した制御応答（プロキシ、キュー、再試行を促す明示的エラーなど）を返す

本実験は、DROP ベースのアプローチが有効であることを示している。

### 要件 C: ハンドオフ後に L2/L3 キャッシュ収束を強制する

実ネットワークでは、MAC 学習テーブル（FDB）や ARP キャッシュの更新に時間がかかることがある。堅牢な実装では、引き継ぎ直後に **Gratuitous ARP (GARP)** を送信するのが一般的である。

例：

* `arping -A -I <iface> <ip0>`

スクリプト内ではコメントアウトされているが、本番では標準手順として組み込むべきである。



## Suspend + WoL システムへの対応付け

名前空間を実システムに対応付けると以下のようになる。

* **A（スタンバイ／フロント）**: 常時稼働ノード

  * 対象がサスペンド中は `ip0/mac0` を保持
  * 即失敗を防ぐため `ip0:PORT` への TCP を DROP
  * 需要を検知したら WoL を送信
* **B（対象マシン）**: サスペンドされるマシン

  * 起動／復帰時に `ip0/mac0` を取得
  * GARP を送信し、サービスを開始
* **C（クライアント）**: 変更なし

  * 常に `ip0:PORT` に接続

これにより、「クライアントは常に安定したエンドポイントを使用し、バックエンドは動的に起動して引き継ぐ」という明確な契約が成立する。



## 次の検証項目

本事前実験は機能的な正しさを示したに過ぎない。本番対応に向けては、以下の検証が必要である。

* 起動レイテンシの分布（Suspend → リンクアップ → サービス開始）
* クライアントの再試行戦略とタイムアウト
* 実スイッチでの挙動（MAC 学習、ポートセキュリティ、エージング）
* IPv6 への影響（NDP、DAD、デュアルスタック時のバインディング）
* 既存接続の扱い（本実験は新規接続のみを対象）



## まとめ

本実験は、**固定されたサービス IP/MAC（ip0/mac0）をノード間で移動させる**ことで、引き継ぎ後に TCP 接続試行が成功し得ることを示した。これは **Suspend + WoL によるオンデマンドアクセス** アーキテクチャの重要な構成要素である。

設計上の要点は次の通り。

* サービス識別情報の単一所有を厳密に保証すること
* 起動中は即失敗させない（DROP ベースの待機セマンティクス）
* 引き継ぎ後にキャッシュ収束（GARP）を行うこと
* ARP フラックスを防ぐための設定を行うこと

これらの要件を明確化することで、名前空間レベルのシミュレーションから実機実装へと設計を進めることができる。
