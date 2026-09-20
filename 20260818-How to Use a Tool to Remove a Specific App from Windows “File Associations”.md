---
title: "Windowsの「関連付け」から特定アプリを削除するツールの使い方"
description: ""
pubDatetime: 2026-08-18T08:41:19.711Z
---

Windowsでは、アンインストール済みのアプリやポータブル版のアプリが、ファイルの関連付け情報だけをレジストリに残してしまうことがあります。

たとえば `.epub` や `.pdf` などのファイルで「プログラムから開く」を表示したとき、すでに使っていないアプリが候補として残るケースです。

ここでは、`HKEY_CLASSES_ROOT` に登録されているファイル関連付けを調べ、指定した実行ファイルに紐付いているレジストリキーを削除するPythonツールの使い方を説明します。

## このツールがすること

このツールは `HKEY_CLASSES_ROOT`、略して `HKCR` の直下にあるキーを調べます。

各キーについて、

```text
HKEY_CLASSES_ROOT\<キー名>\shell\open\command
```

を確認し、既定値に設定されているコマンドの実行ファイル名を取得します。

たとえば次のような登録があったとします。

```text
"C:\Program Files\Calibre2\ebook-viewer.exe" "%1"
```

指定した実行ファイル名が `ebook-viewer.exe` なら、この関連付けを対象として検出します。

通常実行すると対象のレジストリキーを削除し、`--dry-run` を付けると削除せず対象だけを表示します。

## 動作環境

このスクリプトはWindows専用です。

Pythonの標準ライブラリである、

* `argparse`
* `ctypes`
* `os`
* `winreg`

だけを使用しているため、追加パッケージのインストールは必要ありません。

Python 3をインストールしたWindows環境で使用できます。

## まずはスクリプトを保存する

コードを、たとえば次の名前で保存します。

```text
remove_association.py
```

コマンドプロンプトまたはPowerShellを開き、ファイルを保存したディレクトリへ移動してください。

例:

```powershell
cd C:\Tools
```

## 最初は必ず `--dry-run` で確認する

レジストリキーを削除するツールなので、最初から削除を実行するのではなく、まず `--dry-run` を使って対象を確認するのが安全です。

書式は次のとおりです。

```powershell
python remove_association.py <実行ファイル名> --dry-run
```

たとえば `calibre.exe` に関連付けられた項目を調べる場合は、

```powershell
python remove_association.py calibre.exe --dry-run
```

と実行します。

対象が見つかると、次のような形式で表示されます。

```text
Found 2 association(s).

HKEY_CLASSES_ROOT\Calibre...
  command = "C:\Program Files\Calibre2\calibre.exe" "%1"

HKEY_CLASSES_ROOT\...
  command = "C:\Program Files\Calibre2\calibre.exe" "%1"

dry-run: No keys were deleted.
```

最後に

```text
dry-run: No keys were deleted.
```

と表示されている場合、レジストリは変更されていません。

## 実際に削除する

`--dry-run` の結果を確認し、本当に不要な関連付けだけが検出されていることを確認したら、`--dry-run` を外して実行します。

```powershell
python remove_association.py calibre.exe
```

削除に成功すると、

```text
DELETED: HKEY_CLASSES_ROOT\...
```

と表示されます。

複数の関連付けが見つかった場合は、検出されたキーが順番に削除されます。

## 実行ファイル名だけを指定する

引数に指定するのは、基本的には実行ファイルのフルパスではなくファイル名です。

たとえば、

```text
C:\Program Files\Calibre2\calibre.exe
```

ではなく、

```text
calibre.exe
```

を指定します。

このツールはレジストリに登録されたコマンドをWindowsのルールに従って分解し、その先頭にある実行ファイルのファイル名だけを取り出して比較しています。

比較では大文字・小文字を区別しません。そのため、

```text
CALIBRE.EXE
```

と

```text
calibre.exe
```

は同じものとして扱われます。

## なぜ単純な文字列検索をしていないのか

関連付けのコマンドは、単純に実行ファイルのパスだけが保存されているとは限りません。

たとえば、

```text
"C:\Program Files\Example\viewer.exe" "%1"
```

や、

```text
"C:\Program Files\Example\viewer.exe" --open "%1"
```

のような形式があります。

さらにWindowsのコマンドラインでは、引用符や空白の扱いに独自の規則があります。

そこでこのツールでは、

```text
CommandLineToArgvW
```

というWindows APIを使ってコマンド文字列を解析しています。

解析後の最初の引数から、

```python
os.path.basename(parts[0])
```

