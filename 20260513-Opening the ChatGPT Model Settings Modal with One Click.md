---
pubDatetime: 2026-05-13T13:03:10+09:00
title: "ChatGPTのモデル設定モーダルを1クリックで開く"
description: "目的、主な動作、使い方を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

このスクリプトは、ChatGPTの画面上に表示される「モデル設定モーダル」に関連する要素を自動でクリックするためのものです。Tampermonkeyなどのユーザースクリプト管理ツールで利用することを想定しています。

## 目的

このスクリプトの目的は、ChatGPTの利用中に特定のモーダル要素が表示されたとき、それを手動でクリックする手間を減らすことです。

対象となるのは、次の属性を持つ要素です。

```javascript
data-testid="model-configure-modal"
```

ページ上にこの要素が見つかると、スクリプトが自動的にクリックを実行します。

## 主な動作

このスクリプトは、ChatGPTのページでのみ動作します。

```javascript
https://chatgpt.com/*
```

ページを開いた直後に対象要素を探し、見つかった場合は自動でクリックします。さらに、ページの内容が後から変化した場合にも再度チェックを行います。

そのため、最初は表示されていなかったモーダルが後から出てきた場合でも、自動クリックの対象になります。

## 使い方

## 1. Tampermonkeyを用意する

まず、ブラウザにTampermonkeyなどのユーザースクリプト管理拡張機能をインストールします。

## 2. 新しいスクリプトを作成する

Tampermonkeyの管理画面を開き、新しいユーザースクリプトを作成します。

## 3. スクリプトを貼り付ける

作成画面に、このスクリプト全体を貼り付けます。

スクリプトの冒頭には、名前、バージョン、説明、動作対象URLなどが書かれています。これにより、ChatGPTのページでのみ実行されるようになっています。

## 4. 保存してChatGPTを開く

スクリプトを保存したあと、`https://chatgpt.com/` を開きます。

対象のモーダル要素が表示されると、スクリプトが自動的にクリックします。

## ログの確認

このスクリプトは、ブラウザの開発者ツールにログを出力します。

たとえば、次のような情報を確認できます。

```text
Started.
Found 1 target(s).
Clicking target #1.
MutationObserver attached to document.body.
```

これにより、スクリプトが起動しているか、対象要素を見つけたか、クリックが実行されたかを確認できます。

## ページ変化への対応

ChatGPTの画面では、ページ全体を再読み込みしなくても、新しい要素が後から追加されることがあります。

このスクリプトでは `MutationObserver` を使って、ページ内の変化を監視しています。対象の要素が後から追加された場合でも、それを検出して自動クリックを試みます。

## 注意点

このスクリプトは、ChatGPTの画面構造に依存しています。対象要素の属性や構造が変更された場合、自動クリックが動作しなくなる可能性があります。

また、自動クリックによって意図しない操作が行われる可能性もあります。利用する際は、どの要素がクリックされるのかを理解したうえで使うことが重要です。

## まとめ

このユーザースクリプトは、ChatGPT上で特定のモデル設定モーダル要素を自動クリックするための簡単な補助ツールです。

ページ表示直後だけでなく、後から追加された要素にも対応できるため、繰り返し表示されるモーダルに対する操作を効率化できます。

```javascript
// ==UserScript==
// @name         ChatGPT Model Configure Modal Auto Clicker
// @namespace    http://tampermonkey.net/
// @version      1.3
// @description  Automatically clicks ChatGPT model configure modal elements when they appear.
// @match        https://chatgpt.com/*
// @grant        none
// ==/UserScript==

(function () {
  const selector = '*[data-testid="model-configure-modal"]';
  const intervalMs = 100;

  let lastExecutedAt = 0;
  let reservedTimer = null;

  const log = (...args) => {
    console.log('[bookmarklet:model-configure-modal]', ...args);
  };

  const clickIfFound = () => {
    lastExecutedAt = Date.now();
    reservedTimer = null;

    const targets = document.querySelectorAll(selector);

    log(`Found ${targets.length} target(s).`);

    targets.forEach((target, index) => {
      log(`Clicking target #${index + 1}.`, target);
      target.click();
    });
  };

  const requestClickIfFound = () => {
    const now = Date.now();
    const elapsed = now - lastExecutedAt;

    if (elapsed >= intervalMs) {
      clickIfFound();
      return;
    }

    if (reservedTimer !== null) {
      log('Execution already reserved. Skipped.');
      return;
    }

    const delay = intervalMs - elapsed;

    log(`Executed recently. Reserving execution in ${delay}ms.`);

    reservedTimer = setTimeout(() => {
      clickIfFound();
    }, delay);
  };

  log('Started.');

  requestClickIfFound();

  const observer = new MutationObserver((mutations) => {
    log(`Mutation observed. mutation count: ${mutations.length}`);

    requestClickIfFound();
  });

  observer.observe(document.body, {
    childList: true,
    subtree: true
  });

  log('MutationObserver attached to document.body.');
})();
```
