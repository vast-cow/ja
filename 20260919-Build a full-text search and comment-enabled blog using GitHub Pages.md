---
title: "GitHub Pagesで全文検索・コメント対応ブログを構築する"
description: "推奨構成、Static Site Generator、検索対象を記事本文だけにするを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-09-19T11:48:36.439Z
---

GitHub Pages 上で「Markdownベースのブログ」「全文検索」「コメント」を実現するなら、次の構成が扱いやすいです。

## 推奨構成

```text
GitHub Repository
│
├─ src/content/blog/*.md / *.mdx
│        │
│        ▼
│   Astro
│   静的HTML生成
│        │
│        ▼
│   Pagefind
│   全文検索インデックス生成
│        │
│        ▼
├─ dist/
│   ├─ index.html
│   ├─ posts/...
│   └─ pagefind/...
│
│   GitHub Actions
│        │
│        ▼
└─ GitHub Pages
     │
     ├─ 記事閲覧
     ├─ Pagefind全文検索
     └─ giscus
          │
          ▼
       GitHub Discussions
```

GitHub Pages 自体は PHP/Python/Ruby 等のサーバーサイド処理を実行しない静的ホスティングです。一方、GitHub Actions で任意の静的サイトジェネレーターをビルドして Pages にデプロイできます。([GitHub Docs][1])

### 第一候補

| レイヤー                  | 採用候補               |
| --------------------- | ------------------ |
| コンテンツ                 | Markdown / MDX     |
| Static Site Generator | **Astro**          |
| 全文検索                  | **Pagefind**       |
| コメント                  | **giscus**         |
| コメント保存先               | GitHub Discussions |
| CI/CD                 | GitHub Actions     |
| Hosting               | GitHub Pages       |
| ソース管理                 | GitHub             |

この組み合わせなら、**検索サーバー・DB・アプリケーションサーバーを持たずに運用**できます。

---

## 1. Static Site Generator

### Astro を第一候補にする場合

ブログ記事を、

```text
src/content/blog/
├─ 2026-09-01-first-post.md
├─ 2026-09-10-github-pages.md
└─ 2026-09-19-pagefind.md
```

のように配置します。

Frontmatter は例えば、

```yaml
---
title: "GitHub Pagesでブログを作る"
description: "Astro + Pagefind + giscus の構成"
date: 2026-09-19
tags:
  - GitHub
  - Astro
  - Pagefind
commentId: "2026-09-19-pagefind"
---
```

程度を持たせます。

Astro は GitHub Pages 向けの公式デプロイ手順を用意しており、GitHub Actions からプリレンダリング済みサイトを公開できます。([Astro Docs][2])

### 他の候補との比較

|                    | Astro | Hugo | Jekyll |
| ------------------ | ----- | ---- | ------ |
| Markdownブログ        | ◎     | ◎    | ◎      |
| GitHub Pages       | ◎     | ◎    | ◎      |
| Pagefind連携         | ◎     | ◎    | ○      |
| UIカスタマイズ           | ◎     | ○    | ○      |
| ビルド速度              | ○     | ◎    | △      |
| JS/TSとの親和性         | ◎     | △    | △      |
| GitHub Pages標準との近さ | ○     | ○    | ◎      |
| 将来の機能追加            | ◎     | ○    | △      |

**記事中心で極力シンプル**なら Hugo もかなり有力です。

一方、

* 検索UIを作り込みたい
* MDXを使いたい
* Web Components / React / Vueを一部使いたい
* 後から機能を増やしたい

なら Astro の方が構成しやすいでしょう。

Jekyll は GitHub Pages との親和性が高いですが、今回は Pagefind のような後処理を入れるため、結局 GitHub Actions によるカスタムビルドが便利です。GitHub も Jekyll 以外のジェネレーターについて Actions を使ったビルド・公開をサポートしています。([GitHub Docs][3])

---

# 2. 全文検索：Pagefind

ここは **Pagefind がかなり適しています**。

ビルドフローを、

```text
Markdown
   ↓
Astro build
   ↓
静的HTML
   ↓
Pagefind
   ↓
検索インデックス付き静的サイト
```

とします。

例えば概念的には、

```bash
npm run build
npx pagefind --site dist
```

です。

Pagefind は生成されたHTMLを解析して検索インデックスを生成するため、検索用APIサーバーが不要です。

