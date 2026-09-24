---
pubDatetime: 2026-09-24T17:17:0+09:00
title: "tmuxの中からOSC 52で外側ターミナルのクリップボードへ文字列を送る"
description: "tmux 内から OSC 52 を使って、外側のターミナルのクリップボードへ文字列を送る方法を解説します。`set-clipboard` を使う方法と、`allow-passthrough` で OSC 52 を直接外側へ通す方法の違いと設定方法を紹介します。"
---

tmux の中で動いているプログラムから、tmux を表示している外側のターミナルエミュレータへ文字列を送り、そのターミナルのクリップボードを書き換えたいことがあります。

この用途には **OSC 52** が使えます。

ただし tmux を挟む場合、単に OSC 52 を出力すれば常に外側へ届くわけではありません。大きく分けて、次の2通りがあります。

1. tmux 自身に OSC 52 を処理させる
2. tmux の passthrough を使って OSC 52 をそのまま外へ通す

通常は、まず1つ目の `set-clipboard` を使う方法を検討するのがよいです。

## OSC 52とは

OSC 52 は、端末エミュレータに対してクリップボードの読み書きを指示する escape sequence です。

クリップボードへ文字列を書き込む場合、概念的には次の形式になります。

```text
OSC 52 ; c ; BASE64_ENCODED_TEXT BEL
```

実際の制御文字で書くと、例えば `hello` を送る場合は次のようになります。

```sh
printf '\033]52;c;%s\a' "$(printf %s hello | base64 | tr -d '\n')"
```

OSC 52 のデータ部分は Base64 で渡します。

外側のターミナルエミュレータが OSC 52 に対応し、かつクリップボード書き込みを許可していれば、これだけでクリップボードへ `hello` が入ります。

問題は tmux の中にいる場合です。

# 方法1: tmuxの `set-clipboard` を使う

tmux には、内側のアプリケーションが送った OSC 52 を受け取り、外側のターミナルへ適切に転送する仕組みがあります。

設定は次の通りです。

```tmux
set -s set-clipboard on
```

この設定なら、tmux の中のプログラムは特別なラッピングをせず、普通の OSC 52 を送れます。

```sh
printf '\033]52;c;%s\a' "$(printf %s hello | base64 | tr -d '\n')"
```

tmux がこの OSC 52 を認識し、自身の paste buffer を更新したうえで、外側のターミナルへクリップボード設定用のシーケンスを送ります。

つまり、

```text
アプリ
  ↓ OSC 52
tmux
  ↓ OSC 52
外側ターミナル
  ↓
OSのクリップボード
```

という流れです。

## `Ms` capabilityを確認する

tmux が外側ターミナルへクリップボード設定シーケンスを送るには、term capability の `Ms` が使われます。

確認できます。

```sh
tmux info | grep 'Ms:'
```

例えば次のような出力なら、OSC 52 用 capability が認識されています。

```text
Ms: (string) \033]52;%p1%s;%p2%s\a
```

もし認識されていない場合、tmux 3.2 以降では `terminal-features` で clipboard capability を明示できます。

例えば xterm 系なら、

```tmux
set -as terminal-features ',xterm*:clipboard'
```

のように設定できます。

実際には使用している `$TERM` やターミナルエミュレータに応じて対象を調整します。

## `set-clipboard on` と `external`

tmux の `set-clipboard` には複数のモードがあります。

`on` の場合、tmux 自身の copy-mode だけでなく、tmux 内部のアプリケーションから送られた OSC 52 も受け付けます。

そのため、

```tmux
set -s set-clipboard on
```

としておけば、Neovim や Emacs、自作ツールなどが普通の OSC 52 を出力するだけで、外側のターミナルのクリップボードへ届く構成にできます。

「tmux 内の任意アプリから OSC 52 を送りたい」という用途では、この方法が最も自然です。

# 方法2: tmux passthroughでOSC 52を直接外へ通す

もう1つは、tmux に OSC 52 を解釈させず、escape sequence そのものを外側のターミナルへ通す方法です。

tmux 3.3 以降では、まず passthrough を許可します。

```tmux
set -g allow-passthrough on
```

または、

```tmux
set -g allow-passthrough all
```

とします。

そのうえで、外側へ送りたい OSC 52 を tmux 用 DCS sequence で包みます。

例えば `hello` をクリップボードへ送る場合です。

