---
title: "GitHubのユーザー名だけでコミット用ユーザー情報を設定するGitエイリアス"
description: "GitHub REST APIからユーザー情報を取得し、現在のリポジトリにnoreplyメールアドレスを設定するGitエイリアスを.gitconfigに定義する。"
pubDatetime: 2026-10-07T12:53:00+09:00
---

複数のGitHubアカウントを使っていると、リポジトリごとにコミットの作成者情報を設定するのが少し面倒です。

```bash
git config --local user.name "your-username"
git config --local user.email "12345678+your-username@users.noreply.github.com"
```

特にGitHubの数値ユーザーIDを毎回調べるのは手間です。そこで、**GitHubのユーザー名を渡すだけで、現在のリポジトリに `user.name` と `user.email` を設定する**Gitエイリアスを作りました。

`~/.gitconfig` に直接定義するため、別のスクリプトファイルを用意する必要はありません。

## 何をするエイリアス？

例えば、Gitリポジトリ内で次のように実行します。

```bash
git config-gh octocat
```

エイリアスはGitHub REST APIからユーザーの `id`、`login`、`name` を取得し、次の形式で設定します。

```text
user.name  = The Octocat
user.email = 583231+octocat@users.noreply.github.com
```

`user.name` にはGitHubプロフィールの表示名（`name`）を使用し、未設定の場合はユーザー名（`login`）を使用します。

また、`git config --local` を使うため、**コミット用ユーザー情報の変更は現在のリポジトリだけ**に適用されます。グローバルな `user.name` や `user.email` は変更しません。

## 必要なもの

- Git
- `curl`
- `jq`
- POSIX互換のシェル（`sh`）

Gitエイリアスのシェル実行機能を利用するため、Bashを明示的に起動する必要はありません。

また、`gh`（GitHub CLI）のインストールや、`gh auth switch` によるログインアカウントの切り替えも不要です。

## Gitエイリアスの設定

`~/.gitconfig` の `[alias]` セクションに、以下を追加します。

```gitconfig
[alias]
    config-gh = "!f() { \
        if [ \"$#\" -ne 1 ] || [ -z \"$1\" ]; then \
            printf 'Usage: git config-gh <github-username>\\n' >&2; return 1; \
        fi; \
        username=$1; \
        case \"$username\" in \
            -*|*-|*[!A-Za-z0-9-]*|*--*) \
                printf 'Invalid GitHub username: %s\\n' \"$username\" >&2; return 1 ;; \
        esac; \
        if [ \"${#username}\" -gt 39 ]; then \
            printf 'Invalid GitHub username: %s\\n' \"$username\" >&2; return 1; \
        fi; \
        for cmd in curl jq git; do \
            if ! command -v \"$cmd\" >/dev/null 2>&1; then \
                printf 'Required command not found: %s\\n' \"$cmd\" >&2; return 1; \
            fi; \
        done; \
        if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then \
            printf 'Run this command inside a Git working tree.\\n' >&2; return 1; \
        fi; \
        if ! response=$(curl --fail --silent --show-error --location \
            --connect-timeout 5 --max-time 15 \
            \"https://api.github.com/users/${username}\"); then \
            printf 'Failed to fetch GitHub user: %s\\n' \"$username\" >&2; return 1; \
        fi; \
        if ! user_info=$(printf '%s' \"$response\" | jq -er ' \
            select(.id | type == \"number\") \
            | select(.login | type == \"string\" and length > 0) \
            | [.id, .login, (.name // .login)] \
            | @tsv \
        '); then \
            printf 'Unexpected GitHub API response for: %s\\n' \"$username\" >&2; return 1; \
        fi; \
        tab=$(printf '\\t'); \
        id=${user_info%%\"$tab\"*}; \
        user_info=${user_info#*\"$tab\"}; \
        login=${user_info%%\"$tab\"*}; \
        name=${user_info#*\"$tab\"}; \
        email=\"${id}+${login}@users.noreply.github.com\"; \
        git config --local user.name \"$name\" || return 1; \
        git config --local user.email \"$email\" || return 1; \
        printf 'user.name  = %s\\n' \"$(git config --local user.name)\"; \
        printf 'user.email = %s\\n' \"$(git config --local user.email)\"; \
    }; f"
```

エイリアスの先頭にある `!` は、Gitのサブコマンドではなく、シェルコマンドとして実行するための指定です。

```sh
!f() { ...; }; f
```

この形式では、シェル関数 `f` を定義した直後に実行します。

Gitエイリアスに渡した追加引数はシェル関数にも渡されるため、`$1` でGitHubのユーザー名を取得できます。

処理はPOSIX `sh` 互換の構文で記述しているので、`bash -c` は不要です。

なお、`.gitconfig` 内ではダブルクォートやバックスラッシュをエスケープする必要があります。そのため、通常のシェルスクリプトとは異なり、`\"` や `\\n` といった表記になっています。

## 使い方

設定したいGitリポジトリに移動し、GitHubのユーザー名を指定して実行します。

```bash
cd /path/to/repository
git config-gh octocat
```

設定された情報を確認するには、次のコマンドを使います。

```bash
git config --local --get user.name
git config --local --get user.email
```

エイリアスが登録されているかは、次のコマンドで確認できます。

