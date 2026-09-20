---
pubDatetime: 2026-06-11T20:28:50+09:00
title: "`gio trash` 入門：Linux CLI でファイルを「削除」せずゴミ箱へ送る"
description: "Linux のコマンドラインでファイルを消すとき、反射的に rm を使う人は多い。しかし rm は基本的にゴミ箱を経由しない完全削除であり、操作ミスに弱い。 GNOME / GLib 系の環境では、gio trash を使うことで、ファイルやディレクトリを GUI のファイルマネージャと同じようにゴ…"
---

Linux のコマンドラインでファイルを消すとき、反射的に `rm` を使う人は多い。しかし `rm` は基本的に**ゴミ箱を経由しない完全削除**であり、操作ミスに弱い。

GNOME / GLib 系の環境では、`gio trash` を使うことで、ファイルやディレクトリを GUI のファイルマネージャと同じように**ゴミ箱へ移動**できる。`gio trash` は `gio` コマンドのサブコマンドで、Trash の一覧表示、復元、空にする操作にも対応している。Ubuntu や Arch Linux の manpage でも、`gio trash --list`、`--restore`、`--empty` などが説明されている。([Ubuntu Manpages][1])

---

## `gio trash` とは

`gio` は GLib/GIO が提供するコマンドラインツールで、ローカルファイルだけでなく、GIO が扱える URI や仮想ファイルシステムに対して操作できる。

その中の `trash` サブコマンドが `gio trash` である。

```bash
gio trash FILE...
```

これは `rm FILE` のように即座に削除するのではなく、対象を Trash、つまりゴミ箱へ送る。

GNOME Files、旧称 Nautilus、でファイルを右クリックして「ゴミ箱へ移動」する操作に近い。GVfs は GIO のためのユーザー空間仮想ファイルシステム実装で、trash、SFTP、SMB、WebDAV などのバックエンドを含む。([wiki.gnome.org][2])

---

## 基本的な使い方

### ファイルをゴミ箱へ送る

```bash
gio trash memo.txt
```

複数ファイルも指定できる。

```bash
gio trash a.txt b.txt c.txt
```

ディレクトリも対象にできる。

```bash
gio trash old-project/
```

この操作では、可能であれば元の場所に復元できるよう、ゴミ箱側にメタデータが保存される。

---

## ゴミ箱の中身を確認する

```bash
gio trash --list
```

または次のように `trash://` を一覧表示することもできる。

```bash
gio list trash://
```

Arch Linux の manpage では、ゴミ箱の確認方法として `gio trash --list` または `gio list trash://` が示されている。([Archマニュアルページ][3])

例:

```text
trash:///memo.txt	/home/user/Documents/memo.txt
trash:///old-project	/home/user/work/old-project
```

表示形式は環境やバージョンで多少異なるが、基本的には Trash 内の URI と元の場所を確認できる。

---

## ゴミ箱から復元する

復元には `--restore` を使う。

```bash
gio trash --restore trash:///memo.txt
```

重要なのは、復元対象には通常のパスではなく、`trash://` で始まる URI を指定する点である。

Ubuntu の manpage でも、`--restore` は Trash から元の場所へ復元し、指定には `trash://` で始まる URI を期待すると説明されている。元のディレクトリが存在しない場合は再作成される。([Ubuntu Manpages][1])

典型的な流れは次の通り。

```bash
gio trash --list
gio trash --restore trash:///memo.txt
```

---

## ゴミ箱を空にする

ゴミ箱を完全に空にするには次を使う。

```bash
gio trash --empty
```

これは取り消しにくい操作なので注意する。`gio trash` は安全な削除のために便利だが、`--empty` は最終削除に近い。

---

## 存在しないファイルを無視する

`-f` または `--force` を使うと、存在しないファイルや削除できないファイルを無視する。

```bash
gio trash -f maybe-exists.txt
```

Arch Linux の manpage では、`-f, --force` は存在しないファイルや trash 不可能なファイルを無視するオプションとして説明されている。([Archマニュアルページ][3])

---

## `rm` との違い

| コマンド                | 動作       | 復元    |
| ------------------- | -------- | ----- |
| `rm file`           | ファイルを削除  | 通常は困難 |
| `gio trash file`    | ゴミ箱へ移動   | 可能    |
| `gio trash --empty` | ゴミ箱を空にする | 通常は困難 |

`rm` はスクリプトや一時ファイル削除には強力だが、手作業での削除には危険な場合がある。特に次のような場面では `gio trash` のほうが安全である。

```bash
gio trash ~/Downloads/*
gio trash old-report.pdf
gio trash tmp-output/
```

ただし、`gio trash` はあくまで「ゴミ箱に送る」コマンドであり、バックアップの代替ではない。

---

## `gvfs-trash` との関係

古い記事では `gvfs-trash` というコマンドが紹介されていることがある。

```bash
gvfs-trash file.txt
```

