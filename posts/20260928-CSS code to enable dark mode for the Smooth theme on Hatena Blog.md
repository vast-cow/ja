---
pubDatetime: 2026-09-28T00:05:00+09:00
title: "はてなブログのテーマ「Smooth」をダークモード対応にするCSS"
description: "`prefers-color-scheme` を利用し、OSの設定に応じて配色が自動で切り替わるようにしました。元の雰囲気やアクセントカラーをなるべく維持しつつ、記事本文からサイドバー、コメント、検索、フッターまで一通り調整しています。"
---

はてなブログのデザインテーマ「Smooth」を使っています。

シンプルで読みやすく、余計な装飾も少ないので気に入っているのですが、一つだけ欲しかったのが**ダークモード対応**です。

そこで、OSやブラウザの設定がダークモードになっている場合、自動的にブログも暗い配色へ切り替わるCSSを追加してみました。

JavaScriptは使いません。

CSSの `prefers-color-scheme` を利用するだけなので、比較的簡単に導入できます。

## こんな感じのダークモードにしたい

今回の方針は、単純に「背景を黒、文字を白」にするのではなく、**Smooth本来の雰囲気をできるだけ残したまま暗くする**ことです。

特に意識したのは次の点です。

- 真っ黒ではなく、少し緑がかったダークグレーを背景にする
- 真っ白な文字を避けて、コントラストを少し抑える
- Smoothで使われている緑系のリンクカラーを残す
- 記事だけでなく、サイドバーや検索、コメント、アーカイブなどもまとめて対応する
- コードブロックは元々暗いデザインなので、大きく変更しない

OSがライトモードなら従来のSmoothのまま、ダークモードなら自動的に暗いデザインになります。

## 追加するCSS

はてなブログの

**デザイン → カスタマイズ → デザインCSS**

を開き、以下のCSSを追加します。

```css
/* ==============================
   Smooth - Auto Dark Mode
   ============================== */

:root {
  color-scheme: light dark;
}

@media (prefers-color-scheme: dark) {

  /*
   * Base
   */
  html,
  body {
    background: #121616;
    color: #d9dddd;
  }

  #globalheader-container {
    background: #121616;
  }

  a {
    color: #d9dddd;
  }

  a:hover {
    color: #b7c3c3;
  }


  /*
   * Blog title
   */
  #title,
  #title a,
  #blog-description {
    color: #e5e8e8;
  }


  /*
   * Entry
   */
  .entry {
    background: #1a1f1f;
    border-color: #303838;
  }

  .entry-title,
  .entry-title a {
    color: #e5e8e8;
  }

  .entry-content {
    color: #d9dddd;
  }


  /*
   * Links
   *
   * Smooth の緑を維持しつつ
   * dark background で見やすくする
   */
  .entry-content a,
  .comment-box .comment a,
  .hatena-module-profile .id a,
  .archive-module-calendar .calendar-day a {
    color: #64d991;
  }

  .entry-content a:hover,
  .comment-box .comment a:hover,
  .hatena-module-profile .id a:hover,
  .archive-module-calendar .calendar-day a:hover {
    color: #8ae5ab;
  }


  /*
   * Date / secondary text
   */
  .date a,
  .date-last-updated,
  .entry-footer-section,
  .entry-footer-section a,
  .hatena-urllist .urllist-date-link a,
  #footer,
  #footer a {
    color: #94a3a3;
  }


  /*
   * Category labels
   */
  .categories a,
  .hatena-urllist .urllist-category-link {
    background: #293232;
    color: #d9dddd;
  }

  .hatena-urllist .urllist-category-link:hover {
    background: #354141;
  }


  /*
   * Headings
   */
  .entry-content h1,
  .entry-content h3,
  .entry-content h5 {
    border-color: #394343;
  }


  /*
   * Table of contents
   */
  .entry-content .table-of-contents {
    background: #222929;
  }


  /*
   * Blockquote
   */
  .entry-content blockquote {
    background: #1e2424;
    border-color: #394343;
  }

  .entry-content blockquote::before {
    color: #829393;
  }


  /*
   * Table
   */
  .entry-content table th,
  .entry-content table td {
    border-color: #394343;
  }

  .entry-content table th {
    background: #222929;
  }


  /*
   * Inline code
   */
  .entry-content code {
    background: #293232;
    color: #e3e8e8;
  }


  /*
   * Code block
   *
   * Smooth は元から dark なので
   * 大きく変更しない
   */
  .entry-content pre {
    background: #202727;
    color: #d6dddd;
  }


  /*
   * Keyword links
   */
  .entry-content a.keyword {
    text-decoration-color: #657575;
  }

  .entry-content a.keyword:hover {
    color: #b7c3c3;
    text-decoration-color: #94a3a3;
  }


  /*
   * Pager
   */
  .pager a {
    background: #222929;
    color: #d9dddd;
  }

  .pager a:hover {
    background: #2b3434;
  }


  /*
   * Header menu
   */
  .entry-header-menu a {
    background: #222929;
    border-color: #445050;
    color: #d9dddd;
  }

  .entry-header-menu a:hover {
    background: #2b3434;
  }


  /*
   * Comments
   */
  .comment-box {
    border-color: #394343;
  }

  .comment-box .leave-comment-title {
    background: #222929;
    border-color: #445050;
    color: #d9dddd;
  }

  .comment-box .leave-comment-title:hover {
    background: #2b3434;
  }

  .comment-box .comment .entry-comment {
    border-color: #394343;
    color: #94a3a3;
  }


  /*
   * Sidebar
   */
  #box2 {
    background: transparent;
  }

  .hatena-module {
    color: #d9dddd;
  }

  .hatena-module-title,
  .hatena-module-title a,
  .hatena-module-body a {
    color: #d9dddd;
  }

  .hatena-urllist li,
  #box2 .hatena-urllist > li:last-child {
    border-color: #445050;
  }


  /*
   * Search
   */
  .search-form,
  .search-result-form {
    background: #1a1f1f;
    border-color: #394343;
  }

  .search-module-input,
  .search-result-form .search-result-input {
    background: #1a1f1f;
    color: #e5e8e8;
  }

  .search-module-input::placeholder,
  .search-result-form .search-result-input::placeholder {
    color: #819090;
  }


  /*
   * Archive
   */
  .page-archive .archive-entry {
    background: #1a1f1f;
    border-color: #303838;
  }

  .page-archive .archive-heading {
    color: #e5e8e8;
  }


  /*
   * Footer
   */
  #footer {
    background: #181d1d;
  }

  .blog-controlls {
    background-color: rgb(24, 29, 29);
  }

  .blog-controlls-title a,
  .blog-controlls-title a:hover,
  .blog-controlls-title a:visited {
    color: rgb(229, 232, 232);
  }

  .blog-controlls-subscribe-btn {
    background-color: rgb(41, 50, 50);
    color: rgb(229, 232, 232);
  }

  .blog-controlls-subscribe-btn:hover {
    background-color: rgb(53, 65, 65);
  }

  .blog-controlls-subscribe-btn:visited {
    color: rgb(229, 232, 232);
  }

  @media (min-width: 768px) {
    .blog-controlls {
      background-color: transparent;
    }
  }

  #globalheader-container {
    background: rgb(18, 22, 22);
    filter: none;
  }

  #globalheader {
    filter: invert(1) hue-rotate(180deg);
  }

}
```

