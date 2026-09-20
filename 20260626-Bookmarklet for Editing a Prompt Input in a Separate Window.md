---
pubDatetime: 2026-06-26T14:30:00+09:00
title: "プロンプト入力欄を別ウィンドウで編集するブックマークレット"
description: "このブックマークレットは、ページ内のプロンプト入力欄を別ウィンドウで開き、広いテキストエリアで編集できるようにするためのものです。現在入力されている内容を読み取り、編集画面の初期値として表示します。編集後に Set ボタンを押すと、元のページの入力欄へ内容が反映されます。 目的 長いプロンプトをブラ…"
---

このブックマークレットは、ページ内のプロンプト入力欄を別ウィンドウで開き、広いテキストエリアで編集できるようにするためのものです。現在入力されている内容を読み取り、編集画面の初期値として表示します。編集後に `Set` ボタンを押すと、元のページの入力欄へ内容が反映されます。

## 目的

長いプロンプトをブラウザ上の小さな入力欄で編集するのは不便です。このブックマークレットを使うと、別ウィンドウに専用の編集画面を開き、より見やすいテキストエリアで内容を確認・編集できます。

主な用途は次のとおりです。

* 入力済みのプロンプトを別画面で編集する
* 長文のプロンプトを見やすく整える
* 改行を含むテキストを扱いやすくする
* 編集後の内容を元の入力欄へ戻す

## 使い方

まず、この JavaScript コードをブックマークレットとしてブラウザに登録します。登録後、対象のページを開いた状態でそのブックマークレットを実行します。

実行すると、現在のページタイトルを見出しにした編集用ウィンドウが開きます。そこには、元のページのプロンプト入力欄に入っていた内容が最初から表示されます。

内容を編集したら、`Set` ボタンを押します。すると、編集した内容が元のページのプロンプト入力欄に反映され、編集用ウィンドウは閉じられます。

## 表示テーマの自動切り替え

編集用ウィンドウは、ブラウザやOSのテーマ設定に合わせて自動的にライトテーマとダークテーマを切り替えます。

ライトテーマでは白背景と黒文字を使用し、ダークテーマでは暗い背景と明るい文字を使用します。これにより、普段使っている画面設定に近い見た目で編集できます。

## 現在値の読み込み

ブックマークレットを実行すると、最初に元ページの `#prompt-textarea` から現在の入力内容を読み取ります。その内容は、編集用HTMLを作成する時点であらかじめ `<textarea>` に埋め込まれます。

そのため、編集用ウィンドウを開いた後にあらためて元ページから内容を取得するのではなく、実行時点の内容がそのまま初期表示されます。

## 編集内容の反映

`Set` ボタンを押すと、編集用ウィンドウのテキストエリアに入力されている内容が行ごとに分割され、元ページのプロンプト入力欄へ反映されます。

空行も保持されるため、複数行のプロンプトや段落を含む文章でも扱いやすくなっています。

## 後片付け

編集用ウィンドウは一時的なHTMLを `Blob` として作成し、そのURLを使って開かれます。処理が完了した後は、`URL.revokeObjectURL()` によってそのURLを破棄します。

これにより、不要になった一時URLを残さずに済みます。ポップアップを開けなかった場合や、処理中にエラーが起きた場合にもURLを破棄するようになっています。

## 注意点

このブックマークレットは、対象ページに `#prompt-textarea` という要素が存在することを前提にしています。その要素が見つからない場合は、エラーメッセージが表示され、処理は行われません。

また、ブラウザの設定でポップアップがブロックされている場合、編集用ウィンドウを開けないことがあります。その場合は、対象ページでポップアップを許可する必要があります。

