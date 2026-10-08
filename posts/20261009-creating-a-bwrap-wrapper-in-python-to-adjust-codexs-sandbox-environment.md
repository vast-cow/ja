---
title: "Codexのサンドボックス環境を調整するbwrapラッパーをPythonで作る"
description: "Python製のラッパーで、Codexが生成するBubblewrapの引数を部分変換し、ホームディレクトリをtmpfsで覆って必要なファイルだけを再公開する。"
pubDatetime: 2026-10-09T00:10:00+09:00
---

CodexをLinux環境で利用する際、Bubblewrap（`bwrap`）によるサンドボックスの構成を調整したいことがある。

ただし、Codex本体が生成する`bwrap`の引数を直接変更するのは、実装やバージョンへの依存が大きい。

そこで、`bwrap`の起動を仲介するPython製のラッパー、bwrapwrapを作成した。

このラッパーでは、Codexが指定したマウント設定の一部を書き換え、ホームディレクトリの扱いを調整する。また、実際に渡された引数や環境変数をログに記録することで、サンドボックスの挙動を確認しやすくしている。

## bwrapwrapで実現したいこと

今回のラッパーには、主に次の4つの役割を持たせた。

1. `--dev /dev`を`--dev-bind /dev /dev`に変換する
2. 作業ディレクトリのbindマウントを検出し、その直前にホームディレクトリ用のマウント設定を追加する
3. Codexの実行ファイルと関連ディレクトリをサンドボックス内から参照できるようにする
4. `bwrap`の起動引数と環境変数をログに保存する

重要なのは、Codexが生成した引数をすべて置き換えるのではなく、必要な部分だけを変更するという点だ。

それ以外の引数は基本的にそのまま実際の`bwrap`へ渡す。

## 1. ラッパーの全体コード

実装にはPython 3を使用した。

```python
#!/usr/bin/env python3

import os
import sys
import shlex
import pwd
import subprocess
from pathlib import Path
from datetime import datetime


def format_args(args: list[str]) -> str:
    return " ".join(shlex.quote(arg) for arg in args)


def main() -> None:
    os.umask(0o077)

    home = os.environ["HOME"]
    pwd_path = os.environ["PWD"]

    real_bwrap = os.environ.get("BWRAP_REAL", "/usr/bin/bwrap")
    log_file = os.environ.get(
        "BWRAP_LOG",
        f"{home}/.local/state/bwrap-wrapper.log",
    )

    Path(log_file).parent.mkdir(parents=True, exist_ok=True)

    timestamp = datetime.now().astimezone().isoformat(timespec="seconds")
    user_name = pwd.getpwuid(os.getuid()).pw_name
    pid = os.getpid()

    original_args = sys.argv[1:]
    args = []

    tmpfs_inserted = False

    i = 0

    while i < len(original_args):
        arg = original_args[i]

        # --dev /dev を変換
        if (
            arg == "--dev"
            and i + 1 < len(original_args)
            and original_args[i + 1] == "/dev"
        ):
            args.extend(["--dev-bind", "/dev", "/dev"])
            i += 2
            continue

        # --bind の変換
        if arg == "--bind" and i + 2 < len(original_args):
            src = original_args[i + 1]
            dst = original_args[i + 2]

            # SRCとDSTが両方PWDと完全一致する場合
            if src == pwd_path and dst == pwd_path:

                # 最初の該当箇所にだけtmpfsを挿入
                if not tmpfs_inserted:
                    args.extend([
                        "--tmpfs", home,
                        "--dir", f"{home}/.codex",
                        "--ro-bind",
                        f"{home}/.codex/packages",
                        f"{home}/.codex/packages",
                    ])
                    tmpfs_inserted = True

            args.extend(["--bind", src, dst])
            i += 3
            continue

        args.append(arg)
        i += 1

    # ログ記録
    with open(log_file, "a", encoding="utf-8") as log:
        log.write("\n========== BWRAP START ==========\n")
        log.write(f"Timestamp: {timestamp}\n")
        log.write(f"User: {user_name}\n")
        log.write(f"PID: {pid}\n")
        log.write(f"Bwrap: {real_bwrap}\n")

        log.write(f"Original args: {format_args(original_args)}\n")
        log.write(f"Modified args: {format_args(args)}\n")

        log.write("\nEnvironment:\n")
        for key, value in sorted(os.environ.items()):
            log.write(f"{key}={value}\n")

        log.write("=========== BWRAP END ===========\n")

    # Bashのexecと同等（プロセスを置換）
    os.execv(real_bwrap, [real_bwrap, *args])


if __name__ == "__main__":
    main()
```

