---
pubDatetime: 2026-01-19T20:01:32+09:00
title: "ChatGPTの「新しいチャット」を新規タブで開けるTampermonkeyスクリプト"
description: "現在の会話を残したまま、別の会話をワンクリックで始められるようにします。複数のテーマを並行して扱うことが多い人に便利な小さなカスタマイズです。"
---

ChatGPTを使っていて、地味に欲しかったのが **「新しいチャットを新規タブで開く」ボタン** です。

通常の「新しいチャット」を押すと、現在のタブがそのまま新しいチャットに切り替わります。

しかし、

- 今の会話を残したまま別の質問をしたい
- 調査中に複数のチャットを並行して開きたい
- Ctrl / Cmd + クリックを意識せず、ワンクリックで新しいタブを作りたい

という使い方をしていると、専用ボタンが欲しくなります。

そこでTampermonkeyで、ChatGPTの「新しいチャット」ボタンの横に **「新規タブ」ボタンを自動追加** するUserScriptを作りました。

## 完成イメージ

ChatGPTの画面にある

**「新しいチャット」**

の近くに、

**「新規タブ」**

というボタンを追加します。

「新規タブ」をクリックすると、現在開いているチャットはそのまま残して、ChatGPTの新しいチャット画面を別タブで開けます。

複数のテーマについてChatGPTと並行して会話するときにかなり便利です。

## Tampermonkeyスクリプト

以下をTampermonkeyに登録します。

```javascript
// ==UserScript==
// @name         Add New Chat Link
// @namespace    tampermonkey
// @version      1.0
// @description  「新しいチャット」ボタンを複製してリンクを追加
// @match        https://chatgpt.com/*
// @grant        none
// ==/UserScript==

(function () {
    'use strict';

    const TARGET_TEXT = '新しいチャット';
    const CLONED_TEXT = '新規タブ';
    const ADDED_LINK_SELECTOR = 'a[data-tm-new-chat-link]';

    /**
     * root 自身または子孫から
     * 「新しいチャット」の span を探す
     */
    function findTargetSpan(targetRoot) {
        // root 自身を確認
        if (
            targetRoot instanceof HTMLSpanElement &&
            targetRoot.innerText.trim() === TARGET_TEXT
        ) {
            return targetRoot;
        }

        // 子孫を確認
        if (targetRoot.querySelectorAll) {
            for (const span of targetRoot.querySelectorAll('span')) {
                if (span.innerText.trim() === TARGET_TEXT) {
                    return span;
                }
            }
        }

        return null;
    }

    function initialize() {
        const root = document.querySelector('#root');

        if (!root) {
            console.warn('[Tampermonkey] #root not found');
            return;
        }

        const observerOptions = {
            childList: true,
            subtree: true
        };

        let observer;

        function process(targetRoot) {
            // process 中は observer を停止
            observer?.disconnect();

            try {
                const span = findTargetSpan(targetRoot);

                if (!span) {
                    return;
                }

                // 祖先の button を取得
                const button = span.closest('button');

                if (!button) {
                    return;
                }

                // button の parent を取得
                const parent = button.parentElement;

                if (!parent) {
                    return;
                }

                // すでに追加済みなら何もしない
                if (parent.querySelector(ADDED_LINK_SELECTOR)) {
                    return;
                }

                // button を複製
                const buttonCloned = button.cloneNode(true);

                // 複製した button 内の
                // 「新しいチャット」を「新規タブ」に変更
                const clonedSpan = findTargetSpan(buttonCloned);

                if (clonedSpan) {
                    clonedSpan.innerText = CLONED_TEXT;
                }

                buttonCloned.firstChild.classList.replace(
                    "text-emphasis",
                    "text-default"
                );

                // <a href="/"> を作成
                const anchor = document.createElement('a');

                anchor.href = '/';
                anchor.target = '_blank';
                anchor.dataset.tmNewChatLink = 'true';

                anchor.appendChild(buttonCloned);

                // 元 button の parent に追加
                parent.prepend(anchor);

                console.log('[Tampermonkey] link added:', anchor);
            } finally {
                // process 完了後に監視を再開
                observer?.observe(root, observerOptions);
            }
        }

        observer = new MutationObserver((mutations) => {
            // callback 開始時点で止める
            observer.disconnect();

            try {
                for (const mutation of mutations) {
                    for (const node of mutation.addedNodes) {
                        if (!(node instanceof Element)) {
                            continue;
                        }

                        process(node);
                    }
                }
            } finally {
                observer.observe(root, observerOptions);
            }
        });

        // 初回実行
        process(root);

        // #root 以下への子孫要素追加を監視
        observer.observe(root, observerOptions);
    }

    if (document.readyState === 'complete') {
        initialize();
    } else {
        window.addEventListener('load', initialize, { once: true });
    }
})();
```

## 何をしているのか

仕組み自体はシンプルです。

ChatGPTの画面から「新しいチャット」という文字列を持つ`span`を探し、その親にある`button`を取得します。

