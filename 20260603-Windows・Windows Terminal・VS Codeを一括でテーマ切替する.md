---
pubDatetime: 2026-06-03T15:22:08+09:00
title: "Windows・Windows Terminal・VS Codeを一括でテーマ切替する"
description: "以下の Python スクリプトは、Windows のライト／ダークテーマを明示的に切り替えつつ、Windows Terminal・VS Code・壁紙色も連動して変更する CLI ツールです。対象は Windows 環境です。 何をするスクリプトか このスクリプトは --day または --nig…"
---

以下の Python スクリプトは、**Windows のライト／ダークテーマを明示的に切り替えつつ、Windows Terminal・VS Code・壁紙色も連動して変更する CLI ツール**です。対象は Windows 環境です。

## 何をするスクリプトか

このスクリプトは `--day` または `--night` を指定して実行します。

* `--day`

  * Windows をライトテーマにする
  * デスクトップ背景を白にする
  * Windows Terminal のカラースキームを昼用にする
  * VS Code のテーマを昼用にする

* `--night`

  * Windows をダークテーマにする
  * デスクトップ背景を黒にする
  * Windows Terminal のカラースキームを夜用にする
  * VS Code のテーマを夜用にする

時刻による自動判定はありません。**昼用／夜用を手動で明示的に適用する設計**です。

---

## 使い方

### 1. 必要ファイル

スクリプトと同じフォルダに、次の名前で設定ファイルを置く必要があります。

```text
config.toml
```

スクリプト内では以下のように定義されています。

```python
CONFIG_PATH = Path(__file__).with_name("config.toml")
```

つまり、たとえばスクリプトが以下にあるなら、

```text
C:\Users\you\theme_switcher.py
```

設定ファイルは以下に必要です。

```text
C:\Users\you\config.toml
```

---

### 2. `config.toml` の例

このスクリプトは、次の TOML 構造を期待しています。

```toml
[terminal]
day_color_scheme = "One Half Light"
night_color_scheme = "One Half Dark"

[vscode]
day_theme = "Default Light Modern"
night_theme = "Default Dark Modern"
```

実際の値は、自分の Windows Terminal と VS Code に存在するテーマ名に合わせる必要があります。

---

### 3. 昼テーマを適用する

```powershell
python theme_switcher.py --day
```

実行すると、以下が適用されます。

```text
Windows: ライトテーマ
壁紙: 白
Windows Terminal: config.toml の terminal.day_color_scheme
VS Code: config.toml の vscode.day_theme
```

---

### 4. 夜テーマを適用する

```powershell
python theme_switcher.py --night
```

実行すると、以下が適用されます。

```text
Windows: ダークテーマ
壁紙: 黒
Windows Terminal: config.toml の terminal.night_color_scheme
VS Code: config.toml の vscode.night_theme
```

---

## 主な機能

### 1. Windows のライト／ダークテーマ切り替え

`set_theme(light: bool)` が、Windows レジストリを書き換えます。

対象レジストリキーは以下です。

```text
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize
```

変更する値は以下の 2 つです。

```text
AppsUseLightTheme
SystemUsesLightTheme
```

値の意味は以下です。

|   値 | 意味     |
| --: | ------ |
| `1` | ライトテーマ |
| `0` | ダークテーマ |

変更後は `WM_SETTINGCHANGE` をブロードキャストして、Windows にテーマ変更を通知します。

---

### 2. 壁紙を単色に変更

`set_solid_wallpaper(r, g, b)` が、Windows の COM API を使ってデスクトップ背景を単色にします。

このスクリプトでは固定で以下の色が使われます。

```python
DAY_WALLPAPER_RGB = (255, 255, 255)
NIGHT_WALLPAPER_RGB = (0, 0, 0)
```

つまり、

| モード       | 壁紙色 |
| --------- | --- |
| `--day`   | 白   |
| `--night` | 黒   |

