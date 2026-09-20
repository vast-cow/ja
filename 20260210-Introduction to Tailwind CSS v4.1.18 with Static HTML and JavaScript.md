---
pubDatetime: 2026-02-10T12:05:08+09:00
title: "Tailwind CSS v4.1.18 入門（静的 HTML + JavaScript 編）"
description: "作るもの、ディレクトリ構成、Tailwind CSS をインストール（v4.1.18）を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

![screenshot.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4339611/a1b35182-acec-48f2-a734-0742daee9e4d.jpeg)

*“使った分だけ生成”で CSS を最小化しつつ、凝った UI と Light/Dark 切替を作る*

Tailwind CSS は「ユーティリティファースト」な CSS フレームワークですが、静的サイトで特に効くのは次の点です。

* Tailwind が **HTML/JS をスキャン**
* 実際に使っているクラスだけを **CSS として生成**
* 結果として `output.css` が **小さく**なる

この記事では **Tailwind CSS v4.1.18** を前提に、**静的 HTML + vanilla JS** で「凝ったデザイン」のページと **Light/Dark トグル**を作り、ビルドと運用まで通します。



## 作るもの

* フレームワークなし（静的 HTML + JS）
* Tailwind CLI で CSS を生成
* ガラス風ヘッダー、グラデ背景、ブラーのブロブ、薄いグリッドなどの “凝った” 見た目
* Light/Dark 切替（`<html>` に `.dark` を付け外し）
* 設定は `localStorage` に保存



## ディレクトリ構成

```
project/
  index.html
  src/
    app.js
    input.css
  dist/
    output.css   # 生成物
  tailwind.config.js
  package.json
```



## 1) Tailwind CSS をインストール（v4.1.18）

プロジェクト直下で実行：

```bash
npm init -y
npm i -D tailwindcss
npx tailwindcss init
```



## 2) `tailwind.config.js`（スキャン対象の設定）

Tailwind は `content` に指定したファイルからクラスを抽出します。静的サイトではここが最重要です。

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./index.html",
    "./src/**/*.{js,html}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

> **ポイント**：クラスを書いているファイル（HTML/JS）が `content` に入っていないと、CSS が生成されず「デザインが反映されない」状態になります。



## 3) `src/input.css`（Tailwind v4 の書き方 + dark の落とし穴対策）

Tailwind v4 は、基本的に **`@import "tailwindcss";`** を使います。
さらに今回は JS で `.dark` を付け外しして切替したいので、**dark variant を class ベースに上書き**します。

`src/input.css` を作成：

```css
@import "tailwindcss";

/* `dark:` を `.dark` クラスで発火させる（手動トグル用） */
@custom-variant dark (&:where(.dark, .dark *));
```

### なぜこれが必要？

Tailwind v4 では `dark:` が **OS のテーマ（prefers-color-scheme）**に反応する挙動になりがちです。
その場合、JS で `<html class="dark">` を付けても `dark:*` が効かず、**トグルが動かない**原因になります。

上の `@custom-variant` を入れると、`dark:` が `.dark` に反応するようになります。



## 4) CSS をビルドする

### 1回だけビルド（本番向け）

```bash
npx tailwindcss -i ./src/input.css -o ./dist/output.css --minify
```

### watch（開発向け）

```bash
npx tailwindcss -i ./src/input.css -o ./dist/output.css --watch
```



## 5) 凝ったデザインの HTML 例（そのまま使える）

あなたが提示した “凝った” サンプルをそのまま `index.html` に置きます（以下）。

> **注意**：`<link rel="stylesheet" href="./dist/output.css" />` の相対パスは、`index.html` と `dist/` が同階層である前提です。