以下、それぞれの処理を見ていく。

## 2. `/dev`のマウント方法を変更する

最初の変更点は、`--dev /dev`の置き換えだ。

Bubblewrapの`--dev`は、指定先に新しいデバイス用ファイルシステムを構成する。一方、`--dev-bind`はホスト側のデバイスディレクトリをbindマウントする。

今回のラッパーでは、次の変換を行っている。

```
# 変更前
--dev /dev

# 変更後
--dev-bind /dev /dev
```

実装は単純で、引数を先頭から走査し、完全一致した場合のみ置き換える。

```
if (
    arg == "--dev"
    and i + 1 < len(original_args)
    and original_args[i + 1] == "/dev"
):
    args.extend(["--dev-bind", "/dev", "/dev"])
    i += 2
    continue
```

この変更によって、サンドボックス側からホストの`/dev`を参照できるようにする。

ただし、`--dev`と`--dev-bind`はセキュリティ上同等ではない。ホストのデバイスへのアクセス範囲が広がる可能性があるため、必要性を理解したうえで利用する。

## 3. ホームディレクトリをtmpfsにする

このラッパーの中心となる処理が、ホームディレクトリへの`tmpfs`の追加だ。

Codexが生成した引数の中から、次の条件に一致するbindマウントを探す。

```
--bind "$PWD" "$PWD"
```

ここでは、マウント元とマウント先の両方が、環境変数`PWD`と完全一致していることを条件にしている。

```
if src == pwd_path and dst == pwd_path:
```

条件に一致すると、そのbindマウントより前に次の設定を挿入する。

```
--tmpfs "$HOME" \
--dir "$HOME/.codex" \
--ro-bind "$HOME/.codex/packages" "$HOME/.codex/packages" \
--dir "$HOME/.local/bin" \
--symlink \
  "$HOME/.codex/packages/standalone/current/bin/codex" \
  "$HOME/.local/bin/codex"
```

### ホームディレクトリを隠す

```
--tmpfs "$HOME"
```

この指定によって、サンドボックス内のホームディレクトリに一時的なファイルシステムをマウントする。

元のホームディレクトリ全体をそのまま見せるのではなく、必要なディレクトリやファイルを後から追加する構成だ。

例えば、ホストのホームディレクトリに以下のファイルがあったとする。

```
/home/user/
├── .ssh/
├── .aws/
├── .config/
├── .codex/
├── .local/
└── projects/
```

`tmpfs`で覆った後は、後続のマウントで明示的に公開しない限り、これらはそのまま見えなくなる。

ただし、これだけでホームディレクトリ全体へのアクセスが完全に遮断されるわけではない。ほかのbindマウントやファイルディスクリプタ、後続のマウント設定などによってアクセスできる領域は変わる。

### Codexの実行ファイルを見せる

ホームディレクトリを`tmpfs`で覆うと、その中に配置されているCodex関連ファイルも参照できなくなる。

そこで、次の設定を追加している。

```
--dir "$HOME/.codex"

--ro-bind \
  "$HOME/.codex/packages" \
  "$HOME/.codex/packages"
```

`.codex`ディレクトリを作成し、その下の`packages`を読み取り専用でbindマウントする。

これにより、サンドボックス内ではCodexのパッケージを参照できる一方、ホスト側のパッケージへの書き込みは制限される。

さらに、Codexの実行ファイルへのシンボリックリンクを作成する。

```
--dir "$HOME/.local/bin"

--symlink \
  "$HOME/.codex/packages/standalone/current/bin/codex" \
  "$HOME/.local/bin/codex"
```

今回の実装では、Codexが以下にインストールされていることを前提にしている。

```
~/.codex/packages/standalone/current/bin/codex
```

別のインストール方式を利用している場合、このパスは調整する必要がある。

## 4. 作業ディレクトリはbindマウントで戻す

