---
title: "Gitで `.gitattributes` をコミットせずに改行コードを統一する方法"
description: "特定のリポジトリだけ設定する、すべてのリポジトリで共通設定する、.git/info/attributes とグローバル設定の使い分けを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-08-17T06:02:23.416Z
---

Gitで開発していると、WindowsとLinux/macOSの間で **CRLFとLFの違い**が問題になることがあります。

一般的な解決方法は、リポジトリに `.gitattributes` を配置して改行コードのルールを定義することです。

しかし、

* リポジトリには `.gitattributes` を追加したくない
* 自分の環境だけ改行コードを統一したい
* 複数のリポジトリで同じルールを適用したい

といったケースもあります。

Gitでは、`.gitattributes` をリポジトリにコミットしなくても、同等の設定を行うことができます。

## 特定のリポジトリだけ設定する

特定のリポジトリにだけ適用したい場合は、次のファイルを使用できます。

```text
.git/info/attributes
```

このファイルは `.git` ディレクトリ内にあるため、通常はGitの管理対象になりません。

例えば、次のように設定します。

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf
```

この設定では、基本的にテキストファイルをLFとして扱い、WindowsのバッチファイルだけCRLFにします。

それぞれの意味は次のとおりです。

```gitattributes
* text=auto eol=lf
```

テキストとして認識されたファイルについて、ワーキングツリー上の改行コードをLFにします。

一方、

```gitattributes
*.bat text eol=crlf
*.cmd text eol=crlf
```

では、`.bat` と `.cmd` をテキストファイルとして扱い、ワーキングツリー上ではCRLFにします。

つまり、基本方針としては、

```text
通常のテキストファイル → LF
Windowsバッチファイル → CRLF
```

という構成になります。

## すべてのリポジトリで共通設定する

毎回 `.git/info/attributes` を作るのが面倒な場合は、Gitのグローバル設定を利用できます。

まず、attributesファイルの場所を設定します。

```bash
git config --global core.attributesFile ~/.gitattributes
```

続いて `~/.gitattributes` を作成します。

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf
```

これで、自分の環境で扱うGitリポジトリに共通のattributes設定を適用できます。

## `.git/info/attributes` とグローバル設定の使い分け

用途によって使い分けると分かりやすいです。

| 方法                     | 適用範囲       | Git管理         |
| ---------------------- | ---------- | ------------- |
| `.gitattributes`       | リポジトリ      | される           |
| `.git/info/attributes` | 特定のリポジトリ   | されない          |
| `core.attributesFile`  | 自分のGit環境全体 | 各リポジトリからはされない |

チーム全体で同じルールを適用する必要があるなら、リポジトリに `.gitattributes` を置く方法が適しています。

一方、「自分の開発環境だけLFを基本にしたい」といった個人的な設定なら、

```bash
git config --global core.attributesFile ~/.gitattributes
```

を設定しておくと便利です。

特定のリポジトリだけ例外的なルールが必要なら、

```text
.git/info/attributes
```

を利用できます。

## まとめ

リポジトリに `.gitattributes` を追加できない場合でも、Git attributesの仕組み自体は利用できます。

全リポジトリ共通なら、

```bash
git config --global core.attributesFile ~/.gitattributes
```

として、

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf
```

を設定します。

特定のリポジトリだけなら、同じ内容を、

```text
.git/info/attributes
```

に記述します。

これにより、**基本はLF、WindowsのバッチファイルだけCRLF**というルールを、リポジトリに設定ファイルをコミットすることなく適用できます。
