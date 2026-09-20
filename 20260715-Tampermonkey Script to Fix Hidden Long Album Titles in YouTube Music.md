---
pubDatetime: 2026-07-15T15:31:15+09:00
title: "YouTube Musicで長いアルバムタイトルが見えなくなる問題を修正するTampermonkeyスクリプト"
description: "YouTube Musicでは、アルバムタイトルが長い場合にタイトル要素自体は存在しているにもかかわらず、画面上では表示されなくなることがあります。 このTampermonkeyスクリプトは、その表示不具合をCSSの上書きによって修正し、長いアルバムタイトルが正しく表示されるようにします。 このスク…"
---

YouTube Musicでは、アルバムタイトルが長い場合にタイトル要素自体は存在しているにもかかわらず、画面上では表示されなくなることがあります。

このTampermonkeyスクリプトは、その表示不具合をCSSの上書きによって修正し、長いアルバムタイトルが正しく表示されるようにします。

## このスクリプトでできること

* 長いアルバムタイトルが消えてしまう問題を修正する
* タイトルを2行まで表示し、それ以降は省略表示する
* YouTube Music本来のレイアウトをできるだけ維持する

通常のタイトルには影響を与えず、長いタイトルだけを適切に表示できるようになります。

## 使い方

1. ブラウザにTampermonkeyをインストールします。
2. 新しいユーザースクリプトを作成します。
3. スクリプトを貼り付けて保存します。
4. YouTube Musicを再読み込みします。

ローカルのMHTMLファイルに適用したい場合は、Tampermonkeyの設定で**「ファイルのURLへのアクセスを許可する」**を有効にしてください。

## 修正内容

このスクリプトは、タイトル表示に使用されるCSSを上書きし、タイトルそのものに2行までの省略表示を適用します。

その結果、長いアルバムタイトルが表示領域の外へ押し出されて見えなくなる問題を防ぐことができます。

## まとめ

このTampermonkeyスクリプトを利用することで、YouTube Musicで長いアルバムタイトルが非表示になってしまう問題を簡単に改善できます。

インストール後は特別な操作は不要で、通常どおりYouTube Musicを利用するだけで修正内容が適用されます。

```javascript
// ==UserScript==
// @name         YouTube Music Long Album Title Display Fix
// @namespace    local.youtube-music-fixes
// @version      1.0.0
// @description  Fixes an issue where long album titles move outside the overflow clipping area and become invisible.
// @match        https://music.youtube.com/*
// @match        file:///*
// @grant        GM_addStyle
// @run-at       document-start
// ==/UserScript==

(() => {
  'use strict';

  const css = `
    /*
     * Do not apply line clamping to the title-group itself.
     * Instead, apply the two-line clamp to the actual title element.
     */
    ytmusic-two-row-item-renderer
      .title-group.ytmusic-two-row-item-renderer {
      display: block !important;
      -webkit-line-clamp: unset !important;
      -webkit-box-orient: initial !important;
      overflow: visible !important;
      max-height: none !important;
    }

    /*
     * Clamp the title element itself to two lines.
     */
    ytmusic-two-row-item-renderer
      .title.ytmusic-two-row-item-renderer {
      display: -webkit-box !important;
      -webkit-box-orient: vertical !important;
      -webkit-line-clamp: 2 !important;

      overflow: hidden !important;
      white-space: normal !important;
      text-overflow: ellipsis !important;

      max-height: calc(
        var(--ytmusic-responsive-font-size, 14px)
        * 2
        * var(--ytmusic-title-line-height, 1.3)
      ) !important;
    }

    /*
     * Prevent the inline-block link from becoming a single multi-line box
     * that gets pushed outside the parent's clipping region.
     */
    ytmusic-two-row-item-renderer
      .title.ytmusic-two-row-item-renderer
      > a.yt-simple-endpoint.yt-formatted-string {
      display: inline !important;
    }
  `;

  function installStyle() {
    if (typeof GM_addStyle === 'function') {
      GM_addStyle(css);
      return;
    }

    const style = document.createElement('style');
    style.id = 'ytmusic-long-album-title-fix';
    style.textContent = css;

    const target = document.head || document.documentElement;
    target.appendChild(style);
  }

  installStyle();
})();
```