```bash
git config --global --get alias.config-gh
```

`~/.gitconfig` に直接定義しているため、スクリプトファイルの作成、実行権限の付与、`PATH` の設定は不要です。

## 仕組み

### 1. Gitエイリアスからシェル関数を実行

Gitには、エイリアスの定義を `!` で始めることで、任意のシェルコマンドを実行できる仕組みがあります。

例えば、次のようなエイリアスを定義できます。

```gitconfig
[alias]
    hello = "!f() { echo \"Hello, $1\"; }; f"
```

実行すると、引数がシェル関数に渡されます。

```bash
git hello octocat
# Hello, octocat
```

今回の `config-gh` も同じ仕組みを使っています。

### 2. 引数と実行環境を検証

最初に、引数が1つだけ指定されていることを確認します。

続いて、`case` と文字数チェックを使い、GitHubのユーザー名として不正な文字列を除外します。

ユーザー名は英数字とハイフンで構成され、先頭と末尾にはハイフンを使用できません。また、連続するハイフンや39文字を超える文字列も拒否します。

さらに、`curl`、`jq`、`git` が実行可能であることと、現在のディレクトリがGitワークツリー内にあることを確認します。

### 3. GitHub REST APIからユーザー情報を取得

GitHubには、ユーザー名から公開プロフィールを取得できるAPIがあります。

```bash
curl -fsSL https://api.github.com/users/octocat \
  | jq '{id, login, name}'
```

`id` はGitHub内部の数値ユーザーID、`login` はユーザー名、`name` はプロフィールの表示名です。

エイリアスでは `jq` を使って必要な情報を取り出します。

```jq
select(.id | type == "number")
| select(.login | type == "string" and length > 0)
| [.id, .login, (.name // .login)]
| @tsv
```

`name` が `null` の場合は `login` を使用します。

取得した情報はタブ区切りで出力し、シェルのパラメータ展開を使って `id`、`login`、`name` に分割しています。

### 4. GitHubのnoreplyメールアドレスを組み立てる

取得した情報から、次の形式のメールアドレスを組み立てます。

```text
<ID>+<USERNAME>@users.noreply.github.com
```

これはGitHubが提供するプライベートメールアドレスの形式の一つです。

実際に利用できるnoreplyアドレスは、GitHubアカウントの作成時期やメールのプライバシー設定によって異なる場合があります。GitHubの「Settings → Emails」で確認してください。

### 5. リポジトリのローカル設定を書き換える

取得したプロフィール表示名と生成したメールアドレスを使って、現在のリポジトリの設定を変更します。

```sh
git config --local user.name "$name"
git config --local user.email "$email"
```

`--local` は現在のリポジトリの設定を変更するオプションです。

エイリアスの定義自体は `~/.gitconfig` にありますが、`user.name` と `user.email` の変更先は現在のリポジトリのローカル設定です。

ほかのリポジトリやグローバルなコミット用ユーザー情報には影響しません。

## 注意点

### GitHubの認証アカウントは切り替わらない

このエイリアスが変更するのは、Gitコミットに記録される作成者名とメールアドレスだけです。

`git push` 時に使うSSHキー、認証トークン、GitHub CLIのログイン状態は変更されません。

また、変更前に作成したコミットの作成者情報も書き換えられません。

### noreplyメールアドレスは設定を確認する

GitHubのnoreplyメールアドレスを組み立てても、それが必ずそのアカウントで有効とは限りません。

アカウントの作成時期やメールのプライバシー設定によっては、異なる形式のnoreplyアドレスが使用されている場合があります。

コミットをGitHubアカウントに正しく紐付けるには、実際にアカウントに関連付けられているメールアドレスを使用してください。

### GitHub APIのレート制限

GitHub REST APIを認証なしで呼び出すため、レート制限があります。

頻繁に大量実行する用途には向きません。また、ネットワーク障害や存在しないユーザー名の場合はエラー終了します。

### user.nameにはプロフィール表示名を使用する

`user.name` にはGitHubの `name` を使用し、未設定の場合は `login` にフォールバックします。

常にGitHubのユーザー名を設定したい場合は、`jq` の次の部分を変更してください。

```jq
[.id, .login, (.name // .login)]
```

変更後：

```jq
[.id, .login, .login]
```

これで `user.name` に常にGitHubの `login` が設定されます。

## まとめ

GitHubのユーザー名さえ分かれば、REST APIを使って数値ユーザーIDとプロフィール情報を取得し、現在のリポジトリのコミット用ユーザー情報を設定できます。

Gitエイリアスとして `~/.gitconfig` に直接定義できるため、外部スクリプトの管理や実行権限の設定は必要ありません。

また、`gh auth switch` で認証アカウントを切り替える必要がなく、普段の認証状態を維持できる点も利点です。

複数のGitHubアカウントを扱う開発環境で、リポジトリごとの設定作業を簡略化できます。

参考：

- [Git — git-config Documentation](https://git-scm.com/docs/git-config)
- [GitHub REST API — Get a user](https://docs.github.com/en/rest/users/users#get-a-user)
- [GitHub — Setting your commit email address](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/setting-your-commit-email-address)
- [GitHub — Email addresses reference](https://docs.github.com/en/account-and-profile/reference/email-addresses-reference)
