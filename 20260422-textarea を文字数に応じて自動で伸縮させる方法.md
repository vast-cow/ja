---
pubDatetime: 2026-04-22T22:45:06+09:00
title: "textarea を文字数に応じて自動で伸縮させる方法"
description: "フォームの入力欄で、textarea の内容が増えたら高さも自然に伸びてほしい、という場面は多いです。 この実装はシンプルで、JavaScript で scrollHeight を使うだけです。 基本の考え方 やることは2段階です。 1. いったん高さを auto に戻す 2. 中身全体が入る高さ…"
---

フォームの入力欄で、`textarea` の内容が増えたら高さも自然に伸びてほしい、という場面は多いです。
この実装はシンプルで、JavaScript で `scrollHeight` を使うだけです。

### 基本の考え方

やることは2段階です。

1. いったん高さを `auto` に戻す
2. 中身全体が入る高さ (`scrollHeight`) を `height` に設定する

```js
function autoResize(textarea) {
  textarea.style.height = "auto";
  textarea.style.height = textarea.scrollHeight + "px";
}
```

### 入力時に反映する

`input` イベントでこの関数を呼べば、文字列内容に応じて `textarea` が伸縮します。

```js
const textarea = document.getElementById("message");

textarea.addEventListener("input", () => {
  autoResize(textarea);
});
```

### CSS で合わせておくこと

```css
textarea {
  width: 100%;
  min-height: 120px;
  resize: none;
  overflow: hidden;
  box-sizing: border-box;
}
```

`resize: none;` で手動リサイズを無効にし、`overflow: hidden;` でスクロールバーが出にくくなります。

### コピペ用の最小実装

```html
<textarea id="message" placeholder="入力してください"></textarea>

<style>
  textarea {
    width: 100%;
    min-height: 120px;
    resize: none;
    overflow: hidden;
    box-sizing: border-box;
  }
</style>

<script>
  const textarea = document.getElementById("message");

  function autoResize(textarea) {
    textarea.style.height = "auto";
    textarea.style.height = textarea.scrollHeight + "px";
  }

  textarea.addEventListener("input", () => {
    autoResize(textarea);
  });

  autoResize(textarea);
</script>
```

### 補足

JavaScript で `textarea.value` を書き換えた場合は、自動では高さが更新されません。
そのため、値を変更した直後に `autoResize(textarea)` を呼ぶのがポイントです。

短くまとめると、`textarea` の自動伸縮は **`scrollHeight` を `height` に反映する** 実装で実現できます。
シンプルで再利用しやすく、別ページにもそのまま持っていける定番パターンです。