```javascript
const span = findTargetSpan(targetRoot);
const button = span.closest('button');
```

そして、そのボタンを丸ごと複製します。

```javascript
const buttonCloned = button.cloneNode(true);
```

ゼロからCSSを書くのではなく、 **ChatGPT自身のボタンを複製している** のがポイントです。

これによって、ChatGPT側のUIと比較的馴染みやすい見た目になります。

複製したボタンについては、表示を

```text
新しいチャット
```

から

```text
新規タブ
```

へ変更します。

```javascript
const clonedSpan = findTargetSpan(buttonCloned);

if (clonedSpan) {
    clonedSpan.innerText = CLONED_TEXT;
}
```

## 新しいタブでChatGPTを開く

複製したボタンは`a`要素の中に入れています。

```javascript
const anchor = document.createElement('a');

anchor.href = '/';
anchor.target = '_blank';
```

ポイントは、

```javascript
anchor.target = '_blank';
```

です。

これによってボタンを押したとき、現在のChatGPT画面を置き換えるのではなく、 **新しいタブでChatGPTを開く** ようになります。

現在のチャットを残したまま別の会話を開始できます。

## MutationObserverを使っている理由

ChatGPTは画面遷移のたびにページ全体を読み込み直す一般的なWebサイトとは少し違い、JavaScriptによって画面の内容が動的に変更されます。

そのため、

```javascript
window.addEventListener('load', ...)
```

だけを利用してボタンを追加すると、ChatGPT側でUIが再描画されたときに追加したボタンが消える可能性があります。

そこで`MutationObserver`を利用しています。

```javascript
observer = new MutationObserver((mutations) => {
    // ...
});
```

そして、

```javascript
observer.observe(root, {
    childList: true,
    subtree: true
});
```

として、`#root`以下に追加されるDOM要素を監視します。

ChatGPT側でUIが更新された場合にも「新しいチャット」を探して、必要なら「新規タブ」を追加し直す仕組みです。

## ボタンが大量に増えないようにする

MutationObserverを使うと注意したいのが、自分自身がDOMを書き換えた結果まで監視対象になることです。

そのため、このスクリプトでは追加するリンクに

```javascript
anchor.dataset.tmNewChatLink = 'true';
```

という目印を付けています。

HTML上では、

```html
<a data-tm-new-chat-link="true">
```

のようになります。

そして、

```javascript
if (parent.querySelector(ADDED_LINK_SELECTOR)) {
    return;
}
```

として、すでにボタンが存在している場合には追加しません。

さらに`process()`中は一時的にMutationObserverを停止し、処理が終わったら監視を再開しています。

```javascript
observer?.disconnect();

try {
    // DOM操作
} finally {
    observer?.observe(root, observerOptions);
}
```

これによって、自分自身のDOM変更を拾って処理が繰り返されることを避けています。

## インストール方法

Tampermonkeyを導入済みなら設定は簡単です。

1. Tampermonkeyのダッシュボードを開く
2. 「新規スクリプトを追加」を選択
3. 最初から入っているコードを削除
4. 上記のUserScriptを貼り付ける
5. 保存する
6. ChatGPTを再読み込みする

これでChatGPTの画面に「新規タブ」が追加されます。

## こういう人には特におすすめ

ChatGPTを「1つのチャットですべて質問する」のではなく、 **テーマごとにチャットを分けて使っている人** には特に便利です。

例えば、

```text
タブ1：プログラムについて質問
タブ2：メール文章を作成
タブ3：技術情報を調査
タブ4：アイデア出し
```

といった使い方がしやすくなります。

元のチャットから離れずに次のチャットを作れるので、ChatGPTを大量に使うほど恩恵があります。

## 注意点

このUserScriptはChatGPTのDOM構造を利用しています。

特に、

```javascript
#root
```

や、

```javascript
span.closest('button')
```

といったChatGPT側のHTML構造を前提にしています。

そのため、ChatGPTのUIがアップデートされると動作しなくなる可能性があります。

また、「新しいチャット」という日本語表示を探しているため、ChatGPTを英語など別の言語設定で使用している場合は、そのままでは動作しません。

例えば英語UIに対応させる場合は、検索対象を`New chat`にも対応させるなどの変更が必要です。

TampermonkeyのUserScriptはWebページ上でJavaScriptを実行する仕組みなので、内容を理解できるスクリプトだけを導入することも重要です。

## まとめ

今回のUserScriptを入れると、ChatGPTの「新しいチャット」の近くに **「新規タブ」ボタン** を追加できます。

やっていることは小さなUI変更ですが、

**「今のチャットを残しておきたい → 新しいタブを開く → ChatGPTを開く」**

という操作を、

**「新規タブ」を1回クリック**

に短縮できます。

複数のチャットを並行して使うことが多い人ほど便利になるTampermonkeyスクリプトです。

ChatGPTをブラウザで頻繁に使っているなら、試してみる価値があります。