```html
<!doctype html>
<html lang="en" class="h-full">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Fancy Tailwind Theme Toggle (Static)</title>

    <link rel="stylesheet" href="./dist/output.css" />
  </head>

  <body class="h-full antialiased selection:bg-indigo-500/30 selection:text-slate-900 dark:selection:text-slate-100">
    <!-- Background -->
    <div aria-hidden="true" class="fixed inset-0 -z-10">
      <div
        class="absolute inset-0 bg-gradient-to-br from-slate-50 via-indigo-50 to-rose-50 dark:from-slate-950 dark:via-indigo-950/30 dark:to-rose-950/20"
      ></div>

      <!-- soft blobs -->
      <div class="absolute -top-20 left-1/2 h-72 w-72 -translate-x-1/2 rounded-full bg-indigo-400/20 blur-3xl dark:bg-indigo-400/10"></div>
      <div class="absolute top-40 -left-20 h-80 w-80 rounded-full bg-rose-400/20 blur-3xl dark:bg-rose-400/10"></div>
      <div class="absolute bottom-0 right-0 h-96 w-96 rounded-full bg-emerald-400/15 blur-3xl dark:bg-emerald-400/10"></div>

      <!-- subtle grid -->
      <div class="absolute inset-0 bg-[linear-gradient(to_right,rgba(15,23,42,0.06)_1px,transparent_1px),linear-gradient(to_bottom,rgba(15,23,42,0.06)_1px,transparent_1px)] bg-[size:48px_48px] opacity-40 dark:opacity-20"></div>
    </div>

    <div class="min-h-full">
      <!-- Header -->
      <header class="sticky top-0 z-20 border-b border-slate-200/60 bg-white/60 backdrop-blur-xl dark:border-slate-800/60 dark:bg-slate-950/40">
        <div class="mx-auto flex max-w-5xl items-center justify-between px-4 py-4">
          <div class="flex items-center gap-3">
            <div class="relative">
              <div class="h-10 w-10 rounded-2xl bg-gradient-to-br from-indigo-600 to-rose-500 shadow-sm"></div>
              <div class="absolute -bottom-1 -right-1 h-5 w-5 rounded-full bg-white shadow ring-1 ring-slate-200 dark:bg-slate-950 dark:ring-slate-800"></div>
            </div>
            <div>
              <p class="text-sm font-semibold tracking-tight text-slate-900 dark:text-slate-100">Aurora UI</p>
              <p class="text-xs text-slate-600 dark:text-slate-400">Static Tailwind + Theme Toggle</p>
            </div>
          </div>

          <!-- Toggle -->
          <div class="flex items-center gap-3">
            <div class="hidden text-xs text-slate-600 dark:text-slate-400 sm:block" id="themeHint">
              Theme follows your choice
            </div>

            <button
              id="themeToggle"
              type="button"
              class="group relative inline-flex items-center gap-3 rounded-2xl border border-slate-200/70 bg-white/70 px-3 py-2 shadow-sm backdrop-blur transition hover:bg-white/90 active:translate-y-px dark:border-slate-800/70 dark:bg-slate-900/60 dark:hover:bg-slate-900/80"
              aria-label="Toggle theme"
            >
              <span class="text-xs font-semibold text-slate-700 dark:text-slate-200" id="themeModeLabel">Dark</span>

              <!-- switch track -->
              <span class="relative inline-flex h-6 w-11 items-center rounded-full bg-slate-200 p-1 transition dark:bg-slate-700">
                <!-- knob -->
                <span
                  id="themeKnob"
                  class="inline-block h-4 w-4 translate-x-0 rounded-full bg-white shadow-sm ring-1 ring-slate-200 transition-transform duration-300 dark:translate-x-5 dark:bg-slate-950 dark:ring-slate-700"
                ></span>
              </span>

              <!-- icon -->
              <span class="grid h-6 w-6 place-items-center rounded-xl bg-slate-100 text-sm ring-1 ring-slate-200 transition dark:bg-slate-800 dark:ring-slate-700" id="themeIcon">
                🌙
              </span>

              <!-- subtle glow on hover -->
              <span class="pointer-events-none absolute inset-0 -z-10 rounded-2xl opacity-0 blur-xl transition group-hover:opacity-100 dark:opacity-0"
                style="background: radial-gradient(120px 60px at 70% 30%, rgba(99,102,241,0.25), transparent 60%);"></span>
            </button>
          </div>
        </div>
      </header>

      <main class="mx-auto max-w-5xl px-4 py-10">
        <!-- Hero -->
        <section class="relative overflow-hidden rounded-3xl border border-slate-200/60 bg-white/60 p-8 shadow-sm backdrop-blur-xl dark:border-slate-800/60 dark:bg-slate-950/40">
          <div class="absolute -right-16 -top-16 h-64 w-64 rounded-full bg-indigo-500/10 blur-3xl dark:bg-indigo-500/10"></div>
          <div class="absolute -bottom-24 -left-16 h-72 w-72 rounded-full bg-rose-500/10 blur-3xl dark:bg-rose-500/10"></div>

          <div class="relative">
            <div class="flex flex-wrap items-center gap-2">
              <span class="inline-flex items-center gap-2 rounded-full border border-slate-200/60 bg-white/60 px-3 py-1 text-xs font-semibold text-slate-700 shadow-sm dark:border-slate-800/60 dark:bg-slate-900/50 dark:text-slate-200">
                <span class="h-1.5 w-1.5 rounded-full bg-emerald-500"></span>
                Ready to ship (static)
              </span>
              <span class="inline-flex items-center gap-2 rounded-full border border-slate-200/60 bg-white/60 px-3 py-1 text-xs font-semibold text-slate-700 shadow-sm dark:border-slate-800/60 dark:bg-slate-900/50 dark:text-slate-200">
                <span class="h-1.5 w-1.5 rounded-full bg-indigo-500"></span>
                Tailwind CLI build
              </span>
            </div>

            <h1 class="mt-5 text-3xl font-semibold tracking-tight text-slate-900 dark:text-slate-100 sm:text-4xl">
              Fancy static UI with a clean Light/Dark switch
            </h1>
            <p class="mt-3 max-w-2xl text-sm leading-6 text-slate-600 dark:text-slate-400">
              Tailwind generates only the utilities it finds in your HTML/JS. The theme toggle simply adds/removes the
              <code class="rounded bg-slate-200/70 px-1 py-0.5 text-xs text-slate-800 dark:bg-slate-800/70 dark:text-slate-100">dark</code>
              class on <code class="rounded bg-slate-200/70 px-1 py-0.5 text-xs text-slate-800 dark:bg-slate-800/70 dark:text-slate-100">&lt;html&gt;</code>.
            </p>

            <div class="mt-6 flex flex-wrap gap-3">
              <a
                href="#"
                class="inline-flex items-center justify-center rounded-2xl bg-slate-900 px-4 py-2 text-sm font-semibold text-white shadow-sm transition hover:translate-y-[-1px] hover:shadow dark:bg-white dark:text-slate-900"
              >
                Primary action
              </a>
              <a
                href="#"
                class="inline-flex items-center justify-center rounded-2xl border border-slate-200/70 bg-white/60 px-4 py-2 text-sm font-semibold text-slate-700 shadow-sm backdrop-blur transition hover:bg-white/80 dark:border-slate-800/70 dark:bg-slate-900/50 dark:text-slate-200 dark:hover:bg-slate-900/70"
              >
                Secondary
              </a>

              <div class="flex items-center gap-2 text-xs text-slate-600 dark:text-slate-400">
                <span class="inline-block h-2 w-2 rounded-full bg-emerald-500/80"></span>
                Preference saved in localStorage
              </div>
            </div>
          </div>
        </section>

        <!-- Feature grid -->
        <section class="mt-8 grid gap-4 md:grid-cols-3">
          <!-- Card -->
          <article class="group relative overflow-hidden rounded-3xl border border-slate-200/60 bg-white/60 p-6 shadow-sm backdrop-blur transition hover:shadow-md dark:border-slate-800/60 dark:bg-slate-950/40">
            <div class="absolute -right-16 -top-16 h-56 w-56 rounded-full bg-indigo-500/10 blur-3xl"></div>
            <h2 class="relative text-sm font-semibold text-slate-900 dark:text-slate-100">Utility-first</h2>
            <p class="relative mt-2 text-sm text-slate-600 dark:text-slate-400">
              Compose styles with small, predictable classes. Your build outputs only what you use.
            </p>
            <div class="relative mt-4 inline-flex items-center gap-2 text-xs font-semibold text-indigo-700 dark:text-indigo-300">
              Learn more
              <span class="transition-transform group-hover:translate-x-0.5">→</span>
            </div>
          </article>

          <article class="group relative overflow-hidden rounded-3xl border border-slate-200/60 bg-white/60 p-6 shadow-sm backdrop-blur transition hover:shadow-md dark:border-slate-800/60 dark:bg-slate-950/40">
            <div class="absolute -left-16 -top-16 h-56 w-56 rounded-full bg-rose-500/10 blur-3xl"></div>
            <h2 class="relative text-sm font-semibold text-slate-900 dark:text-slate-100">Dark variants</h2>
            <p class="relative mt-2 text-sm text-slate-600 dark:text-slate-400">
              Use <code class="rounded bg-slate-200/70 px-1 py-0.5 text-xs dark:bg-slate-800/70">dark:</code> to define
              alternate styles. Switching is instant.
            </p>
            <div class="relative mt-4 flex flex-wrap gap-2">
              <span class="rounded-full bg-slate-100 px-3 py-1 text-xs font-semibold text-slate-700 ring-1 ring-slate-200 dark:bg-slate-800 dark:text-slate-200 dark:ring-slate-700">
                dark:bg-*
              </span>
              <span class="rounded-full bg-slate-100 px-3 py-1 text-xs font-semibold text-slate-700 ring-1 ring-slate-200 dark:bg-slate-800 dark:text-slate-200 dark:ring-slate-700">
                dark:text-*
              </span>
            </div>
          </article>

          <article class="group relative overflow-hidden rounded-3xl border border-slate-200/60 bg-white/60 p-6 shadow-sm backdrop-blur transition hover:shadow-md dark:border-slate-800/60 dark:bg-slate-950/40">
            <div class="absolute -right-20 bottom-0 h-60 w-60 rounded-full bg-emerald-500/10 blur-3xl"></div>
            <h2 class="relative text-sm font-semibold text-slate-900 dark:text-slate-100">Static-friendly</h2>
            <p class="relative mt-2 text-sm text-slate-600 dark:text-slate-400">
              No framework required. Point Tailwind at your HTML/JS files and build.
            </p>
            <div class="relative mt-4 rounded-2xl border border-slate-200/60 bg-white/60 p-3 text-xs text-slate-700 dark:border-slate-800/60 dark:bg-slate-900/50 dark:text-slate-200">
              <div class="font-semibold">Command</div>
              <div class="mt-1 font-mono">
                tailwindcss -i ./src/input.css -o ./dist/output.css --watch
              </div>
            </div>
          </article>
        </section>

        <!-- Footer -->
        <footer class="mt-10 flex flex-wrap items-center justify-between gap-3 text-xs text-slate-600 dark:text-slate-400">
          <span>© 2026 vast-cow — Static demo</span>
          <span class="rounded-full border border-slate-200/60 bg-white/50 px-3 py-1 dark:border-slate-800/60 dark:bg-slate-950/40">
            Tip: disable localStorage to follow OS preference
          </span>
        </footer>
      </main>
    </div>

    <script src="./src/app.js"></script>
  </body>
</html>
```



