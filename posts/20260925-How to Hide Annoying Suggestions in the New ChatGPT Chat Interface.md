---
pubDatetime: 2026-09-25T14:45:00+09:00
title: "ChatGPTの新しいチャット画面に表示される邪魔な提案を非表示にする方法"
description: "Tampermonkeyを使って、ChatGPTの新しいチャット画面下部に表示される不要な提案を非表示にする方法を紹介します。CSSを適用するだけのシンプルな仕組みなので、常時監視する処理が不要で、軽量に動作します。"
---

ChatGPTの新しいチャット画面では、画面下部に提案やおすすめの項目が表示されることがあります。

便利な場合もありますが、普段使わない場合は表示領域を圧迫したり、チャット画面を見づらく感じることがあります。

そこで、Tampermonkeyを使って、この提案部分を自動的に非表示にします。

## 使用するスクリプト

```javascript
// ==UserScript==
// @name         Hide Thread Bottom UL
// @namespace    tampermonkey
// @version      1.1
// @description  対象の ul を CSS で非表示にする
// @match        *://*/*
// @run-at       document-start
// @grant        none
// ==/UserScript==

(function () {
  'use strict';

  const style = document.createElement('style');
  style.textContent = `
    #thread-bottom div.contents section ul {
      display: none !important;
    }
  `;

  document.documentElement.appendChild(style);
})();
```

## 何をするスクリプトなのか

このスクリプトは、ChatGPTの画面下部にある提案一覧を非表示にします。

対象となっているのは、次の部分です。

```css
#thread-bottom div.contents section ul
```

ここに対して、

```css
display: none !important;
```

を適用することで、画面上から見えなくしています。

要素自体を削除するわけではなく、表示だけを消す仕組みです。

## Tampermonkeyに登録する

まず、ブラウザにTampermonkeyをインストールします。

Tampermonkeyの管理画面を開き、新しいユーザースクリプトを作成します。

最初から入力されている内容を削除し、上記のスクリプトを貼り付けて保存します。

その後ChatGPTを開き直すと、対象となる提案が自動的に非表示になります。

## ChatGPTだけで動かすようにする

上記のスクリプトでは、

```javascript
// @match        *://*/*
```

となっているため、すべてのWebサイトでスクリプトが実行されます。

ChatGPTでしか使わない場合は、次のように変更しておく方が扱いやすくなります。

```javascript
// @match        https://chatgpt.com/*
```

完成形は次のようになります。

```javascript
// ==UserScript==
// @name         Hide ChatGPT Thread Bottom Suggestions
// @namespace    tampermonkey
// @version      1.1
// @description  ChatGPTの画面下部に表示される提案を非表示にする
// @match        https://chatgpt.com/*
// @run-at       document-start
// @grant        none
// ==/UserScript==

(function () {
  'use strict';

  const style = document.createElement('style');

  style.textContent = `
    #thread-bottom div.contents section ul {
      display: none !important;
    }
  `;

  document.documentElement.appendChild(style);
})();
```

## MutationObserverを使わない理由

ChatGPTのようなWebアプリでは、画面の内容があとから動的に追加されることがあります。

そのため、最初はMutationObserverで画面の変化を監視する方法も考えられます。

しかし今回の目的は、特定の要素を常に非表示にするだけです。

CSSを最初に追加しておけば、あとから対象の要素が表示された場合にも自動的に同じCSSが適用されます。

そのため、画面の変化を常時監視する必要はありません。

処理も単純で、余計な監視処理を動かさずに済みます。

## 元に戻す方法

元の表示に戻したい場合は、Tampermonkeyでこのユーザースクリプトを無効にするか削除します。

ページを再読み込みすれば、ChatGPT本来の表示に戻ります。

## 注意点

この方法はChatGPTの画面構造を利用しています。

ChatGPT側のアップデートによってHTML構造や要素名が変更された場合、

```css
#thread-bottom div.contents section ul
```

が一致しなくなり、非表示にならなくなる可能性があります。

その場合は、対象となるCSSセレクタを現在の画面構造に合わせて修正する必要があります。