### 日本語対応

重要なのはここです。

Pagefind は日本語 `ja` を明示的にサポートしており、日本語・中国語・韓国語については空白区切りではない文章のセグメンテーションにも対応しています。`npx pagefind` では、この特殊言語対応を含む extended release がデフォルトです。([Pagefind][4])

したがって、

```text
GitHub Pagesで全文検索を実装する
```

のような日本語本文についても検索対象にできます。

---

## 検索対象を記事本文だけにする

ページ全体を検索対象にすると、

* ナビゲーション
* footer
* 関連記事
* サイドバー

などまでインデックスされます。

そのため記事レイアウトを、

```html
<article data-pagefind-body>
  ...
</article>
```

としておくのがよいです。

Pagefind は `data-pagefind-body` によってインデックス対象領域を限定できます。([Pagefind][5])

つまり、

```text
Header            ← 対象外

記事タイトル
記事本文           ← Pagefind対象
コード
見出し

関連記事           ← 対象外
giscusコメント     ← 対象外
Footer             ← 対象外
```

という状態にできます。

giscus のコメントはビルド後にブラウザ上でロードされるので、そもそも Pagefind の静的インデックスには含まれません。ブログ検索としてはこちらの方が自然です。

---

## タグ絞り込みも可能

Pagefind にはフィルター機構があります。([Pagefind][6])

したがって将来的には、

```text
検索
┌───────────────────────────────┐
│ github pages                  │
└───────────────────────────────┘

タグ
☑ GitHub
□ Astro
□ Linux
□ Python

12件
```

のような検索UIも構築できます。

例えば記事側に、

```html
<span data-pagefind-filter="tag">
  GitHub
</span>
```

などを生成します。

---

# 3. コメント：giscus

GitHub Pages と非常に相性がいいのが **giscus** です。

仕組みは、

```text
ブログ記事
    │
    │ giscus iframe
    ▼
GitHub Discussions
    │
    ├─ コメント
    ├─ 返信
    └─ Reaction
```

です。

独自DBを用意する必要はありません。giscus はコメントを GitHub Discussions に保持し、コメント・リアクションをブログ側に表示します。([Giscus][7])

---

## コメント用Repositoryは分けてもよい

例えば、

```text
myname/blog
    └─ ブログ本体

myname/blog-comments
    └─ GitHub Discussions
```

という構成です。

これは特に、

```text
blog repository
    Private

blog-comments repository
    Public
```

にしたい場合に有効です。

giscus は訪問者が Discussion を閲覧するため、接続先Repositoryを **public** にする必要があります。さらに giscus App のインストールと Discussions の有効化が必要です。([Giscus][7])

---

## コメントと記事の紐付け

giscus は、

* pathname
* URL
* title
* og:title
* 特定文字列

などによって記事とDiscussionを対応付けられます。([Giscus][7])

単純なサイトなら、

```text
pathname
```

で十分です。

ただし長期運用するなら、私は **記事固有ID** を持たせる構成を選びます。

例えば、

```yaml
commentId: "20260919-pagefind"
```

として、

```text
記事
/blog/pagefind/

↓

commentId
20260919-pagefind

↓

GitHub Discussion
```

とします。

これなら、

```text
/blog/pagefind/
    ↓ URL変更
/articles/pagefind/
```

となってもコメントの関連付けを維持しやすくなります。

タイトルを変更しても影響を受けません。

---

# 4. giscus の制約

最大の制約は、

> **コメントする人にもGitHubアカウントが必要**

という点です。

giscus では訪問者が GitHub OAuth を使って投稿するか、GitHub Discussion 上で直接コメントします。([GitHub][8])

そのため対象読者が、

```text
エンジニア
OSSユーザー
GitHubユーザー
```

なら非常に適しています。

逆に一般消費者向けブログで、

```text
名前
メール
本文
[送信]
```

という匿名・準匿名コメントを想定するなら、giscus は要件に合いません。その場合は外部コメントサービス、または Cloudflare Workers / Supabase 等を使ったコメントAPIを別途持つ構成になります。

---

# 5. GitHub Actions

デプロイパイプラインはシンプルにします。

```text
git push
   ↓
GitHub Actions
   │
   ├─ npm install
   │
   ├─ Astro build
   │
   ├─ Pagefind index
   │
   └─ Pages artifact
   ↓
GitHub Pages
```

