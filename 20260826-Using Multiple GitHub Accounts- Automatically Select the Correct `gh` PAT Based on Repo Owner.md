---
title: "GitHub複数アカウント運用で、repo ownerから自動的にghのPATを選ぶ"
description: "前提、.gitconfig、credential.useHttpPath = true が重要を中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
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

そこで、remote URLの`owner`部分をGitHub CLIのアカウント名として解釈し、

```bash
gh auth token --user ACCOUNT
```

からPATを取り出してGitに返すcredential helperを`.gitconfig`へ直接埋め込みます。

個人repositoryではownerをそのままaccountとして使用し、Organization配下などでownerと認証に使うaccountが異なる場合には、明示的なマッピングを追加できるようにします。

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

この方法では、基本的にはrepository URLが、

```text
https://github.com/aont/foo.git
```

なら、

```bash
gh auth token --user aont
```

を実行します。

したがって、デフォルトでは、

```text
repository owner = ghに登録したaccount名
```

という運用を前提にします。

この関係が成立しない場合だけ、明示的なマッピングで上書きします。

## `.gitconfig`

Organization単位やrepository単位のマッピングにも対応する場合、設定は次のようにします。

```gitconfig
[credential]
    useHttpPath = true

[credential "https://github.com/"]
    helper =
    helper = "!f() { \
        [ \"$1\" = get ] || exit 0; \
        path=''; \
        while IFS='=' read -r k v; do \
            [ \"$k\" = path ] && path=$v; \
        done; \
        [ -n \"$path\" ] || exit 0; \
        owner=${path%%/*}; \
        repo=${path#*/}; \
        repo=${repo%.git}; \
        case \"$owner/$repo\" in \
            my-org/special-repo) account='special-account' ;; \
            *) \
                case \"$owner\" in \
                    my-org) account='my-work-account' ;; \
                    another-org) account='another-account' ;; \
                    *) account=\"$owner\" ;; \
                esac \
                ;; \
        esac; \
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

ポイントは4つあります。

## `credential.useHttpPath = true` が重要

通常、Gitのcredential helperには、

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

## pathからownerとrepository名を取得する

helper内では、まずpathを取得します。

```sh
path=''
while IFS='=' read -r k v; do
    [ "$k" = path ] && path=$v
done
```

その後、

```sh
owner=${path%%/*}
```

でownerを取り出します。

repository名についても、

```sh
repo=${path#*/}
repo=${repo%.git}
```

として取得します。

たとえば、

```text
path=aont/foo.git
```

なら、

```text
owner=aont
repo=foo
```

となります。

明示的なマッピングが存在しなければ、

```sh
account="$owner"
```

として、ownerをそのままGitHub CLIのaccountとして使用します。

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

## Organizationのownerを別のaccountへマッピングする

単純な、

```text
owner = account
```

というルールは個人repositoryでは扱いやすいですが、Organization配下のrepositoryでは成立しない場合があります。

たとえばremote URLが、

```text
https://github.com/my-org/foo.git
```

であっても、そのOrganizationへアクセスするGitHubアカウントが、

```text
my-work-account
```

だったとします。

この場合、

```text
repository owner = my-org
認証に使うaccount = my-work-account
```

なので、

```bash
gh auth token --user my-org
```

としても正しいcredentialは取得できません。

そこで、owner単位のマッピングを追加します。

```sh
case "$owner" in
    my-org) account='my-work-account' ;;
    another-org) account='another-account' ;;
    *) account="$owner" ;;
esac
```

これによって、

```text
github.com/my-org/foo
        ↓
owner = my-org
        ↓
account = my-work-account
        ↓
gh auth token --user my-work-account
```

という動作になります。

明示的に指定していないownerについては、

```sh
*) account="$owner" ;;
```

にフォールバックするため、従来どおりownerをそのままaccountとして使用します。

つまり、個人repositoryのためにすべてのownerを列挙する必要はありません。

## 特定のrepositoryだけ別accountを使う

同じOrganization配下でも、repositoryによって認証に使うアカウントを変えたい場合があります。

たとえば、

```text
https://github.com/my-org/foo.git
https://github.com/my-org/special-repo.git
```

があり、通常は、

```text
my-work-account
```

を使うものの、`special-repo`だけは、

```text
special-account
```

を使いたいとします。

その場合は、owner単位のマッピングより先に`owner/repo`単位で判定します。

```sh
case "$owner/$repo" in
    my-org/special-repo) account='special-account' ;;
    *)
        case "$owner" in
            my-org) account='my-work-account' ;;
            another-org) account='another-account' ;;
            *) account="$owner" ;;
        esac
        ;;
