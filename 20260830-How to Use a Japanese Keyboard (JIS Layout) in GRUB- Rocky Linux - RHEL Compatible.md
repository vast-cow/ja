---
title: "GRUBで日本語キーボード（JIS配列）を使う方法：Rocky Linux / RHEL系対応"
description: ""
pubDatetime: 2026-08-30T07:46:21.679Z
---

GRUBのメニューやコマンドラインを操作していると、キーボードがUS配列として扱われて困ることがあります。

特に日本語キーボードでは、

* `@`
* `:`
* `"`
* `\`
* `_`

などの記号位置がUS配列とJIS配列で異なるため、GRUB上でカーネルパラメータを編集したり、コマンドを入力したりする際にかなり不便です。

この記事では、Rocky Linux、AlmaLinux、RHEL、CentOS StreamなどのRHEL系Linuxで、GRUBのキーボードレイアウトを日本語JIS配列に変更する方法をまとめます。

なお、ここでいう「日本語キーボード」は日本語入力IMEのことではありません。GRUB上で日本語文字を入力できるようにするものではなく、**物理的なJISキーボードのキー配置に合わせるための設定**です。

---

## Linuxのキーボード設定とGRUBのキーボード設定は別物

通常、Linux側のコンソールキーマップは次のようなコマンドで変更できます。

```bash
localectl set-keymap jp
```

あるいはカーネルパラメータや設定ファイルで、

```text
vconsole.keymap=jp
```

のように指定することもあります。

しかし、これらはLinuxカーネルが起動した後の設定です。

GRUBはLinuxより前に動作しているため、Linux側のキーボード設定はGRUBには反映されません。

つまり、

```text
GRUB
  ↓
Linux kernel
  ↓
