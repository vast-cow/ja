---
pubDatetime: 2026-04-24T10:52:32+09:00
title: "`sudoers` を編集して `sudo -e` を通す方法と、Emacs TRAMP `/sudo::` で編集する方法"
description: "背景: なぜ sudo -e が止めるのか、方法1: sudoers を編集して sudo -e の制限を緩める、emacsclient を sudoedit から使うを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

`sudo -e` は便利ですが、最近の `sudo` では安全策として制限が強めです。とくに、**呼び出しユーザーが書き込み可能なディレクトリ配下のファイルは `sudoedit` で拒否される**ことがあります。これは `sudoers` の `sudoedit_checkdir` が既定で有効だからです。`sudoedit` は一時ファイル経由で編集し、`SUDO_EDITOR` / `VISUAL` / `EDITOR` を使ってエディタを起動します。 ([man7.org][1])

この記事では、次の 2 つを整理します。

1. `sudoers` を調整して `sudo -e` を使い続ける方法
2. Emacs の TRAMP `/sudo::` を使って、ユーザー権限の `emacs -daemon` + `emacsclient` で編集する方法

---

## 背景: なぜ `sudo -e` が止めるのか

`sudoedit` は、対象ファイルを直接 root でエディタに渡すのではなく、**一時コピーを作ってユーザー権限のエディタで編集し、最後に元へ戻す**仕組みです。そのうえで、追加の保護として次の制限があります。

* シンボリックリンクを既定では開かない
* パス途中にユーザー書き込み可能ディレクトリがある場合はリンク追跡を拒否する
* **ユーザー書き込み可能ディレクトリ内のファイル編集を拒否する**

これらは `sudoedit` の安全策として documented されています。 ([man7.org][1])

---

## 方法1: `sudoers` を編集して `sudo -e` の制限を緩める

### 何を変えるのか

対象は `sudoers` の `sudoedit_checkdir` です。これが有効だと、`sudoedit` は**パス中のディレクトリの書き込み可否を検査し、ユーザーが書き込めるディレクトリ内のファイル編集を拒否**します。既定値は **on** です。 ([Ubuntu Manpages][2])

### 編集手順

`sudoers` は直接編集せず、`visudo` を使います。`sudo` の公式 man page でも、構文エラーを避けるため `visudo` の利用が推奨されています。 ([man7.org][1])

全体に適用するなら:

```sudoers
Defaults !sudoedit_checkdir
```

ユーザー限定にするなら:

```sudoers
Defaults:yourname !sudoedit_checkdir
```

分離ファイルにするなら:

```bash
sudo visudo -f /etc/sudoers.d/sudoedit
```

中身:

```sudoers
Defaults:yourname !sudoedit_checkdir
```

### 併せて知っておくべき設定

シンボリックリンクの扱いは別設定の `sudoedit_follow` です。既定では off で、必要なときだけ明示的に有効化する設計です。 ([Ubuntu Manpages][2])

### 利点

* 既存の `sudoedit` ワークフローを維持できる
* `SUDO_EDITOR=emacsclient` と組み合わせやすい
* 一時ファイル経由という `sudoedit` 本来の流儀を保てる

### 欠点

* 安全策を弱める
* マシン全体、または対象ユーザーの `sudo` ポリシーを変更する必要がある
* 管理者権限が必要

---

## `emacsclient` を `sudoedit` から使う

`sudoedit` はエディタとして `SUDO_EDITOR`、次いで `VISUAL`、`EDITOR` を参照します。したがって、ユーザー権限で起動した Emacs デーモンをそのまま使えます。 ([man7.org][1])

GUI フレームを開くなら:

```bash
export SUDO_EDITOR='emacsclient -c'
sudoedit /etc/hosts
```

端末内で開くなら:

```bash
export SUDO_EDITOR='emacsclient -t'
sudoedit /etc/hosts
```

この方式は、**`sudoedit` の制約はそのまま受ける**点が重要です。つまり、`sudoedit_checkdir` に引っかかるなら、`SUDO_EDITOR` を `emacsclient` にしても解決しません。根本の判定は `sudoers` 側だからです。 ([man7.org][1])

---

## 方法2: Emacs TRAMP の `/sudo::` で編集する

もうひとつの実践的な方法が、Emacs の TRAMP を使うやり方です。TRAMP の `sudo` メソッドは `sudo` を使って別ユーザーとしてシェルを開始し、適切な権限でファイルにアクセスします。TRAMP マニュアルでは、`sudo` メソッドは `sudo` を使い、「shell を開始する十分な権限が必要」と説明されています。 ([gnu.org][3])

たとえば:

```bash
emacsclient /sudo::/etc/hosts
```

あるいは Emacs 内で:

```text
C-x C-f /sudo::/etc/hosts
```

さらに TRAMP には、今のバッファや Dired 項目を `sudo` 付きで開き直す支援コマンドもあります。`tramp-revert-buffer-with-sudo` と `tramp-dired-find-file-with-sudo` が用意されており、既定メソッドは `sudo` です。 ([gnu.org][3])

