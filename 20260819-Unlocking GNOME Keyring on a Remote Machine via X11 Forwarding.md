---
title: "リモートマシンのGNOME KeyringをX11 ForwardingでUnlockする"
description: "GNOME KeyringをUnlockする、SSH agentを使う、なぜ dbus-update-activation-environment が必要なのかを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
pubDatetime: 2026-08-19T09:42:29.761Z
updatedDate: 2026-08-19T10:54:07.822Z
---

SSHで接続したLinuxサーバー上のGNOME Keyringを、X11 Forwarding経由でSeahorseを使ってUnlockしたときのメモ。

あわせて、`secret-tool`でUnlockする方法と、GNOME Keyring / GCRが提供するSSH agentをSSHセッションから利用する方法も記載する。

## 環境

SSHのX11 Forwardingを使い、リモート側で起動したSeahorseやGNOME Keyringのパスワードプロンプトをローカル側に表示する。

まずSSH接続する。

```bash
ssh -XY server
```

接続後、X11用の環境変数をD-Bus / systemd user session側にも反映する。

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

## GNOME KeyringをUnlockする

### Seahorseを使う方法

Seahorseを起動する。

```bash
seahorse
```

Seahorseが表示されたら、

**Passwords → Login → Unlock**

を選択し、GNOME Keyringのパスワードを入力する。

これでLogin keyringをUnlockできる。

### `secret-tool`を使う方法

Seahorseを起動せず、`secret-tool`からUnlockを要求することもできる。

```bash
secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null
```

Keyringがロックされている場合はUnlock用のパスワードプロンプトが表示されるので、そこでパスワードを入力する。

検索結果そのものは不要なので、標準出力は`/dev/null`へ捨てている。

X11 Forwarding経由でプロンプトを表示するため、先に以下を実行しておく。

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

したがって、GUIのSeahorseを開く必要がなければ、次の手順だけでもよい。

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null
```

## SSH agentを使う

GNOME Keyring / GCR側のSSH agentを利用する場合は、`SSH_AUTH_SOCK`をGCRのソケットに向ける。

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
```

設定後、agentに認識されている鍵を確認する。

```bash
ssh-add -l
```

一連の操作は例えば以下になる。

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null

export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"

ssh-add -l
```

Seahorseを使う場合は、Unlock部分を次のように置き換える。

```bash
seahorse
# Passwords → Login → Unlock
```

毎回GCRのSSH agentを使う場合は、利用しているshellの設定ファイルなどに次を追加しておいてもよい。

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
```

ただし、別の`ssh-agent`やagent forwardingを併用している場合は、既存の`SSH_AUTH_SOCK`を上書きすることになるため注意する。

## なぜ `dbus-update-activation-environment` が必要なのか

SSHのX11 Forwardingを使うと、SSHセッションには例えば以下のような`DISPLAY`が設定される。

```text
localhost:10.0
```

一方、GNOME Keyring周辺のGUIプロンプトはD-Busやsystemdのuser session経由で起動されることがある。

そのため、SSHシェル上では`DISPLAY`が正しく設定されていても、D-Bus経由で起動されたプロセス側ではSSHのX11 displayを認識できない場合がある。

そこで、

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

を実行して、現在のSSHセッションの`DISPLAY`と`XAUTHORITY`をactivation environment側にも渡しておく。

## 手順まとめ

Seahorseを使う場合。

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

seahorse
# Passwords → Login → Unlock

export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"

ssh-add -l
```

`secret-tool`を使う場合。

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null

export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"

ssh-add -l
```

## 不具合時のメモ

GNOME Keyring daemonが動いているか確認する。

```bash
pgrep -af gnome-keyring-daemon
```

挙動がおかしい場合は、user serviceを再起動する。

```bash
systemctl --user stop gnome-keyring-daemon.service
systemctl --user start gnome-keyring-daemon.service
```

ログをリアルタイムで確認する場合は以下。

```bash
journalctl --user-unit gnome-keyring-daemon -fe
```

別ターミナルでこのログを流しながらSeahorseや`secret-tool`を操作すると、daemon側で何が起きているか確認しやすい。

SSH agent側の確認には以下も使える。

```bash
echo "$SSH_AUTH_SOCK"
ls -l "$XDG_RUNTIME_DIR/gcr/ssh"
ssh-add -l
```

期待するソケットは以下。

```text
$XDG_RUNTIME_DIR/gcr/ssh
```

## 補足

`ssh -X`は通常のX11 Forwarding、`ssh -Y`はtrusted X11 Forwardingになる。

今回は、

```bash
ssh -XY server
```

で動作を確認した。

trusted X11 Forwardingは通常の`-X`よりリモートアプリケーションに広い権限を与えるため、信頼できるサーバーに対して使用するのが前提。

また、

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

は、そのユーザーのD-Bus / systemd user sessionのactivation environmentを更新する。

同じユーザーでローカルのGUIセッションも並行して利用している環境では、GUIアプリケーションの表示先に影響する可能性があるため注意する。

同様に、

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
```

は現在のshellから利用するSSH agentをGCR側に切り替える設定になる。

すでにOpenSSHの`ssh-agent`やSSH agent forwardingを利用している場合、そのソケットではなくGCR側のagentを利用することになる。

## 結論

SSH越しにリモートマシンのGNOME KeyringをUnlockする場合は、まず

```bash
ssh -XY server
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

とした上で、Seahorseから

**Passwords → Login → Unlock**

を実行するか、

```bash
secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null
```

でUnlockを要求する。

さらにGCRのSSH agentを利用する場合は、

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
ssh-add -l
```

を設定する。

GNOME Keyring daemon周辺で問題が起きた場合は、`systemctl --user`での再起動と`journalctl`でのログ確認が有効だった。