しかし現在は `gio trash` を使うのが一般的である。Ubuntu 系の情報でも、`gvfs-trash` は deprecated であり、`gio trash` を使うよう案内される例が示されている。([Ask Ubuntu][4])

新しく設定するなら、`gvfs-trash` ではなく `gio trash` を使うべきである。

---

## `trash-cli` との違い

Linux CLI でゴミ箱を扱うツールとしては、`trash-cli` もよく使われる。

```bash
trash-put file.txt
trash-list
trash-restore
trash-empty
```

両者の使い分けは次のように考えるとよい。

| 用途                          | 向いているもの     |
| --------------------------- | ----------- |
| GNOME / GLib 系環境で標準機能を使いたい  | `gio trash` |
| デスクトップ環境に依存しにくい CLI ツールがほしい | `trash-cli` |
| 復元操作を対話的にやりたい               | `trash-cli` |
| すでに `gio` が入っている環境で軽く使いたい   | `gio trash` |

GNOME デスクトップや GLib/GVfs が入っている環境では、`gio trash` は追加インストールなしで使えることが多い。一方、サーバーや最小構成の環境では入っていない場合もある。

---

## シェル alias の例

`rm` の代わりに使いやすくするなら、別名を付けるのがよい。

```bash
alias del='gio trash'
alias trash='gio trash'
```

`~/.bashrc` や `~/.zshrc` に書く。

```bash
echo "alias del='gio trash'" >> ~/.bashrc
source ~/.bashrc
```

これで次のように使える。

```bash
del old.txt
del old-directory/
```

`rm` 自体を置き換える alias も可能ではある。

```bash
alias rm='gio trash'
```

しかしこれは推奨しにくい。スクリプト、システム管理作業、他人の端末操作で挙動の前提が崩れるからである。安全にしたいなら、`rm` はそのまま残し、`del` や `trash` のような別名を使うほうがよい。

---

## 注意点

### 1. すべてのファイルシステムで同じように動くとは限らない

Trash の場所は対象ファイルのある場所によって変わることがある。Red Hat のドキュメントでも、ホームディレクトリ配下では通常 `$XDG_DATA_HOME/Trash` が使われるが、ファイルの場所によって Trash の場所が異なる場合があり、すべてのファイルシステムがこの概念をサポートするわけではないと説明されている。([Red Hat Documentation][5])

### 2. `sudo gio trash` は避ける

root 権限で `gio trash` を実行すると、ユーザーのゴミ箱ではなく root 側の環境や権限で処理される可能性がある。通常の自分のファイルは、通常ユーザーで実行する。

```bash
gio trash file.txt
```

どうしても root 所有ファイルを消す必要がある場合は、ゴミ箱運用よりも削除ポリシーを明確にした上で `sudo rm` などを使う場面が多い。

### 3. 復元には `trash://` URI が必要

次のような指定は失敗することがある。

```bash
gio trash --restore /home/user/Documents/memo.txt
```

復元対象は `gio trash --list` で確認し、`trash://...` の形式で指定する。

```bash
gio trash --restore trash:///memo.txt
```

---

## 実用例

### Downloads 配下の不要ファイルをゴミ箱へ送る

```bash
gio trash ~/Downloads/*.zip
```

### カレントディレクトリの `.log` をまとめてゴミ箱へ送る

```bash
gio trash ./*.log
```

### ゴミ箱の中身を確認してから空にする

```bash
gio trash --list
gio trash --empty
```

### 削除用 alias を設定する

```bash
cat >> ~/.bashrc <<'EOF'
alias del='gio trash'
alias trash='gio trash'
EOF

source ~/.bashrc
```

---

## まとめ

`gio trash` は、Linux のコマンドラインからファイルを安全にゴミ箱へ送るための実用的なコマンドである。

```bash
gio trash file.txt
gio trash --list
gio trash --restore trash:///file.txt
gio trash --empty
```

`rm` は即時削除、`gio trash` はゴミ箱への移動である。この違いを理解して使い分けるだけで、CLI 作業中の誤削除リスクをかなり下げられる。

日常的な手作業では `gio trash`、明確に完全削除したい場面では `rm`、という分担にしておくのが現実的である。

[1]: https://manpages.ubuntu.com/manpages/jammy/man1/gio.1.html?utm_source=chatgpt.com "gio - GIO commandline tool"
[2]: https://wiki.gnome.org/Projects/gvfs?utm_source=chatgpt.com "Projects/gvfs"
[3]: https://man.archlinux.org/man/gio.1?utm_source=chatgpt.com "gio(1) - Arch manual pages"
[4]: https://askubuntu.com/questions/213533/command-to-move-a-file-to-trash-via-terminal?utm_source=chatgpt.com "Command to move a file to Trash via Terminal"
[5]: https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/9/html/administering_the_system_using_the_gnome_desktop_environment/available-gio-commands_managing-storage-volumes-in-gnome?utm_source=chatgpt.com "12.6. 사용 가능한 GIO 명령 | GNOME 데스크탑 환경을 ..."
