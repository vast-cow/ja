---
pubDatetime: 2026-04-20T18:20:54+09:00
title: "Windows Terminal用の白背景カラースキームの使い方"
description: "カラースキームの目的、設定内容のポイント、導入方法を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## 概要

Windows Terminalでは、カラースキームをカスタマイズすることで、表示の見やすさや作業効率を向上させることができます。本記事では、白背景をベースとしたシンプルなカラースキームの目的と、その導入方法について説明します。

このカラースキームは、明るい背景と落ち着いた文字色の組み合わせにより、長時間の作業でも目が疲れにくい点が特徴です。

---

## カラースキームの目的

白背景のカラースキームには、以下のような利点があります。

* **視認性の向上**：黒系の文字がはっきり表示される
* **ドキュメントとの統一感**：エディタやブラウザと似た配色で違和感が少ない
* **長時間作業への適性**：暗すぎないため、目への負担が軽減される

特に、日中の明るい環境で作業する場合に適しています。

---

## 設定内容のポイント

提供された設定では、以下のような構成になっています。

* **背景色（background）**：`#FFFFFF`（白）
* **前景色（foreground）**：`#0C0C0C`（濃い黒）
* **カーソル色（cursorColor）**：黒系で視認性を確保
* **選択範囲（selectionBackground）**：暗色で強調

また、基本色（赤・青・緑など）とその明るいバリエーション（bright系）も定義されており、コマンド出力の色分けが適切に行われます。

---

## 導入方法

### 1. 設定ファイルを開く

Windows Terminalを起動し、設定（Settings）を開きます。JSON形式の設定ファイルを編集します。

### 2. カラースキームを追加

以下のように、`"schemes"` 配列に指定された内容を追加します。

```json
"schemes": [
    {
        "name": "Theme Name",
        "background": "#FFFFFF",
        "foreground": "#0C0C0C",
        ...
    }
]
```

※既存の `"schemes"` がある場合は、その中に追加してください。

---

### 3. プロファイルに適用

次に、使用したいプロファイル（例：PowerShellやCommand Prompt）に対して、以下を設定します。

```json
"colorScheme": "Theme Name"
```

これにより、指定したカラースキームが適用されます。

---

## 使用時の注意点

* 明るい背景のため、暗い環境ではまぶしく感じる場合があります
* 一部のツールは色前提で設計されているため、表示が見づらくなる可能性があります

必要に応じて色を微調整することで、より快適な環境を構築できます。

---

## まとめ

この白背景カラースキームは、シンプルで視認性が高く、日常的なターミナル作業に適しています。設定はJSONファイルに追加するだけで簡単に適用できるため、自分の作業スタイルに合わせて導入してみてください。

```json
{
    "$help": "https://aka.ms/terminal-documentation",
    "$schema": "https://aka.ms/terminal-profiles-schema",
    "schemes": 
    [
        {
            "background": "#FFFFFF",
            "black": "#0C0C0C",
            "blue": "#0037DA",
            "brightBlack": "#767676",
            "brightBlue": "#3B78FF",
            "brightCyan": "#61D6D6",
            "brightGreen": "#16C60C",
            "brightPurple": "#B4009E",
            "brightRed": "#E74856",
            "brightWhite": "#F2F2F2",
            "brightYellow": "#C19C00",
            "cursorColor": "#0C0C0C",
            "cyan": "#3A96DD",
            "foreground": "#0C0C0C",
            "green": "#13A10E",
            "name": "My Light Theme",
            "purple": "#881798",
            "red": "#C50F1F",
            "selectionBackground": "#0C0C0C",
            "white": "#CCCCCC",
            "yellow": "#856B00"
        }
    ]
}
```
