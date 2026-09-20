---
pubDatetime: 2026-04-20T17:05:05+09:00
title: "[利用不可] ChatGPTでモデル設定画面を開きやすくするTampermonkeyスクリプト"
description: "このスクリプトは、ChatGPTの画面上に「Config」というボタンを追加し、モデル設定画面を開きやすくするためのTampermonkey用ユーザースクリプトです。毎回メニューをたどる手間を減らし、設定画面にすばやくアクセスしたいときに役立ちます。 目的 ChatGPTでは、モデルに関する設定を開…"
---

このスクリプトは、ChatGPTの画面上に「Config」というボタンを追加し、モデル設定画面を開きやすくするためのTampermonkey用ユーザースクリプトです。毎回メニューをたどる手間を減らし、設定画面にすばやくアクセスしたいときに役立ちます。

## 目的

ChatGPTでは、モデルに関する設定を開くまでにいくつかの操作が必要になることがあります。このスクリプトの目的は、その操作を簡単にすることです。

ページ上のヘッダー部分に独自の「Config」ボタンを追加し、そのボタンを押すだけでモデル設定画面を呼び出せるようにします。日常的に設定を調整したい人にとって、操作を短くできるのが大きな利点です。

## どのように動くか

このスクリプトは、ChatGPTのページが開かれたあとにヘッダー部分の表示を待ちます。対象の場所が見つかると、そこに「Config」ボタンを追加します。

その後、ボタンが押されると、もともとページ内にあるモデル切り替え用のボタンを利用して設定画面を開こうとします。つまり、ChatGPTの既存UIを使いながら、入口だけをわかりやすく増やす仕組みです。

## 使い方

### 1. Tampermonkeyにスクリプトを登録する

まず、ブラウザにTampermonkeyを入れた状態で、新しいユーザースクリプトを作成します。そこに提示されたコードを貼り付けて保存します。

### 2. ChatGPTを開く

対象ページは `https://chatgpt.com/*` です。ChatGPTを開くと、スクリプトが自動で動作します。

### 3. 「Config」ボタンを押す

ページ上部のヘッダー付近に「Config」ボタンが追加されます。このボタンを押すと、モデル設定画面を開く動作が実行されます。

## 便利な点

このスクリプトのよいところは、操作を単純化できることです。特に、モデル設定を頻繁に開く場合には、毎回同じ場所を探す必要がなくなります。

また、ページの読み込み直後に要素がまだ表示されていない場合でも、しばらく待ってからボタンを追加するようになっているため、比較的安定して使えるよう工夫されています。

## 注意点

このスクリプトは、ChatGPTのページ構造やボタンの属性に依存しています。そのため、ChatGPT側の画面構成が変わると、ボタンが表示されなくなったり、クリックしても設定画面が開かなくなったりする可能性があります。

また、これは公式機能の追加ではなく、ブラウザ上で見た目と操作を補助するためのものです。利用する環境によっては期待どおりに動かないこともあります。

## まとめ

このTampermonkeyスクリプトは、ChatGPTに「Config」ボタンを追加し、モデル設定画面を開きやすくするための簡単な補助ツールです。複雑な仕組みを意識せず、設定画面へのアクセスを手早くしたい人に向いています。

ChatGPTをより使いやすくしたいときの、小さく実用的なカスタマイズの一例といえます。

```javascript
// ==UserScript==
// @name         Add Config Button With page-header Reattach Detection
// @namespace    http://tampermonkey.net/
// @version      1.2
// @description  Wait for #page-header, add Config button, and re-add it when #page-header is recreated
// @match        https://chatgpt.com/*
// @grant        none
// ==/UserScript==

(function () {
  'use strict';

  const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

  const BUTTON_ID = 'tm-config-button';
  let headerRemovalObserver = null;
  let isRunning = false;

  async function waitForPageHeader(timeoutMs = 30000, intervalMs = 100) {
    const start = Date.now();

    while (Date.now() - start < timeoutMs) {
      const pageHeader = document.querySelector('#page-header');
      if (pageHeader) {
        return pageHeader;
      }
      await sleep(intervalMs);
    }

    throw new Error('Timeout: #page-header was not found.');
  }

  function findButtonContainer(pageHeader) {
    return pageHeader.childNodes[1] || null;
  }

  async function onConfigClick(e) {
    console.log('Config button clicked', e);

    const switcherButton = document.querySelector('button[data-testid="model-switcher-dropdown-button"]');
    if (!switcherButton) {
      console.warn('model-switcher-dropdown-button not found');
      return;
    }

    switcherButton.click();

    await sleep(0);

    const configureModal = document.querySelector('div[data-testid="model-configure-modal"]');
    if (!configureModal) {
      console.warn('model-configure-modal not found');
      return;
    }

    configureModal.click();
  }

  function createConfigButton() {
    const button = document.createElement('button');
    button.id = BUTTON_ID;
    button.innerText = 'Config';
    button.className =
      'group flex cursor-pointer justify-center items-center gap-1 rounded-lg min-h-9 touch:min-h-10 px-2.5 text-lg hover:bg-token-surface-hover focus-visible:bg-token-surface-hover font-normal whitespace-nowrap focus-visible:outline-none';
    button.addEventListener('click', onConfigClick);
    return button;
  }

  function addConfigButton(pageHeader) {
    const target = findButtonContainer(pageHeader);
    if (!target) {
      console.warn('#page-header.childNodes[1] not found');
      return false;
    }

    if (target.querySelector(`#${BUTTON_ID}`)) {
      return true;
    }

    const configButton = createConfigButton();
    target.appendChild(configButton);
    console.log('Config button added');

    return true;
  }

  function observePageHeaderRemoval(pageHeader) {
    if (headerRemovalObserver) {
      headerRemovalObserver.disconnect();
      headerRemovalObserver = null;
    }

    headerRemovalObserver = new MutationObserver(() => {
      if (!document.contains(pageHeader)) {
        console.log('#page-header was removed. Restarting...');
        headerRemovalObserver.disconnect();
        headerRemovalObserver = null;
        run();
      }
    });

    const root = document.body || document.documentElement;
    headerRemovalObserver.observe(root, {
      childList: true,
      subtree: true,
    });
  }

  async function run() {
    if (isRunning) return;
    isRunning = true;

    try {
      // (0) Wait for #page-header
      const pageHeader = await waitForPageHeader();

      // (1) Add config button
      const added = addConfigButton(pageHeader);
      if (!added) {
        isRunning = false;
        await sleep(300);
        run();
        return;
      }

      // (2) Observe removal of #page-header
      // (3) If removed, restart from (1)
      observePageHeaderRemoval(pageHeader);
    } catch (err) {
      console.error(err);
    } finally {
      isRunning = false;
    }
  }

  run();
})(); 
```
