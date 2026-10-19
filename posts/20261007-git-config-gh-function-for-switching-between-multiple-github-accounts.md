---
title: "GitHubのユーザー名だけでGitのコミット用ユーザー情報を設定するシェルスクリプト"
description: "GitHub APIからのユーザー情報取得を利用し、現在のリポジトリにnoreplyメールアドレスを自動設定する。"
pubDatetime: 2026-10-07T12:53:00+09:00
---

複数のGitHubアカウントを使っていると、リポジトリごとにコミットの作成者情報を設定するのが少し面倒です。

```bash
git config --local user.name "your-username"
git config --local user.email "12345678+your-username@users.noreply.github.com"
```

特にGitHubの数値ユーザーIDを毎回調べるのは手間です。そこで、**GitHubのユーザー名を渡すだけで、現在のリポジトリに `user.name` と `user.email` を設定する**スクリプトを作りました。

## 何をするスクリプト？

例えば、Gitリポジトリ内で次のように実行します。

```bash
./git-config-gh.sh octocat
```

スクリプトはGitHub REST APIからユーザーの `id` と `login` を取得し、次の形式で設定します。

```text
user.name  = octocat
user.email = <数値ID>+octocat@users.noreply.github.com
```

`git config --local` を使うため、**現在のリポジトリだけ**に適用され、グローバル設定は変更しません。

## 必要なもの

- Bash
- `curl`
- `jq`
- Git

`gh`（GitHub CLI）のインストールや、`gh auth switch` によるログインアカウントの切り替えは不要です。

## スクリプト

以下を `git-config-gh.sh` として保存します。

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  printf 'Usage: %s <github-username>\n' "${0##*/}" >&2
}

if [[ $# -ne 1 || -z "$1" ]]; then
  usage
  exit 1
fi

username="$1"
if [[ ! "$username" =~ ^[A-Za-z0-9]([A-Za-z0-9-]{0,37}[A-Za-z0-9])?$ ]]; then
  printf 'Invalid GitHub username: %s\n' "$username" >&2
  exit 1
fi

for command in curl jq git; do
  if ! command -v "$command" >/dev/null 2>&1; then
    printf 'Required command not found: %s\n' "$command" >&2
    exit 1
  fi
done

if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  printf 'Run this command inside a Git working tree.\n' >&2
  exit 1
fi

if ! response="$(curl --fail --silent --show-error --location \
  --connect-timeout 5 --max-time 15 \
  "https://api.github.com/users/${username}")"; then
  printf 'Failed to fetch GitHub user: %s\n' "$username" >&2
  exit 1
fi

if ! user_info="$(jq -er '
  select(.id | type == "number")
  | select(.login | type == "string" and length > 0)
  | [.id, .login, (.name // .login)]
  | @tsv
' <<< "$response")"; then
  printf 'Unexpected GitHub API response for: %s\n' "$username" >&2
  exit 1
fi

IFS=$'\t' read -r id login name <<< "$user_info"
email="${id}+${login}@users.noreply.github.com"

git config --local user.name "$name"
git config --local user.email "$email"

printf 'user.name  = %s\n' "$(git config --local user.name)"
printf 'user.email = %s\n' "$(git config --local user.email)"
```

## 使い方

実行権限を付けます。

```bash
chmod +x git-config-gh.sh
```

設定したいリポジトリ内で実行します（スクリプトを別の場所に保存した場合は、そのパスを指定します）。

```bash
cd /path/to/repository
/path/to/git-config-gh.sh octocat
```

設定を確認するには、次のコマンドを使います。

```bash
git config --local --get user.name
git config --local --get user.email
```

頻繁に使うなら、スクリプトを `~/bin/git-config-gh` などのPATHが通った場所に置くと便利です。

## 仕組み

### 1. GitHub REST APIからユーザー情報を取得

GitHubには、ユーザー名から公開プロフィールを取得できるAPIがあります。

```bash
curl -fsSL https://api.github.com/users/octocat | jq '{id, login}'
```

`id` はGitHub内部の数値ユーザーID、`login` はユーザー名です。スクリプトでは、この2つをJSONから取り出します。

### 2. GitHubのnoreplyメールアドレスを組み立てる

取得した情報から、次の形式のメールアドレスを組み立てます。

```text
<ID>+<USERNAME>@users.noreply.github.com
```

これはGitHubが利用するプライベートメールアドレスの形式の一つです。実際にGitHubアカウントに関連付けられたnoreplyアドレスと一致するかは、GitHubのメール設定で確認してください。

### 3. リポジトリのローカル設定を書き換える

```bash
git config --local user.name "$login"
git config --local user.email "$email"
```

`--local` は現在のリポジトリの設定を変更します。別のリポジトリや `~/.gitconfig` には影響しません。

## 注意点

**GitHubへの認証アカウントは切り替わりません。** このスクリプトが変更するのは、Gitコミットに記録される作成者名とメールアドレスだけです。`git push` 時に使うSSHキー、認証トークン、GitHub CLIのログイン状態は変更されません。

また、**GitHubのnoreplyメールアドレスを組み立てても、それが必ずそのアカウントで有効とは限りません。** GitHubの「Settings → Emails」で実際のアドレスを確認してください。GitHubへのコミットの紐付けやメールアドレスのプライバシー設定は、アカウント側の設定にも依存します。

GitHub APIを認証なしで呼ぶため、レート制限があります。頻繁に大量実行する用途には向きません。また、ネットワーク障害や存在しないユーザー名ではエラー終了します。

なお、`user.name` に設定するのはGitHubの `login` です。プロフィールの表示名（`name`）を設定するわけではありません。

## まとめ

GitHubのユーザー名さえ分かれば、REST APIを使って数値IDを取得し、現在のリポジトリのコミット用ユーザー情報を設定できます。

`gh auth switch` でアカウントを切り替える必要がなく、普段の認証状態を維持できる点が利点です。複数アカウントを扱う開発環境で、リポジトリごとの設定作業を簡略化できます。

参考: [GitHub REST API — Get a user](https://docs.github.com/en/rest/users/users#get-a-user) / [GitHub — Setting your commit email address](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/setting-your-commit-email-address)
