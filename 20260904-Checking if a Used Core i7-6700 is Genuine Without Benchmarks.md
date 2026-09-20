---
title: "中古のCore i7-6700が本物か、ベンチマークを使わずに確認してみた"
description: "今回確認するCore i7-6700、FPOとは？、ATPOとは？を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-09-04T04:07:06.484Z
updatedDate: 2026-09-04T04:13:02.754Z
---

中古でIntel CPUを入手したとき、「ヒートスプレッダだけ別のCPUに交換されていないか」「型番を書き換えた偽物ではないか」が気になることがあります。

今回は実際の**Intel Core i7-6700**を例に、性能測定を使わず、

* ヒートスプレッダの刻印
* S-Spec
* FPO（Batch Number）
* ATPO（Serial Number）
* Data Matrix
* Partial ATPO

を使って、どこまで真贋を確認できるのか試してみました。

## 今回確認するCore i7-6700

CPU表面には次のように刻印されています。

```text
i7-6700
SR2L2 3.40GHZ
X012C345 (e4)
```

まず、それぞれの意味を整理します。

| 表示       | 意味                |
| -------- | ----------------- |
| i7-6700  | CPUの製品名           |
| SR2L2    | S-Spec            |
| 3.40GHZ  | ベースクロック           |
| X012C345 | FPO（Batch Number） |
| (e4)     | ATPOではない          |

Intel公式情報でも、Core i7-6700の製品版にはS-Spec `SR2L2`、R0ステッピングが存在します。また、i7-6700のベースクロックは3.40GHzです。

したがって、ここまでの表面刻印には矛盾がありません。

## FPOとは？

FPOは「Finished Process Order」の略で、IntelではBatch Numberとして扱われています。

今回のCPUでは、

```text
X012C345
```

がFPOだとします。

FPOはヒートスプレッダ上にあります。

ここで重要なのは、`SR2L2`とは別物ということです。

```text
SR2L2    → S-Spec
X012C345 → FPO
```

IntelのWarranty InformationでもFPOは「Batch number」として入力します。

## ATPOとは？

ATPOは「Assembly Test Process Order」の略で、CPUのSerial Numberとして扱われています。

FPOがバッチ番号なのに対して、ATPOは個々のプロセッサを識別するための番号です。

Intel CPUでは、基板外周に非常に小さな**2D Matrix**があります。

この2D Matrixをデコードすると、Full ATPO（完全なシリアル番号）を取得できます。

## 2D Matrixをスマートフォンで読み取る

今回、iPhoneではApp Storeの「Code Scan - QR & Barcode Scanner」を使用すると読み取ることができました。

https://apps.apple.com/jp/app/code-scan-qr-barcode-scanner/id1554812545

Intel CPUのコードは非常に小さいため、通常のQRコードリーダーではうまく認識されない場合があります。Intelも、スマートフォンのカメラを利用する対応アプリで2D Matrixを読み取れると案内しています。

読み取り時は、

* CPU表面の汚れを落とす
* 明るい場所で撮影する
* 反射する場合は光を斜めから当てる
* コードにピントを合わせる
* 必要ならマクロ撮影する
* **2D Matrix以外の部分を暗めの紙や物で隠し、コード部分と基板とのコントラストを高くする**

と成功しやすくなります。

特に今回試した範囲では、**コード以外の部分を暗いもので覆い、2D Matrixだけが目立つ状態にすると認識しやすくなりました。**

CPU基板には文字や配線パターン、金属部分などがあり、カメラから見ると2D Matrix以外にも細かな模様が多数存在します。そのため、周囲を暗く隠して読み取らせたいコードだけを目立たせる方法は、スキャンが安定しない場合に試す価値があります。

今回のi7-6700では、この方法も併用しながらiPhoneでスキャンしたところ、アプリから次のような結果が得られました。

```text
U6SG012345678
```

必要なのは`Content`側です。

したがって、このCPUのFull ATPOは、

```text
U6SG012345678
```

となります。

なお、`Hex`は別のシリアル番号ではありません。

例えば、

```text
55 → U
36 → 6
53 → S
47 → G
```

というように、Contentの文字列を16進数で表現したものです。

## Partial ATPOを探す

次に、CPU基板の外周をよく確認します。

今回のCPUには、

```text
45678
```

という非常に小さな文字が印刷されていました。

これがPartial ATPOです。

IntelによるとPartial ATPOは、Full ATPOの末尾3～5文字です。

そこで先ほどData Matrixから読み取ったFull ATPOと比較します。

```text
Full ATPO
U6SG012345678
        ↓↓↓↓↓
        45678

基板上のPartial ATPO
45678
```

**末尾5文字が完全に一致しました。**

これは重要なチェックポイントです。

Intel自身も偽造CPUを確認する手順として、2D MatrixからFull ATPOを読み取り、その末尾5文字とCPU外周に印刷されたPartial ATPOを比較し、正規品なら一致することを確認するよう案内しています。

つまり今回のCPUでは、

```text
Data Matrix
     ↓
U6SG012345678
        ↓
      45678
        ↑
基板上の「45678」
```

という対応が確認できました。

少なくとも、**基板上のData Matrixと人間が読めるPartial ATPOは正しく対応しています。**


## ヒートスプレッダと基板は直接照合できるのか？

ここが今回調べていて特に気になった部分です。

ヒートスプレッダ側には、

```text
i7-6700
SR2L2
X012C345
```

