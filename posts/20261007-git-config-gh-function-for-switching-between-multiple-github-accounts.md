---
title: "複数のGitHubアカウントを使い分けるための `git-config-gh` 関数"
description: "Bash関数で、ghのアカウント切り替えとGitHub APIからのユーザー情報取得を利用し、現在のリポジトリにnoreplyメールアドレスを自動設定する。"
pubDatetime: 2026-10-07T12:53:00+09:00
---

仕事用・個人用など、複数のGitHubアカウントを1台のPCで使い分けていると、リポジトリごとに `git config user.name` と `git config user.email` を切り替えたくなることがあります。

特にGitHubのメールアドレス非公開設定を使っている場合、コミットには次のような noreply アドレスを設定できます。

```text
12345678+username@users.noreply.github.com
```

これを毎回手作業で設定するのは面倒です。

そこで、GitHub CLI の `gh` を使って対象アカウントへ一時的に切り替え、GitHub APIからユーザー情報を取得して、現在のGitリポジトリへ `--local` 設定する Bash 関数 `git-config-gh` を作ります。

## 完成形

```bash
git-config-gh() (
  local username="$1"
  local original
  local login
  local id

  if [[ -z "$username" ]]; then
    echo "usage: git-config-gh <github-username>" >&2
    return 1
  fi

  original="$(gh api user --jq '.login')" || return 1

  # 成功・失敗にかかわらず元のアカウントへ戻す
  trap 'gh auth switch --hostname github.com --user "$original" >/dev/null 2>&1 || true' EXIT

  gh auth switch --hostname github.com --user "$username" || return 1

  read -r id login < <(
    gh api user --jq '"\(.id) \(.login)"'
  ) || return 1

  git config --local user.name "$login"
  git config --local user.email "${id}+${login}@users.noreply.github.com"

  echo "user.name  = $(git config --local user.name)"
  echo "user.email = $(git config --local user.email)"
)
```

たとえば `alice` というGitHubアカウントを現在のリポジトリに設定する場合は、次のように実行します。

```bash
git-config-gh alice
```

実行後は、現在のリポジトリの `.git/config` に次のような設定が入ります。

```ini
[user]
    name = alice
    email = 12345678+alice@users.noreply.github.com
```

## 前提

この関数では GitHub CLI の `gh` を利用します。

また、使用したいGitHubアカウントがあらかじめ `gh` にログイン済みである必要があります。

ログイン状態は次のコマンドで確認できます。

```bash
gh auth status
```

複数アカウントを登録している場合、`gh auth switch` でアクティブなアカウントを切り替えられます。

```bash
gh auth switch --user alice
```

今回の関数では、この切り替えを内部で自動的に行います。

## 処理の流れ

`git-config-gh alice` を実行すると、内部ではおおむね次の処理が行われます。

```text
現在の gh アカウントを取得
        ↓
alice に gh auth switch
        ↓
GitHub API から ID と login を取得
        ↓
git config --local user.name を設定
        ↓
git config --local user.email を設定
        ↓
元の gh アカウントへ戻す
```

ポイントは、Gitの設定だけを変更し、`gh` 側のアクティブアカウントは最終的に元へ戻すことです。

## 現在のGitHubアカウントを保存する

まず、現在 `gh` でアクティブになっているアカウントを取得します。

```bash
original="$(gh api user --jq '.login')" || return 1
```

`gh api user` は、現在認証に使われているGitHubユーザーの情報を取得します。

`--jq '.login'` を指定することで、レスポンス全体ではなくGitHubのユーザー名だけを取り出しています。

たとえば現在のアカウントが `bob` なら、

```text
bob
```

が `original` に保存されます。

## 対象アカウントへ切り替える

次に、引数で指定されたGitHubアカウントへ切り替えます。

```bash
gh auth switch --hostname github.com --user "$username" || return 1
```

たとえば、

```bash
git-config-gh alice
```

と実行した場合は、実質的に次の操作が行われます。

