---
pubDatetime: 2026-09-27T13:53:00+09:00
title: "Chrome DevToolsで覚えておきたい実践テクニック集"
description: "Chrome DevTools には、DOM・イベント・通信・描画まわりの調査を効率化できる便利な機能が多数あります。  `inspect()` や `$0`、`monitorEvents()` など、日常のデバッグで使いやすい機能を用途別にまとめます。"
---

Chrome DevTools には、知っているとデバッグ効率がかなり上がる便利な機能があります。

実務で使いやすいものを挙げると、次あたりです。

## `inspect()`

指定した要素を DevTools 上で直接確認できます。

DOM 要素を渡すと、Elements パネルでその要素が選択された状態になります。

```javascript
inspect(document.querySelector('.button'))
inspect($0)
```

JavaScript で見つけた要素を、そのまま Elements で確認したいときに便利です。

---

## `$0` / `$1` / `$2`

Elements で選択した要素を Console から参照できます。

- `$0`：現在選択している要素
- `$1`：1つ前に選択していた要素
- `$2`：2つ前に選択していた要素

```javascript
$0
$0.textContent
$0.getBoundingClientRect()
```

Elements と Console を行き来しながら調査するときに便利です。

---

## `$()` / `$$()`

DevTools Console 専用の簡易セレクタです。

- `$()`：`querySelector()` のように1件取得
- `$$()`：`querySelectorAll()` のように複数取得

```javascript
$('.button')
$$('a')
```

ちょっとDOMを確認したいときに、毎回 `document.querySelector()` と書かずに済みます。

---

## `monitorEvents()`

指定した要素で発生するイベントを Console に表示できます。

```javascript
monitorEvents($0, 'click')
monitorEvents($0, ['click', 'input', 'change'])
```

イベント監視を解除するときは、

```javascript
unmonitorEvents($0)
```

を使います。

「この操作で何のイベントが発生している？」を調べたいときに便利です。

---

## `getEventListeners()`

対象要素に登録されているイベントリスナーを確認できます。

```javascript
getEventListeners($0)
```

`click` や `keydown` など、どんなイベントが登録されているのかを調べるときに使えます。

---

## `copy()`

値をクリップボードへコピーできます。

```javascript
copy($0.outerHTML)
copy(JSON.stringify(data, null, 2))
```

DOM やAPIレスポンスなどを、そのままエディタやチャットに貼りたいときに便利です。

---

## `console.table()`

配列やオブジェクトを表形式で表示できます。

```javascript
console.table(users)
```

オブジェクトの配列を見るときは、普通の `console.log()` よりかなり確認しやすくなります。

---

## `console.trace()`

現在の処理がどこから呼び出されたのか、スタックトレースを確認できます。

```javascript
console.trace()
```

「この関数、どこから呼ばれている？」という調査に便利です。

---

## DOM変更でブレーク

Elements で対象要素を右クリックし、

`Break on`

から次のような条件を指定できます。

- subtree modifications
- attribute modifications
- node removal

「誰がこのDOMを書き換えた？」を追跡するときにかなり強力です。

---

## Event Listener Breakpoints

Sources → Event Listener Breakpoints から設定できます。

`click`、`input`、`keydown` などのイベントが発生した瞬間に JavaScript の実行を停止できます。

イベントハンドラがどこに書かれているのか分からないときに便利です。

---

## XHR / fetch Breakpoints

Sources → XHR/fetch Breakpoints から設定できます。

URL の一部を指定すると、その通信が発生した瞬間に JavaScript の実行を停止できます。

「このAPIを呼んでいるコードはどこ？」を探すときに便利です。

---

## Network の Preserve log

Network パネルの `Preserve log` を有効にすると、ページ遷移やリロード後も通信履歴を残せます。

ログイン、OAuth、リダイレクトなど、ページ遷移をまたぐ問題の調査に便利です。

---

## Network の Copy as fetch / Copy as cURL

Network パネルのリクエストを右クリックすると、

- Copy as fetch
- Copy as cURL

などが使えます。

ブラウザで発生したリクエストを、そのままコードやターミナルで再現したいときに便利です。

---

## Request blocking

特定のJS、API、画像などを意図的にブロックできます。

例えば、

「このAPIが失敗したら画面はどうなるか」

「このJSが読み込めなかったらどうなるか」

といった異常系の確認に使えます。

---

## Local Overrides

Network 上の JavaScript、CSS、HTML などをローカルで差し替え、リロード後も変更を保持できます。

本番環境のコードを直接変更することなく、

「ここを修正したらどうなる？」

という検証ができます。

---

## Coverage

Command Menu を開いて、

```text
Ctrl/Cmd + Shift + P
```

`Show Coverage`

を実行すると、読み込まれた CSS / JavaScript のうち、実際にどれだけ使用されているか確認できます。

未使用CSSや、巨大なJavaScriptバンドルを調査するときに便利です。

---

## Performance monitor

Command Menu から

`Show Performance monitor`

を実行すると、

- CPU使用率
- JS heap
- DOM node数
- Event Listener数
- Layout / Style recalculation

などをリアルタイムで確認できます。

「操作しているうちにDOMやメモリが増え続けていないか」といった調査にも使えます。

---

## Rendering

Command Menu → `Show Rendering` から、レンダリング関連のデバッグ機能を利用できます。

例えば、

- Paint flashing
- Layout Shift Regions
- FPS meter
- CSS media feature emulation

などがあります。

再描画が多い箇所やレイアウトシフト、描画パフォーマンスを調べるときに便利です。

---

## まず覚えておきたい7つ

特に使用頻度が高いのは、この7つです。

```javascript
inspect(el)
$0
$$()
monitorEvents()
getEventListeners()
copy()
console.table()
```

全部覚える必要はありません。

まずは `$0`、`$$()`、`inspect()` あたりから使い始めるだけでも、Chrome DevTools でのDOM調査はかなり楽になります。