画像ファイルを壁紙にするのではなく、`IDesktopWallpaper::SetBackgroundColor` と `IDesktopWallpaper::Enable(False)` を使って、壁紙画像を無効化し、背景色を指定しています。

---

### 3. Windows Terminal のカラースキーム変更

`set_windows_terminal_color_scheme(color_scheme)` が Windows Terminal の `settings.json` を編集します。

スクリプトは以下の場所から Windows Terminal の設定ファイルを探します。

```text
%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_*\LocalState\settings.json
```

変更する JSON 項目は以下です。

```json
{
    "profiles": {
        "defaults": {
            "colorScheme": "..."
        }
    }
}
```

たとえば `--night` の場合、`config.toml` の以下が使われます。

```toml
[terminal]
night_color_scheme = "One Half Dark"
```

---

### 4. VS Code のテーマ変更

`set_vscode_color_theme(theme_name)` が VS Code のユーザー設定を編集します。

対象ファイルは以下です。

```text
%USERPROFILE%\AppData\Roaming\Code\User\settings.json
```

変更する JSON 項目は以下です。

```json
{
    "workbench.colorTheme": "Default Dark Modern"
}
```

`settings.json` が存在しない場合は、新規に作成されます。

---

### 5. 不要なファイル書き換えを避ける

`write_json_if_changed()` は、変更後の JSON 文字列と既存ファイルの内容を比較します。

内容が同じ場合は書き込みません。

```python
if old_text == new_text:
    return False
```

そのため、すでに目的の設定になっている場合は無駄な更新を避けます。

---

## 実行時の流れ

### `--day` の場合

```text
main()
  └─ apply_day_theme()
      └─ apply_explicit_theme(
             light=True,
             terminal_color_scheme=CONFIG["day_terminal_color_scheme"],
             vscode_theme=CONFIG["day_vscode_theme"],
             wallpaper_rgb=(255, 255, 255)
         )
```

### `--night` の場合

```text
main()
  └─ apply_night_theme()
      └─ apply_explicit_theme(
             light=False,
             terminal_color_scheme=CONFIG["night_terminal_color_scheme"],
             vscode_theme=CONFIG["night_vscode_theme"],
             wallpaper_rgb=(0, 0, 0)
         )
```

`apply_explicit_theme()` の中で、以下が順に実行されます。

1. 現在の Windows テーマを確認
2. 必要なら Windows テーマを変更
3. 壁紙色を変更
4. Windows Terminal のカラースキームを変更
5. VS Code のテーマを変更
6. Terminal または VS Code の設定を変更した場合、環境変更通知を送信

---

## 必要な環境・依存関係

### 必須

* Windows
* Python 3.11 以上なら標準ライブラリだけで動作
* Python 3.10 以下なら `tomli` が必要

スクリプトでは以下のように処理しています。

```python
try:
    import tomllib
except ModuleNotFoundError:
    import tomli as tomllib
```

Python 3.11 以降では `tomllib` が標準搭載されています。Python 3.10 以下では以下が必要です。

```powershell
pip install tomli
```

---

## 注意点

### 1. Windows 専用

このスクリプトは以下を使っています。

```python
import winreg
ctypes.WinDLL("ole32")
ctypes.WinDLL("user32")
```

そのため、macOS や Linux では動きません。

---

### 2. Windows Terminal がインストールされている必要がある

`Microsoft.WindowsTerminal_*` パッケージフォルダを探しているため、Windows Terminal が未インストールだと以下のエラーになります。

```text
FileNotFoundError: Windows Terminal settings.json was not found.
```

---

### 3. `config.toml` がないと起動時に失敗する

この行で、モジュール読み込み時に設定ファイルを読み込みます。

```python
CONFIG = load_config()
```

そのため、`config.toml` がない場合は、`--help` 以外の実行以前に失敗する可能性があります。

---

### 4. JSON のコメントは消える可能性がある