```bash
gh auth switch --hostname github.com --user alice
```

ここでアカウントを切り替えているため、その後の `gh api user` は `alice` の情報を返します。

## GitHubのユーザーIDとloginを取得する

GitHubの noreply メールアドレスには、ユーザー名だけでなく数値のユーザーIDも使います。

そこで、APIから両方を取得します。

```bash
read -r id login < <(
  gh api user --jq '"\(.id) \(.login)"'
)
```

たとえばAPI上の情報が、

```json
{
  "login": "alice",
  "id": 12345678
}
```

だった場合、次の値がセットされます。

```text
id=12345678
login=alice
```

この2つを使って、

```text
12345678+alice@users.noreply.github.com
```

というメールアドレスを組み立てます。

## `git config --local` を設定する

取得した情報を現在のリポジトリへ設定します。

```bash
git config --local user.name "$login"
git config --local user.email "${id}+${login}@users.noreply.github.com"
```

ここでは明示的に `--local` を付けています。

そのため変更されるのは、現在のGitリポジトリの `.git/config` だけです。

グローバル設定、

```bash
git config --global user.name
git config --global user.email
```

には影響しません。

複数のGitHubアカウントを使い分ける用途では、このリポジトリ単位の設定のほうが事故を起こしにくくなります。

## 終了時に元のアカウントへ戻す

この関数で重要なのが `trap` です。

```bash
trap 'gh auth switch --hostname github.com --user "$original" || true' EXIT
```

関数の途中で処理が終了しても、`EXIT` 時に元のGitHubアカウントへ戻します。

通常終了だけでなく、途中で `return 1` になった場合にも復帰処理が走ります。

たとえば、

```text
bob がアクティブ
↓
git-config-gh alice
↓
alice へ切り替え
↓
Git設定
↓
bob へ復帰
```

という動作になります。

## なぜ関数を `()` で囲んでいるのか

通常のBash関数は、

```bash
foo() {
  ...
}
```

のように `{}` で書くことが多いですが、今回の関数は、

```bash
git-config-gh() (
  ...
)
```

としています。

`()` の中はサブシェルで実行されます。

これにより、関数内で設定した `trap` やローカルな実行環境を、呼び出し元のシェルから分離できます。

今回のように、

```bash
trap ... EXIT
```

を使う関数では扱いやすい書き方です。

## `.bashrc` や `.zshrc` に登録する

日常的に使うなら、関数をシェルの設定ファイルへ追加します。

Bashなら、

```bash
~/.bashrc
```

macOSなどでZshを使っているなら、

```bash
~/.zshrc
```

に関数を書いておきます。

設定を反映したあと、

```bash
git-config-gh alice
```

のように実行できます。

## 設定結果を確認する

関数の最後では、設定された値を表示しています。

```bash
echo "user.name  = $(git config --local user.name)"
echo "user.email = $(git config --local user.email)"
```

たとえば、

```text
user.name  = alice
user.email = 12345678+alice@users.noreply.github.com
```

のように表示されます。

手動で確認する場合は次のコマンドでも確認できます。

```bash
git config --local user.name
git config --local user.email
```

あるいは、

```bash
git config --local --list
```

でも確認できます。

## まとめ

`git-config-gh` を用意しておくと、複数のGitHubアカウントを使っている環境でも、

```bash
git-config-gh alice
```

だけで現在のリポジトリのコミットユーザーを切り替えられます。

この関数のポイントは次の通りです。

- `gh auth switch` で対象GitHubアカウントへ一時的に切り替える
- GitHub APIからユーザーIDとloginを取得する
- `ID+login@users.noreply.github.com` を自動生成する
- `git config --local` だけを書き換える
- 処理終了時には元の `gh` アカウントへ戻す
- `GH_TOKEN` を明示的に取り回す必要がない

仕事用・個人用など、リポジトリごとにGitHubアカウントを分けている場合には、シンプルながら使いやすい補助関数です。