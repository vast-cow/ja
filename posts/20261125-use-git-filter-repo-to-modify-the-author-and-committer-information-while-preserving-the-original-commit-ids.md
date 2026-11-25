---
title: "git filter-repo で過去のコミットIDを維持しながら Author / Committer を修正する"
description: "git filter-repo の --refs オプションで書き換え範囲を限定し、--commit-callback で Author と Committer を修正する方法を解説する。"
pubDatetime: 2026-11-25T22:32:00+09:00
---

Git で `user.name` や `user.email` の設定を間違えたまま、複数のコミットを作成してしまうことがある。

過去のコミットに記録された Author や Committer の情報を一括で修正するには、`git filter-repo` が便利だ。

ただし、リポジトリの全履歴を書き換えるのではなく、修正が必要な範囲だけを書き換え、それ以前のコミットIDは維持したい。

今回は、`git log` で書き換え対象を特定し、`git filter-repo` の `--refs` オプションを使って履歴の書き換え範囲を限定する。

## やりたいこと

以下のようなコミット履歴があるとする。

```text
A --- B --- C --- D --- E --- F (main)
            *           *
```

`C` と `E` が、誤ったユーザー情報で作成されたコミットだとする。

書き換え後は、以下のような履歴にしたい。

```text
A --- B --- C' -- D' -- E' -- F' (main)
```

- `A`、`B` のコミットIDは維持する
- `C`、`E` の Author / Committer を修正する
- `D`、`F` の Author / Committer は変更しない

Git のコミットIDは親コミットのIDにも依存するため、`C` を書き換えると、その子孫である `D`、`E`、`F` のコミットIDも変化する。

一方、`C` より前の履歴を書き換える必要はない。

## 1. 書き換え対象のコミットを特定する

まずは `git log` を使い、誤ったユーザー情報が記録されているコミットを列挙する。

今回は、以下のユーザー情報を修正するものとする。

| 項目 | 修正前 | 修正後 |
|---|---|---|
| Name | Old Name | New Name |
| Email | old@example.com | new@example.com |

Author または Committer の名前とメールアドレスの組が一致するコミットを抽出する。

```bash
git log main --reverse --topo-order \
  --format='%H%x09%an%x09%ae%x09%cn%x09%ce%x09%s' |
  awk -F '\t' \
    -v name='Old Name' \
    -v email='old@example.com' \
    '($2 == name && $3 == email) || ($4 == name && $5 == email)'
```

`git log` のフォーマット指定には、以下のプレースホルダーを使用している。

| プレースホルダー | 内容 |
|---|---|
| `%H` | コミットID |
| `%an` | Author の名前 |
| `%ae` | Author のメールアドレス |
| `%cn` | Committer の名前 |
| `%ce` | Committer のメールアドレス |
| `%s` | コミットメッセージの件名 |
| `%x09` | タブ文字 |

`--reverse --topo-order` を指定することで、親コミットが子コミットより先に表示される。

直線的なコミット履歴であれば、出力の先頭が書き換え対象のうち最も古いコミットになる。

なお、`git log --author=... --committer=...` と指定すると AND 条件になるため、今回は `awk` を使って OR 条件で抽出している。

## 2. 書き換え範囲を決定する

最も古い書き換え対象のコミットIDが `abc1234` だったとする。

このコミット自身を含めて `main` の先端までを書き換えるには、以下の範囲を指定すればよい。

```bash
abc1234^..main
```

`A..B` は、`B` から到達可能なコミットのうち、`A` から到達可能なコミットを除いた範囲を表す。

したがって、`abc1234` 自身を含めるためには、その親コミットを表す `^` が必要になる。

実際に書き換える範囲は、事前に以下のコマンドで確認できる。

```bash
git log --oneline abc1234^..main
```

また、書き換え後の検証に備えて、境界となるコミットIDと現在のブランチ先端を記録しておく。

```bash
base=$(git rev-parse abc1234^)
old_tip=$(git rev-parse main)
```