があります。

一方、基板側には、

```text
Full ATPO:
U6SG012345678

Partial ATPO:
45678
```

があります。

整理すると、

```text
【ヒートスプレッダ】
i7-6700
SR2L2
FPO: X012C345
       │
       │ ← 本来この対応を確認したい
       │
【CPU基板】
Full ATPO: U6SG012345678
Partial ATPO: 45678
```

という構造です。

Full ATPOとPartial ATPOについては、

```text
U6SG012345678
        ↓↓↓↓↓
        45678
```

と物理的に確認できます。

しかし、

```text
FPO X012C345
      ↕
ATPO U6SG012345678
```

について、ユーザーが番号から計算して確認できる公開された規則は見当たりません。

IntelのWarranty Informationが正常に検索できれば、FPOとATPOを組み合わせて確認できますが、今回のi7-6700では検索結果が得られませんでした。

そのため、**ヒートスプレッダが本当にこの基板に元々取り付けられていたものなのかを、刻印だけで100％証明するところまでは到達できませんでした。**

## 「別CPUにi7-6700のヒートスプレッダを載せた偽物」はどう確認する？

例えば、

```text
別のCPU基板
   ＋
i7-6700と書かれたヒートスプレッダ
```

という偽装を考えます。

この場合、表面を見るだけではi7-6700に見えてしまいます。

そこで有効なのがCPU内部の識別情報です。

Intel自身も、CPUをPCに搭載できる場合は**Intel Processor Diagnostic Tool**を使用し、

```text
Genuine Intel: Pass
```

となること、および製品名が一致することを確認する方法を案内しています。

さらにCPU-ZやHWiNFOなどで、

* Core i7-6700
* Skylake
* 4コア8スレッド
* 8MB L3
* R0ステッピング
* CPUIDがSkylakeとして整合

などを確認すれば、「ヒートスプレッダにはi7-6700と書いてあるが、中身は別CPU」という単純な偽装はかなり見抜きやすくなります。

これはベンチマークのスコア比較とは異なり、CPU自身が返す識別情報を確認する方法です。

## 今回の結果

今回のCore i7-6700では、

| チェック                 | 結果           |
| -------------------- | ------------ |
| 表面にi7-6700と刻印        | ○            |
| S-Spec `SR2L2`       | Intel公式情報と一致 |
| 3.40GHz              | Intel公式仕様と一致 |
| FPO `X012C345`       | 確認           |
| 2D Matrix            | 読み取り成功       |
| Full ATPO            | 取得成功         |
| Partial ATPO         | `45678`      |
| Full ATPO末尾          | `45678`      |
| Full/Partial ATPO    | **5文字完全一致**  |
| Warranty Information | 製品を発見できず     |
| FPOとATPOのIntel DB照合  | **未確認**      |

したがって、今回確認できた範囲では、**正規のCore i7-6700と矛盾する情報は見つかりませんでした。**

特に、

**Data Matrixから取得したFull ATPOの末尾5文字と、基板に直接印刷されたPartial ATPOが一致した**

ことは強い確認材料です。

ただし、Intel Warranty InformationでFPOとATPOの組み合わせを確認できなかったため、

**「ヒートスプレッダと基板が工場出荷時から同じ組み合わせだった」ことまで完全に証明できたわけではありません。**

## 中古Intel CPUを確認するときの手順まとめ

性能測定を使わず確認するなら、次の順番が実用的だと思います。

```text
① CPU表面の型番を確認
        ↓
② S-SpecをIntel公式情報と照合
        ↓
③ FPOを記録
        ↓
④ 基板外周の2D Matrixをスキャン
        ↓
⑤ Full ATPOを取得
        ↓
⑥ 基板上のPartial ATPOを探す
        ↓
⑦ Full ATPO末尾3～5文字と比較
        ↓
⑧ Intel Warranty Informationに
   FPO + ATPOを入力
        ↓
⑨ PCに搭載できるなら
   Intel Processor Diagnostic Toolで
   Genuine Intelと製品名を確認
```

特に④～⑦は、CPUを動作させなくても確認できます。

## 注意：ATPOは公開しない方がよい

ATPOはCPUのSerial Numberです。

ブログやフリマサイトなどで写真を公開する場合は、

```text
U6SG0123*****
```

のように一部を隠しておいた方がよいでしょう。

Data Matrixそのものを鮮明な写真で公開すると、文字を隠していてもコードからFull ATPOを読み取れる可能性があるため、**Data Matrixにもモザイクをかける**のが安全です。

## まとめ

Intel CPUには、単に「Core i7」と書かれているだけではなく、

**S-Spec → FPO → Data Matrix → Full ATPO → Partial ATPO**

という複数の識別情報があります。

今回のi7-6700では、

```text
SR2L2
FPO: X012C345
Full ATPO: U6SG0123*****
Partial ATPO: 45678
```

まで確認でき、Full ATPOとPartial ATPOの対応も確認できました。

中古CPUの真贋確認ではベンチマーク結果だけを見るのではなく、こうした**物理的なトレーサビリティ情報を相互照合する方法も有効**です。

ただし、これだけで半導体の真正性を暗号学的に証明できるわけではありません。特に古いCPUではIntelの現行Warranty検索で結果が得られない場合もあるため、

**物理刻印の整合性 + ATPOの整合性 + CPU内部の識別情報**

を組み合わせて判断するのが現実的です。
