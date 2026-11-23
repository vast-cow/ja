---
title: "GitHub の Organization ごとに Git の user.name / user.email を自動で切り替える方法"
description: "Git の includeIf を使って、リモート URL に応じて user.name と user.email を自動で切り替える方法を解説。"
pubDatetime: 2026-11-23T10:48:00+09:00
---

複数の GitHub アカウントや Organization を使い分けていると、リポジトリごとに Git の `user.name` と `user.email` を設定するのが面倒になる。

例えば、次のようなケースだ。

- 個人リポジトリでは個人用メールアドレスを使用したい
- 会社の Organization では仕事用メールアドレスを使用したい
- 特定の OSS リポジトリでは別のメールアドレスを使用したい

通常は `git config --local` でリポジトリごとに設定するが、Git の **`includeIf`** を使えば、リモートリポジトリの URL に応じて Git の設定を自動的に切り替えられる。

しかも、Git hook や外部ツールを用意する必要はない。

この記事では、`includeIf "hasconfig:remote.*.url"` を利用して、GitHub のリポジトリ所有者に応じた Git identity の切り替えを実現する。

## 1. includeIf とは

`includeIf` は、指定した条件を満たした場合に別の Git 設定ファイルを読み込む機能だ。

代表的な条件として、以下がある。

| 条件 | 用途 |
|---|---|
| `gitdir:` | リポジトリのディレクトリで判定 |
| `onbranch:` | 現在のブランチで判定 |
| `hasconfig:remote.*.url:` | リモート URL で判定 |

今回利用するのは `hasconfig:remote.*.url` だ。

Git 2.36 以降で利用でき、リポジトリに設定された remote URL を条件に設定ファイルを読み込める。

例えば、次のように設定できる。

```gitconfig
[includeIf "hasconfig:remote.*.url:git@github.com:my-company/**"]
    path = ~/.config/git/identities/work.gitconfig
```

これは、`git@github.com:my-company/` 以下に一致する remote URL が存在する場合に、`work.gitconfig` を読み込むという意味になる。

重要なのは、ディレクトリではなく **GitHub の remote URL を基準に判定できる**点だ。

リポジトリをどのディレクトリに clone しても、同じ設定が適用される。

## 2. 設定ファイルを分割する

まず、identity ごとの設定ファイルを用意する。

構成例：

```text
~
├── .gitconfig
└── .config/
    └── git/
        └── identities/
            ├── personal.gitconfig
            ├── work.gitconfig
            └── oss.gitconfig
```

ディレクトリを作成する。

```bash
mkdir -p ~/.config/git/identities
```

### 個人用 identity

`~/.config/git/identities/personal.gitconfig`

```gitconfig
[user]
    name = My Name
    email = personal@example.com
```

### 仕事用 identity

`~/.config/git/identities/work.gitconfig`

```gitconfig
[user]
    name = My Name
    email = me@company.com
```

### OSS 用 identity

`~/.config/git/identities/oss.gitconfig`

```gitconfig
[user]
    name = My Name
    email = oss@example.com
```

それぞれのファイルには、通常の Git 設定と同じ形式で `user.name` と `user.email` を記述する。

## 3. remote URL による切り替え

次に `~/.gitconfig` に条件付き読み込みを設定する。

```gitconfig
[user]
    useConfigOnly = true

# 個人アカウント
[includeIf "hasconfig:remote.*.url:git@github.com:my-personal/**"]
    path = ~/.config/git/identities/personal.gitconfig

[includeIf "hasconfig:remote.*.url:https://github.com/my-personal/**"]
    path = ~/.config/git/identities/personal.gitconfig

# 会社の Organization
[includeIf "hasconfig:remote.*.url:git@github.com:my-company/**"]
    path = ~/.config/git/identities/work.gitconfig

[includeIf "hasconfig:remote.*.url:https://github.com/my-company/**"]
    path = ~/.config/git/identities/work.gitconfig
```

SSH と HTTPS の両方に対応させるため、それぞれの URL 形式を設定している。

これで、次のような切り替えが実現できる。

| remote URL | 適用される identity |
|---|---|
| `git@github.com:my-personal/app.git` | personal |
| `https://github.com/my-personal/app.git` | personal |
| `git@github.com:my-company/backend.git` | work |
| `https://github.com/my-company/frontend.git` | work |

**`git config --local` を実行する必要はない。**

Git が設定を読み込むたびに、現在のリポジトリに設定されている remote URL を条件として、適切な identity が選択される。

### user.useConfigOnly について

冒頭の設定には、次の項目を追加している。

```gitconfig
[user]
    useConfigOnly = true
```

これは Git が OS のユーザー名などから commit identity を推測するのを防ぐ設定だ。

global にデフォルトの `user.name` / `user.email` を設定しなければ、どの `includeIf` にもマッチしないリポジトリでは、有効な identity が不足した場合にコミットが失敗する。

仕事用のメールアドレスで個人リポジトリにコミットしてしまう、といった事故を防ぎたい場合に有効だ。

## 4. 特定のリポジトリだけ例外を設定する

Organization 単位だけでなく、特定のリポジトリだけ異なる identity を適用することもできる。

例えば、`my-company` のリポジトリは基本的に仕事用 identity を使うが、`my-company/opensource` だけ OSS 用 identity を使いたい場合。

先ほどの `~/.gitconfig` の末尾に追加する。

