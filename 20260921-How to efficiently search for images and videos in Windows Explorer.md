---
pubDatetime: 2026-09-21T10:33:00+09:00
title: "Windowsエクスプローラーで画像・動画を効率よく検索する方法｜`*.jpg` と `kind:` の使い方"
description: "`*.jpg OR *.png OR *.heic` のように指定すると、複数の画像形式をまとめて検索できます。また、`kind:picture` を使えば画像全般、`kind:video` なら動画全般を拡張子を意識せず検索できます。"
---

Windowsのエクスプローラーでは、検索欄に条件を入力することで、特定の拡張子や種類のファイルを効率よく探せます。

たとえば、

* JPG・PNG・HEICをまとめて探したい
* PC内にある画像を一通り探したい
* 写真と動画をまとめて検索したい
* デジタルカメラのRAWファイルも探したい

といった場合に便利です。

この記事では、画像ファイルを中心に、Windowsエクスプローラーで使える検索方法を紹介します。

---

## JPG・PNG・HEICをまとめて検索する

特定の拡張子を検索する場合は、ワイルドカード `*` を使って、

```text id="acv86d"
*.jpg
```

のように指定できます。

JPG、PNG、HEICをまとめて探す場合は、`OR` でつなぎます。

```text id="ig63wp"
*.jpg OR *.png OR *.heic
```

`OR` は「または」という意味なので、

> JPG、PNG、HEICのいずれかに該当するファイル

を検索できます。

JPEGも対象にするなら、

```text id="19yynk"
*.jpg OR *.jpeg OR *.png OR *.heic
```

とします。

---

## 画像にはどんな拡張子がある？

画像ファイルには、JPGやPNG以外にもさまざまな形式があります。

| 拡張子               | 主な用途                       |
| ----------------- | -------------------------- |
| `.jpg` / `.jpeg`  | デジカメやスマートフォンなどの一般的な写真      |
| `.png`            | スクリーンショット、透過画像             |
| `.heic` / `.heif` | iPhoneなどで使われる高効率画像形式       |
| `.webp`           | Webサイトでよく使われる画像            |
| `.gif`            | Web画像、アニメーション              |
| `.bmp`            | Windowsで古くから使われる画像形式       |
| `.tif` / `.tiff`  | スキャン、印刷、高画質画像              |
| `.avif`           | 高圧縮の比較的新しい画像形式             |
| `.jxl`            | JPEG XL                    |
| `.dng`            | Adobe系やスマートフォンなどで使われるRAW形式 |
| `.cr2` / `.cr3`   | CanonのRAW                  |
| `.nef`            | NikonのRAW                  |
| `.arw`            | SonyのRAW                   |
| `.orf`            | OM SYSTEM / OlympusのRAW    |
| `.rw2`            | PanasonicのRAW              |
| `.raf`            | FujifilmのRAW               |

---

## 一般的な画像をまとめて検索する

写真、スクリーンショット、Webから保存した画像などを幅広く探したい場合は、次のように検索できます。

```text id="2ep8br"
*.jpg OR *.jpeg OR *.png OR *.heic OR *.heif OR *.webp OR *.gif OR *.bmp OR *.tif OR *.tiff OR *.avif
```

これで、一般的な画像形式をかなり広くカバーできます。

---

## カメラのRAWファイルも検索する

デジタルカメラで撮影したRAWファイルも含めたい場合は、さらにRAW形式の拡張子を追加します。

```text id="8a91h8"
*.jpg OR *.jpeg OR *.png OR *.heic OR *.heif OR *.webp OR *.gif OR *.bmp OR *.tif OR *.tiff OR *.avif OR *.dng OR *.raw OR *.cr2 OR *.cr3 OR *.nef OR *.arw OR *.orf OR *.rw2 OR *.raf
```

写真データをPCや外付けドライブからまとめて発掘したい場合などに使えます。

ただし、これだけ多くの拡張子を毎回指定するのは面倒です。

そこで便利なのが `kind:` を使った検索です。

---

## `kind:picture` で画像全般を検索する

Windowsエクスプローラーでは、拡張子ではなくファイルの「種類」を条件に検索することもできます。

画像を検索する場合は、

```text id="i26rcn"
kind:picture
```

