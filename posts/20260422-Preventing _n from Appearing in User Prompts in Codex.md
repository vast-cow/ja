---
pubDatetime: 2026-04-22T21:28:00+09:00
title: "Codexでユーザー入力のプロンプトに\\nが入るのを防ぐ方法"
description: "これは何をするものか、何のために使うのか、使い方を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Codexを使っていると、プロンプト入力欄に意図しない `\n` が入ってしまい、文章が崩れたり、入力しづらく感じたりすることがあります。ここでは、その問題を防ぐためのUserScriptの目的と使い方を、簡単にまとめます。

## これは何をするものか

このUserScriptは、ChatGPT Codex Cloudの `#prompt-textarea` で貼り付けが発生したときだけ動作します。

貼り付け後、入力欄の直下にある `<p>` 要素のテキストノードを確認し、改行コードを次のように正規化します。

* `CRLF`（`\r\n`）→ `LF`（`\n`）
* `CR`（`\r`）→ `LF`（`\n`）
* `LF`（`\n`）→ そのまま

つまり、文章中の改行そのものは維持し、改行コードの種類だけをLFに揃えます。

## 何のために使うのか

コピー元のOSやアプリケーションによって、テキストに含まれる改行コードが異なる場合があります。

このスクリプトは、それらを貼り付け後にLFへ統一し、Codexへ入力するテキストの改行形式を一定にするためのものです。

たとえば、内部的に次のような文字列があった場合、

```text
AAA\r\nBBB\rCCC\nDDD
```

正規化後は次の状態になります。

```text
AAA\nBBB\nCCC\nDDD
```

表示上の複数行構造は維持されます。

## 軽量化した仕組み

以前の実装では、`#prompt-textarea` の出現や差し替えを検出するために `MutationObserver` でページ全体のDOM変更を監視していました。

今回の実装では、この常時監視を行いません。

代わりに `document` に `paste` イベントリスナーを1つだけ登録し、イベントが `#prompt-textarea` から発生した場合だけ処理します。

この方式には次の特徴があります。

* ページ全体を `MutationObserver` で常時監視しない
* エディタを繰り返し `querySelector` で探索しない
* `#prompt-textarea` が差し替えられてもリスナーを付け直す必要がない
* 貼り付けが発生したときだけ正規化処理を行う
* `innerHTML` 全体を書き換えず、必要なテキストノードだけ変更する
* デバッグ用の大量のconsoleログを出力しない

## 使い方

### UserScriptとして登録する

ブラウザにUserScriptを実行できる拡張機能を導入し、後述のコードを新しいUserScriptとして登録します。

登録後、対象のCodex Cloudページを開くと自動的に有効になります。

### 対象ページ

このスクリプトは次のURLを対象にしています。

* `https://chatgpt.com/codex/cloud`
* `https://chatgpt.com/codex/cloud/*`
* `https://chatgpt.com/codex/cloud?*`

## 動作の流れ

処理の流れは次のとおりです。

1. ページ読み込み時に `document` へ `paste` リスナーを1つ登録する
2. 貼り付けイベントが発生する
3. `event.composedPath()` から `#prompt-textarea` 内での貼り付けか確認する
4. 対象外なら何もしない
5. 対象なら、通常の貼り付け処理がDOMへ反映されるのを待つ
6. `#prompt-textarea` 直下の `<p>` を調べる
7. 各 `<p>` 内のテキストノードについて、`CRLF` と `CR` を `LF` に変換する

`LF` はそのまま残すため、改行を削除する処理ではありません。

## UserScript

