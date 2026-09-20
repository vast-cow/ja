---
pubDatetime: 2026-07-15T21:17:03+09:00
title: "Google翻訳をダークテーマに自動切り替えする方法"
description: "主な目的、必要なもの、導入方法を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Google翻訳を長時間使うと、明るい画面がまぶしく感じることがあります。そこで便利なのが、パソコンやブラウザの表示設定に合わせて、Google翻訳を自動的にダークテーマへ切り替えるユーザースクリプトです。

このスクリプトを導入すると、端末がダークモードのときだけGoogle翻訳の背景や入力欄、翻訳結果、メニューなどが暗い配色になります。ライトモードに戻すと、Google翻訳も通常の表示に戻ります。

## 主な目的

このスクリプトの目的は、Google翻訳を見やすい暗色表示に変更することです。

特に、夜間や暗い部屋でGoogle翻訳を使う場合に、画面のまぶしさを抑えられます。また、OSやブラウザのテーマ設定と連動するため、毎回手動で表示を切り替える必要がありません。

## 必要なもの

使用するには、ユーザースクリプトを実行できるブラウザ拡張機能が必要です。

代表的なものには、TampermonkeyやViolentmonkeyがあります。これらをChrome、Edge、Firefoxなどに追加して使用します。

## 導入方法

### 1. ユーザースクリプト拡張機能を追加する

ブラウザにTampermonkeyまたはViolentmonkeyをインストールします。

### 2. 新しいスクリプトを作成する

拡張機能の管理画面を開き、「新しいスクリプトを作成」などの項目を選びます。

### 3. スクリプトを貼り付ける

初期状態で入力されている内容を削除し、Google翻訳用の自動ダークテーマスクリプトを貼り付けます。

### 4. 保存する

スクリプトを保存し、有効な状態にします。

### 5. Google翻訳を開く

Google翻訳のページを開きます。すでに開いている場合は、ページを再読み込みしてください。

## 使い方

基本的に、導入後の操作は必要ありません。

パソコンやブラウザがダークモードの場合、Google翻訳も自動的に暗い表示になります。ライトモードに切り替えると、Google翻訳は通常の明るい表示に戻ります。

このスクリプトは、次のような部分を暗色表示に調整します。

* ページ全体の背景
* 翻訳文の入力欄
* 翻訳結果の表示領域
* 言語選択メニュー
* ボタンやタブ
* ポップアップメニュー
* スクロールバー

Google翻訳の画面内に新しく追加された要素も自動的に確認されるため、ページを操作した後もダークテーマが維持されます。

## 注意点

Google翻訳の画面構成が変更されると、一部の場所が正しく暗くならないことがあります。その場合は、スクリプトの更新が必要です。

また、ほかのGoogle翻訳用ユーザースクリプトや外観変更系の拡張機能を同時に使用すると、表示が崩れる可能性があります。

問題が発生した場合は、ほかのテーマ変更機能を一時的に無効にして確認してください。

## まとめ

Google翻訳の自動ダークテーマスクリプトを使うと、OSやブラウザの表示設定に合わせて、Google翻訳の外観を自動的に切り替えられます。

一度導入すれば毎回設定する必要がなく、夜間でも見やすい環境で翻訳作業を行えます。

