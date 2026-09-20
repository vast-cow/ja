---
pubDatetime: 2026-06-23T19:15:02+09:00
title: "MSVC で `getopt.h` / `getopt.c` を使って `getopt` を利用する方法とライセンス上の注意点"
description: "前提、ファイルを用意する、最小サンプルを中心に、記事全体の背景・手順・確認方法・注意点を整理します。"
---

## 前提

Windows の MSVC 環境には、POSIX 系でよく使われる `getopt.h` / `getopt()` が標準では用意されていません。そのため、Linux 向けに書かれた C/C++ コードを MSVC でビルドすると、典型的には次のような問題が出ます。

```text
fatal error C1083: cannot open include file: 'getopt.h'
```

また、ヘッダだけを追加しても、`getopt()` / `getopt_long()` / `optind` / `optarg` などの実体がなければリンクに失敗します。

MSYS2 UCRT64 には MinGW-w64 由来の `getopt.h` が含まれています。MSYS2 の `mingw-w64-ucrt-x86_64-headers` パッケージは MinGW-w64 headers for Windows として提供され、UCRT64 repo に属します。パッケージ表示上のライセンスは `ZPL-2.1 AND LGPL-2.1-or-later` です。([MSYS2 Packages][1])

一方、`getopt.h` ファイル自体には、著作権が割り当てられておらず Public Domain に置かれている旨の disclaimer が入っています。([GitHub][2])

この記事では、この MinGW-w64 由来の `getopt.h` と `getopt.c` を MSVC プロジェクトに取り込み、`getopt()` / `getopt_long()` を使えるようにする方法をまとめます。

---

## 結論

MSVC で `getopt` を使うには、次の構成が最も安全です。

```text
your_project/
  include/
    getopt.h
  src/
    getopt.c
    main.c
```

そして、MSVC で `getopt.c` を自分のプロジェクトの一部として一緒にコンパイルします。

```bat
cl /Iinclude src\main.c src\getopt.c
```

やってはいけない構成は、MSVC プロジェクトに MSYS2 の `/ucrt64/include` や `/ucrt64/lib` を丸ごと追加する方法です。MinGW-w64 と MSVC の CRT・ヘッダ・ライブラリを混在させることになり、ビルドやリンクの問題を起こしやすくなります。

---

## 1. ファイルを用意する

MinGW-w64 由来の次の 2 ファイルをプロジェクトにコピーします。

```text
getopt.h
getopt.c
```

MSYS2 UCRT64 をインストールしている場合、`getopt.h` は通常次の場所にあります。

```text
C:\msys64\ucrt64\include\getopt.h
```

ただし、MSVC プロジェクトでは `C:\msys64\ucrt64\include` 全体を include path に入れるのではなく、必要な `getopt.h` だけを自分のプロジェクト配下へコピーする方が安全です。

`getopt.c` は MinGW-w64 CRT 側の実装ファイルです。MSYS2 の `mingw-w64-ucrt-x86_64-crt` パッケージは MinGW-w64 CRT for Windows として提供され、パッケージ表示上のライセンスは `ZPL-2.1` です。([MSYS2 Packages][3])

ただし、`getopt.c` ファイル自体には Todd C. Miller 由来の permissive license と NetBSD 由来の BSD 系ライセンス文が含まれています。ソース・バイナリ再配布を許可する一方で、著作権表示、条件文、免責文の維持・再掲が求められています。([GitHub][4])

---

## 2. 最小サンプル

### `main.c`

```c
#include <stdio.h>
#include <getopt.h>

int main(int argc, char *argv[])
{
    int opt;

    while ((opt = getopt(argc, argv, "ab:c")) != -1) {
        switch (opt) {
        case 'a':
            printf("option -a\n");
            break;

        case 'b':
            printf("option -b with argument: %s\n", optarg);
            break;

        case 'c':
            printf("option -c\n");
            break;

        default:
            printf("usage: %s [-a] [-b value] [-c]\n", argv[0]);
            return 1;
        }
    }

    for (int i = optind; i < argc; ++i) {
        printf("non-option argument: %s\n", argv[i]);
    }

    return 0;
}
```

### ビルド

Developer Command Prompt for VS などで、次のようにビルドします。

```bat
cl /Iinclude src\main.c src\getopt.c
```

出力例:

```bat
main.exe -a -b hello file1 file2
```

```text
option -a
option -b with argument: hello
non-option argument: file1
non-option argument: file2
```

---

## 3. `getopt_long()` を使う例

MinGW-w64 の `getopt.h` には `struct option`、`no_argument`、`required_argument`、`optional_argument`、`getopt_long()`、`getopt_long_only()` の宣言も含まれています。([GitHub][2])