```gitconfig
# 特定リポジトリの例外
[includeIf "hasconfig:remote.*.url:git@github.com:my-company/opensource.git"]
    path = ~/.config/git/identities/oss.gitconfig

[includeIf "hasconfig:remote.*.url:https://github.com/my-company/opensource.git"]
    path = ~/.config/git/identities/oss.gitconfig
```

Git の設定は後から読み込まれた値が優先されるため、**Organization の設定よりも後に、リポジトリ固有の設定を書く**ことで上書きできる。

この例では以下のようになる。

```text
my-company/
├── backend       → work
├── frontend      → work
└── opensource    → oss
```

なお、ここでは `.git` で終わる URL を完全一致の条件にしている。実際の remote URL に `.git` が付いていない場合は、その形式についても条件を追加する必要がある。

## 5. 設定が反映されたか確認する

適当なリポジトリで remote URL を確認する。

```bash
git remote -v
```

例：

```text
origin  git@github.com:my-company/backend.git (fetch)
origin  git@github.com:my-company/backend.git (push)
```

次に、有効な identity を確認する。

```bash
git config --get user.name
git config --get user.email
```

結果：

```text
My Name
me@company.com
```

さらに、どの設定ファイルから読み込まれているかも確認できる。

```bash
git config --show-origin --get user.email
```

結果：

```text
file:/home/user/.config/git/identities/work.gitconfig  me@company.com
```

複数の設定が存在する場合は、以下も役立つ。

```bash
git config --show-origin --get-all user.email
```

これにより、どのファイルが値を設定しているかを調査できる。

## 6. 注意点

### origin だけを判定するわけではない

`hasconfig:remote.*.url` は、`origin` に限定した条件ではない。

`upstream` なども含めて、**いずれかの remote URL が一致すれば条件が成立する**。

例えば、次のようなリポジトリを考える。

```text
origin   git@github.com:my-personal/fork.git
upstream git@github.com:my-company/original.git
```

この場合、個人用と仕事用の両方の条件が成立する可能性がある。

そのため、複数の Organization に所属するリポジトリを扱う場合は、設定順序に注意が必要だ。

厳密に `origin` だけで判定したい場合は、`includeIf` だけでは対応できないため、Git hook などによる独自処理が必要になる。

### git config --local は優先される

リポジトリにローカル設定が存在する場合、通常はそちらが優先される。

```bash
git config --local user.email override@example.com
```

この設定がある場合、`includeIf` によって読み込まれた global スコープのメールアドレスよりも、ローカル設定が優先される。

既存の hook が `git config --local` を書き込んでいた場合も同様だ。

自動切り替えへ完全に移行するなら、該当リポジトリのローカル設定を確認し、不要な設定を削除する。

```bash
git config --local --unset user.name
git config --local --unset user.email
```

ただし、意図的に設定されたローカルの identity は削除しないこと。

### 設定ファイルに remote URL を含めない

`hasconfig:remote.*.url` によって読み込まれる設定ファイルでは、`remote.*.url` を定義できない。

例えば、`work.gitconfig` に remote URL を設定するような構成は避ける必要がある。

この制約は、読み込んだ設定ファイル自身が条件判定に影響を与える循環依存を防ぐためのものだ。

## 7. Git hook と比較した場合のメリット

以前は `post-checkout` hook で remote URL を取得し、GitHub API からユーザー情報を取得して `git config --local` に保存する方法を検討していた。

しかし、identity が固定されているのであれば、`includeIf` の方がシンプルだ。

| 比較項目 | post-checkout hook | includeIf |
|---|---|---|
| 外部スクリプト | 必要 | 不要 |
| GitHub API | 利用可能 | 不要 |
| リモート URL による切り替え | 可能 | 可能 |
| Organization 単位の設定 | 可能 | 可能 |
| リポジトリ単位の例外 | 可能 | 可能 |
| origin 限定の判定 | 可能 | 不可 |
| ローカル設定への書き込み | 実装次第 | 不要 |
| 設定変更の反映 | hook の再実行が必要になる場合がある | 次回の設定読み込み時に反映 |

特に大きな違いは、`includeIf` が **設定を永続的に書き換えない**ことだ。

例えば、リポジトリの remote URL を変更すると、新しい条件に基づいて identity が切り替わる。

また、`work.gitconfig` のメールアドレスを変更した場合も、各リポジトリのローカル設定を更新する必要がない。

## まとめ

GitHub の Organization やリポジトリごとに commit identity を切り替えたい場合、Git の `includeIf "hasconfig:remote.*.url"` が便利だ。

- Git 標準機能のみで実現できる
- SSH と HTTPS の両方に対応できる
- Organization やリポジトリ単位で切り替えられる
- `git config --local` への書き込みが不要
- 設定ファイルの変更を既存リポジトリにも反映できる

ただし、複数の remote URL が条件に一致する場合や、`origin` だけを厳密に判定したい場合は注意が必要だ。

**identity の対応関係が静的に決まっているなら `includeIf`、GitHub API を使った動的な判定や複雑な条件分岐が必要なら hook**、という使い分けがよいだろう。

### 参考資料

- [Git 公式ドキュメント：git-config](https://git-scm.com/docs/git-config)
- [Git 2.36 リリースノート](https://github.com/git/git/blob/master/Documentation/RelNotes/2.36.0.adoc)