```javascript
(() => {
  const escapeHtml = (s) =>
    (s || '')
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  const promptTextarea = document.querySelector('#prompt-textarea');
  if (!promptTextarea) {
    alert('Could not find #prompt-textarea');
    return;
  }
  const escapedParentTitle = escapeHtml(document.title || 'Prompt Setter');
  const initialText = ((el) => {
    const nodes = Array.from(el.childNodes);
    return nodes.length
      ? nodes
          .map((node) =>
            3 === node.nodeType
              ? node.textContent || ''
              : 'BR' === node.nodeName
                ? ''
                : node.textContent || '',
          )
          .join('\n')
      : el.innerText || el.textContent || '';
  })(promptTextarea);
  const blob = new Blob(
    [
      `<!doctype html>\n<html lang="en">\n<head>\n  <meta charset="utf-8">\n  <title>${escapedParentTitle}</title>\n  <style>\n    :root {\n      color-scheme: light dark;\n      --bg: #ffffff;\n      --fg: #111111;\n      --muted: #333333;\n      --border: #cccccc;\n      --textarea-bg: #ffffff;\n      --textarea-fg: #111111;\n      --button-bg: #f3f3f3;\n      --button-fg: #111111;\n      --button-border: #999999;\n    }\n\n    @media (prefers-color-scheme: dark) {\n      :root {\n        --bg: #111111;\n        --fg: #eeeeee;\n        --muted: #cccccc;\n        --border: #444444;\n        --textarea-bg: #1e1e1e;\n        --textarea-fg: #eeeeee;\n        --button-bg: #2a2a2a;\n        --button-fg: #eeeeee;\n        --button-border: #666666;\n      }\n    }\n\n    body {\n      font-family: sans-serif;\n      margin: 16px;\n      background: var(--bg);\n      color: var(--fg);\n    }\n\n    .page-title {\n      font-size: 24px;\n      font-weight: 700;\n      line-height: 1.4;\n      margin: 0 0 16px;\n      word-break: break-word;\n    }\n\n    textarea {\n      width: 100%;\n      height: 240px;\n      box-sizing: border-box;\n      font-family: monospace;\n      font-size: 14px;\n      background: var(--textarea-bg);\n      color: var(--textarea-fg);\n      border: 1px solid var(--border);\n    }\n\n    button {\n      margin-top: 12px;\n      padding: 8px 16px;\n      font-size: 14px;\n      background: var(--button-bg);\n      color: var(--button-fg);\n      border: 1px solid var(--button-border);\n      border-radius: 4px;\n      cursor: pointer;\n    }\n\n    button:hover {\n      filter: brightness(0.96);\n    }\n\n    @media (prefers-color-scheme: dark) {\n      button:hover {\n        filter: brightness(1.15);\n      }\n    }\n\n    .msg {\n      margin-top: 12px;\n      color: var(--muted);\n      white-space: pre-wrap;\n    }\n  </style>\n</head>\n<body>\n  <div class="page-title">${escapedParentTitle}</div>\n  <textarea id="usertext" placeholder="Enter text here">${escapeHtml(initialText)}</textarea>\n  <br>\n  <button id="setButton" type="button">Set</button>\n  <div class="msg" id="msg"></div>\n</body>\n</html>`,
    ],
    { type: 'text/html' },
  );
  const url = URL.createObjectURL(blob);
  let revoked = !1;
  const revokeUrl = () => {
    if (!revoked) {
      URL.revokeObjectURL(url);
      revoked = !0;
    }
  };
  const child = window.open(url, '_blank');
  if (!child) {
    revokeUrl();
    alert('Could not open popup');
    return;
  }
  const setup = () => {
    try {
      const childDoc = child.document;
      const usertextEl = childDoc.getElementById('usertext');
      const setButton = childDoc.getElementById('setButton');
      const msgEl = childDoc.getElementById('msg');
      if (!usertextEl || !setButton || !msgEl) {
        setTimeout(setup, 50);
        return;
      }
      try {
        child.addEventListener('beforeunload', revokeUrl, { once: !0 });
      } catch (_) {}
      setButton.addEventListener('click', async () => {
        try {
          if (!child.opener || child.opener.closed)
            throw new Error('Cannot access parent window');
          const openerDoc = child.opener.document;
          const promptTextarea = openerDoc.querySelector('#prompt-textarea');
          if (!promptTextarea)
            throw new Error('Could not find #prompt-textarea');
          const usertext = usertextEl.value;
          const newNodes = usertext
            .replace(/\r\n/g, '\n')
            .split('\n')
            .map((promptLine) => {
              const p = openerDoc.createElement('p');
              '' === promptLine
                ? p.appendChild(openerDoc.createElement('br'))
                : (p.textContent = promptLine);
              return p;
            });
          promptTextarea.replaceChildren(...newNodes);
          await new Promise((resolve) => setTimeout(resolve, 0));
          promptTextarea.lastChild &&
            promptTextarea.lastChild.scrollIntoView &&
            promptTextarea.lastChild.scrollIntoView();
          revokeUrl();
          child.close();
        } catch (err) {
          msgEl.textContent = 'Error: ' + err.message;
          revokeUrl();
        }
      });
    } catch (e) {
      setTimeout(setup, 50);
    }
  };
  setup();
})();
```