### `main.c`

```c
#include <stdio.h>
#include <getopt.h>

int main(int argc, char *argv[])
{
    static struct option long_options[] = {
        { "help",    no_argument,       0, 'h' },
        { "output",  required_argument, 0, 'o' },
        { "verbose", no_argument,       0, 'v' },
        { 0,         0,                 0,  0  }
    };

    int opt;
    int option_index = 0;

    while ((opt = getopt_long(argc, argv, "ho:v", long_options, &option_index)) != -1) {
        switch (opt) {
        case 'h':
            printf("help\n");
            break;

        case 'o':
            printf("output: %s\n", optarg);
            break;

        case 'v':
            printf("verbose\n");
            break;

        default:
            printf("usage: %s [--help] [--output file] [--verbose]\n", argv[0]);
            return 1;
        }
    }

    return 0;
}
```

### 実行例

```bat
main.exe --verbose --output result.txt
```

```text
verbose
output: result.txt
```

---

## 4. Visual Studio プロジェクトで使う場合

Visual Studio の `.vcxproj` で使う場合は、次のように設定します。

### ファイル配置

```text
project/
  include/
    getopt.h
  src/
    getopt.c
    main.c
```

### 設定

| 項目                                               | 設定                     |
| ------------------------------------------------ | ---------------------- |
| C/C++ → General → Additional Include Directories | `$(ProjectDir)include` |
| Source Files                                     | `getopt.c` を追加         |
| Linker                                           | 特別な追加ライブラリは不要          |

`getopt.c` をプロジェクトに追加していない場合、コンパイルは通ってもリンクで失敗します。`getopt.h` は宣言を提供するだけで、`getopt()` や `optind` などの実体は `getopt.c` 側にあるためです。MinGW-w64 の `getopt.c` では、`optind`、`opterr`、`optopt`、`optarg` の定義、および `getopt()` / `getopt_long()` / `getopt_long_only()` の実装が提供されています。([GitHub][4])

---

## 5. MSYS2 の include/lib を MSVC に直接混ぜない

次のような設定は避けるべきです。

```text
C:\msys64\ucrt64\include
C:\msys64\ucrt64\lib
```

これらを MSVC の include path / library path に丸ごと追加すると、MinGW-w64 用のヘッダや import library が MSVC の Windows SDK / UCRT / CRT と混在します。

特に問題になりやすいのは次の領域です。

| 領域          | 問題                                                       |
| ----------- | -------------------------------------------------------- |
| CRT ヘッダ     | MSVC CRT と MinGW-w64 CRT の前提が異なる                         |
| Windows ヘッダ | Windows SDK 版と MinGW-w64 版が混在しうる                         |
| `.a` ライブラリ  | MinGW-w64/GCC 向けの import library は MSVC の `.lib` と前提が異なる |
| ABI         | C++ やランタイム境界で不整合が起きやすい                                   |

したがって、MSVC で `getopt` だけ使いたい場合は、MSYS2 環境そのものをリンク対象にするのではなく、`getopt.h` と `getopt.c` をローカルに取り込んで MSVC でコンパイルするのが妥当です。

---

## 6. ライセンス上の注意点

### `getopt.h`

MinGW-w64 の `getopt.h` には、著作権が割り当てられておらず Public Domain に置かれている旨の disclaimer が記載されています。([GitHub][2])

したがって、`getopt.h` を include するだけで、自分のプログラムに GPL/LGPL のようなソース公開義務が生じる、という扱いには通常なりません。

### `getopt.c`

注意すべきなのは `getopt.c` です。

`getopt.c` には少なくとも次のライセンス文が含まれています。

| 由来                     | 性質                              |
| ---------------------- | ------------------------------- |
| Todd C. Miller 由来部分    | ISC/BSD 系に近い permissive license |
| NetBSD Foundation 由来部分 | 2-clause BSD 系ライセンス             |

どちらも permissive 系であり、商用・クローズドソース製品に組み込むこと自体は通常可能です。ただし、`getopt.c` のライセンス文では、ソース形式の再配布時には著作権表示・条件・免責文を保持し、バイナリ形式の再配布時にはそれらをドキュメントまたはその他の配布物に再掲することが条件になっています。([GitHub][4])

### 実務上の対応

バイナリを配布する場合は、次のような `THIRD-PARTY-NOTICES.txt` または `LICENSES/` ディレクトリを用意します。

```text
THIRD-PARTY-NOTICES.txt
LICENSES/
  getopt.txt
```

`getopt.txt` には、取り込んだ `getopt.c` の冒頭ライセンスコメントをそのまま保存しておくのが安全です。

