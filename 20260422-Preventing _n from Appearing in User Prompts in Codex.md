---
pubDatetime: 2026-04-22T21:28:00+09:00
title: "Codexでユーザー入力のプロンプトに\\nが入るのを防ぐ方法"
description: "これは何をするものか、何のために使うのか、使い方を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Codexを使っていると、プロンプト入力欄に意図しない `\n` が入ってしまい、文章が崩れたり、入力しづらく感じたりすることがあります。ここでは、その問題を防ぐためのUserScriptの目的と使い方を、簡単にまとめます。

## これは何をするものか

このUserScriptは、ChatGPT Codex Cloudの入力欄を監視し、ユーザーが入力した内容の中に入ってしまった改行コードを取り除くためのものです。

特に、`#prompt-textarea` という入力エリアの中にある段落要素を対象にして、`CR` や `LF` といった改行文字を削除します。これにより、プロンプトが意図せず複数行になってしまうのを防ぎます。

また、改行を削除したあとにカーソル位置が大きくずれないように調整する仕組みも入っています。そのため、入力中の操作感ができるだけ損なわれにくくなっています。

## 何のために使うのか

このスクリプトの主な目的は、プロンプトを1行のまま扱いやすくすることです。

たとえば、次のような場面で役立ちます。

* 入力中に不要な改行が混ざるのを避けたい
* プロンプトを整った形でそのまま送信したい
* 改行によって見た目や編集操作が不安定になるのを減らしたい

細かな内部処理はありますが、利用者としては「入力欄の改行を自動で消してくれる補助ツール」と考えれば十分です。

## 使い方

### UserScriptとして登録する

このコードはUserScriptとして使います。一般的には、ブラウザにUserScriptを実行できる拡張機能を入れたうえで、そこにスクリプトを登録します。

登録すると、対象ページで自動的に動作するようになります。

### 対象ページ

このスクリプトは、次のURLに対応しています。

* `https://chatgpt.com/codex/cloud`
* `https://chatgpt.com/codex/cloud/*`

そのため、Codex Cloudの該当ページを開いたときに動作します。

### 動作の流れ

ページを開くと、スクリプトが入力欄の出現を待ちます。
入力欄が見つかると、入力イベントを監視し、文字が入力されるたびに改行が含まれていないかを確認します。

改行があれば自動で削除し、その後にカーソル位置をできるだけ自然な位置へ戻します。

## 使うときのポイント

### 改行を残したい用途には向かない

このスクリプトは改行を消すことが前提です。複数行のプロンプトをそのまま書きたい場合には不向きです。

### 入力の見た目を整えたい人向け

1行で簡潔にプロンプトを書きたい場合や、意図しない改行が気になる場合には便利です。入力時のストレスを減らしたい人に向いています。

## まとめ

このUserScriptは、Codex Cloudの入力欄で不要な改行が入るのを防ぐためのシンプルな補助ツールです。
入力欄を監視し、改行を自動で削除しながら、カーソル位置もできるだけ保つように動きます。

技術的な仕組みを深く理解しなくても、「Codexのプロンプトを1行で安定して入力しやすくするためのもの」として使えます。

