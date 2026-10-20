---
title: "Tampermonkeyで特定のWebサイトをダークモード化する方法【OS設定に自動連動】"
description: "特定のWebサイトだけをOSやブラウザのダークモード設定に自動連動させて色反転できるTampermonkeyユーザースクリプトの作成方法を紹介。CSSのfilterプロパティとprefers-color-schemeを利用する。"
pubDatetime: 2026-10-20T12:36:00+09:00
---

ダークモードに対応していないWebサイトを、ブラウザの設定に合わせて自動的にダークモード化したいと思ったことはありませんか？

最近はダークモードに対応したWebサイトも増えていますが、まだライトモードしか用意されていないサイトも少なくありません。

そこで今回は、ブラウザ拡張機能の **Tampermonkey** を使い、指定したWebサイトにだけダークモードを適用するユーザースクリプトを紹介します。

この方法には、次のような特徴があります。

- OS・ブラウザのダークモード設定に自動連動する
- ダークモード時だけWebページの色を反転する
- 複数のURLを指定し、対象サイトを限定できる
- JavaScriptによる複雑な処理が不要
- ライトモードに戻すと、自動的に色反転も解除される

## 1. 仕組み

今回の方法では、CSSの `filter` プロパティを使用してWebページ全体の色を反転します。

使用するCSSは次のとおりです。

```css
html {
    filter: invert(1) hue-rotate(180deg) !important;
}
```

`invert(1)` は色を反転するCSSフィルターです。白い背景は黒に、黒い文字は白に変わります。

ただし、単純に色を反転すると、青が黄色になるなど、色相も大きく変わってしまいます。

そこで `hue-rotate(180deg)` を組み合わせ、色相を180度回転させることで、元の色合いに近づけています。

さらに、CSSの `prefers-color-scheme` メディアクエリを使います。

```css
@media (prefers-color-scheme: dark) {
    html {
        filter: invert(1) hue-rotate(180deg) !important;
    }
}
```

これにより、ブラウザがダークモードを使用している場合にのみフィルターが適用されます。

## 2. Tampermonkeyをインストール

まず、ブラウザにTampermonkeyをインストールします。

Tampermonkeyは、任意のWebサイト上で独自のJavaScriptを実行できるブラウザ拡張機能です。

公式サイト：

https://www.tampermonkey.net/

インストール後、Tampermonkeyのメニューから「新規スクリプトを追加」を選択します。

## 3. ユーザースクリプト

以下のコードを、新規スクリプトの編集画面に貼り付けて保存します。

```javascript
// ==UserScript==
// @name         Auto Dark Mode (Invert)
// @namespace    https://tampermonkey.net/
// @version      1.1
// @description  Apply invert filter on selected sites in dark mode
// @match        https://example.com/*
// @match        https://example.org/*
// @match        https://sub.example.net/*
// @run-at       document-start
// @grant        GM_addStyle
// ==/UserScript==

(function () {
    'use strict';

    GM_addStyle(`
        html {
          background-color: #fff !important;
        }
        @media (prefers-color-scheme: dark) {
            html {
                filter: invert(1) hue-rotate(180deg) !important;
            }
        }
    `);
})();
```

このスクリプトでは、`GM_addStyle()` を使用してCSSをページに追加しています。

`@run-at document-start` を指定することで、ページの読み込み初期段階からスタイルを適用し、表示後に色が切り替わる現象をできるだけ抑えています。

## 4. 適用するURLを指定する

このスクリプトでは、Tampermonkeyの `@match` を利用して、ダークモード化するWebサイトを指定しています。

例えば次の部分です。

```javascript
// @match        https://example.com/*
// @match        https://example.org/*
// @match        https://sub.example.net/*
```

`@match` は複数行記述できるため、対象のWebサイトを自由に追加できます。

指定方法の例を紹介します。

| 記述方法 | 適用対象 |
|---|---|
| `https://example.com/*` | example.comの全ページ |
| `https://*.example.com/*` | example.comとそのサブドメイン |
| `https://example.com/docs/*` | docs以下のページ |
| `*://example.com/*` | HTTP・HTTPSの両方 |
| `https://example.com/page` | 特定のページ |

例えば、GitHubとStack Overflowを対象にする場合は、次のように記述します。

```javascript
// @match        https://github.com/*
// @match        https://stackoverflow.com/*
```

すべてのWebサイトではなく、**ダークモードに対応していないサイトだけを指定する** のがおすすめです。

## 5. 背景色を白に指定する理由

今回のスクリプトでは、色反転の設定に加えて、次のCSSも指定しています。

```css
html {
    background-color: #fff !important;
}
```

これは、HTML要素の背景色を明示的に白へ設定するためです。

色反転フィルターを使用すると、白い背景は黒く表示されます。

一方、背景が透明なページでは、期待したとおりの背景色にならない場合があります。

そこで、あらかじめ `html` の背景色を白に設定することで、ダークモード時に黒い背景として表示されるようにしています。

なお、この背景色指定はメディアクエリの外側にあるため、ライトモード時にも適用されます。

サイトによっては本来の背景色を上書きしてしまう可能性があるので、必要に応じて調整してください。

## 6. OSのダークモード設定と自動連動

このスクリプトの便利な点は、OSやブラウザのカラースキームに自動連動することです。

例えば、WindowsやmacOSでダークモードを有効にすると、`prefers-color-scheme: dark` が一致し、色反転フィルターが適用されます。

反対に、ライトモードへ戻すとフィルターの適用が解除されます。

JavaScriptの `matchMedia()` で設定変更を監視する必要もありません。

CSSのメディアクエリだけで動作するため、構成がシンプルです。

## 7. 注意点

手軽にダークモード化できる方法ですが、いくつか注意点もあります。

### 画像や動画も色が変わる

`html` 要素全体にフィルターを適用しているため、文字や背景だけでなく、画像や動画も処理の対象になります。

`hue-rotate(180deg)` により色相の変化はある程度補正されますが、元の画像と完全に同じ色になるわけではありません。

特に写真などでは、色や明るさが不自然に見える場合があります。

### 既にダークモードに対応しているサイトには不向き

Webサイトが独自のダークモードを提供している場合、もともと暗い背景が反転され、逆に明るく表示される可能性があります。

そのため、ダークモード未対応のWebサイトを中心に `@match` で指定するのが適切です。

### サイトのデザインによっては表示が崩れる

CSSのフィルターはページ全体の描画に影響します。

サイトによっては、固定配置の要素、ポップアップ、背景画像などの表示に問題が生じる場合があります。

また、iframe内のコンテンツには別途対応が必要になる場合もあります。

### `background-color` は常に適用される

今回のスクリプトでは、ライトモードでも `html` の背景色が白に固定されます。

通常の表示を優先したい場合は、この指定自体をダークモード用のメディアクエリの中に移動する方法もあります。

## まとめ

TampermonkeyとCSSの `prefers-color-scheme` を組み合わせることで、ダークモード非対応のWebサイトを簡単にダークモード化できます。

今回の方法は、CSSの `filter` による簡易的なダークモード化であり、サイト本来の配色を個別に調整するものではありません。

そのため、画像の色や一部のUIに問題が生じる場合はありますが、背景色や文字色を手作業で設定する必要がない点は大きなメリットです。

特に、**普段利用する特定のWebサイトだけを、OSの設定に合わせて自動的にダークモード化したい** という用途に向いています。

既存サイトのCSSを大幅に書き換えることなく、数行のCSSとTampermonkeyだけで実現できるので、ダークモード未対応のサイトで試してみてください。