ホームディレクトリを一時的なファイルシステムにした後も、Codexが作業ディレクトリへアクセスできなければ意味がない。

そのため、元の`--bind`は削除せず、変更後の引数にも残している。

```
args.extend(["--bind", src, dst])
```

たとえば作業ディレクトリが次の場合、

```
/home/user/projects/example
```

引数は概念的に次の順序になる。

```
--tmpfs /home/user

--dir /home/user/.codex

--ro-bind \
  /home/user/.codex/packages \
  /home/user/.codex/packages

--dir /home/user/.local/bin

--symlink \
  /home/user/.codex/packages/standalone/current/bin/codex \
  /home/user/.local/bin/codex

--bind \
  /home/user/projects/example \
  /home/user/projects/example
```

つまり、ホームディレクトリ全体を一度隠したうえで、作業対象だけを再公開する構成を狙っている。

想定するサンドボックス内のファイル構成

/home/user/

`.codex/`

`packages/` 読み取り専用

`.local/bin/`

`codex` シンボリックリンク

`projects/`

`example/` 読み書き可能

元のbwrap設定との組み合わせによって、実際の可視範囲は変わる。

ただし、この構成には前提がある。

Bubblewrapではマウントの順序が重要であり、対象ディレクトリの親パスが正しく存在しなければ、後続のbindマウントが失敗することがある。

今回のコードでは`projects`などの中間ディレクトリを明示的に作成していない。そのため、作業ディレクトリの深さや元の`bwrap`引数によっては、`--dir`を追加する必要がある。

また、作業ディレクトリがホームの外にある場合でも、`PWD`のbindマウントが一致すれば`tmpfs`は挿入される。この動作を限定したい場合は、ホーム配下かどうかの判定を加えるとよい。

## 5. tmpfsの挿入は一度だけ

Codexが生成する引数に、同じ作業ディレクトリへのbindマウントが複数含まれている可能性も考慮した。

そこで、次のフラグを使用している。

```
tmpfs_inserted = False
```

条件に一致し、実際に挿入した時点で`True`に変更する。

```
if not tmpfs_inserted:
    args.extend([
        # 追加するマウント設定
    ])
    tmpfs_inserted = True
```

これにより、`tmpfs`などの追加設定は最初の一致箇所にだけ挿入される。

一方、元から存在する`--bind`はすべて維持する。

マウント設定を何度も追加しないための単純な対策だ。

## 6. bwrapの実行内容をログに残す

もう一つ重要なのがログ機能だ。

Bubblewrapは複数のマウント設定や名前空間の設定を組み合わせるため、意図した構成になっているかを確認しにくい。

そこで、ラッパーが受け取った引数と、変更後の引数を記録する。

ログの保存先はデフォルトで次のパスにしている。

```
~/.local/state/bwrap-wrapper.log
```

環境変数`BWRAP_LOG`を設定すれば変更可能だ。

```
export BWRAP_LOG="$HOME/bwrap-debug.log"
```

ログには次の内容が記録される。

- 実行日時、ユーザー名、PID
- 実際に起動する`bwrap`のパス
- 変更前と変更後の引数
- 起動時の環境変数

引数の記録には`shlex.quote()`を利用している。

```
def format_args(args: list[str]) -> str:
    return " ".join(shlex.quote(arg) for arg in args)
```

パスにスペースやシェルの特殊文字が含まれていても、引数の境界が分かりやすい形式で出力できる。

また、ファイル作成時の権限を制限するために、冒頭で次の設定を行っている。

```
os.umask(0o077)
```

ログが新規作成される際、通常は所有ユーザー以外から読み書きできない権限となる。

ただし、既存ファイルの権限は変更されない。

注意したいのは、環境変数をすべて記録している点だ。 APIキーや認証トークンなどの機密情報がログに残る可能性がある。

常用する場合は、機密性の高い環境変数を除外するか、必要な項目だけを記録する方式に変更した方が安全だ。

## 7. 最後はexecで本物のbwrapを起動する

引数の変更とログの記録が終わったら、実際のBubblewrapを起動する。

```
os.execv(real_bwrap, [real_bwrap, *args])
```

ここでは`subprocess.run()`ではなく、`os.execv()`を使用している。