GitHub Pages は現在、カスタムActionsワークフローによる任意の静的サイトビルドを正式にサポートしています。([GitHub Docs][1])

したがって、`gh-pages` ブランチを人間が管理する必要もありません。

---

# 6. Repository構成案

最終的には例えばこうします。

```text
blog/
├─ .github/
│  └─ workflows/
│     └─ deploy.yml
│
├─ src/
│  ├─ components/
│  │  ├─ Search.astro
│  │  ├─ Comments.astro
│  │  ├─ Header.astro
│  │  └─ Footer.astro
│  │
│  ├─ content/
│  │  └─ blog/
│  │     ├─ post-a.md
│  │     ├─ post-b.md
│  │     └─ post-c.md
│  │
│  ├─ layouts/
│  │  └─ BlogPost.astro
│  │
│  └─ pages/
│     ├─ index.astro
│     ├─ search.astro
│     └─ blog/
│
├─ public/
│  ├─ favicon.svg
│  └─ ...
│
├─ astro.config.mjs
├─ package.json
└─ tsconfig.json
```

生成後は、

```text
dist/
├─ index.html
├─ search/
├─ blog/
└─ pagefind/
   ├─ pagefind.js
   ├─ pagefind-ui.js
   └─ 検索index...
```

となります。

---

# 7. 検索ページ

例えば、

```text
/
├─ 記事一覧
│
├─ /blog/xxx/
│      記事
│      ↓
│      giscusコメント
│
└─ /search/
       ↓
   Pagefind UI
```

とします。

ヘッダーに、

```text
Blog        Tags        About        🔎 Search
```

を置き、

```text
/search/?q=github
```

のように検索状態をURLに持たせると、検索結果へのリンク共有にも対応しやすくなります。

---

# 8. この構成のメリット

運用面がかなり軽くなります。

```text
                管理対象
──────────────────────────
Web Server       なし
Database         なし
Search Server    なし
Comment Server   なし
CMS Server       なし

GitHub Repo       あり
GitHub Actions    あり
GitHub Pages      あり
GitHub Discussionsあり
```

つまり障害ポイントが少ない。

記事の追加も基本的に、

```text
Markdown追加
      ↓
git push
      ↓
自動build
      ↓
検索index更新
      ↓
公開
```

だけです。

---

## 構成としてはこれを基準にするのがよいと思います

```text
           GitHub
             │
      ┌──────┴──────┐
      │             │
 Blog Repository  Comments Repository
      │             │
 Markdown       Discussions
      │             ▲
    Astro           │
      │           giscus
   Pagefind          │
      │             │
      └──────┬──────┘
             │
       GitHub Pages
```

特に **Astro + Pagefind + giscus + GitHub Actions + GitHub Pages** は、今回の「静的ブログ・日本語全文検索・コメント」という要件にかなりよく一致します。

次に設計を進めるなら、**①URL設計、②Markdown Frontmatter仕様、③Pagefind検索UI、④giscusのDiscussionマッピング、⑤GitHub Actions の `deploy.yml`** まで決めると、そのまま実装に入れる状態になります。

[1]: https://docs.github.com/ja/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages?utm_source=chatgpt.com "GitHub ページでのカスタム ワークフローの使用 - GitHubドキュメント"
[2]: https://docs.astro.build/ja/guides/deploy/github/?utm_source=chatgpt.com "AstroサイトをGitHub Pagesにデプロイする | Docs"
[3]: https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site?utm_source=chatgpt.com "Creating a GitHub Pages site - GitHub Docs"
[4]: https://pagefind.app/docs/multilingual/?utm_source=chatgpt.com "Multilingual search | Pagefind"
[5]: https://pagefind.app/docs/indexing/?utm_source=chatgpt.com "Configuring what content is indexed | Pagefind"
[6]: https://pagefind.app/docs/filtering/?utm_source=chatgpt.com "Setting up filters | Pagefind"
[7]: https://giscus.app/?utm_source=chatgpt.com "giscus"
[8]: https://github.com/giscus/giscus?utm_source=chatgpt.com "GitHub - giscus/giscus: A commenting system powered by GitHub Discussions. :speech_balloon: :gem: · GitHub"
