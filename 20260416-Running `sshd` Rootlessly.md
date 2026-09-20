---
pubDatetime: 2026-04-16T14:38:37+09:00
title: "rootless での `sshd` の実行"
description: "この記事では、root 権限なしで通常ユーザーとして sshd を実行する方法を説明します。この構成では、ユーザーのホームディレクトリ配下にカスタム設定ディレクトリを作成し、非特権ポートで待ち受けます。 概要 通常、sshd はシステムによって root として起動され、ポート 22 で待ち受けます…"
---

この記事では、root 権限なしで通常ユーザーとして `sshd` を実行する方法を説明します。この構成では、ユーザーのホームディレクトリ配下にカスタム設定ディレクトリを作成し、非特権ポートで待ち受けます。

## 概要

通常、`sshd` はシステムによって root として起動され、ポート `22` で待ち受けます。しかし、環境によっては root アクセスなしで SSH サーバーを実行することが有用な場合があります。ルートレス構成は、テスト、開発、またはユーザー空間での一時的なリモートアクセスに利用できます。

この例では、`~/.sshd` にプライベートな SSH サーバー環境を作成し、必要に応じてホストキーを生成し、最小限の `sshd_config` を書き込み、設定を検証したうえで、デーモンをフォアグラウンドで起動します。

## ディレクトリと変数の設定

`setup.sh` スクリプトは、いくつかの変数を定義することから始まります：

```bash
BASE="$HOME/.sshd"
PORT=2222
SSHD="$(command -v sshd)"
```

`BASE` は SSH サーバーファイルの作業ディレクトリです。`PORT` は `2222` に設定されており、これは非特権ポートで、root なしで使用できます。`SSHD` は `sshd` 実行ファイルのパスを保持します。

次に、必要なディレクトリを作成します：

```bash
install -d -m 700 "$BASE" "$HOME/.ssh"
```

これにより、`~/.sshd` と `~/.ssh` の両方が安全なパーミッションで存在することが保証されます。

## ホストキーの生成

SSH サーバーには独自のホストキーが必要です。スクリプトは、Ed25519 ホストキーが既に存在するかを確認します：

```bash
if [ ! -f "$BASE/ssh_host_ed25519_key" ]; then
  ssh-keygen -q -t ed25519 -N '' -f "$BASE/ssh_host_ed25519_key"
fi
chmod 600 "$BASE/ssh_host_ed25519_key"
```

キーが存在しない場合は生成されます。その後、パーミッションは所有者のみに制限されます。これは、アクセス権が広すぎる秘密鍵を SSH が拒否するため重要です。

## `sshd` 設定の作成

スクリプトは、ユーザー所有ディレクトリにカスタムの `sshd_config` ファイルを書き込みます。

```bash
cat > "$BASE/sshd_config" <<EOF
Port $PORT
ListenAddress 0.0.0.0

UsePAM no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes

HostKey $BASE/ssh_host_ed25519_key
PidFile none

PermitRootLogin no
PrintMotd no
PrintLastLog no
X11Forwarding no
AllowUsers $USER

Subsystem sftp internal-sftp
EOF
```

### 主な設定項目

#### ポートとリッスンアドレス

* `Port $PORT` はサーバーがポート `2222` で待ち受けることを指定します。
* `ListenAddress 0.0.0.0` はすべてのネットワークインターフェースで接続を受け付けます。

#### 認証

* `UsePAM no` は PAM を無効化します（通常はシステムレベルの設定が必要なため）。
* `PasswordAuthentication no` はパスワードログインを無効化します。
* `KbdInteractiveAuthentication no` はキーボードインタラクティブ認証を無効化します。
* `PubkeyAuthentication yes` は公開鍵認証を有効化します。

これにより、SSH キーのみに依存したシンプルで安全なルートレス構成になります。

#### ホストキーと PID ファイル

* `HostKey $BASE/ssh_host_ed25519_key` は先ほど生成したホストキーを指定します。
* `PidFile none` は PID ファイルの書き込みを行わず、軽量なユーザー空間実行に適しています。

#### 制限

* `PermitRootLogin no` は root ログインを禁止します。
* `PrintMotd no` と `PrintLastLog no` はログイン時の追加メッセージを抑制します。
* `X11Forwarding no` は X11 フォワーディングを無効化します。
* `AllowUsers $USER` はアクセスを現在のユーザーのみに制限します。

#### SFTP サポート

* `Subsystem sftp internal-sftp` は組み込みサブシステムを使用して SFTP を有効化します。

## 設定の検証

`setup.sh` の最後で、スクリプトは設定を検証します：

```bash
"$SSHD" -t -f "$BASE/sshd_config"
```

これはサーバー起動前に構文をチェックする有用なステップです。

## SSH サーバーの起動

`start.sh` スクリプトは非常にシンプルです：

```bash
#!/bin/bash
BASE="$HOME/.sshd"
SSHD="$(command -v sshd)"
exec "$SSHD" -D -e -f "$BASE/sshd_config"
```

### これが行うこと

* `-D` は `sshd` をフォアグラウンドで実行します。
* `-e` はログ出力を標準エラーに送ります。
* `-f "$BASE/sshd_config"` はカスタム設定ファイルを使用することを指定します。

`exec` を使うことで、シェルプロセスが `sshd` に置き換えられ、クリーンにサーバーを起動できます。

## 使用方法

一般的な手順は次のとおりです：

1. `setup.sh` を一度実行して設定とホストキーを作成する。
2. 公開鍵が `~/.ssh/authorized_keys` に存在することを確認する。
3. `start.sh` を実行して SSH サーバーを起動する。
4. ポート `2222` で接続する。

例：

```bash
ssh -p 2222 user@host
```

## このアプローチの利点

このルートレス SSH 構成にはいくつかの利点があります：

* システム全体の設定を必要としない。
* 特権ポートを使用しない。
* 個人環境や一時的なセットアップに適している。
* SSH サーバー関連のすべてのファイルをユーザーのホームディレクトリ配下に保持できる。

## まとめ

この例は、root 権限なしで `sshd` を実行するためのコンパクトな方法を提供します。プライベートな設定ディレクトリ、非特権ポート、および公開鍵認証のみを使用することで、ユーザー空間で動作するシンプルで実用的な SSH サーバーを構築できます。

## スクリプト

setup.sh

```bash
#!/bin/bash 
BASE="$HOME/.sshd" 
PORT=2222 
SSHD="$(command -v sshd)" 
 
install -d -m 700 "$BASE" "$HOME/.ssh" 
 
# server host key 
if [ ! -f "$BASE/ssh_host_ed25519_key" ]; then 
  ssh-keygen -q -t ed25519 -N '' -f "$BASE/ssh_host_ed25519_key" 
fi 
chmod 600 "$BASE/ssh_host_ed25519_key" 
 
cat > "$BASE/sshd_config" <<EOF 
Port $PORT 
ListenAddress 0.0.0.0 
 
UsePAM no 
PasswordAuthentication no 
KbdInteractiveAuthentication no 
PubkeyAuthentication yes 
 
HostKey $BASE/ssh_host_ed25519_key 
PidFile none 
 
PermitRootLogin no 
PrintMotd no 
PrintLastLog no 
X11Forwarding no 
AllowUsers $USER 
 
Subsystem sftp internal-sftp 
EOF 
 
"$SSHD" -t -f "$BASE/sshd_config" 
```

start.sh

```bash
#!/bin/bash 
BASE="$HOME/.sshd" 
SSHD="$(command -v sshd)" 
exec "$SSHD" -D -e -f "$BASE/sshd_config" 
```
