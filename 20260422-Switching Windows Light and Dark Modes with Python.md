---
pubDatetime: 2026-04-22T21:28:46+09:00
title: "Windowsのライト／ダークモードをPythonで切り替える方法"
description: ":::note 壁紙（単色）, VS Code, Windows Terminalの配色も変更する版を作りました。 https://qiita.com/vast-cow/items/3215d89727cd47cefbc1 ::: このスクリプトは、WindowsのライトモードとダークモードをPyt…"
---

:::note
壁紙（単色）, VS Code, Windows Terminalの配色も変更する版を作りました。
https://qiita.com/vast-cow/items/3215d89727cd47cefbc1
:::

このスクリプトは、WindowsのライトモードとダークモードをPythonから簡単に切り替えるためのものです。レジストリを操作し、設定変更をシステム全体に通知することで、即座にテーマを反映させます。

## 目的

Windowsでは通常、設定画面からライト／ダークモードを切り替えますが、このコードを使うと以下が可能になります。

* ワンクリックでテーマを切り替える
* 自動化スクリプトに組み込む
* ショートカットやタスクスケジューラと連携する

## 仕組みの概要

このスクリプトは主に2つの処理を行います。

1. **レジストリの値を変更する**
   Windowsのテーマ設定は、ユーザーのレジストリに保存されています。
   以下の2つの値を変更します：

   * `AppsUseLightTheme`（アプリのテーマ）
   * `SystemUsesLightTheme`（システムのテーマ）

2. **設定変更を通知する**
   レジストリを書き換えただけでは見た目は変わりません。
   そのため、Windowsに「設定が変わった」ことを通知して、テーマを即時反映させます。

## 主な機能

### 現在のテーマを取得

```python
get_current_theme()
```

現在がライトモードかどうかを確認します。
戻り値は以下の通りです：

* `1` → ライトモード
* `0` → ダークモード

### テーマを設定

```python
set_theme(light: bool)
```

引数に応じてテーマを変更します。

* `True` → ライトモード
* `False` → ダークモード

### テーマを切り替え

```python
toggle_theme()
```

現在の状態を反転します。
ライトならダークへ、ダークならライトへ切り替えます。

## 使い方

### 基本的な実行

スクリプトをそのまま実行すると、自動的にテーマが切り替わります。

```bash
python script.py
```

実行後、現在のモードが表示されます：

* `Light mode`
* `Dark mode`

### 応用例

* ショートカットに登録してワンクリック切替
* 時間帯で自動切替（例：夜はダークモード）
* 他のPythonツールと連携

## 注意点

* Windows専用のコードです
* レジストリを操作するため、環境によっては権限が必要になる場合があります
* 一部のアプリは即時反映されないことがあります

## まとめ

このスクリプトを使うことで、Windowsのテーマ切り替えをシンプルに自動化できます。日常的にライト／ダークモードを切り替える場合や、作業環境を効率化したい場合に有用です。

```python
import ctypes
from ctypes import wintypes
import winreg


PERSONALIZE_KEY = r"Software\Microsoft\Windows\CurrentVersion\Themes\Personalize"
APPS_KEY = "AppsUseLightTheme"
SYSTEM_KEY = "SystemUsesLightTheme"

# Win32 constants
HWND_BROADCAST = 0xFFFF
WM_SETTINGCHANGE = 0x001A
SMTO_ABORTIFHUNG = 0x0002

# Fallback for environments where ULONG_PTR is missing
if hasattr(wintypes, "ULONG_PTR"):
    ULONG_PTR = wintypes.ULONG_PTR
else:
    ULONG_PTR = ctypes.c_size_t


def get_current_theme() -> int:
    try:
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, PERSONALIZE_KEY) as key:
            value, regtype = winreg.QueryValueEx(key, APPS_KEY)
            if regtype == winreg.REG_DWORD:
                return int(value)
    except FileNotFoundError:
        pass
    return 1


def set_theme(light: bool) -> None:
    value = 1 if light else 0

    with winreg.CreateKey(winreg.HKEY_CURRENT_USER, PERSONALIZE_KEY) as key:
        winreg.SetValueEx(key, APPS_KEY, 0, winreg.REG_DWORD, value)
        winreg.SetValueEx(key, SYSTEM_KEY, 0, winreg.REG_DWORD, value)

    notify_theme_changed()


def toggle_theme() -> bool:
    current = get_current_theme()
    new_light = not bool(current)
    set_theme(new_light)
    return new_light


def notify_theme_changed() -> None:
    user32 = ctypes.WinDLL("user32", use_last_error=True)

    SendMessageTimeoutW = user32.SendMessageTimeoutW
    SendMessageTimeoutW.argtypes = [
        wintypes.HWND,
        wintypes.UINT,
        wintypes.WPARAM,
        wintypes.LPCWSTR,
        wintypes.UINT,
        wintypes.UINT,
        ctypes.POINTER(ULONG_PTR),
    ]
    SendMessageTimeoutW.restype = wintypes.LPARAM

    result = ULONG_PTR()
    ret = SendMessageTimeoutW(
        HWND_BROADCAST,
        WM_SETTINGCHANGE,
        0,
        "ImmersiveColorSet",
        SMTO_ABORTIFHUNG,
        5000,
        ctypes.byref(result),
    )

    if ret == 0:
        raise ctypes.WinError(ctypes.get_last_error())


if __name__ == "__main__":
    is_light = toggle_theme()
    print("Light mode" if is_light else "Dark mode")
```