Windows Terminal や VS Code の `settings.json` を `json.load()` / `json.dumps()` で読み書きしています。

標準 JSON として処理されるため、以下に注意が必要です。

* コメント付き JSONC には弱い
* ファイル全体が再整形される
* コメントは保持されない
* キー順も変わる可能性がある

VS Code の `settings.json` は実質 JSONC としてコメントを書けるため、コメントを使っている場合は注意が必要です。

---

## まとめ

このスクリプトは、次の 4 つをまとめて切り替える Windows 用テーマ切替ツールです。

| 対象               | `--day`   | `--night` |
| ---------------- | --------- | --------- |
| Windows テーマ      | ライト       | ダーク       |
| 壁紙               | 白単色       | 黒単色       |
| Windows Terminal | 昼用カラースキーム | 夜用カラースキーム |
| VS Code          | 昼用テーマ     | 夜用テーマ     |

基本的な実行コマンドは以下です。

```powershell
python theme_switcher.py --day
python theme_switcher.py --night
```

設定値は、同じフォルダの `config.toml` で管理します。

```python
import argparse
import ctypes
import json
import logging
import os
import uuid
from ctypes import wintypes
from pathlib import Path
import winreg

try:
    import tomllib
except ModuleNotFoundError:
    import tomli as tomllib

PERSONALIZE_KEY = "Software\\Microsoft\\Windows\\CurrentVersion\\Themes\\Personalize"
APPS_KEY = "AppsUseLightTheme"
SYSTEM_KEY = "SystemUsesLightTheme"

HWND_BROADCAST = 0xFFFF
WM_SETTINGCHANGE = 0x001A
SMTO_ABORTIFHUNG = 0x0002

CONFIG_PATH = Path(__file__).with_name("config.toml")

CLSID_DESKTOP_WALLPAPER = "C2CF3110-460E-4FC1-B9D0-8A1C0C9CC4BD"
IID_IDESKTOP_WALLPAPER = "B92B56A9-8B55-4E14-9A89-0199BBB6F93B"

COINIT_APARTMENTTHREADED = 0x2
CLSCTX_ALL = 0x17
S_OK = 0
S_FALSE = 1
RPC_E_CHANGED_MODE = 0x80010106

DAY_WALLPAPER_RGB = (255, 255, 255)
NIGHT_WALLPAPER_RGB = (0, 0, 0)


if hasattr(wintypes, "ULONG_PTR"):
    ULONG_PTR = wintypes.ULONG_PTR
else:
    ULONG_PTR = ctypes.c_size_t


logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
logger = logging.getLogger(__name__)


class GUID(ctypes.Structure):
    _fields_ = [
        ("Data1", wintypes.DWORD),
        ("Data2", wintypes.WORD),
        ("Data3", wintypes.WORD),
        ("Data4", ctypes.c_ubyte * 8),
    ]

    @classmethod
    def from_string(cls, value: str) -> "GUID":
        return cls.from_buffer_copy(uuid.UUID(value).bytes_le)


def check_hresult(hr: int) -> None:
    if ctypes.c_long(hr).value < 0:
        raise OSError(f"HRESULT 0x{hr & 0xFFFFFFFF:08X}")


def rgb_to_colorref(r: int, g: int, b: int) -> int:
    """Return Windows COLORREF value. COLORREF is 0x00BBGGRR."""
    return (r & 0xFF) | ((g & 0xFF) << 8) | ((b & 0xFF) << 16)


def set_solid_wallpaper(r: int, g: int, b: int) -> None:
    """
    Set the Windows desktop background to a solid color without using an image file.

    Specify the background color via IDesktopWallpaper::SetBackgroundColor,
    and disable wallpaper image display via IDesktopWallpaper::Enable(False).
    """
    logger.debug("Setting solid desktop wallpaper color to RGB(%d, %d, %d)", r, g, b)

    ole32 = ctypes.WinDLL("ole32")

    ole32.CoInitializeEx.argtypes = [ctypes.c_void_p, wintypes.DWORD]
    ole32.CoInitializeEx.restype = ctypes.c_long

    ole32.CoUninitialize.argtypes = []
    ole32.CoUninitialize.restype = None

    ole32.CoCreateInstance.argtypes = [
        ctypes.POINTER(GUID),
        ctypes.c_void_p,
        wintypes.DWORD,
        ctypes.POINTER(GUID),
        ctypes.POINTER(ctypes.c_void_p),
    ]
    ole32.CoCreateInstance.restype = ctypes.c_long

    co_initialized_here = False
    hr = ole32.CoInitializeEx(None, COINIT_APARTMENTTHREADED)
    hr_unsigned = hr & 0xFFFFFFFF

    if hr in (S_OK, S_FALSE):
        co_initialized_here = True
    elif hr_unsigned == RPC_E_CHANGED_MODE:
        logger.debug("COM was already initialized with a different threading model")
    else:
        check_hresult(hr)

    p_wallpaper = ctypes.c_void_p()

    try:
        clsid = GUID.from_string(CLSID_DESKTOP_WALLPAPER)
        iid = GUID.from_string(IID_IDESKTOP_WALLPAPER)

        hr = ole32.CoCreateInstance(
            ctypes.byref(clsid),
            None,
            CLSCTX_ALL,
            ctypes.byref(iid),
            ctypes.byref(p_wallpaper),
        )
        check_hresult(hr)

        vtable = ctypes.cast(
            p_wallpaper,
            ctypes.POINTER(ctypes.POINTER(ctypes.c_void_p)),
        ).contents

        release = ctypes.WINFUNCTYPE(
            wintypes.ULONG,
            ctypes.c_void_p,
        )(vtable[2])

        set_background_color = ctypes.WINFUNCTYPE(
            ctypes.c_long,
            ctypes.c_void_p,
            wintypes.DWORD,
        )(vtable[8])

        enable = ctypes.WINFUNCTYPE(
            ctypes.c_long,
            ctypes.c_void_p,
            wintypes.BOOL,
        )(vtable[18])

        hr = set_background_color(p_wallpaper, rgb_to_colorref(r, g, b))
        check_hresult(hr)

        hr = enable(p_wallpaper, False)
        check_hresult(hr)

        logger.debug("Solid desktop wallpaper color applied")

    finally:
        if p_wallpaper:
            release(p_wallpaper)

        if co_initialized_here:
            ole32.CoUninitialize()


def set_day_wallpaper() -> None:
    set_solid_wallpaper(*DAY_WALLPAPER_RGB)


def set_night_wallpaper() -> None:
    set_solid_wallpaper(*NIGHT_WALLPAPER_RGB)


def load_config(path: Path = CONFIG_PATH) -> dict:
    if not path.is_file():
        raise FileNotFoundError(f"Config file was not found: {path}")

    with path.open("rb") as f:
        config = tomllib.load(f)

    return {
        "day_terminal_color_scheme": config["terminal"]["day_color_scheme"],
        "night_terminal_color_scheme": config["terminal"]["night_color_scheme"],
        "day_vscode_theme": config["vscode"]["day_theme"],
        "night_vscode_theme": config["vscode"]["night_theme"],
    }


CONFIG = load_config()


def get_windows_terminal_settings_path() -> Path:
    packages_dir = Path(os.environ["LOCALAPPDATA"]) / "Packages"
    logger.debug("Searching for Windows Terminal settings.json under: %s", packages_dir)

    for package_dir in packages_dir.glob("Microsoft.WindowsTerminal_*"):
        settings_path = package_dir / "LocalState" / "settings.json"
        logger.debug("Checking candidate path: %s", settings_path)
        if settings_path.is_file():
            logger.debug("Found Windows Terminal settings.json: %s", settings_path)
            return settings_path

    raise FileNotFoundError("Windows Terminal settings.json was not found.")


def get_vscode_settings_path() -> Path:
    path = Path.home() / "AppData" / "Roaming" / "Code" / "User" / "settings.json"
    logger.debug("Using VS Code settings.json path: %s", path)
    return path


def get_current_theme() -> int:
    try:
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, PERSONALIZE_KEY) as key:
            value, regtype = winreg.QueryValueEx(key, APPS_KEY)
            if regtype == winreg.REG_DWORD:
                logger.debug("Current Windows app theme registry value: %s", value)
                return int(value)
    except FileNotFoundError:
        logger.debug("Theme registry key/value not found. Falling back to light theme.")

    return 1


def send_setting_change(param: str) -> None:
    logger.debug("Broadcasting WM_SETTINGCHANGE with param=%r", param)

    user32 = ctypes.WinDLL("user32", use_last_error=True)
    send_message_timeout = user32.SendMessageTimeoutW
    send_message_timeout.argtypes = [
        wintypes.HWND,
        wintypes.UINT,
        wintypes.WPARAM,
        wintypes.LPCWSTR,
        wintypes.UINT,
        wintypes.UINT,
        ctypes.POINTER(ULONG_PTR),
    ]
    send_message_timeout.restype = wintypes.LPARAM

    result = ULONG_PTR()
    ret = send_message_timeout(
        HWND_BROADCAST,
        WM_SETTINGCHANGE,
        0,
        param,
        SMTO_ABORTIFHUNG,
        5000,
        ctypes.byref(result),
    )

    if ret == 0:
        raise ctypes.WinError(ctypes.get_last_error())

    logger.debug("WM_SETTINGCHANGE broadcast completed successfully for param=%r", param)


def notify_theme_changed() -> None:
    logger.debug("Notifying system that theme has changed")
    send_setting_change("ImmersiveColorSet")


def notify_environment_changed() -> None:
    logger.debug("Notifying system that environment/settings may have changed")
    send_setting_change("Environment")


def set_theme(light: bool) -> None:
    value = 1 if light else 0
    logger.debug("Setting Windows theme to: %s", "light" if light else "dark")

    with winreg.CreateKey(winreg.HKEY_CURRENT_USER, PERSONALIZE_KEY) as key:
        winreg.SetValueEx(key, APPS_KEY, 0, winreg.REG_DWORD, value)
        winreg.SetValueEx(key, SYSTEM_KEY, 0, winreg.REG_DWORD, value)

    notify_theme_changed()
    logger.debug("Windows theme updated")


def write_json_if_changed(path: Path, data: dict, description: str) -> bool:
    new_text = json.dumps(data, ensure_ascii=False, indent=4) + "\n"

    old_text = None
    if path.exists():
        old_text = path.read_text(encoding="utf-8")

    if old_text == new_text:
        logger.debug("%s already matches desired content; no update needed", description)
        return False

    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(new_text, encoding="utf-8", newline="\n")
    logger.debug("%s updated: %s", description, path)
    return True


def set_windows_terminal_color_scheme(color_scheme: str) -> bool:
    settings_path = get_windows_terminal_settings_path()
    logger.debug("Loading Windows Terminal settings from: %s", settings_path)

    with settings_path.open("r", encoding="utf-8") as f:
        settings = json.load(f)

    profiles = settings.setdefault("profiles", {})
    defaults = profiles.setdefault("defaults", {})

    current_scheme = defaults.get("colorScheme")
    logger.debug("Current Terminal default colorScheme: %r", current_scheme)
    logger.debug("Desired Terminal default colorScheme: %r", color_scheme)

    if current_scheme == color_scheme:
        return False

    defaults["colorScheme"] = color_scheme
    return write_json_if_changed(settings_path, settings, "Windows Terminal settings.json")


def set_vscode_color_theme(theme_name: str) -> bool:
    settings_path = get_vscode_settings_path()

    if not settings_path.is_file():
        raise FileNotFoundError(f"VS Code settings.json was not found: {settings_path}")

    text = settings_path.read_text(encoding="utf-8")

    key = '"workbench.colorTheme"'
    key_pos = text.find(key)

    if key_pos == -1:
        raise ValueError(
            f'{key} was not found in VS Code settings.json: {settings_path}'
        )

    colon_pos = text.find(":", key_pos + len(key))
    if colon_pos == -1:
        raise ValueError(f"Malformed {key} setting in: {settings_path}")

    value_start = colon_pos + 1

    while value_start < len(text) and text[value_start].isspace():
        value_start += 1

    if value_start >= len(text) or text[value_start] != '"':
        raise ValueError(f"{key} does not have a quoted string value")

    value_end = value_start + 1

    while value_end < len(text):
        if text[value_end] == '"' and text[value_end - 1] != "\\":
            break
        value_end += 1

    if value_end >= len(text):
        raise ValueError(f"Unterminated string value for {key}")

    current_theme = text[value_start + 1 : value_end]

    logger.debug("Current VS Code color theme: %r", current_theme)
    logger.debug("Desired VS Code color theme: %r", theme_name)

    if current_theme == theme_name:
        return False

    escaped_theme_name = (
        theme_name
        .replace("\\", "\\\\")
        .replace('"', '\\"')
    )

    new_text = (
        text[: value_start + 1]
        + escaped_theme_name
        + text[value_end:]
    )

    settings_path.write_text(new_text, encoding="utf-8", newline="\n")
    logger.debug("VS Code settings.json updated: %s", settings_path)
    return True


def apply_explicit_theme(
    *,
    light: bool,
    terminal_color_scheme: str,
    vscode_theme: str,
    wallpaper_rgb: tuple[int, int, int],
) -> bool:
    """
    Apply the specified theme immediately without using time-based determination.

    Used by the CLI options --day / --night.
    """
    current_is_light = bool(get_current_theme())

    if current_is_light != light:
        set_theme(light)

    set_solid_wallpaper(*wallpaper_rgb)

    terminal_changed = set_windows_terminal_color_scheme(terminal_color_scheme)
    vscode_changed = set_vscode_color_theme(vscode_theme)

    if terminal_changed or vscode_changed:
        try:
            notify_environment_changed()
        except OSError:
            logger.exception("Failed to broadcast environment change notification")

    logger.info(
        "Applied explicit %s theme: terminal=%r, vscode=%r, wallpaper_rgb=%r",
        "day/light" if light else "night/dark",
        terminal_color_scheme,
        vscode_theme,
        wallpaper_rgb,
    )

    return light


def apply_day_theme() -> bool:
    return apply_explicit_theme(
        light=True,
        terminal_color_scheme=CONFIG["day_terminal_color_scheme"],
        vscode_theme=CONFIG["day_vscode_theme"],
        wallpaper_rgb=DAY_WALLPAPER_RGB,
    )


def apply_night_theme() -> bool:
    return apply_explicit_theme(
        light=False,
        terminal_color_scheme=CONFIG["night_terminal_color_scheme"],
        vscode_theme=CONFIG["night_vscode_theme"],
        wallpaper_rgb=NIGHT_WALLPAPER_RGB,
    )


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Switch Windows, Windows Terminal, and VS Code themes explicitly."
    )

    mode_group = parser.add_mutually_exclusive_group(required=True)
    mode_group.add_argument(
        "--day",
        action="store_true",
        help="Apply day/light settings immediately and exit. Ignores current time.",
    )
    mode_group.add_argument(
        "--night",
        action="store_true",
        help="Apply night/dark settings immediately and exit. Ignores current time.",
    )

    return parser.parse_args()


def main() -> None:
    args = parse_args()

    if args.day:
        apply_day_theme()
        return

    if args.night:
        apply_night_theme()
        return


if __name__ == "__main__":
    main()
```