```javascript
// ==UserScript==
// @name         Google Translate Auto Dark Theme
// @namespace    https://translate.google.com/
// @version      3.0.0
// @description  Follows prefers-color-scheme. Uses the default theme in light mode and applies a custom theme in dark mode.
// @match        https://translate.google.com/*
// @match        https://translate.google.co.jp/*
// @grant        GM_addStyle
// @run-at       document-start
// ==/UserScript==

(() => {
    'use strict';

    const STYLE_ID = 'tm-google-translate-auto-theme-v3';

    const css = String.raw`
        /*
         * Apply the custom theme only when the OS or browser
         * is in dark mode.
         *
         * In light mode, no styles are overridden, so
         * Google Translate's default theme remains unchanged.
         */
        @media (prefers-color-scheme: dark) {
            :root {
                color-scheme: dark !important;

                --tm-page: #0f1115;
                --tm-surface: #171a20;
                --tm-surface-2: #1d222b;
                --tm-surface-blue: #1d2735;
                --tm-hover: #282e38;
                --tm-border: #3b424d;
                --tm-text: #e8eaed;
                --tm-text-2: #bdc1c6;
                --tm-muted: #9aa0a6;
                --tm-accent: #8ab4f8;
                --tm-selection: #3d5f91;
                --tm-scrollbar: #565e69;
                --tm-scrollbar-hover: #707985;

                /*
                 * Google Material Design color variables.
                 */
                --gm3-sys-color-surface: var(--tm-surface) !important;
                --gm3-sys-color-surface-container: var(--tm-surface-2) !important;
                --gm3-sys-color-surface-container-low: var(--tm-surface) !important;
                --gm3-sys-color-surface-container-lowest: var(--tm-page) !important;
                --gm3-sys-color-surface-container-high: var(--tm-surface-2) !important;
                --gm3-sys-color-surface-container-highest: var(--tm-hover) !important;
                --gm3-sys-color-surface-variant: var(--tm-border) !important;
                --gm3-sys-color-on-surface: var(--tm-text) !important;
                --gm3-sys-color-on-surface-variant: var(--tm-text-2) !important;
                --gm3-sys-color-outline: var(--tm-border) !important;
                --gm3-sys-color-outline-variant: var(--tm-border) !important;
            }

            html,
            body,
            #yDmH0d {
                background-color: var(--tm-page) !important;
                color: var(--tm-text) !important;
            }

            /*
             * Primary containers that often remain white.
             */
            .ccvoYb,
            .gb_fd,
            .gb_cd,
            .gb_jd {
                background-color: var(--tm-surface) !important;
                border-color: var(--tm-border) !important;
            }

            /*
             * Pale blue translation result panel.
             */
            .QcsUad,
            .QcsUad.sMVRZe,
            .QcsUad.hCXDsb,
            .QcsUad.wneUed {
                background-color: var(--tm-surface-blue) !important;
                color: var(--tm-text) !important;
            }

            /*
             * Rounded buttons such as History and Saved.
             */
            .ySES5 {
                background-color: var(--tm-surface-2) !important;
                color: var(--tm-accent) !important;
                border-color: var(--tm-border) !important;
            }

            /*
             * White gradient at the right end of the language tabs.
             */
            .X4DQ0::after {
                background: linear-gradient(
                    to right,
                    rgba(23, 26, 32, 0),
                    var(--tm-surface)
                ) !important;
            }

            /*
             * Similar pseudo-elements used in horizontal scroll areas.
             */
            [class*="X4DQ0"]::after {
                background-image: linear-gradient(
                    to right,
                    rgba(23, 26, 32, 0),
                    var(--tm-surface)
                ) !important;
            }

            header,
            nav,
            footer,
            main,
            [role="banner"],
            [role="navigation"] {
                border-color: var(--tm-border) !important;
            }

            textarea,
            input,
            select,
            [role="textbox"],
            [contenteditable="true"] {
                background-color: transparent !important;
                color: var(--tm-text) !important;
                caret-color: var(--tm-accent) !important;
                border-color: var(--tm-border) !important;
            }

            textarea::placeholder,
            input::placeholder {
                color: var(--tm-muted) !important;
                opacity: 1 !important;
            }

            button,
            [role="button"],
            [role="tab"],
            [role="option"] {
                border-color: var(--tm-border) !important;
            }

            button:hover,
            [role="button"]:hover,
            [role="tab"]:hover,
            [role="option"]:hover {
                background-color: var(--tm-hover) !important;
            }

            [role="tab"][aria-selected="true"],
            [aria-checked="true"],
            [aria-current="true"] {
                color: var(--tm-accent) !important;
            }

            [role="dialog"],
            [role="menu"],
            [role="listbox"],
            [role="tooltip"],
            [aria-modal="true"],
            .goog-menu,
            .goog-menuitem,
            .goog-tooltip,
            .ita-kd-menu,
            .ita-kd-dropdown-menu,
            .ita-kd-statusbar {
                background-color: var(--tm-surface-2) !important;
                color: var(--tm-text) !important;
                border-color: var(--tm-border) !important;
                box-shadow: 0 6px 18px rgba(0, 0, 0, 0.55) !important;
            }

            [role="option"][aria-selected="true"],
            .goog-menuitem-highlight,
            .goog-menuitem:hover,
            .ita-kd-menuitem:hover {
                background-color: var(--tm-hover) !important;
            }

            hr,
            [role="separator"] {
                background-color: var(--tm-border) !important;
                border-color: var(--tm-border) !important;
            }

            a,
            a:visited {
                color: var(--tm-accent) !important;
            }

            /*
             * Bright backgrounds detected by JavaScript.
             */
            .tm-gt-auto-bg {
                background-color: var(--tm-surface) !important;
                border-color: var(--tm-border) !important;
            }

            .tm-gt-auto-bg-blue {
                background-color: var(--tm-surface-blue) !important;
                border-color: var(--tm-border) !important;
            }

            /*
             * Bright gradients detected by JavaScript.
             */
            .tm-gt-auto-gradient {
                background-image: linear-gradient(
                    to right,
                    rgba(23, 26, 32, 0),
                    var(--tm-surface)
                ) !important;
            }

            /*
             * Dark text detected by JavaScript.
             */
            .tm-gt-auto-text {
                color: var(--tm-text) !important;
            }

            button svg,
            [role="button"] svg,
            [role="tab"] svg,
            [role="option"] svg,
            .TYVfy svg {
                color: inherit !important;
            }

            button svg path:not([fill="none"]),
            [role="button"] svg path:not([fill="none"]),
            [role="tab"] svg path:not([fill="none"]),
            [role="option"] svg path:not([fill="none"]),
            .TYVfy svg path:not([fill="none"]) {
                fill: currentColor !important;
            }

            ::selection {
                background-color: var(--tm-selection) !important;
                color: #ffffff !important;
            }

            * {
                scrollbar-color:
                    var(--tm-scrollbar)
                    var(--tm-page) !important;
            }

            *::-webkit-scrollbar {
                width: 12px;
                height: 12px;
            }

            *::-webkit-scrollbar-track {
                background: var(--tm-page) !important;
            }

            *::-webkit-scrollbar-thumb {
                background: var(--tm-scrollbar) !important;
                border: 3px solid var(--tm-page) !important;
                border-radius: 10px;
            }

            *::-webkit-scrollbar-thumb:hover {
                background: var(--tm-scrollbar-hover) !important;
            }
        }
    `;

    function installStyle() {
        if (document.getElementById(STYLE_ID)) {
            return;
        }

        if (typeof GM_addStyle === 'function') {
            const style = GM_addStyle(css);

            if (style) {
                style.id = STYLE_ID;
            }

            return;
        }

        const style = document.createElement('style');

        style.id = STYLE_ID;
        style.textContent = css;

        (document.head || document.documentElement).appendChild(style);
    }

    function parseRgb(value) {
        const match = value.match(
            /^rgba?\(\s*([\d.]+)[,\s]+([\d.]+)[,\s]+([\d.]+)(?:\s*[,/]\s*([\d.]+))?\s*\)$/i
        );

        if (!match) {
            return null;
        }

        return {
            r: Number(match[1]),
            g: Number(match[2]),
            b: Number(match[3]),
            a: match[4] === undefined
                ? 1
                : Number(match[4])
        };
    }

    function luminance({ r, g, b }) {
        return (
            0.2126 * r +
            0.7152 * g +
            0.0722 * b
        ) / 255;
    }

    function chroma({ r, g, b }) {
        return (
            Math.max(r, g, b) -
            Math.min(r, g, b)
        ) / 255;
    }

    function isBlueTint({ r, g, b }) {
        return b >= r + 10 && b >= g + 4;
    }

    function shouldSkip(element) {
        return element.matches([
            'img',
            'picture',
            'video',
            'canvas',
            'iframe',
            'object',
            'embed',
            'svg defs',
            'svg linearGradient',
            'svg radialGradient',
            'svg stop'
        ].join(','));
    }

    function hasLightGradient(backgroundImage) {
        if (
            !backgroundImage ||
            backgroundImage === 'none' ||
            !backgroundImage.includes('gradient')
        ) {
            return false;
        }

        return (
            /rgba?\(\s*25[0-5]\s*[,\s]+\s*25[0-5]\s*[,\s]+\s*25[0-5]/i.test(
                backgroundImage
            ) ||
            /rgba?\(\s*24\d\s*[,\s]+\s*24\d\s*[,\s]+\s*24\d/i.test(
                backgroundImage
            ) ||
            /#fff(?:fff)?\b/i.test(backgroundImage) ||
            /\bwhite\b/i.test(backgroundImage)
        );
    }

    function darkenElement(element) {
        if (
            !(element instanceof Element) ||
            shouldSkip(element)
        ) {
            return;
        }

        const style = getComputedStyle(element);

        /*
         * Detect bright backgrounds and assign classes.
         *
         * These classes may also be assigned in light mode,
         * but the corresponding CSS exists only inside
         * prefers-color-scheme: dark, so they have no effect
         * while light mode is active.
         */
        if (
            !element.classList.contains('tm-gt-auto-bg') &&
            !element.classList.contains('tm-gt-auto-bg-blue') &&
            style.backgroundImage === 'none'
        ) {
            const background = parseRgb(style.backgroundColor);

            if (
                background &&
                background.a >= 0.2 &&
                luminance(background) >= 0.68 &&
                chroma(background) <= 0.30
            ) {
                element.classList.add(
                    isBlueTint(background)
                        ? 'tm-gt-auto-bg-blue'
                        : 'tm-gt-auto-bg'
                );
            }
        }

        /*
         * Detect bright gradients.
         */
        if (
            !element.classList.contains('tm-gt-auto-gradient') &&
            hasLightGradient(style.backgroundImage)
        ) {
            element.classList.add('tm-gt-auto-gradient');
        }

        /*
         * Detect dark neutral-colored text.
         */
        if (!element.classList.contains('tm-gt-auto-text')) {
            const foreground = parseRgb(style.color);

            if (
                foreground &&
                foreground.a >= 0.5 &&
                luminance(foreground) <= 0.48 &&
                chroma(foreground) <= 0.18
            ) {
                element.classList.add('tm-gt-auto-text');
            }
        }
    }

    let scanScheduled = false;

    function scan(root = document) {
        installStyle();

        if (root instanceof Element) {
            darkenElement(root);
        }

        const elements = root.querySelectorAll
            ? root.querySelectorAll('*')
            : [];

        for (const element of elements) {
            darkenElement(element);
        }
    }

    function scheduleScan(root = document) {
        if (scanScheduled) {
            return;
        }

        scanScheduled = true;

        requestAnimationFrame(() => {
            scanScheduled = false;
            scan(root);
        });
    }

    installStyle();

    const start = () => {
        scan(document);

        const observer = new MutationObserver(mutations => {
            for (const mutation of mutations) {
                if (mutation.type === 'childList') {
                    for (const node of mutation.addedNodes) {
                        if (node instanceof Element) {
                            scheduleScan(node);
                        }
                    }
                } else if (mutation.target instanceof Element) {
                    scheduleScan(mutation.target);
                }
            }
        });

        observer.observe(document.documentElement, {
            childList: true,
            subtree: true,
            attributes: true,
            attributeFilter: [
                'style',
                'class',
                'hidden',
                'aria-hidden',
                'aria-selected',
                'aria-checked',
                'aria-expanded'
            ]
        });

        /*
         * Periodically recheck elements that are reused
         * by visibility toggling rather than being recreated.
         */
        window.setInterval(() => {
            scheduleScan(document);
        }, 2500);
    };

    if (document.readyState === 'loading') {
        document.addEventListener(
            'DOMContentLoaded',
            start,
            { once: true }
        );
    } else {
        start();
    }
})();
```