### 利点

* `sudoers` を変えなくてよい
* ユーザー権限の `emacs -daemon` をそのまま使える
* `emacsclient` から自然に扱える
* `sudoedit_checkdir` のような **`sudoedit` 固有の制約に縛られない**

### 欠点

* これは `sudoedit` ではなく、`sudo` メソッドでの編集
* `sudoers` 全体の許可ルールを回避するわけではない
* root で開いたバッファとして扱うので、`sudoedit` と同じ安全モデルではない

---

## `/sudo::` は `sudoedit` の代替になるのか

実務上は、かなり代替になります。理由は単純で、`sudoedit_checkdir` は **`sudoedit` 専用フラグ**だからです。`sudoers(5)` でも、その設定は `sudoedit` が書き込み可能ディレクトリや経路上のリンクをどう扱うか、という文脈で定義されています。 ([Ubuntu Manpages][2])

一方で TRAMP の `/sudo::` は `sudoedit` ではなく `sudo` メソッドです。したがって、`sudoedit` 固有の拒否に悩んでいるなら、`/sudo::` は現実的な逃げ道になります。TRAMP 公式にも `sudo` メソッドと `sudoedit` メソッドは明確に分けて記述されています。 ([gnu.org][3])

---

## では `/sudoedit::` はどうか

TRAMP には `sudoedit` メソッドもあります。マニュアルでは、これは **TRAMP による `sudoedit` 実装**であり、`sudo` メソッドとは異なり、各操作を単発の `sudo ...` で実行して、Emacs 背後にセッションを残さないようにしている、と説明されています。目的は「可能な限り安全に編集すること」で、外部プロセスは実装されません。 ([gnu.org][4])

つまり、**`sudoedit` 的な安全モデルを保ちたいなら `/sudoedit::`**、**制約を避けて編集したいなら `/sudo::`** という使い分けになります。

---

## 相対パスは使えるか

TRAMP では相対パスも使えますが、基準は Emacs の `default-directory` です。実用上は、先にディレクトリを `sudo` 付きで開いておくのが簡単です。

```text
C-x C-f /sudo::/etc/
```

その後で:

```text
C-x C-f hosts
```

のように相対指定できます。普段の操作では、Dired から `@` で `sudo` 付きに切り替える流れも扱いやすいです。TRAMP には Dired/バッファを `sudo` で開き直す専用コマンドが用意されています。 ([gnu.org][3])

---

## どちらを選ぶべきか

### `sudoers` を編集するべきケース

* `sudo -e` のワークフローを維持したい
* 一時ファイル経由の `sudoedit` を使い続けたい
* 管理者として、その制限を緩める判断ができる

この場合は `visudo` で `sudoedit_checkdir` を調整するのが筋です。 ([man7.org][1])

### TRAMP `/sudo::` を選ぶべきケース

* ユーザー権限の `emacs -daemon` を使いたい
* `emacsclient` で自然に編集したい
* `sudoedit_checkdir` に引っかかるが、`sudoers` は触りたくない

この場合は `/sudo::` が最も実用的です。TRAMP 側でも `sudo` メソッドを使うための支援コマンドが整備されています。 ([gnu.org][3])

---

## 最小構成のおすすめ

運用としては、まずこれで十分です。

```bash
alias ec='emacsclient -c'
```

必要なときだけ:

```bash
ec /sudo::/etc/hosts
```

`sudoedit` を残したいなら:

```bash
export SUDO_EDITOR='emacsclient -c'
```

そのうえで、本当に必要な場合だけ `sudoers` の `sudoedit_checkdir` を見直します。`sudoedit` は既定で保護寄り、TRAMP `/sudo::` は運用寄り、という整理にしておくと迷いません。 ([man7.org][1])

---

## まとめ

`sudo -e` で困る原因が `sudoedit_checkdir` なら、解決策は 2 つです。

* **管理側で直す**: `visudo` で `sudoedit_checkdir` を無効化する
* **編集側で回避する**: Emacs TRAMP の `/sudo::` を使う

前者は `sudoedit` の流儀を保ち、後者は Emacs 運用に自然に馴染みます。`emacs -daemon` と `emacsclient` を中心に使うなら、多くの環境では **TRAMP `/sudo::` のほうが実践的**です。一方で、組織や端末のポリシーとして `sudoedit` を標準にしたいなら、`sudoers` の明示的な管理が必要です。 ([man7.org][1])

[1]: https://man7.org/linux/man-pages/man8/sudoedit.8.html "sudoedit(8) - Linux manual page"
[2]: https://manpages.ubuntu.com/manpages/noble/man5/sudoers.5.html "Ubuntu Manpage: sudoers - default sudo security policy plugin"
[3]: https://www.gnu.org/software/tramp/ "TRAMP 2.8.1 User Manual"
[4]: https://www.gnu.org/software/emacs/manual/html_node/tramp/External-methods.html "External methods (TRAMP 2.7.3.30.2 User Manual)"