で実行ファイル名を取り出しています。

そのため、単純に

```python
if "calibre.exe" in command:
```

のような部分一致で判定するより、誤検出が起こりにくい構造になっています。

## `REG_EXPAND_SZ` にも対応

レジストリのコマンドが通常の文字列 `REG_SZ` だけでなく、環境変数を含む `REG_EXPAND_SZ` だった場合にも対応しています。

たとえば、

```text
"%ProgramFiles%\Example\viewer.exe" "%1"
```

のような値です。

この場合は、

```python
os.path.expandvars(value)
```

によって環境変数を展開してから解析します。

## 関連付けの検索方法

検索の中心となっているのが `find_associations()` です。

```python
def find_associations(target_exe: str):
```

ここでは `HKEY_CLASSES_ROOT` の直下を `winreg.EnumKey()` で順番に走査しています。

それぞれについて、

```text
<キー>\shell\open\command
```

を読み取り、実行ファイル名が指定された名前と一致すると、

```python
yield key_name, command
```

で削除候補として返します。

つまり、このツールの検索対象は **HKCR直下の各キーに存在する `shell\open\command`** です。

Windowsに存在するあらゆる種類の関連付け情報を網羅的に検索するツールではない点には注意してください。

## 削除はサブキーも含めて行われる

レジストリキーは、その下にサブキーが存在すると単純には削除できません。

そのため `delete_registry_tree()` では、子キーを再帰的に削除してから親キーを削除しています。

概念的には、

```text
対象キー
├─ DefaultIcon
├─ shell
│  └─ open
│     └─ command
└─ その他
```

という構造があった場合、下の階層から順番に削除し、最後に「対象キー」そのものを削除します。

したがって、このツールで検出されたキーを削除すると、`shell\open\command` だけではなく **その関連付けキー全体が削除されます。**

これは重要なポイントです。

## 権限エラーが出た場合

削除時に、

```text
ACCESS DENIED: HKEY_CLASSES_ROOT\...
```

と表示されることがあります。

これは、そのレジストリキーを書き換えるための権限が現在のユーザーにない場合などに発生します。

必要であれば「管理者として実行」したPowerShellやコマンドプロンプトから実行します。

ただし、権限エラーが出たからといって、無条件に管理者権限で削除すべきとは限りません。

まず `--dry-run` の出力を見て、そのキーが本当に不要なものか確認してください。

## 対象が見つからない場合

一致する関連付けが存在しない場合は、

```text
No associations matching 'calibre.exe' were found.
```

のように表示されます。

この場合、何も削除されません。

ただし、目的の関連付けがWindows上に表示されているにもかかわらず検出されないこともあります。

Windowsのファイル関連付けは複数の場所に保存されており、このツールが調べているのは、

```text
HKEY_CLASSES_ROOT\<key>\shell\open\command
```

という形式の登録だけだからです。

## 使用例

`foo.exe` に紐付く関連付けを確認するだけなら、

```powershell
python remove_association.py foo.exe --dry-run
```

確認後、削除するなら、

```powershell
python remove_association.py foo.exe
```

です。

たとえば検出結果が、

```text
Found 1 association(s).

HKEY_CLASSES_ROOT\Foo.Document
  command = "C:\OldApps\Foo\foo.exe" "%1"
```

だったとします。

この状態で通常実行すると、削除対象は単に、

```text
HKEY_CLASSES_ROOT\Foo.Document\shell\open\command
```

だけではありません。

```text
HKEY_CLASSES_ROOT\Foo.Document
```

以下のツリー全体です。

そのため、`Foo.Document` に別の情報も登録されている場合、それらも失われます。

## 使用上の注意

このツールはレジストリを直接削除します。削除したキーを元に戻す機能はありません。

特に注意すべきなのは、指定したEXE名に一致した `shell\open\command` を見つけると、その `command` キーだけでなく **HKCR直下の関連付けキーそのものを再帰的に削除する** ことです。

そのため、基本的には次の順序で使うことを推奨します。

1. 対象となるEXE名を確認する
2. `--dry-run` で検索する
3. 表示されたすべてのレジストリキーを確認する
4. 必要ならレジストリエディターで対象キーをエクスポートしてバックアップする
5. 問題がないことを確認してから通常実行する

特にWindows標準アプリや現在使用中のアプリを対象にするのは避けたほうがよいでしょう。

## まとめ

このツールは、Windowsの `HKEY_CLASSES_ROOT` を調べて、

```text
<キー>\shell\open\command
```

に指定した実行ファイルが登録されている関連付けを探し、そのキーを削除するためのものです。

