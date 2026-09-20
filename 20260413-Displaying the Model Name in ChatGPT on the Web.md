---
pubDatetime: 2026-04-13T19:04:44+09:00
title: "Web版 ChatGPT にモデル名を表示する方法"
description: "この記事では、Chrome 拡張機能 User-Agent Switcher and Manager を使用し、chatgpt.com に対してカスタム設定を行うことで、ChatGPT のインターフェース上にモデル名をより明確に表示させるシンプルな方法を説明します。 概要 User-Agent Sw…"
---

この記事では、Chrome 拡張機能 **User-Agent Switcher and Manager** を使用し、`chatgpt.com` に対してカスタム設定を行うことで、ChatGPT のインターフェース上にモデル名をより明確に表示させるシンプルな方法を説明します。

## 概要

**User-Agent Switcher and Manager** を設定し、`chatgpt.com` に対してのみ iPhone の Chrome のユーザーエージェントを送信することで、ChatGPT サイトはモデル選択エリアにモデル名をより明確に表示する場合があります。

重要なポイントは、ブラウザ全体の識別情報を変更するのではなく、特定のドメインに対してのみカスタムユーザーエージェントを適用することです。

## 使用した設定

拡張機能には以下の JSON 設定を使用しました：

```json
{ 
  "blacklist": [], 
  "custom": { 
    "chatgpt.com": "Mozilla/5.0 (iPhone; CPU iPhone OS 13_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/146.0.0.0 Mobile/15E148 Safari/604.1" 
  }, 
  "last-update": 1776073925509, 
  "mode": "custom", 
  "parser": {}, 
  "popular-browsers": [ 
    "Internet Explorer", 
    "Safari", 
    "Chrome", 
    "Firefox", 
    "Opera", 
    "Edge", 
    "Vivaldi" 
  ], 
  "popular-oss": [ 
    "Windows", 
    "Mac OS", 
    "Linux", 
    "Chromium OS", 
    "Android" 
  ], 
  "protected": [ 
    "google.com/recaptcha", 
    "gstatic.com/recaptcha", 
    "accounts.google.com", 
    "accounts.youtube.com", 
    "gitlab.com/users/sign_in", 
    "challenges.cloudflare.com" 
  ], 
  "remote-address": "", 
  "user-styling": "", 
  "userAgentData": true, 
  "whitelist": [], 
  "json-guid": "8455108d-186e-47c2-89ec-41d7384aec1e", 
  "json-forced": false 
}
```

## この設定が行うこと

設定の中で重要なのは次の部分です：

```json
"custom": { 
  "chatgpt.com": "Mozilla/5.0 (iPhone; CPU iPhone OS 13_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/146.0.0.0 Mobile/15E148 Safari/604.1" 
}
```

これは、`chatgpt.com` にアクセスした際に、特定の **iPhone Chrome のユーザーエージェント** を使用するよう拡張機能に指示しています。この設定はドメイン単位で適用されるため、他のウェブサイトには影響しません。

拡張機能は以下の設定にもなっています：

```json
"mode": "custom"
```

つまり、ブラウザは `custom` セクションで定義されたルールを適用します。

## なぜモデル名が表示されるのか

この設定により、ChatGPT は検出したブラウザ情報に基づいて異なるインターフェースを提供する場合があります。その結果、モデル選択エリアでモデル名がより明確に表示されることがあります。

実際には、ユーザーエージェントを変更することで、サイトがブラウザをモバイルクライアントのように扱い、インターフェース内のモデル情報の表示方法が変わる可能性があります。

## 重要なポイント

### サイト限定の動作

この設定が適用されるのは以下のみです：

* `chatgpt.com`

すべてのサイトに iPhone のユーザーエージェントが適用されるわけではありません。

### 保護されたドメイン

`protected` リストには、reCAPTCHA、Google サインインページ、Cloudflare の認証ページなどが含まれています。これらはログインや認証の問題を避けるため、ユーザーエージェントの偽装対象から除外されています。

### User-Agent Client Hints

以下の設定：

```json
"userAgentData": true
```

は、ユーザーエージェントに関連するクライアントヒントの挙動も有効にしていることを示します。現代のウェブサイトでは、どのインターフェースを表示するかを判断する際に、従来のユーザーエージェント文字列以外の情報も利用されることがあるため、重要になる場合があります。

## 実用上の結果

**User-Agent Switcher and Manager** にこの設定を適用すると、ChatGPT がモデル選択部分で現在のモデルをより識別しやすい形で表示される場合があります。

これは、ブラウジング環境全体を変更することなく、使用中のモデルをより明確に確認したいユーザー向けの軽量な回避策です。

## 結論

`chatgpt.com` 向けにカスタム設定を行った **User-Agent Switcher and Manager** を使用することで、ChatGPT のインターフェース表示を変更する簡単な方法が得られます。上記の iPhone Chrome のユーザーエージェントを使うことで、モデル選択エリアにモデル名が明示的に表示される可能性があり、現在使用しているモデルの確認がしやすくなります。