---

## 7. 自作バイナリのライセンスはどうなるか

`getopt.h` / `getopt.c` を取り込んで MSVC で静的にコンパイルした場合でも、自作アプリケーション全体を GPL や LGPL にしなければならない、という性質のものではありません。

扱いとしては次のようになります。

| 項目         | 扱い                                       |
| ---------- | ---------------------------------------- |
| 自作ソースコード   | 任意のライセンスにできる                             |
| `getopt.h` | Public Domain disclaimer あり              |
| `getopt.c` | permissive 系ライセンス条件に従う                   |
| バイナリ配布     | `getopt.c` の著作権表示・条件・免責文を notice として同梱する |
| ソース公開義務    | 通常なし                                     |
| 商用利用       | 通常可能                                     |

重要なのは、「ソース公開義務」ではなく「notice 義務」です。

---

## 8. 推奨する配布物構成

商用・非商用を問わず、配布物には次のような構成を推奨します。

```text
dist/
  your_app.exe
  README.txt
  THIRD-PARTY-NOTICES.txt
```

`THIRD-PARTY-NOTICES.txt` の例:

```text
This product includes getopt implementation derived from MinGW-w64 / OpenBSD / NetBSD sources.

The getopt.h file from MinGW-w64 states that it has no copyright assigned
and is placed in the Public Domain.

The getopt.c file contains permissive license notices from Todd C. Miller
and The NetBSD Foundation, Inc. The original copyright notices, permission
notice, conditions, and disclaimers are reproduced below.

[ここに getopt.c 冒頭のライセンスコメントをそのまま貼る]
```

日本語だけの配布物でも、ライセンス原文は英語のまま同梱するのが無難です。翻訳文を付ける場合でも、原文を削除しないでください。

---

## 9. 実装上の注意点

### `optind` をリセットしたい場合

同一プロセス内で複数回 `getopt()` を呼び直す場合、`optind` を戻す必要があります。

```c
optind = 1;
```

MinGW-w64 の実装では BSD 系の `optreset` もサポートされていますが、移植性を考えるなら `optind = 1` を基本にした方が扱いやすいです。`getopt.h` 側にも BSD の非標準 `optreset` について、移植性のため開発者には使用を避けることが推奨される旨のコメントがあります。([GitHub][2])

### C++ から使う場合

`getopt.h` は `extern "C"` に対応しているため、C++ ソースからも利用できます。([GitHub][2])

```cpp
#include "getopt.h"
```

ただし、`getopt.c` は C ソースとしてコンパイルするのが自然です。Visual Studio では拡張子 `.c` のままプロジェクトに追加してください。

### `/utf-8` や Unicode とは別問題

`getopt()` が扱うのは `main(int argc, char *argv[])` の narrow 文字列です。Windows の Unicode コマンドラインを厳密に扱いたい場合は、`wmain(int argc, wchar_t *argv[])` や `CommandLineToArgvW()` を使う設計も検討する必要があります。

`getopt` はあくまで POSIX 風の narrow argv パーサとして使うものです。

---

## 10. まとめ

MSVC で `getopt` を使う場合は、次の方針が安全です。

1. `getopt.h` と `getopt.c` をプロジェクトにコピーする。
2. `getopt.c` を MSVC で自分のプロジェクトの一部としてコンパイルする。
3. MSYS2 の `/ucrt64/include` や `/ucrt64/lib` を MSVC に丸ごと混ぜない。
4. `getopt.h` は Public Domain disclaimer ありと扱う。
5. `getopt.c` は permissive 系ライセンスとして、著作権表示・条件・免責文を配布物に同梱する。
6. 自作バイナリのソース公開義務は通常発生しないが、third-party notice 対応は行う。

要するに、MSVC で使う場合の実務上のポイントは、**「ヘッダだけではなく実装ファイル `getopt.c` も一緒にビルドする」こと**と、**「`getopt.c` のライセンス表示を配布物に残す」こと**です。

[1]: https://packages.msys2.org/package/mingw-w64-ucrt-x86_64-headers "Package: mingw-w64-ucrt-x86_64-headers - MSYS2 Packages"
[2]: https://raw.githubusercontent.com/mingw-w64/mingw-w64/master/mingw-w64-headers/crt/getopt.h "raw.githubusercontent.com"
[3]: https://packages.msys2.org/package/mingw-w64-ucrt-x86_64-crt "Package: mingw-w64-ucrt-x86_64-crt - MSYS2 Packages"
[4]: https://raw.githubusercontent.com/mingw-w64/mingw-w64/master/mingw-w64-crt/misc/getopt.c "raw.githubusercontent.com"