基本的な使い方はシンプルです。

確認だけなら、

```powershell
python remove_association.py calibre.exe --dry-run
```

実際に削除するなら、

```powershell
python remove_association.py calibre.exe
```

です。

レジストリを直接操作するため、**最初に `--dry-run` で対象を確認することが最も重要です。**

アンインストール後も残っている古いファイル関連付けや、不要になったアプリの関連付けを整理するときに使えるツールですが、削除対象は関連付けキー全体なので、内容を確認したうえで慎重に使用してください。

```python
import argparse
import ctypes
import os
import winreg
from ctypes import wintypes


shell32 = ctypes.WinDLL("shell32", use_last_error=True)
kernel32 = ctypes.WinDLL("kernel32", use_last_error=True)

shell32.CommandLineToArgvW.argtypes = [
    wintypes.LPCWSTR,
    ctypes.POINTER(ctypes.c_int),
]
shell32.CommandLineToArgvW.restype = ctypes.POINTER(wintypes.LPWSTR)

kernel32.LocalFree.argtypes = [wintypes.HLOCAL]
kernel32.LocalFree.restype = wintypes.HLOCAL


def split_windows_command(command: str) -> list[str]:
    argc = ctypes.c_int()

    argv = shell32.CommandLineToArgvW(command, ctypes.byref(argc))
    if not argv:
        raise ctypes.WinError(ctypes.get_last_error())

    parts = [argv[i] for i in range(argc.value)]
    kernel32.LocalFree(argv)

    return parts


def get_open_command(root, key_name: str) -> str | None:
    subkey = rf"{key_name}\shell\open\command"

    try:
        with winreg.OpenKey(root, subkey, 0, winreg.KEY_READ) as key:
            value, value_type = winreg.QueryValueEx(key, "")

            if value_type in (winreg.REG_SZ, winreg.REG_EXPAND_SZ):
                if value_type == winreg.REG_EXPAND_SZ:
                    value = os.path.expandvars(value)
                return value

    except (FileNotFoundError, PermissionError, OSError):
        pass

    return None


def executable_name_from_command(command: str) -> str | None:
    try:
        parts = split_windows_command(command)
    except (ValueError, OSError):
        return None

    if not parts:
        return None

    return os.path.basename(parts[0])


def find_associations(target_exe: str):
    target_exe = target_exe.casefold()

    root = winreg.HKEY_CLASSES_ROOT
    index = 0

    while True:
        try:
            key_name = winreg.EnumKey(root, index)
        except OSError:
            break

        index += 1

        command = get_open_command(root, key_name)
        if command is None:
            continue

        executable_name = executable_name_from_command(command)
        if executable_name is None:
            continue

        if executable_name.casefold() == target_exe:
            yield key_name, command


def delete_registry_tree(root, subkey: str):
    try:
        with winreg.OpenKey(
            root,
            subkey,
            0,
            winreg.KEY_READ | winreg.KEY_WRITE,
        ) as key:
            while True:
                try:
                    child = winreg.EnumKey(key, 0)
                except OSError:
                    break

                delete_registry_tree(root, rf"{subkey}\{child}")

        winreg.DeleteKey(root, subkey)

    except FileNotFoundError:
        pass


def main():
    parser = argparse.ArgumentParser(
        description=(
            r"Checks the executable name in HKCR\<key>\shell\open\command and "
            "deletes keys associated with the specified exe."
        )
    )

    parser.add_argument(
        "exe",
        help="Executable filename to search for. Example: calibre.exe",
    )

    parser.add_argument(
        "--dry-run",
        action="store_true",
        help="List only the target keys without deleting them",
    )

    args = parser.parse_args()

    matches = list(find_associations(args.exe))

    if not matches:
        print(f"No associations matching {args.exe!r} were found.")
        return

    print(f"Found {len(matches)} association(s).")
    print()

    for key_name, command in matches:
        print(fr"HKEY_CLASSES_ROOT\{key_name}")
        print(f"  command = {command}")

    if args.dry_run:
        print()
        print("dry-run: No keys were deleted.")
        return

    print()
    print("Deleting.")

    for key_name, command in matches:
        full_name = fr"HKEY_CLASSES_ROOT\{key_name}"

        try:
            delete_registry_tree(winreg.HKEY_CLASSES_ROOT, key_name)
            print(f"DELETED: {full_name}")
        except PermissionError:
            print(f"ACCESS DENIED: {full_name}")
        except OSError as e:
            print(f"ERROR: {full_name}: {e}")


if __name__ == "__main__":
    main()
```
