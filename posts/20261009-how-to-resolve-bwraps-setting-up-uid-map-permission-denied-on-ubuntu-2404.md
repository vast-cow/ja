---
title: "Ubuntu 24.04でbwrapの「setting up uid map: Permission denied」を解決する方法"
description: "AppArmorのプロファイルをbwrap専用に適用して、システム全体の制限を維持したままエラーを解決する方法を解説。"
pubDatetime: 2026-10-09T02:30:00+09:00
---

Ubuntu 24.04でBubblewrap（`bwrap`）を使用してサンドボックス環境を構築しようとしたところ、次のエラーが発生しました。

```text
bwrap: setting up uid map: Permission denied
```

調査した結果、Ubuntu 24.04で導入されているAppArmorのUser Namespace制限が原因でした。

今回は、AppArmorのセキュリティ制限をシステム全体で無効化することなく、**bwrap専用のAppArmorプロファイルを適用することで解決**できました。

この記事では、エラーの原因、調査方法、実際に解決した手順を紹介します。

## 環境

- OS：Ubuntu 24.04 LTS
- 実行環境：仮想マシン
- 使用ツール：Bubblewrap（bwrap）
- セキュリティ機構：AppArmor
- 発生したエラー：`bwrap: setting up uid map: Permission denied`

## 1. 発生した問題

Bubblewrapは、LinuxのNamespaceなどの機能を利用して、プロセスを隔離された環境で実行するためのツールです。

今回は、Codex CLIを隔離された環境で実行するためにbwrapを使用していました。

しかし、サンドボックスを起動しようとすると、以下のエラーで停止しました。

```text
bwrap: setting up uid map: Permission denied
```

`uid map` は、User Namespace内のユーザーIDとホスト側のユーザーIDの対応関係を設定する仕組みです。

このエラーは、bwrapがUIDマッピングを設定できなかったことを示しています。

## 2. 原因はUbuntu 24.04のAppArmor制限

Ubuntu 24.04では、非特権ユーザーによるUser Namespaceの利用に対して、AppArmorによる追加の制限が適用されます。

そのため、Linuxカーネル側でUser Namespaceが有効になっていても、AppArmorのプロファイルによってbwrapの動作が拒否される場合があります。

実際にカーネルログを確認したところ、AppArmorによる拒否が記録されていました。

### AppArmorのログを確認する

以下のコマンドで、今回のエラーに関連するログを抽出します。

```bash
sudo journalctl -k -b --no-pager |
  grep -Ei 'apparmor="DENIED"|uid_map|bwrap'
```

実際に確認できたログの一部です。

```text
apparmor="AUDIT"
operation="userns_create"
profile="unconfined"
comm="bwrap"
target="unprivileged_userns"
```

```text
apparmor="DENIED"
operation="capable"
profile="unprivileged_userns"
comm="bwrap"
capname="setpcap"
```

```text
apparmor="DENIED"
operation="open"
profile="unprivileged_userns"
name="proc/5296/uid_map"
requested_mask="wr"
denied_mask="wr"
```

重要なのは、最後の `uid_map` に対するアクセス拒否です。

bwrapがUIDマッピングの設定に必要なファイルを開こうとしたところ、AppArmorによって読み書きが拒否されていました。

つまり、単純なファイルパーミッションの問題ではなく、**AppArmorのセキュリティポリシーによる制限**だったわけです。

## 3. 解決方法：bwrap専用のAppArmorプロファイルを適用する

今回、以下の手順で問題が解決しました。

### 手順① 必要なパッケージをインストールする

まず、パッケージ一覧を更新し、AppArmor関連のパッケージをインストールします。

```bash
sudo apt update
sudo apt install apparmor-profiles apparmor-utils
```

`apparmor-profiles` は追加のAppArmorプロファイルを提供するパッケージです。

`apparmor-utils` はAppArmorの設定や状態確認に使用する管理ツールを提供します。

### 手順② bwrap用プロファイルを配置する

次に、bwrap用のAppArmorプロファイルを `/etc/apparmor.d/` にコピーします。

```bash
sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict
```

このコマンドでは、次の処理を行っています。

- `/usr/share/apparmor/extra-profiles/` に用意されているbwrap用プロファイルを使用
- `/etc/apparmor.d/` に配置して管理対象にする
- `-m 0644` でファイルのアクセス権限を設定

AppArmorのUser Namespace制限をシステム全体で無効化するのではなく、bwrap向けの専用プロファイルを使用する点が重要です。

※ 実行前に、コピー先に既存のbwrap用プロファイルがないことを確認してください。`install` は同名ファイルを上書きします。

### 手順③ AppArmorプロファイルを読み込む

配置したプロファイルをAppArmorに読み込ませます。

```bash
sudo apparmor_parser -r \
  /etc/apparmor.d/bwrap-userns-restrict
```

`apparmor_parser` は、AppArmorプロファイルを解析してカーネルへロードするためのコマンドです。

`-r` は既存のプロファイルを置き換えて読み込むオプションです。

これでbwrapに対応するAppArmorプロファイルが有効化されます。

## 4. 動作確認

設定後、bwrapの基本的な動作を確認します。

```bash
bwrap \
  --ro-bind / / \
  --dev /dev \
  --proc /proc \
  --unshare-user \
  --unshare-pid \
  -- /bin/sh -c 'id; echo "bwrap OK"'
```

正常に動作すれば、次のように表示されます。

```text
uid=1000(...) gid=1000(...) groups=...
bwrap OK
```

UIDやGIDなどの表示は実行環境によって異なります。

今回の環境では、AppArmorプロファイルを導入したことで、元のbwrapコマンドでも `setting up uid map: Permission denied` が発生しなくなりました。

## 5. なぜUser Namespaceの制限を無効化しないのか

インターネット上では、この問題への対処として以下のコマンドが紹介されることがあります。

```bash
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

これは、非特権User Namespaceに対するAppArmorの追加制限を、システム全体で無効化する設定です。

確かに、この方法でbwrapが動作する場合もあります。

しかし、bwrapを動かすためだけに、システム全体のセキュリティ制限を緩和する必要はありません。

今回のようにbwrap専用のAppArmorプロファイルを導入することで、システム全体の制限を維持したまま対応できます。

なお、専用プロファイルの適用もbwrapに対する権限の許可を伴うため、完全にリスクがなくなるわけではありません。使用するプロファイルの内容を確認し、必要な範囲で権限を与えることが重要です。

## 6. まとめ

Ubuntu 24.04で以下のエラーが発生した場合、

```text
bwrap: setting up uid map: Permission denied
```

AppArmorによるUser Namespace制限が原因となっている可能性があります。

まずはカーネルログで `unprivileged_userns` や `uid_map` に対する拒否が発生していないかを確認しましょう。

今回の環境では、以下のコマンドで解決できました。

```bash
sudo apt update
sudo apt install apparmor-profiles apparmor-utils

sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict

sudo apparmor_parser -r \
  /etc/apparmor.d/bwrap-userns-restrict
```

**ポイントは、AppArmorの制限をシステム全体で無効化するのではなく、bwrap専用プロファイルを適用することです。**

Ubuntu 24.04上でBubblewrapを使用する際に、同じエラーで困っている方の参考になれば幸いです。