```sh
data=$(printf %s 'hello' | base64 | tr -d '\n')
printf '\033Ptmux;\033\033]52;c;%s\007\033\\' "$data"
```

構造としては、

```text
ESC P tmux;
    ESC ESC ]52;c;BASE64 BEL
ESC \
```

となっています。

ポイントは、passthrough 内部に含まれる `ESC` を `ESC ESC` と2個重ねることです。

つまり、本来外側のターミナルへ送りたい

```text
ESC ] 52 ; c ; ... BEL
```

を、

```text
ESC P tmux;
ESC ESC ] 52 ; c ; ... BEL
ESC \
```

で包んでいます。

この方法では tmux は OSC 52 の意味を処理せず、内側の escape sequence を外側のターミナルへ通します。

## シェル関数にする

毎回 escape sequence を書くのは面倒なので、関数にすると便利です。

引数版なら次のようにできます。

```sh
osc52_copy() {
    local b64
    b64=$(printf %s "$1" | base64 | tr -d '\n')
    printf '\033Ptmux;\033\033]52;c;%s\007\033\\' "$b64"
}
```

使い方は、

```sh
osc52_copy 'clipboard text'
```

です。

stdin から受け取りたい場合は、

```sh
osc52_copy() {
    local b64
    b64=$(base64 | tr -d '\n')
    printf '\033Ptmux;\033\033]52;c;%s\007\033\\' "$b64"
}
```

として、

```sh
printf 'clipboard text' | osc52_copy
```

のように使えます。

# tmuxのbufferを経由する方法

OSC 52 を自分で組み立てず、tmux の buffer を使う方法もあります。

例えば、

```sh
printf %s hello | tmux load-buffer -
```

で tmux の paste buffer に文字列を読み込めます。

tmux の clipboard integration を有効にしている構成では、copy-mode や buffer 操作と組み合わせて、外側のクリップボードへ送信できます。

tmux 自身の copy-mode からシステムクリップボードへコピーしたいだけなら、この系統の機能を使うほうが自然です。

# どの方法を使うべきか

整理すると次のようになります。

| 方法                 | tmux内から送るもの         | `allow-passthrough` |
| ------------------ | ------------------- | ------------------- |
| `set-clipboard on` | 通常の OSC 52          | 不要                  |
| DCS passthrough    | tmux DCSで包んだ OSC 52 | 必要                  |
| tmux buffer        | `load-buffer` 等     | 不要                  |

一般的には、

```tmux
set -s set-clipboard on
```

を設定し、アプリケーション側から普通の OSC 52 を出す方法が扱いやすいです。

```sh
printf '\033]52;c;%s\a' "$(printf %s hello | base64 | tr -d '\n')"
```

一方、

* tmux に OSC 52 を解釈させたくない
* OSC 52 以外の escape sequence も外側へ通したい
* 端末固有の制御シーケンスを直接送信したい

といった場合は、`allow-passthrough` と DCS wrapping を使う方法が向いています。

```sh
printf '\033Ptmux;\033\033]52;c;%s\007\033\\' "$base64_data"
```

# 注意点

どちらの方法でも、最終的には **外側のターミナルエミュレータ自身が OSC 52 に対応している必要があります**。

また、セキュリティ上の理由から OSC 52 によるクリップボード書き込みを無効化していたり、確認を要求したりするターミナルもあります。

したがって動かない場合は、

1. 外側ターミナルが OSC 52 に対応しているか
2. OSC 52 clipboard write が許可されているか
3. tmux の `set-clipboard` が有効か
4. `tmux info` で `Ms` が認識されているか
5. passthrough を使う場合は `allow-passthrough` が有効か

を順に確認すると切り分けやすくなります。

## まとめ

tmux 内から外側のターミナルのクリップボードへ文字列を送りたい場合、第一候補はこれです。

```tmux
set -s set-clipboard on
```

そして tmux 内から普通の OSC 52 を送ります。

```sh
printf '\033]52;c;%s\a' "$(printf %s hello | base64 | tr -d '\n')"
```

tmux を完全に素通ししたい場合は、

```tmux
set -g allow-passthrough on
```

として、

```sh
printf '\033Ptmux;\033\033]52;c;%s\007\033\\' "$base64_data"
```

のように DCS passthrough を使います。

「tmux が OSC 52 を仲介する」のか、「OSC 52 自体を tmux の外へ貫通させる」のか。この違いを押さえておくと、tmux 越しのクリップボード連携を整理しやすくなります。