```javascript
// ==UserScript==
// @name         ChatGPT Codex Cloud - Normalize Newlines to LF
// @namespace    https://chatgpt.com/
// @version      1.2.0
// @description  Normalize CRLF/CR to LF in direct <p> children after paste.
// @match        https://chatgpt.com/codex/cloud
// @match        https://chatgpt.com/codex/cloud/*
// @match        https://chatgpt.com/codex/cloud?*
// @grant        none
// ==/UserScript==

(() => {
  "use strict";

  /**
   * CRLF / CR を LF に統一する。
   *
   * \r\n -> \n
   * \r   -> \n
   * \n   -> \n
   */
  function normalizeNewlines(text) {
    return text.replace(/\r\n?/g, "\n");
  }

  /**
   * #prompt-textarea 直下の <p> に含まれる
   * テキストノードだけを処理する。
   */
  function normalizeParagraphs(editor) {
    if (!editor?.isConnected) return;

    for (const p of editor.children) {
      if (p.tagName !== "P") continue;

      const walker = document.createTreeWalker(
        p,
        NodeFilter.SHOW_TEXT
      );

      let node;

      while ((node = walker.nextNode())) {
        const before = node.data;
        const after = normalizeNewlines(before);

        if (before !== after) {
          node.data = after;
        }
      }
    }
  }

  /**
   * paste の発生元が #prompt-textarea 内か確認する。
   *
   * イベント委譲を使うため、エディタ自体が
   * 差し替えられてもリスナーの再登録は不要。
   */
  function getEditorFromPasteEvent(event) {
    const path = event.composedPath();

    for (const node of path) {
      if (
        node instanceof Element &&
        node.id === "prompt-textarea"
      ) {
        return node;
      }
    }

    return null;
  }

  /**
   * document に paste listener を1つだけ登録する。
   * MutationObserver は使用しない。
   */
  document.addEventListener(
    "paste",
    event => {
      const editor = getEditorFromPasteEvent(event);

      if (!editor) return;

      // 通常のpaste処理がDOMへ反映された後に実行する。
      setTimeout(() => {
        normalizeParagraphs(editor);
      }, 0);
    },
    true
  );
})();
```

## 改行正規化のポイント

正規化には次の処理を使っています。

```javascript
text.replace(/\r\n?/g, "\n");
```

ここでは `\r\n?` によって、まず `CRLF`（`\r\n`）を1つの改行として扱い、単独の `CR`（`\r`）にも対応します。

置換先はいずれも `\n` です。

そのため、`CRLF` を誤って2つのLFへ変換することなく、すべてLFへ統一できます。

## DOM全体を書き換えない理由

このスクリプトでは、段落の `innerHTML` を取得して再代入する方法は使いません。

代わりに `TreeWalker` でテキストノードだけを取得し、実際に改行コードの変換が必要なノードだけ `node.data` を変更します。

これにより、段落内部のDOM構造を必要以上に再構築せずに済みます。

## MutationObserverを使わない理由

この処理が必要になるのは貼り付け時だけです。

そのため、入力欄の出現・削除・差し替えを検出する目的でページ全体を常時監視する必要はありません。

`paste` イベントを `document` で受けるイベント委譲方式にすると、`#prompt-textarea` が後から作成された場合や別のDOMノードへ差し替えられた場合でも、そのまま貼り付けイベントを処理できます。

## 動作確認

確認する場合は、改行コードの異なるテキストをCodex Cloudの入力欄へ貼り付けます。

想定する変換は次のとおりです。

| 貼り付け前 | 貼り付け後 |
| --- | --- |
| `A\r\nB` | `A\nB` |
| `A\rB` | `A\nB` |
| `A\nB` | `A\nB` |
| `A\r\nB\rC\nD` | `A\nB\nC\nD` |

重要なのは、**改行の数や文章の複数行構造を消すのではなく、改行コードだけをLFへ統一する**ことです。

## 注意点

### 貼り付け時だけ動作する

このUserScriptは `paste` イベントを契機にしています。

キーボード入力など、貼り付け以外の方法で入力された内容を常時監視して正規化するものではありません。

### 対象は直下の `<p>` 要素

処理対象は `#prompt-textarea` の直下にある `<p>` 要素です。

Codex Cloud側のDOM構造が将来変更された場合は、対象要素の条件を調整する必要が出る可能性があります。

### LFは削除しない

このスクリプトの目的は1行化ではありません。

複数行のプロンプトは複数行のまま維持され、`LF` も残ります。

## まとめ

このUserScriptは、ChatGPT Codex Cloudへテキストを貼り付けたときに、`CRLF` と `CR` を `LF` へ統一します。

改行そのものは維持するため、複数行のプロンプトをそのまま利用できます。

また、ページ全体を `MutationObserver` で監視せず、`document` の `paste` イベントを利用する構成にすることで、常時監視を避けています。

処理対象についても `innerHTML` 全体ではなくテキストノードだけを変更するため、必要なタイミングに必要な範囲だけ処理する構成になっています。