systemd / console
```

という起動順序において、GRUB用のキーボードレイアウトはGRUB自身に設定する必要があります。

---

# Rocky Linux / RHEL系で必要なパッケージ

Rocky LinuxやRHEL系では、GRUB関連コマンドの名前に `grub2-` が付いています。

必要なパッケージをインストールします。

```bash
sudo dnf install grub2-tools-extra kbd
```

主に使用するのは次のコマンドです。

```text
grub2-kbdcomp
grub2-mkconfig
```

`grub2-kbdcomp` を使って、GRUBが読み込めるキーボードレイアウトファイルを作成します。

---

# GRUB用の日本語キーマップを作成する

まず保存先を作ります。

```bash
sudo mkdir -p /boot/grub2/layouts
```

続いて、日本語JIS配列のキーマップを作成します。

```bash
sudo grub2-kbdcomp -o /boot/grub2/layouts/jp.gkb jp
```

生成できたか確認します。

```bash
ls -l /boot/grub2/layouts/jp.gkb
```

例えば次のようにファイルが存在していれば準備完了です。

```text
/boot/grub2/layouts/jp.gkb
```

GRUBでは、この `.gkb` ファイルを読み込むことでキーボード配列を変更します。

---

# まずはGRUB起動中だけ一時的にJIS配列へ変更する

いきなり永続設定を変更するより、最初はGRUBのコマンドラインから一時的に読み込んで動作確認するのがおすすめです。

GRUBメニューが表示されたら、

```text
c
```

を押してGRUBコマンドラインに入ります。

次のコマンドを実行します。

```grub
insmod keylayouts
keymap jp
```

これで、そのGRUBセッション中だけ日本語JIS配列になります。

再起動すれば元に戻るため、設定テストにも向いています。

---

## `keymap jp` で読み込めない場合

GRUBが使用している `prefix` を確認します。

```grub
echo $prefix
```

例えば、

```text
(hd0,gpt2)/grub2
```

のように表示されることがあります。

次に、キーマップが見えるか確認します。

```grub
ls $prefix/layouts/
```

ここで、

```text
jp.gkb
```

が表示されれば、GRUBからファイルにアクセスできています。

その場合は通常、

```grub
keymap jp
```

で読み込めます。

環境によっては明示的にファイルを指定して確認してもよいでしょう。

```grub
keymap $prefix/layouts/jp.gkb
```

---

# 永続的にJIS配列へ変更する

動作確認できたら、GRUBの設定に組み込みます。

RHEL系では `/etc/grub.d/40_custom` に追記する方法が分かりやすいです。

```bash
sudo vi /etc/grub.d/40_custom
```

次の内容を追加します。

```grub
insmod keylayouts
keymap ${prefix}/layouts/jp.gkb
```

あるいは、環境によっては次のような指定でも動作します。

```grub
insmod keylayouts
keymap jp
```

個人的には、どのファイルを使用しているか分かりやすいので、

```grub
keymap ${prefix}/layouts/jp.gkb
```

のように明示しておく方がトラブルシュートしやすいでしょう。

---

# grub.cfgを再生成する

設定を変更したら、GRUB設定を再生成します。

Rocky Linux 9などの比較的新しいRHEL系では、

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

を使用します。

再起動します。

```bash
sudo reboot
```

GRUBメニューで `c` を押してコマンドラインを開き、記号キーを確認します。

例えば、

```text
@
:
"
\
_
```

などがキートップの刻印どおり入力できれば、JIS配列が適用されています。

---

# UEFI環境での注意点

古い記事を見ると、UEFI環境では次のようなファイルへ直接 `grub2-mkconfig` を実行する例があります。

```text
/boot/efi/EFI/redhat/grub.cfg
```

Rocky Linuxでは、

```text
/boot/efi/EFI/rocky/grub.cfg
```

となっている場合があります。

ただし、最近のRHEL系ではUEFI側の `grub.cfg` は、本体となる `/boot/grub2/grub.cfg` を読み込むためのスタブとして使用される構成があります。

そのため、Rocky Linux 9などでは基本的に、

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

を使用します。

UEFI側の `grub.cfg` を不用意に直接上書きしない方が安全です。

---

# `terminal_input at_keyboard` は必要か

GRUBのキーボード設定例では、

```grub
insmod at_keyboard
terminal_input at_keyboard
```

といった設定を見かけることがあります。

ただし、USBキーボードやUEFI環境では、入力経路を明示的に `at_keyboard` へ切り替えることで、逆にキーボード入力ができなくなる場合があります。

JIS配列へ変更するだけであれば、まずは、

```grub
insmod keylayouts
keymap ${prefix}/layouts/jp.gkb
```

だけで試すのが無難です。

特別な理由がない限り、入力端末そのものを変更する必要はありません。

---

# 一時変更と永続変更の違い

整理すると、次のようになります。

| 方法                 | 設定                                | 再起動後        |
| ------------------ | --------------------------------- | ----------- |
| GRUBコマンドライン        | `insmod keylayouts` → `keymap jp` | 元に戻る        |
| `40_custom`へ追加     | `keymap ${prefix}/layouts/jp.gkb` | 維持される       |
| Linuxの `localectl` | Linuxコンソールのみ変更                    | GRUBには影響しない |

まず一時変更でテストし、問題がなければ永続設定にするのが安全です。

---

# `.gkb`を作らずGRUBだけでJIS配列へ変更できるか

基本的にはできません。

GRUB上の `keymap` コマンドは、あらかじめ作成されたGRUB用キーマップファイルを読み込む仕組みです。

そのため、Linux起動中に、

```bash
grub2-kbdcomp
```

を使って、

```text
jp.gkb
```

を事前に作っておく必要があります。

最低限、次の準備だけはLinux側で行います。

```bash
sudo mkdir -p /boot/grub2/layouts
sudo grub2-kbdcomp -o /boot/grub2/layouts/jp.gkb jp
```

その後であれば、GRUB起動中にいつでも、

```grub
insmod keylayouts
keymap jp
```

と実行して、一時的にJIS配列へ切り替えられます。

---

# まとめ

Rocky LinuxやRHEL系でGRUBを日本語JISキーボード配列にする場合、ポイントは次の3つです。

まずGRUB用のキーマップを作ります。

```bash
sudo dnf install grub2-tools-extra kbd
sudo mkdir -p /boot/grub2/layouts
sudo grub2-kbdcomp -o /boot/grub2/layouts/jp.gkb jp
```

GRUB上で一時的に試す場合は、

```grub
insmod keylayouts
keymap jp
```

とします。

永続化する場合は `/etc/grub.d/40_custom` に、

```grub
insmod keylayouts
keymap ${prefix}/layouts/jp.gkb
```

を追加し、

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

を実行します。

GRUBはLinux起動前に動作するため、`localectl set-keymap jp` などのLinux側設定とは完全に別です。

GRUB上で頻繁にカーネルパラメータを編集したり、レスキュー操作をしたりする環境では、JISキーボードの設定を入れておくとかなり操作しやすくなります。