## ポイントは `prefers-color-scheme`

今回のCSSで重要なのが、ここです。

```css
@media (prefers-color-scheme: dark) {
  /* ダークモード用CSS */
}
```

`prefers-color-scheme: dark` は、ユーザーがOSやブラウザでダーク系の配色を選択している場合に適用されるメディアクエリです。

つまり、ブログ側に「ダークモード切り替えボタン」を実装しなくても、

- ライトモードの人 → 通常のSmooth
- ダークモードの人 → 今回追加したダークテーマ

という形で自動的に切り替わります。

また、

```css
:root {
  color-scheme: light dark;
}
```

も指定しています。

ブラウザに、このページがライト・ダーク両方のカラースキームに対応していることを伝えるための指定です。

## 真っ黒にはしていない

ダークモードというと、

```css
background: #000;
color: #fff;
```

のようにしたくなりますが、今回はそうしていません。

ベースの背景は、

```css
#121616
```

記事部分は、

```css
#1a1f1f
```

本文は、

```css
#d9dddd
```

としています。

完全な黒と白を組み合わせるより少しコントラストを抑えつつ、Smoothに合うように若干グリーン寄りのグレーで統一しています。

## Smoothの緑色は残す

Smoothらしさとして残したかったのが、リンクの緑です。

本文中のリンクなどは、

```css
.entry-content a {
  color: #64d991;
}
```

としています。

ホバー時は少し明るくして、

```css
.entry-content a:hover {
  color: #8ae5ab;
}
```

としました。

ダークモードでもリンクだと認識しやすく、Smoothの印象もそれほど変わりません。

## 記事本文以外も対応

最初は背景と本文だけ変更すればいいかと思ったのですが、実際にダークモードにしてみると細かい部分が気になります。

今回のCSSでは、記事本文に加えて、

- ブログタイトル
- 日付
- カテゴリー
- 見出し
- 目次
- 引用
- テーブル
- インラインコード
- コードブロック
- ページャー
- コメント
- サイドバー
- 検索フォーム
- アーカイブ
- フッター
- はてなブログのヘッダー

あたりも調整しています。

特に背景だけ暗くすると、ボーダーや補助テキストがライトモード用の色のまま残ってしまいがちです。

そのため、`#394343` や `#445050` といった暗めのボーダーカラーも合わせて指定しています。

## はてなのグローバルヘッダーも暗くする

少し特殊なのが最後の部分です。

```css
#globalheader-container {
  background: rgb(18, 22, 22);
  filter: none;
}

#globalheader {
  filter: invert(1) hue-rotate(180deg);
}
```

ブログ本体だけ暗くすると、上部のはてな側のヘッダーが浮いて見えるため、ここもダークモードに馴染むよう調整しています。

`invert()` と `hue-rotate()` を使っているので少々力技ですが、CSSだけで対応する方法としては手軽です。

## JavaScript不要なのがいい

この方法の良いところは、**CSSを追加するだけ**という点です。

ダークモード用のJavaScriptを書いたり、CookieやLocalStorageに設定を保存したりする必要はありません。

ユーザーが普段使っているOSの設定に従って、自動的に切り替わります。

もちろん「ブログだけライトモードにしたい」「手動切り替えボタンが欲しい」という用途には向きません。

しかし、

> OSがダークモードならブログもダークモードにする

というシンプルな仕様でよければ、`prefers-color-scheme` だけで十分だと思います。

## まとめ

Smoothは元々かなりシンプルなテーマなので、ダークモードとの相性も悪くありません。

今回のCSSではデザインそのものを作り直すのではなく、**Smoothのレイアウトや緑色のアクセントを残したまま、配色だけ自然にダークモード化する**ことを目指しました。

Smoothを使っていて、

「デザインは気に入っているけど、ダークモードにも対応したい」

という人は、デザインCSSへの追加だけで済むので試してみてください。

なお、はてなブログ側やSmooth側のHTML・CSSが将来変更された場合、一部の指定が効かなくなる可能性があります。その場合は該当するセレクタを調整してください。
