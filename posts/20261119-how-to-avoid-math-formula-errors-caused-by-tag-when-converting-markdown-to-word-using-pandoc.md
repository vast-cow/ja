---
title: "PandocでMarkdownをWordに変換するとき、`\\tag`による数式エラーを回避する方法"
description: "Pandoc で Markdown を Word に変換する際に、LaTeX の \\tag{} コマンドが原因で警告が出る問題を、sed を使って \\tag{} を除去してから変換することで解決する方法を紹介する。"
pubDatetime: 2026-11-19T18:16:00+09:00
---

Pandocを使うと、MarkdownファイルをWord（DOCX）形式に簡単に変換できます。しかし、LaTeXの数式を含むMarkdownでは、数式番号を指定する `\tag{}` が原因で変換時に警告が発生することがあります。

今回は、`sed` を使って `\tag{}` を無視し、PandocでMarkdownをWordに変換する方法を紹介します。

## 発生した問題

次のコマンドでMarkdownをWordに変換しようとしました。

```bash
pandoc -f markdown+tex_math_single_backslash doc.md -o doc.docx
```

ところが、次のような警告が表示されました。

```text
[WARNING] Could not convert TeX math ..., rendering as TeX:
  P_G\otimes P_L \tag{30}
                     ^
  unexpected control sequence \tag
```

原因は、Markdown内のLaTeX数式に含まれる `\tag{30}` です。

`\tag{}` は数式番号を明示的に指定するLaTeXのコマンドです。

例えば、次のような数式で使用します。

```latex
$$
E = mc^2 \tag{1}
$$
```

Pandocでは、このような数式をWordの数式形式（OMML）に変換する際に、`\tag` を処理できず警告が発生する場合があります。

## 解決方法：sedで`\tag`を除去する

数式番号が不要であれば、Pandocに渡す前に `\tag{}` を削除すれば解決できます。

そこで、次のコマンドを使用します。

```bash
sed -E 's/\\tag\{[^}]*\}//g' doc.md | \
pandoc -f markdown+tex_math_single_backslash \
  -o doc.docx
```

これだけで、Markdown内の `\tag{1}` や `\tag{30}` などを削除した状態で、Wordへの変換を実行できます。

元の `doc.md` は変更されません。

## コマンドの仕組み

### 1. sedで数式番号を削除する

```bash
sed -E 's/\\tag\{[^}]*\}//g' doc.md
```

このコマンドは、Markdownファイル中の `\tag{...}` に一致する文字列を削除します。

正規表現の意味は次のとおりです。

| 記述 | 意味 |
|---|---|
| `-E` | 拡張正規表現を使用 |
| `s/...//g` | 一致した文字列をすべて削除 |
| `\\tag` | 文字列 `\tag` に一致 |
| `\{` | `{` に一致 |
| `[^}]*` | `}` 以外の文字が0文字以上 |
| `\}` | `}` に一致 |

例えば、変換前の数式が次の場合、

```latex
$$
E = mc^2 \tag{1}
$$
```

`sed` によって次のようになります。

```latex
$$
E = mc^2
$$
```

数式本体は維持され、番号だけが削除されます。

### 2. パイプでPandocに渡す

```bash
| pandoc -f markdown+tex_math_single_backslash -o doc.docx
```

パイプ `|` を使用すると、`sed` が出力したテキストを、そのままPandocの標準入力に渡すことができます。

このため、一時的なMarkdownファイルを作成する必要がありません。

また、`-f markdown+tex_math_single_backslash` は、Markdown入力において `\(...\)` や `\[...\]` 形式のLaTeX数式も認識するための指定です。

Pandocは、修正後のMarkdownを読み込み、Wordファイル `doc.docx` を生成します。

## この方法のメリット

- **元のMarkdownを変更しない**：`doc.md` はそのまま残ります。
- **一括処理できる**：ファイル内の `\tag{}` をまとめて除去できます。
- **追加のスクリプトが不要**：`sed` とPandocだけで実行できます。
- **数式本体を維持できる**：数式番号以外のLaTeXコードは原則としてそのまま渡されます。

ただし、この方法は数式内かどうかを区別せず、ファイル中の `\tag{...}` に一致する文字列をすべて削除します。また、タグの内容に入れ子の波括弧がある場合には対応していません。

## まとめ

PandocでMarkdownをWordに変換する際に、LaTeXの `\tag{}` が原因で数式変換の警告が発生する場合があります。

数式番号が不要であれば、`sed` で除去してからPandocに渡すことで回避できます。

最終的なコマンドはこちらです。

```bash
sed -E 's/\\tag\{[^}]*\}//g' doc.md | \
pandoc -f markdown+tex_math_single_backslash \
  -o doc.docx
```

LaTeX数式を多く含む論文や技術文書をMarkdownからWordへ変換するときに便利な方法です。