と入力します。

Windowsが画像として認識しているファイルをまとめて検索できるため、拡張子を一つずつ列挙する必要がありません。

そのため、

**特定の画像形式を探す場合は `*.jpg` などを指定し、画像全般を探す場合は `kind:picture` を使う**

という使い分けが便利です。

---

## `kind:` には画像以外も指定できる

`kind:` は画像だけでなく、動画や音楽、文書などの検索にも利用できます。

代表的な指定方法は次のとおりです。

| 検索条件                 | 対象          |
| -------------------- | ----------- |
| `kind:picture`       | 画像          |
| `kind:video`         | 動画          |
| `kind:music`         | 音楽・音声       |
| `kind:document`      | 文書          |
| `kind:folder`        | フォルダー       |
| `kind:program`       | プログラム・アプリ   |
| `kind:email`         | メール         |
| `kind:calendar`      | カレンダー項目     |
| `kind:contact`       | 連絡先         |
| `kind:communication` | 通信・メッセージ関連  |
| `kind:link`          | ショートカット・リンク |
| `kind:task`          | タスク         |
| `kind:note`          | ノート         |
| `kind:journal`       | ジャーナル項目     |
| `kind:feed`          | RSSなどのフィード  |
| `kind:game`          | ゲーム         |
| `kind:recordedtv`    | 録画テレビ       |

利用できる検索条件や実際に分類されるファイルは、Windowsのバージョンや環境などによって異なる場合があります。

---

## 写真と動画をまとめて検索する

`OR` は `kind:` に対しても利用できます。

画像と動画をまとめて検索する場合は、

```text id="u0jv3m"
kind:picture OR kind:video
```

とします。

スマートフォンやデジタルカメラから取り込んだ写真・動画をまとめて探したい場合などに便利です。

---

## `*.jpg` と `kind:picture` はどう使い分ける？

### 特定の画像形式だけ探したい

拡張子がわかっている場合は、

```text id="o7u0a3"
*.jpg
```

のように検索します。

複数の形式なら、

```text id="ob3pua"
*.jpg OR *.png OR *.heic
```

のように `OR` でつなぎます。

この方法なら、「HEICだけ」「JPGとPNGだけ」といった細かな指定ができます。

### 画像形式を問わず探したい

どんな拡張子なのかわからない場合は、

```text id="0xznvu"
kind:picture
```

が便利です。

画像の拡張子をすべて把握していなくても、Windowsが画像として分類しているファイルを検索できます。

---

## 目的別の検索条件まとめ

### JPGだけ

```text id="x1x60r"
*.jpg
```

### JPGとJPEG

```text id="ymh5g4"
*.jpg OR *.jpeg
```

### JPG・PNG・HEIC

```text id="iy5db1"
*.jpg OR *.png OR *.heic
```

### よく使われる画像形式をまとめて検索

```text id="uwn5kq"
*.jpg OR *.jpeg OR *.png OR *.heic OR *.heif OR *.webp OR *.gif OR *.bmp OR *.tif OR *.tiff OR *.avif
```

### 画像全般

```text id="4ol8ic"
kind:picture
```

### 動画全般

```text id="wkj1aa"
kind:video
```

### 画像と動画

```text id="42ynp0"
kind:picture OR kind:video
```

### 音楽・音声

```text id="wxf2r9"
kind:music
```

### 文書

```text id="cbntz9"
kind:document
```

---

## まとめ

Windowsエクスプローラーでは、検索欄に条件を入力することで目的のファイルを効率よく探せます。

覚えておくと便利なのは、次の3つです。

* 特定の拡張子を検索する → `*.jpg`
* 複数の条件を検索する → `OR`
* ファイルの種類で検索する → `kind:picture` など

たとえば、JPG・PNG・HEICだけを探すなら、

```text id="93afue"
*.jpg OR *.png OR *.heic
```

画像全般を探すなら、

```text id="7zjzve"
kind:picture
```

画像と動画をまとめて探すなら、

```text id="8h4u6p"
kind:picture OR kind:video
```

となります。

「拡張子がわかっているなら `*.拡張子`、大まかなファイルの種類しかわからないなら `kind:`」と覚えておくと、エクスプローラーの検索を使い分けやすくなります。
