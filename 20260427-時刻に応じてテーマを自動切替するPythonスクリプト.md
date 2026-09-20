---
pubDatetime: 2026-04-27T16:27:48+09:00
title: "時刻に応じてテーマを自動切替するPythonスクリプト"
description: ":::note warn 筆者は、自動切換え機能は不要だと思ったので、使わなくなってしまいました。代わりに手動切り替えのみするスクリプトをこちらに記事にしました。単色壁紙も設定します。 https://qiita.com/vast-cow/items/3215d89727cd47cefbc1 :::…"
---

:::note warn
筆者は、自動切換え機能は不要だと思ったので、使わなくなってしまいました。代わりに手動切り替えのみするスクリプトをこちらに記事にしました。単色壁紙も設定します。
https://qiita.com/vast-cow/items/3215d89727cd47cefbc1
:::

このスクリプトは、Windowsの外観テーマ（ライト／ダーク）および、Windows TerminalとVisual Studio Codeのカラーテーマを、時刻に応じて自動的に切り替えるためのツールです。

## 目的

日中と夜間で画面の見やすさを最適化することが目的です。例えば：

* 昼間：明るいライトテーマ
* 夜間：目に優しいダークテーマ

これらを手動で切り替えるのではなく、設定した時間に基づいて自動化します。

## 基本的な仕組み

このスクリプトは以下の流れで動作します：

1. 設定ファイル（`config.toml`）から時間とテーマを読み込む
2. 現在時刻を取得
3. 「昼」か「夜」かを判定
4. 対応するテーマを適用
5. 次の切替時刻まで待機し、繰り返す

## 設定方法

スクリプトと同じディレクトリにある `config.toml` に以下を定義します：

* 昼開始時刻（例：08:00）
* 夜開始時刻（例：20:00）
* Terminalの昼・夜のカラースキーム
* VS Codeの昼・夜のテーマ

これにより、自分の作業スタイルに合わせた切替が可能になります。

## 使用方法

### 自動スケジューリングで実行

```bash
python script.py
```

この場合、スクリプトは常駐し、指定時間に応じて自動でテーマを切り替えます。

### 手動で即時切替

* 昼テーマを適用：

```bash
python script.py --day
```

* 夜テーマを適用：

```bash
python script.py --night
```

これらは現在時刻を無視して、即座に指定のテーマを適用します。

## 主な機能

* Windowsのライト／ダークテーマ切替
* Windows Terminalのカラースキーム変更
* VS Codeのテーマ変更
* 設定変更をOSに通知して即時反映
* JSON設定ファイルの安全な更新（変更時のみ書き込み）

## 利用シーン

* 長時間作業時の目の負担軽減
* 開発環境の見やすさ改善
* 手動操作の削減による効率化

## まとめ

このスクリプトは、日常的に使用する開発環境の見た目を自動的に最適化するシンプルなツールです。設定を一度行えば、以降は意識せずに快適な表示環境を維持できます。