esac
```

この構成では、優先順位が、

```text
repository単位のマッピング
        ↓
owner単位のマッピング
        ↓
ownerをそのままaccountとして使用
```

となります。

たとえば、

```text
github.com/my-org/special-repo
        ↓
special-account

github.com/my-org/other-repo
        ↓
my-work-account

github.com/aont/foo
        ↓
aont
```

という形です。

これなら、個人repository、Organization配下のrepository、さらに一部repositoryだけの例外を同じcredential helperで扱えます。

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

GitHub repositoryとGistでは多少ルールが異なりますが、どちらもURLのpathをcredential routingに利用できます。

```text
github.com/OWNER/REPO
gist.github.com/USER/GIST
                ↓
        accountを決定
                ↓
gh auth token --user ACCOUNT
```

通常の個人repositoryやGistではownerまたはUSERをそのまま使い、Organization配下のrepositoryについては必要に応じてマッピングで上書きする形です。

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

Organization単位のマッピングについても確認できます。

```bash
printf '%s\n' \
  'protocol=https' \
  'host=github.com' \
  'path=my-org/foo.git' \
  '' |
git credential fill
```

上記の設定なら、期待するusernameは、

```text
username=my-work-account
```

です。

さらに、repository単位のoverrideについて、

```bash
printf '%s\n' \
  'protocol=https' \
  'host=github.com' \
  'path=my-org/special-repo.git' \
  '' |
git credential fill
```

を実行すると、

```text
username=special-account
```

となります。

もし`username`が想定と異なる場合は、

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

ただ、複数アカウントを同一ホスト`github.com`で使っている場合、「repository ownerやrepositoryそのものに応じて、どの`gh` accountを使うか」を明示的に制御したくなります。

今回のhelperでは、

```text
remote URL
    ↓
owner/repositoryを抽出
    ↓
repository単位のマッピングがあれば使用
    ↓
owner単位のマッピングがあれば使用
    ↓
なければownerをaccountとして使用
    ↓
gh auth token --user account
```

という規則にしているため、現在どのアカウントが`gh`でactiveになっているかを意識する必要がありません。

## この構成の利点

この方法だとPATそのものを`.gitconfig`に保存しません。

PATは必要になるたびに、

```bash
gh auth token --user ACCOUNT
```

から取得します。

そのため、設定として保存されるのは「どのrepositoryにどのアカウントを使うか」というルールだけです。

また、repositoryごとに、

```bash
gh auth switch
```

したり、

```bash
git config credential.username ...
```

を設定したりする必要もありません。

remote URLそのものがcredential routingの入力になります。

個人repositoryなら、

```text
github.com/aont/foo
        ↓
account = aont
```

Organization配下なら、

```text
github.com/my-org/foo
        ↓
account = my-work-account
```

さらに特定repositoryだけ例外にしたければ、

```text
github.com/my-org/special-repo
        ↓
account = special-account
```

という形で扱えます。

## まとめ

複数GitHubアカウントをHTTPSで使う場合、remote URLのpathを使って、適切な`gh` accountを自動的に選択できます。

仕組みとしては、

```text
Git remote URL
  ↓
credential.useHttpPath
  ↓
path=owner/repo.git
  ↓
ownerとrepoを抽出
  ↓
repository単位のマッピングがある？
  ├─ yes → 指定されたaccount
  └─ no
       ↓
     owner単位のマッピングがある？
       ├─ yes → 指定されたaccount
       └─ no → ownerをそのままaccountとして使用
  ↓
gh auth token --user account
  ↓
username/passwordとしてGitへ返す
```

という流れです。

基本ルールはこれまでと同じく、

```text
repository owner = GitHub account
```

です。

そのうえで、Organization配下など、

```text
repository owner != 認証に使うaccount
```

となるケースだけowner単位のマッピングを追加できます。

さらに、同じOrganization内でも認証アカウントを分ける必要があれば、repository単位でoverrideできます。

これによって、PAT自体を`.gitconfig`へ保存したり、外部スクリプトを用意したりすることなく、複数GitHubアカウントのcredential routingを`.gitconfig`だけで完結できます。