`execv()`は新しい子プロセスを生成するのではなく、現在のプロセスを指定したプログラムに置き換える。

そのため、ラッパーを経由しても、プロセスの階層が余計に一段増えない。

また、`bwrap`が正常に起動した場合は、実行結果がそのまま呼び出し元に返る。

この用途では自然な実装だ。

## 8. 利用方法

たとえばスクリプトを以下の場所に保存する。

```
~/.local/bin/bwrapwrap
```

実行権限を付与する。

```
chmod 700 ~/.local/bin/bwrapwrap
```

次に、実際の`bwrap`のパスを確認する。

```
command -v bwrap
```

通常の環境では`/usr/bin/bwrap`などが返る。

異なる場所にインストールされている場合は、環境変数で指定できる。

```
export BWRAP_REAL=/usr/bin/bwrap
```

Codex側でBubblewrapの実行パスを設定または差し替えられる環境であれば、その実行先として`bwrapwrap`を指定する。

なお、Codexの起動方法やバージョンによって、外部の`bwrap`が使われるかどうか、またその差し替え方法は異なる。

単にスクリプトを配置するだけでは、Codexが自動的にこのラッパーを使うわけではない。

### 単体で動作確認する

まずは簡単な引数でテストできる。

```
bwrapwrap \
  --ro-bind / / \
  --proc /proc \
  --dev /dev \
  -- \
  /bin/true
```

この例では、少なくとも`--dev /dev`の変換と`bwrap`の起動処理を確認できる。

ただし、作業ディレクトリのbindマウントが含まれていないため、ホームディレクトリ用の`tmpfs`は挿入されない。

引数の変換結果はログから確認する。

```
tail -n 30 ~/.local/state/bwrap-wrapper.log
```

## 9. 実装上の注意点と改善余地

今回の実装は、特定のCodex環境に合わせた小さなラッパーだ。汎用的なBubblewrap引数パーサーではない。

運用する際には、次の点に注意する必要がある。

| 項目          | 注意点                              |
| ----------- | -------------------------------- |
| `PWD`の比較    | 文字列の完全一致。シンボリックリンクやパスの表記揺れは考慮しない |
| `--bind`の検出 | `--bind SRC DST`形式のみを処理する        |
| マウントの順序     | 後続のマウントや中間ディレクトリの有無で結果が変わる       |
| Codexのパス    | 特定のstandalone配置を前提としている          |
| `/dev`の公開   | ホスト側デバイスへのアクセスが広がる可能性がある         |
| ログ          | 引数や環境変数から機密情報が漏れるリスクがある          |

さらに、このコードには`is_under_home()`という関数が定義されているが、現状では呼び出していない。

```
def is_under_home(path: str, home: str) -> bool:
    return path == home or path.startswith(home + "/")
```

ホームディレクトリ配下の作業ディレクトリにだけ`tmpfs`を挿入したい場合は、次のように利用できる。

```
if (
    src == pwd_path
    and dst == pwd_path
    and is_under_home(pwd_path, home)
):
```

ただし、この判定でもシンボリックリンクなどの解決は行っていない。

また、セキュリティ目的で使う場合は、元の`bwrap`引数全体を検証することが重要だ。今回のラッパーは既存の設定を基本的に維持するため、他のマウント経由でホーム内のファイルが公開されている可能性までは排除できない。

## まとめ

今回作成した`bwrapwrap`は、Codexから呼び出されるBubblewrapの引数を部分的に変換するPython製ラッパーだ。

Codex本体を変更せずに、サンドボックスのマウント構成を調整し、実際に使用された引数を記録できる。

特に、ホームディレクトリを`tmpfs`で覆い、Codexの実行に必要なファイルと作業ディレクトリだけを再公開するという構成は、サンドボックスのファイルアクセス範囲を調整するうえで有用だ。

一方で、`/dev`の公開範囲や元のマウント設定など、セキュリティ上の前提にも注意が必要になる。

bwrapwrapは独立したセキュリティ境界ではなく、既存のBubblewrap設定を調整するための仕組みとして位置付けるのが適切だ。

Codexのサンドボックスが実際にどのような引数で構築されているのかを観察しながら、必要な設定を少しずつ調整したい場合に役立つ。
