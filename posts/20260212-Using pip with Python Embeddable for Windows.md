---
pubDatetime: 2026-02-12T20:50:30+09:00
title: "Windows版 Python embeddable で pip を使う方法"
description: "対象環境、① embeddable を展開、② get-pip.py を取得を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

Windows の **Python embeddable 配布版**（`python-3.x.x-embed-amd64.zip`）は軽量な実行環境ですが、**pip や venv は標準では利用できません**。
本記事では `get-pip.py` を使って pip を導入する方法と、**venv の代替として virtualenv を利用する方法**まで整理します。



## 対象環境

* Windows
* `python-3.x.x-embed-amd64.zip`
* 例：Python 3.11.x



# 1. pip を導入する（方式A）

## ① embeddable を展開

任意のフォルダに展開します。

例：

```
C:\python311-embed\
```

`python.exe` が存在することを確認してください。



## ② get-pip.py を取得

展開フォルダで PowerShell を開きます。

```powershell
curl -o get-pip.py https://bootstrap.pypa.io/get-pip.py
```



## ③ pip をインストール

```powershell
.\python.exe .\get-pip.py
```

確認：

```powershell
.\python.exe -m pip --version
```



# 2. `_pth` ファイルを修正（重要）

embeddable 版はモジュール検索パスが固定されています。
そのため `site-packages` を有効化する必要があります。

## 編集対象

例：

```
python311._pth
```

## 修正内容

* `import site` のコメントを解除
* `Lib\site-packages` を追加

### 修正例

```
python311.zip
.
Lib\site-packages
import site
```

この設定を行わないと、`pip install` 後に `import` できません。



# 3. 動作確認

```powershell
.\python.exe -m pip install requests
```

```powershell
.\python.exe
>>> import requests
```

エラーが出なければ成功です。



# 4. venv は使えない？ → virtualenv が代替になる

embeddable 版では **標準の venv モジュールは基本的に利用できません**。

理由：

* embeddable 版には `venv` 関連モジュールが含まれていない
* ensurepip も含まれていない

## 代替手段：virtualenv を使用

pip 導入後、`virtualenv` をインストールします。

```powershell
.\python.exe -m pip install virtualenv
```

### 仮想環境を作成

```powershell
.\python.exe -m virtualenv venv
```

### 仮想環境を有効化

```powershell
.\venv\Scripts\activate
```

これで通常の Python と同様に仮想環境を利用できます。



## venv と virtualenv の違い（簡易比較）

| 項目            | venv        | virtualenv |
| - | -- | - |
| 標準同梱          | 通常版Pythonのみ | 外部パッケージ    |
| embeddableで利用 | 原則不可        | 利用可能       |
| pip必要性        | 不要（通常版）     | 必要         |

embeddable 環境では **virtualenv が事実上の標準選択肢**になります。



# よくある問題

### pip install は成功するが import できない

→ `_pth` に `Lib\site-packages` がない
→ `import site` が無効

### SSL 証明書エラー

→ embeddable 環境では証明書設定が不足する場合あり



# まとめ

Windows の Python embeddable で pip を使うには：

1. `get-pip.py` で pip を導入
2. `_pth` を修正して `site-packages` を有効化
3. 仮想環境が必要なら `virtualenv` を使用

embeddable 版は配布用途向けの軽量ランタイムですが、適切に設定すれば通常の開発用途にも利用可能です。