## 3. git filter-repo で履歴を書き換える

履歴の書き換えは破壊的な操作なので、元のリポジトリは残し、新しく clone した作業用リポジトリで実行することを推奨する。

`--refs` で書き換え範囲を限定し、`--commit-callback` で Author と Committer を修正する。

```bash
git filter-repo \
  --refs "$base..main" \
  --commit-callback '
    old_name = b"Old Name"
    old_email = b"old@example.com"

    new_name = b"New Name"
    new_email = b"new@example.com"

    if (commit.author_name == old_name and
        commit.author_email == old_email):
        commit.author_name = new_name
        commit.author_email = new_email

    if (commit.committer_name == old_name and
        commit.committer_email == old_email):
        commit.committer_name = new_name
        commit.committer_email = new_email
  '
```

Author と Committer は独立して判定しているため、どちらか一方だけが誤っている場合も、該当する情報だけが修正される。

### --refs の役割

`--refs` を指定すると、履歴の部分的な書き換えを行う `--partial` が暗黙的に有効になる。

今回のように書き換え範囲を限定することで、範囲より前のコミットを再生成せず、そのコミットIDを維持できる。

```text
書き換え前:

A --- B --- C --- D --- E --- F
            |                 |
            +--- 書き換え範囲 -+

書き換え後:

A --- B --- C' -- D' -- E' -- F'
```

なお、実際に書き換える前に `--dry-run` オプションを追加して確認することもできる。

## 4. 書き換え結果を確認する

まず、Author と Committer の情報を確認する。

```bash
git log main \
  --format='%h | %an <%ae> | %cn <%ce> | %s'
```

手順1のコマンドを再実行し、誤ったユーザー情報を持つコミットが残っていないことも確認する。

次に、書き換え範囲より前のコミットが、新しい履歴でも祖先として維持されていることを確認する。

```bash
git merge-base --is-ancestor "$base" main \
  && echo "Original history preserved"
```

ソースコードなどのファイル内容に差分がないことも確認しておく。

```bash
git diff --exit-code "$old_tip" main
```

また、`git filter-repo` は書き換え前後のコミットIDの対応表を生成する。

```bash
cat .git/filter-repo/commit-map
```

このファイルを参照することで、どのコミットIDがどのコミットIDに変更されたかを確認できる。

## 5. user.name と user.email を修正する

今後、同じ問題が発生しないように、Git の設定も修正しておく。

```bash
git config --local user.name "New Name"
git config --local user.email "new@example.com"
```

設定値と設定元は、以下のコマンドで確認できる。

```bash
git config --show-origin --get user.name
git config --show-origin --get user.email
```

## 注意点

- マージを含む履歴では、単純に最初に表示されたコミットの親を境界にすると、修正対象の一部が範囲外になる場合がある。すべての修正対象が範囲に含まれることを確認する必要がある。
- 最初のコミットが修正対象の場合は親コミットが存在しないため、`--refs main` などで指定する。
- 書き換え対象外のブランチやタグには、修正前の履歴が残る可能性がある。
- 書き換え範囲内の署名付きコミットは、署名が失われる場合がある。
- すでにリモートへ push している場合、履歴の更新には force push が必要になる。共有ブランチでは他の開発者への影響にも注意する。

## まとめ

誤った `user.name` や `user.email` を修正する際、リポジトリの全履歴を書き換える必要はない。

今回の手順は以下のとおり。

1. `git log` で修正対象のコミットを列挙する。
2. 最も古い修正対象のコミットを特定する。
3. `--refs` で書き換え範囲を限定する。
4. `--commit-callback` で Author / Committer を修正する。
5. 書き換え前後のコミットIDとファイル内容を確認する。

ポイントは、**修正対象のコミットだけを選ぶことと、履歴を書き換える範囲を限定することは別の操作である**という点だ。

`--commit-callback` と `--refs` を組み合わせることで、必要なユーザー情報だけを修正しながら、影響を受ける必要のない古いコミットのIDを維持できる。