## 6) `src/app.js`（Light/Dark トグル + 永続化）

```js
(function () {
  const STORAGE_KEY = "theme"; // "light" | "dark"
  const root = document.documentElement;

  const toggleBtn = document.getElementById("themeToggle");
  const themeIcon = document.getElementById("themeIcon");
  const themeModeLabel = document.getElementById("themeModeLabel");
  const themeHint = document.getElementById("themeHint");

  function systemPref() {
    return window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light";
  }

  function savedTheme() {
    const v = localStorage.getItem(STORAGE_KEY);
    return v === "dark" || v === "light" ? v : null;
  }

  function applyTheme(theme, source) {
    const isDark = theme === "dark";
    root.classList.toggle("dark", isDark);

    // ボタンは「次に切り替わるモード」を表示
    themeModeLabel.textContent = isDark ? "Light" : "Dark";
    themeIcon.textContent = isDark ? "☀️" : "🌙";

    if (themeHint) {
      themeHint.textContent =
        source === "saved" ? "Theme follows your choice" : "Theme follows system preference";
    }
  }

  // 初期化：保存があればそれを優先、なければOS設定
  const initial = savedTheme() ?? systemPref();
  applyTheme(initial, savedTheme() ? "saved" : "system");

  toggleBtn.addEventListener("click", () => {
    const next = root.classList.contains("dark") ? "light" : "dark";
    localStorage.setItem(STORAGE_KEY, next);
    applyTheme(next, "saved");
  });

  // 未選択（localStorageなし）なら OS 変更に追従
  const mql = window.matchMedia("(prefers-color-scheme: dark)");
  mql.addEventListener?.("change", () => {
    if (savedTheme() === null) applyTheme(systemPref(), "system");
  });
})();
```