```python
import argparse
import asyncio
import ctypes
import json
import logging
import os
from ctypes import wintypes
from datetime import datetime, timedelta
from pathlib import Path
import winreg

try:
    import tomllib
except ModuleNotFoundError:
    import tomli as tomllib

from sleep_absolute import wait_until


PERSONALIZE_KEY = "Software\\Microsoft\\Windows\\CurrentVersion\\Themes\\Personalize"
APPS_KEY = "AppsUseLightTheme"
SYSTEM_KEY = "SystemUsesLightTheme"

HWND_BROADCAST = 0xFFFF
WM_SETTINGCHANGE = 0x001A
SMTO_ABORTIFHUNG = 0x0002

CONFIG_PATH = Path(__file__).with_name("config.toml")


if hasattr(wintypes, "ULONG_PTR"):
    ULONG_PTR = wintypes.ULONG_PTR
else:
    ULONG_PTR = ctypes.c_size_t


logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
logger = logging.getLogger(__name__)


def parse_hhmm(value: str) -> tuple[int, int]:
    try:
        hour_text, minute_text = value.split(":", 1)
        hour = int(hour_text)
        minute = int(minute_text)
    except ValueError as exc:
        raise ValueError(f"Invalid time format: {value!r}. Expected HH:MM.") from exc

    if not (0 <= hour <= 23):
        raise ValueError(f"Invalid hour in time: {value!r}")

    if not (0 <= minute <= 59):
        raise ValueError(f"Invalid minute in time: {value!r}")

    return hour, minute


def load_config(path: Path = CONFIG_PATH) -> dict:
    if not path.is_file():
        raise FileNotFoundError(f"Config file was not found: {path}")

    with path.open("rb") as f:
        config = tomllib.load(f)

    day_start_hour, day_start_minute = parse_hhmm(
        config["schedule"]["day_start"]
    )
    night_start_hour, night_start_minute = parse_hhmm(
        config["schedule"]["night_start"]
    )

    return {
        "day_terminal_color_scheme": config["terminal"]["day_color_scheme"],
        "night_terminal_color_scheme": config["terminal"]["night_color_scheme"],
        "day_vscode_theme": config["vscode"]["day_theme"],
        "night_vscode_theme": config["vscode"]["night_theme"],
        "day_start_hour": day_start_hour,
        "day_start_minute": day_start_minute,
        "night_start_hour": night_start_hour,
        "night_start_minute": night_start_minute,
        "day_start_minutes": day_start_hour * 60 + day_start_minute,
        "night_start_minutes": night_start_hour * 60 + night_start_minute,
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


def desired_theme_for_now(now: datetime | None = None) -> bool:
    now = now or datetime.now()
    minutes = now.hour * 60 + now.minute

    is_light = (
        CONFIG["day_start_minutes"]
        <= minutes
        < CONFIG["night_start_minutes"]
    )

    logger.debug(
        "Evaluated desired Windows theme at %s -> %s",
        now.isoformat(),
        "light" if is_light else "dark",
    )
    return is_light


def desired_terminal_color_scheme_for_now(now: datetime | None = None) -> str:
    now = now or datetime.now()

    scheme = (
        CONFIG["day_terminal_color_scheme"]
        if desired_theme_for_now(now)
        else CONFIG["night_terminal_color_scheme"]
    )

    logger.debug(
        "Evaluated desired Terminal color scheme at %s -> %s",
        now.isoformat(),
        scheme,
    )
    return scheme


def desired_vscode_theme_for_now(now: datetime | None = None) -> str:
    now = now or datetime.now()

    theme = (
        CONFIG["day_vscode_theme"]
        if desired_theme_for_now(now)
        else CONFIG["night_vscode_theme"]
    )

    logger.debug(
        "Evaluated desired VS Code theme at %s -> %s",
        now.isoformat(),
        theme,
    )
    return theme


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

    if settings_path.exists():
        with settings_path.open("r", encoding="utf-8") as f:
            settings = json.load(f)
    else:
        settings = {}

    current_theme = settings.get("workbench.colorTheme")

    if current_theme == theme_name:
        return False

    settings["workbench.colorTheme"] = theme_name
    return write_json_if_changed(settings_path, settings, "VS Code settings.json")


def apply_explicit_theme(
    *,
    light: bool,
    terminal_color_scheme: str,
    vscode_theme: str,
) -> bool:
    """
    時刻判定を使わず、指定されたテーマを即時適用する。
    CLI の --day / --night から使う。
    """
    current_is_light = bool(get_current_theme())

    if current_is_light != light:
        set_theme(light)

    terminal_changed = set_windows_terminal_color_scheme(terminal_color_scheme)
    vscode_changed = set_vscode_color_theme(vscode_theme)

    if terminal_changed or vscode_changed:
        try:
            notify_environment_changed()
        except OSError:
            logger.exception("Failed to broadcast environment change notification")

    logger.info(
        "Applied explicit %s theme: terminal=%r, vscode=%r",
        "day/light" if light else "night/dark",
        terminal_color_scheme,
        vscode_theme,
    )

    return light


def apply_day_theme() -> bool:
    return apply_explicit_theme(
        light=True,
        terminal_color_scheme=CONFIG["day_terminal_color_scheme"],
        vscode_theme=CONFIG["day_vscode_theme"],
    )


def apply_night_theme() -> bool:
    return apply_explicit_theme(
        light=False,
        terminal_color_scheme=CONFIG["night_terminal_color_scheme"],
        vscode_theme=CONFIG["night_vscode_theme"],
    )


def apply_theme_for_now() -> bool:
    now = datetime.now()

    should_use_light_theme = desired_theme_for_now(now)
    current_is_light = bool(get_current_theme())

    if current_is_light != should_use_light_theme:
        set_theme(should_use_light_theme)

    desired_terminal_scheme = desired_terminal_color_scheme_for_now(now)
    terminal_changed = set_windows_terminal_color_scheme(desired_terminal_scheme)

    desired_vscode_theme = desired_vscode_theme_for_now(now)
    vscode_changed = set_vscode_color_theme(desired_vscode_theme)

    if terminal_changed or vscode_changed:
        try:
            notify_environment_changed()
        except OSError:
            logger.exception("Failed to broadcast environment change notification")

    return should_use_light_theme


def next_switch_datetime(now: datetime | None = None) -> datetime:
    now = now or datetime.now()

    today_day_start = now.replace(
        hour=CONFIG["day_start_hour"],
        minute=CONFIG["day_start_minute"],
        second=0,
        microsecond=0,
    )

    today_night_start = now.replace(
        hour=CONFIG["night_start_hour"],
        minute=CONFIG["night_start_minute"],
        second=0,
        microsecond=0,
    )

    if now < today_day_start:
        next_dt = today_day_start
    elif now < today_night_start:
        next_dt = today_night_start
    else:
        next_dt = today_day_start + timedelta(days=1)

    logger.debug("Next scheduled switch time: %s", next_dt.isoformat())
    return next_dt


async def scheduler() -> None:
    apply_theme_for_now()

    while True:
        next_dt = next_switch_datetime()
        await wait_until(next_dt)
        apply_theme_for_now()


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Switch Windows, Windows Terminal, and VS Code themes by schedule."
    )

    mode_group = parser.add_mutually_exclusive_group()
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

    asyncio.run(scheduler())


if __name__ == "__main__":
    main()
```
