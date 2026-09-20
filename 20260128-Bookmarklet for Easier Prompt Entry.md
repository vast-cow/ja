---
pubDatetime: 2026-01-28T02:35:45+09:00
title: "プロンプト入力を簡単にするブックマークレット"
description: "このツールの動作、仕組み、使い方を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## 概要

このJavaScriptコードは、特定の入力欄にテキストを簡単に挿入するための**ブックマークレット**です。
手動でコピー＆ペーストしたり、改行を整えたりする手間を省き、別ウィンドウから効率よくテキストを入力できます。

主な目的は、**複数行のテキストを整理された形で素早く入力すること**です。

---

## このツールの動作

このスクリプトを実行すると、次の処理が行われます：

* 小さな**ポップアップウィンドウ**が開く
* テキストを入力できる**入力欄（textarea）**が表示される
* **「Set」ボタン**で入力内容を反映
* 元のページの特定フィールド（`#prompt-textarea`）にテキストを挿入
* 改行ごとに**段落として整形**される

---

## 仕組み（概要）

### 1. 入力用ウィンドウの生成

* `Blob`を使って簡易HTMLページを動的に生成
* 新しいウィンドウとして表示
* 以下の要素を含む：

  * ページタイトル
  * テキスト入力欄
  * ボタン

---

### 2. 元ページへの反映

「Set」ボタンを押すと：

* 元のページ（親ウィンドウ）にアクセス
* `#prompt-textarea`を取得
* 入力内容を**改行ごとに分割**
* 各行を段落（`<p>`）として追加
* 元の内容を削除して置き換え
* 最後の行まで自動スクロール
* ポップアップを閉じる

---

## 使い方

### 手順1：ブックマークレットを作成

* このコードをそのままブックマークに登録
* URL欄に `javascript:` から始まる形で保存

---

### 手順2：対象ページを開く

* `#prompt-textarea` というIDを持つ入力欄があるページを開く

---

### 手順3：ブックマークレットを実行

* 作成したブックマークをクリック
* ポップアップが表示される

---

### 手順4：テキストを入力

* 複数行のテキストを入力または貼り付け

---

### 手順5：反映

* 「Set」ボタンをクリック
* 元ページに自動で反映される

---

## 主な特徴

* **複数行対応**：改行を維持して入力可能
* **自動整形**：空行も適切に処理
* **一括置き換え**：既存の内容をクリアして反映
* **シンプルUI**：無駄のない操作画面

---

## 制限事項

* `#prompt-textarea` が存在するページでのみ動作
* ポップアップブロックが有効だと使用不可
* 同一オリジン制約により動作しない場合がある

---

## まとめ

このブックマークレットは、テキスト入力作業を効率化するためのシンプルなツールです。
特に、整形された複数行テキストを頻繁に入力する場面で有効に機能します。

```javascript
javascript:(()=>{const escapedParentTitle=(document.title||%22Prompt Setter%22).replace(/&/g,%22&amp;%22).replace(/</g,%22&lt;%22).replace(/>/g,%22&gt;%22);const blob=new Blob([`<!doctype html>\n<html lang=%22en%22>\n<head>\n  <meta charset=%22utf-8%22>\n  <title>${escapedParentTitle}</title>\n  <style>\n    body {\n      font-family: sans-serif;\n      margin: 16px;\n    }\n    .page-title {\n      font-size: 24px;\n      font-weight: 700;\n      line-height: 1.4;\n      margin: 0 0 16px;\n      word-break: break-word;\n    }\n    textarea {\n      width: 100%25;\n      height: 240px;\n      box-sizing: border-box;\n      font-family: monospace;\n      font-size: 14px;\n    }\n    button {\n      margin-top: 12px;\n      padding: 8px 16px;\n      font-size: 14px;\n    }\n    .msg {\n      margin-top: 12px;\n      color: #333;\n%20%20%20%20%20%20white-space:%20pre-wrap;\n%20%20%20%20}\n%20%20%3C/style%3E\n%3C/head%3E\n%3Cbody%3E\n%20%20%3Cdiv%20class=%22page-title%22%3E${escapedParentTitle}%3C/div%3E\n%20%20%3Ctextarea%20id=%22usertext%22%20placeholder=%22Enter%20text%20here%22%3E%3C/textarea%3E\n%20%20%3Cbr%3E\n%20%20%3Cbutton%20id=%22setButton%22%20type=%22button%22%3ESet%3C/button%3E\n%20%20%3Cdiv%20class=%22msg%22%20id=%22msg%22%3E%3C/div%3E\n%3C/body%3E\n%3C/html%3E%60],{type:%22text/html%22});const%20url=URL.createObjectURL(blob);const%20child=window.open(url,%22_blank%22);if(!child){alert(%22Could%20not%20open%20popup%22);return}const%20setup=()=%3E{try{const%20childDoc=child.document;const%20usertextEl=childDoc.getElementById(%22usertext%22);const%20setButton=childDoc.getElementById(%22setButton%22);const%20msgEl=childDoc.getElementById(%22msg%22);if(!usertextEl||!setButton||!msgEl){setTimeout(setup,50);return}setButton.addEventListener(%22click%22,async()=%3E{try{if(!child.opener||child.opener.closed)throw%20new%20Error(%22Cannot%20access%20parent%20window%22);const%20openerDoc=child.opener.document;const%20promptTextarea=openerDoc.querySelector(%22#prompt-textarea%22);if(!promptTextarea)throw%20new%20Error(%22Could%20not%20find%20#prompt-textarea%22);const%20usertext=usertextEl.value;const%20promptInputList=usertext.replace(/\r\n/g,%22\n%22).split(%22\n%22);const%20childNodesSaved=Array.from(promptTextarea.childNodes);for(const%20promptLine%20of%20promptInputList){const%20p=openerDoc.createElement(%22p%22);%22%22===promptLine?p.appendChild(openerDoc.createElement(%22br%22)):p.textContent=promptLine;promptTextarea.appendChild(p)}for(const%20childNodeSaved%20of%20childNodesSaved)promptTextarea.removeChild(childNodeSaved);await%20new%20Promise(resolve=%3EsetTimeout(resolve,0));promptTextarea.lastChild&&promptTextarea.lastChild.scrollIntoView&&promptTextarea.lastChild.scrollIntoView();child.close()}catch(err){msgEl.textContent=%22Error:%20%22+err.message}})}catch(e){setTimeout(setup,50)}};setup()})();
```
