---
pubDatetime: 2026-04-27T17:52:34+09:00
title: "YouTube Musicの非表示ボタンを復元するユーザースクリプト"
description: "YouTube Musicの非表示ボタンを復元するユーザースクリプト 概要 このスクリプトは、YouTube Music上で一部の操作ボタン（巻き戻し・早送り）が非表示になってしまう問題を補正するためのシンプルなユーザースクリプトです。定期的にページ内の要素をチェックし、該当するボタンが非表示になっ…"
---

# YouTube Musicの非表示ボタンを復元するユーザースクリプト

## 概要

このスクリプトは、YouTube Music上で一部の操作ボタン（巻き戻し・早送り）が非表示になってしまう問題を補正するためのシンプルなユーザースクリプトです。定期的にページ内の要素をチェックし、該当するボタンが非表示になっている場合に再び表示されるようにします。

## 目的

YouTube Musicでは、UIの仕様や動作によって以下のような問題が発生することがあります：

* 巻き戻し（10秒戻る）ボタンが表示されない
* 早送り（30秒進む）ボタンが隠れてしまう

このスクリプトは、これらのボタンを自動的に再表示することで、操作性を改善することを目的としています。

## 使い方

### 1. ユーザースクリプト環境を準備

このスクリプトを使用するには、ブラウザにユーザースクリプト管理ツールをインストールする必要があります。代表的なものとして以下があります：

* Tampermonkey（Chrome / Edge / Firefox など）

### 2. スクリプトを登録

1. Tampermonkeyのダッシュボードを開く
2. 新規スクリプトを作成
3. 提供されたコードを貼り付けて保存

### 3. YouTube Musicにアクセス

スクリプトは以下のURLにアクセスした際に自動で動作します：

* [https://music.youtube.com/](https://music.youtube.com/)

特別な操作は不要で、ページを開くだけで機能します。

## 動作の仕組み（簡潔）

* ページ内の「巻き戻し」および「早送り」ボタンを識別
* `hidden` 属性が付いている場合、それを削除
* この処理を5秒ごとに繰り返し実行

これにより、ボタンが再び非表示になっても自動的に復元されます。

## 注意点

* UI構造が変更された場合、スクリプトが正常に動作しなくなる可能性があります
* 定期的にDOMを監視するため、ごくわずかなパフォーマンスへの影響があります

## まとめ

このユーザースクリプトは、YouTube Musicの操作ボタンが消えてしまう問題に対して、簡単かつ自動的に対処できる実用的なツールです。特に頻繁に巻き戻し・早送り操作を行うユーザーにとって、快適な再生環境を維持するのに役立ちます。

```javascript
// ==UserScript==
// @name         YouTube Music Hidden Buttons Fix
// @namespace    http://tampermonkey.net/
// @version      1.2
// @description  Unhide rewind/forward buttons when the left control area changes
// @match        https://music.youtube.com/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const MIN_INTERVAL_MS = 1000;

    let lastRunAt = 0;
    let scheduledTimer = null;

    function unhideButtons() {
        // yt-icon-button elements that contain a replay_10 icon
        document.querySelectorAll(
            'yt-icon-button:has(yt-icon[icon="yt-sys-icons\\:replay_10"])'
        ).forEach(el => {
            el.removeAttribute("hidden");
        });

        // yt-icon-button elements that contain a skip_forward_30 icon
        document.querySelectorAll(
            'yt-icon-button:has(yt-icon[icon="yt-sys-icons\\:skip_forward_30"])'
        ).forEach(el => {
            el.removeAttribute("hidden");
        });
    }

    function runThrottled() {
        const now = Date.now();
        const elapsed = now - lastRunAt;

        // Execute immediately if at least 1000 ms have passed since the previous run
        if (elapsed >= MIN_INTERVAL_MS) {
            if (scheduledTimer !== null) {
                clearTimeout(scheduledTimer);
                scheduledTimer = null;
            }

            lastRunAt = now;
            unhideButtons();
            return;
        }

        // Do not schedule another execution if one is already pending
        if (scheduledTimer !== null) {
            return;
        }

        // Execute after the remaining time needed to satisfy the 1000 ms interval
        const delay = MIN_INTERVAL_MS - elapsed;

        scheduledTimer = setTimeout(() => {
            scheduledTimer = null;
            lastRunAt = Date.now();
            unhideButtons();
        }, delay);
    }

    function observeLeftControls() {
        const target = document.getElementById("left-controls");

        if (!target) {
            // Retry because YouTube Music may not have finished rendering yet
            setTimeout(observeLeftControls, 500);
            return;
        }

        const observer = new MutationObserver(() => {
            runThrottled();
        });

        observer.observe(target, {
            childList: true,
            subtree: true,
            attributes: true,
            attributeFilter: ["hidden", "style", "class"]
        });

        // Run once after setting up the observer
        runThrottled();
    }

    // Initial execution
    runThrottled();

    // Observe left-controls instead of polling every 5 seconds
    observeLeftControls();
})();
```