## 7) ビルドコマンドまとめ

```bash
# 1回だけ
npx tailwindcss -i ./src/input.css -o ./dist/output.css --minify

# 開発時（監視）
npx tailwindcss -i ./src/input.css -o ./dist/output.css --watch
```



## 8) つまずきポイント（今回ハマりやすい所）

### (A) デザインが反映されない

* `output.css` が読み込めているか（DevTools → Network → 200）
* `content` が HTML/JS を拾っているか（config の glob）
* `file://` 直開きより、簡易サーバ推奨：

```bash
npx serve .
# or
python -m http.server 8000
```

### (B) Dark が切り替わらない（Tailwind v4 の定番）

生成 CSS に次が出ている場合：

```css
.dark\:... { @media (prefers-color-scheme: dark) { ... } }
```

これは OS 連動になっていて `.dark` が効きません。
対策は `src/input.css` のこれ：

```css
@custom-variant dark (&:where(.dark, .dark *));
```



## まとめ

* Tailwind は **スキャン → 必要分だけ生成**なので、静的 HTML + JS でも強い
* Tailwind v4.1.18 では **`@import "tailwindcss";`** が基本
* 手動トグルで `.dark` を使うなら、v4 では **`@custom-variant` で dark を class ベースに**するのが重要
* “反映されない”の原因は大抵 **CSSパス / content指定 / dark の挙動**のどれか
