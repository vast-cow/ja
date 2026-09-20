---
title: "GitHub複数アカウント運用で、repo ownerから自動的にghのPATを選ぶ"
description: ""
pubDatetime: 2026-08-26T07:16:37.440Z
---

GitHubで複数アカウントを使っていると、HTTPSでの`git push`や`git pull`時に「どのアカウントのPATを使うか」が地味に面倒です。

特に、

```text
https://github.com/aont/foo.git
https://github.com/another-user/bar.git
```

のように複数ownerのrepositoryを扱っていると、Git Credential Managerや`gh auth git-credential`が現在のアクティブアカウントを使ってしまい、

```text
remote: Permission to owner/repo.git denied to other-user.
```

のようなエラーになることがあります。

そこで、remote URLの`owner`部分をそのままGitHub CLIのアカウント名として解釈し、

```bash
gh auth token --user OWNER
```

からPATを取り出してGitに返すcredential helperを`.gitconfig`へ直接埋め込みます。

外部スクリプトも不要です。

## 前提

まず、利用するGitHubアカウントはあらかじめ`gh`にログインしておきます。

```bash
gh auth login
```

複数アカウントを登録している場合は、

```bash
gh auth status
```

で確認できます。

この方法では、たとえばrepository URLが

```text
https://github.com/aont/foo.git
```

なら、

```bash
gh auth token --user aont
```

を実行します。

したがって、基本的には

```text
repository owner = ghに登録したaccount名
```

という運用を前提にします。

## `.gitconfig`

設定は次のようにします。

```gitconfig
[credential]
    useHttpPath = true

[credential "https://github.com/"]
    helper =
    helper = "!f() { \
        [ \"$1\" = get ] || exit 0; \
        account=''; \
        while IFS='=' read -r k v; do \
            [ \"$k\" = path ] && account=${v%%/*}; \
        done; \
        [ -n \"$account\" ] || exit 0; \
        token=$(gh auth token --user \"$account\") || exit 0; \
        printf 'username=%s\\npassword=%s\\n' \"$account\" \"$token\"; \
    }; f"

[credential "https://gist.github.com/"]
    helper =
    helper = "!f() { \
        [ \"$1\" = get ] || exit 0; \
        account=''; \
        while IFS='=' read -r k v; do \
            [ \"$k\" = path ] && account=${v%%/*}; \
        done; \
        [ -n \"$account\" ] || exit 0; \
        token=$(gh auth token --user \"$account\") || exit 0; \
        printf 'username=%s\\npassword=%s\\n' \"$account\" \"$token\"; \
    }; f"
```

ポイントは3つあります。

## `credential.useHttpPath = true` が重要

通常、Gitのcredential helperには

```text
protocol=https
host=github.com
```

程度しか渡されません。

しかし今回必要なのは、

```text
aont/foo.git
```

というpathです。

そこで、

```gitconfig
[credential]
    useHttpPath = true
```

を設定します。

これによってcredential helperの標準入力に、

```text
protocol=https
host=github.com
path=aont/foo.git
```

のようにpathも渡されます。

## pathの先頭をaccountとして使う

helper内では、

```sh
[ "$k" = path ] && account=${v%%/*}
```

として、pathの最初の要素を取得しています。

たとえば、

```text
path=aont/foo.git
```

なら、

```text
account=aont
```

になります。

その後、

```sh
token=$(gh auth token --user "$account")
```

で、そのアカウントに対応するPATをGitHub CLIから取得します。

つまり、

```text
https://github.com/aont/foo.git
```

へのアクセス時は自動的に、

```bash
gh auth token --user aont
```

相当になります。

一方、

```text
https://github.com/another-user/bar.git
```

なら、

```bash
gh auth token --user another-user
```

になります。

`gh auth switch`を毎回実行する必要はありません。

## `helper =` で既存credential helperをリセットする

もう一つ重要なのが、

```gitconfig
helper =
```