```javascript
// ==UserScript==
// @name         ChatGPT Codex Cloud - Remove Newlines in Prompt
// @namespace    https://chatgpt.com/
// @version      1.1.2-debug
// @description  Remove CR/LF from direct <p> children on paste event with detailed debug logs.
// @match        https://chatgpt.com/codex/cloud
// @match        https://chatgpt.com/codex/cloud/*
// @grant        none
// ==/UserScript==

(() => {
  "use strict";

  const LOG_PREFIX = "[CodexPromptNLDebug]";
  const SCRIPT_VERSION = "1.1.2-debug";

  let observerA = null; // waits for #prompt-textarea to appear
  let observerB = null; // watches #prompt-textarea replacement/removal
  let currentEditor = null;
  let pasteHandler = null;

  let attachCount = 0;
  let pasteCount = 0;
  let sanitizeCount = 0;
  let observerACallbackCount = 0;
  let observerBCallbackCount = 0;

  function log(...args) {
    console.log(LOG_PREFIX, ...args);
  }

  function warn(...args) {
    console.warn(LOG_PREFIX, ...args);
  }

  function error(...args) {
    console.error(LOG_PREFIX, ...args);
  }

  function describeNode(node) {
    if (!node) return null;

    return {
      nodeName: node.nodeName,
      id: node.id || null,
      className: typeof node.className === "string" ? node.className : null,
      isConnected: node.isConnected,
      childElementCount: node.childElementCount,
      textLength: node.textContent?.length ?? null,
      htmlLength: node.innerHTML?.length ?? null,
    };
  }

  function getEditor() {
    return document.querySelector("#prompt-textarea");
  }

  function getParagraphs(editor = getEditor()) {
    if (!editor) return [];
    return editor.querySelectorAll(":scope > p");
  }

  function sanitizeParagraphs(reason = "unknown") {
    sanitizeCount += 1;

    log("sanitizeParagraphs:start", {
      sanitizeCount,
      reason,
      href: location.href,
      activeElement: describeNode(document.activeElement),
      currentEditor: describeNode(currentEditor),
      foundEditor: describeNode(getEditor()),
    });

    try {
      const editor = getEditor();

      if (!editor) {
        warn("sanitizeParagraphs:editor-not-found", {
          sanitizeCount,
          reason,
        });
        return;
      }

      const paragraphs = getParagraphs(editor);

      log("sanitizeParagraphs:editor-found", {
        sanitizeCount,
        reason,
        editor: describeNode(editor),
        paragraphCount: paragraphs.length,
      });

      let changedCount = 0;

      paragraphs.forEach((p, index) => {
        const before = p.innerHTML;
        const after = before.replaceAll("\r", "").replaceAll("\n", "");

        const hasCR = before.includes("\r");
        const hasLF = before.includes("\n");

        log("sanitizeParagraphs:paragraph-check", {
          sanitizeCount,
          index,
          hasCR,
          hasLF,
          beforeHtmlLength: before.length,
          afterHtmlLength: after.length,
          textLength: p.textContent?.length ?? null,
        });

        if (before !== after) {
          changedCount += 1;

          log("sanitizeParagraphs:paragraph-changed", {
            sanitizeCount,
            index,
            before,
            after,
          });

          p.innerHTML = after;
        } else {
          log("sanitizeParagraphs:paragraph-unchanged", {
            sanitizeCount,
            index,
          });
        }
      });

      log("sanitizeParagraphs:done", {
        sanitizeCount,
        reason,
        paragraphCount: paragraphs.length,
        changedCount,
      });
    } catch (err) {
      error("sanitizeParagraphs:error", {
        sanitizeCount,
        reason,
        errorName: err?.name,
        errorMessage: err?.message,
        stack: err?.stack,
      });
    }
  }

  function detachFromCurrentEditor(reason = "unknown") {
    log("detachFromCurrentEditor:start", {
      reason,
      currentEditor: describeNode(currentEditor),
      hasPasteHandler: Boolean(pasteHandler),
    });

    try {
      if (currentEditor && pasteHandler) {
        currentEditor.removeEventListener("paste", pasteHandler);
        log("detachFromCurrentEditor:paste-listener-removed", {
          reason,
        });
      } else {
        log("detachFromCurrentEditor:no-listener-to-remove", {
          reason,
        });
      }
    } catch (err) {
      error("detachFromCurrentEditor:error", {
        reason,
        errorName: err?.name,
        errorMessage: err?.message,
        stack: err?.stack,
      });
    } finally {
      currentEditor = null;
      pasteHandler = null;
    }
  }

  function attachToEditor(editor, reason = "unknown") {
    attachCount += 1;

    log("attachToEditor:start", {
      attachCount,
      reason,
      editor: describeNode(editor),
      sameAsCurrentEditor: editor === currentEditor,
      hasExistingPasteHandler: Boolean(pasteHandler),
      currentEditor: describeNode(currentEditor),
      hasObserverA: Boolean(observerA),
      hasObserverB: Boolean(observerB),
    });

    if (!editor) {
      warn("attachToEditor:called-with-empty-editor", {
        attachCount,
        reason,
      });
      return;
    }

    if (editor === currentEditor && pasteHandler && currentEditor?.isConnected) {
      log("attachToEditor:already-attached-same-connected-editor", {
        attachCount,
        reason,
      });
      return;
    }

    if (currentEditor && pasteHandler) {
      detachFromCurrentEditor("reattach-to-editor");
    }

    currentEditor = editor;

    pasteHandler = (e) => {
      pasteCount += 1;

      log("paste:event-fired", {
        pasteCount,
        eventType: e.type,
        target: describeNode(e.target),
        currentTarget: describeNode(e.currentTarget),
        clipboardTypes: Array.from(e.clipboardData?.types ?? []),
        href: location.href,
      });

      setTimeout(() => {
        log("paste:setTimeout-fired", {
          pasteCount,
          currentEditorConnected: currentEditor?.isConnected ?? null,
          currentEditor: describeNode(currentEditor),
          foundEditor: describeNode(getEditor()),
          paragraphCount: getParagraphs().length,
        });

        sanitizeParagraphs("paste-timeout-0");
      }, 0);
    };

    editor.addEventListener("paste", pasteHandler);

    log("attachToEditor:paste-listener-attached", {
      attachCount,
      reason,
    });

    if (observerA) {
      observerA.disconnect();
      observerA = null;

      log("attachToEditor:observerA-disconnected", {
        attachCount,
        reason,
      });
    }

    ensureObserverB();
  }

  function ensureObserverB() {
    if (observerB) {
      log("ensureObserverB:already-running");
      return;
    }

    observerB = new MutationObserver((mutations) => {
      observerBCallbackCount += 1;

      const foundEditor = getEditor();
      const currentConnected = currentEditor?.isConnected ?? false;
      const sameEditor = foundEditor === currentEditor;

      log("observerB:callback", {
        observerBCallbackCount,
        mutationCount: mutations.length,
        foundEditorExists: Boolean(foundEditor),
        currentConnected,
        sameEditor,
        currentEditor: describeNode(currentEditor),
        foundEditor: describeNode(foundEditor),
      });

      // Case 1:
      // #prompt-textarea が完全に消えた
      if (!foundEditor) {
        log("observerB:editor-not-found", {
          observerBCallbackCount,
        });

        detachFromCurrentEditor("editor-not-found");

        if (observerB) {
          observerB.disconnect();
          observerB = null;

          log("observerB:disconnected", {
            observerBCallbackCount,
          });
        }

        waitForEditor("editor-not-found");
        return;
      }

      // Case 2:
      // 新しい #prompt-textarea が存在するが、currentEditor は古い detached node を指している
      // 今回のログで発生していたのはこのケース
      if (!currentConnected || !sameEditor) {
        warn("observerB:editor-replaced-detected", {
          observerBCallbackCount,
          currentConnected,
          sameEditor,
          currentEditor: describeNode(currentEditor),
          foundEditor: describeNode(foundEditor),
        });

        attachToEditor(foundEditor, "observerB-detected-replacement");
        return;
      }
    });

    observerB.observe(document.documentElement, {
      childList: true,
      subtree: true,
    });

    log("ensureObserverB:started");
  }

  function waitForEditor(reason = "initial") {
    log("waitForEditor:start", {
      reason,
      href: location.href,
      readyState: document.readyState,
      hasObserverA: Boolean(observerA),
      hasObserverB: Boolean(observerB),
      currentEditor: describeNode(currentEditor),
    });

    const editor = getEditor();

    if (editor) {
      log("waitForEditor:editor-already-present", {
        reason,
        editor: describeNode(editor),
      });

      attachToEditor(editor, `waitForEditor:${reason}`);
      return;
    }

    if (observerA) {
      log("waitForEditor:observerA-already-running", {
        reason,
      });
      return;
    }

    observerA = new MutationObserver((mutations) => {
      observerACallbackCount += 1;

      const found = getEditor();

      log("observerA:callback", {
        observerACallbackCount,
        mutationCount: mutations.length,
        found: Boolean(found),
        foundEditor: describeNode(found),
      });

      if (found) {
        log("observerA:editor-appeared", {
          observerACallbackCount,
        });

        attachToEditor(found, "observerA-editor-appeared");
      }
    });

    observerA.observe(document.documentElement, {
      childList: true,
      subtree: true,
    });

    log("waitForEditor:observerA-started", {
      reason,
    });
  }

  log("userscript:initialized", {
    version: SCRIPT_VERSION,
    href: location.href,
    readyState: document.readyState,
    userAgent: navigator.userAgent,
  });

  waitForEditor("initial");
})();
```