です。

Git for Windowsなどでは、すでにGit Credential Managerや`gh auth git-credential`などが設定されている場合があります。

たとえば別のhelperが先にcredentialを返すと、今回のhelperまで処理が回ってきません。

その結果、

```text
remote: Permission to foo/bar.git denied to wrong-account.
```

のようになります。

そこで、

```gitconfig
helper =
helper = "!f() { ... }; f"
```

とします。

最初の空の`helper =`で、それ以前に設定されていたcredential helperをリセットし、その後に今回のhelperだけを登録しています。

複数GitHubアカウントを扱う場合には、この部分がかなり重要です。

## Gistにも同じ仕組みを使う

Gistにも同じcredential helperを設定できます。

ただし通常のGist clone URLは、

```text
https://gist.github.com/GIST_ID.git
```

となっていて、URLからユーザー名を判断できません。

そこで、この運用ではGistのremote URLを意図的に、

```text
https://gist.github.com/USER/GIST_ID.git
```

という形式にします。

たとえば、

```text
https://gist.github.com/aont/0123456789abcdef.git
```

なら、credential helperから見ると、

```text
path=aont/0123456789abcdef.git
```

となるので、

```bash
gh auth token --user aont
```

を自動的に使えます。

つまりGitHub repositoryとGistを同じルールで扱えます。

```text
github.com/USER/REPO
gist.github.com/USER/GIST
                ↓
             USERを抽出
                ↓
gh auth token --user USER
```

## 動作確認

Gitが実際にどのcredentialを取得するかは、`git credential fill`で確認できます。

たとえば、

```bash
printf '%s\n' \
  'protocol=https' \
  'host=github.com' \
  'path=aont/foo.git' \
  '' |
git credential fill
```

を実行します。

期待する出力は、

```text
protocol=https
host=github.com
username=aont
password=...
```

です。

ここで`username`が別アカウントになっている場合は、

```bash
git config --show-origin --get-all credential.helper
```

で、別のcredential helperが残っていないか確認するとよいです。

## なぜ`gh auth git-credential`をそのまま使わないのか

GitHub CLIには標準で、

```bash
gh auth git-credential
```

があります。

通常の単一アカウント運用ならこれで十分です。

ただ、複数アカウントを同一ホスト`github.com`で使っている場合、「repository ownerに応じてどの`gh` accountを使うか」を明示的に制御したくなります。

今回のhelperでは、

```text
remote URL
    ↓
ownerを抽出
    ↓
gh auth token --user owner
```

という非常に単純な規則にしているため、現在どのアカウントが`gh`でactiveになっているかを意識する必要がありません。

## この構成の利点

この方法だとPATそのものを`.gitconfig`に保存しません。

PATは必要になるたびに、

```bash
gh auth token --user ACCOUNT
```

から取得します。

そのため、設定として保存されるのは「どのアカウントを使うか」というルールだけです。

また、repositoryごとに、

```bash
gh auth switch
```

したり、

```bash
git config credential.username ...
```

を設定したりする必要もありません。

remote URLそのものがcredential routingの情報になります。

## まとめ

複数GitHubアカウントをHTTPSで使う場合、

```text
github.com/<account>/<repo>
```

の`account`部分をそのまま`gh`のaccount選択に使うと、かなりシンプルに運用できます。

仕組みとしては、

```text
Git remote URL
  ↓
credential.useHttpPath
  ↓
path=owner/repo.git
  ↓
ownerを抽出
  ↓
gh auth token --user owner
  ↓
username/passwordとしてGitへ返す
```

だけです。

個人アカウントを複数使っていて、

```text
repository owner = GitHub account
```

という関係が成立している環境なら、かなり扱いやすい方法だと思います。

Organization配下のrepositoryなどで、

```text
owner != 認証に使うaccount
```

となる場合だけは別途マッピングが必要ですが、個人アカウント中心ならまずこの構成で十分です。
